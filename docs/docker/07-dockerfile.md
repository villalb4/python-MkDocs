# Dockerfile para la propia API

!!! info "En esta página"

    **Qué vas a aprender:** cómo crear una imagen de Docker con tu propia API usando un `Dockerfile`, cómo construirla y ejecutarla, y por qué el orden de las instrucciones importa.

    **Qué necesitás:** [Variables de entorno en Compose](06-variables-entorno-compose.md) y [Servidor Uvicorn y recarga automática](../fastapi/03-uvicorn.md).

## Tu propia imagen

Hasta ahora usaste imágenes que hizo otra gente: `postgres:17`, `python:3.12-slim`. Para correr **tu API** en un contenedor, necesitás una imagen que tenga Python, las dependencias del proyecto y tu código. Esa imagen la armás vos, describiendo los pasos en un archivo llamado **`Dockerfile`**.

Es una receta: "partí de esta imagen, copiá estos archivos, ejecutá este comando". Docker sigue los pasos en orden y el resultado es una imagen nueva.

## El `Dockerfile`

En la raíz del proyecto `tareas-api` (al lado de `pyproject.toml`), creá un archivo llamado exactamente `Dockerfile`, sin extensión:

```dockerfile title="Dockerfile"
FROM python:3.12-slim

# Copia el ejecutable de uv desde su imagen oficial
COPY --from=ghcr.io/astral-sh/uv:0.13.1 /uv /bin/uv

WORKDIR /app

# Primero solo las dependencias, para aprovechar la caché de Docker
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# Después, el código
COPY . .

ENV PATH="/app/.venv/bin:$PATH"

EXPOSE 8000

CMD ["fastapi", "run", "app/main.py", "--port", "8000"]
```

Instrucción por instrucción:

| Instrucción | Qué hace |
| --- | --- |
| `FROM python:3.12-slim` | Parte de la imagen oficial de Python 3.12. Todo lo demás se agrega encima. |
| `COPY --from=... /uv /bin/uv` | Copia el programa `uv` desde su imagen oficial. Usá la misma versión que tenés instalada (`uv --version`). |
| `WORKDIR /app` | Crea la carpeta `/app` adentro de la imagen y trabaja ahí. |
| `COPY pyproject.toml uv.lock ./` | Copia los archivos de dependencias de tu proyecto a `/app`. |
| `RUN uv sync --frozen --no-dev` | Instala las dependencias en `/app/.venv`, exactamente las versiones de `uv.lock` (`--frozen`). |
| `COPY . .` | Copia el resto del proyecto (tu código) a `/app`. |
| `ENV PATH=...` | Agrega `/app/.venv/bin` a los programas disponibles, para poder usar `fastapi` directamente. |
| `EXPOSE 8000` | Documenta que la app escucha en el puerto 8000. |
| `CMD [...]` | El comando que se ejecuta al arrancar un contenedor de esta imagen. |

Fijate que el `CMD` usa `fastapi run`, no `fastapi dev`: es el modo producción que viste en el Capítulo 2, sin recarga automática y escuchando en `0.0.0.0`. Esto último es imprescindible: con `127.0.0.1`, la API solo aceptaría conexiones desde adentro del propio contenedor.

## Lo que no hay que copiar: `.dockerignore`

`COPY . .` copia **todo** lo que haya en la carpeta. Hay cosas que no deben entrar en la imagen: tu `.venv` (la imagen arma el suyo), los archivos temporales de Python y, sobre todo, el **`.env` con tus secretos**. Creá un archivo `.dockerignore` al lado del `Dockerfile`:

```text title=".dockerignore"
.venv
.env
.git
__pycache__
*.pyc
tareas.json
```

Funciona como el `.gitignore`, pero para Docker.

## Construir la imagen

```bash
docker build -t tareas-api .
```

- `-t tareas-api` le pone nombre (y etiqueta, `latest` por defecto) a la imagen.
- El `.` del final indica que el `Dockerfile` y los archivos a copiar están en la carpeta actual.

La primera vez tarda un poco: descarga las imágenes base e instala las dependencias. Al final vas a ver `naming to docker.io/library/tareas-api:latest`. Con `docker images tareas-api` podés ver que existe y cuánto ocupa (alrededor de 400 MB).

### Por qué el orden importa: la caché

Cada instrucción del `Dockerfile` genera una **capa**, y Docker guarda las capas en una caché. Al volver a construir, reusa todas las capas hasta la primera instrucción cuyo contenido cambió.

Por eso las dependencias se instalan **antes** de copiar el código. Si cambiás un archivo `.py` y volvés a construir, Docker reusa la capa de `uv sync` y solo repite el `COPY . .`:

```text
#10 [stage-0 4/6] COPY pyproject.toml uv.lock ./
#10 CACHED
#11 [stage-0 5/6] RUN uv sync --frozen --no-dev
#11 CACHED
#12 [stage-0 6/6] COPY . .
#12 DONE 0.0s
```

La reconstrucción tarda un segundo. Si copiaras todo el código primero, cualquier cambio invalidaría la caché y Docker reinstalaría todas las dependencias cada vez.

## Ejecutar la API en un contenedor

Con la base levantada por Compose (`docker compose up -d`), probá:

```bash
docker run --rm -p 8000:8000 --env-file .env tareas-api
```

`--env-file .env` le pasa al contenedor las variables de tu `.env`, como `CLAVE_API` y `URL_BASE_DATOS`. La API arranca (`Starting FastAPI in production mode`), pero si abrís `/salud` responde `500`, y en los logs aparece:

```text
ConnectionRefusedError: [Errno 111] Connection refused
```

¿Por qué? La URL dice `localhost`, y **dentro del contenedor, `localhost` es el propio contenedor**, no tu computadora. Ahí no hay ningún PostgreSQL. Es lo mismo que pasaba con Adminer.

Para esta prueba, Docker Desktop ofrece un nombre especial que apunta a tu computadora desde adentro de un contenedor, `host.docker.internal`:

```bash
docker run --rm -p 8000:8000 --env-file .env -e URL_BASE_DATOS=postgresql+asyncpg://tareas:tareas@host.docker.internal:5432/tareas tareas-api
```

`-e` pisa una variable del archivo. Ahora `/salud` responde `{"base_de_datos":"ok"}`: tu API corre en un contenedor y habla con la base. En la próxima página vas a resolver esto de forma prolija, con la API y la base en el mismo Compose.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    El `Dockerfile` de una API de Node sigue la misma estructura:

    ```dockerfile
    FROM node:22-slim
    WORKDIR /app
    COPY package.json package-lock.json ./
    RUN npm ci --omit=dev
    COPY . .
    CMD ["node", "app.js"]
    ```

    El truco de copiar primero los archivos de dependencias es idéntico: `package.json` + `npm ci` cumplen el rol de `pyproject.toml` + `uv sync`.

??? example "Ver comparación en Django ⬇️"

    Igual que para FastAPI, cambiando el `CMD`. En producción, Django se ejecuta con un servidor como Gunicorn (`CMD ["gunicorn", "proyecto.wsgi"]`) en vez de `runserver`, por el mismo motivo que acá se usa `fastapi run` y no `fastapi dev`.

??? example "Ver comparación en Spring Boot ⬇️"

    En Java normalmente se compila primero el `.jar` y la imagen solo lo ejecuta:

    ```dockerfile
    FROM eclipse-temurin:21-jre
    COPY target/app.jar /app.jar
    CMD ["java", "-jar", "/app.jar"]
    ```

    En Python no hay compilación: la imagen lleva el código fuente y las dependencias instaladas. Spring Boot también puede generar la imagen sin `Dockerfile`, con `./mvnw spring-boot:build-image`.

## Mini ejercicio

Construí la imagen con la etiqueta `tareas-api:0.1.0` y ejecutala en el puerto `9000` de tu computadora. Comprobá que `http://localhost:9000/docs` abre la documentación.

??? success "Ver solución"

    ```bash
    docker build -t tareas-api:0.1.0 .
    docker run --rm -p 9000:8000 --env-file .env -e URL_BASE_DATOS=postgresql+asyncpg://tareas:tareas@host.docker.internal:5432/tareas tareas-api:0.1.0
    ```

    Adentro, la API sigue escuchando en el `8000` (es lo que dice el `CMD`); `-p 9000:8000` publica ese puerto como `9000` en tu computadora. Como la segunda construcción reusa todas las capas de la caché, es casi instantánea. Con `docker images tareas-api` vas a ver las dos etiquetas, `latest` y `0.1.0`. Ponerles versiones a las imágenes sirve para saber exactamente qué código está corriendo y poder volver a una versión anterior.

---

**Siguiente:** [Levantar API y base juntas →](08-api-y-base-juntas.md)
