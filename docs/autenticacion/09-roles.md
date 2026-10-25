# Roles y permisos básicos

!!! info "En esta página"

    **Qué vas a aprender:** cómo usar el campo `rol` para separar usuarios comunes de administradores, con una dependencia que exige ser administrador, y cómo dar ese rol de forma segura.

    **Qué necesitás:** [Proteger rutas con `Depends`](08-proteger-rutas.md).

## Autorización: el segundo paso

Con `UsuarioActualDep` resolviste la **autenticación**: cada ruta sabe quién hace el pedido. Ahora toca la **autorización**: decidir qué puede hacer cada uno.

La forma más simple es con **roles**: cada usuario tiene uno, y ciertas acciones están reservadas para algunos roles. Tu modelo ya tiene el campo `rol`, con `"usuario"` por defecto. Vas a sumar un segundo rol, `"admin"`, para quienes administran la app: por ejemplo, ver la lista de todos los usuarios o las tareas de cualquiera.

## Una dependencia que exige ser administrador

Agregá al final de `app/dependencias.py`:

```python title="app/dependencias.py (agregar al final)"
def requerir_admin(usuario: UsuarioActualDep) -> Usuario:
    if usuario.rol != "admin":
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Se necesita ser administrador")
    return usuario


AdminDep = Annotated[Usuario, Depends(requerir_admin)]
```

Fijate cómo se encadenan las dependencias: `requerir_admin` pide `UsuarioActualDep`, que a su vez pide el token. FastAPI resuelve la cadena entera:

1. Sin token válido → `401` (lo responde `obtener_usuario_actual`).
2. Con token, pero sin el rol → `403` (lo responde `requerir_admin`).
3. Con token y rol `admin` → la ruta se ejecuta y recibe al usuario.

Son exactamente las dos preguntas de la primera página del capítulo, cada una en su dependencia.

## Proteger el router de usuarios

El router de usuarios muestra datos de todos: tiene que ser solo para administradores. En `app/routers/usuarios.py`, cambiá la dependencia del router:

```python
from app.dependencias import requerir_admin

router = APIRouter(prefix="/usuarios", tags=["usuarios"], dependencies=[Depends(requerir_admin)])
```

Con eso, las tres rutas del router (listar usuarios, ver un usuario con sus tareas y ver las tareas de un usuario) exigen ser administrador. Las rutas de `/tareas` no cambian: cada usuario, incluido el administrador, sigue viendo ahí **solo sus propias** tareas.

## ¿Cómo se llega a ser administrador?

Por la API, **de ninguna forma**, y es a propósito. Como viste en el registro, nadie puede elegir su rol al crearse la cuenta. El primer administrador se crea a mano, directamente en la base:

```bash
docker compose exec db psql -U tareas -c "UPDATE usuarios SET rol = 'admin' WHERE email = 'ana@ejemplo.com';"
```

Probalo:

| Pedido | Respuesta |
| --- | --- |
| `GET /usuarios` con el token de Ana, antes del `UPDATE` | `403` `Se necesita ser administrador` |
| `GET /usuarios` con el **mismo** token, después del `UPDATE` | `200` con la lista de usuarios |
| `GET /usuarios/{id de Juan}/tareas` como Ana (admin) | `200` con las tareas de Juan |
| Lo mismo, con el token de Juan | `403` |

No hizo falta volver a iniciar sesión: el rol no está guardado en el token, sino en la base, y `obtener_usuario_actual` lee el usuario en cada pedido. Por eso un cambio de rol (para dar o para quitar permisos) se aplica en el acto.

!!! note "¿Y si pongo el rol en el token?"

    Se puede, y ahorra una consulta a la base por pedido. Pero entonces, si le quitás el rol de administrador a alguien, su token viejo lo sigue diciendo hasta que venza. Para una app como esta, leer el usuario de la base en cada pedido es más simple y más seguro.

## Más allá de dos roles

Este esquema alcanza para muchas apps. Cuando los permisos se complican (editores que pueden publicar pero no borrar, moderadores por sección...), se suele pasar a **permisos** individuales asignados a roles, o a reglas que dependen del recurso. La idea de fondo no cambia: una dependencia que recibe al usuario y decide si sigue o responde `403`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Express sería un middleware después del de autenticación:

    ```javascript
    function requerirAdmin(req, res, next) {
      if (req.user.rol !== "admin") return res.status(403).json({ detail: "Se necesita ser administrador" })
      next()
    }

    app.use("/usuarios", autenticar, requerirAdmin, routerUsuarios)
    ```

    `dependencies=[Depends(requerir_admin)]` en el router cumple el rol de ese `app.use` con los dos middlewares.

??? example "Ver comparación en Django ⬇️"

    Django trae permisos y grupos integrados, y el flag `is_staff` / `is_superuser`. En DRF, `requerir_admin` equivale a `permission_classes = [IsAdminUser]`, y para reglas propias se escribe una clase con `has_permission()`. El primer superusuario se crea con `python manage.py createsuperuser`, la versión de Django del `UPDATE` de esta página.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring Security se configura con `.requestMatchers("/usuarios/**").hasRole("ADMIN")` o con `@PreAuthorize("hasRole('ADMIN')")` en el método. Los roles se cargan en el `UserDetails` como *authorities*, igual que acá el rol se lee del usuario.

## Mini ejercicio

Agregá una ruta `PATCH /usuarios/{usuario_id}/rol` para que un administrador cambie el rol de otro usuario. El cuerpo tiene un único campo `rol`, que solo puede valer `"usuario"` o `"admin"`. Un administrador no puede cambiar su propio rol (para no quedarse sin administradores por accidente).

Pista: para limitar los valores posibles, usá `Literal` de `typing`: `rol: Literal["usuario", "admin"]`.

??? success "Ver solución"

    ```python title="app/esquemas.py (agregar; con from typing import Literal)"
    class CambioDeRol(BaseModel):
        rol: Literal["usuario", "admin"]
    ```

    ```python title="app/routers/usuarios.py (agregar; con los imports de CambioDeRol y AdminDep)"
    @router.patch("/{usuario_id}/rol", response_model=UsuarioRespuesta)
    async def cambiar_rol(usuario_id: int, datos: CambioDeRol, admin: AdminDep, sesion: SesionDep):
        """Cambia el rol de un usuario."""
        usuario = await sesion.get(Usuario, usuario_id)
        if usuario is None:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Usuario no encontrado")
        if usuario.id == admin.id:
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="No podés cambiar tu propio rol")
        usuario.rol = datos.rol
        await sesion.commit()
        return usuario
    ```

    Con `"rol": "superusuario"`, Pydantic responde `422` con `Input should be 'usuario' or 'admin'`. Cambiar el propio rol da `400`. El router ya exige ser administrador, pero la ruta pide `AdminDep` igual, porque necesita saber **quién** es el administrador para compararlo. FastAPI no ejecuta la dependencia dos veces: reusa el resultado dentro del mismo pedido.

---

**Siguiente:** [Expiración de tokens y refresh tokens →](10-refresh-tokens.md)
