# Qué es un contenedor y qué es una imagen

!!! info "En esta página"

    **Qué vas a aprender:** qué problema resuelve Docker, la diferencia entre una **imagen** y un **contenedor**, y los primeros comandos para usarlos.

    **Qué necesitás:** haber terminado el [Capítulo 3](../base-de-datos/10-primera-migracion.md). Ya usaste Docker para levantar PostgreSQL; ahora vas a entender qué pasaba.

## "En mi máquina funciona"

Tu API necesita varias cosas para correr: una versión de Python, las librerías del proyecto, PostgreSQL, variables de entorno. En tu computadora ya está todo instalado y configurado. Pero en la de otra persona, o en el servidor donde la vas a publicar, puede faltar algo o haber otra versión. El resultado es la frase más famosa de la programación: *"en mi máquina funciona"*.

**Docker** resuelve esto empaquetando un programa **junto con todo lo que necesita** (el sistema base, el lenguaje, las librerías, la configuración) en un paquete que corre igual en cualquier computadora que tenga Docker.

## Imagen y contenedor

Son las dos ideas centrales:

- Una **imagen** es el **molde**: un paquete de solo lectura con todo lo necesario para correr algo. Por ejemplo, `postgres:17` trae PostgreSQL 17 listo para usar, y `python:3.12-slim`, un sistema mínimo con Python 3.12.
- Un **contenedor** es una **instancia en ejecución** de una imagen: el programa corriendo, aislado del resto de tu computadora.

La relación es la misma que entre una **clase** y un **objeto** del Capítulo 1: de una imagen podés crear muchos contenedores, cada uno con su propio estado. El contenedor `tareas-db` que usaste en el Capítulo 3 era un contenedor creado a partir de la imagen `postgres:17`.

### De dónde salen las imágenes

Las imágenes se descargan de un **registro**. El público más usado es **Docker Hub** (`hub.docker.com`), donde hay imágenes oficiales de PostgreSQL, Python, Redis y miles más.

El nombre de una imagen tiene dos partes, separadas por dos puntos:

```text
postgres:17
───┬──── ┬─
   │     └── etiqueta (tag): qué versión
   └── nombre de la imagen
```

Si no ponés etiqueta, Docker usa `latest` ("la última"). Conviene indicarla siempre, para que la versión no cambie sin aviso.

## Tu primer contenedor

Con Docker Desktop abierto, ejecutá:

```bash
docker run --rm hello-world
```

La primera vez, Docker no tiene la imagen, así que la descarga y después crea el contenedor:

```text
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
...
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

`docker run` hace tres cosas: descarga la imagen si no la tenés, crea un contenedor a partir de ella y lo ejecuta. `--rm` borra el contenedor cuando termina.

## Python sin instalar Python

Ahora algo más útil. Este comando corre un Python 3.12 que **no está instalado en tu computadora**: vive dentro del contenedor.

```bash
docker run --rm python:3.12-slim python -c "print('Hola desde un contenedor')"
```

```text
Hola desde un contenedor
```

Lo que va después del nombre de la imagen es el comando que se ejecuta adentro. Y cambiar de versión es cambiar la etiqueta: `docker run --rm python:3.13-slim python --version` muestra `Python 3.13.17`, sin tocar nada de tu sistema.

## El contenedor está aislado

Un contenedor tiene su propio sistema de archivos, sus propios programas y su propia red. Si mirás adentro, vas a ver un Linux mínimo, aunque estés en macOS o Windows:

```bash
docker run --rm python:3.12-slim cat /etc/os-release
```

```text
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
...
```

Lo que pasa adentro no afecta a tu computadora, y al revés. Por eso hay que indicarle a Docker explícitamente qué cosas conectar con el exterior, como el puerto `-p 5432:5432` que usaste para PostgreSQL.

!!! note "No es una máquina virtual"

    Una máquina virtual simula una computadora entera, con su propio sistema operativo completo: es pesada y tarda en arrancar. Un contenedor comparte el núcleo del sistema y solo aísla el programa: arranca en un segundo y ocupa mucho menos. En macOS y Windows, Docker Desktop usa una pequeña máquina virtual con Linux por debajo, pero vos no la ves.

## Comandos básicos

Sin `--rm`, el contenedor queda guardado (detenido) cuando termina:

```bash
docker run --name mi-primer-contenedor python:3.12-slim python -c "print('hola')"
docker ps -a
```

```text
NAMES                  IMAGE              STATUS
mi-primer-contenedor   python:3.12-slim   Exited (0) Less than a second ago
```

| Comando | Hace |
| --- | --- |
| `docker run imagen` | Crea y ejecuta un contenedor. |
| `docker ps` | Lista los contenedores **en ejecución**. |
| `docker ps -a` | Lista **todos**, también los detenidos. |
| `docker images` | Lista las imágenes descargadas y cuánto ocupan. |
| `docker rm nombre` | Borra un contenedor detenido. |
| `docker rmi imagen` | Borra una imagen. |

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Docker no depende del lenguaje: un proyecto de Node se empaqueta igual, partiendo de una imagen como `node:22-slim` en vez de `python:3.12-slim`. Si ya usaste `nvm` para tener varias versiones de Node, Docker lleva esa idea más lejos: cada proyecto puede tener su propio sistema completo, no solo su versión del lenguaje.

??? example "Ver comparación en Django ⬇️"

    Un proyecto de Django se dockeriza exactamente igual que uno de FastAPI: también parte de una imagen de Python y suele levantar PostgreSQL en otro contenedor. Todo lo que veas en este capítulo te sirve tal cual para Django.

??? example "Ver comparación en Spring Boot ⬇️"

    En Java la promesa histórica fue "escribí una vez, ejecutá en cualquier lado" gracias a la JVM. Docker va un paso más allá: empaqueta también la JVM, la configuración y el sistema base. Un `.jar` de Spring Boot se suele correr en un contenedor creado a partir de una imagen como `eclipse-temurin:21-jre`.

## Mini ejercicio

1. Mirá qué imágenes tenés descargadas.
2. Ejecutá el contenedor `mi-primer-contenedor` de arriba, comprobá con `docker ps -a` que quedó detenido y borralo.
3. ¿Por qué el contenedor no aparecía en `docker ps` (sin `-a`)?

??? success "Ver solución"

    ```bash
    docker images
    docker run --name mi-primer-contenedor python:3.12-slim python -c "print('hola')"
    docker ps -a
    docker rm mi-primer-contenedor
    ```

    `docker images` muestra, entre otras, `python 3.12-slim`, `postgres 17` y `hello-world latest`, con su tamaño. Después de `docker rm`, el contenedor ya no aparece en `docker ps -a`.

    No aparecía en `docker ps` porque ese comando solo muestra los contenedores **en ejecución**. Este ejecutó un `print` y terminó enseguida: quedó con estado `Exited (0)`, que significa "terminó sin errores".

---

**Siguiente:** [Instalar Docker Desktop y comprobar →](02-instalar-docker.md)
