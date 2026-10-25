# Bucles (`for`, `while`)

!!! info "En esta página"

    **Qué vas a aprender:** cómo repetir código con `for` y `while`, cómo recorrer listas y diccionarios, y cómo armar una lista nueva en una sola línea.

    **Qué necesitás:** [Condicionales](07-condicionales.md).

## `for`: hacer algo con cada elemento

Un bucle `for` recorre una lista y ejecuta su bloque **una vez por cada elemento**:

```python title="for.py"
tareas = ["Comprar pan", "Estudiar Python", "Llamar a Juan"]

for tarea in tareas:
    print(f"- {tarea}")

print("Listo")
```

```text
- Comprar pan
- Estudiar Python
- Llamar a Juan
Listo
```

En cada vuelta, la variable `tarea` toma el valor del siguiente elemento. El nombre lo elegís vos; la costumbre es usar el singular de la lista (`tareas` → `tarea`). Igual que en el `if`, el bloque se marca con dos puntos y sangría.

## Acumular y filtrar

Dos usos que vas a repetir todo el tiempo. **Acumular** un resultado:

```python title="acumular.py"
precios = [1500, 3200, 800]
total = 0

for precio in precios:
    total = total + precio

print(total)  # 5500
```

Y **filtrar**, combinando el `for` con un `if`:

```python title="filtrar.py"
tareas = [
    {"titulo": "Comprar pan", "hecha": False},
    {"titulo": "Estudiar Python", "hecha": True},
    {"titulo": "Llamar a Juan", "hecha": False},
]

pendientes = []
for tarea in tareas:
    if not tarea["hecha"]:
        pendientes.append(tarea["titulo"])

print(pendientes)  # ['Comprar pan', 'Llamar a Juan']
```

## `range`: repetir un número de veces

```python title="range.py"
for i in range(3):
    print(i)  # 0, 1, 2

for i in range(1, 4):
    print(i)  # 1, 2, 3
```

`range(3)` genera los números del `0` al `2`; `range(1, 4)`, del `1` al `3`. Igual que en las rebanadas, el último número **no** se incluye.

## Número de vuelta y diccionarios

Si además del elemento necesitás su posición, usá `enumerate`. Y para recorrer un diccionario, `.items()` te da cada clave con su valor. En los dos casos, cada vuelta entrega **dos** valores, que recibís en dos variables separadas por coma:

```python title="enumerate.py"
tareas = ["Comprar pan", "Estudiar Python"]

for numero, tarea in enumerate(tareas, start=1):
    print(f"{numero}. {tarea}")  # 1. Comprar pan / 2. Estudiar Python
```

```python title="dict.py"
producto = {"nombre": "Teclado", "precio": 45000, "stock": 10}

for clave, valor in producto.items():
    print(f"{clave}: {valor}")  # nombre: Teclado / precio: 45000 / stock: 10
```

## Comprensiones de listas

Armar una lista nueva a partir de otra es tan común que Python tiene una forma corta, la **comprensión de lista**:

```python title="comprension.py"
precios = [1500, 3200, 800]

con_iva = [precio * 1.21 for precio in precios]
print(con_iva)  # [1815.0, 3872.0, 968.0]

baratos = [precio for precio in precios if precio < 2000]
print(baratos)  # [1500, 800]
```

Se lee: "una lista con `precio * 1.21` **por cada** `precio` **en** `precios`". El `if` del final, opcional, filtra. Es lo mismo que el `for` con `append` de más arriba, en una línea. Si una comprensión se vuelve difícil de leer, volvé al `for` común.

## `while`: repetir mientras se cumpla algo

`while` repite su bloque **mientras** la condición sea `True`. Sirve cuando no sabés de antemano cuántas vueltas vas a dar:

```python title="while.py"
intentos = 0

while intentos < 3:
    intentos += 1
    print(f"Intento {intentos}")  # Intento 1, Intento 2, Intento 3

print("Sin más intentos")
```

`intentos += 1` es la forma corta de `intentos = intentos + 1`.

!!! warning "Bucles infinitos"

    Si la condición nunca se vuelve `False`, el bucle no termina. Si te pasa, cortalo con <kbd>Ctrl</kbd> + <kbd>C</kbd> en la terminal. En la práctica, `for` se usa mucho más que `while`.

## `break` y `continue`

- `break` **corta** el bucle en el momento.
- `continue` **salta** al siguiente elemento, sin ejecutar el resto del bloque.

```python title="break.py"
usuarios = ["ana", "juan", "luz", "pedro"]

for usuario in usuarios:
    if usuario == "luz":
        print("Encontrada")
        break
    print(f"Revisando {usuario}")  # ana, juan (pedro nunca se revisa)

for numero in range(5):
    if numero % 2 == 0:
        continue
    print(numero)  # 1, 3
```

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | JavaScript | Python |
    | --- | --- |
    | `for (const t of tareas) { ... }` | `for t in tareas:` |
    | `for (let i = 0; i < 3; i++)` | `for i in range(3):` |
    | `tareas.forEach((t, i) => ...)` | `for i, t in enumerate(tareas):` |
    | `Object.entries(obj)` | `obj.items()` |
    | `precios.map(p => p * 1.21)` | `[p * 1.21 for p in precios]` |
    | `precios.filter(p => p < 2000)` | `[p for p in precios if p < 2000]` |

    Ojo: el `for ... in` de JavaScript recorre **claves**; el `for ... in` de Python recorre **elementos**, como el `for ... of`.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. En los templates existe `{% for tarea in tareas %} ... {% endfor %}`, con la misma idea. Y es común recorrer un QuerySet con `for tarea in Tarea.objects.all():`.

??? example "Ver comparación en Spring Boot ⬇️"

    | Java | Python |
    | --- | --- |
    | `for (String t : tareas)` | `for t in tareas:` |
    | `for (int i = 0; i < 3; i++)` | `for i in range(3):` |
    | `map.entrySet()` | `d.items()` |
    | `precios.stream().map(p -> p * 1.21).toList()` | `[p * 1.21 for p in precios]` |
    | `precios.stream().filter(p -> p < 2000).toList()` | `[p for p in precios if p < 2000]` |

    `while`, `break` y `continue` funcionan igual que en Java.

## Mini ejercicio

¿Qué muestra este script?

```python title="ejercicio.py"
productos = [
    {"nombre": "Teclado", "precio": 45000, "stock": 3},
    {"nombre": "Mouse", "precio": 12000, "stock": 0},
    {"nombre": "Monitor", "precio": 250000, "stock": 1},
]

valor_total = 0
for producto in productos:
    if producto["stock"] == 0:
        continue
    print(producto["nombre"])
    valor_total += producto["precio"] * producto["stock"]

print(f"Valor del stock: ${valor_total:,}")
```

??? success "Ver solución"

    ```text
    Teclado
    Monitor
    Valor del stock: $385,000
    ```

    El Mouse tiene stock `0`, así que `continue` lo salta: no se muestra ni se suma. Se acumulan `45000 * 3 = 135000` y `250000 * 1 = 250000`, que dan `385000`.

---

**Siguiente:** [Funciones →](09-funciones.md)
