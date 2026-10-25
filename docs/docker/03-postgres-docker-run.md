# Levantar PostgreSQL con `docker run`

!!! info "En esta página"

    **Qué vas a aprender:** qué hace cada parte del comando `docker run` que usaste para PostgreSQL, cómo ver los logs de un contenedor, y qué pasa con los datos cuando lo detenés y cuando lo borrás.

    **Qué necesitás:** [Instalar Docker Desktop y comprobar](02-instalar-docker.md) y [Qué es PostgreSQL](../base-de-datos/03-que-es-postgresql.md).

## El comando, parte por parte

En el Capítulo 3 levantaste PostgreSQL con este comando, sin entrar en detalles:

```bash
docker run --name tareas-db -e POSTGRES_USER=tareas -e POSTGRES_PASSWORD=tareas -e POSTGRES_DB=tareas -p 5432:5432 -d postgres:17
```

Ahora que sabés qué es una imagen y qué es un contenedor, se lee así:

| Parte | Significa |
| --- | --- |
| `docker run` | Creá un contenedor nuevo y ejecutalo. |
| `--name tareas-db` | Llamalo `tareas-db`. Sin esto, Docker inventa un nombre al azar. |
| `-e POSTGRES_USER=tareas` | Pasale una **variable de entorno** al contenedor. |
| `-p 5432:5432` | Conectá un **puerto** de tu computadora con uno del contenedor. |
| `-d` | Ejecutalo en segundo plano (*detached*) y devolveme la terminal. |
| `postgres:17` | La imagen de la que se crea el contenedor. |

El orden importa en un solo punto: las opciones van **antes** del nombre de la imagen. Lo que va después se toma como el comando a ejecutar adentro, como el `python -c ...` de la primera página.

## Variables de entorno: la configuración del contenedor

La imagen oficial de PostgreSQL lee tres variables al arrancar **por primera vez**:

- `POSTGRES_USER`: el usuario a crear.
- `POSTGRES_PASSWORD`: su contraseña. Es **obligatoria**.
- `POSTGRES_DB`: una base de datos a crear.

Es la misma idea que la configuración de tu API: el programa no trae la configuración escrita adentro, sino que la lee del entorno. Si te olvidás la contraseña, el contenedor arranca y se detiene enseguida, y sus logs dicen:

```text
Error: Database is uninitialized and superuser password is not specified.
       You must specify POSTGRES_PASSWORD to a non-empty value for the
       superuser. For example, "-e POSTGRES_PASSWORD=password" on "docker run".
```

## Puertos: `afuera:adentro`

El contenedor tiene su propia red, aislada. Adentro, PostgreSQL escucha en el puerto `5432`, pero nadie de afuera puede llegar hasta ahí... salvo que se lo publique con `-p`:

```text
-p 5432:5432
   ──┬─ ──┬─
     │    └── puerto adentro del contenedor (donde escucha PostgreSQL)
     └── puerto en tu computadora (el que usás en la URL)
```

Por eso, si el 5432 de tu computadora está ocupado, se cambia **solo el de la izquierda**: con `-p 5433:5432`, desde tu computadora te conectás al `5433`, y adentro PostgreSQL sigue en su `5432` de siempre.

## Ver qué está pasando: `docker logs`

Como el contenedor corre en segundo plano, no ves lo que escribe. Para leerlo:

```bash
docker logs tareas-db
```

Al final vas a ver algo así:

```text
LOG:  starting PostgreSQL 17.11 (Debian 17.11-1.pgdg13+2) on aarch64-unknown-linux-gnu, ...
LOG:  listening on IPv4 address "0.0.0.0", port 5432
LOG:  database system is ready to accept connections
```

`ready to accept connections` significa que ya podés conectarte. Con `docker logs -f tareas-db` los logs se siguen mostrando en vivo, hasta que apretás <kbd>Ctrl</kbd> + <kbd>C</kbd>.

## El ciclo de vida y los datos

| Comando | Qué hace | ¿Los datos siguen? |
| --- | --- | --- |
| `docker stop tareas-db` | Detiene el contenedor. | Sí |
| `docker start tareas-db` | Lo vuelve a arrancar. | Sí |
| `docker rm tareas-db` | **Borra** el contenedor (tiene que estar detenido). | **No** |

Probalo. Creá una tabla con una fila:

```bash
docker exec -it tareas-db psql -U tareas -c "CREATE TABLE notas (texto TEXT); INSERT INTO notas VALUES ('sigo acá');"
```

Detené y arrancá el contenedor, y la fila sigue ahí:

```bash
docker stop tareas-db
docker start tareas-db
docker exec -it tareas-db psql -U tareas -c "SELECT * FROM notas;"
```

Pero si lo **borrás** y creás uno nuevo con el mismo `docker run`:

```bash
docker stop tareas-db
docker rm tareas-db
docker run --name tareas-db -e POSTGRES_USER=tareas -e POSTGRES_PASSWORD=tareas -e POSTGRES_DB=tareas -p 5432:5432 -d postgres:17
docker exec -it tareas-db psql -U tareas -c "SELECT * FROM notas;"
```

```text
ERROR:  relation "notas" does not exist
```

La tabla desapareció. Los datos vivían **dentro** del contenedor, y al borrarlo se fueron con él. El contenedor nuevo es una copia limpia de la imagen. Para guardar los datos aparte, de forma que sobrevivan aunque borres el contenedor, existen los **volúmenes**, que vas a ver en dos páginas.

!!! warning "Dos errores típicos"

    - **`Conflict. The container name "/tareas-db" is already in use`**: ya existe un contenedor con ese nombre (aunque esté detenido). Borralo con `docker rm tareas-db` o usá otro nombre.
    - **`cannot remove container "tareas-db": container is running`**: primero `docker stop`, después `docker rm`. O las dos cosas juntas con `docker rm -f tareas-db`.

## `docker exec`: entrar al contenedor

`docker exec` ejecuta un comando **dentro** de un contenedor que ya está corriendo. Lo usaste para abrir `psql`:

```bash
docker exec -it tareas-db psql -U tareas
```

`-it` hace que la sesión sea interactiva (podés escribir y ver las respuestas). Sin `-it` sirve para comandos que no necesitan que escribas nada, como `psql -c "..."`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Un proyecto de Node con PostgreSQL levanta la base exactamente con este mismo comando. Las variables `-e` cumplen el rol de las que en Node leés con `process.env`, pero acá las lee el programa que corre dentro del contenedor.

??? example "Ver comparación en Django ⬇️"

    Lo mismo: con este contenedor corriendo, alcanza con apuntar `DATABASES` (o tu `DATABASE_URL`) a `localhost:5432` con usuario, contraseña y base `tareas`. Para Django, ese PostgreSQL es igual que uno instalado a mano.

??? example "Ver comparación en Spring Boot ⬇️"

    Igual que para los demás: `spring.datasource.url=jdbc:postgresql://localhost:5432/tareas`. Spring también ofrece *Testcontainers* para levantar contenedores así desde los tests; por debajo hace lo mismo que este `docker run`.

## Mini ejercicio

Levantá un **segundo** PostgreSQL, independiente del primero, llamado `otra-db`, con usuario `ana`, contraseña `secreto` y una base llamada `pruebas`, en el puerto `5434` de tu computadora. Conectate con `psql` para comprobarlo y después borralo.

??? success "Ver solución"

    ```bash
    docker run --name otra-db -e POSTGRES_USER=ana -e POSTGRES_PASSWORD=secreto -e POSTGRES_DB=pruebas -p 5434:5432 -d postgres:17
    docker exec -it otra-db psql -U ana -d pruebas -c "SELECT current_user, current_database();"
    ```

    ```text
     current_user | current_database
    --------------+------------------
     ana          | pruebas
    ```

    `-d pruebas` le dice a `psql` a qué base conectarse (cuando el usuario y la base se llaman distinto, hace falta). Los dos contenedores salen de la misma imagen y no comparten nada: cada uno tiene sus usuarios y sus datos. Para borrarlo: `docker rm -f otra-db`.

---

**Siguiente:** [`docker-compose.yml`: qué es y para qué sirve →](04-docker-compose.md)
