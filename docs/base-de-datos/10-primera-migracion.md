# Primera migración

!!! info "En esta página"

    **Qué vas a aprender:** cómo generar una migración automáticamente con Alembic, revisarla, aplicarla y deshacerla, y cuál es el flujo de trabajo de cada vez que cambiás un modelo.

    **Qué necesitás:** [Migraciones con Alembic: qué son y por qué](09-migraciones-que-son.md), con Alembic ya configurado.

## Paso 1: sacar `create_all`

Desde ahora, las tablas las crea y modifica **Alembic**. Si `create_all` siguiera en el `lifespan`, crearía tablas por su cuenta y se pisaría con las migraciones. En `app/main.py`, el `lifespan` queda solo con el apagado:

```python title="app/main.py"
@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    # Apagado: cierra las conexiones con la base
    await motor.dispose()
```

Y de los imports sacá `from app import modelos` y `Base`, que ya no se usan:

```python
from app.config import obtener_configuracion
from app.db import SesionDep, motor
```

## Paso 2: empezar con la base vacía

Las tablas que tenés las creó `create_all`, y Alembic no sabe nada de ellas. Lo más simple es que la primera migración las cree desde cero, así que borralas (son datos de prueba):

```bash
docker exec -it tareas-db psql -U tareas -c "DROP TABLE tareas, usuarios;"
```

## Paso 3: generar la migración

```bash
uv run alembic revision --autogenerate -m "crear tablas usuarios y tareas"
```

Alembic compara tus modelos con la base, ve que faltan dos tablas y escribe la migración:

```text
INFO  [alembic.autogenerate.compare.tables] Detected added table 'usuarios'
INFO  [alembic.autogenerate.compare.tables] Detected added table 'tareas'
Generating .../migraciones/versions/d96edc695e3a_crear_tablas_usuarios_y_tareas.py ...  done
```

El texto de `-m` es la descripción del cambio, como el mensaje de un commit. El número del principio (`d96edc695e3a`) es el identificador de la migración: el tuyo va a ser otro.

## Paso 4: revisarla

**Siempre** abrí la migración generada antes de aplicarla. La autogeneración es muy buena, pero no adivina todo. Esta es la parte importante del archivo:

```python title="migraciones/versions/d96edc695e3a_crear_tablas_usuarios_y_tareas.py"
revision: str = 'd96edc695e3a'
down_revision: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    """Upgrade schema."""
    op.create_table('usuarios',
    sa.Column('id', sa.Integer(), nullable=False),
    sa.Column('nombre', sa.String(length=100), nullable=False),
    sa.Column('email', sa.String(length=255), nullable=False),
    sa.Column('activo', sa.Boolean(), nullable=False),
    sa.PrimaryKeyConstraint('id'),
    sa.UniqueConstraint('email')
    )
    op.create_table('tareas',
    sa.Column('id', sa.Integer(), nullable=False),
    sa.Column('titulo', sa.String(length=100), nullable=False),
    sa.Column('descripcion', sa.Text(), nullable=True),
    sa.Column('hecha', sa.Boolean(), nullable=False),
    sa.Column('creada_en', sa.DateTime(timezone=True), server_default=sa.text('now()'), nullable=False),
    sa.Column('usuario_id', sa.Integer(), nullable=False),
    sa.ForeignKeyConstraint(['usuario_id'], ['usuarios.id'], ),
    sa.PrimaryKeyConstraint('id')
    )


def downgrade() -> None:
    """Downgrade schema."""
    op.drop_table('tareas')
    op.drop_table('usuarios')
```

- **`revision`** es el id de esta migración, y **`down_revision`** el de la anterior. Como es la primera, vale `None`. Así se arma la cadena.
- **`upgrade()`** crea las dos tablas, con todas sus columnas y restricciones. Fijate que `usuarios` va antes que `tareas`: la clave foránea necesita que la tabla de usuarios ya exista.
- **`downgrade()`** las borra, en el orden inverso.

## Paso 5: aplicarla

```bash
uv run alembic upgrade head
```

```text
INFO  [alembic.runtime.migration] Running upgrade  -> d96edc695e3a, crear tablas usuarios y tareas
```

`head` significa "hasta la última migración". Comprobá en qué versión quedó la base:

```bash
uv run alembic current
```

```text
d96edc695e3a (head)
```

Y con `psql`, que además de tus tablas apareció `alembic_version`, donde Alembic anota la versión actual:

```text
 Schema |      Name       | Type  | Owner
--------+-----------------+-------+--------
 public | alembic_version | table | tareas
 public | tareas          | table | tareas
 public | usuarios        | table | tareas
```

Levantá la API: funciona igual que antes, pero ahora las tablas tienen historia.

## Deshacer

```bash
uv run alembic downgrade -1
```

Ejecuta el `downgrade()` de la última migración aplicada (`-1` es "una para atrás"). En este caso borra las dos tablas; `uv run alembic upgrade head` las vuelve a crear. Para ver la cadena completa: `uv run alembic history`.

## El flujo de cada cambio

De acá en adelante, cada vez que cambies algo en `app/modelos.py`:

1. Modificás el modelo.
2. `uv run alembic revision --autogenerate -m "qué cambió"`.
3. Revisás el archivo generado.
4. `uv run alembic upgrade head`.
5. Subís a Git el modelo **y** la migración juntos.

Quien baje tus cambios solo tiene que ejecutar `uv run alembic upgrade head`.

!!! warning "Columnas obligatorias en tablas con datos"

    Si agregás una columna `NOT NULL` (por ejemplo, `prioridad: Mapped[int] = mapped_column(default=1)`) a una tabla que ya tiene filas, la migración falla: `column "prioridad" of relation "tareas" contains null values`. Las filas existentes no tienen valor para la columna nueva, y el `default` de Python no se aplica a ellas. La solución es darle un valor por defecto **en la base**: `mapped_column(default=1, server_default="1")`. Alembic lo incluye en la migración y las filas viejas quedan con `1`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | Prisma | Alembic |
    | --- | --- |
    | `prisma migrate dev --name crear_tablas` | `alembic revision --autogenerate -m "..."` + `alembic upgrade head` |
    | `prisma migrate deploy` | `alembic upgrade head` |
    | `prisma migrate status` | `alembic current` / `alembic history` |

    Prisma genera las migraciones en SQL; Alembic, en Python con `op.create_table`, `op.add_column`, etc. Prisma no tiene un `downgrade` automático.

??? example "Ver comparación en Django ⬇️"

    | Django | Alembic |
    | --- | --- |
    | `python manage.py makemigrations` | `alembic revision --autogenerate -m "..."` |
    | `python manage.py migrate` | `alembic upgrade head` |
    | `python manage.py migrate tareas 0001` | `alembic downgrade <revisión>` |
    | `python manage.py showmigrations` | `alembic history` |

    El flujo es el mismo. Django arma el nombre de la migración solo; en Alembic lo das con `-m`.

??? example "Ver comparación en Spring Boot ⬇️"

    Con Flyway escribís a mano `V1__crear_tablas.sql` y Flyway lo aplica al arrancar la app. Con Alembic la migración se genera desde los modelos y se aplica con un comando aparte. La tabla `alembic_version` cumple el rol de `flyway_schema_history`.

## Mini ejercicio

Agregá al modelo `Tarea` una columna opcional `vence_en` (fecha y hora con zona horaria, que puede quedar vacía). Generá la migración, revisala y aplicala. Comprobá con `psql` que las tareas que ya existían siguen ahí.

??? success "Ver solución"

    En `app/modelos.py`, dentro de `Tarea`:

    ```python
    vence_en: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    ```

    ```bash
    uv run alembic revision --autogenerate -m "agregar vence_en a tareas"
    ```

    La migración generada solo agrega y saca la columna:

    ```python
    def upgrade() -> None:
        op.add_column('tareas', sa.Column('vence_en', sa.DateTime(timezone=True), nullable=True))


    def downgrade() -> None:
        op.drop_column('tareas', 'vence_en')
    ```

    Con `uv run alembic upgrade head` se aplica, y `SELECT id, titulo, vence_en FROM tareas;` muestra las tareas de antes con `vence_en` vacío: no se perdió nada. Como es `| None`, no hace falta `server_default`. Para que la API la use, agregala también a los esquemas de Pydantic.

---

¡Terminaste el Capítulo 3! Tu API ya guarda los datos en PostgreSQL y tiene migraciones. En el próximo capítulo vas a entender a fondo Docker y a levantar todo el proyecto con un solo comando.

**Siguiente:** [Capítulo 4 · Qué es un contenedor y qué es una imagen →](../docker/01-contenedores-e-imagenes.md)
