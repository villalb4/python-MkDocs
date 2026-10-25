# Qué es un JWT y cómo se usa

!!! info "En esta página"

    **Qué vas a aprender:** qué es un JWT, cómo está armado, por qué nadie lo puede falsificar sin la clave, y cómo hacer que el login devuelva uno usando la librería PyJWT.

    **Qué necesitás:** [Login y verificación de contraseña](06-login.md) y [Configuración con `pydantic-settings`](../fastapi/12-pydantic-settings.md).

## El problema: recordar quién es quién

El login ya comprueba la contraseña. Pero HTTP no tiene memoria: en el próximo pedido, la API no sabe que Ana se identificó hace un minuto. Hay que darle a Ana algo que pueda mostrar en cada pedido, y que la API pueda comprobar **sin volver a pedirle la contraseña**.

Ese algo es un **token**. Y el formato de token más usado en las APIs es el **JWT** (*JSON Web Token*, que se pronuncia "jot").

## Cómo está armado un JWT

Un JWT es un texto con tres partes separadas por puntos:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0IiwiZXhwIjoxNzkyOTA0NDAwfQ.MqfR_z_K7P9Uh8gwtXntyytSs6iCz7Z_igu1r8xuhhM
──────────────┬───────────────────── ──────────────┬─────────────────────── ─────────────────────┬─────────────────────
           cabecera                              datos                                        firma
```

- **La cabecera** dice cómo está firmado: `{"alg":"HS256","typ":"JWT"}`.
- **Los datos** (*payload*) son un JSON con lo que el servidor quiere recordar: `{"sub":"4","exp":1792904400}`.
- **La firma** es un código calculado a partir de la cabecera, los datos y una **clave secreta** que solo conoce el servidor.

Las dos primeras partes están en *Base64*, una forma de escribir datos con letras y números. **No están encriptadas**: cualquiera que tenga el token puede leer los datos. Lo que no puede hacer es **cambiarlos**.

### La firma: por qué no se puede falsificar

Si alguien toma el token de Ana y cambia `"sub":"4"` por `"sub":"1"` (para hacerse pasar por otro usuario), la firma ya no coincide con los datos. Para calcular una firma nueva necesitaría la clave secreta, que nunca sale del servidor. Al verificar, la API lo rechaza:

```text
InvalidSignatureError: Signature verification failed
```

Es como un sobre lacrado con un sello único: cualquiera puede ver lo que dice por fuera, pero si lo abren y lo vuelven a cerrar, el sello se nota.

### Los datos estándar (*claims*)

| Campo | Significa |
| --- | --- |
| `sub` | *Subject*: de quién es el token. Acá, el `id` del usuario, **como texto**. |
| `exp` | *Expiration*: cuándo vence (en segundos desde 1970). Después de esa fecha, el token no sirve. |
| `iat` | *Issued at*: cuándo se creó (opcional). |

!!! warning "Nunca pongas secretos en un JWT"

    Como cualquiera puede leer los datos, en un JWT no van contraseñas, ni datos privados. Alcanza con el `id` del usuario: con él, la API busca el resto en la base.

## La configuración

Instalá PyJWT:

```bash
uv add pyjwt
```

La clave para firmar es un secreto: va en el `.env`. Generá una clave aleatoria larga con:

```bash
uv run python -c "import secrets; print(secrets.token_hex(32))"
```

y agregala al `.env`:

```bash title=".env (agregar)"
CLAVE_JWT=pegá-acá-los-64-caracteres-que-generaste
```

En `Configuracion` (`app/config.py`), agregá los campos:

```python
clave_jwt: str
algoritmo_jwt: str = "HS256"
minutos_token: int = 30
```

- `clave_jwt` no tiene valor por defecto: sin clave, la app no arranca.
- `HS256` es el algoritmo de firma más común: usa una sola clave secreta para firmar y verificar.
- El token va a vencer a los 30 minutos.

!!! tip "La clave tiene que ser larga"

    Con una clave corta, PyJWT avisa: `InsecureKeyLengthWarning: The HMAC key is 5 bytes long, which is below the minimum recommended length of 32 bytes`. `token_hex(32)` genera exactamente 32 bytes aleatorios. Y si la clave se filtra, cualquiera puede fabricar tokens válidos: cuidala como la contraseña de la base.

## Crear y leer tokens

En `app/seguridad.py`, agregá dos funciones (y los imports):

```python title="app/seguridad.py"
from datetime import datetime, timedelta, timezone

import jwt
from pwdlib import PasswordHash

from app.config import obtener_configuracion

hasheador = PasswordHash.recommended()
configuracion = obtener_configuracion()


def hashear_contrasena(contrasena: str) -> str:
    return hasheador.hash(contrasena)


def verificar_contrasena(contrasena: str, hash_guardado: str) -> bool:
    return hasheador.verify(contrasena, hash_guardado)


def crear_token(usuario_id: int) -> str:
    vence = datetime.now(timezone.utc) + timedelta(minutes=configuracion.minutos_token)
    datos = {"sub": str(usuario_id), "exp": vence}
    return jwt.encode(datos, configuracion.clave_jwt, algorithm=configuracion.algoritmo_jwt)


def leer_token(token: str) -> dict:
    return jwt.decode(token, configuracion.clave_jwt, algorithms=[configuracion.algoritmo_jwt])
```

- **`crear_token`** arma los datos, con el vencimiento a 30 minutos desde ahora, y los firma con `jwt.encode`. `timezone.utc` hace que la hora no dependa de la zona horaria del servidor.
- **`str(usuario_id)`**: PyJWT exige que `sub` sea texto. Con un número, al leerlo falla con `InvalidSubjectError: Subject must be a string`.
- **`leer_token`** verifica la firma y el vencimiento con `jwt.decode`. Si todo está bien, devuelve los datos; si no, lanza una excepción: `InvalidSignatureError` si fue alterado, `ExpiredSignatureError` (`Signature has expired`) si venció. La vas a usar en la próxima página.
- **`algorithms=[...]`** es una lista de los algoritmos aceptados. Indicarla explícitamente evita que un token armado con otro algoritmo pase la verificación.

## El login devuelve un token

El estándar OAuth2 fija la forma de la respuesta del login: un `access_token` y su tipo, `bearer`. Agregá el esquema en `app/esquemas.py`:

```python title="app/esquemas.py (agregar)"
class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"
```

Y en `app/routers/auth.py`, importá `Token` y `crear_token`, y cambiá el login para que devuelva el token:

```python
@router.post("/login", response_model=Token)
async def iniciar_sesion(formulario: Annotated[OAuth2PasswordRequestForm, Depends()], sesion: SesionDep):
    """Comprueba el email y la contraseña, y devuelve un token de acceso."""
    # ... (la búsqueda y las verificaciones quedan igual)
    return Token(access_token=crear_token(usuario.id))
```

Ahora el login responde:

```json
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0IiwiZXhwIjoxNzkyOTAzNjc0fQ.s3YlmvThOdEVje7Wix6GsZjYsprsUq_TThnqblMl2uc","token_type":"bearer"}
```

El cliente guarda ese `access_token` y lo manda en cada pedido con `Authorization: Bearer ...`. Las rutas todavía no lo exigen: eso es la próxima página.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Node se usa la librería `jsonwebtoken`, casi idéntica:

    ```javascript
    const token = jwt.sign({ sub: "4" }, process.env.CLAVE_JWT, { expiresIn: "30m" })
    const datos = jwt.verify(token, process.env.CLAVE_JWT)
    ```

    El JWT generado es el mismo formato: un token firmado en Node se puede verificar en Python con la misma clave, y al revés.

??? example "Ver comparación en Django ⬇️"

    Django usa sesiones guardadas en la base, con una cookie. Para APIs con JWT se suma **djangorestframework-simplejwt**, que trae la ruta de login (`TokenObtainPairView`) y devuelve `access` y `refresh`. Por debajo hace lo mismo que `crear_token`: firmar los datos con una clave.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring se usa una librería como `jjwt` o el soporte de *OAuth2 Resource Server* de Spring Security, con `JwtEncoder` y `JwtDecoder`. La clave se configura en `application.properties`, y la respuesta del login suele tener la misma forma OAuth2: `access_token` y `token_type`.

## Mini ejercicio

Hacé login, copiá el `access_token` y leé sus datos **sin la clave**, con `jwt.decode(token, options={"verify_signature": False})`. ¿Qué podés ver? ¿Qué te dice eso sobre lo que se puede guardar en un token?

??? success "Ver solución"

    ```python
    import jwt

    token = "pegá-acá-tu-token"
    print(jwt.decode(token, options={"verify_signature": False}))
    ```

    ```text
    {'sub': '4', 'exp': 1792903674}
    ```

    Cualquiera puede leer el `id` del usuario y cuándo vence el token, sin conocer la clave. Por eso en un JWT solo va información que no importa que se vea. Lo que la clave protege es que nadie pueda **modificar** esos datos ni fabricar un token nuevo. Nunca uses `verify_signature: False` en la API: sirve solo para mirar un token mientras desarrollás.

---

**Siguiente:** [Proteger rutas con `Depends` →](08-proteger-rutas.md)
