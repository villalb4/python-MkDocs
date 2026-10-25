# Migraciones con Alembic: qué son y por qué

!!! info "En esta página"

    **Qué vas a aprender:** qué es una migración, por qué hacen falta para cambiar la estructura de una base sin perder datos, y cómo instalar y configurar Alembic en tu proyecto.

    **Qué necesitás:** [Relaciones entre tablas (uno a muchos)](08-relaciones.md).

## El problema

En la página anterior agregaste una columna (`usuario_id`) al modelo `Tarea`, y la API dejó de funcionar: la tabla de la base no tenía esa columna, y `create_all` no la agrega. La salida fue borrar las tablas, perdiendo todos los datos.

Con datos de prueba, da igual. Pero pensá en una API en producción, con usuarios reales:

- No podés borrar la base cada vez que agregás una columna.
- Si trabajás en equipo, cada persona tiene su propia base en su computadora, y todas tienen que recibir los mismos cambios.
- La base de producción tiene que recibir **exactamente** los mismos cambios que probaste en desarrollo, en el mismo orden.

Hace falta una forma de **versionar** la estructura de la base, como Git versiona el código.

## Qué es una migración

Una **migración** es un archivo de Python que describe **un cambio** en la estructura de la base, con dos funciones:

- **`upgrade()`**: aplica el cambio. Por ejemplo, "agregar la columna `usuario_id` a `tareas`".
- **`downgrade()`**: lo deshace. "Sacar la columna `usuario_id`".

Las migraciones forman una **cadena**: cada una sabe cuál es la anterior. La base guarda en una tabla especial (`alembic_version`) en qué migración está. Así, para ponerla al día, solo hay que aplicar las que le faltan:

```text
 Migración 1          Migración 2              Migración 3
 crear tablas    ──►  agregar usuario_id  ──►  agregar prioridad
                                ▲
                       tu base está acá: falta aplicar la 3
```

Como son archivos, las migraciones se suben a Git junto con el código. Quien baje los cambios ejecuta un comando y su base queda igual a la de todos.

## Alembic

**Alembic** es la herramienta de migraciones de SQLAlchemy, hecha por los mismos autores. Su función más útil es la **autogeneración**: compara tus modelos con la base real y escribe la migración por vos. Si agregaste una columna al modelo, Alembic lo detecta y genera el `upgrade` y el `downgrade` correspondientes.

## Instalarlo

En la raíz del proyecto:

```bash
uv add alembic
uv run alembic init -t async migraciones
```

- `alembic init` crea la configuración inicial. `migraciones` es el nombre de la carpeta donde van a vivir.
- `-t async` usa la plantilla para drivers asincrónicos, como asyncpg.

Se crean estos archivos:

```text
tareas-api/
├── alembic.ini              # configuración general de Alembic
└── migraciones/
    ├── env.py               # cómo se conecta Alembic a tu base y a tus modelos
    ├── script.py.mako       # plantilla para las migraciones nuevas
    ├── README
    └── versions/            # acá van a ir las migraciones (vacía por ahora)
```

## Configurarlo

Alembic necesita saber dos cosas: **dónde está la base** y **cuáles son tus modelos**. Las dos se configuran en `migraciones/env.py`.

Arriba del todo, después de `from alembic import context`, agregá estos imports:

```python title="migraciones/env.py"
from alembic import context
from app import modelos  # noqa: F401  (registra las tablas en Base)
from app.config import obtener_configuracion
from app.db import Base
```

Más abajo, buscá el bloque que termina en `target_metadata = None` y reemplazalo por:

```python title="migraciones/env.py"
# La URL de la base sale de la configuración de la app (el .env), no de alembic.ini
config.set_main_option("sqlalchemy.url", obtener_configuracion().url_base_datos)

# Las tablas que Alembic compara con la base al generar migraciones
target_metadata = Base.metadata
```

- **La URL**: `alembic.ini` trae una línea `sqlalchemy.url = driver://user:pass@localhost/dbname`. En vez de escribir ahí la URL (y con ella la contraseña), la tomamos de tu configuración, que ya lee el `.env`. Así hay un solo lugar donde está la URL.
- **`target_metadata`**: es el registro de todas las tablas de `Base`. Igual que en `main.py`, hace falta importar `app.modelos` para que `Tarea` y `Usuario` se registren.

Comprobá que Alembic puede conectarse (con `tareas-db` prendido):

```bash
uv run alembic current
```

```text
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
```

No muestra ninguna versión porque todavía no hay migraciones: eso cambia en la próxima página. Si en cambio aparece un error de conexión, revisá que el contenedor esté corriendo y que `URL_BASE_DATOS` esté bien en el `.env`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Es el equivalente de **Prisma Migrate** (`prisma migrate dev` compara el `schema.prisma` con la base y genera la migración) o de las migraciones de Knex, Sequelize y TypeORM. La tabla `alembic_version` cumple el rol de `_prisma_migrations`.

??? example "Ver comparación en Django ⬇️"

    Es lo mismo que las migraciones de Django: `makemigrations` genera los archivos comparando los modelos, y `migrate` los aplica. La diferencia es que en Django vienen integradas; con SQLAlchemy se suma Alembic aparte, y se configura una vez en `env.py`. La tabla `alembic_version` es la versión simplificada de `django_migrations`.

??? example "Ver comparación en Spring Boot ⬇️"

    En el mundo Spring se usan **Flyway** o **Liquibase**. Con Flyway escribís las migraciones a mano en SQL (`V1__crear_tablas.sql`); Alembic se parece más a Liquibase, pero además puede autogenerarlas a partir de los modelos. Y reemplaza al `ddl-auto=update` de Hibernate, que en producción no se recomienda por el mismo motivo que `create_all`.

## Mini ejercicio

¿Cuáles de estos cambios necesitan una migración y cuáles no?

1. Agregar una columna `prioridad` al modelo `Tarea`.
2. Cambiar `min_length=1` por `min_length=3` en el `titulo` de `TareaCrear`.
3. Agregar una ruta `GET /tareas/hechas`.
4. Crear un modelo nuevo `Etiqueta` con su tabla.
5. Cambiar `String(100)` por `String(200)` en el `titulo` del modelo `Tarea`.

??? success "Ver solución"

    Necesitan migración **1, 4 y 5**: cambian la estructura de la base (una columna nueva, una tabla nueva y el largo de una columna).

    No la necesitan **2 y 3**: `TareaCrear` es un esquema de Pydantic y la ruta es código de la API; ninguno de los dos toca las tablas. La regla: si cambiaste algo en `app/modelos.py`, probablemente necesitás una migración.

---

**Siguiente:** [Primera migración →](10-primera-migracion.md)
