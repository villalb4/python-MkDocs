# Qué es FastAPI y qué es una API

!!! info "En esta página"

    **Qué vas a aprender:** qué es una API, cómo se comunican un cliente y un servidor por HTTP, y qué es FastAPI. Es una página de conceptos: el código llega en la próxima.

    **Qué necesitás:** haber terminado el [Capítulo 1](../python/21-entornos-virtuales.md).

## Qué es una API

Una **API** (*Application Programming Interface*) es la forma en que un programa le ofrece servicios a otros programas. En este contexto, una **API web**: un programa que corre en un servidor, recibe pedidos por internet y responde con datos.

Pensá en una app de tareas. La pantalla que ves en el celular (el **frontend**) no guarda las tareas: se las pide a un servidor (el **backend**). Ese diálogo tiene dos partes:

- El **cliente** (la app, una página web, otro servidor) hace una **petición**: "dame la lista de tareas".
- El **servidor** (tu API) la procesa y devuelve una **respuesta**: la lista, en formato JSON.

Lo que vas a construir en esta guía es ese servidor.

## Peticiones y respuestas HTTP

Cliente y servidor hablan con un protocolo llamado **HTTP**. Cada petición tiene, como mínimo:

- Un **método**, que dice qué querés hacer: `GET` (leer), `POST` (crear), `PUT` (modificar), `DELETE` (borrar).
- Una **URL**, que dice sobre qué: `/tareas`, `/tareas/2`.
- A veces, un **cuerpo** (*body*) con datos, por ejemplo la tarea nueva que querés crear.

Y cada respuesta trae:

- Un **código de estado**: un número que resume cómo salió. `200` es "todo bien", `404` es "no existe".
- Un **cuerpo** con los datos, casi siempre en **JSON**.

Así se ve una respuesta real de la API que vas a armar, para la petición `GET /tareas/2`:

```http
HTTP/1.1 200 OK
content-type: application/json

{"id":2,"titulo":"Estudiar FastAPI","hecha":true}
```

`200 OK` es el código de estado, `content-type` avisa que el cuerpo es JSON, y abajo está el cuerpo. Fijate que ese JSON es igual a un diccionario de Python: por eso los diccionarios eran tan importantes.

## Recursos y URLs

En una API web, las URLs suelen nombrar **recursos** (las cosas que maneja la API) y el método dice qué hacer con ellos. Es un estilo llamado **REST**:

| Método y URL | Qué hace |
| --- | --- |
| `GET /tareas` | Lista todas las tareas. |
| `GET /tareas/2` | Devuelve la tarea con id 2. |
| `POST /tareas` | Crea una tarea nueva (los datos van en el cuerpo). |
| `PUT /tareas/2` | Modifica la tarea 2. |
| `DELETE /tareas/2` | Borra la tarea 2. |

La misma URL (`/tareas/2`) hace cosas distintas según el método. Los recursos van en plural y los ids, en la URL.

## Qué es FastAPI

**FastAPI** es una librería de Python para construir APIs web. Vos escribís funciones comunes, y FastAPI se encarga de conectarlas con HTTP: recibir peticiones, llamar a la función que corresponde y convertir lo que devuelve en una respuesta JSON.

Así se ve la API que dio la respuesta de arriba. No hace falta que la entiendas entera todavía: la vas a escribir en la próxima página.

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


@app.get("/tareas/{tarea_id}")
def obtener_tarea(tarea_id: int):
    for tarea in tareas:
        if tarea["id"] == tarea_id:
            return tarea
    return {"error": "no existe"}
```

Ya podés reconocer casi todo: un `import`, una lista de diccionarios, funciones con type hints y decoradores que registran rutas.

## Por qué FastAPI

- **Usa type hints para validar.** Como `tarea_id` está anotado como `int`, si alguien pide `/tareas/abc`, FastAPI responde solo con un error `422` que explica el problema, sin que escribas una línea de validación:

    ```http
    HTTP/1.1 422 Unprocessable Content
    content-type: application/json

    {"detail":[{"type":"int_parsing","loc":["path","tarea_id"],"msg":"Input should be a valid integer, unable to parse string as an integer","input":"abc"}]}
    ```

- **Documentación automática.** Genera una página web interactiva con todas tus rutas, donde podés probarlas desde el navegador.
- **Asincrónico.** Acepta rutas `async def`, para atender muchos pedidos mientras espera a la base de datos.
- **Construido sobre piezas sólidas:** Starlette para la parte web y Pydantic para los datos.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    FastAPI ocupa el mismo lugar que Express: un framework web liviano donde definís rutas y devolvés JSON. La diferencia es que FastAPI trae incluidas cosas que en Express agregás aparte: validación de datos (en Express, con Zod o Joi), documentación (con Swagger) y conversión automática a JSON (en FastAPI hacés `return` en vez de `res.json(...)`).

??? example "Ver comparación en Django ⬇️"

    Django es un framework "con baterías incluidas": ORM, admin, templates, autenticación. Para APIs se suele sumar **Django REST Framework**. FastAPI es más chico y se enfoca solo en la API; el resto lo elegís vos (en esta guía, SQLAlchemy para la base, Alembic para migraciones, PyJWT para autenticación). Los *serializers* de DRF cumplen un rol parecido a los modelos de Pydantic.

??? example "Ver comparación en Spring Boot ⬇️"

    Una ruta de FastAPI equivale a un método de un `@RestController`. FastAPI es mucho más liviano: no hay contenedor de inversión de control, ni anotaciones de configuración, ni clases obligatorias. Lo que en Spring resuelven `@Valid` + Bean Validation y springdoc-openapi, en FastAPI viene integrado gracias a los type hints y Pydantic.

## Mini ejercicio

Siguiendo el estilo REST, ¿qué método y qué URL usarías para cada operación sobre una API de productos?

1. Ver todos los productos.
2. Ver el producto con id 15.
3. Dar de alta un producto nuevo.
4. Cambiar el precio del producto 15.
5. Eliminar el producto 15.

??? success "Ver solución"

    1. `GET /productos`
    2. `GET /productos/15`
    3. `POST /productos` (con los datos del producto en el cuerpo)
    4. `PUT /productos/15` (con los datos nuevos en el cuerpo)
    5. `DELETE /productos/15`

    El recurso va en plural, el id va en la URL, y el método dice la acción.

---

**Siguiente:** [Primera app: `FastAPI()` y la primera ruta →](02-primera-app.md)
