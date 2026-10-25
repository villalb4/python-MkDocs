# Clases básicas

!!! info "En esta página"

    **Qué vas a aprender:** qué es una clase, cómo crear objetos a partir de ella, qué son `__init__`, `self`, los atributos y los métodos, y qué significa heredar de otra clase.

    **Qué necesitás:** [Manejo de errores](16-manejo-de-errores.md).

## Para qué sirve una clase

Hasta ahora representaste una tarea con un diccionario: `{"titulo": "Comprar pan", "hecha": False}`. Funciona, pero los datos y las funciones que trabajan con ellos (completar la tarea, describirla) quedan separados, y nada impide escribir mal una clave.

Una **clase** es un molde que junta las dos cosas: define qué **datos** tiene algo y qué **acciones** se pueden hacer con él. A partir del molde creás **objetos** (también llamados **instancias**): cada tarea concreta es un objeto de la clase `Tarea`.

No vas a escribir muchas clases propias en esta guía, pero vas a **usar** clases todo el tiempo: los modelos de Pydantic y de SQLAlchemy son clases. Por eso conviene entender cómo funcionan.

## Definir una clase y crear objetos

```python title="clase.py"
class Tarea:
    def __init__(self, titulo, prioridad="media"):
        self.titulo = titulo
        self.prioridad = prioridad
        self.hecha = False


compras = Tarea("Comprar pan")
estudio = Tarea("Estudiar Python", prioridad="alta")

print(compras.titulo)  # Comprar pan
print(compras.prioridad)  # media
print(estudio.prioridad)  # alta
print(estudio.hecha)  # False
```

Vamos por partes:

- `class Tarea:` define la clase. Por convención, los nombres de clases van con **mayúscula inicial** y sin guiones bajos (`Tarea`, `CarritoDeCompras`).
- `__init__` es una función especial que Python ejecuta **cada vez que creás un objeto**. Ahí se cargan sus datos iniciales.
- `Tarea("Comprar pan")` crea un objeto nuevo: se llama como una función, y los argumentos van a `__init__`. Valen los valores por defecto y los argumentos con nombre que ya conocés.
- `self.titulo = titulo` guarda el dato **en el objeto**. Esas variables pegadas al objeto se llaman **atributos**, y se leen con un punto: `compras.titulo`.

## Qué es `self`

`self` es **el objeto que se está creando o usando**. Cuando escribís `Tarea("Comprar pan")`, Python crea un objeto vacío y llama a `__init__` pasándole ese objeto como `self`. Así, `self.titulo = titulo` significa "guardá el título en *este* objeto".

Por eso `compras` y `estudio` tienen cada uno sus propios datos: son objetos distintos hechos con el mismo molde.

## Métodos: funciones del objeto

Las funciones definidas dentro de una clase se llaman **métodos**. Siempre reciben `self` como primer parámetro, para poder trabajar con los datos del objeto:

```python title="metodos.py"
class Tarea:
    def __init__(self, titulo, prioridad="media"):
        self.titulo = titulo
        self.prioridad = prioridad
        self.hecha = False

    def completar(self):
        self.hecha = True

    def describir(self):
        estado = "✓" if self.hecha else " "
        return f"[{estado}] {self.titulo} ({self.prioridad})"


tarea = Tarea("Comprar pan")
print(tarea.describir())  # [ ] Comprar pan (media)

tarea.completar()
print(tarea.describir())  # [✓] Comprar pan (media)

otra = Tarea("Estudiar Python")
print(otra.hecha)  # False
```

Al llamar `tarea.completar()` **no** pasás `self`: Python lo pone solo con el objeto que está antes del punto. Completar `tarea` no afecta a `otra`.

Ya venías usando métodos sin saberlo: `"hola".upper()` y `lista.append(x)` son métodos de las clases `str` y `list`.

## Errores comunes

**Pedir un atributo que no existe:**

```text
AttributeError: 'Tarea' object has no attribute 'descripcion'
```

**Olvidarte `self` en un método** (`def completar():`). Python igual le pasa el objeto, y no tiene dónde recibirlo:

```text
TypeError: Tarea.completar() takes 0 positional arguments but 1 was given
```

!!! note "¿Y si hago `print(tarea)`?"

    Vas a ver algo como `<__main__.Tarea object at 0x10145e360>`: Python no sabe cómo mostrar tu objeto. Se puede personalizar, pero no hace falta: los modelos de Pydantic que vas a usar ya se muestran de forma legible.

## Herencia: una clase basada en otra

Una clase puede **heredar** de otra: recibe todos sus atributos y métodos, y puede agregar los suyos. La clase de la que hereda va entre paréntesis:

```python title="herencia.py"
class Usuario:
    def __init__(self, nombre, email):
        self.nombre = nombre
        self.email = email

    def saludar(self):
        return f"Hola, {self.nombre}"


class Admin(Usuario):
    def puede_borrar(self):
        return True


ana = Admin("Ana", "ana@ejemplo.com")
print(ana.saludar())  # Hola, Ana
print(ana.puede_borrar())  # True
print(isinstance(ana, Usuario))  # True
```

`Admin` no define `__init__` ni `saludar`: los hereda de `Usuario`. `isinstance` confirma que un `Admin` también es un `Usuario`.

!!! tip "La herencia que vas a ver en FastAPI"

    En el Capítulo 2 vas a escribir cosas como:

    ```python
    class Tarea(BaseModel):
        titulo: str
        hecha: bool = False
    ```

    `Tarea` hereda de `BaseModel`, una clase de Pydantic, y con eso obtiene validación, conversión a JSON y mucho más, sin escribir `__init__`. Esa sintaxis con `:` son los *type hints*, el tema de la próxima página.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | JavaScript | Python |
    | --- | --- |
    | `class Tarea { constructor(titulo) { ... } }` | `class Tarea:` + `def __init__(self, titulo):` |
    | `this.titulo = titulo` | `self.titulo = titulo` |
    | `new Tarea("Comprar pan")` | `Tarea("Comprar pan")` (sin `new`) |
    | `completar() { ... }` | `def completar(self):` |
    | `class Admin extends Usuario` | `class Admin(Usuario):` |

    La diferencia más visible: en Python `self` se escribe **explícitamente** como primer parámetro de cada método; en JavaScript `this` está implícito.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. Tus modelos de Django son clases que heredan de `models.Model`, y las *class-based views* heredan de `View`. FastAPI usa la misma idea con `BaseModel` de Pydantic y, para la base de datos, con los modelos de SQLAlchemy.

??? example "Ver comparación en Spring Boot ⬇️"

    | Java | Python |
    | --- | --- |
    | `public Tarea(String titulo) { this.titulo = titulo; }` | `def __init__(self, titulo): self.titulo = titulo` |
    | `private String titulo;` (declarado arriba) | Se crea al asignarlo en `__init__` |
    | `new Tarea("Comprar pan")` | `Tarea("Comprar pan")` |
    | `class Admin extends Usuario` | `class Admin(Usuario):` |

    En Python no hay `private` ni getters y setters obligatorios: se accede directo con `tarea.titulo`. Por convención, un atributo que empieza con `_` (`_interno`) significa "no lo toques desde afuera", pero el lenguaje no lo impide.

## Mini ejercicio

Escribí una clase `Carrito` que empiece con una lista vacía de productos, con un método `agregar(nombre, precio, cantidad=1)` que guarde un diccionario en la lista y un método `total()` que devuelva la suma de `precio * cantidad`. Probala con un teclado de `45000` y dos mouse de `12000`.

??? success "Ver solución"

    ```python
    class Carrito:
        def __init__(self):
            self.productos = []

        def agregar(self, nombre, precio, cantidad=1):
            self.productos.append({"nombre": nombre, "precio": precio, "cantidad": cantidad})

        def total(self):
            suma = 0
            for producto in self.productos:
                suma += producto["precio"] * producto["cantidad"]
            return suma


    carrito = Carrito()
    carrito.agregar("Teclado", 45000)
    carrito.agregar("Mouse", 12000, cantidad=2)
    print(len(carrito.productos))  # 2
    print(carrito.total())  # 69000
    ```

    La lista se crea en `__init__` (y no como valor por defecto de un parámetro, ¿te acordás de la trampa?), así que cada carrito tiene la suya.

---

**Siguiente:** [Type hints →](18-type-hints.md)
