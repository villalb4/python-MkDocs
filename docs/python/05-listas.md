# Listas

!!! info "En esta página"

    **Qué vas a aprender:** cómo guardar varios datos en orden con una lista, cómo leerlos por posición y cómo agregar, sacar y ordenar elementos.

    **Qué necesitás:** [Strings y f-strings](04-strings-y-f-strings.md).

## Qué es una lista

Una **lista** guarda varios valores en orden, dentro de una sola variable. Se escribe con corchetes `[]` y los elementos separados por comas:

```python title="crear.py"
tareas = ["Comprar pan", "Estudiar Python", "Llamar a Juan"]
precios = [1500, 3200.5, 800]
vacia = []

print(tareas)  # ['Comprar pan', 'Estudiar Python', 'Llamar a Juan']
print(len(tareas))  # 3
print(len(vacia))  # 0
```

`len(...)`, la misma función que usaste con los textos, te dice cuántos elementos tiene.

En una API, las listas aparecen todo el tiempo: la lista de tareas, de usuarios o de productos que devuelve una ruta.

## Leer elementos por posición

Cada elemento tiene un **índice**: su posición. El primero es el **0**, no el 1.

```python title="indices.py"
tareas = ["Comprar pan", "Estudiar Python", "Llamar a Juan"]

print(tareas[0])  # Comprar pan
print(tareas[2])  # Llamar a Juan
print(tareas[-1])  # Llamar a Juan

tareas[1] = "Estudiar FastAPI"
print(tareas)  # ['Comprar pan', 'Estudiar FastAPI', 'Llamar a Juan']
```

- Los índices negativos cuentan desde el final: `-1` es el último, `-2` el anteúltimo.
- Con `lista[índice] = valor` reemplazás un elemento.

Si pedís una posición que no existe, Python da error:

```python title="error.py"
tareas = ["Comprar pan", "Estudiar Python", "Llamar a Juan"]
print(tareas[3])
```

```text
IndexError: list index out of range
```

Hay 3 elementos, así que los índices válidos son `0`, `1` y `2`.

## Agregar y sacar elementos

```python title="modificar.py"
tareas = ["Comprar pan"]

tareas.append("Lavar los platos")
print(tareas)  # ['Comprar pan', 'Lavar los platos']

tareas.insert(0, "Despertarse")
print(tareas)  # ['Despertarse', 'Comprar pan', 'Lavar los platos']

tareas.remove("Comprar pan")
print(tareas)  # ['Despertarse', 'Lavar los platos']

ultima = tareas.pop()
print(ultima)  # Lavar los platos
print(tareas)  # ['Despertarse']
```

| Método | Qué hace |
| --- | --- |
| `.append(x)` | Agrega `x` al final. |
| `.insert(i, x)` | Agrega `x` en la posición `i`. |
| `.remove(x)` | Saca el primer elemento igual a `x`. Si no está, da `ValueError`. |
| `.pop()` | Saca el último elemento y lo devuelve. |

!!! warning "Las listas sí cambian"

    En la página anterior viste que los métodos de los strings **no** modifican el original. Con las listas es al revés: `append`, `insert`, `remove` y `pop` modifican la lista misma. Por eso no hace falta escribir `tareas = tareas.append(...)` (y de hecho estaría mal).

Para buscar, `in` funciona igual que con los textos: `"azul" in ["rojo", "azul"]` da `True`. Y `.index(x)` devuelve la posición de `x` (o error, si no está).

## Rebanadas: una parte de la lista

Con `[inicio:fin]` obtenés una lista nueva con una parte de la original. El elemento en `fin` **no** se incluye:

```python title="rebanadas.py"
numeros = [10, 20, 30, 40, 50]

print(numeros[1:3])  # [20, 30]
print(numeros[:2])  # [10, 20]
print(numeros[2:])  # [30, 40, 50]
```

Si omitís el inicio, arranca desde el principio; si omitís el fin, llega hasta el final. Esto también funciona con strings: `"Python"[:2]` da `"Py"`.

## Ordenar y resumir

```python title="ordenar.py"
precios = [3200, 800, 1500]

ordenados = sorted(precios)
print(ordenados)  # [800, 1500, 3200]
print(precios)  # [3200, 800, 1500] (no cambió)

precios.sort(reverse=True)
print(precios)  # [3200, 1500, 800]

print(sum(precios), min(precios), max(precios))  # 5500 800 3200
```

- `sorted(lista)` devuelve una lista **nueva** ordenada y deja la original como estaba.
- `lista.sort()` ordena **la misma** lista. Con `reverse=True`, de mayor a menor.
- `sum`, `min` y `max` hacen lo que dicen.

## De texto a lista y de vuelta

Dos métodos de los strings que trabajan con listas:

```python title="texto.py"
etiquetas = "python,fastapi,docker".split(",")
print(etiquetas)  # ['python', 'fastapi', 'docker']

print(" | ".join(etiquetas))  # python | fastapi | docker
```

`.split(separador)` corta un texto en partes; `separador.join(lista)` las une.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Una lista de Python es como un array de JavaScript:

    | JavaScript | Python |
    | --- | --- |
    | `arr.length` | `len(lista)` |
    | `arr.push(x)` | `lista.append(x)` |
    | `arr.pop()` | `lista.pop()` |
    | `arr.includes(x)` | `x in lista` |
    | `arr.slice(1, 3)` | `lista[1:3]` |
    | `arr.at(-1)` | `lista[-1]` |
    | `arr.join(", ")` | `", ".join(lista)` |

    Una diferencia: en JavaScript, `arr[10]` en un array de 3 elementos da `undefined`. En Python da `IndexError`.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. En Django vas a ver los *QuerySets*, que se parecen a una lista (se pueden recorrer, indexar y rebanar) pero no lo son: son consultas a la base que se ejecutan recién cuando las usás. Con SQLAlchemy pasa algo parecido, lo vas a ver en el Capítulo 3.

??? example "Ver comparación en Spring Boot ⬇️"

    Una lista de Python se parece a un `ArrayList` de Java, pero sin declarar el tipo de los elementos:

    | Java | Python |
    | --- | --- |
    | `new ArrayList<>(List.of("a", "b"))` | `["a", "b"]` |
    | `lista.size()` | `len(lista)` |
    | `lista.get(0)` | `lista[0]` |
    | `lista.add(x)` | `lista.append(x)` |
    | `lista.contains(x)` | `x in lista` |
    | `Collections.sort(lista)` | `lista.sort()` |

## Mini ejercicio

¿Qué muestra este script? Pensalo antes de ejecutarlo.

```python title="ejercicio.py"
compras = ["leche", "pan"]
compras.append("huevos")
compras.insert(1, "café")
compras.remove("pan")

print(compras)
print(len(compras))
print(compras[-1])
```

??? success "Ver solución"

    ```text
    ['leche', 'café', 'huevos']
    3
    huevos
    ```

    Paso a paso: `append` deja `['leche', 'pan', 'huevos']`; `insert(1, "café")` mete café en la posición 1: `['leche', 'café', 'pan', 'huevos']`; `remove("pan")` lo saca. Quedan 3 elementos y el último es `huevos`.

---

**Siguiente:** [Diccionarios →](06-diccionarios.md)
