# Instalar y usar `uv`

!!! info "En esta página"

    **Qué vas a aprender:** qué es `uv`, cómo instalarlo y cómo ejecutar scripts de Python con él, sin instalar Python a mano.

    **Qué necesitás:** [Qué es Python y cómo se ejecuta un script](01-que-es-python.md).

## Qué es `uv`

`uv` es una herramienta que se encarga de todo lo que rodea a Python:

- **Descarga el intérprete** de Python que haga falta, sin instaladores.
- **Ejecuta tus scripts** con ese intérprete.
- Más adelante, **instala las librerías** de cada proyecto (como FastAPI).

La ventaja es que te olvidás de instalar Python, de qué versión tiene tu sistema y de configurar el `PATH`. Instalás `uv` una vez y él resuelve el resto.

## Instalarlo

Abrí una terminal y ejecutá el comando de tu sistema:

=== "macOS / Linux"

    ```bash
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```

    Si usás Homebrew, también podés hacer `brew install uv`.

=== "Windows (PowerShell)"

    ```powershell
    powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
    ```

    Si usás winget, también podés hacer `winget install --id=astral-sh.uv -e`.

Cuando termine, **cerrá la terminal y abrí una nueva** (así toma el `PATH` actualizado) y comprobá:

```bash
uv --version
```

Tiene que mostrar algo como `uv 0.13.1`. El número exacto no importa.

## Ejecutar un script

Creá una carpeta para practicar, abrila en VS Code (`code .`) y creá el archivo `hola.py` de la página anterior:

```python title="hola.py"
print("Hola, mundo")
print("Esto es Python")
```

Desde la terminal, parado en esa carpeta:

```bash
uv run hola.py
```

La primera vez, si no tenés Python, `uv` lo descarga antes de ejecutar (vas a ver un renglón que dice `Downloading cpython-...`). Después muestra:

```text
Hola, mundo
Esto es Python
```

Las siguientes veces ya no descarga nada: el intérprete queda guardado.

## Ver qué Python estás usando

```bash
uv run python --version
```

Muestra algo como `Python 3.13.16`. Esta guía funciona con Python 3.12 o más nuevo.

Si necesitás una versión puntual, se la pedís con `--python`, y `uv` la descarga si no la tenés:

```bash
uv run --python 3.12 hola.py
```

## Probar código suelto: la consola interactiva

A veces querés probar una línea sin crear un archivo. Para eso está la **consola interactiva** (también llamada REPL):

```bash
uv run python
```

Aparece el símbolo `>>>`. Escribí código, apretá <kbd>Enter</kbd> y ves el resultado al instante:

```pycon
>>> print("hola")
hola
>>> 2 + 3
5
```

Fijate que en la consola no hace falta `print` para ver el resultado de una cuenta. Para salir, escribí `exit()` y apretá <kbd>Enter</kbd>.

!!! tip "Consola o script"

    La consola sirve para probar cosas rápidas. Todo lo que quieras guardar o volver a ejecutar va en un archivo `.py`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    `uv` junta en una sola herramienta lo que en Node hacen varias: un gestor de versiones como `nvm` (para elegir qué Node usar), `node` (para ejecutar) y `npm` (para instalar paquetes, que vas a ver más adelante).

    | Node | Python con `uv` |
    | --- | --- |
    | `nvm install 22` | `uv python install 3.13` |
    | `node hola.js` | `uv run hola.py` |
    | `node` (consola) | `uv run python` |

??? example "Ver comparación en Django ⬇️"

    En muchos tutoriales de Django vas a ver Python instalado a mano, `pip` y `venv`. `uv` reemplaza todo eso con un solo programa, mucho más rápido. Un proyecto de Django se puede manejar perfectamente con `uv`.

??? example "Ver comparación en Spring Boot ⬇️"

    Es parecido a usar SDKMAN! para instalar el JDK y Maven o Gradle para el proyecto: `uv` cumple los dos roles. Elige e instala la versión del lenguaje y, más adelante, maneja las dependencias del proyecto.

## Mini ejercicio

Abrí la consola interactiva y averiguá qué muestra `"hola" * 3`. Después salí de la consola.

??? success "Ver solución"

    ```bash
    uv run python
    ```

    ```pycon
    >>> "hola" * 3
    'holaholahola'
    >>> exit()
    ```

    Multiplicar un texto por un número lo repite. Las comillas simples que aparecen alrededor son la forma en que la consola te avisa que el resultado es un texto.

---

**Siguiente:** [Variables y tipos básicos →](03-variables-y-tipos.md)
