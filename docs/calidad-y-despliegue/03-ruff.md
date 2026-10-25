# Linter y formateo con Ruff

!!! info "En esta página"

    **Qué vas a aprender:** qué es un linter y qué es un formateador, cómo usar Ruff para encontrar errores antes de correr el código, cómo dejar que arregle lo que puede solo, y cómo configurarlo en `pyproject.toml`.

    **Qué necesitás:** [Tests de endpoints con `httpx`](02-tests-de-endpoints.md).

## Linter y formateador

Los tests comprueban que el código hace lo que tiene que hacer. Pero hay errores que se pueden encontrar **sin ejecutar nada**, solo leyendo el código: un `import` que no se usa, una variable mal escrita, una trampa conocida del lenguaje. Eso es lo que hace un **linter**.

Un **formateador** se ocupa de otra cosa: de la forma. Sangrías, espacios, comillas, dónde se corta una línea larga. No cambia lo que hace el código, solo cómo se ve, y así todo el proyecto queda con el mismo estilo sin discutirlo.

**Ruff** hace las dos cosas, y muy rápido. Se instala como dependencia de desarrollo, igual que pytest:

```bash
uv add --dev ruff
```

## El linter: `ruff check`

Para ver cómo trabaja, creá un archivo `ejemplo.py` en la carpeta del proyecto, con varios errores típicos:

```python title="ejemplo.py"
import os


def agregar_etiqueta(etiqueta: str, etiquetas: list[str] = []) -> list[str]:
    cantidad = len(etiquetas)
    etiquetas.append(etiqueta)
    return etiqutas
```

Y revisalo con Ruff (`--output-format concise` muestra un renglón por problema):

```bash
uv run ruff check ejemplo.py --output-format concise
```

```text
ejemplo.py:1:8: F401 [*] `os` imported but unused
ejemplo.py:4:60: B006 Do not use mutable data structures for argument defaults
ejemplo.py:5:5: F841 Local variable `cantidad` is assigned to but never used
ejemplo.py:7:12: F821 Undefined name `etiqutas`
Found 4 errors.
[*] 1 fixable with `--fix` (2 unsafe fixes available with `--fix --unsafe-fixes`).
```

Cada renglón dice el archivo, la línea y la columna, un **código de regla** y qué encontró:

| Código | Qué encontró |
| --- | --- |
| `F401` | `os` se importa, pero no se usa. |
| `B006` | Una lista como valor por defecto. Es una trampa clásica: la lista se crea **una sola vez** y se comparte entre llamadas. Llamar dos veces a `agregar_etiqueta` sin la lista devolvería `['urgente']` y después `['urgente', 'casa']`. |
| `F841` | `cantidad` se calcula, pero nunca se usa. |
| `F821` | `etiqutas` no existe: es un error de tipeo. Sin Ruff, te enterarías recién al ejecutar la función, con un `NameError`. |

Sin el `--output-format concise`, Ruff muestra además el código alrededor de cada problema y una sugerencia de cómo arreglarlo.

### Arreglos automáticos

Los problemas marcados con `[*]` Ruff los puede arreglar solo:

```bash
uv run ruff check ejemplo.py --fix
```

Borra el `import os` y avisa que quedan 3. Los otros dos que tienen arreglo (`B006` y `F841`) son *unsafe*: Ruff los sabe arreglar, pero el cambio podría alterar lo que hace el programa, así que no los aplica salvo que se lo pidas con `--unsafe-fixes`. Y el `F821` no tiene arreglo automático: Ruff no puede adivinar qué quisiste escribir. Esos se arreglan a mano (es el mini ejercicio del final).

## Ruff en la API

Corré `uv run ruff check` en la carpeta del proyecto, sin nombrar archivos, y revisa todo:

```text
app/seguridad.py:21:72: UP017 [*] Use `datetime.UTC` alias
migraciones/env.py:1:1: I001 [*] Import block is un-sorted or un-formatted
migraciones/versions/1bbfd1e48383_crear_tablas.py:8:1: UP035 [*] Import from `collections.abc` instead: `Sequence`
...
tests/test_tokens.py:1:1: I001 [*] Import block is un-sorted or un-formatted
Found 8 errors.
```

(Esa salida es con `--output-format concise`; el nombre de tu migración va a ser otro.) La mayoría de los avisos vienen de dos lugares:

- **Los archivos de Alembic.** `migraciones/` la generó Alembic con su propio estilo, y cada migración nueva va a salir igual. No tiene sentido corregirlos a mano: conviene que Ruff no los revise.
- **El largo de línea.** Ruff usa por defecto un máximo de 88 caracteres por línea. El código de esta guía usa hasta 120, y por eso pide partir el import largo de `tests/test_tokens.py`.

## Configurar Ruff

Las dos cosas se configuran en `pyproject.toml`. Agregá al final:

```toml title="pyproject.toml (agregar al final)"
[tool.ruff]
line-length = 120
extend-exclude = ["migraciones"]
```

`line-length` vale para el linter y para el formateador, y `extend-exclude` suma carpetas a las que Ruff ya ignora (como `.venv`). Corré de nuevo `uv run ruff check` y queda un solo aviso:

```text
UP017 [*] Use `datetime.UTC` alias
  --> app/seguridad.py:21:72
```

Las reglas que empiezan con `UP` sugieren formas más modernas de escribir lo mismo. Desde Python 3.11, `datetime.UTC` es un nombre más corto para `timezone.utc`: son exactamente lo mismo. Con `uv run ruff check --fix`, Ruff aplica el cambio, borra el `timezone` que queda sin usar y reordena el import:

```text
Found 3 errors (3 fixed, 0 remaining).
```

```python title="app/seguridad.py (cambios)"
from datetime import UTC, datetime, timedelta

# ...

    datos = {"sub": str(usuario_id), "tipo": tipo, "exp": datetime.now(UTC) + duracion}
```

## El formateador: `ruff format`

`uv run ruff format --check` dice qué archivos cambiaría, sin tocarlos, y `--diff` muestra cómo. En la API queda uno solo, el decorador de `borrar_tarea`, que pasa de 120 caracteres:

```text
-@router.delete("/{tarea_id}", status_code=status.HTTP_204_NO_CONTENT, responses={404: {"description": "La tarea no existe"}})
+@router.delete(
+    "/{tarea_id}", status_code=status.HTTP_204_NO_CONTENT, responses={404: {"description": "La tarea no existe"}}
+)
```

Para aplicar el formato, sin `--check`:

```bash
uv run ruff format
```

```text
1 file reformatted, 17 files left unchanged
```

Corré `uv run pytest` después: tiene que seguir dando `13 passed`. Ruff no cambia lo que hace el código, pero es una buena costumbre comprobarlo. Antes de guardar cambios en Git, lo habitual es correr los tres: `ruff check`, `ruff format` y `pytest`.

!!! tip "Ruff en el editor"

    La extensión **Ruff** de VS Code (de Astral) muestra los avisos mientras escribís y puede formatear el archivo cada vez que lo guardás. Usa la misma configuración de `pyproject.toml`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Node, los dos trabajos los hacen dos herramientas: **ESLint** (linter, configurado en `eslint.config.js`) y **Prettier** (formateador). `npx eslint . --fix` y `npx prettier --write .` equivalen a `ruff check --fix` y `ruff format`. Hay herramientas que hacen las dos cosas juntas, como Biome, más parecidas a Ruff.

??? example "Ver comparación en Django ⬇️"

    En proyectos de Django se usaban Flake8 (linter), isort (orden de imports) y Black (formateador), cada uno con su configuración. Ruff reemplaza a los tres: reimplementa las reglas de Flake8, ordena los imports como isort (`I001`) y formatea casi idéntico a Black. Por eso muchos proyectos de Django ya migraron a Ruff.

??? example "Ver comparación en Spring Boot ⬇️"

    En Java, el linter suele ser **Checkstyle**, **PMD** o **SpotBugs**, y el formato se controla con **Spotless** (que usa google-java-format, por ejemplo), todos como plugins de Maven o Gradle: `./mvnw spotless:apply`. Muchos errores que en Python encuentra Ruff, como un nombre mal escrito, en Java los encuentra el compilador.

## Mini ejercicio

En `ejemplo.py` quedaron los tres avisos que `--fix` no arregló. Corregilos a mano, sin `--unsafe-fixes`, hasta que `uv run ruff check ejemplo.py` diga `All checks passed!`. Para `B006`, la forma habitual es usar `None` como valor por defecto y crear la lista adentro de la función.

??? success "Ver solución"

    ```python title="ejemplo.py"
    def agregar_etiqueta(etiqueta: str, etiquetas: list[str] | None = None) -> list[str]:
        if etiquetas is None:
            etiquetas = []
        etiquetas.append(etiqueta)
        return etiquetas
    ```

    - `F821`: `etiqutas` pasa a ser `etiquetas`.
    - `F841`: la línea de `cantidad` no hacía nada útil, así que se borra.
    - `B006`: con `None` por defecto, cada llamada sin lista crea una lista nueva. Ahora `agregar_etiqueta("urgente")` y después `agregar_etiqueta("casa")` devuelven `['urgente']` y `['casa']`.

    Cuando termines, borrá `ejemplo.py`: era solo para practicar.

---

**Siguiente:** [Logs: qué registrar y qué no →](04-logs.md)
