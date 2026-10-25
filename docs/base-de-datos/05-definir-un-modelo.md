# Definir un modelo

!!! info "En esta página"

    **Qué vas a aprender:** cómo describir una tabla con una clase de SQLAlchemy (un **modelo**), cómo se traducen los type hints a columnas y restricciones, y cómo ver el SQL que genera.

    **Qué necesitás:** [Qué es un ORM y qué es SQLAlchemy](04-orm-sqlalchemy.md) y [Tablas, filas y columnas](02-tablas-filas-columnas.md).

## Antes: un cambio de nombre

Hasta ahora, tus modelos de Pydantic estaban en `app/modelos.py`. Como ahora van a convivir con los modelos de SQLAlchemy, conviene separarlos con nombres claros, siguiendo una convención muy común en proyectos de FastAPI:

- **`app/esquemas.py`**: los modelos de **Pydantic** (lo que entra y sale de la API).
- **`app/modelos.py`**: los modelos de **SQLAlchemy** (las tablas).

Renombrá `app/modelos.py` a `app/esquemas.py`, y en los dos routers cambiá el import:

```python
from app.esquemas import TareaCrear      # en app/routers/tareas.py
from app.esquemas import UsuarioCrear    # en app/routers/usuarios.py
```

## La clase base

Todos los modelos de SQLAlchemy heredan de una clase base común, que se crea una sola vez. Creá `app/db.py`:

```python title="app/db.py"
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
```

`Base` lleva el registro de todas las tablas de tu aplicación. Por ahora alcanza con eso; en la próxima página este archivo va a tener también la conexión a la base.

## El modelo `Tarea`

Creá el nuevo `app/modelos.py`:

```python title="app/modelos.py"
from datetime import datetime

from sqlalchemy import DateTime, String, Text, func
from sqlalchemy.orm import Mapped, mapped_column

from app.db import Base


class Tarea(Base):
    __tablename__ = "tareas"

    id: Mapped[int] = mapped_column(primary_key=True)
    titulo: Mapped[str] = mapped_column(String(100))
    descripcion: Mapped[str | None] = mapped_column(Text)
    hecha: Mapped[bool] = mapped_column(default=False)
    creada_en: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

Cada atributo es una columna. Se lee así:

- **`__tablename__`** es el nombre de la tabla en la base.
- **`Mapped[int]`** dice el tipo de Python de la columna. SQLAlchemy elige el tipo de SQL que corresponde: `int` → `INTEGER`, `str` → `VARCHAR`, `bool` → `BOOLEAN`.
- **`Mapped[str]`** (sin `None`) significa **`NOT NULL`**. **`Mapped[str | None]`** significa que la columna acepta `NULL`. Es la misma idea que en Pydantic: el type hint dice si el dato puede faltar.
- **`mapped_column(...)`** agrega detalles: un tipo de SQL más específico o restricciones.

Las opciones de `mapped_column` que usamos:

| Opción | Significa |
| --- | --- |
| `primary_key=True` | Es la clave primaria. Con un `int`, la base genera el valor sola. |
| `String(100)` | `VARCHAR(100)`: texto de hasta 100 caracteres. |
| `Text` | `TEXT`: texto sin límite. |
| `DateTime(timezone=True)` | `TIMESTAMP WITH TIME ZONE`, el `TIMESTAMPTZ` de la página de tablas. |
| `default=False` | Valor por defecto que pone **SQLAlchemy** al insertar, si no indicaste otro. |
| `server_default=func.now()` | Valor por defecto que pone la **base de datos**. `func.now()` es la función `now()` de SQL: la fecha y hora del momento. |
| `unique=True` | Restricción `UNIQUE`. |

## Ver el SQL que genera

Para comprobar qué tabla describe el modelo, creá en la raíz del proyecto este script:

```python title="ver_sql.py"
from sqlalchemy.dialects import postgresql
from sqlalchemy.schema import CreateTable

from app.modelos import Tarea

print(CreateTable(Tarea.__table__).compile(dialect=postgresql.dialect()))
```

Al ejecutarlo con `uv run ver_sql.py` muestra:

```sql
CREATE TABLE tareas (
	id SERIAL NOT NULL,
	titulo VARCHAR(100) NOT NULL,
	descripcion TEXT,
	hecha BOOLEAN NOT NULL,
	creada_en TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
	PRIMARY KEY (id)
)
```

Es casi lo mismo que escribiste a mano en la página de tablas. `SERIAL` es otra forma de pedirle a PostgreSQL que genere los ids. Fijate que `hecha` no tiene `DEFAULT` en el SQL: ese valor lo pone SQLAlchemy desde Python, no la base. Este script solo sirve para mirar; borralo cuando quieras.

## Crear un objeto no lo guarda

Un modelo se usa como cualquier clase: `Tarea(titulo="Comprar pan")` crea un objeto. Pero **todavía no está en la base**, y algunos atributos están vacíos:

```python
tarea = Tarea(titulo="Comprar pan")
print(tarea.titulo, tarea.hecha, tarea.id, tarea.creada_en)  # Comprar pan None None None
```

El `id` y `creada_en` los genera la base, y el `default` de `hecha` se aplica al insertar. Todos se completan recién cuando el objeto se guarda, algo que vas a ver en la página de CRUD.

!!! warning "Toda tabla necesita una clave primaria"

    Si definís un modelo sin ninguna columna con `primary_key=True`, SQLAlchemy da error apenas se carga el archivo: `could not assemble any primary key columns for mapped table`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Prisma, el mismo modelo se escribe en `schema.prisma`:

    ```prisma
    model Tarea {
      id          Int      @id @default(autoincrement())
      titulo      String   @db.VarChar(100)
      descripcion String?
      hecha       Boolean  @default(false)
      creadaEn    DateTime @default(now())
    }
    ```

    El `?` de Prisma equivale a `| None`. En TypeORM o Drizzle se define en código TypeScript, muy parecido a SQLAlchemy.

??? example "Ver comparación en Django ⬇️"

    ```python
    class Tarea(models.Model):
        titulo = models.CharField(max_length=100)
        descripcion = models.TextField(null=True)
        hecha = models.BooleanField(default=False)
        creada_en = models.DateTimeField(auto_now_add=True)
    ```

    En Django el tipo va en el campo (`CharField`); en SQLAlchemy 2 va en el type hint (`Mapped[str]`), y `null=True` se escribe `| None`. Django agrega el `id` solo; en SQLAlchemy lo declarás vos.

??? example "Ver comparación en Spring Boot ⬇️"

    ```java
    @Entity
    @Table(name = "tareas")
    public class Tarea {
        @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Integer id;

        @Column(length = 100, nullable = false)
        private String titulo;

        private boolean hecha = false;
        // ...
    }
    ```

    `__tablename__` es `@Table`, `primary_key=True` es `@Id` + `@GeneratedValue`, y `Mapped[str]` sin `None` equivale a `nullable = false`.

## Mini ejercicio

Agregá a `app/modelos.py` un modelo `Usuario` para la tabla `usuarios`, con: `id`, `nombre` (hasta 100 caracteres, obligatorio), `email` (hasta 255 caracteres, obligatorio y único) y `activo` (booleano, por defecto `True`). Miralo con `ver_sql.py`.

??? success "Ver solución"

    ```python
    class Usuario(Base):
        __tablename__ = "usuarios"

        id: Mapped[int] = mapped_column(primary_key=True)
        nombre: Mapped[str] = mapped_column(String(100))
        email: Mapped[str] = mapped_column(String(255), unique=True)
        activo: Mapped[bool] = mapped_column(default=True)
    ```

    Cambiando `Tarea` por `Usuario` en `ver_sql.py`, muestra:

    ```sql
    CREATE TABLE usuarios (
    	id SERIAL NOT NULL,
    	nombre VARCHAR(100) NOT NULL,
    	email VARCHAR(255) NOT NULL,
    	activo BOOLEAN NOT NULL,
    	PRIMARY KEY (id),
    	UNIQUE (email)
    )
    ```

    Lo vas a usar más adelante, en la página de relaciones y en el capítulo de autenticación.

---

**Siguiente:** [Sesiones y conexión a la base →](06-sesiones-y-conexion.md)
