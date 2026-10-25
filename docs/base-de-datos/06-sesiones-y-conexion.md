# Sesiones y conexión a la base

!!! info "En esta página"

    **Qué vas a aprender:** cómo conectar tu API a PostgreSQL con SQLAlchemy: el **motor**, las **sesiones**, una dependencia que entrega una sesión por pedido, y cómo crear las tablas al arrancar.

    **Qué necesitás:** [Definir un modelo](05-definir-un-modelo.md), [`lifespan`](../fastapi/14-lifespan.md) y el contenedor `tareas-db` de [Qué es PostgreSQL](03-que-es-postgresql.md) corriendo (`docker start tareas-db`).

## Motor y sesión

Para hablar con la base, SQLAlchemy usa dos piezas:

- **El motor** (*engine*): sabe **dónde** está la base y mantiene un grupo de conexiones abiertas listas para usar (un *pool*). Se crea **uno solo** para toda la app.
- **La sesión**: es la "conversación" con la base durante **un** pedido. A través de ella hacés consultas, agregás objetos y confirmás los cambios. Se crea una **por pedido** y se cierra al terminar.

Pensalo como un restaurante: el motor es la cocina, que está siempre; cada sesión es un pedido de una mesa, que empieza y termina.

## La URL en la configuración

Agregá el campo a `Configuracion` en `app/config.py`:

```python
url_base_datos: str
```

Y la URL al `.env`, con el driver `asyncpg`:

```bash title=".env"
URL_BASE_DATOS=postgresql+asyncpg://tareas:tareas@localhost:5432/tareas
```

(Si en la página de PostgreSQL usaste otro puerto, como `5433`, ponelo acá.)

## El motor, la sesión y la dependencia

Actualizá `app/db.py`:

```python title="app/db.py"
from collections.abc import AsyncIterator
from typing import Annotated

from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

from app.config import obtener_configuracion

configuracion = obtener_configuracion()

motor = create_async_engine(configuracion.url_base_datos, echo=configuracion.debug)
CrearSesion = async_sessionmaker(motor, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


async def obtener_sesion() -> AsyncIterator[AsyncSession]:
    async with CrearSesion() as sesion:
        yield sesion


SesionDep = Annotated[AsyncSession, Depends(obtener_sesion)]
```

Parte por parte:

- **`create_async_engine`** crea el motor a partir de la URL. Con `echo=True` muestra en la terminal **cada consulta SQL** que se ejecuta; lo atamos a `debug` para verlo solo en desarrollo.
- **`async_sessionmaker`** es una "fábrica" de sesiones: cada vez que llamás a `CrearSesion()` te da una nueva. `expire_on_commit=False` evita que los objetos se "vacíen" después de confirmar los cambios, algo que en una API asincrónica traería errores.
- **`obtener_sesion`** es una **dependencia con `yield`**. Funciona como `lifespan`, pero por pedido: lo de antes del `yield` corre antes de la ruta, el `yield` le entrega la sesión, y al terminar la ruta se sigue ejecutando. El `async with` se encarga de **cerrar la sesión** pase lo que pase, incluso si la ruta lanzó un error.
- **`SesionDep`** es un atajo: en vez de escribir `Annotated[AsyncSession, Depends(obtener_sesion)]` en cada ruta, escribís `sesion: SesionDep`.

## Crear las tablas y cerrar al apagar

En `app/main.py`, el `lifespan` ya no lee ni escribe `tareas.json`: ahora prepara la base. Reemplazá los imports, la constante `ARCHIVO_DATOS` y la función `lifespan` por esto:

```python title="app/main.py (principio del archivo)"
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy import text

from app import modelos  # noqa: F401  (registra las tablas en Base)
from app.config import obtener_configuracion
from app.db import Base, SesionDep, motor
from app.routers import tareas, usuarios

configuracion = obtener_configuracion()


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Arranque: crea las tablas que falten (provisorio, hasta usar Alembic)
    async with motor.begin() as conexion:
        await conexion.run_sync(Base.metadata.create_all)

    yield

    # Apagado: cierra las conexiones con la base
    await motor.dispose()
```

La creación de `app`, el middleware de CORS y los `include_router` quedan igual. Al final del archivo, agregá una ruta para comprobar la conexión:

```python title="app/main.py (final del archivo)"
@app.get("/salud", tags=["salud"])
async def salud(sesion: SesionDep):
    """Comprueba que la API puede hablar con la base de datos."""
    resultado = await sesion.execute(text("SELECT 1"))
    return {"base_de_datos": "ok" if resultado.scalar() == 1 else "error"}
```

- **`Base.metadata.create_all`** crea en la base todas las tablas registradas en `Base` que **todavía no existan**. Las que ya existen, no las toca.
- **`from app import modelos`** parece no usarse, pero es necesario: al importar el módulo, las clases `Tarea` y `Usuario` se registran en `Base`. Sin ese import, `create_all` no sabría que existen. El comentario `# noqa: F401` le avisa al linter (lo vas a ver en el Capítulo 6) que el import es a propósito.
- **`motor.dispose()`** cierra las conexiones del pool al apagar.
- **`/salud`** es una ruta para comprobar la conexión: pide una sesión y ejecuta `SELECT 1`, la consulta más simple posible. `text(...)` permite mandar SQL escrito a mano.

!!! warning "`create_all` es provisorio"

    `create_all` crea tablas nuevas, pero **no modifica** las que ya existen: si mañana le agregás una columna a `Tarea`, no la va a agregar. Para eso existen las migraciones con Alembic, el tema de las últimas páginas del capítulo. Por ahora alcanza.

## Probarlo

Con `tareas-db` corriendo, levantá la API. Como `debug` está activado, en la terminal vas a ver el SQL que SQLAlchemy ejecuta, entre otras cosas los `CREATE TABLE`:

```text
INFO sqlalchemy.engine.Engine BEGIN (implicit)
INFO sqlalchemy.engine.Engine
CREATE TABLE tareas (
...
INFO sqlalchemy.engine.Engine COMMIT
```

Abrí `http://127.0.0.1:8000/salud`:

```json
{"base_de_datos":"ok"}
```

Y comprobá con `psql` que las tablas existen:

```bash
docker exec -it tareas-db psql -U tareas -c "\dt"
```

```text
 Schema |   Name   | Type  | Owner
--------+----------+-------+--------
 public | tareas   | table | tareas
 public | usuarios | table | tareas
```

Al pedir `/salud`, en el log vas a ver `BEGIN`, `SELECT 1` y `ROLLBACK`. El `ROLLBACK` es normal: la sesión solo leyó, no confirmó cambios, y al cerrarse descarta todo.

Las rutas de `/tareas` todavía usan la lista en memoria. En la próxima página las vas a pasar a la base.

!!! note "Si la base está apagada"

    Si `tareas-db` no está corriendo, la API no arranca: el `lifespan` intenta conectarse y falla con `ConnectionRefusedError: [Errno 61] Connection refused` (el número cambia según el sistema). Prendé el contenedor con `docker start tareas-db` y guardá algún archivo para que el servidor reintente.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Con `pg`, el motor equivale a un `Pool` y la sesión a un cliente que pedís con `pool.connect()` en cada pedido y devolvés con `cliente.release()` en un `finally`. La dependencia `obtener_sesion` hace ese `connect` y ese `release` por vos. Con Prisma, el `PrismaClient` cumple el rol del motor.

??? example "Ver comparación en Django ⬇️"

    Django maneja la conexión por vos: se configura en `DATABASES` y cada pedido usa una conexión automáticamente, sin sesiones explícitas. Con SQLAlchemy el manejo es explícito: la sesión llega a cada ruta por `Depends`. Y `create_all` hace algo parecido a un `migrate` muy limitado: crea tablas, pero no las modifica.

??? example "Ver comparación en Spring Boot ⬇️"

    | Spring Boot (JPA) | SQLAlchemy |
    | --- | --- |
    | `DataSource` + pool HikariCP | El motor (`create_async_engine`) |
    | `EntityManager` | La sesión (`AsyncSession`) |
    | Inyección de `EntityManager` | `sesion: SesionDep` |
    | `spring.jpa.show-sql=true` | `echo=True` |
    | `spring.jpa.hibernate.ddl-auto=update` | `create_all` (solo crea, no modifica) |

## Mini ejercicio

Cambiá la ruta `/salud` para que, además de `"ok"`, devuelva la versión de PostgreSQL. La consulta SQL `SELECT version()` devuelve un texto con la versión.

??? success "Ver solución"

    ```python
    @app.get("/salud", tags=["salud"])
    async def salud(sesion: SesionDep):
        """Comprueba que la API puede hablar con la base de datos."""
        resultado = await sesion.execute(text("SELECT version()"))
        return {"base_de_datos": "ok", "version": resultado.scalar()}
    ```

    ```json
    {"base_de_datos":"ok","version":"PostgreSQL 17.11 (Debian 17.11-1.pgdg13+2) on aarch64-unknown-linux-gnu, ..."}
    ```

    `resultado.scalar()` devuelve el primer valor de la primera fila, que es justo lo que trae una consulta de un solo dato.

---

**Siguiente:** [Crear, leer, actualizar y borrar (CRUD) con SQLAlchemy →](07-crud.md)
