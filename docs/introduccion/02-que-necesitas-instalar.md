# Qué necesitás instalar

!!! info "En esta página"

    **Qué vas a aprender:** qué programas tener instalados antes de empezar y cómo comprobar que funcionan.

    **Qué necesitás:** [Qué vas a aprender y en qué orden](01-que-vas-a-aprender.md).

!!! note "¿Y Python?"

    No lo instales a mano. En el Capítulo 1 vas a usar `uv`, que se encarga de descargar la versión de Python que haga falta.

## 1. Un editor de código

Recomendamos **Visual Studio Code** (gratis): [code.visualstudio.com](https://code.visualstudio.com/).

Una vez instalado, agregale la extensión **Python** (de Microsoft) desde el panel de extensiones. Te da colores, autocompletado y avisos de errores.

## 2. Una terminal

Ya la tenés:

=== "macOS / Linux"

    Usá la app **Terminal**. También podés abrir una adentro de VS Code con el atajo <kbd>Ctrl</kbd> + <kbd>`</kbd>.

=== "Windows (PowerShell)"

    Usá **PowerShell** o, mejor, **Windows Terminal** (viene en Windows 11). También podés abrir una adentro de VS Code con el atajo <kbd>Ctrl</kbd> + <kbd>`</kbd>.

## 3. Git

Git guarda el historial de cambios de tu código.

=== "macOS / Linux"

    En macOS, Git viene con las herramientas de desarrollo de Xcode. Si no lo tenés, al escribir `git` en la terminal el sistema te ofrece instalarlo.

    En Linux, instalalo con el gestor de paquetes de tu distribución (por ejemplo `sudo apt install git`).

=== "Windows (PowerShell)"

    Descargalo de [git-scm.com](https://git-scm.com/download/win) y seguí el instalador con las opciones por defecto.

## 4. Docker Desktop

Docker lo vas a usar recién en el Capítulo 4, pero conviene dejarlo instalado. Descargalo de [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/) y abrilo una vez para que termine de configurarse.

## Comprobar que todo funciona

Abrí una terminal nueva y ejecutá:

```bash
git --version
docker --version
```

Cada comando tiene que mostrar un número de versión, algo como `git version 2.47.0`. Si alguno dice "command not found" o "no se reconoce como comando", cerrá la terminal, abrí una nueva y probá otra vez. Si sigue fallando, reinstalá ese programa.

## Mini ejercicio

Ejecutá `code --version` en la terminal. ¿Qué pasa?

??? success "Ver solución"

    Si VS Code está bien instalado, muestra su número de versión. Eso significa que podés abrir cualquier carpeta en el editor desde la terminal con `code .` (el punto es "la carpeta en la que estoy").

    Si en macOS dice "command not found", abrí VS Code, presioná <kbd>Cmd</kbd> + <kbd>Shift</kbd> + <kbd>P</kbd>, buscá **Shell Command: Install 'code' command in PATH** y ejecutalo.

---

**Siguiente:** [Cómo leer las comparaciones →](03-como-leer-las-comparaciones.md)
