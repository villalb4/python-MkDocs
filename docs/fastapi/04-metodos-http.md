# Rutas con métodos HTTP (`GET`, `POST`, `PUT`, `DELETE`)

!!! info "En esta página"

    **Qué vas a aprender:** qué significa cada método HTTP, cómo definir rutas para cada uno en FastAPI y cómo probar los que el navegador no puede hacer, usando `curl`.

    **Qué necesitás:** [Servidor Uvicorn y recarga automática](03-uvicorn.md).

## Los cuatro métodos principales

En la primera página del capítulo viste que cada petición HTTP tiene un **método** que dice qué querés hacer:

| Método | Para qué | ¿Modifica datos? |
| --- | --- | --- |
| `GET` | Leer | No |
| `POST` | Crear algo nuevo | Sí |
| `PUT` | Modificar algo que ya existe | Sí |
| `DELETE` | Borrar | Sí |

La regla más importante: **`GET` nunca debe modificar nada.** Los navegadores, buscadores y otras herramientas asumen que pueden repetir un `GET` sin consecuencias.

También existe `PATCH`, para modificar solo una parte de algo. Se usa menos y funciona igual que los demás.

## Un decorador por método

En FastAPI hay un decorador para cada método: `@app.get`, `@app.post`, `@app.put` y `@app.delete`. Actualizá tu `main.py`:

```python title="main.py"
from fastapi import FastAPI

app = FastAPI()

tareas = [
    {"id": 1, "titulo": "Comprar pan", "hecha": False},
    {"id": 2, "titulo": "Estudiar FastAPI", "hecha": True},
]


@app.get("/tareas")
def listar_tareas():
    return tareas


@app.post("/tareas")
def crear_tarea():
    nueva = {"id": len(tareas) + 1, "titulo": "Tarea de ejemplo", "hecha": False}
    tareas.append(nueva)
    return nueva


@app.put("/tareas/{tarea_id}")
def completar_tarea(tarea_id: int):
    for tarea in tareas:
        if tarea["id"] == tarea_id:
            tarea["hecha"] = True
            return tarea
    return {"error": "no existe"}


@app.delete("/tareas")
def borrar_tareas():
    tareas.clear()
    return {"mensaje": "Se borraron todas las tareas"}
```

Fijate en tres cosas:

- **La misma URL con distintos métodos** son rutas distintas: `GET /tareas`, `POST /tareas` y `DELETE /tareas` ejecutan funciones diferentes.
- Por ahora, `crear_tarea` agrega siempre la misma tarea de ejemplo. En dos páginas vas a aprender a mandarle los datos de la tarea nueva.
- `{tarea_id}` en la URL de `completar_tarea` es un **parámetro de ruta**: la parte de la URL que cambia (`/tareas/1`, `/tareas/2`) llega a la función como argumento. Es el tema de la próxima página; por ahora alcanza con eso.

`tareas.clear()` vacía la lista. Y como `tareas` es una lista y los diccionarios se modifican en el lugar, los cambios se mantienen entre peticiones (hasta que el servidor se reinicie).

## Probar con `curl`

El navegador solo hace `GET` cuando escribís una URL. Para los otros métodos, usá **`curl`**, un programa de terminal que hace peticiones HTTP. Ya viene instalado en macOS, Linux y Windows 10 o superior.

Con el servidor prendido (`uv run fastapi dev main.py`), abrí **otra** terminal y probá:

=== "macOS / Linux"

    ```bash
    curl http://127.0.0.1:8000/tareas
    curl -X POST http://127.0.0.1:8000/tareas
    curl -X PUT http://127.0.0.1:8000/tareas/1
    curl -X DELETE http://127.0.0.1:8000/tareas
    ```

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://127.0.0.1:8000/tareas
    curl.exe -X POST http://127.0.0.1:8000/tareas
    curl.exe -X PUT http://127.0.0.1:8000/tareas/1
    curl.exe -X DELETE http://127.0.0.1:8000/tareas
    ```

    En PowerShell escribí `curl.exe` y no `curl`: en algunas versiones, `curl` es un alias de otro comando que funciona distinto.

`-X` indica el método; sin `-X`, `curl` hace un `GET`. Las respuestas, en orden:

```text
[{"id":1,"titulo":"Comprar pan","hecha":false},{"id":2,"titulo":"Estudiar FastAPI","hecha":true}]
{"id":3,"titulo":"Tarea de ejemplo","hecha":false}
{"id":1,"titulo":"Comprar pan","hecha":true}
{"mensaje":"Se borraron todas las tareas"}
```

Si después hacés otro `GET`, la respuesta es `[]`: la lista quedó vacía. Y en la terminal del servidor vas a ver cada petición con su método:

```text
INFO:     127.0.0.1:62202 - "POST /tareas HTTP/1.1" 200 OK
INFO:     127.0.0.1:62204 - "PUT /tareas/1 HTTP/1.1" 200 OK
INFO:     127.0.0.1:62206 - "DELETE /tareas HTTP/1.1" 200 OK
```

!!! tip "Hay una forma más cómoda"

    Escribir `curl` sirve para entender qué pasa, pero FastAPI genera una página web donde podés probar todas tus rutas con botones, en `http://127.0.0.1:8000/docs`. La vas a ver en detalle más adelante en este capítulo; si querés, ya podés curiosear.

## Método equivocado: `405`

Si pedís una URL que existe pero con un método que no tiene ruta, FastAPI responde con el código `405`:

```bash
curl -i -X PATCH http://127.0.0.1:8000/tareas
```

```http
HTTP/1.1 405 Method Not Allowed
content-type: application/json

{"detail":"Method Not Allowed"}
```

`-i` le pide a `curl` que muestre también el código de estado y los encabezados de la respuesta. Compará con el `404` de la página anterior: `404` es "esa URL no existe"; `405` es "la URL existe, pero no con ese método".

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | Express | FastAPI |
    | --- | --- |
    | `app.get("/tareas", ...)` | `@app.get("/tareas")` |
    | `app.post("/tareas", ...)` | `@app.post("/tareas")` |
    | `app.put("/tareas/:id", ...)` | `@app.put("/tareas/{tarea_id}")` |
    | `app.delete("/tareas", ...)` | `@app.delete("/tareas")` |

    Igual que en Express, la misma URL con distinto método es otra ruta. Los parámetros se escriben con llaves (`{tarea_id}`) en vez de dos puntos (`:id`).

??? example "Ver comparación en Django ⬇️"

    En Django, una view recibe todos los métodos y adentro preguntás `if request.method == "POST":`, o usás `@require_http_methods(["POST"])`. En Django REST Framework, las acciones de un `ViewSet` (`list`, `create`, `update`, `destroy`) se mapean a métodos. En FastAPI cada método tiene su propia función con su propio decorador, sin `if`.

??? example "Ver comparación en Spring Boot ⬇️"

    | Spring Boot | FastAPI |
    | --- | --- |
    | `@GetMapping("/tareas")` | `@app.get("/tareas")` |
    | `@PostMapping("/tareas")` | `@app.post("/tareas")` |
    | `@PutMapping("/tareas/{id}")` | `@app.put("/tareas/{tarea_id}")` |
    | `@DeleteMapping("/tareas")` | `@app.delete("/tareas")` |

    Hasta las llaves del parámetro de ruta son iguales.

## Mini ejercicio

Agregá una ruta `DELETE /tareas/hechas` que saque de la lista solo las tareas con `"hecha": True` y devuelva cuántas quedaron. Probala con `curl` (si el servidor se recargó, la lista vuelve a tener las dos tareas iniciales).

??? success "Ver solución"

    ```python
    @app.delete("/tareas/hechas")
    def borrar_hechas():
        pendientes = [tarea for tarea in tareas if not tarea["hecha"]]
        tareas.clear()
        tareas.extend(pendientes)
        return {"quedan": len(tareas)}
    ```

    ```bash
    curl -X DELETE http://127.0.0.1:8000/tareas/hechas
    ```

    Con la lista inicial, responde `{"quedan":1}`: solo queda "Comprar pan". `tareas.extend(lista)` agrega todos los elementos de otra lista. Vaciamos y volvemos a llenar `tareas` en vez de hacer `tareas = pendientes`, porque esto último crearía una variable nueva dentro de la función y la lista de afuera no cambiaría.

---

**Siguiente:** [Parámetros de ruta y de query →](05-parametros.md)
