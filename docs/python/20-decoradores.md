# Decoradores

!!! info "En esta página"

    **Qué vas a aprender:** qué hace una línea como `@app.get("/tareas")` arriba de una función, y cómo leer cualquier decorador que te encuentres. No vas a escribir decoradores propios: alcanza con entender qué hacen.

    **Qué necesitás:** [`async` / `await`](19-async-await.md).

## Qué es un decorador

Un **decorador** es la línea que empieza con `@` justo arriba de un `def`:

```python
@app.get("/tareas")
def listar_tareas():
    return [{"titulo": "Comprar pan"}]
```

Le da a la función un **comportamiento extra** sin cambiar lo que tiene adentro. En FastAPI es lo que convierte una función común en una ruta de la API, así que vas a escribir uno arriba de cada ruta.

Por dentro, un decorador es una función que **recibe tu función** y hace algo con ella. Hay dos usos típicos: **envolverla** para cambiar cómo se comporta, o **registrarla** en algún lado.

## Un decorador que envuelve: `@cache`

La biblioteca estándar trae `cache`, que hace que una función **recuerde** sus resultados:

```python title="cache.py"
import time
from functools import cache


@cache
def cotizacion_dolar():
    print("Consultando el servicio...")
    time.sleep(1)
    return 1050.0


print(cotizacion_dolar())  # Consultando el servicio... / 1050.0
print(cotizacion_dolar())  # 1050.0 (al instante, sin consultar)
```

La primera llamada ejecuta la función: muestra el mensaje y tarda 1 segundo. La segunda devuelve el resultado guardado al instante, **sin ejecutar** el cuerpo. No cambiaste ni un renglón de `cotizacion_dolar`: el comportamiento nuevo lo agregó `@cache`.

## Qué significa la `@`

Esta es la clave para leer cualquier decorador. Escribir:

```python
@cache
def cotizacion_dolar():
    ...
```

es exactamente lo mismo que:

```python title="equivalente.py"
def cotizacion_dolar():
    ...


cotizacion_dolar = cache(cotizacion_dolar)
```

Es decir: se define la función, se la pasa a `cache`, y lo que `cache` devuelve **reemplaza** a la original bajo el mismo nombre. La `@` es solo una forma más cómoda y legible de escribir eso, pegada a la función.

## Un decorador que registra: `@app.get`

Los decoradores de FastAPI son del otro tipo: **anotan** tu función en una lista de rutas y te la devuelven sin cambios.

```python title="rutas.py"
from fastapi import FastAPI

app = FastAPI()


@app.get("/tareas")
def listar_tareas():
    return [{"titulo": "Comprar pan"}]


@app.post("/tareas")
def crear_tarea():
    return {"ok": True}


for ruta in app.routes:
    if ruta.path == "/tareas":
        print(ruta.methods, ruta.name)

print(listar_tareas())  # [{'titulo': 'Comprar pan'}]
```

```text
{'GET'} listar_tareas
{'POST'} crear_tarea
[{'titulo': 'Comprar pan'}]
```

Podés probarlo con `uv run --with fastapi rutas.py`. Después de los decoradores, `app` tiene registradas las dos funciones: una para `GET /tareas` y otra para `POST /tareas`. Cuando llegue un pedido, FastAPI busca en esa lista qué función ejecutar. Y la función sigue funcionando normalmente si la llamás a mano.

### Decoradores con paréntesis

`@cache` va solo; `@app.get("/tareas")` lleva argumentos. Se lee en dos pasos:

1. `app.get("/tareas")` se ejecuta primero y **devuelve un decorador** configurado para esa ruta.
2. Ese decorador recibe la función, como siempre.

En otras palabras, es como escribir `listar_tareas = app.get("/tareas")(listar_tareas)`. No hace falta que lo memorices; con saber que "lo de los paréntesis configura al decorador" alcanza.

## Otros decoradores que vas a ver

| Decorador | Qué hace |
| --- | --- |
| `@app.get(...)`, `@app.post(...)` | Registran una ruta en FastAPI (Capítulo 2). |
| `@router.get(...)` | Lo mismo, pero en un `APIRouter` (Capítulo 2). |
| `@property` | Dentro de una clase, permite usar un método como si fuera un atributo, sin paréntesis. |
| `@cache` | Guarda los resultados de una función. |
| `@pytest.fixture` | Marca una función como preparación para los tests (Capítulo 6). |

Por ejemplo, con `@property`:

```python title="property.py"
class Producto:
    def __init__(self, precio):
        self.precio = precio

    @property
    def precio_con_iva(self):
        return self.precio * 1.21


teclado = Producto(1000)
print(teclado.precio_con_iva)  # 1210.0
```

`precio_con_iva` es un método, pero gracias a `@property` se usa como un atributo: `teclado.precio_con_iva`, sin `()`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Express no hay decoradores: registrás la ruta llamando a un método y pasándole la función:

    ```javascript
    app.get("/tareas", (req, res) => res.json([{ titulo: "Comprar pan" }]))
    ```

    Es lo mismo que hace `@app.get("/tareas")`, pero en Python la función se define aparte, con nombre, y el decorador la registra. (TypeScript tiene decoradores con `@`, que usa por ejemplo NestJS, con una idea muy parecida.)

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. Seguramente usaste `@login_required`, `@require_POST` o `@admin.register(Tarea)`. Los dos primeros envuelven la view; el último la registra en el admin, como `@app.get` registra una ruta.

??? example "Ver comparación en Spring Boot ⬇️"

    Los decoradores **se parecen** a las anotaciones de Java y cumplen un rol similar:

    | Spring Boot | FastAPI |
    | --- | --- |
    | `@GetMapping("/tareas")` | `@app.get("/tareas")` |
    | `@PostMapping("/tareas")` | `@app.post("/tareas")` |
    | `@Cacheable` | `@cache` |

    La diferencia de fondo: una anotación de Java es una **etiqueta** que el framework lee después. Un decorador de Python es **código que se ejecuta** en el momento en que se define la función y puede reemplazarla por otra.

## Mini ejercicio

¿Qué muestra este script?

```python title="ejercicio.py"
from functools import cache


@cache
def doble(numero):
    print(f"Calculando el doble de {numero}")
    return numero * 2


print(doble(5))
print(doble(5))
print(doble(7))
```

??? success "Ver solución"

    ```text
    Calculando el doble de 5
    10
    10
    Calculando el doble de 7
    14
    ```

    `@cache` recuerda un resultado **por cada argumento distinto**. La segunda vez que se pide `doble(5)` devuelve el `10` guardado sin ejecutar la función (por eso no aparece el mensaje). `doble(7)` es un argumento nuevo, así que sí se calcula.

---

**Siguiente:** [Entornos virtuales y por qué existen →](21-entornos-virtuales.md)
