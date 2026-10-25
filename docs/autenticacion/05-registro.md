# Registro de usuarios

!!! info "En esta página"

    **Qué vas a aprender:** cómo crear una ruta pública de registro en un router propio de autenticación, que guarde la contraseña hasheada, evite emails repetidos y no deje que nadie se registre como administrador.

    **Qué necesitás:** [Modelo de usuario](04-modelo-de-usuario.md).

## Un router para la autenticación

Hasta ahora, los usuarios se creaban con `POST /usuarios`, dentro de un router protegido por la `X-Clave`. Pero registrarse tiene que ser **público**: alguien que todavía no tiene cuenta no puede tener ninguna credencial. Y en la próxima página vas a agregar el login, que también es público.

Por eso, todo lo relacionado con la autenticación va en un router propio, con el prefijo `/auth`. Creá `app/routers/auth.py`:

```python title="app/routers/auth.py"
from fastapi import APIRouter, HTTPException, status
from sqlalchemy import select

from app.db import SesionDep
from app.esquemas import UsuarioCrear, UsuarioRespuesta
from app.modelos import Usuario
from app.seguridad import hashear_contrasena

router = APIRouter(prefix="/auth", tags=["auth"])


@router.post("/registro", response_model=UsuarioRespuesta, status_code=status.HTTP_201_CREATED)
async def registrar(datos: UsuarioCrear, sesion: SesionDep):
    """Crea una cuenta nueva con email y contraseña."""
    email = datos.email.lower()
    existente = await sesion.scalar(select(Usuario).where(Usuario.email == email))
    if existente is not None:
        raise HTTPException(status_code=status.HTTP_409_CONFLICT, detail="Ya existe una cuenta con ese email")

    usuario = Usuario(
        nombre=datos.nombre,
        email=email,
        hash_contrasena=hashear_contrasena(datos.contrasena),
    )
    sesion.add(usuario)
    await sesion.commit()
    await sesion.refresh(usuario)
    return usuario
```

Es casi el `crear_usuario` que ya tenías, con una mejora: el email se pasa a minúsculas antes de buscarlo y de guardarlo.

### Por qué pasar el email a minúsculas

Para la gente, `Ana@Ejemplo.com` y `ana@ejemplo.com` son el mismo email. Para la base, son dos textos distintos, y la restricción `UNIQUE` dejaría crear las dos cuentas. `EmailStr` valida el formato y pasa a minúsculas el **dominio** (lo que va después de la `@`), pero **no** la parte de antes. Por eso lo hacemos nosotros con `.lower()`. Lo mismo vas a hacer en el login, para que la persona pueda escribir su email como quiera.

## Sacar la creación del router de usuarios

Ahora que el registro vive en `/auth`, borrá la ruta `POST /usuarios` (la función `crear_usuario`) de `app/routers/usuarios.py`, y sacá de ese archivo los imports que ya no se usan (`UsuarioCrear` y `hashear_contrasena`). El router de usuarios queda solo para **consultar** usuarios. Todavía está protegido por la `X-Clave`; en las próximas páginas la vas a reemplazar por la autenticación de verdad.

Por último, incluí el router nuevo en `app/main.py`:

```python title="app/main.py (cambios)"
from app.routers import auth, tareas, usuarios

# ...

app.include_router(auth.router)
app.include_router(tareas.router)
app.include_router(usuarios.router)
```

## Probarlo

En `/docs` aparece un grupo nuevo, **auth**, con `POST /auth/registro`. Fijate que no pide ningún encabezado. Probá estos casos:

| Cuerpo | Respuesta |
| --- | --- |
| `{"nombre": "Ana", "email": "ana@ejemplo.com", "contrasena": "una-clave-larga"}` | `201` con el usuario creado |
| El mismo, con `"email": "ANA@Ejemplo.com"` | `409` `Ya existe una cuenta con ese email` |
| Un email sin `@` o una contraseña de menos de 8 caracteres | `422` con los errores |

Y `POST /usuarios` ahora responde `405 Method Not Allowed`: esa ruta ya no existe.

## Nadie se registra como administrador

Probá registrarte mandando un campo de más:

```json
{"nombre": "Pícaro", "email": "picaro@ejemplo.com", "contrasena": "una-clave-larga", "rol": "admin"}
```

```json
{"id":5,"nombre":"Pícaro","email":"picaro@ejemplo.com","rol":"usuario","activo":true,"creado_en":"..."}
```

El usuario se crea con rol `usuario`. `UsuarioCrear` no tiene un campo `rol`, así que Pydantic **ignora** ese dato: nunca llega a tu función. Y como el usuario se arma campo por campo, el rol queda con su valor por defecto.

Esto evita un error de seguridad clásico, la **asignación masiva** (*mass assignment*): copiar todo lo que manda el cliente directamente al modelo, incluidos campos que no debería poder tocar, como el rol o si la cuenta está activa. La regla: el esquema de entrada tiene **solo** los campos que el usuario puede decidir.

!!! note "No le cuentes a nadie qué emails existen"

    El `409` le avisa a cualquiera si un email ya tiene cuenta. En el registro es difícil evitarlo (la persona necesita saber por qué no puede registrarse), pero en el **login** vas a tener cuidado de responder lo mismo si el email no existe o si la contraseña es incorrecta, para no revelar qué emails están registrados.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Una ruta `POST /auth/registro` en Express hace los mismos pasos: validar el cuerpo (con Zod), buscar el email, hashear con `argon2.hash` y guardar. El riesgo de asignación masiva es frecuente en Node cuando se hace `prisma.usuario.create({ data: req.body })`: con un esquema de entrada explícito, como acá, no pasa.

??? example "Ver comparación en Django ⬇️"

    Django trae formularios de registro (`UserCreationForm`), y en DRF se arma un serializer con `create()` que llama a `User.objects.create_user()`, que hashea la contraseña. El serializer declara qué campos acepta, y eso protege de la asignación masiva igual que `UsuarioCrear`. Django también normaliza el dominio del email con `normalize_email()`.

??? example "Ver comparación en Spring Boot ⬇️"

    Sería un `@PostMapping("/auth/registro")` en un controller público, permitido en la configuración de Spring Security con `.requestMatchers("/auth/**").permitAll()`. El DTO de entrada sin campo `rol` cumple el mismo rol que `UsuarioCrear`.

## Mini ejercicio

El chequeo del email tiene un hueco: si dos personas se registran con el mismo email **exactamente al mismo tiempo**, las dos pueden pasar el `if` antes de que la otra guarde, y una termina en un `IntegrityError` (un `500`). Atrapá ese error en el `commit` y respondé `409` también en ese caso.

??? success "Ver solución"

    Importá `IntegrityError` de `sqlalchemy.exc` y envolvé el `commit`:

    ```python
    sesion.add(usuario)
    try:
        await sesion.commit()
    except IntegrityError:
        await sesion.rollback()
        raise HTTPException(status_code=status.HTTP_409_CONFLICT, detail="Ya existe una cuenta con ese email")
    await sesion.refresh(usuario)
    return usuario
    ```

    Si el `INSERT` choca con la restricción `UNIQUE`, `rollback()` deshace la operación fallida y se responde `409`. El chequeo previo sigue siendo útil, porque cubre el caso normal con un mensaje claro y sin intentar el `INSERT`; el `try` cubre la carrera. Es la combinación de las dos capas que viste en el Capítulo 3: la validación en la API y la restricción en la base.

---

**Siguiente:** [Login y verificación de contraseña →](06-login.md)
