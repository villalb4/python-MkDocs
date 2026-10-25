# Parámetros de ruta y de query

!!! info "En esta página"

    **Qué vas a aprender:** cómo recibir datos desde la URL, ya sea como parte de la ruta (`/tareas/2`) o como parámetros de query (`/tareas?hecha=true`), y cómo FastAPI los convierte y valida usando los type hints.

    **Qué necesitás:** [Rutas con métodos HTTP](04-metodos-http.md) y [Type hints](../python/18-type-hints.md).

## Dos formas de mandar datos en la URL

```text
/tareas/2                    ← parámetro de ruta: identifica UN recurso
/tareas?hecha=true&limite=5  ← parámetros de query: filtran u ordenan
```

- **Parámetro de ruta:** forma parte del camino. Se usa para decir **cuál** recurso querés.
- **Parámetros de query:** van después de un `?`, como pares `nombre=valor` separados por `&`. Se usan para **opciones**: filtros, cantidad de resultados, orden.

En FastAPI los dos se reciben como **parámetros de la función**. La diferencia la marca la URL del decorador.

## Parámetros de ruta

Se declaran con llaves en la URL, y un parámetro de la función con el **mismo nombre** los recibe:

```python
@app.get("/tareas/{tarea_id}")
def obtener_tarea(tarea_id: int):
    for tarea in tareas:
        if tarea["id"] == tarea_id:
            return tarea
    return {"error": "no existe"}
```

Con `GET /tareas/2`, FastAPI llama a `obtener_tarea(tarea_id=2)`. Y el type hint `int` hace dos cosas:

1. **Convierte:** en la URL todo es texto (`"2"`), pero la función recibe el número `2`. Por eso `tarea["id"] == tarea_id` funciona.
2. **Valida:** si pedís `/tareas/abc`, FastAPI ni llama a tu función. Responde `422` explicando qué parámetro falló y por qué:

    ```json
    {"detail":[{"type":"int_parsing","loc":["path","tarea_id"],"msg":"Input should be a valid integer, unable to parse string as an integer","input":"abc"}]}
    ```

    `loc` dice **dónde** está el error: en la ruta (`path`), en el parámetro `tarea_id`.

!!! note "¿Y si la tarea no existe?"

    `/tareas/99` responde `{"error": "no existe"}` con código `200`, como si todo hubiera salido bien. No es lo correcto: debería ser un `404`. Lo vas a arreglar en la página de códigos de estado.

### El orden de las rutas importa

FastAPI revisa las rutas **en el orden en que las definiste** y usa la primera que coincida. Si tenés una ruta fija como `/tareas/pendientes`, tiene que ir **antes** que `/tareas/{tarea_id}`. Si no, `pendientes` se toma como un `tarea_id`, y como no es un número:

```json
{"detail":[{"type":"int_parsing","loc":["path","tarea_id"],"msg":"Input should be a valid integer, unable to parse string as an integer","input":"pendientes"}]}
```

## Parámetros de query

Todo parámetro de la función que **no** aparece en la URL del decorador es un parámetro de query. Actualizá `listar_tareas` en tu `main.py`, y agregá una tercera tarea para que los filtros se noten:

```python title="main.py"
tareas = [
    {"id": 1, "titulo": "Comprar pan", "hecha": False},
    {"id": 2, "titulo": "Estudiar FastAPI", "hecha": True},
    {"id": 3, "titulo": "Llamar a Juan", "hecha": False},
]


@app.get("/tareas")
def listar_tareas(hecha: bool | None = None, limite: int = 10):
    resultado = tareas
    if hecha is not None:
        resultado = [tarea for tarea in resultado if tarea["hecha"] == hecha]
    return resultado[:limite]
```

Ahora la misma ruta responde distinto según la query:

| URL | Resultado |
| --- | --- |
| `/tareas` | Las tres tareas (`hecha` vale `None`: no se filtra). |
| `/tareas?hecha=true` | Solo "Estudiar FastAPI". |
| `/tareas?hecha=false&limite=1` | Solo "Comprar pan": la primera pendiente. |
| `/tareas?limite=dos` | Error `422`: `limite` tiene que ser un número. |

Algunos detalles:

- **Los valores por defecto los vuelven opcionales.** `limite: int = 10` significa "si no lo mandan, vale 10". Y `hecha: bool | None = None` es la forma de decir "filtro opcional".
- **Los `bool` aceptan varias formas:** `true`, `false`, `1`, `0`, `yes`, `no`, `on`, `off`. Algo como `hecha=si` da `422`.
- **Los parámetros de query desconocidos se ignoran:** `/tareas?otro=x` responde como `/tareas`.

### Parámetros de query obligatorios

Si un parámetro de query **no tiene valor por defecto**, es obligatorio:

```python
@app.get("/saludo")
def saludar(nombre: str):
    return {"mensaje": f"Hola, {nombre}"}
```

`/saludo?nombre=Ana` responde `{"mensaje":"Hola, Ana"}`. Sin `nombre`, FastAPI responde `422` con `"type":"missing"` y `"msg":"Field required"`.

!!! tip "Comillas en `curl`"

    Para probar URLs con `?` o `&` en la terminal, envolvelas en comillas: `curl "http://127.0.0.1:8000/tareas?hecha=false&limite=1"`. Sin comillas, la terminal interpreta esos símbolos a su manera.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    ```javascript
    app.get("/tareas/:tareaId", (req, res) => {
      const tareaId = Number(req.params.tareaId)   // hay que convertir a mano
      ...
    })

    app.get("/tareas", (req, res) => {
      const limite = Number(req.query.limite ?? 10)
      ...
    })
    ```

    En Express, `req.params` y `req.query` traen siempre strings, y convertir y validar es tarea tuya. En FastAPI los recibís como argumentos ya convertidos, y si no se pueden convertir, el `422` sale solo.

??? example "Ver comparación en Django ⬇️"

    Los parámetros de ruta se parecen a los *path converters* de `urls.py` (`path("tareas/<int:tarea_id>/", ...)`), que también convierten a `int`. Los de query, en cambio, se leen a mano con `request.GET.get("limite", "10")` y llegan como texto. En FastAPI ambos se declaran igual, como parámetros de la función con su tipo.

??? example "Ver comparación en Spring Boot ⬇️"

    | Spring Boot | FastAPI |
    | --- | --- |
    | `@GetMapping("/tareas/{id}")` + `@PathVariable int id` | `@app.get("/tareas/{tarea_id}")` + `tarea_id: int` |
    | `@RequestParam(defaultValue = "10") int limite` | `limite: int = 10` |
    | `@RequestParam(required = false) Boolean hecha` | `hecha: bool | None = None` |
    | `@RequestParam String nombre` (obligatorio) | `nombre: str` |

    En FastAPI no hacen falta anotaciones: si el nombre está en la URL, es de ruta; si no, es de query.

## Mini ejercicio

Agregá una ruta `GET /buscar` con un parámetro de query obligatorio `texto` y uno opcional `hecha`, que devuelva las tareas cuyo título contenga ese texto (sin importar mayúsculas), filtradas por `hecha` si lo mandan.

??? success "Ver solución"

    ```python
    @app.get("/buscar")
    def buscar_tareas(texto: str, hecha: bool | None = None):
        resultado = [tarea for tarea in tareas if texto.lower() in tarea["titulo"].lower()]
        if hecha is not None:
            resultado = [tarea for tarea in resultado if tarea["hecha"] == hecha]
        return resultado
    ```

    | URL | Resultado |
    | --- | --- |
    | `/buscar?texto=pan` | "Comprar pan" |
    | `/buscar?texto=A&hecha=false` | "Comprar pan" y "Llamar a Juan" |
    | `/buscar` | `422`: falta `texto` |

---

**Siguiente:** [Cuerpo de la petición (body) con JSON →](06-body-json.md)
