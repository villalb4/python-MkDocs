# Qué es un ORM y qué es SQLAlchemy

!!! info "En esta página"

    **Qué vas a aprender:** qué es un ORM, qué ventajas tiene frente a escribir SQL a mano, qué es SQLAlchemy y cómo instalarlo en tu proyecto junto con el driver de PostgreSQL.

    **Qué necesitás:** [Qué es PostgreSQL](03-que-es-postgresql.md).

## El problema de escribir SQL a mano

En la primera página del capítulo escribiste SQL como texto dentro de Python:

```python
conexion.execute("INSERT INTO tareas (titulo) VALUES ('Comprar pan')")
```

Funciona, pero a medida que la API crece aparecen problemas:

- **El editor no te ayuda.** Si escribís mal el nombre de una columna, te enterás recién al ejecutar.
- **Hay que convertir todo a mano.** La base devuelve filas como `(1, 'Comprar pan', 0)` y vos tenés que armar diccionarios u objetos.
- **Es peligroso armar SQL con datos del usuario.** Si metés un título que llegó en un pedido dentro del texto del SQL, alguien puede mandar un "título" que en realidad es código SQL. Ese ataque se llama ***inyección SQL*** y es uno de los más comunes.

## Qué es un ORM

Un **ORM** (*Object-Relational Mapper*) es una librería que traduce entre el mundo de las tablas y el de los objetos de Python:

| Base de datos | Python con un ORM |
| --- | --- |
| Tabla `tareas` | Clase `Tarea` |
| Columna `titulo` | Atributo `titulo` |
| Una fila | Un objeto `Tarea` |
| `SELECT ... WHERE ...` | Código Python que arma la consulta |

Vos escribís clases y código Python; el ORM genera el SQL, lo manda a la base de forma segura y te devuelve objetos.

## SQLAlchemy

**SQLAlchemy** es el ORM más usado de Python. Es maduro, muy completo y funciona con PostgreSQL, SQLite, MySQL y otras bases. En esta guía vas a usar su **versión 2**, con soporte asincrónico, que encaja con las rutas `async` de FastAPI.

Así se ve una tabla definida con SQLAlchemy, y una consulta:

```python title="vistazo.py"
from sqlalchemy import select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Tarea(Base):
    __tablename__ = "tareas"

    id: Mapped[int] = mapped_column(primary_key=True)
    titulo: Mapped[str]
    hecha: Mapped[bool] = mapped_column(default=False)


consulta = select(Tarea).where(Tarea.titulo == "Comprar pan")
print(consulta)
```

No te preocupes por cada detalle: la clase `Tarea` es el tema de la próxima página. Lo interesante es lo que muestra el `print`, el SQL que SQLAlchemy generó:

```text
SELECT tareas.id, tareas.titulo, tareas.hecha
FROM tareas
WHERE tareas.titulo = :titulo_1
```

Fijate en `:titulo_1`: el valor `"Comprar pan"` **no** quedó pegado en el SQL. SQLAlchemy lo manda a la base por separado, como un **parámetro**, y la base lo trata siempre como un dato, nunca como código. Así la inyección SQL es imposible.

Y al ser código Python, el editor te autocompleta `Tarea.titulo` y te avisa si escribiste `Tarea.tituo`.

### Ventajas y límites

**A favor:** menos código repetitivo, autocompletado, protección contra inyección SQL, y las tablas definidas en un solo lugar.

**En contra:** es una capa más para aprender, y para consultas muy complejas a veces conviene SQL directo (SQLAlchemy también lo permite). Por eso entender SQL, aunque sea lo básico, sigue siendo importante.

## Instalarlo en tu proyecto

En tu proyecto `tareas-api`, ejecutá:

```bash
uv add "sqlalchemy[asyncio]" asyncpg
```

Son dos piezas:

- **`sqlalchemy[asyncio]`**: el ORM, con el extra para trabajar de forma asincrónica.
- **`asyncpg`**: el **driver**, la librería que sabe hablar con PostgreSQL por la red. SQLAlchemy arma las consultas; asyncpg las transporta. Por eso la URL de conexión va a empezar con `postgresql+asyncpg://`: "PostgreSQL, usando el driver asyncpg".

Podés probar el `vistazo.py` de arriba con `uv run vistazo.py` desde la carpeta del proyecto: no necesita que PostgreSQL esté prendido, porque solo arma la consulta, no la ejecuta.

!!! note "Dos clases `Tarea` distintas"

    Ya tenés una clase de Pydantic (`TareaCrear`) y ahora vas a tener una de SQLAlchemy (`Tarea`). No compiten: hacen trabajos distintos. **Pydantic** describe los datos que **entran y salen** de la API (y los valida). **SQLAlchemy** describe cómo se **guardan** en la base. En la página de CRUD vas a ver cómo se pasan los datos de una a la otra.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Node los ORMs más usados son **Prisma**, **Sequelize**, **TypeORM** y **Drizzle**. SQLAlchemy cumple el mismo rol; lo más parecido en estilo es TypeORM o Drizzle, donde las tablas se definen en código. Y el par `sqlalchemy` + `asyncpg` equivale a un ORM + el driver `pg`.

??? example "Ver comparación en Django ⬇️"

    Django trae su propio ORM integrado (`models.Model`, `Tarea.objects.filter(...)`), y no se suele usar fuera de Django. FastAPI no trae ORM, así que se elige uno: SQLAlchemy es el más común. La idea es la misma: `select(Tarea).where(Tarea.hecha == False)` equivale a `Tarea.objects.filter(hecha=False)`.

??? example "Ver comparación en Spring Boot ⬇️"

    SQLAlchemy es el equivalente de **Hibernate** (JPA): clases que representan tablas y un lenguaje para armar consultas sin escribir SQL. `select(Tarea).where(...)` se parece a la *Criteria API* de JPA, y asyncpg cumple el rol del driver JDBC de PostgreSQL.

## Mini ejercicio

Sin ejecutarlo, ¿qué SQL creés que genera esta consulta? Agregala al final de `vistazo.py` y comprobalo.

```python
print(select(Tarea.titulo).where(Tarea.id > 10).order_by(Tarea.titulo))
```

??? success "Ver solución"

    ```text
    SELECT tareas.titulo
    FROM tareas
    WHERE tareas.id > :id_1 ORDER BY tareas.titulo
    ```

    `select(Tarea.titulo)` pide solo esa columna (en vez de `select(Tarea)`, que pide todas). `order_by` agrega el orden, y el `10` vuelve a viajar como parámetro (`:id_1`).

---

**Siguiente:** [Definir un modelo →](05-definir-un-modelo.md)
