# Levantar API y base juntas

!!! info "En esta página"

    **Qué vas a aprender:** cómo agregar tu API como un servicio más del `docker-compose.yml`, para que la API y la base se levanten juntas con un solo comando, se encuentren por nombre y arranquen en el orden correcto.

    **Qué necesitás:** [Dockerfile para la propia API](07-dockerfile.md).

## El objetivo

En la página anterior, tu API corrió en un contenedor, pero tuviste que levantar la base por un lado, la API por otro y usar `host.docker.internal` para conectarlas. Ahora vas a describir las dos en el mismo `docker-compose.yml`, de forma que alguien que clona tu proyecto solo tenga que crear su `.env` y ejecutar un comando.

## El `docker-compose.yml` completo

```yaml title="docker-compose.yml"
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Falta POSTGRES_PASSWORD en el .env}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "${PUERTO_DB:-5432}:5432"
    volumes:
      - datos_db:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 2s
      timeout: 5s
      retries: 10

  api:
    build: .
    env_file: .env
    environment:
      URL_BASE_DATOS: postgresql+asyncpg://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy

volumes:
  datos_db:
```

Lo nuevo, parte por parte.

### El servicio `api`

- **`build: .`** en vez de `image:`. Le dice a Compose que construya la imagen con el `Dockerfile` de la carpeta actual.
- **`env_file: .env`** le pasa a la API todas las variables de tu `.env` (`CLAVE_API`, `DEBUG`, `ORIGENES_PERMITIDOS`...). Es el caso que se mencionó en la página de variables: acá sí conviene, porque la API las necesita.
- **`environment:`** define `URL_BASE_DATOS` y **pisa** la que viene del `.env`. La diferencia está en el servidor: en vez de `localhost`, dice **`db`**, el nombre del servicio. Compose conecta los servicios en una misma red donde se encuentran por nombre, como viste con Adminer. Y como se arma con `${...}`, el usuario y la contraseña salen de las mismas variables que usa PostgreSQL.

Fijate que la URL usa el puerto **`5432`**, aunque hayas publicado la base en otro (`PUERTO_DB`): entre contenedores se usa el puerto de adentro. El de afuera es solo para conectarte desde tu computadora.

### Orden de arranque: `healthcheck` y `depends_on`

Si la API arranca antes de que PostgreSQL esté listo, el primer pedido falla. Para evitarlo:

- **`healthcheck`** en `db` le enseña a Compose a saber cuándo la base está **lista**: ejecuta `pg_isready` cada 2 segundos, hasta 10 veces.
- **`depends_on`** con `condition: service_healthy` en `api` dice "no arranques la API hasta que `db` esté sana".

## Levantar todo

Si te quedó corriendo la base de páginas anteriores, no hace falta bajarla. Desde la raíz del proyecto:

```bash
docker compose up -d --build
```

`--build` construye la imagen de la API antes de arrancar (la primera vez lo haría igual, pero es una buena costumbre, para que siempre use tu código actual):

```text
 Image tareas-api-api Built
 Container tareas-api-db-1 Started
 Container tareas-api-db-1 Healthy
 Container tareas-api-api-1 Started
```

Fijate el orden: la API arranca recién cuando la base está `Healthy`. Comprobá los dos servicios con `docker compose ps`, y abrí `http://localhost:8000/salud`: responde `{"base_de_datos":"ok"}`.

## Aplicar las migraciones

Si la base es nueva (por ejemplo, después de un `down -v`), está vacía, y `/tareas` responde `500` (`relation "tareas" does not exist`). Las migraciones se aplican **dentro del contenedor de la API**, que tiene Alembic, las migraciones y la URL correcta:

```bash
docker compose exec api alembic upgrade head
```

```text
INFO  [alembic.runtime.migration] Running upgrade  -> d96edc695e3a, crear tablas usuarios y tareas
```

Ahora sí: creá un usuario y una tarea desde `http://localhost:8000/docs`. Todo corre en Docker, y con el volumen los datos sobreviven a un `docker compose down`.

## El día a día

| Querés... | Comando |
| --- | --- |
| Levantar todo | `docker compose up -d` |
| Ver los logs de la API en vivo | `docker compose logs -f api` |
| Aplicar migraciones | `docker compose exec api alembic upgrade head` |
| Probar un cambio en el código | `docker compose up -d --build` |
| Apagar todo (conservando los datos) | `docker compose down` |

!!! tip "¿Y para programar?"

    La imagen de la API no tiene recarga automática: cada cambio en el código requiere `--build`. Para programar, lo más cómodo suele ser levantar **solo la base** con Compose (`docker compose up -d db`) y correr la API en tu computadora con `uv run fastapi dev app/main.py`, como venías haciendo. La configuración completa en Docker sirve para comprobar que todo funciona junto y para que cualquiera levante el proyecto sin instalar nada más que Docker.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    El `docker-compose.yml` de una API de Node con PostgreSQL es prácticamente igual: un servicio `db`, un servicio `api` con `build: .`, la URL apuntando a `db`, y `depends_on` con `healthcheck`. Lo único que cambia es el `Dockerfile` y el comando de las migraciones (por ejemplo, `npx prisma migrate deploy`).

??? example "Ver comparación en Django ⬇️"

    Es el clásico Compose de Django con los servicios `db` y `web`. Las migraciones se aplican de la misma forma: `docker compose exec web python manage.py migrate`.

??? example "Ver comparación en Spring Boot ⬇️"

    Con Spring sería un servicio `app` con la imagen del `.jar`, y `SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/tareas` en `environment`. Si usás Flyway, las migraciones corren solas al arrancar la app, que es justamente lo que vas a hacer en el ejercicio.

## Mini ejercicio

Hacé que la API aplique las migraciones **sola** cada vez que arranca, agregando una línea `command:` al servicio `api`. Probalo con una base vacía (`docker compose down -v` y después `up -d`) y comprobá en los logs que se aplicaron.

??? success "Ver solución"

    ```yaml title="docker-compose.yml (servicio api)"
      api:
        build: .
        command: sh -c "alembic upgrade head && fastapi run app/main.py --port 8000"
    ```

    `command:` reemplaza al `CMD` del `Dockerfile`. `sh -c "A && B"` ejecuta `A` y, solo si salió bien, `B`. Después de `docker compose down -v` y `docker compose up -d`, `docker compose logs api` muestra:

    ```text
    api-1  | INFO  [alembic.runtime.migration] Running upgrade  -> d96edc695e3a, crear tablas usuarios y tareas
    api-1  | INFO:     Application startup complete.
    ```

    En los arranques siguientes, Alembic ve que la base ya está al día y no hace nada. Es cómodo, pero tiene un riesgo: si una migración falla, la API no arranca. En proyectos grandes se suele migrar como un paso aparte del despliegue.

---

¡Terminaste el Capítulo 4! Tu proyecto entero (API y base) se levanta con un solo comando. En el próximo capítulo vas a agregar usuarios de verdad, con contraseñas y tokens.

**Siguiente:** [Capítulo 5 · Qué es autenticar y qué es autorizar →](../autenticacion/01-autenticar-y-autorizar.md)
