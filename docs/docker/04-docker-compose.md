# `docker-compose.yml`: qué es y para qué sirve

!!! info "En esta página"

    **Qué vas a aprender:** cómo describir tus contenedores en un archivo `docker-compose.yml` en vez de escribir comandos largos, y cómo levantarlos, verlos y apagarlos con Docker Compose.

    **Qué necesitás:** [Levantar PostgreSQL con `docker run`](03-postgres-docker-run.md).

## El problema con `docker run`

El comando para levantar PostgreSQL ya es largo, y es uno solo:

```bash
docker run --name tareas-db -e POSTGRES_USER=tareas -e POSTGRES_PASSWORD=tareas -e POSTGRES_DB=tareas -p 5432:5432 -d postgres:17
```

Cuando tu proyecto tenga varios servicios (la base, la API, quizás algo más), vas a tener varios comandos así, que hay que recordar, escribir sin errores y explicarle a cualquiera que se sume al proyecto.

**Docker Compose** resuelve esto: describís todos los servicios en un archivo, y los levantás con un solo comando. El archivo vive en el repositorio, junto al código, así que cualquiera que lo clone tiene exactamente la misma configuración.

## El archivo

Primero, si te quedó el contenedor `tareas-db` de páginas anteriores, borralo para liberar el puerto: `docker rm -f tareas-db`.

En la raíz de tu proyecto `tareas-api`, creá `docker-compose.yml`:

```yaml title="docker-compose.yml"
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: tareas
      POSTGRES_PASSWORD: tareas
      POSTGRES_DB: tareas
    ports:
      - "5432:5432"
```

Es el mismo `docker run` de arriba, ordenado:

| `docker run` | `docker-compose.yml` |
| --- | --- |
| `--name tareas-db` | El nombre del servicio: `db` |
| `-e POSTGRES_USER=tareas` | `environment:` con `POSTGRES_USER: tareas` |
| `-p 5432:5432` | `ports:` con `- "5432:5432"` |
| `postgres:17` | `image: postgres:17` |
| `-d` | La opción `-d` de `docker compose up` |

(Si el puerto 5432 de tu computadora está ocupado, poné `"5433:5432"`, como antes.)

### Lo mínimo de YAML

El archivo está escrito en **YAML**, un formato para datos de configuración:

- `clave: valor` es un par, como en un diccionario.
- La **sangría** indica qué está adentro de qué, igual que en Python. Se usan espacios, nunca tabulaciones.
- Los renglones que empiezan con `-` son elementos de una lista (como los puertos).
- Las comillas en `"5432:5432"` evitan que YAML malinterprete los dos puntos.

Si la sangría está mal, Compose no arranca y señala el renglón con el problema, con un error como `did not find expected key`.

## Levantar todo

Desde la carpeta del proyecto:

```bash
docker compose up -d
```

```text
 Network tareas-api_default Created
 Container tareas-api-db-1 Created
 Container tareas-api-db-1 Started
```

Compose creó una **red** para el proyecto (lo vas a aprovechar en un momento) y el contenedor `tareas-api-db-1`. El nombre se arma con la carpeta del proyecto (`tareas-api`), el servicio (`db`) y un número.

## Los comandos del día a día

Todos se ejecutan desde la carpeta donde está el `docker-compose.yml`:

| Comando | Hace |
| --- | --- |
| `docker compose up -d` | Crea y arranca los servicios (los que ya existen, los deja como están). |
| `docker compose ps` | Lista los servicios del proyecto y su estado. |
| `docker compose logs db` | Muestra los logs del servicio `db`. Con `-f`, en vivo. |
| `docker compose exec db psql -U tareas` | Ejecuta un comando dentro del servicio `db`. |
| `docker compose stop` | Detiene los servicios, sin borrarlos. |
| `docker compose start` | Los vuelve a arrancar. |
| `docker compose down` | Detiene y **borra** los contenedores y la red. |

Fijate que con Compose usás el **nombre del servicio** (`db`), no el del contenedor. Por ejemplo, para abrir `psql`:

```bash
docker compose exec db psql -U tareas -c "SELECT current_database();"
```

Tu API no necesita ningún cambio: se sigue conectando a `localhost:5432`, igual que con `docker run`.

!!! warning "`down` borra los datos"

    Igual que `docker rm`, `docker compose down` borra los contenedores, y con ellos los datos de la base. Para el día a día usá `stop` y `start`. En la próxima página vas a agregar un volumen para que los datos sobrevivan incluso a un `down`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    El archivo es el mismo para cualquier lenguaje. Muchos proyectos de Node traen un `docker-compose.yml` para levantar PostgreSQL o Redis en desarrollo, y lo usan con los mismos comandos. Si alguna vez usaste scripts de `package.json` para levantar servicios, Compose cumple ese rol de forma estándar.

??? example "Ver comparación en Django ⬇️"

    Un `docker-compose.yml` con servicios `db` y `web` es casi un estándar en proyectos de Django, y la documentación oficial de Docker tiene un ejemplo con Django. Lo que armes en este capítulo para FastAPI se traslada casi sin cambios.

??? example "Ver comparación en Spring Boot ⬇️"

    Desde Spring Boot 3.1, si el proyecto tiene un `compose.yaml` y la dependencia `spring-boot-docker-compose`, Spring ejecuta `docker compose up` solo al arrancar la app y configura la conexión. En FastAPI lo levantás vos con el comando, pero el archivo es el mismo formato.

!!! tip "¿`docker-compose.yml` o `compose.yaml`?"

    Compose acepta los dos nombres. `compose.yaml` es el nombre que recomienda la documentación actual, y `docker-compose.yml` el histórico, que vas a ver en muchísimos proyectos. En esta guía usamos el segundo.

## Mini ejercicio

Agregá un segundo servicio, `adminer`, con la imagen `adminer:5` y el puerto `8080`. Adminer es una página web para ver y editar bases de datos. Levantá todo, abrí `http://localhost:8080` y entrá a tu base. Pista: en el campo **Server**, poné el nombre del servicio de la base.

??? success "Ver solución"

    ```yaml title="docker-compose.yml"
    services:
      db:
        image: postgres:17
        environment:
          POSTGRES_USER: tareas
          POSTGRES_PASSWORD: tareas
          POSTGRES_DB: tareas
        ports:
          - "5432:5432"

      adminer:
        image: adminer:5
        ports:
          - "8080:8080"
    ```

    Después de `docker compose up -d`, en `http://localhost:8080` elegí **System: PostgreSQL**, **Server: `db`**, usuario `tareas`, contraseña `tareas` y base `tareas`.

    ¿Por qué `db` y no `localhost`? Adminer corre **en su propio contenedor**: para él, `localhost` es su propio contenedor, no tu computadora. Compose conecta todos los servicios del proyecto en una misma red, donde cada uno se encuentra con los demás **por el nombre del servicio**. Esto va a ser clave cuando la API también corra en un contenedor.

---

**Siguiente:** [Volúmenes: dónde se guardan los datos →](05-volumenes.md)
