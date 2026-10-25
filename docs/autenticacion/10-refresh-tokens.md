# Expiración de tokens y refresh tokens

!!! info "En esta página"

    **Qué vas a aprender:** por qué los tokens tienen que vencer, cómo combinar un token de acceso corto con un **token de refresco** largo, y cómo agregar una ruta para renovar los tokens sin volver a pedir la contraseña.

    **Qué necesitás:** [Roles y permisos básicos](09-roles.md) y [Qué es un JWT y cómo se usa](07-jwt.md).

## El dilema de la duración

Tu token de acceso vence a los 30 minutos. ¿Por qué no hacerlo durar un mes, así nadie tiene que volver a iniciar sesión?

Porque un JWT, una vez entregado, **no se puede anular**: es válido hasta que vence, lo tenga quien lo tenga. Si alguien lo roba (de una computadora compartida, de un log, de una extensión maliciosa del navegador), puede usarlo hasta su vencimiento. Cuanto más dura, más tiempo tiene el atacante.

Pero si dura 15 minutos, el usuario tendría que escribir la contraseña cada 15 minutos. Ninguna de las dos opciones sirve sola.

## Dos tokens con dos trabajos

La solución habitual es entregar **dos** tokens en el login:

| | Token de acceso | Token de refresco |
| --- | --- | --- |
| Para qué sirve | Usar la API (va en cada pedido) | Pedir tokens nuevos (solo a una ruta) |
| Cuánto dura | Poco: 15 minutos | Mucho: 7 días |
| Cuánto se expone | Mucho: viaja en cada pedido | Poco: viaja solo al renovar |

El cliente usa el token de acceso hasta que vence (recibe un `401`). Entonces manda el de refresco a `POST /auth/refrescar` y recibe un par nuevo, sin molestar al usuario. Si alguien roba un token de acceso, le sirve como mucho 15 minutos. Y como el de refresco viaja mucho menos, es más difícil de robar.

Para que uno no se pueda usar en lugar del otro, cada token lleva un dato extra con su **tipo**: `"acceso"` o `"refresco"`.

## La configuración

En `Configuracion` (`app/config.py`), acortá el acceso y agregá la duración del refresco:

```python
minutos_token: int = 15
dias_refresco: int = 7
```

## Crear los dos tipos de token

En `app/seguridad.py`, reemplazá `crear_token` por estas tres funciones:

```python title="app/seguridad.py"
def crear_token(usuario_id: int, tipo: str, duracion: timedelta) -> str:
    datos = {"sub": str(usuario_id), "tipo": tipo, "exp": datetime.now(timezone.utc) + duracion}
    return jwt.encode(datos, configuracion.clave_jwt, algorithm=configuracion.algoritmo_jwt)


def crear_token_acceso(usuario_id: int) -> str:
    return crear_token(usuario_id, "acceso", timedelta(minutes=configuracion.minutos_token))


def crear_token_refresco(usuario_id: int) -> str:
    return crear_token(usuario_id, "refresco", timedelta(days=configuracion.dias_refresco))
```

`crear_token` hace el trabajo; las otras dos solo fijan el tipo y la duración. `leer_token` queda igual.

## Las rutas protegidas solo aceptan acceso

En `app/dependencias.py`, `obtener_usuario_actual` tiene que rechazar un token de refresco. Agregá el chequeo después del `try`:

```python
    try:
        datos = leer_token(token)
    except jwt.InvalidTokenError:
        raise error
    if datos.get("tipo") != "acceso":
        raise error
```

`datos.get("tipo")` devuelve `None` si el token no tiene tipo (por ejemplo, uno creado antes de este cambio), así que esos también se rechazan.

## El login devuelve los dos

En `app/esquemas.py`, agregá el token de refresco a la respuesta y un esquema para el pedido de renovación:

```python title="app/esquemas.py"
class Token(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"


class PedidoRefresco(BaseModel):
    refresh_token: str
```

En `app/routers/auth.py`, importá `jwt`, `PedidoRefresco`, `crear_token_acceso`, `crear_token_refresco` y `leer_token` (en vez de `crear_token`), y cambiá el final del login:

```python
    return Token(
        access_token=crear_token_acceso(usuario.id),
        refresh_token=crear_token_refresco(usuario.id),
    )
```

## La ruta para renovar

Agregala en `app/routers/auth.py`:

```python title="app/routers/auth.py"
@router.post("/refrescar", response_model=Token)
async def refrescar(pedido: PedidoRefresco, sesion: SesionDep):
    """Cambia un token de refresco por un par de tokens nuevos."""
    error = HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token de refresco inválido")
    try:
        datos = leer_token(pedido.refresh_token)
    except jwt.InvalidTokenError:
        raise error
    if datos.get("tipo") != "refresco":
        raise error
    usuario = await sesion.get(Usuario, int(datos["sub"]))
    if usuario is None or not usuario.activo:
        raise error
    return Token(
        access_token=crear_token_acceso(usuario.id),
        refresh_token=crear_token_refresco(usuario.id),
    )
```

Es una ruta **pública** (no pide `UsuarioActualDep`): justamente se usa cuando el token de acceso ya venció. La protege el propio token de refresco, que se verifica igual que el de acceso, pero exigiendo el tipo `"refresco"`. Y como se vuelve a buscar el usuario, una cuenta desactivada ya no puede renovar, aunque tenga un token de refresco válido.

Devolver también un token de refresco **nuevo** (en vez de solo el de acceso) se llama **rotación**: mientras el usuario use la app, su sesión se extiende sola.

## Probarlo

| Pedido | Respuesta |
| --- | --- |
| `POST /auth/login` | `access_token`, `refresh_token` y `token_type` |
| `GET /auth/yo` con el token de acceso | `200` |
| `GET /auth/yo` con el token de **refresco** | `401` `No se pudo validar el token` |
| `POST /auth/refrescar` con el token de **acceso** | `401` `Token de refresco inválido` |
| `POST /auth/refrescar` con el token de refresco | `200` con un par nuevo |
| Lo mismo, con un refresco vencido o de una cuenta desactivada | `401` `Token de refresco inválido` |

Para `/auth/refrescar`, el cuerpo es JSON: `{"refresh_token": "..."}`.

!!! warning "Los tokens de refresco tampoco se pueden anular"

    Con esta implementación, un token de refresco robado sirve durante 7 días (salvo que desactives la cuenta). Para poder cerrar sesiones de verdad (por ejemplo, "cerrar sesión en todos los dispositivos"), los tokens de refresco se guardan en la base con un identificador, y la ruta de renovación comprueba que sigan habilitados. Es la mejora natural de este esquema cuando la app lo necesite.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    El esquema es el mismo en Node: `jwt.sign` con `expiresIn: "15m"` para el acceso, otro con `expiresIn: "7d"` para el refresco, y una ruta `/refresh`. En muchas apps el token de refresco se manda en una cookie `httpOnly` en vez de en el cuerpo, algo que vas a ver en la próxima página.

??? example "Ver comparación en Django ⬇️"

    djangorestframework-simplejwt trae exactamente este esquema: `TokenObtainPairView` devuelve `access` y `refresh`, y `TokenRefreshView` renueva. Con `ROTATE_REFRESH_TOKENS = True` hace la rotación, y con la app `token_blacklist` permite anular tokens de refresco, la mejora del recuadro de arriba.

??? example "Ver comparación en Spring Boot ⬇️"

    En el mundo Spring, el estándar OAuth2 que implementa Spring Authorization Server incluye el *refresh token grant*, con la misma división: un *access token* corto y un *refresh token* largo, que se cambia en el endpoint de tokens.

## Mini ejercicio

Comprobá que el esquema funciona de punta a punta: levantá la API con un token de acceso de **1 minuto** (con la variable de entorno `MINUTOS_TOKEN=1`), iniciá sesión, esperá a que venza y renovalo.

??? success "Ver solución"

    ```bash
    MINUTOS_TOKEN=1 uv run fastapi dev app/main.py
    ```

    (En PowerShell: `$env:MINUTOS_TOKEN = "1"` y después el comando.) Con el `access_token` del login:

    1. Recién creado, `GET /auth/yo` responde `200`.
    2. Después de un poco más de un minuto, el mismo pedido responde `401` `No se pudo validar el token`: venció.
    3. `POST /auth/refrescar` con el `refresh_token` devuelve un `access_token` nuevo, y con él `GET /auth/yo` vuelve a responder `200`. El usuario nunca tuvo que escribir su contraseña otra vez.

    La variable de la terminal pisa el valor por defecto de `Configuracion`, como viste en el Capítulo 2.

---

**Siguiente:** [Dónde guardar el token en el frontend →](11-donde-guardar-token.md)
