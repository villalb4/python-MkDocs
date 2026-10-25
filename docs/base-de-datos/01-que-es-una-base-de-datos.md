# Qué es una base de datos relacional

!!! info "En esta página"

    **Qué vas a aprender:** por qué una API necesita una base de datos, qué significa que sea "relacional" y qué es SQL, el lenguaje para hablar con ella. Vas a escribir tus primeras consultas con una base que ya viene con Python.

    **Qué necesitás:** haber terminado el [Capítulo 2](../fastapi/14-lifespan.md).

## Por qué no alcanza con una lista (ni con un archivo)

Al final del Capítulo 2, tus tareas se guardan en `tareas.json` al apagar el servidor. Funciona para aprender, pero en una API de verdad se rompe por varios lados:

- **Si el servidor se corta de golpe** (un corte de luz, un error grave), todo lo creado desde el último arranque se pierde.
- **Dos pedidos al mismo tiempo** pueden pisarse: los dos leen la lista, los dos agregan algo, y uno de los cambios desaparece.
- **Para buscar algo hay que leer todo.** Con diez tareas no importa; con un millón, sí.
- **Nadie controla las reglas.** Nada impide que se guarde una tarea sin título o con un id repetido.
- **Varias copias de la API** (algo normal en producción) tendrían cada una su propia lista.

Una **base de datos** resuelve todo eso. Es un programa dedicado exclusivamente a guardar datos de forma segura: los escribe en disco apenas se confirman, maneja muchos pedidos a la vez sin que se pisen, busca rápido aunque haya millones de registros y hace cumplir las reglas que le definas. Tu API deja de guardar datos ella misma y pasa a **pedírselos** a la base.

## Qué significa "relacional"

En una base de datos **relacional**, los datos se organizan en **tablas**, parecidas a una hoja de cálculo: cada tabla guarda un tipo de cosa, cada **fila** es un registro y cada **columna**, un dato de ese registro.

```text
Tabla usuarios                    Tabla tareas
┌────┬────────┐                   ┌────┬──────────────────┬───────┬────────────┐
│ id │ nombre │                   │ id │ titulo           │ hecha │ usuario_id │
├────┼────────┤                   ├────┼──────────────────┼───────┼────────────┤
│ 1  │ Ana    │ ◄─────────────────│ 1  │ Comprar pan      │ no    │ 1          │
│ 2  │ Juan   │ ◄──────┐          │ 2  │ Estudiar SQL     │ sí    │ 1          │
└────┴────────┘        └──────────│ 3  │ Llamar al médico │ no    │ 2          │
                                  └────┴──────────────────┴───────┴────────────┘
```

Lo "relacional" es la columna `usuario_id`: cada tarea guarda el `id` del usuario al que pertenece. Así las tablas se **relacionan** entre sí sin repetir datos: el nombre "Ana" está escrito una sola vez, aunque tenga muchas tareas. Las tablas y las relaciones las vas a ver en detalle en las próximas páginas.

Existen también bases **no relacionales** (llamadas *NoSQL*, como MongoDB o Redis), que guardan los datos de otras formas. Para la mayoría de las aplicaciones, una base relacional es la opción más sólida, y es la que vas a usar en esta guía: **PostgreSQL**.

## SQL: el lenguaje de las bases relacionales

Para pedirle cosas a una base relacional se usa **SQL** (*Structured Query Language*). Son instrucciones que se parecen a frases en inglés. Las cuatro básicas se corresponden con los métodos HTTP que ya conocés:

| SQL | Hace | Método HTTP equivalente |
| --- | --- | --- |
| `SELECT` | Leer filas | `GET` |
| `INSERT` | Agregar filas | `POST` |
| `UPDATE` | Modificar filas | `PUT` |
| `DELETE` | Borrar filas | `DELETE` |

Por ejemplo, "dame el id y el título de las tareas no hechas" se escribe:

```sql
SELECT id, titulo FROM tareas WHERE hecha = false;
```

## Probalo sin instalar nada

Python trae incluido **SQLite**, una base de datos relacional que guarda todo en un solo archivo. No es la que vas a usar en la API, pero sirve para probar SQL ahora mismo. Creá este script en una carpeta de práctica:

```python title="probar_sql.py"
import sqlite3

conexion = sqlite3.connect("prueba.db")

conexion.execute("""
    CREATE TABLE IF NOT EXISTS tareas (
        id INTEGER PRIMARY KEY,
        titulo TEXT NOT NULL,
        hecha BOOLEAN NOT NULL DEFAULT 0
    )
""")
conexion.execute("INSERT INTO tareas (titulo) VALUES ('Comprar pan')")
conexion.execute("INSERT INTO tareas (titulo, hecha) VALUES ('Estudiar SQL', 1)")
conexion.commit()

for fila in conexion.execute("SELECT id, titulo, hecha FROM tareas WHERE hecha = 0"):
    print(fila)

conexion.close()
```

Ejecutalo con `uv run probar_sql.py`:

```text
(1, 'Comprar pan', 0)
```

Qué pasó:

1. `sqlite3.connect("prueba.db")` abre (o crea) el archivo de la base.
2. `CREATE TABLE` crea la tabla `tareas` con tres columnas, si no existía. `NOT NULL` es una regla: el título no puede faltar.
3. Dos `INSERT` agregan tareas. Fijate que no pasamos el `id`: la base lo asigna sola.
4. `commit()` **confirma** los cambios. Hasta ese momento, no están guardados.
5. El `SELECT` con `WHERE hecha = 0` trae solo las pendientes. SQLite guarda `True`/`False` como `1`/`0`.

Ahora ejecutalo **otra vez**:

```text
(1, 'Comprar pan', 0)
(3, 'Comprar pan', 0)
```

Los datos de la primera vez **siguieron ahí**: quedaron guardados en `prueba.db`. El script volvió a insertar, y la base asignó ids nuevos sin repetir ninguno. Eso es justamente lo que tu lista en memoria no podía hacer.

!!! note "No vas a escribir mucho SQL a mano"

    En la API vas a usar SQLAlchemy, que escribe el SQL por vos a partir de código Python. Igual conviene entender qué pasa por debajo: cuando algo falle, el error va a hablar de tablas, columnas y consultas.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Las bases de datos son las mismas en cualquier lenguaje: un proyecto de Node con PostgreSQL usa exactamente la misma base que vas a usar acá. Cambia la librería para hablar con ella: en Node, `pg` para SQL directo, o un ORM como Prisma, Sequelize o TypeORM. En Python, `sqlite3` o `asyncpg` para SQL directo, y SQLAlchemy como ORM.

??? example "Ver comparación en Django ⬇️"

    Django viene configurado con SQLite por defecto (el archivo `db.sqlite3`), que es la misma base que probaste acá. En proyectos reales se cambia a PostgreSQL en `DATABASES`. Lo que en Django hace su ORM (`models.Model`), en esta guía lo va a hacer SQLAlchemy.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring Boot hablás con la base con JDBC (SQL directo) o, lo más común, con JPA/Hibernate. Para probar sin instalar nada se suele usar H2, una base en memoria: cumple el mismo rol que SQLite en esta página. SQLAlchemy ocupa el lugar de Hibernate.

## Mini ejercicio

Borrá `prueba.db`, ejecutá `probar_sql.py` una vez, y después escribí otro script que, sobre esa misma base, marque como hecha la tarea con id 1, borre la tarea "Estudiar SQL" y muestre **todas** las tareas que quedan.

??? success "Ver solución"

    ```python title="ejercicio.py"
    import sqlite3

    conexion = sqlite3.connect("prueba.db")

    conexion.execute("UPDATE tareas SET hecha = 1 WHERE id = 1")
    conexion.execute("DELETE FROM tareas WHERE titulo = 'Estudiar SQL'")
    conexion.commit()

    for fila in conexion.execute("SELECT id, titulo, hecha FROM tareas"):
        print(fila)

    conexion.close()
    ```

    ```text
    (1, 'Comprar pan', 1)
    ```

    `UPDATE ... SET ... WHERE` modifica solo las filas que cumplen la condición, y `DELETE ... WHERE` borra solo esas. ¡Cuidado! Sin `WHERE`, `UPDATE` y `DELETE` afectan a **todas** las filas de la tabla.

---

**Siguiente:** [Tablas, filas y columnas →](02-tablas-filas-columnas.md)
