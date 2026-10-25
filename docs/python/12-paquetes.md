# Paquetes y `__init__.py`

!!! info "En esta página"

    **Qué vas a aprender:** cómo agrupar módulos en carpetas (paquetes), para qué sirve el archivo `__init__.py` y cómo importar entre módulos de un mismo paquete.

    **Qué necesitás:** [Módulos e `import`](11-modulos-e-import.md).

## De archivos sueltos a carpetas

Con módulos ya podés repartir el código en varios archivos. Pero una API real tiene decenas: rutas, modelos, configuración, base de datos. Ponerlos todos en la misma carpeta sería un caos.

Un **paquete** es una carpeta que agrupa módulos. Se reconoce porque tiene adentro un archivo llamado `__init__.py` (con dos guiones bajos a cada lado).

## Armar un paquete

Creá esta estructura:

```text
proyecto/
├── main.py
└── tienda/
    ├── __init__.py
    ├── carrito.py
    └── precios.py
```

`__init__.py` puede estar **vacío**: alcanza con que exista. Los otros dos archivos:

```python title="tienda/precios.py"
IVA = 0.21


def agregar_iva(precio):
    return precio * (1 + IVA)
```

```python title="tienda/carrito.py"
from .precios import agregar_iva


def calcular_total(productos):
    subtotal = 0
    for producto in productos:
        subtotal += producto["precio"] * producto["cantidad"]
    return agregar_iva(subtotal)
```

## Importar desde un paquete

Desde `main.py`, se usa un punto para separar el paquete del módulo:

```python title="main.py"
from tienda.carrito import calcular_total
from tienda.precios import IVA

carrito = [
    {"nombre": "Teclado", "precio": 1000, "cantidad": 2},
    {"nombre": "Mouse", "precio": 500, "cantidad": 1},
]

print(f"IVA: {IVA:.0%}")  # IVA: 21%
print(calcular_total(carrito))  # 3025.0
```

Ejecutalo **desde la carpeta `proyecto/`**:

```bash
uv run main.py
```

`tienda.carrito` se lee "el módulo `carrito` del paquete `tienda`", y corresponde al archivo `tienda/carrito.py`. También funciona `import tienda.precios`, pero después tenés que escribir `tienda.precios.agregar_iva(...)` completo.

## Imports relativos: el punto

Fijate el primer renglón de `carrito.py`:

```python
from .precios import agregar_iva
```

El punto del principio significa "**desde este mismo paquete**". Es un **import relativo**: no importa cómo se llame el paquete, busca `precios.py` al lado de `carrito.py`. Con dos puntos (`..`) subís un nivel, al paquete que contiene a este.

También podrías escribir `from tienda.precios import agregar_iva` (un **import absoluto**). Los dos funcionan; dentro de un paquete, cada proyecto elige un estilo y lo mantiene.

!!! warning "No ejecutes directamente un archivo de adentro del paquete"

    Si hacés `uv run tienda/carrito.py`, Python lo trata como un script suelto y no sabe que pertenece a `tienda`:

    ```text
    ImportError: attempted relative import with no known parent package
    ```

    Siempre ejecutá el punto de entrada que está **afuera** del paquete (`main.py`). Si de verdad necesitás correr un módulo interno, usá `uv run python -m tienda.carrito` desde `proyecto/`: con `-m`, Python lo carga como parte del paquete.

## Para qué sirve el contenido de `__init__.py`

`__init__.py` se ejecuta **una sola vez**, la primera vez que se importa algo del paquete. Si le agregás un `print("Cargando el paquete tienda")` y corrés `main.py`, el mensaje aparece una vez, aunque `main.py` importa dos módulos de `tienda`.

El uso más común es **reexportar**: traer al paquete los nombres más usados para que se importen más corto.

```python title="tienda/__init__.py"
from .carrito import calcular_total
from .precios import IVA, agregar_iva
```

Ahora, desde afuera, se puede importar directamente del paquete:

```python
from tienda import agregar_iva, calcular_total
```

Así es como funciona `from fastapi import FastAPI`: `fastapi` es un paquete, y su `__init__.py` reexporta `FastAPI`, que en realidad está definido en otro módulo de adentro.

!!! note "¿Y si no pongo `__init__.py`?"

    En Python moderno una carpeta sin `__init__.py` también se puede importar (se llama *namespace package*), pero se comporta distinto en algunos casos y confunde a herramientas como pytest. Poné siempre un `__init__.py`, aunque esté vacío.

## Cómo se ve en una API de FastAPI

Más adelante vas a organizar tu API con una estructura parecida a esta, donde cada carpeta es un paquete:

```text
app/
├── __init__.py
├── main.py
├── config.py
├── routers/
│   ├── __init__.py
│   ├── tareas.py
│   └── usuarios.py
└── models/
    ├── __init__.py
    └── tarea.py
```

Y desde `app/main.py` vas a importar con cosas como `from app.routers import tareas`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Un paquete de Python se parece a una carpeta con un `index.js`: `__init__.py` cumple ese rol, y reexportar en él es como hacer `export { calcularTotal } from "./carrito.js"` en el `index.js`.

    Una diferencia: en Node las rutas son relativas al archivo (`../services/precios.js`). En Python los imports absolutos parten de la carpeta desde la que ejecutás el programa (`tienda.precios`), por eso importa desde dónde lo corrés.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. Cada *app* de Django es un paquete: por eso tiene su `__init__.py`, y por eso en `urls.py` escribís `from . import views`. El proyecto se ejecuta siempre desde afuera, con `manage.py`, por el mismo motivo que acá usás `main.py`.

??? example "Ver comparación en Spring Boot ⬇️"

    Un paquete de Python es parecido a un paquete de Java (`com.ejemplo.tienda`), con algunas diferencias:

    - No hace falta declarar `package` arriba de cada archivo: el paquete es la carpeta.
    - El nombre lo da la estructura de carpetas desde donde ejecutás, no un dominio invertido.
    - `__init__.py` no tiene equivalente directo en Java; lo más cercano es `package-info.java`, pero `__init__.py` además puede tener código y reexportar nombres.

## Mini ejercicio

Agregale al paquete `tienda` un módulo `descuentos.py` con una función `aplicar_descuento(precio, porcentaje)` que use `agregar_iva` de `precios.py` para devolver el precio con descuento **y** con IVA. Llamala desde `main.py`.

??? success "Ver solución"

    ```python title="tienda/descuentos.py"
    from .precios import agregar_iva


    def aplicar_descuento(precio, porcentaje):
        con_descuento = precio * (1 - porcentaje / 100)
        return agregar_iva(con_descuento)
    ```

    ```python title="main.py"
    from tienda.descuentos import aplicar_descuento

    print(aplicar_descuento(1000, 10))  # 1089.0
    ```

    `1000` con 10 % de descuento queda en `900`, y con IVA, `900 * 1.21 = 1089.0`.

---

**Siguiente:** [`__name__` y `__file__` →](13-name-y-file.md)
