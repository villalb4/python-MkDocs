# Configuración con `pydantic-settings`

!!! info "En esta página"

    **Qué vas a aprender:** cómo leer la configuración de tu API desde variables de entorno y un archivo `.env` con `pydantic-settings`, con tipos y validación, y cómo usarla en las rutas a través de una dependencia.

    **Qué necesitás:** [Inyección de dependencias con `Depends`](11-depends.md), [Variables de entorno y archivos `.env`](../python/15-variables-de-entorno.md) y [Rutas con `pathlib`](../python/14-pathlib.md).

## El problema

En la página anterior, la clave del router de usuarios quedó escrita en el código: `"secreta"`. Es justo lo que no hay que hacer con un secreto. Y pronto vas a tener más datos que cambian según dónde corre la API: la dirección de la base de datos, si está en modo desarrollo, la clave para firmar tokens.

En el Capítulo 1 viste cómo leer variables de entorno con `os.environ` y las trampas que tiene: todo llega como texto, `bool("false")` da `True` y nadie te avisa si falta una variable. `pydantic-settings` resuelve todo eso con algo que ya conocés: un modelo de Pydantic.

## Instalarlo

```bash
uv add pydantic-settings
```

Ya viene incluido en `fastapi[standard]`, pero agregarlo explícitamente deja claro en `pyproject.toml` que tu código lo usa.

## La clase de configuración

Creá `app/config.py`:

```python title="app/config.py"
from functools import cache
from pathlib import Path

from pydantic_settings import BaseSettings, SettingsConfigDict

BASE_DIR = Path(__file__).resolve().parent.parent


class Configuracion(BaseSettings):
    model_config = SettingsConfigDict(env_file=BASE_DIR / ".env")

    nombre_app: str = "API de tareas"
    debug: bool = False
    clave_api: str


@cache
def obtener_configuracion() -> Configuracion:
    return Configuracion()
```

Parte por parte:

- **`BaseSettings`** funciona como `BaseModel`, pero los valores no se los pasás vos: los busca en las **variables de entorno**. El campo `clave_api` se llena con la variable `CLAVE_API` (sin importar mayúsculas).
- **Los type hints convierten y validan:** `debug: bool` entiende `true`, `false`, `1`, `0`, como en los parámetros de query. Se acabó la trampa de `bool("false")`.
- **Sin valor por defecto es obligatorio:** si `CLAVE_API` no está definida, la app **no arranca**.
- **`env_file`** le dice dónde está el archivo `.env`. Usamos `BASE_DIR`, como en la página de `pathlib`, para que lo encuentre sin importar desde qué carpeta ejecutes.
- **`@cache`** hace que `obtener_configuracion` cree el objeto una sola vez y después devuelva siempre el mismo, en vez de releer el `.env` en cada pedido. Es el decorador que viste en la página de decoradores.

## El archivo `.env`

En la raíz del proyecto (al lado de `pyproject.toml`), creá el `.env`:

```bash title=".env"
NOMBRE_APP="Tareas (desarrollo)"
DEBUG=true
CLAVE_API=una-clave-larga-de-desarrollo
```

Y, como viste en el Capítulo 1: el `.env` **no va a Git**. El `.gitignore` que creó `uv init` no lo incluye, así que agregale la línea `.env`. Lo que sí se sube es un `.env.example` con los nombres y valores de ejemplo.

Si una variable está definida **en la terminal y en el `.env`**, gana la de la terminal. Así podés pisar un valor puntual sin tocar el archivo: `CLAVE_API=otra uv run fastapi dev app/main.py`.

## Si falta algo, la app no arranca

Borrá (o renombrá) el `.env` y levantá el servidor. En vez de arrancar con una clave vacía, la app falla de entrada con un error claro:

```text
ValidationError: 1 validation error for Configuracion
clave_api
  Field required [type=missing, input_value={}, input_type=dict]
```

Es exactamente lo que querés: enterarte al arrancar, no cuando un usuario intenta usar la API. Lo mismo pasa con un valor inválido, como `DEBUG="tal vez"` (`Input should be a valid boolean`).

## Usar la configuración

**En `main.py`**, para datos que se necesitan al crear la app, se llama a la función directamente:

```python title="app/main.py"
from fastapi import FastAPI

from app.config import obtener_configuracion
from app.routers import tareas, usuarios

configuracion = obtener_configuracion()

app = FastAPI(
    title=configuracion.nombre_app,
    description="Una API para organizar tareas pendientes.",
    version="0.1.0",
    debug=configuracion.debug,
)

app.include_router(tareas.router)
app.include_router(usuarios.router)
```

En `/docs`, el título ahora es "Tareas (desarrollo)". `debug=True` hace que, ante un error `500`, la respuesta incluya el traceback: útil para desarrollar, peligroso en producción.

**En las rutas**, se pide como una dependencia. Así queda la verificación de la clave en `app/routers/usuarios.py`:

```python title="app/routers/usuarios.py (principio del archivo)"
from typing import Annotated

from fastapi import APIRouter, Depends, Header, HTTPException, status

from app.config import Configuracion, obtener_configuracion
from app.modelos import UsuarioCrear


def verificar_clave(
    x_clave: Annotated[str, Header()],
    configuracion: Annotated[Configuracion, Depends(obtener_configuracion)],
):
    if x_clave != configuracion.clave_api:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Clave incorrecta")
```

Ahora `X-Clave: secreta` da `401`, y la clave que funciona es la del `.env`. Fijate que una dependencia puede, a su vez, depender de otra: `verificar_clave` pide `obtener_configuracion`, y FastAPI resuelve la cadena solo.

!!! tip "¿Por qué una dependencia y no importar `configuracion`?"

    Funcionaría, pero con `Depends` se puede reemplazar la configuración en los tests (por ejemplo, con otra clave) sin tocar el código. Lo vas a ver en el Capítulo 6.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Node lo habitual es `dotenv` + `process.env`, y validar con algo como Zod o `envalid`:

    ```javascript
    require("dotenv").config()
    const config = z.object({
      CLAVE_API: z.string(),
      DEBUG: z.coerce.boolean().default(false),
    }).parse(process.env)
    ```

    `pydantic-settings` hace las tres cosas (leer el `.env`, convertir y validar) con una sola clase.

??? example "Ver comparación en Django ⬇️"

    En Django la configuración es `settings.py`, y para sacar los secretos de ahí se suele usar `django-environ` o `os.environ`. `pydantic-settings` ocupa ese lugar, con la ventaja de que los tipos y la validación vienen del propio modelo. Y en vez de `from django.conf import settings`, en FastAPI se pide con `Depends`.

??? example "Ver comparación en Spring Boot ⬇️"

    Es el equivalente de `@ConfigurationProperties`:

    | Spring Boot | FastAPI |
    | --- | --- |
    | `application.properties` | `.env` |
    | `@ConfigurationProperties` en una clase | `class Configuracion(BaseSettings)` |
    | Variables de entorno pisan las propiedades | Variables de entorno pisan el `.env` |
    | `@Validated` + `@NotBlank` | Campo sin valor por defecto |
    | Inyectar la clase con `@Autowired` | `Depends(obtener_configuracion)` |

## Mini ejercicio

Agregá a `Configuracion` un campo `max_tareas_por_pagina: int = 50` y usalo en la dependencia `paginacion` del router de tareas, para que `limite` no pueda superar ese valor (respondiendo `422` si lo supera). Probalo poniendo `MAX_TAREAS_POR_PAGINA=2` en el `.env`.

??? success "Ver solución"

    En `app/config.py`, agregá el campo:

    ```python
    max_tareas_por_pagina: int = 50
    ```

    En `app/routers/tareas.py`, `paginacion` pide la configuración y valida a mano:

    ```python
    from app.config import Configuracion, obtener_configuracion


    def paginacion(
        configuracion: Annotated[Configuracion, Depends(obtener_configuracion)],
        saltear: int = 0,
        limite: int = 10,
    ) -> dict:
        if limite > configuracion.max_tareas_por_pagina:
            raise HTTPException(
                status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
                detail=f"limite no puede superar {configuracion.max_tareas_por_pagina}",
            )
        return {"saltear": saltear, "limite": limite}
    ```

    Con `MAX_TAREAS_POR_PAGINA=2`, `/tareas?limite=3` responde `422` con ese detalle. `Query(le=...)` no sirve acá porque necesita un número fijo al escribir el código, y este valor recién se conoce al leer la configuración (podés sacar `Query` del import, porque ya no se usa). Como `@cache` guarda la configuración, después de cambiar el `.env` tenés que reiniciar el servidor (o guardar algún archivo `.py` para que se recargue).

---

**Siguiente:** [CORS (qué es y por qué aparece) →](13-cors.md)
