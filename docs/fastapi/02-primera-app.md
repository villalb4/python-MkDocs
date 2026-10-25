# Primera app: `FastAPI()` y la primera ruta

!!! info "En esta página"

    **Qué vas a aprender:** cómo crear un proyecto con FastAPI, escribir tu primera ruta, levantar el servidor y verla funcionando en el navegador.

    **Qué necesitás:** [Qué es FastAPI y qué es una API](01-que-es-fastapi.md) y [Entornos virtuales](../python/21-entornos-virtuales.md).

## Crear el proyecto

Abrí una terminal en la carpeta donde guardás tus proyectos y ejecutá:

```bash
uv init --no-package tareas-api
cd tareas-api
uv add "fastapi[standard]"
```

`fastapi[standard]` instala FastAPI junto con sus extras recomendados: entre ellos, el servidor **Uvicorn** y el comando `fastapi`, que vas a usar para levantar la app. Las comillas evitan que algunas terminales se confundan con los corchetes.

Abrí la carpeta en VS Code con `code .`. Esta es la API de tareas que vas a ir construyendo durante todo el capítulo.

## La app mínima

Reemplazá el contenido de `main.py` por esto:

```python title="main.py"
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def inicio():
    return {"mensaje": "Hola, FastAPI"}
```

Son tres piezas:

1. **`app = FastAPI()`** crea la aplicación. `FastAPI` es una clase, y `app` es el objeto que representa toda tu API: ahí se van a registrar las rutas.
2. **`@app.get("/")`** es el decorador que registra la función de abajo para las peticiones `GET` a la URL `/` (la raíz del sitio).
3. **`def inicio():`** es la función que se ejecuta cuando llega esa petición. Lo que devuelve con `return`, FastAPI lo convierte en JSON y lo manda como respuesta.

A la combinación de método + URL + función se la llama **ruta** (o *endpoint*).

## Levantar el servidor

Tu código define la app, pero hace falta un **servidor** que escuche las peticiones y se las pase. Ejecutá:

```bash
uv run fastapi dev main.py
```

Vas a ver algo así:

```text
 ⚡️ Starting FastAPI in development mode

 🌐 Server started at http://127.0.0.1:8000
    Documentation at http://127.0.0.1:8000/docs

INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Application startup complete.
```

El servidor queda **corriendo**: la terminal no te devuelve el control mientras esté prendido. Para apagarlo, apretá <kbd>Ctrl</kbd> + <kbd>C</kbd>.

!!! warning "`uv run main.py` no levanta la API"

    Si ejecutás `uv run main.py` como hacías con los scripts, no pasa nada: el archivo define `app` y termina. Una API necesita un servidor que la mantenga escuchando, y eso es lo que hace `fastapi dev`. En la próxima página vas a ver qué pasa por debajo.

## Probarla en el navegador

Con el servidor corriendo, abrí **<http://127.0.0.1:8000>** en el navegador. Vas a ver:

```json
{"mensaje":"Hola, FastAPI"}
```

Acabás de hacer una petición `GET /` y recibir la respuesta de tu API. El navegador hace peticiones `GET` cada vez que escribís una URL.

Fijate en la terminal del servidor: cada petición deja un renglón.

```text
INFO:     127.0.0.1:62089 - "GET / HTTP/1.1" 200 OK
```

Probá ahora **<http://127.0.0.1:8000/nada>**. Como no hay ninguna ruta para `/nada`, FastAPI responde `{"detail":"Not Found"}` con código `404`.

## Agregar más rutas

Una app puede tener todas las rutas que quieras: cada una es una función con su decorador.

```python title="main.py"
from fastapi import FastAPI

app = FastAPI()

tareas = [
    {"id": 1, "titulo": "Comprar pan", "hecha": False},
    {"id": 2, "titulo": "Estudiar FastAPI", "hecha": True},
]


@app.get("/")
def inicio():
    return {"mensaje": "Hola, FastAPI"}


@app.get("/tareas")
def listar_tareas():
    return tareas


@app.get("/version")
def version():
    return "1.0"
```

Guardá el archivo. No hace falta reiniciar el servidor: `fastapi dev` detecta el cambio y se recarga solo. Abrí **<http://127.0.0.1:8000/tareas>** y vas a ver la lista en JSON.

Podés devolver diccionarios, listas, strings, números, `True`/`False` o `None`: FastAPI convierte todo a JSON. Por eso `/version` responde `"1.0"`, con comillas, que es como se escribe un texto en JSON.

!!! note "Los datos se pierden al reiniciar"

    La lista `tareas` vive en la memoria del programa. Si la modificás y reiniciás el servidor, vuelve a su estado inicial. Para que los datos persistan hace falta una base de datos, que llega en el Capítulo 3. Mientras tanto, la lista alcanza para aprender.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    ```javascript
    const express = require("express")
    const app = express()

    app.get("/", (req, res) => {
      res.json({ mensaje: "Hola, FastAPI" })
    })

    app.listen(8000)
    ```

    - `app = FastAPI()` equivale a `const app = express()`.
    - En vez de `res.json(...)`, en FastAPI hacés `return`.
    - En Express, `app.listen()` levanta el servidor desde el código; en FastAPI lo levanta un comando aparte (`fastapi dev`), y tu archivo solo define la app.

??? example "Ver comparación en Django ⬇️"

    Para lo mismo en Django necesitás crear un proyecto con `django-admin startproject`, una view en `views.py`, registrarla en `urls.py` y levantar con `python manage.py runserver`. En FastAPI cabe en un archivo de 8 renglones: la ruta y la función van juntas gracias al decorador. `fastapi dev` cumple el rol de `runserver`.

??? example "Ver comparación en Spring Boot ⬇️"

    ```java
    @RestController
    public class InicioController {

        @GetMapping("/")
        public Map<String, String> inicio() {
            return Map.of("mensaje", "Hola, FastAPI");
        }
    }
    ```

    `@app.get("/")` equivale a `@GetMapping("/")`, y la función equivale al método del controller. No hace falta una clase ni la `@SpringBootApplication` con su `main`: el objeto `app` es la aplicación, y `fastapi dev` cumple el rol de `./mvnw spring-boot:run`.

## Mini ejercicio

Agregá una ruta `GET /usuarios` que devuelva una lista con dos usuarios (cada uno, un diccionario con `nombre` y `email`). Probala en el navegador sin reiniciar el servidor.

??? success "Ver solución"

    ```python
    @app.get("/usuarios")
    def listar_usuarios():
        return [
            {"nombre": "Ana", "email": "ana@ejemplo.com"},
            {"nombre": "Juan", "email": "juan@ejemplo.com"},
        ]
    ```

    Al guardar, el servidor se recarga solo. En <http://127.0.0.1:8000/usuarios> vas a ver:

    ```json
    [{"nombre":"Ana","email":"ana@ejemplo.com"},{"nombre":"Juan","email":"juan@ejemplo.com"}]
    ```

---

**Siguiente:** [Servidor Uvicorn y recarga automática →](03-uvicorn.md)
