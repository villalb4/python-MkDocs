# Códigos de estado y respuestas de error

!!! info "En esta página"

    **Qué vas a aprender:** qué significan los códigos de estado HTTP, cómo elegir el código de cada ruta y cómo responder errores como "no encontrado" con `HTTPException`.

    **Qué necesitás:** [Validación con Pydantic: modelos](07-pydantic-modelos.md) y [Manejo de errores](../python/16-manejo-de-errores.md).

## Qué es un código de estado

Cada respuesta HTTP trae un número de tres cifras que resume cómo salió la petición. El cliente lo mira **antes** que el cuerpo: así sabe si tiene que mostrar los datos o un mensaje de error. La primera cifra indica la familia:

| Familia | Significa | Culpa de... |
| --- | --- | --- |
| `2xx` | Salió bien | — |
| `4xx` | La petición tiene un problema | El cliente |
| `5xx` | El servidor falló | El servidor (o sea, tu código) |

Los que más vas a usar:

| Código | Nombre | Cuándo |
| --- | --- | --- |
| `200` | OK | Respuesta exitosa con datos. Es el que FastAPI usa por defecto. |
| `201` | Created | Se creó algo (un `POST` exitoso). |
| `204` | No Content | Salió bien y no hay nada que devolver (un `DELETE`). |
| `400` | Bad Request | La petición no tiene sentido por algún motivo de la lógica. |
| `401` | Unauthorized | Falta identificarse (lo vas a ver en el Capítulo 5). |
| `403` | Forbidden | Estás identificado, pero no tenés permiso. |
| `404` | Not Found | El recurso no existe. |
| `409` | Conflict | Choca con algo que ya existe (por ejemplo, un duplicado). |
| `422` | Unprocessable Content | Los datos no pasaron la validación (FastAPI lo usa solo). |
| `500` | Internal Server Error | Tu código lanzó una excepción que nadie atrapó. |

Hasta ahora, `/tareas/99` respondía `200` con `{"error": "no existe"}`: el cliente creía que todo había salido bien. Vas a arreglarlo.

## Elegir el código de una ruta: `status_code`

En el decorador podés indicar qué código devuelve la ruta cuando todo sale bien:

```python
@app.post("/tareas", status_code=status.HTTP_201_CREATED)
```

`status` se importa de `fastapi` y tiene una constante con nombre para cada código: `status.HTTP_201_CREATED` es `201`. Podrías escribir `status_code=201`, pero el nombre se entiende mejor y el editor te lo autocompleta.

## Responder un error: `HTTPException`

Para responder un error, **lanzás** una `HTTPException` con `raise`, igual que cualquier excepción:

```python
raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Tarea no encontrada")
```

FastAPI la atrapa y la convierte en una respuesta con ese código, y con el `detail` como cuerpo:

```json
{"detail":"Tarea no encontrada"}
```

Como es un `raise`, la función se corta en ese punto. Y funciona aunque la excepción se lance **dentro de otra función** que tu ruta llamó: sube hasta FastAPI, como viste en la página de manejo de errores. Eso permite escribir una función auxiliar que busque la tarea o lance el `404`, y reusarla en todas las rutas.

## La API con los códigos correctos

Este es tu `main.py` completo, con una ruta para borrar una tarea y la búsqueda en una función aparte:

```python title="main.py"
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, Field

app = FastAPI()


class TareaCrear(BaseModel):
    titulo: str = Field(min_length=1, max_length=100)
    descripcion: str | None = None
    hecha: bool = False


tareas = [
    {"id": 1, "titulo": "Comprar pan", "descripcion": None, "hecha": False},
    {"id": 2, "titulo": "Estudiar FastAPI", "descripcion": None, "hecha": True},
    {"id": 3, "titulo": "Llamar a Juan", "descripcion": None, "hecha": False},
]


def buscar_tarea(tarea_id: int) -> dict:
    for tarea in tareas:
        if tarea["id"] == tarea_id:
            return tarea
    raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Tarea no encontrada")


@app.get("/tareas")
def listar_tareas():
    return tareas


@app.get("/tareas/{tarea_id}")
def obtener_tarea(tarea_id: int):
    return buscar_tarea(tarea_id)


@app.post("/tareas", status_code=status.HTTP_201_CREATED)
def crear_tarea(datos: TareaCrear):
    ultimo_id = max([tarea["id"] for tarea in tareas], default=0)
    nueva = {
        "id": ultimo_id + 1,
        "titulo": datos.titulo,
        "descripcion": datos.descripcion,
        "hecha": datos.hecha,
    }
    tareas.append(nueva)
    return nueva


@app.put("/tareas/{tarea_id}")
def modificar_tarea(tarea_id: int, datos: TareaCrear):
    tarea = buscar_tarea(tarea_id)
    tarea["titulo"] = datos.titulo
    tarea["descripcion"] = datos.descripcion
    tarea["hecha"] = datos.hecha
    return tarea


@app.delete("/tareas/{tarea_id}", status_code=status.HTTP_204_NO_CONTENT)
def borrar_tarea(tarea_id: int):
    tarea = buscar_tarea(tarea_id)
    tareas.remove(tarea)
```

Fijate en:

- **`buscar_tarea`** devuelve la tarea o lanza el `404`. Las tres rutas que reciben un `tarea_id` la usan, y ninguna necesita un `if`.
- **`borrar_tarea`** no tiene `return`: con `204` la respuesta va sin cuerpo. `lista.remove(x)` saca ese elemento de la lista.
- **El id nuevo** ahora sale del id más alto + 1, no de `len(tareas) + 1`. Con la posibilidad de borrar, la cantidad de tareas ya no sirve: si borrás la 2 de tres tareas, `len + 1` daría `3`, un id que ya existe. `max(..., default=0)` devuelve `0` si la lista está vacía.

## Probarlo

```bash
curl -i http://127.0.0.1:8000/tareas/99
curl -i -X DELETE http://127.0.0.1:8000/tareas/3
curl -i -X DELETE http://127.0.0.1:8000/tareas/3
```

```http
HTTP/1.1 404 Not Found
content-type: application/json

{"detail":"Tarea no encontrada"}
```

```http
HTTP/1.1 204 No Content
```

El segundo `DELETE` a la misma tarea da `404`: ya no existe. Y un `POST` exitoso ahora responde `HTTP/1.1 201 Created`.

!!! warning "Un `500` siempre es un bug tuyo"

    Si ves un `500`, significa que tu código lanzó una excepción que nadie convirtió en una respuesta (como el `KeyError` de dos páginas atrás). Buscá el traceback en la terminal del servidor y corregilo. Los errores esperables, como "no existe" o "datos inválidos", tienen que responder un `4xx`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | Express | FastAPI |
    | --- | --- |
    | `res.status(201).json(nueva)` | `status_code=201` en el decorador + `return nueva` |
    | `return res.status(404).json({ detail: "..." })` | `raise HTTPException(status_code=404, detail="...")` |
    | `res.sendStatus(204)` | `status_code=204` y la función sin `return` |

    Como es una excepción, en FastAPI el `404` se puede lanzar desde una función auxiliar; en Express, esa función tendría que devolver algo y la ruta revisarlo con un `if`.

??? example "Ver comparación en Django ⬇️"

    `raise HTTPException(status_code=404, ...)` es el equivalente de `raise Http404(...)` o de `get_object_or_404(Tarea, pk=tarea_id)`, y `buscar_tarea` cumple el mismo rol que este último. En DRF, el código se elige con `Response(data, status=status.HTTP_201_CREATED)`: hasta las constantes se llaman igual.

??? example "Ver comparación en Spring Boot ⬇️"

    | Spring Boot | FastAPI |
    | --- | --- |
    | `@ResponseStatus(HttpStatus.CREATED)` | `status_code=status.HTTP_201_CREATED` |
    | `throw new ResponseStatusException(HttpStatus.NOT_FOUND, "...")` | `raise HTTPException(status_code=404, detail="...")` |
    | `ResponseEntity.noContent().build()` | `status_code=204` y sin `return` |

## Mini ejercicio

Hacé que `crear_tarea` responda `409 Conflict` con el detalle `"Ya existe una tarea con ese título"` si ya hay una tarea con el mismo título, sin importar mayúsculas.

??? success "Ver solución"

    Agregá esto al principio de `crear_tarea`:

    ```python
    for tarea in tareas:
        if tarea["titulo"].lower() == datos.titulo.lower():
            raise HTTPException(status_code=status.HTTP_409_CONFLICT, detail="Ya existe una tarea con ese título")
    ```

    Con `{"titulo": "comprar PAN"}` responde `409` y `{"detail":"Ya existe una tarea con ese título"}`. Con un título nuevo, sigue respondiendo `201`.

---

**Siguiente:** [Documentación automática (`/docs`) →](09-documentacion.md)
