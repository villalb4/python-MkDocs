# Preparar la app para producción

!!! info "En esta página"

    **Qué vas a aprender:** qué cambia entre desarrollo y producción, por qué `debug` tiene que estar apagado, cómo evitar que los secretos aparezcan en logs y errores con `SecretStr`, cómo hacer que la app se niegue a arrancar con una configuración insegura, y qué variables definir al publicarla.

    **Qué necesitás:** [Logs: qué registrar y qué no](04-logs.md) y las [recomendaciones de seguridad](../autenticacion/12-recomendaciones-seguridad.md) del Capítulo 5.

## El mismo código, otra configuración

**Producción** es la API publicada, la que usan las personas de verdad. El código es el mismo que en tu computadora; lo que cambia es la **configuración**. Por eso, desde el Capítulo 2, todo lo que depende del entorno sale de variables de entorno. Esta página repasa esa configuración para que esté lista para publicar.

## Apagar `debug`

Para ver qué cambia, agregá por un momento esta ruta al final de `app/main.py`:

```python
@app.get("/romper")
async def romper():
    return 1 / 0
```

Con `DEBUG=true`, `curl -i http://127.0.0.1:8000/romper` devuelve el **traceback entero** (unos 5000 caracteres):

```text
HTTP/1.1 500 Internal Server Error
content-type: text/plain; charset=utf-8

Traceback (most recent call last):
  File "/.../tareas-api/.venv/lib/python3.12/site-packages/starlette/middleware/errors.py", line 164, in __call__
...
  File "/.../tareas-api/app/main.py", line 62, in romper
    return 1 / 0
           ~~^~~
ZeroDivisionError: division by zero
```

Para vos es útil. Para un atacante también: le muestra las rutas de tu servidor, las librerías que usás, tu código y, según el error, hasta datos. Con `DEBUG=false`, la misma ruta responde solo:

```text
HTTP/1.1 500 Internal Server Error
content-type: text/plain; charset=utf-8

Internal Server Error
```

En los dos casos, el traceback queda en el log del servidor (`ERROR:    Exception in ASGI application`), que es donde tiene que estar. Además, con `debug` apagado, SQLAlchemy deja de registrar el SQL con sus parámetros, como viste en la página anterior.

Fijate que en `Configuracion` el valor por defecto es `debug: bool = False`. Es a propósito: si en producción alguien se olvida de definir `DEBUG`, queda apagado. Borrá la ruta `/romper` antes de seguir.

## Secretos que no se muestran: `SecretStr`

En la página anterior viste que registrar la configuración filtraba la clave de los JWT y la contraseña de la base. Pydantic tiene un tipo para eso: **`SecretStr`**. Guarda el texto, pero al mostrarlo lo tapa con asteriscos. Aprovechá también para borrar `clave_api`, que no se usa desde el Capítulo 5 (y su `CLAVE_API` del `.env`). `app/config.py` queda así:

```python title="app/config.py"
from functools import cache
from pathlib import Path

from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

BASE_DIR = Path(__file__).resolve().parent.parent


class Configuracion(BaseSettings):
    model_config = SettingsConfigDict(env_file=BASE_DIR / ".env", extra="ignore")

    nombre_app: str = "API de tareas"
    debug: bool = False
    origenes_permitidos: list[str] = []
    url_base_datos: SecretStr
    clave_jwt: SecretStr = Field(min_length=32)
    algoritmo_jwt: str = "HS256"
    minutos_token: int = 15
    dias_refresco: int = 7
    nivel_log: str = "INFO"


@cache
def obtener_configuracion() -> Configuracion:
    return Configuracion()
```

Para usar el valor real hay que pedirlo explícitamente, con `.get_secret_value()`. Son cuatro lugares:

| Archivo | Queda |
| --- | --- |
| `app/db.py` | `create_async_engine(configuracion.url_base_datos.get_secret_value())` |
| `migraciones/env.py` | `obtener_configuracion().url_base_datos.get_secret_value()` |
| `app/seguridad.py` (en `crear_token`) | `jwt.encode(datos, configuracion.clave_jwt.get_secret_value(), algorithm=...)` |
| `app/seguridad.py` (en `leer_token`) | `jwt.decode(token, configuracion.clave_jwt.get_secret_value(), algorithms=[...])` |

Ahora el mismo log de la configuración sale así:

```text
2026-10-25 02:05:32,953 INFO app.main: Configuración: nombre_app='Tareas (desarrollo)' debug=True origenes_permitidos=['http://localhost:5500'] url_base_datos=SecretStr('**********') clave_jwt=SecretStr('**********') ...
```

Lo mismo pasa con cualquier `print` o mensaje que incluya la configuración: para ver el secreto hay que pedirlo a propósito.

## Que no arranque con una configuración insegura

El `Field(min_length=32)` de `clave_jwt` hace que la app **se niegue a arrancar** si la clave es corta. Si alguien deja en producción una clave como `cambiame`, el error aparece al levantar la API, y no meses después:

```text
pydantic_core._pydantic_core.ValidationError: 1 validation error for Configuracion
clave_jwt
  Value should have at least 32 items after validation, not 8 [type=too_short, input_value='cambiame', input_type=str]
```

Ojo: el error muestra el valor que llegó (`input_value`), así que también es un mensaje para no compartir. Es la misma idea que los campos obligatorios: es mejor que la app no arranque a que arranque mal. Corré `uv run pytest` para comprobar que todo sigue andando: `14 passed`.

## Las variables en producción

En producción, la configuración no sale de tu `.env` de desarrollo (que no se sube a Git), sino de las **variables de entorno de la plataforma** donde publiques. Cada plataforma tiene una pantalla o un archivo para definirlas. Estas son las que cambian:

| Variable | En producción |
| --- | --- |
| `DEBUG` | `false` (o no definirla: el valor por defecto ya es `false`). |
| `CLAVE_JWT` | Una nueva, generada con `secrets.token_hex(32)`. Nunca la de desarrollo. |
| `URL_BASE_DATOS` | La de la base de producción, con su propio usuario y contraseña. |
| `ORIGENES_PERMITIDOS` | El dominio real del frontend, por ejemplo `["https://tareas.ejemplo.com"]`. |
| `NIVEL_LOG` | `INFO` (o `WARNING`, si hay demasiados mensajes). |

Con la imagen de Docker del Capítulo 4, una variable se puede pisar al arrancar con `-e`, que tiene prioridad sobre `--env-file`: `docker run --env-file .env -e DEBUG=false ...`.

!!! warning "Con Compose no alcanza con el shell"

    `DEBUG=false docker compose up` **no** apaga `debug`: la variable del shell solo se usa para completar los `${...}` del `docker-compose.yml`, y el contenedor sigue recibiendo `DEBUG=true` del `env_file`. Para cambiarla, va en `environment:` del servicio o en el archivo de variables.

## `fastapi run` y no `fastapi dev`

En producción la API se levanta con `fastapi run`, como en el Dockerfile: sin recarga automática (que consume recursos y no tiene sentido si el código no cambia) y escuchando en `0.0.0.0`.

`fastapi run --workers 4` levanta 4 procesos para atender más pedidos a la vez. Pero ojo: cada proceso tiene su propia memoria, y el [límite de intentos](../autenticacion/12-recomendaciones-seguridad.md) guarda los intentos en un diccionario. Con `--workers 2`, en una prueba hubo **siete** logins fallidos antes del primer `429`, en vez de cinco. Para varios procesos, el límite tiene que estar en un almacenamiento compartido o en la plataforma.

!!! tip "¿Y `/docs`?"

    La documentación automática queda pública. Para muchas APIs está bien, pero si preferís esconderla en producción, `FastAPI(docs_url=None, redoc_url=None, openapi_url=None)` la apaga (las tres rutas responden `404`). Se puede condicionar a `configuracion.debug`.

## Lista para antes de publicar

- `DEBUG` apagado.
- Secretos nuevos, largos y solo en la plataforma, como `SecretStr` en la configuración.
- CORS con los orígenes reales.
- Migraciones aplicadas en la base de producción (`alembic upgrade head`).
- `uv run ruff check` y `uv run pytest` sin errores.
- HTTPS delante de la API (lo da la plataforma o el proxy).

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Node, el equivalente de `debug` es `NODE_ENV=production`: Express deja de mandar el stack trace en los errores y activa algunas optimizaciones. La validación de la configuración al arrancar se suele hacer con zod o envalid, y la app se levanta con `node` (o un gestor de procesos como PM2) en vez de `nodemon`.

??? example "Ver comparación en Django ⬇️"

    Es el famoso `DEBUG = False` de `settings.py`, que además exige definir `ALLOWED_HOSTS`. Django trae `python manage.py check --deploy`, que revisa una lista de configuraciones inseguras, la misma idea que esta página. En producción se usa Gunicorn o Uvicorn en vez de `runserver`.

??? example "Ver comparación en Spring Boot ⬇️"

    Spring usa **perfiles**: `application-prod.properties` con la configuración de producción, activado con `SPRING_PROFILES_ACTIVE=prod`. `server.error.include-stacktrace=never` cumple el papel de `debug` apagado, y la validación al arrancar se hace con `@ConfigurationProperties` y `@Validated`.

## Mini ejercicio

Si en el `.env` escribís `NIVEL_LOG=info` (en minúsculas), la API no arranca, pero con un error poco claro de `logging`: `ValueError: Unknown level: 'info'`. Cambiá el tipo de `nivel_log` para que pydantic-settings lo rechace al leer la configuración, con un mensaje que diga qué valores acepta.

??? success "Ver solución"

    Con `Literal`, que viste en [Roles y permisos](../autenticacion/09-roles.md):

    ```python
    from typing import Literal

    nivel_log: Literal["DEBUG", "INFO", "WARNING", "ERROR"] = "INFO"
    ```

    Ahora el error aparece al leer la configuración, y dice exactamente qué está mal:

    ```text
    nivel_log
      Input should be 'DEBUG', 'INFO', 'WARNING' or 'ERROR' [type=literal_error, input_value='info', input_type=str]
    ```

    Es la misma idea que el `min_length` de `clave_jwt`: que la configuración se valide entera antes de que la app empiece a atender pedidos.

---

**Siguiente:** [Desplegar el backend →](06-desplegar.md)
