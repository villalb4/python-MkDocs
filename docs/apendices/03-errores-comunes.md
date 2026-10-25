# Errores comunes por capítulo

!!! info "En esta página"

    **Qué vas a aprender:** a reconocer los errores que más aparecen al seguir la guía, por el mensaje que muestran, y a resolverlos rápido.

    **Qué necesitás:** nada. Buscá acá el mensaje que te apareció (con el buscador de arriba o con <kbd>Ctrl</kbd> + <kbd>F</kbd>).

Cada error tiene el **mensaje** tal como aparece, la **causa** y la **solución**, con un enlace a la página donde se explica. En un traceback largo, el mensaje que importa casi siempre es **la última línea**.

## 1 · Python básico

- **`NameError: name 'nombre' is not defined`**: usaste una variable que no existe, casi siempre por un error de tipeo o porque se definió dentro de una función. Revisá cómo se escribe y dónde se creó. ([Qué es Python](../python/01-que-es-python.md), [Funciones](../python/09-funciones.md))
- **`TypeError: can only concatenate str (not "int") to str`**: sumaste texto y número. Usá una f-string: `f"Tenés {edad} años"`. ([Variables y tipos](../python/03-variables-y-tipos.md))
- **`IndentationError: expected an indented block after 'if' statement`**: después de una línea que termina en `:` tiene que venir un bloque con sangría. ([Condicionales](../python/07-condicionales.md))
- **`KeyError: 'email'`** o **`IndexError: list index out of range`**: pediste una clave o una posición que no existe. Para diccionarios, `.get("email")` devuelve `None` en vez de fallar. ([Diccionarios](../python/06-diccionarios.md), [Listas](../python/05-listas.md))
- **`ModuleNotFoundError: No module named 'fastapi'`**: ejecutaste con el Python del sistema (`python main.py`), que no tiene las librerías del proyecto. Usá siempre `uv run`. ([Entornos virtuales](../python/21-entornos-virtuales.md))
- **`AttributeError: module 'random' has no attribute 'choice' (consider renaming 'random.py' ...)`**: un archivo tuyo se llama igual que un módulo de Python y lo tapa. Cambiale el nombre. ([Módulos e import](../python/11-modulos-e-import.md))
- **`ImportError: attempted relative import with no known parent package`**: ejecutaste como script un archivo que está dentro de un paquete. Ejecutá desde la carpeta del proyecto, con imports absolutos (`from app.config import ...`). ([Paquetes](../python/12-paquetes.md))

## 2 · FastAPI

- **`[Errno 48] error while attempting to bind on address ('127.0.0.1', 8000): address already in use`**: ya hay otro servidor en el puerto 8000, muchas veces uno que quedó abierto en otra terminal. Cerralo o usá `--port 8001`. (El número del error cambia según el sistema.) ([Uvicorn](../fastapi/03-uvicorn.md))
- **`{"detail":"Not Found"}` en una ruta que existe**: falta `app.include_router(...)` para su router, o la URL no coincide con el `prefix`. Mirá en `/docs` qué rutas tiene la app. ([APIRouter](../fastapi/10-apirouter.md))
- **`{"detail":"Method Not Allowed"}` (`405`)**: la URL existe, pero no con ese método (por ejemplo, `DELETE /tareas` sin id). ([Métodos HTTP](../fastapi/04-metodos-http.md))
- **`422` con `"type":"missing"` o `"int_parsing"`**: los datos no cumplen lo que pide el esquema o el parámetro. El campo `loc` de la respuesta dice cuál. ([Parámetros](../fastapi/05-parametros.md), [Códigos de estado](../fastapi/08-codigos-de-estado.md))
- **`ValidationError` ... `Field required [type=missing, ...]` al arrancar**: falta una variable obligatoria en el `.env` (o el `.env` no está en la carpeta del proyecto). ([Configuración](../fastapi/12-pydantic-settings.md))
- **`blocked by CORS policy: No 'Access-Control-Allow-Origin' header`**: el origen del frontend no está en `ORIGENES_PERMITIDOS`. Fijate que coincida exacto: `localhost` y `127.0.0.1` son orígenes distintos. ([CORS](../fastapi/13-cors.md))

## 3 · Base de datos

- **`ConnectionRefusedError: [Errno 61] Connection refused`**: la base no está andando, o el puerto de `URL_BASE_DATOS` no es el del contenedor. Levantala con `docker compose up -d db`. La API arranca igual: el error aparece en el primer pedido que usa la base, que responde `500`. ([Sesiones y conexión](../base-de-datos/06-sesiones-y-conexion.md))
- **`relation "usuarios" does not exist`**: la base está vacía, sin las tablas. Aplicá las migraciones con `uv run alembic upgrade head`. ([Primera migración](../base-de-datos/10-primera-migracion.md))
- **`UndefinedColumnError: column tareas.usuario_id does not exist`**: cambiaste un modelo pero no la base. Generá y aplicá una migración. ([Relaciones](../base-de-datos/08-relaciones.md))
- **`AttributeError: 'coroutine' object has no attribute 'all'`**, con el aviso `RuntimeWarning: coroutine 'AsyncSession.scalars' was never awaited`: te olvidaste el `await` delante de una operación de la sesión. Ruff no lo detecta. ([CRUD](../base-de-datos/07-crud.md))
- **`MissingGreenlet: greenlet_spawn has not been called`**: leíste una relación (`usuario.tareas`) que no estaba cargada. Cargala en la consulta con `selectinload`. ([Relaciones](../base-de-datos/08-relaciones.md))
- **Una migración generada con `pass` y nada más**: corriste `--autogenerate` sin cambiar ningún modelo. Borrá el archivo. ([Chuleta de comandos](02-chuleta-de-comandos.md#alembic-migraciones))

## 4 · Docker

- **`Bind for 0.0.0.0:5432 failed: port is already allocated`**: otro programa o contenedor ya usa ese puerto de tu máquina. Cambiá el de tu lado: `-p 5433:5432` (o `PUERTO_DB=5433`), y el puerto en `URL_BASE_DATOS`. ([PostgreSQL con docker run](../docker/03-postgres-docker-run.md))
- **`You must specify POSTGRES_PASSWORD to a non-empty value`**: el contenedor de PostgreSQL necesita una contraseña. ([PostgreSQL con docker run](../docker/03-postgres-docker-run.md))
- **`password authentication failed`** después de cambiar `POSTGRES_PASSWORD`: las variables `POSTGRES_*` solo se aplican cuando el volumen está vacío. Usá la contraseña vieja, o borrá el volumen (y los datos) con `docker compose down -v`. ([Volúmenes](../docker/05-volumenes.md))
- **`Connection refused` desde el contenedor de la API**: adentro de un contenedor, `localhost` es el propio contenedor. Con Compose, la base se llama `db`; con `docker run`, `host.docker.internal`. ([Dockerfile](../docker/07-dockerfile.md))
- **`The "POSTGRES_PASSWORD" variable is not set. Defaulting to a blank string.`**: el `docker-compose.yml` usa una variable que no está en el `.env`. ([Variables en Compose](../docker/06-variables-entorno-compose.md))
- **El nombre de la app aparece con comillas**, como `"Tareas (desarrollo)"`: `docker run --env-file` no saca las comillas de los valores del `.env` (Compose sí). Con `docker run`, escribí los valores sin comillas.
- **Los cambios del código no aparecen en el contenedor**: la imagen se construyó con el código viejo. Usá `docker compose up -d --build`. ([API y base juntas](../docker/08-api-y-base-juntas.md))

## 5 · Autenticación

Este capítulo tiene su propia página de errores, con los `401` que no se entienden, los tokens que vencen al instante y la cookie que no llega: [Errores comunes de autenticación](../autenticacion/13-errores-comunes.md).

## 6 · Calidad y despliegue

- **`ModuleNotFoundError: No module named 'app'` al correr pytest**: falta `pythonpath = ["."]` en `[tool.pytest]` de `pyproject.toml`. ([Primer test](../calidad-y-despliegue/01-pytest-primer-test.md))
- **`RuntimeError: ... attached to a different loop`** en los tests: falta un `motor.dispose()`, en la fixture o en el `lifespan`. ([Tests de endpoints](../calidad-y-despliegue/02-tests-de-endpoints.md))
- **`KeyError: 'access_token'`** en tests que hacen login: el límite de intentos respondió `429`. Vaciá `intentos_por_ip` en la fixture. ([Tests de endpoints](../calidad-y-despliegue/02-tests-de-endpoints.md))
- **`database "tareas_test" does not exist`**: falta crear la base de los tests con `createdb`. ([Tests de endpoints](../calidad-y-despliegue/02-tests-de-endpoints.md))
- **Los `logger.info` no aparecen**: nadie configuró `logging`. Falta el `basicConfig`. ([Logs](../calidad-y-despliegue/04-logs.md))
- **Cada línea de SQL sale dos veces**: el motor tiene `echo=True` y además está el `basicConfig`. Sacá el `echo`. ([Logs](../calidad-y-despliegue/04-logs.md))
- **`Value should have at least 32 items after validation`** en `clave_jwt`: la clave es demasiado corta. Generá una con `secrets.token_hex(32)`. ([Preparar para producción](../calidad-y-despliegue/05-preparar-para-produccion.md))
- **Detrás de un proxy, todos los logins dicen `Demasiados intentos desde 172.20.0.4`**: la API ve la IP del proxy. Falta `--forwarded-allow-ips`. ([Desplegar](../calidad-y-despliegue/06-desplegar.md))

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Muchos de estos errores tienen su versión en Node: `EADDRINUSE` es el puerto ocupado, `Cannot find module` es el `ModuleNotFoundError`, y olvidarse un `await` deja una `Promise` donde esperabas datos (`undefined` al leer una propiedad), el primo del `'coroutine' object has no attribute`.

??? example "Ver comparación en Django ⬇️"

    En Django, `relation "..." does not exist` también significa "falta migrar" (`python manage.py migrate`), y `runserver` avisa al arrancar que hay migraciones sin aplicar (`You have ... unapplied migration(s)`). Los errores de Docker y de configuración son los mismos.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring, el puerto ocupado es `Port 8080 was already in use`, la base caída es `Connection to localhost:5432 refused`, y una tabla que falta es `relation "..." does not exist`, el mismo mensaje porque lo genera PostgreSQL. `MissingGreenlet` es pariente de la `LazyInitializationException` de Hibernate.

## Mini ejercicio

Levantás la API con `uv run fastapi dev app/main.py`, abrís `/docs`, probás `POST /auth/registro` y recibís `500 Internal Server Error`. En la terminal, la última línea del error dice `relation "usuarios" does not exist`. ¿Qué pasó y cómo lo resolvés? ¿Por qué la API arrancó sin quejarse?

??? success "Ver solución"

    La base a la que se conecta la API no tiene las tablas: probablemente es una base nueva (un volumen recién creado, o borraste el volumen con `down -v`) y nunca se aplicaron las migraciones. Se resuelve con:

    ```bash
    uv run alembic upgrade head
    ```

    La API arrancó bien porque el `lifespan` no consulta las tablas: solo se conecta a la base cuando llega un pedido que la usa. Por eso `/salud` responde `ok` (hace `SELECT 1`, que no usa ninguna tabla) y el registro falla. Si querés que la app avise antes, el arranque podría comprobar con Alembic que la base esté al día.

---

**Siguiente:** [Cómo está hecha esta guía →](04-como-esta-hecha.md)
