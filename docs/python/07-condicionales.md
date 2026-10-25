# Condicionales (`if`, `elif`, `else`)

!!! info "En esta página"

    **Qué vas a aprender:** cómo hacer que tu programa tome decisiones con `if`, `elif` y `else`, y por qué en Python la sangría es obligatoria.

    **Qué necesitás:** [Diccionarios](06-diccionarios.md).

## `if`: ejecutar algo solo si se cumple una condición

Hasta ahora, todos los scripts ejecutaban todos sus renglones. Con `if` hacés que un bloque se ejecute **solo si** una condición es `True`:

```python title="if.py"
edad = 20

if edad >= 18:
    print("Podés registrarte")
    print("Bienvenida")

print("Fin del script")
```

```text
Podés registrarte
Bienvenida
Fin del script
```

Si cambiás `edad` a `15`, solo se muestra `Fin del script`.

La estructura es siempre la misma: `if`, la condición y **dos puntos** `:`; abajo, el bloque **con sangría**. Cuando el código vuelve a la izquierda, el bloque terminó: por eso `print("Fin del script")` se ejecuta siempre.

## La sangría es obligatoria

En Python, la sangría (los espacios al principio del renglón) **no es decoración**: es lo que indica qué código está adentro del `if`. Si te la olvidás:

```python title="sangria.py"
edad = 20

if edad >= 18:
print("Podés registrarte")
```

```text
IndentationError: expected an indented block after 'if' statement on line 3
```

La convención es usar **4 espacios**. VS Code los pone solo cuando apretás <kbd>Enter</kbd> después de los dos puntos; lo que no tenés que hacer es mezclar espacios con tabulaciones.

## `else`: qué hacer si no se cumple

```python title="else.py"
stock = 0

if stock > 0:
    print("Disponible")
else:
    print("Sin stock")
```

Muestra `Sin stock`. `else` no lleva condición: se ejecuta cuando la del `if` es `False`. Siempre se ejecuta **uno** de los dos bloques, nunca ambos.

## `elif`: varias opciones

Cuando hay más de dos caminos, usás `elif` (abreviatura de *else if*):

```python title="elif.py"
codigo = 404

if codigo == 200:
    print("Todo bien")
elif codigo == 404:
    print("No encontrado")
elif codigo >= 500:
    print("Error del servidor")
else:
    print("Otro código")
```

Muestra `No encontrado`. Python revisa las condiciones **de arriba hacia abajo** y ejecuta el bloque de la **primera** que sea `True`. Las demás ni las mira. El `else` del final es opcional.

## Combinar condiciones: `and`, `or`, `not`

```python title="logicos.py"
edad = 25
tiene_cuenta = True
es_admin = False

if edad >= 18 and tiene_cuenta:
    print("Puede comprar")

if es_admin or edad >= 21:
    print("Puede ver la sección")

if not es_admin:
    print("No es administrador")
```

| Operador | Es `True` cuando... |
| --- | --- |
| `a and b` | ...los dos son `True`. |
| `a or b` | ...al menos uno es `True`. |
| `not a` | ...`a` es `False`. |

## Valores que cuentan como falsos

No hace falta que la condición sea un `bool`. Python considera **falsos** estos valores:

- `0` y `0.0`
- `""` (texto vacío)
- `[]` y `{}` (lista y diccionario vacíos)
- `None`

Todo lo demás cuenta como verdadero. Esto permite escribir condiciones cortas:

```python title="verdad.py"
carrito = []
nombre = ""
descuento = None

if not carrito:
    print("El carrito está vacío")

if not nombre:
    print("Falta el nombre")

if descuento is None:
    print("Sin descuento")
```

Para preguntar por `None`, la forma recomendada es `is None` (o `is not None`), no `== None`.

!!! warning "Cuidado con el 0"

    `if not descuento:` también sería verdadero si `descuento` fuera `0`. Si `0` es un valor válido y lo que querés saber es si "no hay valor", usá `is None`.

## Condicional en una línea

Para elegir entre dos valores existe una forma corta:

```python title="en_linea.py"
stock = 3
estado = "Disponible" if stock > 0 else "Sin stock"
print(estado)  # Disponible
```

Se lee: "`estado` vale `"Disponible"` si `stock > 0`; si no, `"Sin stock"`". Usala solo para casos simples.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | JavaScript | Python |
    | --- | --- |
    | `if (x) { ... }` | `if x:` + bloque con sangría |
    | `else if` | `elif` |
    | `&&`, `||`, `!` | `and`, `or`, `not` |
    | `===` | `==` |
    | `x === null` | `x is None` |
    | `cond ? a : b` | `a if cond else b` |

    Ojo: en JavaScript un array vacío `[]` es **verdadero**; en Python una lista vacía es **falsa**. Y en Python no hay `===`: `==` ya compara sin conversiones raras (`"5" == 5` da `False`).

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. En los templates de Django existe `{% if %} ... {% elif %} ... {% else %} ... {% endif %}`, que imita esta misma lógica, pero en Python no hay `endif`: el bloque termina cuando termina la sangría.

??? example "Ver comparación en Spring Boot ⬇️"

    | Java | Python |
    | --- | --- |
    | `if (x > 0) { ... }` | `if x > 0:` + bloque con sangría |
    | `else if` | `elif` |
    | `&&`, `||`, `!` | `and`, `or`, `not` |
    | `x == null` | `x is None` |
    | `cond ? a : b` | `a if cond else b` |

    En Java la condición tiene que ser un `boolean`: `if (lista)` no compila. En Python podés escribir `if lista:` y significa "si la lista no está vacía".

## Mini ejercicio

¿Qué muestra este script? ¿Y si `total` fuera `6000`?

```python title="ejercicio.py"
total = 3500
es_socio = True

if total >= 5000:
    envio = 0
elif es_socio:
    envio = 500
else:
    envio = 1200

print(f"Envío: ${envio}")
```

??? success "Ver solución"

    Muestra `Envío: $500`. La primera condición (`3500 >= 5000`) es `False`, así que pasa al `elif`; `es_socio` es `True`, así que `envio` vale `500` y el `else` ni se mira.

    Con `total = 6000` muestra `Envío: $0`: la primera condición ya es `True` y las demás no se evalúan, aunque `es_socio` también sea `True`.

---

**Siguiente:** [Bucles →](08-bucles.md)
