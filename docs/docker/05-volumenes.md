# Volúmenes: dónde se guardan los datos

!!! info "En esta página"

    **Qué vas a aprender:** por qué los datos de un contenedor se pierden al borrarlo, qué es un **volumen**, y cómo agregarle uno a PostgreSQL en tu `docker-compose.yml` para que los datos sobrevivan.

    **Qué necesitás:** [`docker-compose.yml`: qué es y para qué sirve](04-docker-compose.md).

## Los contenedores son descartables

Ya lo viste dos veces: con `docker rm` y con `docker compose down`, los datos de la base desaparecen. Es porque todo lo que un contenedor escribe queda **dentro del contenedor**, y borrar el contenedor borra eso también.

Y está bien que sea así: la idea de Docker es que un contenedor se pueda tirar y crear de nuevo en cualquier momento (para actualizar la imagen, cambiar la configuración, moverlo de máquina). Pero los **datos** no son descartables. Hace falta guardarlos en otro lado.

## Qué es un volumen

Un **volumen** es un espacio de almacenamiento que maneja Docker, **separado de cualquier contenedor**. Se "monta" en una carpeta del contenedor: el programa escribe en esa carpeta como siempre, pero los archivos en realidad quedan en el volumen.

```text
┌─────────────────────────────┐
│ Contenedor tareas-api-db-1  │      ┌──────────────────────┐
│                             │      │ Volumen              │
│ /var/lib/postgresql/data ───┼─────►│ tareas-api_datos_db  │
│                             │      └──────────────────────┘
└─────────────────────────────┘
   se puede borrar y recrear          sigue existiendo
```

Si borrás el contenedor y creás otro que monte el mismo volumen, el nuevo encuentra todos los datos del anterior.

## Agregar un volumen en Compose

Actualizá tu `docker-compose.yml`:

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
    volumes:
      - datos_db:/var/lib/postgresql/data

volumes:
  datos_db:
```

Son dos partes:

- **Dentro del servicio**, `volumes:` dice qué montar y dónde, con el formato `volumen:carpeta-adentro`. `/var/lib/postgresql/data` es la carpeta donde PostgreSQL guarda sus datos en esta imagen.
- **Al final del archivo**, `volumes:` declara el volumen `datos_db`, para que Compose lo cree.

La carpeta de adentro depende de cada imagen: la documentación de la imagen en Docker Hub dice cuál es. Si algún día cambiás a otra versión mayor de PostgreSQL, revisala, porque puede cambiar.

## Comprobarlo

Levantá la base (si ya estaba corriendo, Compose la recrea con el volumen):

```bash
docker compose up -d
```

```text
 Volume tareas-api_datos_db Created
 Container tareas-api-db-1 Started
```

Creá una tabla con una fila, **bajá todo con `down`** y volvé a levantar:

```bash
docker compose exec db psql -U tareas -c "CREATE TABLE notas (texto TEXT); INSERT INTO notas VALUES ('sobreviví al down');"
docker compose down
docker compose up -d
docker compose exec db psql -U tareas -c "SELECT * FROM notas;"
```

```text
       texto
-------------------
 sobreviví al down
(1 row)
```

El contenedor es nuevo, pero los datos siguen ahí: estaban en el volumen. El `down` borró el contenedor y la red, pero **no** el volumen. Lo podés ver con:

```bash
docker volume ls
```

```text
DRIVER    VOLUME NAME
local     tareas-api_datos_db
```

El nombre se arma con el proyecto y el nombre que le diste: `tareas-api_datos_db`.

## Borrar los datos a propósito

A veces querés empezar de cero (por ejemplo, para probar las migraciones desde una base vacía). Para borrar también los volúmenes:

```bash
docker compose down -v
```

La `-v` es de *volumes*. Al volver a levantar, la base arranca vacía. Usalo con cuidado: **no hay vuelta atrás**.

!!! warning "Las variables de PostgreSQL solo se usan la primera vez"

    `POSTGRES_USER`, `POSTGRES_PASSWORD` y `POSTGRES_DB` se aplican solo cuando la base se **inicializa**, es decir, cuando el volumen está vacío. Si después cambiás `POSTGRES_PASSWORD` en el archivo y hacés `up`, la contraseña **no cambia**: la vieja sigue funcionando y la nueva da `password authentication failed for user "tareas"`. Para que tome los valores nuevos, hay que empezar con un volumen vacío (`down -v`) o cambiar la contraseña desde `psql`.

## Volúmenes y carpetas compartidas

Hay otra forma de montar: en vez de un volumen con nombre, una **carpeta de tu computadora**, por ejemplo `./datos:/var/lib/postgresql/data`. Se llama *bind mount*. Es útil para compartir archivos entre tu computadora y el contenedor (como el código, mientras desarrollás), pero para los datos de una base conviene un volumen con nombre: lo maneja Docker, funciona igual en todos los sistemas y evita problemas de permisos.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    El volumen es el mismo sin importar el lenguaje de tu API. Un proyecto de Node con PostgreSQL en Compose usa exactamente estas líneas. Lo que sí vas a ver en proyectos de Node es un *bind mount* del código para desarrollar con recarga automática, la misma idea de la sección anterior.

??? example "Ver comparación en Django ⬇️"

    Igual: en el `docker-compose.yml` típico de Django, el servicio `db` tiene un volumen exactamente así, como `postgres_data:/var/lib/postgresql/data`. Sin él, cada `down` borraría la base y habría que volver a ejecutar las migraciones.

??? example "Ver comparación en Spring Boot ⬇️"

    Igual que en los demás casos. Si usás Testcontainers en los tests, ahí pasa lo contrario: los contenedores se crean **sin** volumen a propósito, para que cada ejecución de tests empiece con una base limpia.

## Mini ejercicio

1. Con el volumen configurado, creá una tabla de prueba, hacé `docker compose down` y comprobá que sigue estando.
2. Ahora hacé `docker compose down -v`, volvé a levantar y fijate qué pasa con la tabla. ¿Qué muestra `docker volume ls` justo después del `down -v`?

??? success "Ver solución"

    1. Después de `down` y `up`, `SELECT * FROM notas;` muestra la fila: el volumen conservó los datos.
    2. Después de `down -v` y `up`, `SELECT * FROM notas;` da `ERROR: relation "notas" does not exist`: la base se inicializó de nuevo sobre un volumen vacío. Justo después del `down -v`, `docker volume ls` ya no muestra `tareas-api_datos_db`; el `up` siguiente lo crea de nuevo (vas a ver `Volume tareas-api_datos_db Created`).

    Si tu API usa esta base, después de un `down -v` tenés que volver a aplicar las migraciones con `uv run alembic upgrade head`.

---

**Siguiente:** [Variables de entorno en Compose →](06-variables-entorno-compose.md)
