# Qué es PostgreSQL

!!! info "En esta página"

    **Qué vas a aprender:** qué es PostgreSQL, en qué se diferencia de SQLite, cómo es una URL de conexión, y cómo levantarlo en tu computadora con un solo comando de Docker para empezar a usarlo.

    **Qué necesitás:** [Tablas, filas y columnas](02-tablas-filas-columnas.md) y Docker Desktop instalado ([Qué necesitás instalar](../introduccion/02-que-necesitas-instalar.md)).

## Una base de datos servidor

**PostgreSQL** (o simplemente *Postgres*) es una base de datos relacional de código abierto, gratuita, con más de 30 años de desarrollo. Es una de las más usadas del mundo y la opción por defecto para muchos backends.

La gran diferencia con el SQLite de la primera página es **cómo funciona**:

| | SQLite | PostgreSQL |
| --- | --- | --- |
| Qué es | Una librería que lee y escribe un archivo | Un **servidor**: un programa que queda corriendo |
| Cómo te conectás | Abrís el archivo | Por la red, a un puerto (por defecto, el `5432`) |
| Usuarios y contraseñas | No tiene | Sí |
| Muchos pedidos a la vez | Limitado | Diseñado para eso |
| Ideal para | Pruebas, apps chicas, celulares | APIs y aplicaciones en producción |

Es el mismo modelo que ya conocés con tu API: PostgreSQL es un servidor que escucha en un puerto, y tu API es su **cliente**. Un mismo servidor puede tener varias **bases de datos** adentro (por ejemplo, una por proyecto), y cada una tiene sus propias tablas.

## La URL de conexión

Para conectarse, un cliente necesita cinco datos, que se suelen escribir juntos en una **URL de conexión**:

```text
postgresql://tareas:tareas@localhost:5432/tareas
             ──┬─── ──┬─── ───┬───── ─┬── ──┬───
               │      │       │       │     └── nombre de la base
               │      │       │       └── puerto
               │      │       └── servidor (host)
               │      └── contraseña
               └── usuario
```

Esta URL va a ser un dato de tu configuración (en el `.env`, nunca en el código), y es lo único que tu API necesita para encontrar la base.

## Levantarlo con Docker

PostgreSQL se puede instalar como cualquier programa, pero la forma más simple y prolija es correrlo dentro de **Docker**: no ensucia tu sistema, y se borra con un comando. En el Capítulo 4 vas a entender en detalle qué hace cada parte; por ahora, con el Docker Desktop abierto, ejecutá:

=== "macOS / Linux"

    ```bash
    docker run --name tareas-db -e POSTGRES_USER=tareas -e POSTGRES_PASSWORD=tareas -e POSTGRES_DB=tareas -p 5432:5432 -d postgres:17
    ```

=== "Windows (PowerShell)"

    ```powershell
    docker run --name tareas-db -e POSTGRES_USER=tareas -e POSTGRES_PASSWORD=tareas -e POSTGRES_DB=tareas -p 5432:5432 -d postgres:17
    ```

La primera vez tarda un poco, porque descarga PostgreSQL. A grandes rasgos:

- `--name tareas-db`: el nombre con el que vas a referirte a este PostgreSQL.
- `-e POSTGRES_...`: crea un usuario `tareas`, con contraseña `tareas`, y una base llamada `tareas`. Son valores de práctica; en un servidor real, la contraseña tiene que ser larga y secreta.
- `-p 5432:5432`: conecta el puerto 5432 de tu computadora con el del PostgreSQL.
- `-d`: lo deja corriendo en segundo plano.
- `postgres:17`: la versión 17 de PostgreSQL.

Comprobá que está corriendo con `docker ps`. Vas a ver una fila como esta (recorté algunas columnas):

```text
NAMES       IMAGE         STATUS         PORTS
tareas-db   postgres:17   Up 5 seconds   0.0.0.0:5432->5432/tcp
```

!!! warning "Si el puerto 5432 está ocupado"

    Si ya tenés otro PostgreSQL corriendo, Docker responde `Bind for 0.0.0.0:5432 failed: port is already allocated`. Usá otro puerto de tu lado, por ejemplo `-p 5433:5432`, y acordate de poner `5433` en la URL de conexión. Antes de reintentar, borrá el intento fallido con `docker rm tareas-db`.

Para apagarlo y volver a prenderlo, sin perder los datos:

```bash
docker stop tareas-db
docker start tareas-db
```

## Hablar con PostgreSQL: `psql`

PostgreSQL trae `psql`, una consola para escribir SQL directamente. Como está dentro del contenedor, se abre así:

```bash
docker exec -it tareas-db psql -U tareas
```

Aparece el símbolo `tareas=#`. Probá:

```sql
CREATE TABLE prueba (id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY, nombre TEXT NOT NULL);
INSERT INTO prueba (nombre) VALUES ('hola');
SELECT * FROM prueba;
```

```text
 id | nombre
----+--------
  1 | hola
(1 row)
```

Cada instrucción SQL termina en `;`. Además del SQL, `psql` tiene comandos propios que empiezan con `\`:

| Comando | Hace |
| --- | --- |
| `\dt` | Lista las tablas. |
| `\d prueba` | Muestra las columnas y restricciones de la tabla `prueba`. |
| `\l` | Lista las bases de datos del servidor. |
| `\q` | Sale de `psql`. |

!!! tip "Clientes con ventanas"

    Si preferís algo visual, hay programas gratuitos como **DBeaver** o **pgAdmin** que se conectan con los mismos datos de la URL y muestran las tablas como planillas.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Es el mismo PostgreSQL que usaría un proyecto de Node, con la misma URL de conexión. Lo único que cambia es el cliente: en Node se pasa a `pg` o a Prisma (`DATABASE_URL="postgresql://..."`); en esta guía, a SQLAlchemy.

??? example "Ver comparación en Django ⬇️"

    En Django, pasar de SQLite a PostgreSQL es cambiar `ENGINE` a `django.db.backends.postgresql` y completar `NAME`, `USER`, `PASSWORD`, `HOST` y `PORT` en `DATABASES`: los mismos cinco datos de la URL de conexión. Con `dj-database-url` se escriben como una sola URL, igual que acá.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring se configura con `spring.datasource.url=jdbc:postgresql://localhost:5432/tareas`, más `username` y `password` aparte. Es la misma información con otro formato: en Python el usuario y la contraseña van dentro de la URL.

## Mini ejercicio

Con `psql`, mirá la estructura de la tabla `prueba` y agregale dos filas más. Después salí de `psql`, apagá el contenedor, volvé a prenderlo y comprobá que las filas siguen ahí. Al final, borrá la tabla de práctica.

??? success "Ver solución"

    ```bash
    docker exec -it tareas-db psql -U tareas
    ```

    ```sql
    \d prueba
    INSERT INTO prueba (nombre) VALUES ('mundo'), ('chau');
    \q
    ```

    ```bash
    docker stop tareas-db
    docker start tareas-db
    docker exec -it tareas-db psql -U tareas -c "SELECT * FROM prueba;"
    ```

    Las tres filas siguen ahí: detener el contenedor no borra los datos. Con `-c`, `psql` ejecuta una sola instrucción y sale. Para borrar la tabla: `docker exec -it tareas-db psql -U tareas -c "DROP TABLE prueba;"`.

---

**Siguiente:** [Qué es un ORM y qué es SQLAlchemy →](04-orm-sqlalchemy.md)
