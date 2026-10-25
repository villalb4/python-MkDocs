# Tablas, filas y columnas

!!! info "En esta página"

    **Qué vas a aprender:** cómo se define una tabla, qué tipos de datos pueden tener sus columnas, qué es `NULL`, y qué son la clave primaria y las restricciones que protegen tus datos.

    **Qué necesitás:** [Qué es una base de datos relacional](01-que-es-una-base-de-datos.md).

## Una tabla por tipo de cosa

Cada tabla guarda **un** tipo de cosa: usuarios, tareas, productos. Por convención, el nombre va en plural y en minúsculas con guiones bajos (`usuarios`, `tareas_compartidas`).

Dentro de la tabla:

- Cada **columna** es un dato que tienen **todos** los registros: el título, si está hecha. Tiene un nombre y un **tipo**.
- Cada **fila** es **un** registro: una tarea concreta.

Antes de guardar filas hay que **definir** la tabla: decir qué columnas tiene, de qué tipo es cada una y qué reglas cumplen. A eso se lo llama el **esquema**. Así se definen las tablas de usuarios y tareas en PostgreSQL:

```sql
CREATE TABLE usuarios (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    creado_en TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tareas (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    titulo VARCHAR(100) NOT NULL,
    descripcion TEXT,
    hecha BOOLEAN NOT NULL DEFAULT false,
    usuario_id INTEGER NOT NULL REFERENCES usuarios (id)
);
```

No hace falta que lo ejecutes: vas a tener PostgreSQL funcionando en el Capítulo 4, y para entonces SQLAlchemy va a escribir este SQL por vos. Pero vale la pena saber leerlo. Cada renglón de adentro es una columna: **nombre**, **tipo** y, opcionalmente, **reglas**.

## Tipos de datos

A diferencia de Python, en una base de datos cada columna tiene un tipo fijo, y la base rechaza lo que no encaja. Los más comunes en PostgreSQL:

| Tipo | Guarda | En Python |
| --- | --- | --- |
| `INTEGER` | Números enteros | `int` |
| `VARCHAR(100)` | Texto de hasta 100 caracteres | `str` |
| `TEXT` | Texto sin límite de largo | `str` |
| `BOOLEAN` | `true` / `false` | `bool` |
| `NUMERIC(10, 2)` | Números decimales exactos (10 dígitos, 2 decimales) | `Decimal` |
| `TIMESTAMPTZ` | Fecha y hora, con zona horaria | `datetime` |
| `DATE` | Solo fecha | `date` |

Si intentás guardar algo que no corresponde, la base se niega:

```text
ERROR:  invalid input syntax for type boolean: "tal vez"
ERROR:  value too long for type character varying(100)
```

!!! warning "Plata: `NUMERIC`, nunca `float`"

    Los `float` no pueden representar exactamente algunos decimales: en PostgreSQL, igual que en Python, `0.1 + 0.2` con `float` da `0.30000000000000004`. Con `NUMERIC` da `0.3` exacto. Para precios y montos, usá siempre `NUMERIC`.

## `NULL`: "no hay valor"

`NULL` es el equivalente de `None` en Python: significa que la columna **no tiene valor**. No es lo mismo que `0` ni que un texto vacío `""`.

En la tabla `tareas`, `descripcion` no dice `NOT NULL`, así que puede quedar vacía. Para preguntar por ella se usa `IS NULL` (no `= NULL`, que no funciona):

```sql
SELECT titulo FROM tareas WHERE descripcion IS NULL;
```

## Restricciones: reglas que la base hace cumplir

Las palabras que van después del tipo son **restricciones** (*constraints*). La base las verifica en **cada** `INSERT` y `UPDATE`, venga de tu API o de cualquier otro lado.

| Restricción | Significa |
| --- | --- |
| `NOT NULL` | La columna no puede quedar vacía. |
| `DEFAULT valor` | Si no se indica un valor, usa este. `now()` es la fecha y hora actuales. |
| `UNIQUE` | No puede haber dos filas con el mismo valor (dos usuarios con el mismo email). |
| `PRIMARY KEY` | Identifica a cada fila: es única y no puede ser `NULL`. |
| `REFERENCES tabla (columna)` | El valor tiene que existir en otra tabla (lo vas a ver en la página de relaciones). |

Cuando se viola una, la base rechaza la operación con un error:

```text
ERROR:  null value in column "titulo" of relation "tareas" violates not-null constraint

ERROR:  duplicate key value violates unique constraint "usuarios_email_key"
DETAIL:  Key (email)=(ana@ejemplo.com) already exists.

ERROR:  insert or update on table "tareas" violates foreign key constraint "tareas_usuario_id_fkey"
DETAIL:  Key (usuario_id)=(99) is not present in table "usuarios".
```

Fijate que es la misma idea que la validación de Pydantic, pero en otra capa: Pydantic revisa lo que **llega** a tu API; las restricciones protegen lo que **queda guardado**. Se usan las dos.

## La clave primaria

Toda tabla necesita una forma de identificar cada fila sin ambigüedad: la **clave primaria** (*primary key*). Casi siempre es una columna `id` numérica:

```sql
id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

- `PRIMARY KEY`: el valor es único y nunca es `NULL`.
- `GENERATED ALWAYS AS IDENTITY`: lo genera la base, empezando en 1 y sumando de a uno. Nunca se repite, aunque borres filas.

Esto reemplaza al `max(...) + 1` que usabas con la lista. De hecho, la base no te deja elegir el id a mano:

```text
ERROR:  cannot insert a non-DEFAULT value into column "id"
DETAIL:  Column "id" is an identity column defined as GENERATED ALWAYS.
```

Con estas tablas, al insertar dos tareas y pedirlas con `SELECT * FROM tareas;`, PostgreSQL muestra:

```text
 id |    titulo    | descripcion | hecha | usuario_id
----+--------------+-------------+-------+------------
  1 | Comprar pan  |             | f     |          1
  2 | Estudiar SQL | Capítulo 3  | t     |          1
```

La `descripcion` vacía es un `NULL`; `f` y `t` son `false` y `true`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Las tablas y el SQL son los mismos que usaría un proyecto de Node con PostgreSQL. Con Prisma, el esquema se escribe en `schema.prisma` (`id Int @id @default(autoincrement())`, `email String @unique`) y Prisma genera el `CREATE TABLE`. En esta guía, ese rol lo cumple un modelo de SQLAlchemy.

??? example "Ver comparación en Django ⬇️"

    En Django, cada `models.Model` se convierte en una tabla, y cada campo en una columna: `CharField(max_length=100)` es `VARCHAR(100)`, `null=False` es `NOT NULL`, `unique=True` es `UNIQUE`, y el `id` automático es la clave primaria. Lo vas a ver muy parecido con SQLAlchemy.

??? example "Ver comparación en Spring Boot ⬇️"

    Con JPA, una clase `@Entity` es una tabla: `@Id @GeneratedValue(strategy = GenerationType.IDENTITY)` es la clave primaria generada, `@Column(nullable = false, unique = true, length = 100)` define las restricciones, y `BigDecimal` se mapea a `NUMERIC`.

## Mini ejercicio

Escribí el `CREATE TABLE` de una tabla `productos` con: un `id` generado por la base, un `nombre` de hasta 100 caracteres obligatorio y que no se repita, un `precio` obligatorio con dos decimales, un `stock` entero que por defecto sea 0, y un `en_oferta` que por defecto sea falso.

??? success "Ver solución"

    ```sql
    CREATE TABLE productos (
        id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
        nombre VARCHAR(100) NOT NULL UNIQUE,
        precio NUMERIC(10, 2) NOT NULL,
        stock INTEGER NOT NULL DEFAULT 0,
        en_oferta BOOLEAN NOT NULL DEFAULT false
    );
    ```

    Con `INSERT INTO productos (nombre, precio) VALUES ('Teclado', 45000.50);`, la fila queda con `stock` en `0` y `en_oferta` en `false`. Si intentás insertar otro `'Teclado'`, la base responde `duplicate key value violates unique constraint "productos_nombre_key"`.

---

**Siguiente:** [Qué es PostgreSQL →](03-que-es-postgresql.md)
