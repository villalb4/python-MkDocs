# Organizar rutas con `APIRouter`

!!! info "En esta página"

    **Qué vas a aprender:** cómo repartir las rutas de tu API en varios archivos con `APIRouter`, y cómo organizar el proyecto como un paquete de Python.

    **Qué necesitás:** [Documentación automática (`/docs`)](09-documentacion.md) y [Paquetes y `__init__.py`](../python/12-paquetes.md).

## Un solo archivo no escala

Tu `main.py` ya tiene un modelo, una lista de datos, una función auxiliar y cinco rutas. Cuando agregues usuarios, autenticación y más recursos, va a ser un archivo de cientos de renglones donde todo está mezclado.

La solución es la de siempre en Python: repartir el código en módulos y paquetes. FastAPI aporta la pieza que falta, **`APIRouter`**: un "mini `app`" donde registrás rutas en otro archivo, y que después conectás a la app principal.

## La nueva estructura

Vas a pasar de un `main.py` suelto a un paquete `app`:

```text
tareas-api/
├── pyproject.toml
└── app/
    ├── __init__.py
    ├── main.py          # crea la app y conecta los routers
    ├── modelos.py       # los modelos de Pydantic
    └── routers/
        ├── __init__.py
        └── tareas.py    # todas las rutas de /tareas
```

Creá las carpetas `app` y `app/routers`, cada una con un `__init__.py` vacío, y **borrá el `main.py` de la raíz**: su contenido se reparte en los archivos nuevos.

## Los modelos, aparte

```python title="app/modelos.py"
from pydantic import BaseModel, Field


class TareaCrear(BaseModel):
    titulo: str = Field(min_length=1, max_length=100, examples=["Comprar pan"])
    descripcion: str | None = Field(default=None, description="Detalle opcional de la tarea")
    hecha: bool = False
```

## El router de tareas

```python title="app/routers/tareas.py"
from fastapi import APIRouter, HTTPException, status

from app.modelos import TareaCrear

router = APIRouter(prefix="/tareas", tags=["tareas"])

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


@router.get("")
def listar_tareas():
    """Devuelve todas las tareas."""
    return tareas


@router.get("/{tarea_id}", responses={404: {"description": "La tarea no existe"}})
def obtener_tarea(tarea_id: int):
    """Devuelve una tarea por su id."""
    return buscar_tarea(tarea_id)


@router.post("", status_code=status.HTTP_201_CREATED)
def crear_tarea(datos: TareaCrear):
    """Crea una tarea nueva."""
    ultimo_id = max([tarea["id"] for tarea in tareas], default=0)
    nueva = {
        "id": ultimo_id + 1,
        "titulo": datos.titulo,
        "descripcion": datos.descripcion,
        "hecha": datos.hecha,
    }
    tareas.append(nueva)
    return nueva


@router.put("/{tarea_id}", responses={404: {"description": "La tarea no existe"}})
def modificar_tarea(tarea_id: int, datos: TareaCrear):
    """Reemplaza los datos de una tarea."""
    tarea = buscar_tarea(tarea_id)
    tarea["titulo"] = datos.titulo
    tarea["descripcion"] = datos.descripcion
    tarea["hecha"] = datos.hecha
    return tarea


@router.delete("/{tarea_id}", status_code=status.HTTP_204_NO_CONTENT, responses={404: {"description": "La tarea no existe"}})
def borrar_tarea(tarea_id: int):
    """Borra una tarea."""
    tarea = buscar_tarea(tarea_id)
    tareas.remove(tarea)
```

Es el mismo código de antes, con tres cambios:

- **`router = APIRouter(...)`** reemplaza a `app = FastAPI()`, y los decoradores pasan a ser `@router.get`, `@router.post`, etc.
- **`prefix="/tareas"`** se agrega adelante de todas las rutas del router. Por eso las rutas se escriben sin repetirlo: `"/{tarea_id}"` termina siendo `/tareas/{tarea_id}`, y `""` (texto vacío) es `/tareas` a secas.
- **`tags=["tareas"]`** se aplica a todas las rutas: ya no hace falta ponerlo en cada decorador.

El modelo se importa con `from app.modelos import TareaCrear`, un import absoluto desde el paquete `app`.

!!! note "`""` o `"/"`"

    Si escribís `@router.get("/")`, la ruta queda como `/tareas/`, con barra al final. Un pedido a `/tareas` recibe entonces una redirección (`307`) hacia `/tareas/`. Con `""` la ruta queda exactamente como antes, `/tareas`, y los comandos de las páginas anteriores siguen funcionando.

## La app principal

`main.py` queda cortito: crea la app y le **incluye** el router.

```python title="app/main.py"
from fastapi import FastAPI

from app.routers import tareas

app = FastAPI(
    title="API de tareas",
    description="Una API para organizar tareas pendientes.",
    version="0.1.0",
)

app.include_router(tareas.router)
```

`app.include_router(tareas.router)` registra en la app todas las rutas del router, con su prefijo y sus tags.

## Levantar la app

Como `main.py` ahora está dentro de `app/`, el comando cambia:

```bash
uv run fastapi dev app/main.py
```

`fastapi dev` detecta solo que `app` es un paquete y carga `app.main:app` (lo dice en la terminal: `Using import string: app.main:app`). Con Uvicorn directo, el comando es `uv run uvicorn app.main:app --reload`.

Probá las mismas URLs de siempre (`/tareas`, `/tareas/2`, `/tareas/99`, un `POST`): responden exactamente igual. Y en `/docs`, las rutas siguen agrupadas bajo "tareas".

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    `APIRouter` es el equivalente directo de `express.Router()`:

    ```javascript
    // routes/tareas.js
    const router = express.Router()
    router.get("/", (req, res) => res.json(tareas))
    module.exports = router

    // app.js
    app.use("/tareas", require("./routes/tareas"))
    ```

    La diferencia es dónde va el prefijo: en Express, en `app.use("/tareas", ...)`; en FastAPI, normalmente en el propio router (`APIRouter(prefix="/tareas")`), aunque `include_router` también acepta `prefix=`.

??? example "Ver comparación en Django ⬇️"

    Es la misma idea que separar las URLs de cada *app* de Django en su propio `urls.py` y conectarlas con `include()` en el `urls.py` del proyecto: `path("tareas/", include("tareas.urls"))` equivale a `app.include_router(tareas.router)` con `prefix="/tareas"`.

??? example "Ver comparación en Spring Boot ⬇️"

    Un router equivale a un `@RestController` con `@RequestMapping("/tareas")` en la clase:

    | Spring Boot | FastAPI |
    | --- | --- |
    | `@RestController` + `@RequestMapping("/tareas")` | `router = APIRouter(prefix="/tareas")` |
    | `@GetMapping("/{id}")` en un método | `@router.get("/{tarea_id}")` |
    | Spring encuentra los controllers solo | `app.include_router(...)` explícito |

    En FastAPI no hay escaneo automático: cada router se incluye a mano en `main.py`, y eso deja claro qué rutas tiene la app.

## Mini ejercicio

Agregá un router de usuarios en `app/routers/usuarios.py`, con prefijo `/usuarios` y tag `usuarios`, que tenga `GET /usuarios` (listar) y `POST /usuarios` (crear, respondiendo `201`). El modelo `UsuarioCrear` (con `nombre` de al menos 2 caracteres y `email`) va en `app/modelos.py`. Incluí el router en `main.py`.

??? success "Ver solución"

    En `app/modelos.py`, agregá `class UsuarioCrear(BaseModel):` con `nombre: str = Field(min_length=2)` y `email: str`. El router:

    ```python title="app/routers/usuarios.py"
    from fastapi import APIRouter, status

    from app.modelos import UsuarioCrear

    router = APIRouter(prefix="/usuarios", tags=["usuarios"])

    usuarios = []


    @router.get("")
    def listar_usuarios():
        """Devuelve todos los usuarios."""
        return usuarios


    @router.post("", status_code=status.HTTP_201_CREATED)
    def crear_usuario(datos: UsuarioCrear):
        """Crea un usuario nuevo."""
        nuevo = {"id": len(usuarios) + 1, "nombre": datos.nombre, "email": datos.email}
        usuarios.append(nuevo)
        return nuevo
    ```

    En `main.py`, el import pasa a ser `from app.routers import tareas, usuarios`, y abajo se agrega `app.include_router(usuarios.router)`. En `/docs` ahora vas a ver dos grupos: "tareas" y "usuarios".

---

**Siguiente:** [Inyección de dependencias con `Depends` →](11-depends.md)
