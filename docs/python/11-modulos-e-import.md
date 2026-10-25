# Módulos e `import`

!!! info "En esta página"

    **Qué vas a aprender:** cómo repartir tu código en varios archivos y usar lo de un archivo en otro con `import`, y cómo aprovechar la biblioteca estándar de Python.

    **Qué necesitás:** [Parámetros, valores por defecto y argumentos con nombre](10-parametros.md).

## Qué es un módulo

Cuando un programa crece, tener todo en un solo archivo se vuelve inmanejable. La solución es repartirlo en varios archivos.

En Python, **cada archivo `.py` es un módulo**. El nombre del módulo es el nombre del archivo sin el `.py`: `calculos.py` es el módulo `calculos`. Con `import` usás desde un archivo lo que está definido en otro.

## Importar tu propio módulo

Creá estos dos archivos **en la misma carpeta**:

```python title="calculos.py"
IVA = 0.21


def calcular_total(precio, cantidad):
    return precio * cantidad


def agregar_iva(precio):
    return precio * (1 + IVA)
```

```python title="main.py"
import calculos

total = calculos.calcular_total(1000, 3)
print(total)  # 3000
print(calculos.agregar_iva(total))  # 3630.0
print(calculos.IVA)  # 0.21
```

Y ejecutá el principal:

```bash
uv run main.py
```

- `import calculos` busca el archivo `calculos.py` y lo carga.
- Todo lo que el módulo define (funciones, variables) se usa con `módulo.nombre`.
- `calculos.py` no se ejecuta por su cuenta: solo define cosas para que otros las usen.

!!! tip "Constantes en mayúsculas"

    `IVA` está en mayúsculas porque es una **constante**: un valor que no debería cambiar. Python no lo impide, pero la convención avisa "no la modifiques".

## `from ... import ...`: traer nombres puntuales

Si vas a usar mucho ciertas funciones, podés traerlas directamente y llamarlas sin el prefijo:

```python title="main_from.py"
from calculos import agregar_iva, calcular_total

total = calcular_total(1000, 3)
print(agregar_iva(total))  # 3630.0
```

¿Cuál conviene? Las dos son válidas. `from ... import` es más corto y es lo que vas a ver casi siempre con FastAPI:

```python
from fastapi import FastAPI
```

## `as`: cambiar el nombre al importar

```python title="main_as.py"
import calculos as calc
from calculos import calcular_total as total_de

print(calc.IVA)  # 0.21
print(total_de(500, 2))  # 1000
```

Se usa para acortar nombres largos o evitar que dos nombres choquen. No abuses: un nombre claro vale más que uno corto.

## La biblioteca estándar

Python trae muchos módulos ya incluidos, listos para importar sin instalar nada. A eso se lo llama **biblioteca estándar**:

```python title="estandar.py"
import math
import random
from datetime import date

print(math.sqrt(16))  # 4.0
print(random.choice(["rojo", "verde", "azul"]))  # uno al azar
print(date.today())  # la fecha de hoy, por ejemplo 2026-10-24
```

Algunos que vas a usar en esta guía: `datetime` (fechas), `pathlib` (rutas de archivos), `os` (variables de entorno), `json` y `logging`.

Las librerías que **no** vienen con Python, como FastAPI, se instalan aparte. Eso lo hace `uv`, y lo vas a ver en la página de entornos virtuales.

## Errores comunes

**El módulo no existe** (o está mal escrito, o en otra carpeta):

```text
ModuleNotFoundError: No module named 'calculo'
```

**El módulo existe, pero no tiene ese nombre adentro:**

```text
ImportError: cannot import name 'calcular_totales' from 'calculos' (...). Did you mean: 'calcular_total'?
```

**Un archivo tuyo se llama igual que un módulo de Python.** Si creás un `random.py` y adentro hacés `import random`, Python importa **tu** archivo en vez del módulo estándar, y nada funciona:

```text
AttributeError: module 'random' has no attribute 'choice' (consider renaming 'random.py' since it has the same name as the standard library module named 'random' ...)
```

!!! warning "No le pongas a tus archivos nombres de módulos existentes"

    Evitá llamar a tus archivos `random.py`, `math.py`, `json.py`, `fastapi.py`, `pydantic.py`, etc. Es uno de los errores más desconcertantes al empezar. El mensaje de arriba es de Python 3.13; en 3.12 dice algo menos claro sobre una *circular import*.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | Node (ES modules) | Python |
    | --- | --- |
    | `export function calcularTotal()` | `def calcular_total():` (no hace falta exportar) |
    | `import * as calculos from "./calculos.js"` | `import calculos` |
    | `import { calcularTotal } from "./calculos.js"` | `from calculos import calcular_total` |
    | `import { a as b } from "./x.js"` | `from x import a as b` |

    La diferencia principal: en Python **todo** lo que define un módulo se puede importar; no existe `export`. Tampoco se escribe la ruta con `./` ni la extensión.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. Ya lo usaste en `from django.db import models` o en `from . import views` dentro de `urls.py`. Ese punto (`.`) es un *import relativo* a la carpeta actual; lo vas a ver en la próxima página, con los paquetes.

??? example "Ver comparación en Spring Boot ⬇️"

    En Java, `import` trae **clases** de un paquete. En Python podés importar cualquier cosa definida en un módulo: funciones, constantes, clases.

    | Java | Python |
    | --- | --- |
    | `import java.time.LocalDate;` | `from datetime import date` |
    | `Math.sqrt(16)` (sin import) | `import math` y `math.sqrt(16)` |
    | `public static final double IVA` | `IVA = 0.21` (en mayúsculas, por convención) |

    Además, en Python no hace falta que el nombre del archivo coincida con el de una clase.

## Mini ejercicio

Creá un módulo `textos.py` con una función `limpiar(texto)` que saque espacios y pase a minúsculas, y una función `es_email_valido(email)` que use `limpiar`. Después, desde `main.py`, importá las dos funciones con `from` y probalas.

??? success "Ver solución"

    ```python title="textos.py"
    def limpiar(texto):
        return texto.strip().lower()


    def es_email_valido(email):
        email = limpiar(email)
        return "@" in email and "." in email
    ```

    ```python title="main.py"
    from textos import es_email_valido, limpiar

    print(limpiar("  HOLA "))  # hola
    print(es_email_valido(" Ana@Ejemplo.com "))  # True
    ```

    Fijate que `es_email_valido` llama a `limpiar` sin importarla: están en el mismo módulo.

---

**Siguiente:** [Paquetes y `__init__.py` →](12-paquetes.md)
