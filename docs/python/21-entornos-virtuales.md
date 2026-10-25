# Entornos virtuales y por qué existen

!!! info "En esta página"

    **Qué vas a aprender:** qué es un entorno virtual, por qué cada proyecto necesita el suyo, y cómo crear un proyecto e instalar librerías con `uv init` y `uv add`.

    **Qué necesitás:** [Decoradores](20-decoradores.md) y haber instalado `uv` ([Instalar y usar `uv`](02-instalar-uv.md)).

## El problema: librerías compartidas

Hasta ahora usaste solo la biblioteca estándar. Pero FastAPI, Pydantic o SQLAlchemy son librerías externas: hay que instalarlas.

Si las instalaras "en la computadora", todas tus aplicaciones compartirían las mismas versiones. Y eso se rompe enseguida:

- El proyecto A se escribió con FastAPI 0.100 y no anda con versiones más nuevas.
- El proyecto B necesita FastAPI 0.115.
- Actualizar para B rompe A.

Además, si le pasás el proyecto a otra persona, no hay forma de saber qué librerías (y qué versiones) necesita.

## La solución: un entorno por proyecto

Un **entorno virtual** es una carpeta, dentro del proyecto, con **su propio Python y sus propias librerías**, aisladas del resto. Por convención se llama `.venv`. Cada proyecto tiene la suya, con las versiones que necesita, y no se pisan.

Con `uv` casi no tenés que pensar en esto: crea y usa el entorno por vos.

## Crear un proyecto

```bash
uv init --no-package mi-api
cd mi-api
```

`uv init` crea la carpeta `mi-api` con esto adentro:

```text
mi-api/
├── .gitignore
├── .python-version   # qué versión de Python usa el proyecto
├── README.md
├── main.py           # un script de ejemplo
└── pyproject.toml    # la descripción del proyecto y sus dependencias
```

`--no-package` indica que es una aplicación simple, con los archivos en la raíz. Sin esa opción, las versiones recientes de `uv` arman una estructura de paquete con una carpeta `src/`, pensada para publicar librerías.

El archivo más importante es `pyproject.toml`:

```toml title="pyproject.toml"
[project]
name = "mi-api"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.12"
dependencies = []
```

`dependencies` es la lista de librerías que necesita el proyecto. Por ahora, ninguna.

## Ejecutar dentro del proyecto

```bash
uv run main.py
```

```text
Using CPython 3.12.15
Creating virtual environment at: .venv
Hello from mi-api!
```

La primera vez, `uv run` **crea el entorno virtual** `.venv`. A partir de ahí, cada `uv run` dentro de la carpeta usa ese entorno.

## Instalar librerías: `uv add`

```bash
uv add fastapi
```

Este comando:

1. Descarga FastAPI (y las librerías de las que depende) y las instala en `.venv`.
2. La agrega a `dependencies` en `pyproject.toml`:

    ```toml
    dependencies = [
        "fastapi>=0.162.0",
    ]
    ```

3. Crea o actualiza `uv.lock`, con la versión **exacta** de cada librería instalada.

(Los números de versión que veas van a ser otros: son los más nuevos al momento de instalar.)

Para comprobar el aislamiento: dentro del proyecto, `uv run python -c "import fastapi"` funciona. Fuera de él, la misma línea da `ModuleNotFoundError: No module named 'fastapi'`.

Para sacar una librería: `uv remove fastapi`.

## Qué se sube a Git y qué no

| Archivo | ¿Va a Git? | Por qué |
| --- | --- | --- |
| `pyproject.toml` | Sí | Dice qué librerías necesita el proyecto. |
| `uv.lock` | Sí | Fija las versiones exactas, para que todos instalen lo mismo. |
| `.python-version` | Sí | Dice qué Python usar. |
| `.venv/` | **No** | Se regenera; pesa mucho y depende de cada máquina. |

`uv init` ya agrega `.venv` al `.gitignore`. Cuando alguien clona el proyecto, ejecuta:

```bash
uv sync
```

y `uv` crea el `.venv` con exactamente las versiones de `uv.lock`. (`uv run` también lo hace solo, si el entorno falta.)

## Activar el entorno (opcional)

En otras guías vas a ver que hay que **activar** el entorno antes de trabajar. Con `uv run` no hace falta, pero si querés usar `python` directamente:

=== "macOS / Linux"

    ```bash
    source .venv/bin/activate
    ```

=== "Windows (PowerShell)"

    ```powershell
    .venv\Scripts\Activate.ps1
    ```

Mientras esté activado, `python` es el del proyecto. Para salir: `deactivate`.

!!! tip "Que VS Code use el entorno del proyecto"

    Para que el editor reconozca las librerías instaladas (y no subraye `import fastapi` en rojo), abrí la paleta de comandos (<kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>P</kbd>, o <kbd>Cmd</kbd> + <kbd>Shift</kbd> + <kbd>P</kbd> en macOS), buscá **Python: Select Interpreter** y elegí el que está en `.venv`. Normalmente lo detecta solo.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Node el aislamiento viene de fábrica: cada proyecto tiene su `node_modules`. El entorno virtual de Python cumple ese mismo rol.

    | Node | Python con `uv` |
    | --- | --- |
    | `npm init` | `uv init --no-package` |
    | `package.json` | `pyproject.toml` |
    | `package-lock.json` | `uv.lock` |
    | `node_modules/` | `.venv/` |
    | `npm install express` | `uv add fastapi` |
    | `npm install` (al clonar) | `uv sync` |
    | `npm run` / `npx` | `uv run` |

??? example "Ver comparación en Django ⬇️"

    En muchos proyectos de Django vas a ver el flujo clásico: `python -m venv .venv`, activar, `pip install django` y un `requirements.txt`. `uv` hace lo mismo con menos pasos y bastante más rápido: `uv add django` crea el entorno, instala y anota la dependencia. Y `uv.lock` es una versión más completa del `requirements.txt` con versiones fijas.

??? example "Ver comparación en Spring Boot ⬇️"

    En Java no hay entornos virtuales porque Maven y Gradle resuelven las dependencias por proyecto desde su archivo de configuración.

    | Spring Boot | Python con `uv` |
    | --- | --- |
    | `pom.xml` / `build.gradle` | `pyproject.toml` |
    | `<dependency>` | una línea en `dependencies` |
    | `mvn dependency:resolve` | `uv sync` |
    | Spring Initializr | `uv init --no-package` |

## Mini ejercicio

Creá un proyecto `practica`, agregale la librería `pydantic` y reemplazá `main.py` por este código. Ejecutalo con `uv run`.

```python title="main.py"
from pydantic import BaseModel


class Tarea(BaseModel):
    titulo: str
    hecha: bool = False


print(Tarea(titulo="Terminar el Capítulo 1"))
```

??? success "Ver solución"

    ```bash
    uv init --no-package practica
    cd practica
    uv add pydantic
    uv run main.py
    ```

    ```text
    titulo='Terminar el Capítulo 1' hecha=False
    ```

    Si en vez de `uv run main.py` probás con un Python que no es el del proyecto, vas a ver `ModuleNotFoundError: No module named 'pydantic'`: la librería está instalada solo en el `.venv` de `practica`.

---

¡Terminaste el Capítulo 1! Ya tenés todo lo de Python que hace falta para empezar con FastAPI.

**Siguiente:** [Capítulo 2 · Qué es FastAPI y qué es una API →](../fastapi/01-que-es-fastapi.md)
