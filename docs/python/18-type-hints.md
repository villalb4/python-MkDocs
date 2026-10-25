# Type hints

!!! info "En esta página"

    **Qué vas a aprender:** cómo anotar el tipo de las variables, los parámetros y lo que devuelve una función, y por qué en FastAPI esas anotaciones son mucho más que decoración.

    **Qué necesitás:** [Clases básicas](17-clases-basicas.md).

## Qué son

Los **type hints** (anotaciones de tipo) indican de qué tipo es cada cosa:

```python title="basico.py"
def calcular_total(precio: int, cantidad: int) -> int:
    return precio * cantidad


nombre: str = "Teclado"
total = calcular_total(45000, 2)
print(f"{nombre}: {total}")  # Teclado: 90000
```

- Después de un parámetro o variable: dos puntos y el tipo (`precio: int`).
- Después de los paréntesis de la función: una flecha y el tipo que devuelve (`-> int`).
- Con valor por defecto, el tipo va antes del `=`: `cantidad: int = 1`.

## Python no los hace cumplir

Esto sorprende al principio: si pasás un tipo equivocado, Python **igual ejecuta** el código.

```python title="no_obliga.py"
def calcular_total(precio: int, cantidad: int) -> int:
    return precio * cantidad


print(calcular_total("ja", 3))  # jajaja
```

Los type hints son **información**, no reglas. Entonces, ¿para qué sirven?

1. **El editor te ayuda.** VS Code (con la extensión de Python) los usa para autocompletar y para subrayar en rojo `calcular_total("ja", 3)` antes de que lo ejecutes.
2. **El código se explica solo.** `def buscar(id: int) -> Tarea | None` dice qué recibe y qué devuelve sin leer el cuerpo.
3. **Algunas librerías sí los usan.** Y entre ellas están justamente FastAPI y Pydantic.

## Listas y diccionarios

Para colecciones, el tipo de los elementos va entre corchetes:

```python title="colecciones.py"
def promedio(numeros: list[float]) -> float:
    return sum(numeros) / len(numeros)


def contar(tareas: list[dict[str, bool]]) -> dict[str, int]:
    hechas = len([t for t in tareas if t["hecha"]])
    return {"hechas": hechas, "pendientes": len(tareas) - hechas}


print(promedio([8.5, 9, 7]))  # 8.166666666666666
print(contar([{"hecha": True}, {"hecha": False}]))  # {'hechas': 1, 'pendientes': 1}
```

| Anotación | Significa |
| --- | --- |
| `list[str]` | Lista de strings. |
| `dict[str, int]` | Diccionario con claves `str` y valores `int`. |
| `list[dict[str, bool]]` | Lista de diccionarios con claves `str` y valores `bool`. |

Fijate que `promedio` acepta `9` y `7`, que son `int`: donde se pide un `float`, un `int` también vale.

## Valores que pueden faltar: `| None`

Cuando algo puede ser de un tipo **o** `None`, se escribe con una barra vertical:

```python title="opcional.py"
def buscar_email(usuarios: dict[str, str], nombre: str) -> str | None:
    return usuarios.get(nombre)


def saludar(nombre: str, apodo: str | None = None) -> str:
    if apodo is None:
        return f"Hola, {nombre}"
    return f"Hola, {apodo}"


usuarios = {"ana": "ana@ejemplo.com"}
print(buscar_email(usuarios, "ana"))  # ana@ejemplo.com
print(buscar_email(usuarios, "juan"))  # None
print(saludar("Ana"))  # Hola, Ana
print(saludar("Ana", apodo="Anita"))  # Hola, Anita
```

`str | None = None` es la forma de decir "parámetro opcional de texto". Lo vas a escribir muchísimo en FastAPI. En código más viejo vas a ver `Optional[str]`, que significa lo mismo.

## Tus clases también son tipos

```python title="clases.py"
class Tarea:
    def __init__(self, titulo: str) -> None:
        self.titulo = titulo
        self.hecha = False


def pendientes(tareas: list[Tarea]) -> list[str]:
    return [tarea.titulo for tarea in tareas if not tarea.hecha]
```

`-> None` indica que la función no devuelve nada. Y como `tareas` es una `list[Tarea]`, el editor sabe que `tarea` tiene `.titulo` y te lo autocompleta.

## Por qué en FastAPI son clave

Pydantic **lee** los type hints de una clase y los usa para validar y convertir datos:

```python title="pydantic_demo.py"
from pydantic import BaseModel


class Producto(BaseModel):
    nombre: str
    precio: int
    en_stock: bool = True


producto = Producto(nombre="Teclado", precio="45000")
print(producto.precio + 1)  # 45001

Producto(nombre="Mouse", precio="caro")
```

Podés probarlo sin instalar nada a mano con `uv run --with pydantic pydantic_demo.py`. El texto `"45000"` se convirtió en el número `45000`, porque el hint dice `int`. Y `"caro"` no se puede convertir, así que Pydantic lanza un error claro:

```text
pydantic_core._pydantic_core.ValidationError: 1 validation error for Producto
precio
  Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='caro', input_type=str]
```

FastAPI hace lo mismo con los parámetros de tus rutas: si escribís `def listar(limite: int = 10)`, convierte el `?limite=5` de la URL en un `int` y responde con un error si alguien manda letras. En esta guía vas a anotar los tipos **siempre**.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Los type hints son muy parecidos a TypeScript:

    | TypeScript | Python |
    | --- | --- |
    | `function total(precio: number): number` | `def total(precio: int) -> int:` |
    | `string[]` | `list[str]` |
    | `Record<string, number>` | `dict[str, int]` |
    | `string | null` | `str | None` |
    | `apodo?: string` | `apodo: str | None = None` |

    La diferencia: TypeScript **no compila** si los tipos no coinciden; Python ejecuta igual. Y en TypeScript los tipos desaparecen al ejecutar, mientras que en Python librerías como Pydantic los leen en tiempo de ejecución para validar, algo que en Node necesitarías hacer aparte con una librería como Zod.

??? example "Ver comparación en Django ⬇️"

    Django nació antes de los type hints y casi no los usa: los tipos de un modelo se declaran con campos (`models.CharField(...)`), y la validación de formularios, con `forms`. FastAPI va por el otro camino: los type hints **son** la declaración de tipos y la validación.

??? example "Ver comparación en Spring Boot ⬇️"

    En Java los tipos son obligatorios y el compilador los verifica. En Python son opcionales y no se verifican al ejecutar, pero la sintaxis cumple un rol parecido:

    | Java | Python |
    | --- | --- |
    | `public int total(int precio)` | `def total(precio: int) -> int:` |
    | `List<String>` | `list[str]` |
    | `Map<String, Integer>` | `dict[str, int]` |
    | `Optional<String>` o un valor `null` | `str | None` |
    | `void` | `-> None` |

    Lo que en Spring hacen `@Valid` y las anotaciones como `@NotNull`, en FastAPI lo hace Pydantic a partir de los type hints.

## Mini ejercicio

Agregale type hints a estas dos funciones:

```python
def aplicar_descuento(precio, porcentaje=10):
    return precio * (1 - porcentaje / 100)


def buscar_producto(productos, nombre):
    for producto in productos:
        if producto["nombre"] == nombre:
            return producto
    return None
```

??? success "Ver solución"

    ```python
    def aplicar_descuento(precio: float, porcentaje: float = 10) -> float:
        return precio * (1 - porcentaje / 100)


    def buscar_producto(productos: list[dict[str, str]], nombre: str) -> dict[str, str] | None:
        for producto in productos:
            if producto["nombre"] == nombre:
                return producto
        return None
    ```

    `buscar_producto` puede no encontrar nada, por eso devuelve `dict[str, str] | None`. Si pusiste `int` en vez de `float` para el precio, también es razonable: lo importante es que el tipo describa lo que realmente recibe.

---

**Siguiente:** [`async` / `await` →](19-async-await.md)
