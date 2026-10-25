# Variables de entorno en Compose

!!! info "En esta página"

    **Qué vas a aprender:** cómo sacar las contraseñas y la configuración del `docker-compose.yml` y leerlas del archivo `.env`, con valores por defecto y variables obligatorias.

    **Qué necesitás:** [Volúmenes: dónde se guardan los datos](05-volumenes.md) y [Configuración con `pydantic-settings`](../fastapi/12-pydantic-settings.md).

## El problema

Tu `docker-compose.yml` tiene la contraseña de la base escrita adentro:

```yaml
POSTGRES_PASSWORD: tareas
```

Y este archivo **sí** se sube a Git (tiene que estar en el repositorio para que todos levanten lo mismo). Es el mismo problema que resolviste en la API con el `.env`: los secretos y lo que cambia según la máquina no van en archivos versionados.

## `${VARIABLE}`: leer valores de afuera

Compose permite escribir `${NOMBRE}` en cualquier valor del archivo. Al ejecutarse, lo reemplaza por el valor de esa variable, que busca en dos lugares:

1. Las variables de entorno de la terminal.
2. Un archivo llamado **`.env`** en la misma carpeta que el `docker-compose.yml`.

Tu proyecto ya tiene un `.env` en la raíz, el de la API. Compose lee ese mismo archivo, así que la configuración queda toda en un solo lugar.

Agregá al `.env` las variables de la base:

```bash title=".env (agregar)"
POSTGRES_USER=tareas
POSTGRES_PASSWORD=tareas
POSTGRES_DB=tareas
```

Y en el `docker-compose.yml`, reemplazá los valores por referencias:

```yaml title="docker-compose.yml"
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "${PUERTO_DB:-5432}:5432"
    volumes:
      - datos_db:/var/lib/postgresql/data

volumes:
  datos_db:
```

Ahora el archivo ya no tiene ningún secreto, y se puede subir a Git tranquilo.

## Valores por defecto y variables obligatorias

Fijate en el puerto: `${PUERTO_DB:-5432}`. El `:-` significa "si `PUERTO_DB` no está definida (o está vacía), usá `5432`". Así, quien tenga el 5432 libre no necesita hacer nada, y quien lo tenga ocupado agrega `PUERTO_DB=5433` a su `.env`, sin tocar el archivo compartido.

| Forma | Si la variable no está |
| --- | --- |
| `${VARIABLE}` | Usa un texto vacío, con un aviso. |
| `${VARIABLE:-valor}` | Usa `valor`. |
| `${VARIABLE:?mensaje}` | **No arranca** y muestra `mensaje`. |

El primer caso es peligroso: si te olvidás de definir una variable, Compose solo avisa y sigue, con un valor vacío:

```text
level=warning msg="The \"POSTGRES_PASSWORD\" variable is not set. Defaulting to a blank string."
```

Para lo que es obligatorio, como la contraseña, usá `:?` y el error es claro:

```yaml
POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Falta POSTGRES_PASSWORD en el .env}
```

```text
required variable POSTGRES_PASSWORD is missing a value: Falta POSTGRES_PASSWORD en el .env
```

## Ver el resultado: `docker compose config`

Para ver cómo queda el archivo con las variables ya reemplazadas, sin levantar nada:

```bash
docker compose config
```

```yaml
    environment:
      POSTGRES_DB: tareas
      POSTGRES_PASSWORD: tareas
      POSTGRES_USER: tareas
```

Es la forma más rápida de detectar una variable mal escrita.

## Un ajuste en la API

Ahora el `.env` tiene variables que la API no usa (`POSTGRES_USER`, `PUERTO_DB`...). Por defecto, `pydantic-settings` **no lo permite**: si una variable del `.env` no corresponde a ningún campo, la app no arranca:

```text
postgres_db
  Extra inputs are not permitted [type=extra_forbidden, input_value='tareas', input_type=str]
```

Indicale que ignore las que no conoce, con `extra="ignore"` en `app/config.py`:

```python title="app/config.py"
    model_config = SettingsConfigDict(env_file=BASE_DIR / ".env", extra="ignore")
```

Y no te olvides de mantener coherente `URL_BASE_DATOS`: tiene que usar el mismo usuario, contraseña, base y puerto que definiste para Compose.

## `env_file`: pasar el `.env` entero al contenedor

Existe otra opción, `env_file`, que mete **todas** las variables de un archivo dentro del contenedor:

```yaml
services:
  db:
    image: postgres:17
    env_file: .env
```

Son cosas distintas: `${...}` **completa el `docker-compose.yml`** con valores; `env_file` **pasa variables al programa** que corre adentro. Para la base, conviene `${...}`: con `env_file`, PostgreSQL recibiría también `CLAVE_API`, `URL_BASE_DATOS` y todo lo demás, que no necesita. `env_file` va a ser útil para el contenedor de la API, que sí necesita esas variables.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Es la misma convención de `.env` que usás con `dotenv` en Node, y Compose la lee igual para cualquier proyecto. En un proyecto de Node, el mismo `.env` alimenta al `docker-compose.yml` y a `process.env`, como acá alimenta a Compose y a `pydantic-settings`.

??? example "Ver comparación en Django ⬇️"

    Igual: en el `docker-compose.yml` de un proyecto Django se ve `POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}`, y el mismo `.env` lo lee `django-environ` en `settings.py`.

??? example "Ver comparación en Spring Boot ⬇️"

    Spring lee las variables de entorno con nombres como `SPRING_DATASOURCE_PASSWORD`; en Compose se le pasan con `environment:` usando el mismo `${...}`. La sintaxis `${VARIABLE:-valor}` te va a resultar familiar: es parecida a `${propiedad:valor}` de los `application.properties`.

## Mini ejercicio

1. Hacé que la contraseña de la base sea obligatoria con `:?`.
2. Renombrá temporalmente tu `.env` (por ejemplo, a `.env.bak`) y ejecutá `docker compose config`. ¿Qué pasa?
3. Devolvele el nombre y confirmá con `docker compose config` que todo se completa bien.

??? success "Ver solución"

    1. En el `docker-compose.yml`:

        ```yaml
        POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Falta POSTGRES_PASSWORD en el .env}
        ```

    2. Sin `.env`, `docker compose config` falla con `required variable POSTGRES_PASSWORD is missing a value: Falta POSTGRES_PASSWORD en el .env`. Las otras dos variables (sin `:?`) solo muestran un aviso.
    3. Con el `.env` de vuelta, `docker compose config` muestra el archivo completo con `POSTGRES_PASSWORD: tareas`.

    Así, alguien que clona el proyecto y se olvida de crear su `.env` recibe un error claro en vez de una base con una contraseña vacía.

---

**Siguiente:** [Dockerfile para la propia API →](07-dockerfile.md)
