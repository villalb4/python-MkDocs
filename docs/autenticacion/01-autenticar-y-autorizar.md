# Qué es autenticar y qué es autorizar

!!! info "En esta página"

    **Qué vas a aprender:** la diferencia entre autenticación y autorización, cuándo responder `401` y cuándo `403`, cómo una API sabe quién le habla en cada pedido, y qué vas a construir en este capítulo.

    **Qué necesitás:** haber terminado el [Capítulo 4](../docker/08-api-y-base-juntas.md) y conocer las [dependencias con `Depends`](../fastapi/11-depends.md).

## Dos preguntas distintas

Cuando alguien usa tu API, hay dos preguntas que responder, y conviene no mezclarlas:

- **Autenticación** (*authentication*): **¿quién sos?** Comprobar la identidad: que el que dice ser Ana, realmente es Ana.
- **Autorización** (*authorization*): **¿podés hacer esto?** Una vez que sabés quién es, decidir si tiene permiso para lo que pide.

Pensá en un edificio de oficinas. En la entrada te piden el documento: eso es **autenticar**. Ya adentro, tu tarjeta abre algunas puertas y otras no: eso es **autorizar**. Primero va siempre la autenticación: no se puede decidir qué puede hacer alguien sin saber quién es.

## `401` y `403`

Cada pregunta tiene su código de error:

| Código | Nombre | Significa | Ejemplo |
| --- | --- | --- | --- |
| `401` | Unauthorized | No sé quién sos. | No mandaste credenciales, o son inválidas. |
| `403` | Forbidden | Sé quién sos, pero no podés hacer esto. | Un usuario común intentando entrar a una ruta de administración. |

El nombre de `401` confunde: dice *Unauthorized*, pero en realidad significa "no autenticado". Una regla práctica: si **volviendo a identificarte** el pedido podría funcionar, es `401`; si no hay credencial que lo arregle porque te falta el permiso, es `403`.

## Cómo sabe la API quién le habla

HTTP **no tiene memoria**: cada pedido es independiente del anterior. La API no "recuerda" que hace un minuto Ana se identificó. Por eso, **cada pedido** tiene que traer una prueba de quién lo hace.

La forma más común en las APIs es un **token**: un texto que el servidor le entrega al usuario cuando se identifica, y que el cliente manda después en cada pedido, en un encabezado:

```http
GET /tareas HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

`Authorization` es el encabezado estándar para esto, y `Bearer` ("portador") indica que lo que sigue es un token: quien lo porta, tiene acceso. Por eso un token es tan valioso como una contraseña, y hay que cuidarlo igual.

!!! note "¿Y la `X-Clave` del Capítulo 2?"

    Esa clave era un secreto **compartido**: cualquiera que la conociera entraba, y la API no sabía quién era. Servía para entender las dependencias, pero no identifica a nadie. Con usuarios y tokens, cada pedido dice **qué usuario** lo hace.

## Un ejemplo de juguete

Para ver las dos preguntas funcionando, esta mini API tiene dos usuarios con tokens fijos, escritos en el código. **Nunca se hace así de verdad**: en las próximas páginas vas a reemplazarlo por contraseñas guardadas de forma segura y tokens firmados. Pero la estructura es exactamente la que vas a usar.

```python title="juguete.py"
from typing import Annotated

from fastapi import Depends, FastAPI, Header, HTTPException, status

app = FastAPI()

# Un "juguete": tokens fijos escritos en el código. Nunca se hace así de verdad.
TOKENS = {
    "token-de-ana": {"nombre": "Ana", "rol": "admin"},
    "token-de-juan": {"nombre": "Juan", "rol": "usuario"},
}


def usuario_actual(authorization: Annotated[str | None, Header()] = None) -> dict:
    """Autenticación: ¿quién sos?"""
    if authorization is None or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Falta el token")
    token = authorization.removeprefix("Bearer ")
    usuario = TOKENS.get(token)
    if usuario is None:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token inválido")
    return usuario


UsuarioDep = Annotated[dict, Depends(usuario_actual)]


@app.get("/perfil")
def ver_perfil(usuario: UsuarioDep):
    return {"hola": usuario["nombre"]}


@app.get("/admin")
def panel_admin(usuario: UsuarioDep):
    """Autorización: ¿podés hacer esto?"""
    if usuario["rol"] != "admin":
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Solo para administradores")
    return {"mensaje": f"Bienvenida al panel, {usuario['nombre']}"}
```

- **`usuario_actual`** es una dependencia que **autentica**: lee el encabezado `Authorization`, saca el token y busca de quién es. Si falta o no existe, `401`. Si existe, devuelve el usuario.
- **`panel_admin`** primero pide el usuario (autenticación, a través de la dependencia) y después **autoriza**: si su rol no es `admin`, `403`.
- `.removeprefix("Bearer ")` es un método de los strings que saca ese texto del principio.

Probalo con `uv run --with "fastapi[standard]" fastapi dev juguete.py` y `curl`:

| Pedido | Respuesta |
| --- | --- |
| `GET /perfil` sin token | `401` `{"detail":"Falta el token"}` |
| `GET /perfil` con `Bearer cualquier-cosa` | `401` `{"detail":"Token inválido"}` |
| `GET /perfil` con `Bearer token-de-juan` | `200` `{"hola":"Juan"}` |
| `GET /admin` con `Bearer token-de-juan` | `403` `{"detail":"Solo para administradores"}` |
| `GET /admin` con `Bearer token-de-ana` | `200` `{"mensaje":"Bienvenida al panel, Ana"}` |

Para mandar el encabezado con `curl`: `curl -H "Authorization: Bearer token-de-juan" http://127.0.0.1:8000/perfil`.

## Lo que vas a construir en este capítulo

A lo largo del capítulo vas a reemplazar el juguete por un sistema de verdad, en la API de tareas:

1. **Contraseñas guardadas de forma segura**, con *hashing* (nunca en texto plano).
2. **Registro**: crear un usuario con email y contraseña.
3. **Login**: comprobar la contraseña y entregar un **token JWT**, firmado para que nadie lo pueda falsificar.
4. **Rutas protegidas** con una dependencia `usuario_actual`, como la de arriba, pero leyendo el JWT.
5. **Roles y permisos**, para separar usuarios comunes de administradores.
6. **Expiración de tokens** y buenas prácticas de seguridad.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Express, la autenticación se hace con un *middleware* que lee `req.headers.authorization`, valida el token y deja el usuario en `req.user` (por ejemplo, con Passport o `express-jwt`). La autorización suele ser otro middleware que revisa `req.user.rol`. En FastAPI, las dos cosas son dependencias, y el usuario llega como parámetro.

??? example "Ver comparación en Django ⬇️"

    Django trae autenticación integrada: `request.user` ya está disponible, `@login_required` exige estar autenticado y `@permission_required` o los grupos manejan la autorización. En DRF, `authentication_classes` y `permission_classes` separan exactamente estas dos preguntas. En FastAPI no viene nada armado: lo construís con dependencias, como en esta página.

??? example "Ver comparación en Spring Boot ⬇️"

    Spring Security separa los dos conceptos de la misma forma: la *authentication* llena el `SecurityContext` con el usuario, y la *authorization* se configura con reglas como `.hasRole("ADMIN")` o `@PreAuthorize`. Un `AuthenticationEntryPoint` responde `401` y un `AccessDeniedHandler`, `403`.

## Mini ejercicio

¿Qué código corresponde a cada caso: `401` o `403`?

1. Un pedido a `/tareas` sin el encabezado `Authorization`.
2. Juan intenta borrar una tarea que pertenece a Ana.
3. Un pedido con un token que expiró ayer.
4. Un usuario común intenta ver la lista de todos los usuarios, que es solo para administradores.

??? success "Ver solución"

    1. **`401`**: no hay credenciales, no se sabe quién es.
    2. **`403`**: se sabe que es Juan, pero no tiene permiso sobre esa tarea. (Algunas APIs responden `404` en este caso, para no revelar que la tarea existe; lo vas a ver más adelante.)
    3. **`401`**: el token ya no sirve para identificarlo. Con un token nuevo, el pedido funcionaría.
    4. **`403`**: está identificado, pero su rol no alcanza. Ningún token suyo lo arreglaría.

---

**Siguiente:** [Por qué nunca guardar contraseñas en texto plano →](02-contrasenas-texto-plano.md)
