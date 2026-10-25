# Validación con Pydantic: modelos

!!! info "En esta página"

    **Qué vas a aprender:** cómo describir la forma de los datos con un modelo de Pydantic, para que FastAPI valide el cuerpo de cada petición y responda errores claros sin que escribas un solo `if`.

    **Qué necesitás:** [Cuerpo de la petición (body) con JSON](06-body-json.md), [Clases básicas](../python/17-clases-basicas.md) y [Type hints](../python/18-type-hints.md).

## Un modelo describe la forma de los datos

En la página anterior, `datos: dict` aceptaba cualquier cosa, y un cuerpo sin `titulo` terminaba en un `500`. La solución es decirle a FastAPI **qué forma** tiene que tener el cuerpo. Eso es un **modelo** de Pydantic: una clase que hereda de `BaseModel`, con un atributo por campo y su type hint.

```python
from pydantic import BaseModel


class TareaCrear(BaseModel):
    titulo: str
    descripcion: str | None = None
    hecha: bool = False
```

Se lee como una ficha: "una tarea para crear tiene un `titulo` de texto (obligatorio), una `descripcion` de texto que puede faltar, y un `hecha` que si no viene vale `False`". Es lo mismo que ya sabés de type hints y valores por defecto, pero en una clase.

Pydantic viene instalado con FastAPI, así que no hace falta agregar nada.

## Usar el modelo en una ruta

Reemplazá `dict` por el modelo. Así queda tu `main.py` completo:

```python title="main.py"
from fastapi import FastAPI
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


@app.get("/tareas")
def listar_tareas():
    return tareas


@app.post("/tareas")
def crear_tarea(datos: TareaCrear):
    nueva = {
        "id": len(tareas) + 1,
        "titulo": datos.titulo,
        "descripcion": datos.descripcion,
        "hecha": datos.hecha,
    }
    tareas.append(nueva)
    return nueva


@app.put("/tareas/{tarea_id}")
def modificar_tarea(tarea_id: int, datos: TareaCrear):
    for tarea in tareas:
        if tarea["id"] == tarea_id:
            tarea["titulo"] = datos.titulo
            tarea["descripcion"] = datos.descripcion
            tarea["hecha"] = datos.hecha
            return tarea
    return {"error": "no existe"}
```

Ahora `datos` ya no es un diccionario sino un **objeto** `TareaCrear`: se accede con punto (`datos.titulo`), y el editor te autocompleta los campos. Cuando FastAPI llama a tu función, los datos **ya pasaron la validación**: si el cuerpo no tenía la forma correcta, tu función ni se ejecuta.

La lista `tareas` sigue guardando diccionarios: el modelo describe lo que **llega** en la petición. En el Capítulo 3 la lista se va a reemplazar por una base de datos.

## Qué valida

Mandá cuerpos con errores y mirá las respuestas. Todas son `422`, con un mensaje que dice qué campo falló y por qué:

| Cuerpo | `type` | `msg` |
| --- | --- | --- |
| `{"nombre": "x"}` | `missing` | `Field required` |
| `{"titulo": 42}` | `string_type` | `Input should be a valid string` |
| `{"titulo": "x", "hecha": "tal vez"}` | `bool_parsing` | `Input should be a valid boolean, unable to interpret input` |
| `{"titulo": ""}` | `string_too_short` | `String should have at least 1 character` |

La respuesta completa del primer caso:

```json
{"detail":[{"type":"missing","loc":["body","titulo"],"msg":"Field required","input":{"nombre":"x"}}]}
```

`loc` indica que el error está en el cuerpo (`body`), en el campo `titulo`. Si hay varios errores a la vez, `detail` los trae **todos** en la lista, no solo el primero.

Lo que el cliente recibía antes como un `500` sin explicación ahora es un `422` que le dice exactamente qué corregir.

Dos comportamientos para tener en cuenta:

- **Conversiones razonables sí se hacen.** `"hecha": "true"` (un texto) se convierte al `bool` `True`, igual que en los parámetros de query.
- **Los campos que no están en el modelo se ignoran.** Si mandás `"prioridad": "alta"`, no da error, pero tampoco llega a tu función.

## Reglas extra con `Field`

El type hint dice el **tipo**. Para reglas sobre el **valor**, se usa `Field`, como en `titulo` arriba:

```python
titulo: str = Field(min_length=1, max_length=100)
```

Ahora un título vacío o de más de 100 caracteres da `422`. Las reglas más comunes:

| Regla | Para | Significa |
| --- | --- | --- |
| `min_length`, `max_length` | Textos y listas | Largo mínimo y máximo. |
| `ge`, `le` | Números | Mayor o igual / menor o igual (*greater/less or equal*). |
| `gt`, `lt` | Números | Mayor / menor estricto. |
| `default` | Cualquiera | Valor por defecto: `Field(default=10, ge=1)`. |

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Un modelo de Pydantic cumple el rol de un esquema de **Zod** (o Joi), pero integrado con el framework:

    ```javascript
    const TareaCrear = z.object({
      titulo: z.string().min(1).max(100),
      descripcion: z.string().nullable().optional(),
      hecha: z.boolean().default(false),
    })

    app.post("/tareas", (req, res) => {
      const resultado = TareaCrear.safeParse(req.body)
      if (!resultado.success) return res.status(422).json(resultado.error)
      ...
    })
    ```

    En FastAPI no hace falta llamar a la validación ni armar la respuesta de error: alcanza con poner el modelo como tipo del parámetro.

??? example "Ver comparación en Django ⬇️"

    Un modelo de Pydantic se parece a un `Serializer` de Django REST Framework (o a un `Form` de Django): declara campos, tipos y reglas, y valida los datos que llegan. La diferencia es que acá se usan type hints comunes en vez de clases de campo (`serializers.CharField(max_length=100)`). Ojo: **no** es un modelo de base de datos como `models.Model`; eso, en esta guía, lo hace SQLAlchemy.

??? example "Ver comparación en Spring Boot ⬇️"

    Equivale a un DTO con Bean Validation:

    ```java
    public class TareaCrear {
        @NotNull @Size(min = 1, max = 100)
        private String titulo;
        private String descripcion;
        private boolean hecha = false;
        // getters y setters...
    }

    @PostMapping("/tareas")
    public Tarea crear(@RequestBody @Valid TareaCrear datos) { ... }
    ```

    En Pydantic no hacen falta getters, setters ni `@Valid`: la clase son tres renglones, y FastAPI valida siempre.

## Mini ejercicio

Agregá una ruta `POST /usuarios` que reciba un modelo `UsuarioCrear` con `nombre` (texto de al menos 2 caracteres), `email` (texto) y `edad` (entero entre 0 y 120), y guarde el usuario en una lista `usuarios` con un `id`. Probala con datos válidos y con `{"nombre": "A", "email": "a@b.com", "edad": -1}`.

??? success "Ver solución"

    ```python
    usuarios = []


    class UsuarioCrear(BaseModel):
        nombre: str = Field(min_length=2)
        email: str
        edad: int = Field(ge=0, le=120)


    @app.post("/usuarios")
    def crear_usuario(datos: UsuarioCrear):
        nuevo = {"id": len(usuarios) + 1, "nombre": datos.nombre, "email": datos.email, "edad": datos.edad}
        usuarios.append(nuevo)
        return nuevo
    ```

    Con datos inválidos, la respuesta trae los **dos** errores juntos: `string_too_short` en `nombre` y `greater_than_equal` en `edad`. Para validar que el email tenga un formato correcto, Pydantic trae el tipo `EmailStr`; lo vas a usar en el capítulo de autenticación.

---

**Siguiente:** [Códigos de estado y respuestas de error →](08-codigos-de-estado.md)
