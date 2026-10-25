# Chuleta de comandos de `uv`, Docker y FastAPI

!!! info "En esta página"

    **Qué vas a aprender:** nada nuevo. Es una página de referencia con los comandos de la guía, agrupados por herramienta, para tenerlos a mano.

    **Qué necesitás:** nada. Cada sección enlaza a la página donde se explica la herramienta.

Todos los comandos se ejecutan **desde la carpeta del proyecto** y son iguales en macOS, Linux y Windows (PowerShell). Los nombres de ejemplo (`tareas-api`, `tareas-db`, el usuario `tareas`) son los de la guía: cambialos por los tuyos.

## `uv`: proyecto y dependencias

Se explica en [Instalar y usar `uv`](../python/02-instalar-uv.md) y en [Entornos virtuales](../python/21-entornos-virtuales.md). Para instalarlo, mirá la primera de esas páginas: el comando cambia según el sistema.

| Comando | Para qué |
| --- | --- |
| `uv --version` | Comprobar que `uv` está instalado. |
| `uv init --no-package tareas-api` | Crear un proyecto nuevo en la carpeta `tareas-api`. |
| `uv add "fastapi[standard]"` | Agregar una librería (y anotarla en `pyproject.toml` y `uv.lock`). |
| `uv add --dev pytest` | Agregar una herramienta de desarrollo, que no va a producción. |
| `uv remove pyjwt` | Sacar una librería. |
| `uv sync` | Instalar todo lo de `uv.lock`, por ejemplo después de clonar el proyecto. |
| `uv run hola.py` | Ejecutar un script con el Python y las librerías del proyecto. |
| `uv run --python 3.13 hola.py` | Ejecutarlo con otra versión de Python (`uv` la descarga si falta). |
| `uv run --env-file .env config.py` | Ejecutarlo con las variables del `.env`. |
| `uv tree --depth 1` | Ver las dependencias directas del proyecto. |
| `uv lock --upgrade` | Actualizar las dependencias a sus últimas versiones permitidas (y después `uv sync`). |
| `uv python list --only-installed` | Ver qué versiones de Python tenés instaladas. |
| `uv run python -c "import secrets; print(secrets.token_hex(32))"` | Generar una clave secreta de 64 caracteres, como `CLAVE_JWT`. |

## FastAPI: levantar la API

Se explica en [Uvicorn y recarga automática](../fastapi/03-uvicorn.md) y en [Preparar la app para producción](../calidad-y-despliegue/05-preparar-para-produccion.md#fastapi-run-y-no-fastapi-dev).

| Comando | Para qué |
| --- | --- |
| `uv run fastapi dev app/main.py` | Desarrollo: recarga automática al guardar, solo en `127.0.0.1:8000`. |
| `uv run fastapi dev app/main.py --port 8001` | Lo mismo, en otro puerto. |
| `uv run fastapi run app/main.py` | Producción: sin recarga, escuchando en `0.0.0.0:8000`. |
| `uv run fastapi run app/main.py --workers 4` | Producción con 4 procesos. |

Para detener el servidor: <kbd>Ctrl</kbd> + <kbd>C</kbd>. Con la API levantada, la documentación está en `http://127.0.0.1:8000/docs`.

## Alembic: migraciones

Se explica en [Migraciones con Alembic](../base-de-datos/09-migraciones-que-son.md) y en [Primera migración](../base-de-datos/10-primera-migracion.md).

| Comando | Para qué |
| --- | --- |
| `uv run alembic revision --autogenerate -m "agregar vence_en"` | Generar una migración comparando los modelos con la base. |
| `uv run alembic upgrade head` | Aplicar todas las migraciones pendientes. |
| `uv run alembic downgrade -1` | Deshacer la última migración. |
| `uv run alembic current` | Ver en qué migración está la base. |
| `uv run alembic history` | Ver la lista de migraciones. |

!!! note "Revisá siempre la migración generada"

    Si no cambiaste ningún modelo, `--autogenerate` genera igual una migración, **vacía** (con `pass` en `upgrade` y `downgrade`). Si te pasa, borrá el archivo de `migraciones/versions/`. Y recordá que el mensaje de `-m` va al nombre del archivo: mejor sin tildes ni ñ.

## Tests y calidad

Se explican en [Tests con pytest](../calidad-y-despliegue/01-pytest-primer-test.md) y en [Ruff](../calidad-y-despliegue/03-ruff.md).

| Comando | Para qué |
| --- | --- |
| `uv run pytest` | Correr todos los tests. |
| `uv run pytest -v` | Lo mismo, con una línea por test. |
| `uv run pytest -k login` | Correr solo los tests con `login` en el nombre. |
| `uv run pytest -x` | Frenar en el primer test que falle. |
| `uv run ruff check` | Buscar errores con el linter. |
| `uv run ruff check --fix` | Arreglar solo lo que se puede arreglar sin riesgo. |
| `uv run ruff format` | Formatear todo el proyecto. |
| `uv run ruff format --check` | Ver qué archivos formatearía, sin tocarlos. |

## Docker

Se explica en el [Capítulo 4](../docker/01-contenedores-e-imagenes.md).

| Comando | Para qué |
| --- | --- |
| `docker --version` | Comprobar que Docker está instalado. |
| `docker run --name tareas-db -e POSTGRES_USER=tareas -e POSTGRES_PASSWORD=tareas -e POSTGRES_DB=tareas -p 5432:5432 -d postgres:17` | Crear y arrancar un contenedor de PostgreSQL en segundo plano. |
| `docker ps` | Ver los contenedores andando (`docker ps -a`: también los detenidos). |
| `docker logs tareas-db` | Ver lo que escribió un contenedor. |
| `docker exec -it tareas-db psql -U tareas` | Abrir `psql` dentro del contenedor (salís con `\q`). |
| `docker exec tareas-db psql -U tareas -c "SELECT 1;"` | Ejecutar una sola consulta. |
| `docker stop tareas-db` / `docker start tareas-db` | Detener y volver a arrancar un contenedor. |
| `docker rm -v tareas-db` | Borrar un contenedor detenido, con sus volúmenes anónimos. |
| `docker images` | Ver las imágenes descargadas o construidas. |
| `docker build -t tareas-api .` | Construir la imagen de la API con el `Dockerfile`. |
| `docker run --rm -p 8000:8000 --env-file .env tareas-api` | Ejecutar esa imagen (y borrar el contenedor al detenerlo). |
| `docker rmi tareas-api` | Borrar una imagen. |
| `docker volume ls` | Ver los volúmenes. |

!!! warning "`docker rm` sin `-v` deja un volumen huérfano"

    La imagen de PostgreSQL guarda sus datos en un volumen. Si no le pusiste nombre con `-v`, Docker crea uno **anónimo** (un nombre largo de letras y números), y `docker rm` solo no lo borra: se van acumulando. `docker rm -v` lo borra junto con el contenedor.

## Docker Compose

Se explica en [Docker Compose](../docker/04-docker-compose.md) y en [API y base juntas](../docker/08-api-y-base-juntas.md).

| Comando | Para qué |
| --- | --- |
| `docker compose up -d` | Levantar todos los servicios en segundo plano. |
| `docker compose up -d db` | Levantar solo uno. |
| `docker compose up -d --build` | Reconstruir la imagen de la API y levantar todo (después de cambiar código o dependencias). |
| `docker compose ps` | Ver el estado de los servicios. |
| `docker compose logs api` | Ver los logs de un servicio (con `-f`, los va mostrando en vivo; se corta con <kbd>Ctrl</kbd> + <kbd>C</kbd>). |
| `docker compose exec db psql -U tareas` | Abrir `psql` en el servicio `db`. |
| `docker compose exec api alembic upgrade head` | Aplicar las migraciones desde el contenedor de la API. |
| `docker compose restart api` | Reiniciar un servicio. |
| `docker compose stop` | Detener todo, sin borrar los contenedores. |
| `docker compose down` | Detener y borrar los contenedores (los volúmenes, con los datos, quedan). |
| `docker compose down -v` | Lo mismo, y **borra los volúmenes**: se pierden los datos de la base. |
| `docker compose config` | Ver el archivo final, con las variables ya reemplazadas. |
| `docker compose -f docker-compose.prod.yml up -d --build` | Usar otro archivo, como el [de producción](../calidad-y-despliegue/06-desplegar.md). |

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | Acá | En Node |
    | --- | --- |
    | `uv add` / `uv add --dev` | `npm install` / `npm install --save-dev` |
    | `uv sync` | `npm ci` |
    | `uv run fastapi dev` | `npm run dev` (con `nodemon` o `node --watch`) |
    | `uv run pytest` | `npm test` |
    | `uv run ruff check` | `npx eslint .` |

??? example "Ver comparación en Django ⬇️"

    | Acá | En Django |
    | --- | --- |
    | `uv run fastapi dev` | `python manage.py runserver` |
    | `alembic revision --autogenerate` | `python manage.py makemigrations` |
    | `alembic upgrade head` | `python manage.py migrate` |
    | `alembic current` / `history` | `python manage.py showmigrations` |
    | `uv run pytest` | `python manage.py test` |

??? example "Ver comparación en Spring Boot ⬇️"

    | Acá | En Spring Boot |
    | --- | --- |
    | `uv sync` | `./mvnw dependency:resolve` |
    | `uv run fastapi dev` | `./mvnw spring-boot:run` (con DevTools para la recarga) |
    | `uv run pytest` | `./mvnw test` |
    | `docker build -t tareas-api .` | `./mvnw spring-boot:build-image` (o un `Dockerfile` igual) |

    Las migraciones con Flyway o Liquibase se aplican solas al arrancar la app.

## Mini ejercicio

Cambiaste un modelo (agregaste una columna) y querés probar el cambio con todo andando en Compose. ¿Qué comandos corrés, y en qué orden, para generar la migración, aplicarla y que la API corra con el código nuevo?

??? success "Ver solución"

    ```bash
    uv run alembic revision --autogenerate -m "agregar prioridad a tareas"
    docker compose up -d --build
    docker compose exec api alembic upgrade head
    ```

    1. La migración se genera en tu máquina, porque el archivo nuevo tiene que quedar en `migraciones/versions/` (y en Git). Revisalo antes de seguir. Para generarla, la base de Compose tiene que estar andando: Alembic la compara con los modelos.
    2. `--build` reconstruye la imagen, que así incluye el modelo y la migración nuevos.
    3. La migración se aplica desde el contenedor de la API, que se conecta a la base por la red de Compose. Si usás el `command` con `alembic upgrade head` de [API y base juntas](../docker/08-api-y-base-juntas.md), este paso lo hace solo al arrancar.

---

**Siguiente:** [Errores comunes por capítulo →](03-errores-comunes.md)
