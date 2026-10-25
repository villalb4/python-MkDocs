# Diccionarios

!!! info "En esta página"

    **Qué vas a aprender:** cómo guardar datos con nombre usando diccionarios, cómo leerlos sin que el programa se rompa y cómo combinarlos con listas.

    **Qué necesitás:** [Listas](05-listas.md).

## Qué es un diccionario

Una lista guarda valores **por posición**. Un **diccionario** los guarda **por nombre**: cada valor tiene una **clave** que lo identifica. Se escribe con llaves `{}` y pares `clave: valor`:

```python title="crear.py"
usuario = {
    "nombre": "Ana",
    "edad": 30,
    "activo": True,
}

print(usuario)  # {'nombre': 'Ana', 'edad': 30, 'activo': True}
print(len(usuario))  # 3
```

- Las claves casi siempre son strings. Los valores pueden ser de cualquier tipo.
- No puede haber dos claves iguales: si repetís una, queda el último valor.
- La coma después del último par es opcional, pero ayuda cuando agregás renglones.

!!! tip "Diccionarios y JSON"

    Fijate que un diccionario se parece muchísimo a un objeto JSON. Por eso son tan importantes para un backend: lo que tu API recibe y devuelve en JSON, en Python lo vas a manejar como diccionarios (o como modelos, que vas a ver en el Capítulo 2).

## Leer valores

Con corchetes y la clave, o con `.get()`:

```python title="leer.py"
usuario = {"nombre": "Ana", "edad": 30}

print(usuario["nombre"])  # Ana
print(usuario.get("edad"))  # 30
print(usuario.get("email"))  # None
print(usuario.get("email", "sin email"))  # sin email
```

La diferencia aparece cuando la clave **no existe**. Con corchetes, el programa se corta:

```python title="error.py"
usuario = {"nombre": "Ana", "edad": 30}
print(usuario["email"])
```

```text
KeyError: 'email'
```

Con `.get()`, en cambio, obtenés `None`, o el valor por defecto que le pases como segundo argumento.

**Regla práctica:** usá corchetes cuando la clave tiene que estar sí o sí; usá `.get()` cuando puede faltar.

## Agregar, cambiar y borrar

```python title="modificar.py"
producto = {"nombre": "Teclado", "precio": 45000}

producto["precio"] = 42000
producto["stock"] = 10
print(producto)  # {'nombre': 'Teclado', 'precio': 42000, 'stock': 10}

del producto["stock"]
print(producto)  # {'nombre': 'Teclado', 'precio': 42000}

precio = producto.pop("precio")
print(precio)  # 42000
print(producto)  # {'nombre': 'Teclado'}
```

- `dic[clave] = valor` cambia el valor si la clave existe, y la crea si no existe.
- `del dic[clave]` borra el par.
- `dic.pop(clave)` lo borra y te devuelve el valor.

Igual que las listas, los diccionarios se modifican en el lugar.

## Consultar claves y valores

```python title="consultar.py"
producto = {"nombre": "Teclado", "precio": 45000}

print("precio" in producto)  # True
print(45000 in producto)  # False: "in" busca en las claves

print(list(producto.keys()))  # ['nombre', 'precio']
print(list(producto.values()))  # ['Teclado', 45000]
```

`.keys()` y `.values()` devuelven las claves y los valores. Los envolvemos en `list(...)` para verlos como una lista común. También existe `.items()`, que devuelve los pares; lo vas a usar en la página de bucles.

## Listas de diccionarios y diccionarios anidados

En un backend real casi nunca tenés un diccionario solo. Lo típico es una **lista de diccionarios** (por ejemplo, todas las tareas) o diccionarios **adentro** de otros:

```python title="anidados.py"
tareas = [
    {"id": 1, "titulo": "Comprar pan", "hecha": False},
    {"id": 2, "titulo": "Estudiar Python", "hecha": True},
]

print(tareas[1]["titulo"])  # Estudiar Python

tareas.append({"id": 3, "titulo": "Llamar a Juan", "hecha": False})
print(len(tareas))  # 3

pedido = {
    "id": 101,
    "cliente": {"nombre": "Ana", "email": "ana@ejemplo.com"},
    "productos": ["Teclado", "Mouse"],
}

print(pedido["cliente"]["email"])  # ana@ejemplo.com
print(pedido["productos"][0])  # Teclado
```

Se lee de izquierda a derecha: `tareas[1]["titulo"]` es "de la lista `tareas`, el elemento 1; de ese diccionario, la clave `titulo`".

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Un diccionario cumple el rol de un objeto literal de JavaScript:

    | JavaScript | Python |
    | --- | --- |
    | `{ nombre: "Ana" }` | `{"nombre": "Ana"}` |
    | `user.nombre` o `user["nombre"]` | `user["nombre"]` |
    | `user.email ?? "sin email"` | `user.get("email", "sin email")` |
    | `delete user.email` | `del user["email"]` |
    | `"email" in user` | `"email" in user` |
    | `Object.keys(user)` | `list(user.keys())` |

    Dos diferencias: en Python las claves van **siempre** entre comillas y no existe la notación con punto (`user.nombre` da error). Y una clave que no existe da `KeyError`, no `undefined`.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. Ya los usaste si escribiste `JsonResponse({"ok": True})` o `render(request, "x.html", {"tareas": tareas})`: ese contexto es un diccionario. `request.GET` y `request.POST` también se usan como diccionarios, con `.get()` incluido.

??? example "Ver comparación en Spring Boot ⬇️"

    Un diccionario se parece a un `Map<String, Object>` (por ejemplo, un `HashMap`):

    | Java | Python |
    | --- | --- |
    | `Map.of("nombre", "Ana")` | `{"nombre": "Ana"}` |
    | `map.get("nombre")` | `d["nombre"]` o `d.get("nombre")` |
    | `map.getOrDefault("email", "-")` | `d.get("email", "-")` |
    | `map.put("edad", 30)` | `d["edad"] = 30` |
    | `map.containsKey("edad")` | `"edad" in d` |
    | `map.remove("edad")` | `d.pop("edad")` |

    En Spring Boot normalmente devolvés objetos (DTOs) en vez de mapas. En FastAPI pasa lo mismo: vas a usar modelos de Pydantic, que vas a ver en el Capítulo 2.

## Mini ejercicio

¿Qué muestra este script? Pensalo antes de ejecutarlo.

```python title="ejercicio.py"
tarea = {"titulo": "Comprar pan", "hecha": False}

tarea["hecha"] = True
tarea["prioridad"] = "alta"

print(tarea)
print(tarea.get("vence", "sin fecha"))
```

??? success "Ver solución"

    ```text
    {'titulo': 'Comprar pan', 'hecha': True, 'prioridad': 'alta'}
    sin fecha
    ```

    `hecha` ya existía, así que cambia a `True`; `prioridad` no existía, así que se agrega al final. Como `vence` no está, `.get()` devuelve el valor por defecto en vez de dar error.

---

**Siguiente:** [Condicionales →](07-condicionales.md)
