# Dónde guardar el token en el frontend (cookies httpOnly vs localStorage)

!!! info "En esta página"

    **Qué vas a aprender:** dónde puede guardar los tokens una aplicación web, qué riesgo tiene cada opción, y cómo hacer que tu API mande el token de refresco en una cookie `httpOnly`, que JavaScript no puede leer.

    **Qué necesitás:** [Expiración de tokens y refresh tokens](10-refresh-tokens.md) y [CORS](../fastapi/13-cors.md).

## El problema

Tu API entrega tokens; el **frontend** (la página web que usa la API) tiene que guardarlos en algún lado para mandarlos en cada pedido. Esta guía no trata del frontend, pero esta decisión afecta cómo se diseña la API, así que conviene entender las opciones.

El riesgo principal en el navegador se llama **XSS** (*cross-site scripting*): que, por un error en la página o en alguna librería que usa, se ejecute JavaScript de un atacante. Ese código puede hacer todo lo que haga el JavaScript de tu página, incluido leer lo que esté guardado y mandarlo a otro lado.

## Las opciones

| Dónde | ¿JavaScript lo puede leer? | ¿Sobrevive a recargar la página? | Riesgo principal |
| --- | --- | --- | --- |
| `localStorage` | Sí | Sí | XSS: el token se roba fácil y dura mucho |
| Una variable en memoria | Sí, pero solo mientras la página está abierta | No | XSS, pero solo mientras la página está abierta |
| Cookie `httpOnly` | **No** | Sí | CSRF (que se mitiga con `SameSite`) |

**`localStorage`** es lo más fácil, y por eso aparece en muchos tutoriales, pero es la opción más expuesta: cualquier script que corra en la página puede hacer `localStorage.getItem("token")`.

Una **cookie `httpOnly`** la guarda el navegador, la manda **solo** en los pedidos a tu API, y JavaScript **no la puede leer** (`document.cookie` no la muestra). Un XSS no la puede robar. Su riesgo es otro, el **CSRF**: que otro sitio provoque un pedido a tu API y el navegador agregue la cookie solo. Para eso existe el atributo `SameSite`, que le dice al navegador cuándo **no** mandarla.

## La combinación recomendada

Una forma muy usada de aprovechar lo mejor de cada opción, y que encaja con los dos tokens de la página anterior:

- **El token de acceso, en memoria.** Dura 15 minutos y viaja en el encabezado `Authorization`. Si la página se recarga, se pierde, y no importa:
- **el token de refresco, en una cookie `httpOnly`.** Al recargar, el frontend llama a `/auth/refrescar`, el navegador manda la cookie solo, y la API responde con un token de acceso nuevo.

Así, el token largo (el más valioso) es inaccesible para JavaScript, y el corto vive lo menos posible.

## Mandar la cookie desde la API

En `app/routers/auth.py`, agregá `Cookie` y `Response` a los imports de `fastapi`, importá `obtener_configuracion`, y creá una función que arme la cookie:

```python title="app/routers/auth.py"
router = APIRouter(prefix="/auth", tags=["auth"])
configuracion = obtener_configuracion()


def guardar_cookie_refresco(respuesta: Response, token: str) -> None:
    respuesta.set_cookie(
        key="refresh_token",
        value=token,
        httponly=True,
        secure=True,
        samesite="strict",
        path="/auth",
        max_age=configuracion.dias_refresco * 24 * 60 * 60,
    )
```

| Atributo | Para qué |
| --- | --- |
| `httponly=True` | JavaScript no la puede leer. |
| `secure=True` | Solo viaja por HTTPS. Los navegadores hacen una excepción para `localhost` y `127.0.0.1`, así que en desarrollo funciona igual. |
| `samesite="strict"` | El navegador no la manda en pedidos que vienen de **otro sitio**: protege del CSRF. |
| `path="/auth"` | Solo se manda a las rutas que empiezan con `/auth`. El resto de la API no la recibe. |
| `max_age` | Cuántos segundos dura: los mismos 7 días que el token. |

**En el login**, pedí `respuesta: Response` como parámetro (FastAPI te da el objeto de la respuesta para que le agregues cosas) y guardá la cookie antes de responder:

```python
    token_refresco = crear_token_refresco(usuario.id)
    guardar_cookie_refresco(respuesta, token_refresco)
    return Token(access_token=crear_token_acceso(usuario.id))
```

Fijate que el token de refresco ya **no** va en el JSON: si fuera, JavaScript lo podría leer y la cookie no serviría de nada. En `app/esquemas.py`, sacá `refresh_token` de `Token` y borrá `PedidoRefresco`.

**En `/auth/refrescar`**, el token ahora llega en la cookie. `Cookie()` funciona como `Header()`:

```python title="app/routers/auth.py"
@router.post("/refrescar", response_model=Token)
async def refrescar(
    sesion: SesionDep,
    respuesta: Response,
    refresh_token: Annotated[str | None, Cookie()] = None,
):
    """Cambia el token de refresco de la cookie por un par de tokens nuevos."""
    error = HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token de refresco inválido")
    if refresh_token is None:
        raise error
    try:
        datos = leer_token(refresh_token)
    except jwt.InvalidTokenError:
        raise error
    if datos.get("tipo") != "refresco":
        raise error
    usuario = await sesion.get(Usuario, int(datos["sub"]))
    if usuario is None or not usuario.activo:
        raise error
    token_refresco = crear_token_refresco(usuario.id)
    guardar_cookie_refresco(respuesta, token_refresco)
    return Token(access_token=crear_token_acceso(usuario.id))


@router.post("/logout", status_code=status.HTTP_204_NO_CONTENT)
async def cerrar_sesion(respuesta: Response):
    """Borra la cookie con el token de refresco."""
    respuesta.delete_cookie(key="refresh_token", path="/auth", httponly=True, secure=True, samesite="strict")
```

`/auth/logout` le pide al navegador que borre la cookie. Como JavaScript no puede tocar una cookie `httpOnly`, cerrar sesión **tiene** que pasar por la API.

**CORS:** para que un frontend en otro origen pueda mandar y recibir cookies, en `app/main.py` agregá `allow_credentials=True` al `CORSMiddleware`. Con credenciales, `allow_origins` no puede ser `["*"]`: tiene que listar los orígenes reales, como ya hacés.

## Probarlo

Con `curl`, `-c` guarda las cookies que manda la API en un archivo y `-b` las vuelve a mandar:

```bash
curl -i -c cookies.txt -X POST http://127.0.0.1:8000/auth/login -d "username=ana@ejemplo.com&password=una-clave-larga"
```

```http
HTTP/1.1 200 OK
set-cookie: refresh_token=eyJhbGciOi...; HttpOnly; Max-Age=604800; Path=/auth; SameSite=strict; Secure

{"access_token":"eyJhbGciOi...","token_type":"bearer"}
```

Después, `curl -b cookies.txt -c cookies.txt -X POST http://127.0.0.1:8000/auth/refrescar` devuelve un token de acceso nuevo. Sin la cookie responde `401`, y después de `/auth/logout` también.

En un navegador, desde la consola de la página `/docs` (que está en el mismo origen que la API), el login funciona y `/auth/refrescar` también, porque el navegador manda la cookie solo... pero `document.cookie` devuelve `""`. La cookie existe, pero JavaScript no la ve.

!!! warning "Frontend y API en dominios distintos"

    `SameSite=strict` funciona cuando el frontend y la API están en el mismo **sitio**: por ejemplo, `app.ejemplo.com` y `api.ejemplo.com` (los subdominios y los puertos no cuentan). Si estuvieran en dominios distintos (`miapp.com` y `api-de-otro.com`), el navegador no mandaría la cookie, y habría que usar `samesite="none"`, que requiere `secure=True` y otras defensas contra CSRF. La forma más simple de evitar el problema es publicar los dos bajo el mismo dominio.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Express: `res.cookie("refresh_token", token, { httpOnly: true, secure: true, sameSite: "strict", path: "/auth" })`, y para leerla, `req.cookies.refresh_token` con el middleware `cookie-parser`. En CORS, `credentials: true`; y en el frontend, `fetch(url, { credentials: "include" })` cuando la API está en otro origen.

??? example "Ver comparación en Django ⬇️"

    Las sesiones clásicas de Django ya usan una cookie `httpOnly` (`sessionid`), con `SESSION_COOKIE_SECURE` y `SESSION_COOKIE_SAMESITE`, y Django trae protección CSRF integrada. Para JWT en cookies con DRF, se configura simplejwt para leer el token de una cookie en vez del encabezado.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring se arma con `ResponseCookie.from("refresh_token", token).httpOnly(true).secure(true).sameSite("Strict").path("/auth")` y se lee con `@CookieValue`. Spring Security también trae protección CSRF, que se suele activar cuando la autenticación depende de cookies.

## Mini ejercicio

Levantá la API, abrí `http://127.0.0.1:8000/docs` y, en la consola del navegador (en las herramientas de desarrollo), ejecutá un login con `fetch`. Después comprobá qué devuelve `document.cookie` y si `/auth/refrescar` funciona.

```javascript
await fetch("/auth/login", {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: "username=ana%40ejemplo.com&password=una-clave-larga",
})
```

??? success "Ver solución"

    ```javascript
    document.cookie                                    // ""
    const r = await fetch("/auth/refrescar", { method: "POST" })
    r.status                                           // 200
    ```

    `document.cookie` está vacío aunque la cookie existe: es `httpOnly`. Y aun así, `/auth/refrescar` responde `200` con un token de acceso nuevo, porque el navegador manda la cookie solo. Un script malicioso en la página podría llamar a `/auth/refrescar`, pero **no podría llevarse** el token de refresco a otro lado. Si después ejecutás `await fetch("/auth/logout", { method: "POST" })`, el siguiente `/auth/refrescar` responde `401`.

    `%40` es la forma de escribir `@` dentro de un formulario codificado.

---

**Siguiente:** [Recomendaciones de seguridad →](12-recomendaciones-seguridad.md)
