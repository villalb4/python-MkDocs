# Funciones

!!! info "En esta página"

    **Qué vas a aprender:** cómo agrupar código en funciones con `def`, cómo pasarles datos y cómo devolver un resultado con `return`.

    **Qué necesitás:** [Bucles](08-bucles.md).

## Por qué existen las funciones

Una **función** es un bloque de código con nombre que podés ejecutar las veces que quieras. Sirven para:

- **No repetir código:** lo escribís una vez y lo usás en muchos lugares.
- **Ponerle nombre a una idea:** `calcular_total(carrito)` se entiende mejor que cinco renglones de cuentas.

Ya usaste varias: `print`, `len`, `type`, `sorted`. Ahora vas a escribir las tuyas. Y es importante: **en FastAPI, cada ruta de la API es una función**.

## Definir y llamar una función

```python title="definir.py"
def saludar():
    print("Hola")
    print("Bienvenida al sistema")


saludar()
saludar()
```

```text
Hola
Bienvenida al sistema
Hola
Bienvenida al sistema
```

- `def`, el nombre, paréntesis y dos puntos. Abajo, el bloque con sangría.
- **Definir** la función no ejecuta nada: solo la guarda con ese nombre.
- **Llamarla** (`saludar()`) ejecuta su bloque. Los paréntesis son obligatorios.

Los nombres siguen el mismo estilo que las variables: minúsculas con guiones bajos, y en lo posible un verbo (`calcular_total`, `enviar_email`). Por convención se dejan dos renglones en blanco antes y después de cada función.

## Parámetros: pasarle datos

Entre los paréntesis van los **parámetros**: variables que reciben los valores que le pases al llamarla.

```python title="parametros.py"
def saludar(nombre):
    print(f"Hola, {nombre}")


saludar("Ana")  # Hola, Ana
saludar("Juan")  # Hola, Juan
```

En la primera llamada, `nombre` vale `"Ana"`; en la segunda, `"Juan"`. Si la función necesita varios datos, se separan con comas: `def calcular_total(precio, cantidad):`. En la próxima página vas a ver todas las formas de pasar parámetros.

## `return`: devolver un resultado

Con `return`, la función **devuelve** un valor a quien la llamó, y ese valor se puede guardar o usar en otra cuenta:

```python title="return.py"
def calcular_total(precio, cantidad):
    return precio * cantidad


total = calcular_total(1500, 3)
print(total)  # 4500

print(calcular_total(800, 2) + 500)  # 2100
```

### `print` no es lo mismo que `return`

Es la confusión más común al empezar. `print` **muestra** algo en la terminal; `return` **entrega** un valor. Una función sin `return` devuelve `None`:

```python title="print_vs_return.py"
def total_con_print(precio, cantidad):
    print(precio * cantidad)


resultado = total_con_print(1500, 3)  # muestra 4500
print(resultado)  # None
```

El `4500` apareció en pantalla, pero `resultado` quedó vacío. En una API casi nunca vas a usar `print` para entregar datos: la función de cada ruta **devuelve** la respuesta con `return`.

### `return` termina la función

Apenas se ejecuta un `return`, la función termina y lo que sigue no se ejecuta. Esto permite resolver primero los casos especiales:

```python title="temprano.py"
def describir_stock(stock):
    if stock == 0:
        return "Sin stock"
    if stock < 5:
        return "Últimas unidades"
    return "Disponible"


print(describir_stock(0))  # Sin stock
print(describir_stock(3))  # Últimas unidades
print(describir_stock(20))  # Disponible
```

## Las variables de adentro se quedan adentro

Las variables creadas dentro de una función (y sus parámetros) **solo existen ahí adentro**:

```python title="alcance.py"
def calcular_iva(precio):
    iva = precio * 0.21
    return iva


print(calcular_iva(1000))  # 210.0
print(iva)
```

```text
NameError: name 'iva' is not defined
```

Si necesitás el valor afuera, devolvelo con `return` y guardalo: `iva = calcular_iva(1000)`.

!!! note "Primero se define, después se llama"

    Como el script se ejecuta de arriba hacia abajo, si llamás a una función **antes** del renglón donde está su `def`, Python da `NameError`: todavía no la conoce. Por eso las funciones se definen arriba y se usan abajo.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | JavaScript | Python |
    | --- | --- |
    | `function saludar(nombre) { ... }` | `def saludar(nombre):` |
    | `const doble = (x) => x * 2` | `def doble(x): return x * 2` |
    | Devuelve `undefined` sin `return` | Devuelve `None` sin `return` |

    En Express, cada ruta recibe una función `(req, res) => ...` que **no** devuelve la respuesta sino que llama a `res.json(...)`. En FastAPI la función de la ruta simplemente hace `return` de los datos.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. Las *function-based views* de Django son funciones comunes que reciben `request` y hacen `return` de una respuesta. En FastAPI la idea es la misma, pero devolvés directamente los datos (un diccionario, una lista) en vez de un `JsonResponse`.

??? example "Ver comparación en Spring Boot ⬇️"

    En Java todo método vive dentro de una clase y declara el tipo de lo que recibe y devuelve:

    ```java
    public int calcularTotal(int precio, int cantidad) {
        return precio * cantidad;
    }
    ```

    En Python una función puede estar suelta en el archivo, sin clase, y no hace falta declarar tipos (aunque se pueden agregar como *type hints*, que vas a ver más adelante). El equivalente de `void` es una función sin `return`, que devuelve `None`.

## Mini ejercicio

Escribí una función `es_email_valido(email)` que devuelva `True` si el texto, sin espacios al principio ni al final, contiene un `@` y un `.`, y `False` si no. Probala con `"ana@ejemplo.com"`, `"ana.ejemplo.com"` y `"  luz@mail.com "`.

??? success "Ver solución"

    ```python
    def es_email_valido(email):
        email = email.strip()
        return "@" in email and "." in email


    print(es_email_valido("ana@ejemplo.com"))  # True
    print(es_email_valido("ana.ejemplo.com"))  # False
    print(es_email_valido("  luz@mail.com "))  # True
    ```

    La condición `"@" in email and "." in email` ya es un `bool`, así que se puede devolver directamente, sin `if`. (Validar emails de verdad es más complicado; en el Capítulo 2 vas a ver que Pydantic lo hace por vos.)

---

**Siguiente:** [Parámetros, valores por defecto y argumentos con nombre →](10-parametros.md)
