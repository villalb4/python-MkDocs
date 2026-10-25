# Proteger rutas con `Depends`

!!! info "En esta página"

    **Qué vas a aprender:** cómo crear una dependencia que lee el token, lo verifica y devuelve el usuario actual, y cómo usarla para proteger rutas y hacer que cada usuario vea solo sus propias tareas.

    **Qué necesitás:** [Qué es un JWT y cómo se usa](07-jwt.md) y [Inyección de dependencias con `Depends`](../fastapi/11-depends.md).

## La pieza que falta

El login ya entrega un token, pero ninguna ruta lo pide. Falta la pieza que cierra el circuito: una **dependencia** que, en cada pedido, lea el token del encabezado `Authorization`, lo verifique y devuelva el usuario. Es la misma idea que el `usuario_actual` del ejemplo de juguete de la primera página, pero con un JWT de verdad.

## La dependencia del usuario actual

Creá `app/dependencias.py`:

```python title="app/dependencias.py"
from typing import Annotated

import jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

from app.db import SesionDep
from app.modelos import Usuario
from app.seguridad import leer_token

esquema_oauth2 = OAuth2PasswordBearer(tokenUrl="/auth/login")


async def obtener_usuario_actual(token: Annotated[str, Depends(esquema_oauth2)], sesion: SesionDep) -> Usuario:
    error = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="No se pudo validar el token",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        datos = leer_token(token)
    except jwt.InvalidTokenError:
        raise error
    usuario = await sesion.get(Usuario, int(datos["sub"]))
    if usuario is None or not usuario.activo:
        raise error
    return usuario


UsuarioActualDep = Annotated[Usuario, Depends(obtener_usuario_actual)]
```

Parte por parte:

- **`OAuth2PasswordBearer(tokenUrl="/auth/login")`** es una dependencia de FastAPI que lee el encabezado `Authorization: Bearer ...` y devuelve el token. Si falta, responde `401` con `{"detail":"Not authenticated"}` por su cuenta. `tokenUrl` le dice a `/docs` dónde está el login.
- **`leer_token`** verifica la firma y el vencimiento. Si el token es inválido, alterado o venció, PyJWT lanza una excepción. Todas heredan de `jwt.InvalidTokenError`, así que un solo `except` las atrapa a todas.
- **Se busca al usuario** con el `sub` del token (convertido de vuelta a número). Si ya no existe o está desactivado, el token no sirve, aunque la firma sea válida.
- **Un solo error para todo**: no hace falta contarle al cliente si el token venció o fue alterado. El encabezado `WWW-Authenticate: Bearer` es el que indica el estándar para un `401` de este tipo.
- **`UsuarioActualDep`** es el atajo para pedir el usuario en cualquier ruta, como `SesionDep`.

## "¿Quién soy?"

La forma más simple de probarla es una ruta que devuelva el usuario actual. Agregala al final de `app/routers/auth.py` (importando `UsuarioActualDep` de `app.dependencias`):

```python title="app/routers/auth.py (agregar al final)"
@router.get("/yo", response_model=UsuarioRespuesta)
async def ver_mi_usuario(usuario: UsuarioActualDep):
    """Devuelve los datos del usuario que hizo el pedido."""
    return usuario
```

Con pedir `UsuarioActualDep`, la ruta queda protegida: sin un token válido, nunca se ejecuta.

## Cada usuario, sus tareas

Ahora la parte importante: las tareas tienen que pertenecer al usuario que las crea, y nadie tiene que ver las de otro. En `app/routers/tareas.py`, importá `UsuarioActualDep` (y sacá `Usuario` y `verificar_usuario`, que ya no hacen falta). Estos son los cambios.

**Buscar una tarea solo si es tuya:**

```python
async def buscar_tarea(tarea_id: int, sesion: SesionDep, usuario: UsuarioActualDep) -> Tarea:
    tarea = await sesion.get(Tarea, tarea_id)
    if tarea is None or tarea.usuario_id != usuario.id:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Tarea no encontrada")
    return tarea
```

Como `buscar_tarea` pide el usuario, **todas** las rutas que usan `TareaDep` (ver, modificar y borrar) quedan protegidas y filtradas por dueño, sin tocarlas.

**Listar solo las tuyas:**

```python
@router.get("", response_model=list[TareaRespuesta])
async def listar_tareas(sesion: SesionDep, usuario: UsuarioActualDep, pagina: Annotated[dict, Depends(paginacion)]):
    """Devuelve las tareas del usuario, de a una página por vez."""
    consulta = (
        select(Tarea)
        .where(Tarea.usuario_id == usuario.id)
        .order_by(Tarea.id)
        .offset(pagina["saltear"])
        .limit(pagina["limite"])
    )
    resultado = await sesion.scalars(consulta)
    return resultado.all()
```

Los paréntesis permiten partir la consulta en varios renglones.

**Crear a tu nombre:** el `usuario_id` ya no lo manda el cliente, sale del token. En `crear_tarea`, pedí `usuario: UsuarioActualDep` y usá `usuario_id=usuario.id` al crear la tarea. En `modificar_tarea`, sacá las líneas de `verificar_usuario` y de `tarea.usuario_id`. Y en `app/esquemas.py`, sacá `usuario_id` de `TareaCrear` (en `TareaRespuesta` se queda).

**El router de usuarios:** reemplazá la `X-Clave` por la autenticación. Borrá la función `verificar_clave` (con sus imports) y protegé el router con la dependencia nueva:

```python
router = APIRouter(prefix="/usuarios", tags=["usuarios"], dependencies=[Depends(obtener_usuario_actual)])
```

## Probarlo

**Con `curl`**, primero obtené un token y después mandalo en cada pedido:

```bash
curl -X POST http://127.0.0.1:8000/auth/login -d "username=ana@ejemplo.com&password=una-clave-larga"
curl -H "Authorization: Bearer pegá-acá-el-token" http://127.0.0.1:8000/tareas
```

| Pedido | Respuesta |
| --- | --- |
| `GET /tareas` sin token | `401` `{"detail":"Not authenticated"}` |
| `GET /tareas` con un token inventado o vencido | `401` `{"detail":"No se pudo validar el token"}` |
| `GET /auth/yo` con el token de Ana | `200` con los datos de Ana |
| `GET /tareas` con el token de Ana | Solo las tareas de Ana |
| Ana pide (o intenta borrar) una tarea de Juan | `404` `Tarea no encontrada` |

**Desde `/docs`** es mucho más cómodo: arriba a la derecha apareció un botón **Authorize**, y las rutas protegidas tienen un candado. Hacé clic en **Authorize**, completá `username` (tu email) y `password`, y apretá **Authorize**. Desde ese momento, `/docs` manda el token solo en cada pedido: probá **GET /auth/yo** y vas a ver tus datos.

!!! note "¿Por qué `404` y no `403`?"

    Cuando Ana pide una tarea de Juan, la API responde `404` en vez de `403`. Con un `403`, Ana sabría que esa tarea **existe**. Con un `404`, para ella es como si no existiera, que es justamente la idea: las tareas de otros no forman parte de su mundo.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Express sería un middleware que lee `req.headers.authorization`, verifica con `jwt.verify` y deja el usuario en `req.user` (lo que hace `passport-jwt`). Después, en cada consulta filtrás por `req.user.id`. En FastAPI el usuario llega como parámetro con tipo, y si te olvidás de pedirlo, la ruta simplemente no tiene usuario, en vez de tener un `req.user` indefinido.

??? example "Ver comparación en Django ⬇️"

    En DRF, `authentication_classes` (con simplejwt) y `permission_classes = [IsAuthenticated]` cumplen el rol de `UsuarioActualDep`, y filtrar las tareas por dueño se hace en `get_queryset()` con `Tarea.objects.filter(usuario=self.request.user)`.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring Security, un filtro de JWT llena el `SecurityContext`, y en el controller recibís el usuario con `@AuthenticationPrincipal`. Las rutas se protegen con `.anyRequest().authenticated()`. El `401` lo responde el `AuthenticationEntryPoint`, que también agrega `WWW-Authenticate`.

## Mini ejercicio

Agregá una ruta `DELETE /auth/yo` que **desactive** la cuenta del usuario que hace el pedido (sin borrarla), respondiendo `204`. Después comprobá qué pasa con su token y con su próximo login.

??? success "Ver solución"

    ```python
    @router.delete("/yo", status_code=status.HTTP_204_NO_CONTENT)
    async def desactivar_mi_cuenta(usuario: UsuarioActualDep, sesion: SesionDep):
        """Desactiva la cuenta del usuario que hizo el pedido."""
        usuario.activo = False
        await sesion.commit()
    ```

    Después del `DELETE`, el **mismo token** deja de funcionar: `GET /auth/yo` responde `401` `No se pudo validar el token`, porque `obtener_usuario_actual` revisa `activo` en cada pedido. Y el login responde `403` `La cuenta está desactivada`. Revisar el usuario en la base en cada pedido tiene esta ventaja: un token válido no alcanza si la cuenta ya no está habilitada.

---

**Siguiente:** [Roles y permisos básicos →](09-roles.md)
