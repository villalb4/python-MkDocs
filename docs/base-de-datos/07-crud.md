# Crear, leer, actualizar y borrar (CRUD) con SQLAlchemy

!!! info "En esta página"

    **Qué vas a aprender:** cómo pasar las rutas de tareas de la lista en memoria a PostgreSQL, usando la sesión para leer, crear, modificar y borrar filas, y cómo devolver objetos de SQLAlchemy con un modelo de respuesta de Pydantic.

    **Qué necesitás:** [Sesiones y conexión a la base](06-sesiones-y-conexion.md) y [Inyección de dependencias con `Depends`](../fastapi/11-depends.md).

## CRUD

**CRUD** son las cuatro operaciones básicas sobre datos: ***C**reate* (crear), ***R**ead* (leer), ***U**pdate* (actualizar) y ***D**elete* (borrar). Son las mismas que ya hacen tus rutas `POST`, `GET`, `PUT` y `DELETE`; lo que cambia es que ahora van a trabajar sobre la tabla `tareas` en vez de sobre una lista.

## Un modelo para la respuesta

Las rutas van a devolver objetos `Tarea` de SQLAlchemy. FastAPI necesita saber cómo convertirlos a JSON, y para eso se le indica un modelo de Pydantic de **respuesta**. Agregalo en `app/esquemas.py`:

```python title="app/esquemas.py (agregar)"
from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class TareaRespuesta(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    titulo: str
    descripcion: str | None
    hecha: bool
    creada_en: datetime
```

`from_attributes=True` le permite a Pydantic leer los datos de los **atributos** de un objeto (`tarea.titulo`), y no solo de un diccionario. Además, el modelo de respuesta es un **filtro**: solo se devuelven los campos que declara. Cuando un modelo tenga datos privados (como la contraseña de un usuario), no van a salir en la respuesta.

## El router con la base de datos

Reemplazá `app/routers/tareas.py` completo:

```python title="app/routers/tareas.py"
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, Query, status
from sqlalchemy import select

from app.db import SesionDep
from app.esquemas import TareaCrear, TareaRespuesta
from app.modelos import Tarea

router = APIRouter(prefix="/tareas", tags=["tareas"])


def paginacion(saltear: int = 0, limite: Annotated[int, Query(le=100)] = 10) -> dict:
    return {"saltear": saltear, "limite": limite}


async def buscar_tarea(tarea_id: int, sesion: SesionDep) -> Tarea:
    tarea = await sesion.get(Tarea, tarea_id)
    if tarea is None:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Tarea no encontrada")
    return tarea


TareaDep = Annotated[Tarea, Depends(buscar_tarea)]


@router.get("", response_model=list[TareaRespuesta])
async def listar_tareas(sesion: SesionDep, pagina: Annotated[dict, Depends(paginacion)]):
    """Devuelve las tareas, de a una página por vez."""
    consulta = select(Tarea).order_by(Tarea.id).offset(pagina["saltear"]).limit(pagina["limite"])
    resultado = await sesion.scalars(consulta)
    return resultado.all()


@router.get("/{tarea_id}", response_model=TareaRespuesta, responses={404: {"description": "La tarea no existe"}})
async def obtener_tarea(tarea: TareaDep):
    """Devuelve una tarea por su id."""
    return tarea


@router.post("", response_model=TareaRespuesta, status_code=status.HTTP_201_CREATED)
async def crear_tarea(datos: TareaCrear, sesion: SesionDep):
    """Crea una tarea nueva."""
    tarea = Tarea(titulo=datos.titulo, descripcion=datos.descripcion, hecha=datos.hecha)
    sesion.add(tarea)
    await sesion.commit()
    await sesion.refresh(tarea)
    return tarea


@router.put("/{tarea_id}", response_model=TareaRespuesta, responses={404: {"description": "La tarea no existe"}})
async def modificar_tarea(tarea: TareaDep, datos: TareaCrear, sesion: SesionDep):
    """Reemplaza los datos de una tarea."""
    tarea.titulo = datos.titulo
    tarea.descripcion = datos.descripcion
    tarea.hecha = datos.hecha
    await sesion.commit()
    return tarea


@router.delete("/{tarea_id}", status_code=status.HTTP_204_NO_CONTENT, responses={404: {"description": "La tarea no existe"}})
async def borrar_tarea(tarea: TareaDep, sesion: SesionDep):
    """Borra una tarea."""
    await sesion.delete(tarea)
    await sesion.commit()
```

La lista `tareas` desapareció, y las rutas ahora son `async def`, porque hablan con la base usando `await`.

## Cada operación, paso a paso

**Leer varias (`listar_tareas`).** `select(Tarea)` arma la consulta, y se le encadenan `order_by` (orden), `offset` (cuántas saltear) y `limit` (cuántas traer). `sesion.scalars(...)` la ejecuta y `.all()` devuelve la lista de objetos `Tarea`.

**Leer una (`buscar_tarea`).** `sesion.get(Tarea, tarea_id)` busca por clave primaria y devuelve el objeto, o `None` si no existe. Es la misma dependencia de antes, ahora contra la base.

**Crear (`crear_tarea`).** Se arma el objeto con los datos validados por Pydantic, y después:

1. `sesion.add(tarea)`: lo anota en la sesión para insertarlo.
2. `await sesion.commit()`: confirma. Recién acá se ejecuta el `INSERT` y queda guardado.
3. `await sesion.refresh(tarea)`: trae de la base los datos que generó ella (el `id` y `creada_en`).

**Modificar (`modificar_tarea`).** Se cambian los atributos del objeto y se hace `commit`. SQLAlchemy detecta qué cambió y ejecuta el `UPDATE` correspondiente.

**Borrar (`borrar_tarea`).** `sesion.delete(tarea)` y `commit`.

Sin `commit`, nada se guarda: al cerrarse la sesión, los cambios se descartan (ese era el `ROLLBACK` de la página anterior).

!!! note "Una sola sesión por pedido"

    En `modificar_tarea`, la sesión llega dos veces: a través de `buscar_tarea` y como parámetro de la ruta. ¿Son dos sesiones distintas? No: FastAPI ejecuta cada dependencia **una sola vez por pedido** y reusa el resultado. Por eso el `commit` de la ruta guarda los cambios de la tarea que buscó la dependencia.

## Probarlo

Levantá la API (con `tareas-db` corriendo) y probá desde `/docs`. Al crear una tarea, la respuesta trae su `id` y `creada_en`:

```json
{"id":1,"titulo":"Comprar pan","descripcion":null,"hecha":false,"creada_en":"2026-10-25T03:36:31.481769Z"}
```

En la terminal, gracias a `echo`, vas a ver el SQL que generó SQLAlchemy: un `INSERT INTO tareas (...) VALUES ($1, $2, $3) RETURNING tareas.id, tareas.creada_en`, los valores por separado, y el `COMMIT`.

Lo más importante: **apagá y volvé a prender el servidor**. Las tareas siguen ahí. Ya no dependen de un archivo ni de la memoria: están en PostgreSQL, y podés verlas con `psql`:

```bash
docker exec -it tareas-db psql -U tareas -c "SELECT id, titulo, hecha FROM tareas;"
```

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | Prisma | SQLAlchemy |
    | --- | --- |
    | `prisma.tarea.findMany({ skip, take, orderBy })` | `sesion.scalars(select(Tarea).offset(...).limit(...).order_by(...))` |
    | `prisma.tarea.findUnique({ where: { id } })` | `sesion.get(Tarea, id)` |
    | `prisma.tarea.create({ data })` | `sesion.add(tarea)` + `commit()` |
    | `prisma.tarea.update({ where, data })` | Cambiar atributos + `commit()` |
    | `prisma.tarea.delete({ where })` | `sesion.delete(tarea)` + `commit()` |

    Prisma guarda cada operación al instante; SQLAlchemy junta los cambios en la sesión y los guarda todos juntos con `commit()`.

??? example "Ver comparación en Django ⬇️"

    | Django ORM | SQLAlchemy |
    | --- | --- |
    | `Tarea.objects.order_by("id")[saltear:saltear + limite]` | `select(Tarea).order_by(Tarea.id).offset(...).limit(...)` |
    | `get_object_or_404(Tarea, pk=id)` | `sesion.get(Tarea, id)` + `404` si es `None` |
    | `Tarea.objects.create(...)` | `sesion.add(...)` + `commit()` |
    | `tarea.save()` | `commit()` |
    | `tarea.delete()` | `sesion.delete(tarea)` + `commit()` |

    El modelo de respuesta con `from_attributes` cumple el rol de un `ModelSerializer` de DRF.

??? example "Ver comparación en Spring Boot ⬇️"

    | Spring Data JPA | SQLAlchemy |
    | --- | --- |
    | `repo.findById(id).orElseThrow(...)` | `sesion.get(Tarea, id)` + `404` |
    | `repo.save(tarea)` | `sesion.add(tarea)` + `commit()` |
    | `@Transactional` | Lo que va entre la sesión y el `commit()` |

## Mini ejercicio

Pasá el router de usuarios a la base: `GET /usuarios` tiene que listar los usuarios de la tabla y `POST /usuarios` crear uno. Agregá un `UsuarioRespuesta` (con `id`, `nombre`, `email` y `activo`). Bonus: si ya existe un usuario con ese email, respondé `409`.

??? success "Ver solución"

    En `app/esquemas.py`, `UsuarioRespuesta` es igual a `TareaRespuesta`, con `model_config = ConfigDict(from_attributes=True)` y los campos `id`, `nombre`, `email` y `activo`. En `app/routers/usuarios.py`, se agregan los imports de `select`, `SesionDep`, `UsuarioRespuesta` y `Usuario`, se borra la lista y las rutas quedan así:

    ```python
    @router.get("", response_model=list[UsuarioRespuesta])
    async def listar_usuarios(sesion: SesionDep):
        """Devuelve todos los usuarios."""
        resultado = await sesion.scalars(select(Usuario).order_by(Usuario.id))
        return resultado.all()


    @router.post("", response_model=UsuarioRespuesta, status_code=status.HTTP_201_CREATED)
    async def crear_usuario(datos: UsuarioCrear, sesion: SesionDep):
        """Crea un usuario nuevo."""
        existente = await sesion.scalar(select(Usuario).where(Usuario.email == datos.email))
        if existente is not None:
            raise HTTPException(status_code=status.HTTP_409_CONFLICT, detail="Ya existe un usuario con ese email")
        usuario = Usuario(nombre=datos.nombre, email=datos.email)
        sesion.add(usuario)
        await sesion.commit()
        await sesion.refresh(usuario)
        return usuario
    ```

    `sesion.scalar(...)` (en singular) devuelve el primer resultado o `None`. Sin ese chequeo, la restricción `UNIQUE` de la base rechaza el duplicado con un `IntegrityError`, y la API responde `500`. Con `debug` activado, ese `500` incluye todo el traceback en la respuesta: otra razón para apagarlo en producción.

---

**Siguiente:** [Relaciones entre tablas (uno a muchos) →](08-relaciones.md)
