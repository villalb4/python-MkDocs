# Variables y tipos básicos

!!! info "En esta página"

    **Qué vas a aprender:** cómo guardar datos en variables y cuáles son los tipos básicos de Python: `int`, `float`, `str`, `bool` y `None`.

    **Qué necesitás:** [Instalar y usar `uv`](02-instalar-uv.md).

## Variables

Una **variable** es un nombre que apunta a un dato. Se crea con `=`: a la izquierda el nombre, a la derecha el valor.

```python title="variables.py"
nombre = "Ana"
edad = 30

print(nombre)
print(edad)

edad = 31
print(edad)
```

```text
Ana
30
31
```

- No hace falta declarar la variable antes ni decir de qué tipo es: se crea la primera vez que le asignás algo.
- Si le asignás otro valor, el anterior se pierde (`edad` pasa de `30` a `31`).
- El `=` **no** es "igual" de matemática: significa "guardá esto en este nombre".

!!! tip "Cómo nombrar variables"

    En Python se usan minúsculas y guiones bajos: `precio_total`, `en_stock`. A este estilo se lo llama *snake_case*. Elegí nombres que digan qué guardan: `edad` es mejor que `e` o `x`.

## Los tipos básicos

Cada dato tiene un **tipo**, que dice qué clase de dato es y qué se puede hacer con él.

| Tipo | Qué guarda | Ejemplo |
| --- | --- | --- |
| `int` | Números enteros | `3`, `-10`, `2024` |
| `float` | Números con decimales | `19.99`, `0.5` |
| `str` | Texto (entre comillas) | `"Teclado"`, `'hola'` |
| `bool` | Verdadero o falso | `True`, `False` |
| `None` | "No hay valor" | `None` |

La función `type(...)` te dice el tipo de cualquier dato:

```python title="tipos.py"
cantidad = 3
precio = 19.99
producto = "Teclado"
en_stock = True
descuento = None

print(type(cantidad))
print(type(precio))
print(type(producto))
print(type(en_stock))
print(type(descuento))
```

```text
<class 'int'>
<class 'float'>
<class 'str'>
<class 'bool'>
<class 'NoneType'>
```

Dos detalles que suelen confundir:

- `True`, `False` y `None` van con **mayúscula inicial**. `true` o `none` dan error.
- `None` se usa para decir "todavía no hay valor" o "no aplica". Por ejemplo, un producto sin descuento.

## Operaciones con números

```python title="operaciones.py"
print(10 + 3)   # suma: 13
print(10 - 3)   # resta: 7
print(10 * 3)   # multiplicación: 30
print(10 / 2)   # división: 5.0
print(10 // 3)  # división entera, descarta los decimales: 3
print(10 % 3)   # resto de la división: 1
print(2 ** 3)   # potencia: 8
```

Lo que va después de `#` es un **comentario**: Python lo ignora. Sirve para dejar notas en el código. Desde acá, en los ejemplos cortos vamos a anotar el resultado en un comentario en vez de mostrar la salida aparte.

Fijate que `10 / 2` da `5.0` y no `5`: la división con `/` siempre devuelve un `float`.

## Comparaciones: de dónde salen los `bool`

Al comparar dos valores, el resultado es `True` o `False`:

```python title="comparaciones.py"
edad = 20

print(edad >= 18)  # mayor o igual: True
print(edad == 30)  # igual (dos signos =): False
print(edad != 30)  # distinto: True
```

También existen `>`, `<` y `<=`. Ojo con `==` (comparar) y `=` (asignar): son cosas distintas.

## Convertir entre tipos

Python no mezcla tipos por su cuenta. Si querés sumar un número a un texto, da error:

```python title="error.py"
cantidad = "5"
print(cantidad + 3)
```

```text
Traceback (most recent call last):
  File "error.py", line 2, in <module>
    print(cantidad + 3)
          ~~~~~~~~~^~~
TypeError: can only concatenate str (not "int") to str
```

`"5"` es texto, no un número. Para convertir, usá el nombre del tipo como función:

```python title="conversion.py"
texto = "5"
numero = int(texto)

print(numero + 3)         # 8: suma de números
print(str(numero) + "3")  # 53: unión de textos
print(float("2.5"))       # 2.5
print(bool(0), bool(1))   # False True
```

Esto aparece todo el tiempo en un backend: lo que llega de afuera (un formulario, una variable de entorno) suele ser texto y hay que convertirlo.

!!! warning "No todo texto se puede convertir"

    `int("hola")` da `ValueError: invalid literal for int() with base 10: 'hola'`. Solo se convierte un texto que realmente represente un número.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En JavaScript declarás con `let` o `const`; en Python solo escribís el nombre. Además, JavaScript tiene un único tipo `number`, mientras que Python distingue `int` de `float`.

    | JavaScript | Python |
    | --- | --- |
    | `let edad = 30` | `edad = 30` |
    | `true` / `false` | `True` / `False` |
    | `null` / `undefined` | `None` |
    | `Number("5")` | `int("5")` |

    La gran diferencia: `"5" + 3` en JavaScript da `"53"` sin avisar. En Python da error, y eso evita bugs silenciosos.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. Vas a ver estos mismos tipos en los modelos de Django: un `IntegerField` guarda un `int`, un `CharField` un `str`, y un campo con `null=True` puede valer `None`.

??? example "Ver comparación en Spring Boot ⬇️"

    En Java el tipo va escrito y no cambia: `int edad = 30;`. En Python no se escribe el tipo, y una variable podría apuntar después a un dato de otro tipo. Más adelante vas a ver los *type hints*, que se parecen a los tipos de Java pero son opcionales.

    | Java | Python |
    | --- | --- |
    | `int`, `long` | `int` (sin límite de tamaño) |
    | `double` | `float` |
    | `String` | `str` |
    | `boolean` | `bool` |
    | `null` | `None` |
    | `Integer.parseInt("5")` | `int("5")` |

## Mini ejercicio

¿Qué muestra este script? Pensalo antes de ejecutarlo.

```python title="ejercicio.py"
precio = 1500
cantidad = 3
total = precio * cantidad
hay_envio_gratis = total >= 4000

print(total)
print(hay_envio_gratis)
print(type(total / cantidad))
```

??? success "Ver solución"

    ```text
    4500
    True
    <class 'float'>
    ```

    `total` es `1500 * 3 = 4500`. Como `4500 >= 4000`, `hay_envio_gratis` es `True`. Y aunque `4500 / 3` da justo `1500`, la división con `/` siempre devuelve un `float` (`1500.0`).

---

**Siguiente:** [Strings y f-strings →](04-strings-y-f-strings.md)
