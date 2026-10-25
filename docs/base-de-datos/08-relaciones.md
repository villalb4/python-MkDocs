# Relaciones entre tablas (uno a muchos)

!!! info "En esta página"

    **Qué vas a aprender:** cómo conectar dos tablas con una clave foránea, cómo declarar la relación en SQLAlchemy, y cómo consultar las tareas de un usuario.

    **Qué necesitás:** [Crear, leer, actualizar y borrar (CRUD) con SQLAlchemy](07-crud.md), con el router de usuarios ya pasado a la base (el ejercicio de esa página).

## Uno a muchos

Hasta ahora, tareas y usuarios eran tablas sueltas. Pero en una app real, cada tarea **pertenece** a un usuario, y un usuario puede tener **muchas** tareas. A eso se lo llama una relación **uno a muchos**.

En la primera página del capítulo viste cómo se representa: la tabla del lado "muchos" (`tareas`) guarda el `id` del lado "uno" (`usuarios`) en una columna, `usuario_id`. Esa columna es una **clave foránea** (*foreign key*): la base garantiza que siempre apunte a un usuario que existe.

## Declarar la relación

Así queda `app/modelos.py`. Fijate que `Usuario` ahora va primero:

```python title="app/modelos.py"
from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, String, Text, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.db import Base


class Usuario(Base):
    __tablename__ = "usuarios"

    id: Mapped[int] = mapped_column(primary_key=True)
    nombre: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True)
    activo: Mapped[bool] = mapped_column(default=True)

    tareas: Mapped[list["Tarea"]] = relationship(back_populates="usuario")


class Tarea(Base):
    __tablename__ = "tareas"

    id: Mapped[int] = mapped_column(primary_key=True)
    titulo: Mapped[str] = mapped_column(String(100))
    descripcion: Mapped[str | None] = mapped_column(Text)
    hecha: Mapped[bool] = mapped_column(default=False)
    creada_en: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    usuario_id: Mapped[int] = mapped_column(ForeignKey("usuarios.id"))

    usuario: Mapped[Usuario] = relationship(back_populates="tareas")
```

Hay dos cosas distintas:

- **`usuario_id`** es una **columna** de verdad. `ForeignKey("usuarios.id")` le dice a la base que tiene que coincidir con un `id` de la tabla `usuarios`. En el SQL se ve como `FOREIGN KEY(usuario_id) REFERENCES usuarios (id)`.
- **`tareas`** y **`usuario`** **no** son columnas: son atajos de Python que da `relationship`. `una_tarea.usuario` es el objeto `Usuario` dueño, y `un_usuario.tareas` es la lista de sus tareas. `back_populates` conecta los dos lados entre sí.

`"Tarea"` va entre comillas en `Usuario` porque esa clase todavía no está definida en ese punto del archivo; SQLAlchemy la resuelve después por el nombre.

## Las tablas viejas no se actualizan solas

Si levantás la API así, falla:

```text
asyncpg.exceptions.UndefinedColumnError: column tareas.usuario_id does not exist
```

Es lo que advertía la página de sesiones: `create_all` crea las tablas que faltan, pero **no modifica** las que ya existen. La tabla `tareas` de tu base no tiene la columna nueva.

Como por ahora son datos de prueba, la salida más simple es borrar las tablas y dejar que se vuelvan a crear al arrancar:

```bash
docker exec -it tareas-db psql -U tareas -c "DROP TABLE tareas, usuarios;"
```

Esto **borra todos los datos**. En una base con datos reales nunca harías esto: para eso existen las **migraciones**, el tema de la próxima página.

## Esquemas y rutas

Una tarea nueva ahora necesita saber a qué usuario pertenece. En `app/esquemas.py`, agregá `usuario_id: int` al final de `TareaCrear` y de `TareaRespuesta`.

En `app/routers/tareas.py`, antes de guardar hay que comprobar que el usuario existe. Si no, la clave foránea rechazaría el `INSERT` con un `IntegrityError` y la API respondería `500`. Agregá esta función después de `TareaDep` (y `Usuario` al import de `app.modelos`):

```python
async def verificar_usuario(usuario_id: int, sesion: SesionDep) -> None:
    if await sesion.get(Usuario, usuario_id) is None:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Usuario no encontrado")
```

Y usala en `crear_tarea` y en `modificar_tarea`:

```python
    await verificar_usuario(datos.usuario_id, sesion)
    tarea = Tarea(
        titulo=datos.titulo,
        descripcion=datos.descripcion,
        hecha=datos.hecha,
        usuario_id=datos.usuario_id,
    )
```

(En `modificar_tarea`, además de llamarla, agregá `tarea.usuario_id = datos.usuario_id`.)

Por último, una ruta para ver las tareas de un usuario, al final de `app/routers/usuarios.py` (sumá `TareaRespuesta` y `Tarea` a los imports):

```python
@router.get("/{usuario_id}/tareas", response_model=list[TareaRespuesta])
async def listar_tareas_de_usuario(usuario_id: int, sesion: SesionDep):
    """Devuelve las tareas de un usuario."""
    if await sesion.get(Usuario, usuario_id) is None:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Usuario no encontrado")
    consulta = select(Tarea).where(Tarea.usuario_id == usuario_id).order_by(Tarea.id)
    resultado = await sesion.scalars(consulta)
    return resultado.all()
```

La URL `/usuarios/{usuario_id}/tareas` se lee "las tareas del usuario tal": así se suelen expresar las relaciones en una API REST.

## Probarlo

Creá dos usuarios y algunas tareas para cada uno. Después:

| Petición | Respuesta |
| --- | --- |
| `POST /tareas` con `{"titulo": "x"}` | `422`: falta `usuario_id` |
| `POST /tareas` con `"usuario_id": 99` | `404`: `Usuario no encontrado` |
| `GET /usuarios/1/tareas` | Solo las tareas de ese usuario |

La clave foránea también protege en el otro sentido: si intentás borrar desde `psql` un usuario que tiene tareas, la base se niega con `violates foreign key constraint "tareas_usuario_id_fkey"`, para no dejar tareas "huérfanas".

!!! warning "Las relaciones no se cargan solas en modo async"

    Tentador: buscar el usuario y devolver `usuario.tareas`. Pero en modo asincrónico, leer una relación que no se cargó de antemano falla con `sqlalchemy.exc.MissingGreenlet: greenlet_spawn has not been called`. SQLAlchemy necesitaría ir a la base en un momento en que no puede hacerlo con `await`. Por eso la ruta de arriba usa una consulta con `where`. En el ejercicio vas a ver cómo cargar la relación a propósito.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Prisma, la misma relación se declara en los dos modelos: `tareas Tarea[]` en `Usuario`, y `usuario Usuario @relation(fields: [usuarioId], references: [id])` + `usuarioId Int` en `Tarea`. Para traer un usuario con sus tareas se usa `include: { tareas: true }`, el equivalente de `selectinload`.

??? example "Ver comparación en Django ⬇️"

    En Django se declara un solo lado: `usuario = models.ForeignKey(Usuario, on_delete=models.PROTECT, related_name="tareas")`, y Django crea `usuario.tareas` solo. En SQLAlchemy se declaran la columna y los dos `relationship`. Y `selectinload` cumple el rol de `prefetch_related`.

??? example "Ver comparación en Spring Boot ⬇️"

    Equivale a `@ManyToOne @JoinColumn(name = "usuario_id")` en `Tarea` y `@OneToMany(mappedBy = "usuario")` en `Usuario`. El `MissingGreenlet` es primo de la famosa `LazyInitializationException` de Hibernate: en los dos casos se intenta cargar una relación cuando ya no se puede, y la solución es cargarla antes (en JPA, con `JOIN FETCH` o un *entity graph*).

## Mini ejercicio

Agregá una ruta `GET /usuarios/{usuario_id}` que devuelva el usuario **con la lista de sus tareas adentro**. Para eso, creá un esquema `UsuarioConTareas` que herede de `UsuarioRespuesta` y agregue `tareas: list[TareaRespuesta]`, y cargá la relación con `selectinload` (de `sqlalchemy.orm`).

??? success "Ver solución"

    ```python title="app/esquemas.py (agregar)"
    class UsuarioConTareas(UsuarioRespuesta):
        tareas: list[TareaRespuesta]
    ```

    ```python title="app/routers/usuarios.py (agregar)"
    @router.get("/{usuario_id}", response_model=UsuarioConTareas)
    async def obtener_usuario(usuario_id: int, sesion: SesionDep):
        """Devuelve un usuario con todas sus tareas."""
        consulta = select(Usuario).where(Usuario.id == usuario_id).options(selectinload(Usuario.tareas))
        usuario = await sesion.scalar(consulta)
        if usuario is None:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Usuario no encontrado")
        return usuario
    ```

    `.options(selectinload(Usuario.tareas))` le pide a SQLAlchemy que traiga las tareas **en la misma operación**, con una segunda consulta automática. Así `usuario.tareas` ya está cargado cuando Pydantic lo lee, y la respuesta trae el usuario con `"tareas": [...]` adentro.

---

**Siguiente:** [Migraciones con Alembic: qué son y por qué →](09-migraciones-que-son.md)
