# Login y verificación de contraseña

!!! info "En esta página"

    **Qué vas a aprender:** cómo crear la ruta de login que comprueba el email y la contraseña, por qué recibe los datos como formulario, y cómo responder los errores sin revelar qué emails están registrados.

    **Qué necesitás:** [Registro de usuarios](05-registro.md).

## Qué hace el login

El login responde una sola pregunta: **¿esta persona es quien dice ser?** Recibe un email y una contraseña, y:

1. Busca al usuario por email.
2. Verifica la contraseña contra el hash guardado.
3. Si todo está bien, lo deja entrar.

"Dejarlo entrar" significa, en una API, **entregarle un token** para que lo use en los pedidos siguientes. Eso es el tema de la próxima página. Por ahora, si el login es correcto, la ruta devuelve los datos del usuario, para comprobar que la verificación funciona.

## Un formulario en vez de JSON

Hay una convención muy extendida (parte del estándar **OAuth2**) para el login de una API: los datos se mandan como **formulario**, con dos campos que se llaman `username` y `password`. FastAPI trae una clase que los lee, `OAuth2PasswordRequestForm`.

Seguir esta convención tiene una ventaja concreta: la página `/docs` sabe usarla, y en unas páginas vas a poder iniciar sesión desde ahí con un botón **Authorize**. El campo se llama `username` aunque en tu API sea el email: es solo el nombre que fija el estándar.

Para leer formularios, FastAPI usa la librería `python-multipart`, que ya viene con `fastapi[standard]`.

## La ruta de login

Agregala al final de `app/routers/auth.py`, con estos cambios en los imports:

```python title="app/routers/auth.py (imports)"
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy import select

from app.db import SesionDep
from app.esquemas import UsuarioCrear, UsuarioRespuesta
from app.modelos import Usuario
from app.seguridad import hashear_contrasena, verificar_contrasena
```

```python title="app/routers/auth.py (agregar al final)"
@router.post("/login", response_model=UsuarioRespuesta)
async def iniciar_sesion(formulario: Annotated[OAuth2PasswordRequestForm, Depends()], sesion: SesionDep):
    """Comprueba el email y la contraseña."""
    email = formulario.username.lower()
    usuario = await sesion.scalar(select(Usuario).where(Usuario.email == email))
    if usuario is None or not verificar_contrasena(formulario.password, usuario.hash_contrasena):
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Email o contraseña incorrectos")
    if not usuario.activo:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="La cuenta está desactivada")
    return usuario
```

- **`Annotated[OAuth2PasswordRequestForm, Depends()]`**: `OAuth2PasswordRequestForm` es una clase que funciona como dependencia. FastAPI lee el formulario y te entrega un objeto con `.username` y `.password`. El `Depends()` vacío significa "usá la propia clase como dependencia".
- **`.lower()`**: igual que en el registro, para que la persona pueda escribir su email con mayúsculas.
- **`verificar_contrasena`** es la función de la página de hashing: `True` si la contraseña corresponde al hash.
- **Cuenta desactivada**: si la contraseña es correcta pero `activo` es `False`, se responde `403`. La persona está identificada, pero no puede entrar.

## Un solo mensaje para los dos errores

Fijate que el email inexistente y la contraseña incorrecta dan **exactamente la misma** respuesta:

```json
{"detail":"Email o contraseña incorrectos"}
```

Es a propósito. Si respondieras "ese email no existe" en un caso y "contraseña incorrecta" en el otro, cualquiera podría usar el login para averiguar qué emails tienen cuenta en tu app (y después intentar adivinar sus contraseñas, o mandarles correos falsos). Con un mensaje único, el login no revela nada.

## Probarlo

Desde `/docs`, en **POST /auth/login** vas a ver un formulario con `username` y `password` (y algunos campos opcionales del estándar, que podés dejar vacíos). Con `curl`, los datos de formulario se mandan con `-d`, sin el encabezado de JSON:

```bash
curl -X POST http://127.0.0.1:8000/auth/login -d "username=ana@ejemplo.com&password=una-clave-larga"
```

| Datos | Respuesta |
| --- | --- |
| Email y contraseña correctos | `200` con los datos del usuario |
| El email en mayúsculas | `200` (gracias al `.lower()`) |
| Contraseña incorrecta | `401` `Email o contraseña incorrectos` |
| Email que no existe | `401` `Email o contraseña incorrectos` |
| Cuenta con `activo = false` | `403` `La cuenta está desactivada` |

!!! warning "No es JSON"

    Si mandás los datos como JSON (`-H "Content-Type: application/json" -d '{"username": ...}'`), FastAPI responde `422` con `Field required` en `username` y `password`: está esperando un formulario. Si la contraseña tiene símbolos como `&` o `=`, usá `--data-urlencode` en `curl` (uno por campo), que los codifica bien.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Express, el login suele recibir JSON (`req.body.email`, `req.body.password`), aunque con `express.urlencoded()` también puede leer formularios. Librerías como Passport con la estrategia `local` hacen estos mismos pasos: buscar el usuario, verificar la contraseña y responder un error genérico si algo falla.

??? example "Ver comparación en Django ⬇️"

    Django resuelve esto con `authenticate(request, username=..., password=...)`, que busca al usuario, verifica el hash y devuelve `None` si algo falla, sin distinguir el motivo. La vista `LoginView` responde también con un mensaje genérico. En DRF, la vista `obtain_auth_token` recibe `username` y `password`, como `OAuth2PasswordRequestForm`.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring Security, un `AuthenticationManager` con un `DaoAuthenticationProvider` busca el usuario con tu `UserDetailsService` y verifica la contraseña con el `PasswordEncoder`. Si falla, lanza `BadCredentialsException`, que por defecto no distingue entre usuario inexistente y contraseña incorrecta. El `formLogin()` clásico también usa los campos `username` y `password`.

## Mini ejercicio

El mensaje único no alcanza del todo: si el email no existe, la ruta **no calcula ningún hash** (el `or` corta antes), así que responde mucho más rápido. Midiendo el tiempo de respuesta, se puede saber si un email existe:

```text
Email que existe:    32.6 ms
Email que no existe: 4.6 ms
```

Hacé que el login tarde lo mismo en los dos casos. Pista: si el usuario no existe, verificá la contraseña igual, contra un hash falso.

??? success "Ver solución"

    Debajo de `router = APIRouter(...)`, creá un hash falso una sola vez:

    ```python
    HASH_FALSO = hashear_contrasena("contraseña-que-nadie-usa")
    ```

    Y en `iniciar_sesion`, verificá **siempre**:

    ```python
    usuario = await sesion.scalar(select(Usuario).where(Usuario.email == email))
    hash_a_comparar = usuario.hash_contrasena if usuario is not None else HASH_FALSO
    contrasena_ok = verificar_contrasena(formulario.password, hash_a_comparar)
    if usuario is None or not contrasena_ok:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Email o contraseña incorrectos")
    ```

    Ahora los dos casos hacen el trabajo de argon2, y la diferencia desaparece:

    ```text
    Email que existe:    32.2 ms
    Email que no existe: 32.4 ms
    ```

    Esta forma de averiguar información midiendo tiempos se llama **ataque de tiempo** (*timing attack*).

---

**Siguiente:** [Qué es un JWT y cómo se usa →](07-jwt.md)
