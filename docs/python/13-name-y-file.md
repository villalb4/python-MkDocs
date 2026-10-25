# `__name__` y `__file__`

!!! info "En esta página"

    **Qué vas a aprender:** qué son las variables especiales `__name__` y `__file__`, para qué sirve el famoso `if __name__ == "__main__":` y cómo saber dónde está guardado un archivo.

    **Qué necesitás:** [Paquetes y `__init__.py`](12-paquetes.md).

## Importar un módulo ejecuta su código

Cuando importás un módulo, Python **ejecuta el archivo entero**, de arriba hacia abajo. Normalmente eso solo define funciones, pero mirá qué pasa si dejaste una prueba suelta:

```python title="precios.py"
def agregar_iva(precio):
    return precio * 1.21


print("Probando:", agregar_iva(100))
```

```python title="main.py"
from precios import agregar_iva

print("Total:", agregar_iva(1000))
```

Al ejecutar `uv run main.py`:

```text
Probando: 121.0
Total: 1210.0
```

El `Probando:` se coló en `main.py`. Querías que esa prueba corriera solo al ejecutar `precios.py` directamente, no al importarlo. Para distinguir los dos casos está `__name__`.

## `__name__`: ¿me ejecutaron o me importaron?

Python crea solo, en cada módulo, una variable llamada `__name__` (con dos guiones bajos a cada lado). Su valor depende de cómo se cargó el archivo:

```python title="saludo.py"
print(f"__name__ vale: {__name__}")
```

```python title="main.py"
import saludo
```

| Cómo se cargó | Comando | Muestra |
| --- | --- | --- |
| Ejecutado directamente | `uv run saludo.py` | `__name__ vale: __main__` |
| Importado | `uv run main.py` | `__name__ vale: saludo` |

- Si el archivo es el que **ejecutaste**, `__name__` vale `"__main__"`.
- Si fue **importado**, vale el nombre del módulo. Dentro de un paquete, el nombre completo: `tienda.precios`.

## `if __name__ == "__main__":`

Con eso, la solución al problema de arriba es poner el código de prueba dentro de un `if`:

```python title="precios.py"
def agregar_iva(precio):
    return precio * 1.21


if __name__ == "__main__":
    print("Probando:", agregar_iva(100))
```

Ahora:

- `uv run precios.py` → muestra `Probando: 121.0`.
- `uv run main.py` → muestra solo `Total: 1210.0`.

Se lee: "si este archivo es el que se está ejecutando, hacé esto". Vas a encontrar este bloque al final de muchísimos archivos de Python. En proyectos de FastAPI, a veces se usa para arrancar el servidor desde el propio archivo; en esta guía lo vas a arrancar desde la terminal, pero si lo ves, ya sabés qué significa.

!!! tip "Otro uso de `__name__`"

    Más adelante vas a ver `logger = logging.getLogger(__name__)`. Como `__name__` vale el nombre del módulo, cada mensaje de log dice de qué archivo salió.

## `__file__`: dónde está este archivo

Otra variable especial: `__file__` tiene la **ruta completa** del archivo en el que está escrita.

Para ver por qué es útil, compará con la **carpeta de trabajo**: la carpeta desde la que ejecutaste el comando, que te devuelve `os.getcwd()`:

```python title="scripts/donde.py"
import os

print(f"Este archivo:       {__file__}")
print(f"Carpeta de trabajo: {os.getcwd()}")
```

Desde la carpeta `proyecto/`:

```console
$ uv run scripts/donde.py
Este archivo:       /Users/ana/proyecto/scripts/donde.py
Carpeta de trabajo: /Users/ana/proyecto
```

Y desde adentro de `scripts/`:

```console
$ uv run donde.py
Este archivo:       /Users/ana/proyecto/scripts/donde.py
Carpeta de trabajo: /Users/ana/proyecto/scripts
```

`__file__` es siempre el mismo; la carpeta de trabajo **cambia** según desde dónde lo ejecutes. (En Windows las rutas se ven como `C:\Users\ana\proyecto\...`.)

Esto importa cuando tu programa tiene que abrir un archivo que está al lado del código, como un `.env` o un archivo de datos: si usás una ruta relativa, se busca desde la carpeta de trabajo, y falla si ejecutaste desde otro lado. Con `__file__` sabés dónde está tu código sin importar desde dónde lo corrieron. En la próxima página vas a ver cómo armar rutas a partir de `__file__` con `pathlib`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | Node | Python |
    | --- | --- |
    | `require.main === module` (CommonJS) | `__name__ == "__main__"` |
    | `__filename` (CommonJS) | `__file__` |
    | `import.meta.filename` (ES modules) | `__file__` |
    | `process.cwd()` | `os.getcwd()` |

    El problema que resuelve es el mismo: distinguir el archivo que se ejecutó de los que se importaron.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. Abrí `manage.py` y vas a ver un `if __name__ == "__main__": main()` al final. Y en `settings.py`, la línea `BASE_DIR = Path(__file__).resolve().parent.parent` usa `__file__` para encontrar la carpeta del proyecto; la vas a entender del todo en la próxima página.

??? example "Ver comparación en Spring Boot ⬇️"

    `if __name__ == "__main__":` cumple el rol de `public static void main(String[] args)`: marca qué se ejecuta cuando ese archivo es el punto de entrada. La diferencia es que en Java solo se ejecuta lo que está dentro de `main`, mientras que en Python todo el archivo se ejecuta igual, y el `if` es la forma de separar lo que solo debe correr como programa principal.

    Para ubicar archivos, en Spring Boot normalmente usás el *classpath* (`src/main/resources`); en Python lo habitual es partir de `__file__`.

## Mini ejercicio

Tenés este módulo. ¿Qué muestra `uv run main.py`? ¿Y `uv run textos.py`?

```python title="textos.py"
def gritar(texto):
    return texto.upper() + "!"


print("Cargando textos")

if __name__ == "__main__":
    print(gritar("hola"))
```

```python title="main.py"
from textos import gritar

print(gritar("chau"))
```

??? success "Ver solución"

    `uv run main.py` muestra:

    ```text
    Cargando textos
    CHAU!
    ```

    `uv run textos.py` muestra:

    ```text
    Cargando textos
    HOLA!
    ```

    El `print("Cargando textos")` está **afuera** del `if`, así que se ejecuta siempre, tanto al importar como al ejecutar directamente. Lo de adentro del `if` solo corre con `uv run textos.py`.

---

**Siguiente:** [Rutas de archivos con `pathlib` →](14-pathlib.md)
