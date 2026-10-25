# Logs: qué registrar y qué no

!!! info "En esta página"

    **Qué vas a aprender:** qué es un log y por qué conviene más que `print`, cómo usar el módulo `logging` de Python en la API, qué eventos vale la pena registrar y qué datos no tienen que aparecer nunca en un log.

    **Qué necesitás:** [Linter y formateo con Ruff](03-ruff.md). Las páginas de [lifespan](../fastapi/14-lifespan.md) y del [Capítulo 5](../autenticacion/12-recomendaciones-seguridad.md) te van a sonar.

## Por qué no `print`

En el [lifespan](../fastapi/14-lifespan.md) usaste `print` para avisar qué hacía la app. Para probar está bien, pero cuando la API está publicada y algo sale mal, lo único que tenés para entender qué pasó son esos mensajes. Y un `print` no dice **cuándo** pasó, **dónde** (en qué parte del código) ni **qué tan grave** es, y no se puede apagar ni mandar a otro lado sin tocar el código.

Un **log** es un mensaje con todo eso. En Python se escriben con el módulo **`logging`**, que viene incluido. Cada mensaje tiene un **nivel**:

| Nivel | Para qué |
| --- | --- |
| `DEBUG` | Detalles para buscar un error. Normalmente apagado. |
| `INFO` | Cosas normales que vale la pena saber: la app arrancó, alguien inició sesión. |
| `WARNING` | Algo raro, pero la app sigue: un login fallido, demasiados intentos. |
| `ERROR` | Algo falló y un pedido no se pudo atender. |
| `CRITICAL` | Algo tan grave que la app no puede seguir. |

## Un logger por módulo

En cada archivo que registre algo, creá un *logger* con el nombre del módulo:

```python
import logging

logger = logging.getLogger(__name__)
```

`__name__` vale `"app.main"`, `"app.routers.auth"`, etc., y así cada mensaje dice de qué archivo viene. Ruff marca con `LOG015` si escribís `logging.info(...)` directamente, sin un logger propio.

Probalo agregando dos líneas al principio del `lifespan` de `app/main.py`:

```python
    logger.info("Arrancando %s", configuracion.nombre_app)
    logger.warning("Esto es un warning")
```

Al levantar la API con `uv run fastapi dev app/main.py`, pasa algo raro:

```text
INFO:     Waiting for application startup.
Esto es un warning
INFO:     Application startup complete.
```

El `INFO` no aparece, y el warning sale sin fecha ni nivel. Es porque **nadie configuró `logging`**: los `INFO:` que ves son de Uvicorn, que configura solo sus propios loggers. Sin configuración, Python muestra solo los mensajes desde `WARNING`, y sin formato. Borrá la línea del warning: era solo para ver esto.

## Configurar `logging`

El nivel conviene que salga de la configuración, para poder cambiarlo sin tocar el código. Agregá un campo a `Configuracion` en `app/config.py`:

```python
nivel_log: str = "INFO"
```

Y configurá `logging` al principio de `app/main.py`, después de leer la configuración:

```python title="app/main.py (principio del archivo)"
configuracion = obtener_configuracion()
logging.basicConfig(
    level=configuracion.nivel_log,
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
)
if configuracion.debug:
    # Muestra el SQL que ejecuta SQLAlchemy (reemplaza al echo=True del motor)
    logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
logger = logging.getLogger(__name__)
```

`basicConfig` define desde qué nivel se muestran los mensajes y con qué formato: fecha y hora, nivel, nombre del logger y mensaje. Lo del `if` se explica más abajo, en "Qué no registrar". Por ese `if`, en `app/db.py` sacá el `echo=configuracion.debug` del motor: queda `create_async_engine(configuracion.url_base_datos)`.

Con `NIVEL_LOG=DEBUG` en el `.env` ves más detalle, y con `NIVEL_LOG=WARNING`, menos.

## Qué registrar

Registrá los eventos que te ayudarían a entender qué pasó si algo sale mal. En la API:

- **El arranque y el apagado**, en el `lifespan` de `app/main.py`:

    ```python title="app/main.py"
    @asynccontextmanager
    async def lifespan(app: FastAPI):
        logger.info("Arrancando %s", configuracion.nombre_app)
        yield
        # Apagado: cierra las conexiones con la base
        await motor.dispose()
        logger.info("Conexiones con la base cerradas")
    ```

- **Lo que pasa con las cuentas**, en `app/routers/auth.py` (con su propio `logger = logging.getLogger(__name__)`): un registro nuevo (`logger.info("Usuario registrado: id %s", usuario.id)` antes del `return`), y en el login:

    ```python title="app/routers/auth.py (en iniciar_sesion)"
    if usuario is None or not verificar_contrasena(formulario.password, usuario.hash_contrasena):
        logger.warning("Login fallido para %s", email)
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Email o contraseña incorrectos")
    if not usuario.activo:
        logger.warning("Login de una cuenta desactivada: id %s", usuario.id)
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="La cuenta está desactivada")
    token_refresco = crear_token_refresco(usuario.id)
    guardar_cookie_refresco(respuesta, token_refresco)
    logger.info("Login correcto: id %s", usuario.id)
    return Token(access_token=crear_token_acceso(usuario.id))
    ```

- **Los pedidos bloqueados por el límite de intentos**, en `limitar_intentos` de `app/dependencias.py`, justo antes del `raise`: `logger.warning("Demasiados intentos desde %s", ip)`.

Un registro, un login fallido, uno correcto y varios intentos seguidos se ven así:

```text
2026-10-25 02:00:30,538 INFO app.main: Arrancando Tareas (desarrollo)
2026-10-25 02:00:31,458 INFO app.routers.auth: Usuario registrado: id 2
2026-10-25 02:00:31,512 WARNING app.routers.auth: Login fallido para juan@ejemplo.com
INFO:     127.0.0.1:64596 - "POST /auth/login HTTP/1.1" 401 Unauthorized
2026-10-25 02:00:31,549 INFO app.routers.auth: Login correcto: id 2
...
2026-10-25 02:00:31,665 WARNING app.dependencias: Demasiados intentos desde 127.0.0.1
INFO:     127.0.0.1:64601 - "POST /auth/login HTTP/1.1" 429 Too Many Requests
```

Fijate que los mensajes usan `%s` y los datos van aparte, en vez de una f-string: `logging` arma el texto solo si el mensaje se va a mostrar. Los errores inesperados (un `500`) no hace falta registrarlos a mano: Uvicorn ya los registra con todo su traceback.

## Qué no registrar

Los logs se guardan, se copian y los lee más gente que la base de datos. Lo que aparece en un log hay que tratarlo como si fuera público. Nunca registres:

- **Contraseñas**, ni siquiera las equivocadas: suelen ser la correcta con un error de tipeo.
- **Tokens, cookies y el encabezado `Authorization`**. Con un token copiado de un log, cualquiera entra como ese usuario. Por eso los tokens van en encabezados y no en la URL: el log de Uvicorn registra cada ruta, con su query string.
- **Hashes de contraseñas** y **claves de la configuración**. Ojo con registrar objetos enteros. `logger.info("Configuración: %s", configuracion)` parece inocente, pero filtra la clave de los JWT y la contraseña de la base:

    ```text
    2026-10-25 02:03:22,248 INFO app.main: Configuración: nombre_app='Tareas (desarrollo)' debug=True clave_api='una-clave-larga-de-desarrollo' ... url_base_datos='postgresql+asyncpg://tareas:tareas@localhost:5432/tareas' clave_jwt='5f0c3b1e9a7d...' ...
    ```

- **Datos personales de más.** El email del login fallido sirve para detectar ataques contra una cuenta, pero es un dato personal: no agregues otros (nombre, dirección, el cuerpo entero del pedido), y que los logs se guarden por un tiempo limitado.

El SQL de SQLAlchemy es el caso más traicionero. Con `DEBUG=true`, cada consulta se registra **con sus parámetros**, y al registrar un usuario aparece el hash:

```text
2026-10-25 02:03:06,181 INFO sqlalchemy.engine.Engine: [generated in 0.00012s] ('Eva', 'eva@ejemplo.com', '$argon2id$v=19$m=65536,t=3,p=4$9bpxvZmO...', 'usuario', True)
```

Para desarrollar está bien; en producción, `DEBUG` tiene que estar apagado (es el tema de la página siguiente). Y es la razón del `if` de `basicConfig`: si dejabas el `echo=True` del motor, SQLAlchemy agregaba su propio handler además del de `basicConfig`, y cada línea de SQL salía **dos veces**.

!!! tip "Ocultar los parámetros del SQL"

    Si necesitás ver el SQL sin los datos, `create_async_engine(url, hide_parameters=True)` reemplaza los parámetros por `[SQL parameters hidden due to hide_parameters=True]`, en los logs y en los mensajes de error.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Node, `console.log` es el `print`, y para logs de verdad se usan librerías como **pino** o **winston**, con los mismos niveles (`logger.info`, `logger.warn`, `logger.error`). pino tiene una opción `redact` para tapar campos como `req.headers.authorization`, la misma preocupación de "qué no registrar".

??? example "Ver comparación en Django ⬇️"

    Django usa el mismo módulo `logging` de Python, configurado con el diccionario `LOGGING` de `settings.py` en vez de `basicConfig`. `logger = logging.getLogger(__name__)` se escribe igual. Y tiene `@sensitive_variables` y `@sensitive_post_parameters` para que las contraseñas no aparezcan en los reportes de errores.

??? example "Ver comparación en Spring Boot ⬇️"

    Spring Boot usa SLF4J con Logback: `private static final Logger log = LoggerFactory.getLogger(MiClase.class);` y `log.warn("Login fallido para {}", email);`, donde `{}` cumple el papel de `%s`. El nivel se configura en `application.properties` con `logging.level.root=INFO`, como `NIVEL_LOG`.

## Mini ejercicio

pytest tiene una fixture, `caplog`, que captura los logs durante un test (`caplog.text` es todo lo registrado). Escribí `tests/test_logs.py` con un test que haga un login fallido y compruebe que quedó registrado `Login fallido para ...` y que la contraseña **no** aparece en los logs.

??? success "Ver solución"

    ```python title="tests/test_logs.py"
    import logging


    def test_login_fallido_queda_registrado(cliente, caplog):
        datos = {"username": "nadie@ejemplo.com", "password": "clave-secreta-123"}
        with caplog.at_level(logging.WARNING):
            respuesta = cliente.post("/auth/login", data=datos)
        assert respuesta.status_code == 401
        assert "Login fallido para nadie@ejemplo.com" in caplog.text
        assert "clave-secreta-123" not in caplog.text
    ```

    El test pide dos fixtures: `cliente`, la tuya de `conftest.py`, y `caplog`, que viene con pytest. `caplog.at_level` captura desde `WARNING` mientras dura el `with`. El último `assert` es el importante: si mañana alguien agrega la contraseña al mensaje, este test lo frena. `uv run pytest` tiene que mostrar `14 passed`.

---

**Siguiente:** [Preparar la app para producción →](05-preparar-para-produccion.md)
