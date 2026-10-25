# Cuerpo de la petición (body) con JSON

!!! info "En esta página"

    **Qué vas a aprender:** qué es el cuerpo de una petición, cómo mandar JSON con `curl` y cómo recibirlo en FastAPI para crear y modificar tareas de verdad.

    **Qué necesitás:** [Parámetros de ruta y de query](05-parametros.md).

## Qué es el cuerpo

Los parámetros de ruta y de query sirven para datos cortos: un id, un filtro. Pero para **crear** una tarea tenés que mandar sus datos completos, y eso no entra cómodo en una URL.

Para eso está el **cuerpo** (*body*) de la petición: un bloque de datos que viaja junto con ella, casi siempre en JSON. Lo usan `POST` (crear) y `PUT` (modificar). Un `GET` normalmente no lleva cuerpo.

Una petición para crear una tarea se ve así:

```http
POST /tareas HTTP/1.1
Content-Type: application/json

{"titulo": "Pagar la luz"}
```

El encabezado `Content-Type: application/json` le avisa al servidor que el cuerpo es JSON.

## Recibir el cuerpo en FastAPI

Hasta ahora, `crear_tarea` agregaba siempre la misma tarea. Cambiala para que use lo que llega en el cuerpo, y agregá una ruta `PUT` que modifique una tarea:

```python title="main.py"
@app.post("/tareas")
def crear_tarea(datos: dict):
    nueva = {"id": len(tareas) + 1, "titulo": datos["titulo"], "hecha": False}
    tareas.append(nueva)
    return nueva


@app.put("/tareas/{tarea_id}")
def modificar_tarea(tarea_id: int, datos: dict):
    for tarea in tareas:
        if tarea["id"] == tarea_id:
            tarea["titulo"] = datos["titulo"]
            tarea["hecha"] = datos["hecha"]
            return tarea
    return {"error": "no existe"}
```

(Sacá la ruta `PUT /tareas/{tarea_id}` que habías hecho antes para completar tareas: esta la reemplaza.)

¿Cómo sabe FastAPI que `datos` viene del cuerpo y no de la query? Por el tipo: un `int`, `str` o `bool` suelto es un parámetro de query, pero un `dict` (un dato con estructura) solo puede venir del cuerpo. FastAPI lee el JSON y te lo da como diccionario.

En `modificar_tarea` se combinan las dos cosas: `tarea_id` llega por la ruta y `datos` por el cuerpo.

## Mandar JSON con `curl`

Con `curl`, el cuerpo se manda con `-d` y el encabezado con `-H`:

=== "macOS / Linux"

    ```bash
    curl -X POST http://127.0.0.1:8000/tareas \
      -H "Content-Type: application/json" \
      -d '{"titulo": "Pagar la luz"}'
    ```

    La `\` al final de un renglón permite seguir el comando en el siguiente. El JSON va entre comillas simples para que las dobles de adentro lleguen tal cual.

=== "Windows (PowerShell)"

    Escribir JSON dentro de un comando de PowerShell trae problemas con las comillas. Lo más simple es guardarlo en un archivo `tarea.json`:

    ```json title="tarea.json"
    {"titulo": "Pagar la luz"}
    ```

    y mandarlo con `@` adelante del nombre:

    ```powershell
    curl.exe -X POST http://127.0.0.1:8000/tareas -H "Content-Type: application/json" -d "@tarea.json"
    ```

    Esto funciona igual en macOS y Linux.

La respuesta es la tarea creada:

```json
{"id":4,"titulo":"Pagar la luz","hecha":false}
```

Y para modificar la tarea 1:

```bash
curl -X PUT http://127.0.0.1:8000/tareas/1 \
  -H "Content-Type: application/json" \
  -d '{"titulo": "Comprar pan integral", "hecha": true}'
```

```json
{"id":1,"titulo":"Comprar pan integral","hecha":true}
```

!!! warning "No te olvides del `Content-Type`"

    Sin `-H "Content-Type: application/json"`, `curl` manda los datos como si fueran un formulario web, y FastAPI responde `422` con `"msg":"Input should be a valid dictionary"`.

## Cuando el JSON está mal

FastAPI ya te protege de algunos errores. Si el JSON está mal escrito (por ejemplo, con una coma de más: `{"titulo": "x",}`), responde `422`:

```json
{"detail":[{"type":"json_invalid","loc":["body",15],"msg":"JSON decode error","input":{},"ctx":{"error":"Expecting property name enclosed in double quotes"}}]}
```

Y si no mandás cuerpo, también `422`, con `"msg":"Field required"`.

## El problema con `dict`

Pero `dict` acepta **cualquier** diccionario. Mirá qué pasa con datos que tienen la forma equivocada:

- **Falta `titulo`** (`{"nombre": "x"}`): `datos["titulo"]` da `KeyError`, y el cliente recibe un `500 Internal Server Error`, sin ninguna explicación. En la terminal del servidor aparece el traceback:

    ```text
    KeyError: 'titulo'
    INFO:     127.0.0.1:62325 - "POST /tareas HTTP/1.1" 500 Internal Server Error
    ```

- **`titulo` es un número** (`{"titulo": 42}`): se guarda sin problema, y ahora tenés una tarea con título `42`.

Un `500` significa "el servidor falló", cuando en realidad el error fue del cliente. Y datos con tipos equivocados terminan rompiendo otras partes del programa. Podrías revisar todo a mano con `if`, pero sería tedioso y fácil de olvidar.

La solución es describir **qué forma** tiene que tener el cuerpo y dejar que FastAPI lo valide. Para eso existen los modelos de Pydantic, el tema de la próxima página. `datos: dict` fue solo un paso intermedio para entender cómo llega el cuerpo.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    ```javascript
    app.use(express.json())   // sin esto, req.body es undefined

    app.post("/tareas", (req, res) => {
      const nueva = { id: tareas.length + 1, titulo: req.body.titulo, hecha: false }
      tareas.push(nueva)
      res.json(nueva)
    })
    ```

    En Express tenés que activar el lector de JSON con `express.json()`; en FastAPI viene incluido. `req.body.titulo` faltante da `undefined` sin avisar; en Python, `datos["titulo"]` da `KeyError`. En los dos casos, la solución de verdad es validar (en Express, con algo como Zod; en FastAPI, con Pydantic).

??? example "Ver comparación en Django ⬇️"

    En Django leés el cuerpo a mano con `json.loads(request.body)`, y además tenés que lidiar con la protección CSRF. En Django REST Framework, `request.data` ya trae el JSON leído, y un *serializer* lo valida: ese es el rol que van a cumplir los modelos de Pydantic.

??? example "Ver comparación en Spring Boot ⬇️"

    `datos: dict` equivale a `@RequestBody Map<String, Object> datos`: funciona, pero sin tipos ni validación. Lo habitual en Spring es `@RequestBody @Valid TareaDTO datos`, con una clase que describe los campos. En FastAPI, ese DTO es un modelo de Pydantic, y la validación es automática.

## Mini ejercicio

Agregá a `crear_tarea` la posibilidad de mandar `"hecha"` en el cuerpo. Si no viene, la tarea se crea como no hecha. Probalo con y sin `"hecha"`.

??? success "Ver solución"

    ```python
    @app.post("/tareas")
    def crear_tarea(datos: dict):
        nueva = {
            "id": len(tareas) + 1,
            "titulo": datos["titulo"],
            "hecha": datos.get("hecha", False),
        }
        tareas.append(nueva)
        return nueva
    ```

    `.get("hecha", False)` devuelve `False` si la clave no está, en vez de dar `KeyError`. Con `{"titulo": "Ir al banco", "hecha": true}` responde `{"id":4,"titulo":"Ir al banco","hecha":true}`; sin `"hecha"`, la crea con `false`. Fijate que con `dict` nadie te impide mandar `"hecha": "tal vez"`: otra razón para usar Pydantic.

---

**Siguiente:** [Validación con Pydantic: modelos →](07-pydantic-modelos.md)
