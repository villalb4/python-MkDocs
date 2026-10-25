# Manejo de errores (`try` / `except`)

!!! info "En esta página"

    **Qué vas a aprender:** qué son las excepciones, cómo atraparlas con `try` / `except` para que el programa no se corte, y cómo lanzar tus propios errores con `raise`.

    **Qué necesitás:** [Variables de entorno y archivos `.env`](15-variables-de-entorno.md).

## Qué es una excepción

A lo largo del capítulo viste varios errores: `NameError`, `TypeError`, `KeyError`, `FileNotFoundError`. En Python se llaman **excepciones**: avisos de que algo salió mal mientras el programa corría. Si nadie las atiende, el programa se corta:

```python title="sin_try.py"
texto = "doce"
cantidad = int(texto)  # ValueError: invalid literal for int() with base 10: 'doce'
print("Esto no se ejecuta")
```

Muchas veces el error es **esperable**: un usuario escribe letras donde iba un número, falta un archivo, una clave no está. En esos casos no querés que todo se caiga, sino reaccionar.

## `try` / `except`: atrapar un error

```python title="try.py"
texto = "doce"

try:
    cantidad = int(texto)
    print(f"Cantidad: {cantidad}")
except ValueError:
    print(f"'{texto}' no es un número válido")

print("El programa sigue")
```

```text
'doce' no es un número válido
El programa sigue
```

- Python ejecuta el bloque `try`. Si todo sale bien, **saltea** el `except`.
- Si en algún renglón ocurre un `ValueError`, deja el `try` en ese punto (por eso `Cantidad:` no se muestra) y ejecuta el bloque `except`.
- Después sigue normalmente con lo que viene abajo. Con `texto = "12"` se muestra `Cantidad: 12` y después `El programa sigue`.

## Atrapar errores distintos

Podés poner varios `except`, uno por tipo de error. Python usa el primero que coincida:

```python title="varios.py"
def mostrar_precio(producto, campo):
    try:
        precio = int(producto[campo])
        print(f"Precio: ${precio}")
    except KeyError:
        print(f"El producto no tiene el campo '{campo}'")
    except ValueError:
        print(f"El campo '{campo}' no es un número")


producto = {"nombre": "Teclado", "precio": "45000", "stock": "pocos"}

mostrar_precio(producto, "precio")  # Precio: $45000
mostrar_precio(producto, "color")  # El producto no tiene el campo 'color'
mostrar_precio(producto, "stock")  # El campo 'stock' no es un número
```

Si querés ver el mensaje original del error, agregá `as` y un nombre:

```python
try:
    int("doce")
except ValueError as error:
    print(f"Falló: {error}")  # Falló: invalid literal for int() with base 10: 'doce'
```

!!! warning "No atrapes todo"

    Vas a ver código con `except:` o `except Exception:` solos, sin tipo. Eso atrapa **cualquier** error, incluidos los que no esperabas (un nombre mal escrito, un bug), y los esconde. Atrapá solo los errores que sabés manejar; los demás, dejá que se vean.

## `else` y `finally`

Dos bloques opcionales:

- `else`: se ejecuta solo si el `try` **no** tuvo errores.
- `finally`: se ejecuta **siempre**, haya habido error o no. Sirve para limpiar o cerrar cosas.

```python title="archivo.py"
from pathlib import Path

archivo = Path("config.txt")

try:
    contenido = archivo.read_text(encoding="utf-8")
except FileNotFoundError:
    print("No hay archivo de configuración, uso valores por defecto")
    contenido = ""
else:
    print("Configuración leída")
finally:
    print("Listo")
```

Si `config.txt` no existe, muestra el mensaje de valores por defecto y `Listo`. Si existe, `Configuración leída` y `Listo`.

## `raise`: lanzar tus propios errores

Tus funciones también pueden avisar que algo está mal. Con `raise` lanzás una excepción, con un mensaje que explique el problema:

```python title="raise.py"
def crear_usuario(nombre, edad):
    if edad < 0:
        raise ValueError("La edad no puede ser negativa")
    return {"nombre": nombre, "edad": edad}


print(crear_usuario("Ana", 30))  # {'nombre': 'Ana', 'edad': 30}

try:
    crear_usuario("Juan", -5)
except ValueError as error:
    print(f"No se pudo crear: {error}")  # No se pudo crear: La edad no puede ser negativa
```

`raise` corta la función en el momento, como un `return`, pero en vez de devolver un valor le pasa el problema a quien la llamó. Si nadie lo atrapa, el programa se corta y el traceback muestra tu mensaje: `ValueError: La edad no puede ser negativa`.

!!! tip "Así se responden errores en FastAPI"

    En el Capítulo 2 vas a escribir `raise HTTPException(status_code=404, detail="Tarea no encontrada")` dentro de una ruta. FastAPI atrapa esa excepción y la convierte en una respuesta de error para el cliente. Es exactamente esta idea.

## Excepciones comunes

| Excepción | Cuándo aparece |
| --- | --- |
| `ValueError` | El tipo es correcto pero el valor no sirve: `int("doce")`. |
| `TypeError` | Operación con el tipo equivocado: `"5" + 3`. |
| `KeyError` | La clave no está en el diccionario. |
| `IndexError` | La posición no existe en la lista. |
| `NameError` | Usaste una variable o función que no existe. |
| `FileNotFoundError` | El archivo no existe. |
| `ZeroDivisionError` | Dividiste por cero. |

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | JavaScript | Python |
    | --- | --- |
    | `try { ... } catch (e) { ... }` | `try:` ... `except TipoDeError as e:` |
    | `finally { ... }` | `finally:` |
    | `throw new Error("mensaje")` | `raise ValueError("mensaje")` |
    | `e.message` | `str(e)` o directamente `f"{e}"` |

    La gran diferencia: en JavaScript el `catch` atrapa todo y tenés que revisar el tipo adentro (`if (e instanceof TypeError)`). En Python cada `except` dice qué tipo atrapa. Y en Express respondés un error con `res.status(404).json(...)`; en FastAPI, con `raise HTTPException(...)`.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. Seguramente usaste `raise Http404("No existe")` o `get_object_or_404(...)` en una view, o atrapaste `Tarea.DoesNotExist`. En FastAPI el equivalente es `raise HTTPException(status_code=404, ...)`.

??? example "Ver comparación en Spring Boot ⬇️"

    | Java | Python |
    | --- | --- |
    | `try { } catch (NumberFormatException e) { }` | `try:` ... `except ValueError as e:` |
    | `throw new IllegalArgumentException("...")` | `raise ValueError("...")` |
    | `e.getMessage()` | `str(e)` |
    | `throw new ResponseStatusException(HttpStatus.NOT_FOUND)` | `raise HTTPException(status_code=404)` |

    En Python no existen las *checked exceptions*: no hay que declarar `throws` en la función ni el compilador te obliga a atrapar nada.

## Mini ejercicio

Escribí una función `leer_puerto(texto)` que convierta el texto a número y lo devuelva. Si el texto no es un número, o si el número no está entre `1` y `65535`, tiene que lanzar un `ValueError` con un mensaje claro. Probala con `"8000"`, `"ochenta"` y `"70000"`, atrapando los errores para mostrarlos.

??? success "Ver solución"

    ```python
    def leer_puerto(texto):
        try:
            puerto = int(texto)
        except ValueError:
            raise ValueError(f"El puerto debe ser un número, no '{texto}'")
        if not 1 <= puerto <= 65535:
            raise ValueError(f"El puerto {puerto} está fuera de rango")
        return puerto


    for valor in ["8000", "ochenta", "70000"]:
        try:
            print(leer_puerto(valor))
        except ValueError as error:
            print(f"Error: {error}")
    ```

    ```text
    8000
    Error: El puerto debe ser un número, no 'ochenta'
    Error: El puerto 70000 está fuera de rango
    ```

    `1 <= puerto <= 65535` es una comparación encadenada: Python la lee como "`puerto` está entre 1 y 65535". Fijate que adentro del primer `except` lanzamos un error nuevo, con un mensaje más útil que el original.

---

**Siguiente:** [Clases básicas →](17-clases-basicas.md)
