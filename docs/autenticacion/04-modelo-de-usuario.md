# Modelo de usuario

!!! info "En esta página"

    **Qué vas a aprender:** cómo preparar el modelo `Usuario` para la autenticación (hash de la contraseña, rol y fecha de alta), cómo migrar la base, y cómo separar los esquemas para que el hash **nunca** salga en una respuesta.

    **Qué necesitás:** [Hashing de contraseñas (argon2 o bcrypt)](03-hashing-argon2.md) y [Primera migración](../base-de-datos/10-primera-migracion.md). La base tiene que estar corriendo (`docker compose up -d db`).

## Qué le falta al usuario

Tu modelo `Usuario` tiene nombre, email y si está activo. Para que pueda iniciar sesión le faltan:

- **El hash de la contraseña.** Nunca la contraseña: el resultado de `hashear_contrasena`.
- **Un rol**, para separar usuarios comunes de administradores (lo vas a usar en la página de roles).
- **La fecha de alta**, que siempre viene bien tener.

Además, el email pasa a ser el dato con el que la persona se identifica en el login, así que tiene que ser único. Eso ya lo tenías con `unique=True`.

## El modelo

En `app/modelos.py`, agregá tres columnas a `Usuario`:

```python title="app/modelos.py"
class Usuario(Base):
    __tablename__ = "usuarios"

    id: Mapped[int] = mapped_column(primary_key=True)
    nombre: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True)
    hash_contrasena: Mapped[str] = mapped_column(String(255))
    rol: Mapped[str] = mapped_column(String(20), default="usuario", server_default="usuario")
    activo: Mapped[bool] = mapped_column(default=True)
    creado_en: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

    tareas: Mapped[list["Tarea"]] = relationship(back_populates="usuario")
```

- **`hash_contrasena`**: el nombre deja claro qué se guarda. Un hash de argon2 ocupa unos 100 caracteres; `String(255)` deja margen.
- **`rol`**: un texto, por defecto `"usuario"`. Lleva `server_default` (además de `default`) por lo que viste en las migraciones: así la base también sabe el valor por defecto.
- **`creado_en`**: igual que en `Tarea`, lo completa la base con `now()`.

## La migración

```bash
uv run alembic revision --autogenerate -m "agregar autenticacion a usuarios"
```

Alembic detecta las tres columnas nuevas. Revisá la migración: tiene que tener tres `op.add_column` en `upgrade()` y tres `op.drop_column` en `downgrade()`.

!!! tip "Sin tildes ni eñes en el mensaje"

    El texto de `-m` también se usa para el **nombre del archivo** de la migración. Conviene escribirlo sin tildes ni `ñ`, para evitar problemas con algunos sistemas y herramientas.

Antes de aplicarla, un problema: si ya tenés usuarios en la base, la migración falla.

```text
NotNullViolationError: column "hash_contrasena" of relation "usuarios" contains null values
```

Los usuarios que creaste antes no tienen contraseña, y la columna es obligatoria. No hay un valor por defecto razonable para un hash, así que, como son datos de prueba, lo más simple es borrarlos (primero las tareas, por la clave foránea):

```bash
docker compose exec db psql -U tareas -c "DELETE FROM tareas; DELETE FROM usuarios;"
uv run alembic upgrade head
```

En una base real, con usuarios de verdad, se haría en pasos: agregar la columna como opcional, pedirle a cada usuario que defina su contraseña, y recién después volverla obligatoria con otra migración.

## Los esquemas: lo que entra y lo que sale

La contraseña tiene que **entrar** en el registro, pero el hash **nunca** tiene que **salir** en una respuesta. Cada esquema de Pydantic se encarga de una dirección. En `app/esquemas.py`, agregá `EmailStr` al import de `pydantic` y actualizá los dos esquemas de usuario:

```python title="app/esquemas.py"
class UsuarioCrear(BaseModel):
    nombre: str = Field(min_length=2, max_length=100)
    email: EmailStr
    contrasena: str = Field(min_length=8, max_length=128)


class UsuarioRespuesta(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    nombre: str
    email: str
    rol: str
    activo: bool
    creado_en: datetime
```

- **`EmailStr`** valida que el email tenga un formato correcto. Usa la librería `email-validator`, que ya viene con `fastapi[standard]`.
- **`contrasena`** pide al menos 8 caracteres. El máximo evita que alguien mande un texto gigante para que argon2 trabaje de más.
- **`UsuarioRespuesta` no tiene ni `contrasena` ni `hash_contrasena`.** Como el modelo de respuesta filtra los campos, aunque el objeto `Usuario` tenga el hash, ese campo no llega a la respuesta. `UsuarioConTareas` hereda de `UsuarioRespuesta`, así que también queda protegido.

Por último, en `app/routers/usuarios.py`, `crear_usuario` tiene que guardar el hash. Importá `hashear_contrasena` de `app.seguridad` y creá el usuario así:

```python
    usuario = Usuario(
        nombre=datos.nombre,
        email=datos.email,
        hash_contrasena=hashear_contrasena(datos.contrasena),
    )
```

## Probarlo

Creá un usuario desde `/docs` (con el encabezado `X-Clave`, que por ahora sigue protegiendo el router):

```json
{"nombre": "Ana", "email": "ana@ejemplo.com", "contrasena": "una-clave-larga"}
```

La respuesta trae el rol y la fecha, pero ni rastro de la contraseña:

```json
{"id":2,"nombre":"Ana","email":"ana@ejemplo.com","rol":"usuario","activo":true,"creado_en":"2026-10-25T04:11:44.140062Z"}
```

En la base, en cambio, está el hash:

```bash
docker compose exec db psql -U tareas -c "SELECT email, rol, hash_contrasena FROM usuarios;"
```

Y si mandás un email sin `@` y una contraseña corta, la respuesta es un `422` con los dos errores: `value is not a valid email address: An email address must have an @-sign.` y `String should have at least 8 characters`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Con Prisma, el modelo tendría `hashContrasena String` y `rol String @default("usuario")`. Para no devolver el hash, en Node hay que acordarse de excluirlo a mano en cada consulta (`select` o `omit`). En FastAPI, el modelo de respuesta lo filtra siempre, aunque te olvides.

??? example "Ver comparación en Django ⬇️"

    Django ya trae un modelo de usuario (`django.contrib.auth.models.User`) con `password` (que guarda el hash), `is_staff`, `is_superuser` y `date_joined`. En FastAPI armás el tuyo con los campos que necesitás. Lo que en DRF hacés con `extra_kwargs = {"password": {"write_only": True}}`, acá lo resuelven dos esquemas separados.

??? example "Ver comparación en Spring Boot ⬇️"

    Es una `@Entity Usuario` con un campo `password` que guarda el hash del `PasswordEncoder`, más un DTO de respuesta sin ese campo. Si se devolviera la entidad directamente, Jackson incluiría la contraseña salvo que la marques con `@JsonIgnore`; con un DTO de salida separado (lo que hace `UsuarioRespuesta`), el problema no existe.

## Mini ejercicio

Creá un segundo usuario con **la misma contraseña** que Ana. Después mirá en `psql` los dos hashes. ¿Son iguales? ¿Qué parte cambia?

??? success "Ver solución"

    ```json
    {"nombre": "Juan", "email": "juan@ejemplo.com", "contrasena": "una-clave-larga"}
    ```

    ```text
          email       |                  hash_contrasena
    ------------------+---------------------------------------------------------------
     ana@ejemplo.com  | $argon2id$v=19$m=65536,t=3,p=4$YzNqa27HhxJO7MWM5IIzrQ$lJBW/izO...
     juan@ejemplo.com | $argon2id$v=19$m=65536,t=3,p=4$+UyT4ELQX/irgYlIoixarA$xj0tWVOp...
    ```

    Son distintos, aunque la contraseña sea la misma. El principio (`$argon2id$v=19$m=65536,t=3,p=4$`) es igual porque es el algoritmo con sus parámetros; cambian la **sal** y, por lo tanto, el **hash**. Nadie que mire la base puede saber que comparten contraseña.

---

**Siguiente:** [Registro de usuarios →](05-registro.md)
