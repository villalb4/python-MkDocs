# Instalar Docker Desktop y comprobar

!!! info "En esta página"

    **Qué vas a aprender:** cómo instalar Docker en tu sistema, cómo comprobar que funciona (incluido Docker Compose, que vas a usar en las próximas páginas) y cómo resolver los problemas más comunes.

    **Qué necesitás:** [Qué es un contenedor y qué es una imagen](01-contenedores-e-imagenes.md).

!!! note "¿Ya lo instalaste?"

    Si seguiste la Introducción y ya usaste `docker run` en el Capítulo 3, Docker ya está funcionando. Saltá directo a [Comprobar que todo funciona](#comprobar-que-todo-funciona) para verificar que también tenés Docker Compose.

## Docker Desktop y Docker Engine

Docker tiene dos partes:

- El **motor** (*Docker Engine*): el programa que realmente crea y ejecuta los contenedores. Corre en segundo plano.
- El **cliente**: el comando `docker` que escribís en la terminal, que le pide cosas al motor.

**Docker Desktop** es una aplicación que instala las dos partes, más Docker Compose y una ventana para ver tus contenedores e imágenes. Es la forma recomendada en macOS y Windows. Es gratis para uso personal, educación y empresas chicas; las empresas grandes necesitan una suscripción paga.

## Instalarlo

Descargalo desde **<https://www.docker.com/products/docker-desktop/>**.

=== "macOS / Linux"

    **macOS:** elegí la versión según tu procesador: *Apple Silicon* (M1, M2, M3, M4...) o *Intel*. Si no sabés cuál tenés, mirá en el menú de la manzana → **Acerca de esta Mac**. Abrí el `.dmg` descargado y arrastrá Docker a **Aplicaciones**.

    **Linux:** podés instalar Docker Desktop para Linux, o solo el motor con el gestor de paquetes de tu distribución, siguiendo la guía oficial de Docker Engine para tu sistema (Ubuntu, Debian, Fedora...). Con el motor solo no hay ventana, pero todos los comandos de esta guía funcionan igual.

=== "Windows (PowerShell)"

    Docker Desktop en Windows usa **WSL 2** (el subsistema de Linux de Windows) para correr los contenedores. El instalador lo activa solo; si te avisa que falta, abrí PowerShell **como administrador** y ejecutá:

    ```powershell
    wsl --install
    ```

    Reiniciá la computadora y volvé a ejecutar el instalador de Docker Desktop.

Al terminar, **abrí Docker Desktop**. La primera vez te pide aceptar los términos y puede ofrecerte crear una cuenta: no hace falta para usarlo. Esperá a que el ícono de la ballena indique que está corriendo (*Engine running*).

## Comprobar que todo funciona

En una terminal nueva:

```bash
docker --version
docker compose version
```

```text
Docker version 29.8.2, build 7fc2dff
Docker Compose version v5.6.0
```

Los números pueden ser otros: lo importante es que los dos comandos respondan. Fijate que es `docker compose`, con un espacio: es un subcomando de `docker`. En tutoriales viejos vas a ver `docker-compose`, con guion, que era un programa aparte y ya no se usa.

Después, la prueba completa:

```bash
docker run --rm hello-world
```

Si ves `Hello from Docker!`, el motor descargó una imagen y ejecutó un contenedor: está todo listo.

Para ver más detalles del motor:

```bash
docker info
```

Entre muchos datos, muestra `Operating System: Docker Desktop`, `OSType: linux` (los contenedores corren en Linux aunque estés en macOS o Windows) y la cantidad de CPUs y memoria que Docker tiene asignadas.

## Problemas comunes

**"failed to connect to the docker API" o "Cannot connect to the Docker daemon".** El cliente funciona, pero el motor no está corriendo. Casi siempre es porque **Docker Desktop está cerrado**: abrilo y esperá a que arranque.

```text
failed to connect to the docker API at unix:///.../docker.sock; check if the path is correct and if the daemon is running
```

**`docker: command not found` (o "no se reconoce como comando").** La terminal se abrió antes de instalar Docker. Cerrala y abrí una nueva. Si sigue, reiniciá la computadora.

**`port is already allocated`.** Otro programa (o otro contenedor) ya usa ese puerto. Lo viste en la página de PostgreSQL: cambiá el puerto de tu lado, por ejemplo `-p 5433:5432`.

**Todo anda muy lento o se queda sin memoria.** En Docker Desktop, en **Settings → Resources**, podés ajustar cuánta memoria y CPU puede usar. Para esta guía alcanza con lo que viene por defecto.

!!! tip "La ventana de Docker Desktop"

    En la sección **Containers** ves todos tus contenedores, con botones para iniciarlos, detenerlos, ver sus logs y borrarlos. En **Images**, las imágenes descargadas y cuánto ocupan. Es lo mismo que hacés con `docker ps` y `docker images`, pero con clics.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    La instalación es la misma sin importar el lenguaje de tu proyecto: Docker Desktop sirve igual para Node, Python o Java. Si trabajás con Node, vas a usar exactamente los mismos comandos `docker` y `docker compose`.

??? example "Ver comparación en Django ⬇️"

    Igual que para Express: Docker no sabe ni le importa si adentro corre Django o FastAPI. Muchos proyectos de Django incluyen un `docker-compose.yml` en el repositorio, y para usarlos alcanza con esta instalación.

??? example "Ver comparación en Spring Boot ⬇️"

    Spring Boot tiene integración directa con Docker: desde la versión 3.1 puede levantar los servicios de un `compose.yaml` al arrancar la app, y con `./mvnw spring-boot:build-image` genera una imagen. Las dos cosas necesitan este mismo Docker instalado y corriendo.

## Mini ejercicio

1. Comprobá qué versiones de Docker y de Docker Compose tenés.
2. Con `docker info`, averiguá cuántas CPUs puede usar Docker.
3. Cerrá Docker Desktop, ejecutá `docker ps` y mirá el error. Volvé a abrirlo y repetí el comando.

??? success "Ver solución"

    1. `docker --version` y `docker compose version`.
    2. En la salida de `docker info`, el renglón `CPUs:` dice cuántas tiene asignadas.
    3. Con Docker Desktop cerrado, `docker ps` falla con un error de conexión (`failed to connect to the docker API` o `Cannot connect to the Docker daemon`, según la versión). Con Docker Desktop abierto otra vez, responde con la lista de contenedores (vacía, si no tenés ninguno corriendo). Así reconocés ese error cuando aparezca.

---

**Siguiente:** [Levantar PostgreSQL con `docker run` →](03-postgres-docker-run.md)
