# Documentación automática (`/docs`)

!!! info "En esta página"

    **Qué vas a aprender:** cómo usar la página de documentación que FastAPI genera sola para tu API, cómo probar las rutas desde el navegador y cómo mejorar esa documentación con unos pocos agregados.

    **Qué necesitás:** [Códigos de estado y respuestas de error](08-codigos-de-estado.md).

## Tu API ya está documentada

Con el servidor prendido, abrí **<http://127.0.0.1:8000/docs>**. Vas a ver una página con todas tus rutas, cada una con su método en un color:

```text
GET     /tareas              Listar Tareas
POST    /tareas              Crear Tarea
GET     /tareas/{tarea_id}   Obtener Tarea
PUT     /tareas/{tarea_id}   Modificar Tarea
DELETE  /tareas/{tarea_id}   Borrar Tarea
```

No escribiste nada para esto. FastAPI arma la documentación a partir de lo que ya está en tu código:

- Las **rutas y métodos**, de los decoradores.
- Los **parámetros**, con sus tipos y si son obligatorios, de los type hints.
- La **forma del cuerpo**, del modelo `TareaCrear`, con sus reglas (largo mínimo y máximo del título).
- Los **códigos de respuesta**, del `status_code` de cada ruta. Además agrega el `422`, porque sabe que puede aparecer si los datos no son válidos.
- El **nombre de cada ruta** sale del nombre de la función: `crear_tarea` se muestra como "Crear Tarea". Otra razón para elegir buenos nombres.

Esta página se llama **Swagger UI** y es el formato de documentación de APIs más difundido.

## Probar rutas desde el navegador

Es la forma más cómoda de probar tu API, y reemplaza a `curl` en el día a día:

1. Hacé clic en una ruta, por ejemplo **POST /tareas**, para desplegarla.
2. Apretá **Try it out**. El ejemplo del cuerpo se vuelve editable.
3. Escribí el JSON que quieras mandar (por ejemplo, `{"titulo": "Pagar la luz"}`).
4. Apretá **Execute**.

Abajo aparece:

- **Curl**: el comando `curl` equivalente a lo que acabás de hacer. Muy útil para aprender o para compartir.
- **Server response**: el código de estado (`201`) y el cuerpo de la respuesta, con la tarea creada.

Si mandás datos inválidos, vas a ver el `422` con el detalle del error, igual que con `curl`.

!!! tip "Cuidado con los valores de ejemplo"

    Swagger UI completa el cuerpo con valores de ejemplo. Para los textos sin ejemplo pone literalmente `"string"`: si apretás **Execute** sin cambiarlo, vas a crear una tarea con descripción `"string"`.

## Otras dos direcciones

- **<http://127.0.0.1:8000/redoc>**: la misma documentación con otro diseño (**ReDoc**), más cómodo para leer y sin botones para probar.
- **<http://127.0.0.1:8000/openapi.json>**: los datos crudos de los que salen las dos páginas. Es un JSON en un formato estándar llamado **OpenAPI**, que otras herramientas pueden leer para, por ejemplo, generar código del cliente automáticamente.

## Mejorar la documentación

Con unos pocos agregados, la documentación queda mucho más clara.

**Datos generales de la API**, en `FastAPI(...)`:

```python
app = FastAPI(
    title="API de tareas",
    description="Una API para organizar tareas pendientes.",
    version="0.1.0",
)
```

Aparecen arriba de todo en `/docs`, en vez de "FastAPI".

**Grupos y descripciones de cada ruta.** `tags` agrupa rutas bajo un título, y el *docstring* de la función (el texto entre triples comillas en su primer renglón) aparece como descripción:

```python
@app.get("/tareas/{tarea_id}", tags=["tareas"])
def obtener_tarea(tarea_id: int):
    """Devuelve una tarea por su id. Responde 404 si no existe."""
    return buscar_tarea(tarea_id)
```

Cuando tengas también rutas de usuarios, `tags=["usuarios"]` las va a separar en otro grupo.

**Ejemplos y descripciones de los campos**, con `Field`:

```python
class TareaCrear(BaseModel):
    titulo: str = Field(min_length=1, max_length=100, examples=["Comprar pan"])
    descripcion: str | None = Field(default=None, description="Detalle opcional de la tarea")
    hecha: bool = False
```

Con `examples`, el botón **Try it out** completa `"titulo": "Comprar pan"` en vez de `"string"`.

**Errores que lanzás vos.** FastAPI no puede saber que `obtener_tarea` lanza un `404`, porque eso pasa adentro de `buscar_tarea`. Para que figure en la documentación, agregalo con `responses`:

```python
@app.get("/tareas/{tarea_id}", tags=["tareas"], responses={404: {"description": "La tarea no existe"}})
```

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Express la documentación no viene incluida. Lo habitual es sumar `swagger-ui-express` y escribir la especificación OpenAPI a mano (o con comentarios JSDoc mediante `swagger-jsdoc`), y mantenerla sincronizada con el código. En FastAPI se genera del propio código, así que no se desactualiza.

??? example "Ver comparación en Django ⬇️"

    Django REST Framework trae una página navegable propia, y para OpenAPI y Swagger UI se suele agregar **drf-spectacular**. FastAPI lo trae incluido y sin configuración: `/docs` y `/redoc` existen desde la primera ruta.

??? example "Ver comparación en Spring Boot ⬇️"

    El equivalente es **springdoc-openapi**, que también genera Swagger UI a partir del código (en `/swagger-ui.html`).

    | Spring Boot (springdoc) | FastAPI |
    | --- | --- |
    | `@Tag(name = "tareas")` | `tags=["tareas"]` |
    | `@Operation(summary = "...")` | El docstring o `summary="..."` |
    | `@Schema(example = "Comprar pan")` | `Field(examples=["Comprar pan"])` |
    | `@ApiResponse(responseCode = "404")` | `responses={404: {...}}` |

## Mini ejercicio

Desde `/docs`, sin usar la terminal:

1. Creá una tarea con el título `"Probar Swagger"`.
2. Buscala con **GET /tareas/{tarea_id}** usando el id que te devolvió.
3. Borrala con **DELETE** y comprobá que un nuevo **GET** responde `404`.

??? success "Ver solución"

    1. En **POST /tareas** → **Try it out**, cuerpo `{"titulo": "Probar Swagger"}` → **Execute**. Responde `201` con la tarea y su `id` (por ejemplo, `4`).
    2. En **GET /tareas/{tarea_id}** → **Try it out**, escribí `4` en el campo `tarea_id` → **Execute**. Responde `200` con la tarea.
    3. En **DELETE /tareas/{tarea_id}** con el mismo id responde `204`. Si repetís el **GET**, responde `404` con `{"detail": "Tarea no encontrada"}`.

---

**Siguiente:** [Organizar rutas con `APIRouter` →](10-apirouter.md)
