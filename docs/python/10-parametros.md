# Parámetros, valores por defecto y argumentos con nombre

!!! info "En esta página"

    **Qué vas a aprender:** las distintas formas de pasarle datos a una función: por posición, con valores por defecto y con nombre.

    **Qué necesitás:** [Funciones](09-funciones.md).

## Parámetros y argumentos

Dos palabras que se usan mucho:

- **Parámetro:** la variable que aparece en la definición (`def saludar(nombre):`).
- **Argumento:** el valor que pasás al llamar a la función (`saludar("Ana")`).

En la práctica se mezclan bastante; lo importante es entender cómo se conectan.

## Por posición: el orden importa

Si pasás los argumentos sin nombre, se asignan **en orden**: el primero al primer parámetro, el segundo al segundo.

```python title="posicion.py"
def describir(nombre, precio):
    return f"{nombre}: ${precio}"


print(describir("Teclado", 45000))  # Teclado: $45000
print(describir(45000, "Teclado"))  # 45000: $Teclado
```

Python no se queja si los invertís: simplemente hace lo que le pediste. Lo que sí controla es la **cantidad**:

```python title="cantidad.py"
print(describir("Teclado"))
```

```text
TypeError: describir() missing 1 required positional argument: 'precio'
```

## Valores por defecto

Un parámetro puede tener un valor que se usa **si no le pasás nada**. Eso lo vuelve opcional:

```python title="defecto.py"
def saludar(nombre, saludo="Hola"):
    return f"{saludo}, {nombre}"


print(saludar("Ana"))  # Hola, Ana
print(saludar("Ana", "Buen día"))  # Buen día, Ana
```

Los parámetros con valor por defecto van **después** de los obligatorios. Al revés da error:

```python
def saludar(saludo="Hola", nombre):  # SyntaxError
    ...
```

!!! tip "Esto lo vas a ver en FastAPI"

    En FastAPI, un parámetro con valor por defecto en la función de una ruta se convierte en un parámetro **opcional** de la URL. Por ejemplo, con `def listar_tareas(limite=10):`, la ruta responde tanto a `/tareas` como a `/tareas?limite=5`. Lo vas a ver en el Capítulo 2.

## Argumentos con nombre

Al llamar a la función, podés decir **a qué parámetro** va cada valor escribiendo `parametro=valor`:

```python title="nombre.py"
def crear_usuario(nombre, email, activo=True, es_admin=False):
    return {"nombre": nombre, "email": email, "activo": activo, "es_admin": es_admin}


print(crear_usuario("Ana", "ana@ejemplo.com", es_admin=True))
print(crear_usuario(email="juan@ejemplo.com", nombre="Juan"))
```

```text
{'nombre': 'Ana', 'email': 'ana@ejemplo.com', 'activo': True, 'es_admin': True}
{'nombre': 'Juan', 'email': 'juan@ejemplo.com', 'activo': True, 'es_admin': False}
```

Ventajas:

- **Se lee mejor:** `es_admin=True` dice qué significa ese `True`; un `True` suelto, no.
- **Podés saltear opcionales:** pasaste `es_admin` sin tener que escribir `activo`.
- **El orden deja de importar:** en la segunda llamada, `email` va antes que `nombre`.

La única regla al mezclar: primero los argumentos por posición y después los con nombre. `crear_usuario(nombre="Ana", "ana@ejemplo.com")` da `SyntaxError: positional argument follows keyword argument`.

!!! tip "Cuándo usar nombres"

    Usá argumentos con nombre para los valores `True`/`False`, para los opcionales y siempre que haya más de dos o tres argumentos. Vas a ver que en FastAPI, Pydantic y SQLAlchemy se usan todo el tiempo.

## La trampa de los valores por defecto mutables

Nunca uses una lista o un diccionario como valor por defecto. Mirá lo que pasa:

```python title="mutable.py"
def agregar_tarea(tarea, lista=[]):
    lista.append(tarea)
    return lista


print(agregar_tarea("Comprar pan"))  # ['Comprar pan']
print(agregar_tarea("Estudiar"))  # ['Comprar pan', 'Estudiar'] ¡sorpresa!
```

El valor por defecto se crea **una sola vez**, cuando se define la función, y se reutiliza en cada llamada. Como las listas se modifican en el lugar, se van acumulando cosas. La solución es usar `None` y crear la lista adentro:

```python title="mutable_bien.py"
def agregar_tarea(tarea, lista=None):
    if lista is None:
        lista = []
    lista.append(tarea)
    return lista


print(agregar_tarea("Comprar pan"))  # ['Comprar pan']
print(agregar_tarea("Estudiar"))  # ['Estudiar']
```

!!! note "`*args` y `**kwargs`"

    En código ajeno vas a ver `def f(*args, **kwargs):`. Sirven para aceptar una cantidad variable de argumentos. No los vas a necesitar para escribir tu API; si los ves, pensá "esta función acepta argumentos extra y se los pasa a otra".

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Los valores por defecto existen igual en JavaScript: `function saludar(nombre, saludo = "Hola")`. Lo que no existe son los argumentos con nombre; lo más parecido es pasar un objeto y desestructurarlo:

    ```javascript
    function crearUsuario({ nombre, email, esAdmin = false }) { ... }
    crearUsuario({ nombre: "Ana", email: "ana@ejemplo.com", esAdmin: true })
    ```

    En Python no hace falta el objeto: `crear_usuario(nombre="Ana", email="ana@ejemplo.com", es_admin=True)`. Y JavaScript **no** tiene la trampa de los valores mutables: el valor por defecto se crea de nuevo en cada llamada.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. Ya usaste argumentos con nombre todo el tiempo: `models.CharField(max_length=100, blank=True)` o `Tarea.objects.filter(hecha=False)`.

??? example "Ver comparación en Spring Boot ⬇️"

    Java no tiene valores por defecto ni argumentos con nombre. Para lograr algo parecido se usa **sobrecarga** (varios métodos con el mismo nombre y distintos parámetros) o el patrón *builder*:

    ```java
    String saludar(String nombre) { return saludar(nombre, "Hola"); }
    String saludar(String nombre, String saludo) { return saludo + ", " + nombre; }
    ```

    En Python alcanza con una sola función: `def saludar(nombre, saludo="Hola"):`. Y el `@RequestParam(defaultValue = "10") int limite` de Spring en FastAPI es simplemente `limite: int = 10`.

## Mini ejercicio

¿Qué muestran estas tres llamadas?

```python title="ejercicio.py"
def paginar(elementos, pagina=1, por_pagina=2):
    inicio = (pagina - 1) * por_pagina
    fin = inicio + por_pagina
    return elementos[inicio:fin]


tareas = ["a", "b", "c", "d", "e"]
print(paginar(tareas))
print(paginar(tareas, 2))
print(paginar(tareas, por_pagina=3, pagina=2))
```

??? success "Ver solución"

    ```text
    ['a', 'b']
    ['c', 'd']
    ['d', 'e']
    ```

    1. Sin argumentos extra: página 1 de a 2 → `tareas[0:2]`.
    2. `2` va por posición a `pagina`: página 2 de a 2 → `tareas[2:4]`.
    3. Con nombres, en cualquier orden: página 2 de a 3 → `tareas[3:6]`. Como solo hay 5 elementos, la rebanada llega hasta el final sin dar error.

---

**Siguiente:** [Módulos e `import` →](11-modulos-e-import.md)
