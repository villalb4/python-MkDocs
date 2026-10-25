# Inyección de dependencias con `Depends`

!!! info "En esta página"

    **Qué vas a aprender:** qué es una dependencia en FastAPI, cómo usar `Depends` para reutilizar lógica entre rutas (paginación, buscar un recurso o lanzar un `404`) y por qué esto va a ser clave para la base de datos y la autenticación.

    **Qué necesitás:** [Organizar rutas con `APIRouter`](10-apirouter.md).

## Qué es una dependencia

Muchas rutas necesitan lo mismo antes de empezar: leer los parámetros de paginación, buscar la tarea del id (o responder `404`), y más adelante abrir una conexión a la base o saber qué usuario hizo el pedido.

Una **dependencia** es una función común que FastAPI ejecuta **antes** de tu ruta. Lo que devuelve se le pasa a la ruta como un parámetro. Vos no la llamás: la **declarás**, y FastAPI se encarga. A eso se lo llama **inyección de dependencias**.

## Tu primera dependencia: paginación

Si varias rutas devuelven listas, todas van a querer los mismos parámetros `saltear` y `limite`. En vez de repetirlos, se escriben una vez en una función:

```python
def paginacion(saltear: int = 0, limite: Annotated[int, Query(le=100)] = 10) -> dict:
    return {"saltear": saltear, "limite": limite}
```

Y la ruta la **pide** con `Depends`:

```python
@router.get("")
def listar_tareas(pagina: Annotated[dict, Depends(paginacion)]):
    """Devuelve las tareas, de a una página por vez."""
    inicio = pagina["saltear"]
    return tareas[inicio : inicio + pagina["limite"]]
```

Cuando llega `GET /tareas?saltear=1&limite=1`:

1. FastAPI ve que `listar_tareas` necesita el resultado de `paginacion`.
2. Llama a `paginacion`, que a su vez recibe `saltear` y `limite` de la query, ya convertidos y validados como en cualquier ruta.
3. Le pasa el diccionario resultante a `listar_tareas` como `pagina`.

Las URLs y sus respuestas:

| URL | Resultado |
| --- | --- |
| `/tareas` | Las tres tareas (`limite` vale 10). |
| `/tareas?limite=2` | Las dos primeras. |
| `/tareas?saltear=1&limite=1` | Solo "Estudiar FastAPI". |
| `/tareas?limite=500` | `422`: `Input should be less than or equal to 100`. |

`Query(le=100)` agrega una regla a un parámetro de query, igual que `Field` en un modelo: así nadie pide un millón de tareas de una vez. Y en `/docs`, `saltear` y `limite` aparecen como parámetros de `GET /tareas`, porque FastAPI mira también adentro de las dependencias.

### Qué es `Annotated`

`Annotated[dict, Depends(paginacion)]` se lee así: "el tipo es `dict`, y además, para FastAPI, sale de `Depends(paginacion)`". `Annotated` viene de `typing` (la biblioteca estándar) y permite pegarle información extra a un type hint. Es la forma que recomienda la documentación de FastAPI, y por eso la usamos también en `Query`.

Los imports nuevos, arriba de `app/routers/tareas.py`:

```python
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, Query, status
```

## Una dependencia que busca la tarea

La función `buscar_tarea` que ya tenías **es** una dependencia perfecta: recibe un `tarea_id` y devuelve la tarea o lanza un `404`. Las dependencias pueden recibir parámetros de ruta, así que se puede usar tal cual:

```python
@router.get("/{tarea_id}", responses={404: {"description": "La tarea no existe"}})
def obtener_tarea(tarea: Annotated[dict, Depends(buscar_tarea)]):
    """Devuelve una tarea por su id."""
    return tarea


@router.put("/{tarea_id}", responses={404: {"description": "La tarea no existe"}})
def modificar_tarea(tarea: Annotated[dict, Depends(buscar_tarea)], datos: TareaCrear):
    """Reemplaza los datos de una tarea."""
    tarea["titulo"] = datos.titulo
    tarea["descripcion"] = datos.descripcion
    tarea["hecha"] = datos.hecha
    return tarea


@router.delete("/{tarea_id}", status_code=status.HTTP_204_NO_CONTENT, responses={404: {"description": "La tarea no existe"}})
def borrar_tarea(tarea: Annotated[dict, Depends(buscar_tarea)]):
    """Borra una tarea."""
    tareas.remove(tarea)
```

Ahora las rutas reciben **directamente la tarea**, no el id. FastAPI toma el `{tarea_id}` de la URL, se lo pasa a `buscar_tarea` y, si la encuentra, le entrega el resultado a la ruta. Si no existe, el `404` sale de la dependencia y la ruta ni se ejecuta. Lo podés comprobar: `/tareas/2` responde la tarea, `/tareas/99` responde `404`, y `/tareas/abc` responde `422`, igual que antes.

Las demás partes del archivo (la lista, `buscar_tarea` y `crear_tarea`) quedan como estaban.

## Para qué más sirve

Este patrón va a aparecer en el resto de la guía:

- **Base de datos (Capítulo 3):** una dependencia va a abrir una sesión con la base para cada pedido y cerrarla al terminar. Cada ruta que la necesite la pide con `Depends`.
- **Autenticación (Capítulo 5):** una dependencia va a leer el token del pedido y devolver el usuario actual, o responder `401`. Proteger una ruta va a ser tan simple como pedir esa dependencia.
- **Configuración:** en la próxima página, una dependencia va a entregar la configuración de la app.

Además, se puede aplicar una dependencia a **todas** las rutas de un router, sin usar su resultado, con `APIRouter(dependencies=[Depends(...)])`. Lo vas a probar en el ejercicio.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Lo más parecido en Express son los **middlewares** de ruta:

    ```javascript
    function buscarTarea(req, res, next) {
      const tarea = tareas.find(t => t.id === Number(req.params.tareaId))
      if (!tarea) return res.status(404).json({ detail: "Tarea no encontrada" })
      req.tarea = tarea
      next()
    }

    app.get("/tareas/:tareaId", buscarTarea, (req, res) => res.json(req.tarea))
    ```

    La diferencia: en Express el middleware deja el resultado colgado en `req` y nada te garantiza que exista; en FastAPI llega como un parámetro con nombre y tipo, y aparece en la documentación.

??? example "Ver comparación en Django ⬇️"

    Django no tiene un sistema equivalente. Lo más cercano son los decoradores como `@login_required`, los mixins de las class-based views, o en DRF las `permission_classes` y `get_object()`. En FastAPI, todo eso se resuelve con el mismo mecanismo: una función y `Depends`.

??? example "Ver comparación en Spring Boot ⬇️"

    Spring es *el* framework de inyección de dependencias de Java, pero inyecta **objetos** (beans) en clases, con `@Autowired` o por constructor. En FastAPI se inyecta el **resultado de una función**, por cada pedido.

    | Spring Boot | FastAPI |
    | --- | --- |
    | `@Autowired TareaService service` | `servicio: Annotated[..., Depends(obtener_servicio)]` |
    | `Pageable pageable` como parámetro | `pagina: Annotated[dict, Depends(paginacion)]` |
    | Un filtro o `HandlerInterceptor` | `APIRouter(dependencies=[Depends(...)])` |

## Mini ejercicio

Protegé el router de usuarios con una dependencia `verificar_clave` que lea el encabezado `X-Clave` y responda `401` con el detalle `"Clave incorrecta"` si no vale `"secreta"`. Aplicala a todo el router con `dependencies=`. Para leer un encabezado se usa `Header()`, así: `x_clave: Annotated[str, Header()]` (FastAPI convierte `x_clave` en el encabezado `X-Clave`).

??? success "Ver solución"

    ```python title="app/routers/usuarios.py (principio del archivo)"
    from typing import Annotated

    from fastapi import APIRouter, Depends, Header, HTTPException, status

    from app.modelos import UsuarioCrear


    def verificar_clave(x_clave: Annotated[str, Header()]):
        if x_clave != "secreta":
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Clave incorrecta")


    router = APIRouter(prefix="/usuarios", tags=["usuarios"], dependencies=[Depends(verificar_clave)])
    ```

    | Petición | Respuesta |
    | --- | --- |
    | `curl http://127.0.0.1:8000/usuarios` | `422`: falta el encabezado `x-clave` |
    | `curl -H "X-Clave: otra" ...` | `401` con `{"detail":"Clave incorrecta"}` |
    | `curl -H "X-Clave: secreta" ...` | `200` con la lista |

    Las rutas de `/tareas` siguen sin pedir clave: la dependencia es solo de ese router. Esto es una versión muy simplificada de la autenticación; la de verdad, con usuarios y tokens, llega en el Capítulo 5.

---

**Siguiente:** [Configuración con `pydantic-settings` →](12-pydantic-settings.md)
