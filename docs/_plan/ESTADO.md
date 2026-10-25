# Estado de la guía (para retomar el trabajo)

Excluido del sitio publicado, igual que `PLAN.md`.

## Avance

- Capítulos 0 a 5: completos (70 casillas tildadas en `PLAN.md`).
- Falta: Capítulo 6 (6 páginas), Apéndices (4) y los dos pasos de GitHub (crear el repo, activar Pages).
- La última página escrita (`autenticacion/13-errores-comunes.md`) termina con
  `**Siguiente:** Capítulo 6 · Tests con pytest: primer test (en preparación).` → convertirlo en enlace al crear `calidad-y-despliegue/01-...md`.

## Cómo se trabajó cada página

1. Probar todo el código de verdad antes de escribir (en un proyecto de prueba aparte).
2. Escribir la página: bloque "En esta página", explicación, 3 desplegables de comparación (Express, Django, Spring Boot), mini ejercicio con solución, "Siguiente".
3. Comparar los fragmentos de la página con los archivos probados (diff).
4. Enlazar desde la página anterior, agregar al `nav:` de `mkdocs.yml`, `mkdocs build --strict` sin warnings, tildar en `PLAN.md`.
5. Al cerrar un capítulo: revisar cadena de enlaces, `nav:`, estructura y largo (≤ ~200 renglones).

Convenciones que surgieron:

- Resultados cortos como comentario en el código (`print(x)  # 13`).
- Sin `++teclas++` ni `:emoji:` (las extensiones no están): usar `<kbd>`.
- En tablas Markdown, `|` dentro de código se escribe sin escapar (`str | None`); con `\|` aparece la barra en el HTML.
- Mensajes de `alembic revision -m` sin tildes ni ñ (van al nombre del archivo).
- El puerto 5432 de la máquina está ocupado (proyecto `pricescope` del usuario, no tocar): las pruebas usan 5433 (`PUERTO_DB=5433`). Las páginas muestran 5432 y explican cómo cambiarlo.

## Estado del código de la API al final del Capítulo 5

Proyecto `tareas-api` (`uv init --no-package`), paquete `app/`:

- `app/config.py`: `Configuracion(BaseSettings)` con `env_file=BASE_DIR / ".env"`, `extra="ignore"`. Campos: `nombre_app`, `debug`, `clave_api` (ya no se usa en rutas), `origenes_permitidos: list[str]`, `url_base_datos`, `clave_jwt`, `algoritmo_jwt="HS256"`, `minutos_token=15`, `dias_refresco=7`. `obtener_configuracion()` con `@cache`.
- `app/db.py`: `motor` (`create_async_engine`, `echo=debug`), `CrearSesion` (`async_sessionmaker`, `expire_on_commit=False`), `Base`, `obtener_sesion` (con `yield`), `SesionDep`.
- `app/modelos.py`: `Usuario` (id, nombre, email unique, hash_contrasena, rol default "usuario", activo, creado_en, `tareas` relationship) y `Tarea` (id, titulo String(100), descripcion Text nullable, hecha, creada_en, usuario_id FK, vence_en nullable, `usuario` relationship).
- `app/esquemas.py`: `TareaCrear` (sin usuario_id), `TareaRespuesta` (from_attributes, con usuario_id), `UsuarioCrear` (EmailStr, contrasena 8–128), `UsuarioRespuesta` (sin hash), `UsuarioConTareas`, `Token` (access_token, token_type).
- `app/seguridad.py`: pwdlib argon2 (`hashear_contrasena`, `verificar_contrasena`), PyJWT (`crear_token(usuario_id, tipo, duracion)`, `crear_token_acceso`, `crear_token_refresco`, `leer_token`).
- `app/dependencias.py`: `OAuth2PasswordBearer(tokenUrl="/auth/login")`, `obtener_usuario_actual` (exige tipo "acceso", usuario activo), `UsuarioActualDep`, `requerir_admin` / `AdminDep`, `limitar_intentos` (5 por minuto por IP, 429 con Retry-After).
- `app/routers/auth.py`: `POST /auth/registro` (email en minúsculas, 409), `POST /auth/login` (formulario OAuth2, limitado, devuelve access y guarda el refresh en cookie httpOnly `path=/auth`, `samesite=strict`, `secure`), `POST /auth/refrescar` (lee la cookie, rota), `POST /auth/logout` (borra la cookie), `GET /auth/yo`.
- `app/routers/tareas.py`: todo filtrado por el usuario actual; tarea ajena → 404; paginación con dependencia.
- `app/routers/usuarios.py`: protegido con `requerir_admin`; listar, ver con tareas (`selectinload`), tareas de un usuario.
- `app/main.py`: `lifespan` solo hace `motor.dispose()`; CORS con `allow_credentials=True`; ruta `/salud` (`SELECT 1`).
- Alembic (`migraciones/`, plantilla async, URL desde la configuración): migraciones "crear tablas", "agregar vence_en", "agregar autenticacion a usuarios".
- `Dockerfile` (python:3.12-slim + uv 0.13.1, `fastapi run`), `.dockerignore`, `docker-compose.yml` (db con healthcheck y volumen, api con `build: .`, `URL_BASE_DATOS` apuntando a `db`).

Versiones probadas: Python 3.12, FastAPI 0.162, SQLAlchemy 2.1.6, asyncpg 0.32, Alembic 1.20.1, pydantic-settings 2.16, pwdlib 0.3.1, PyJWT 2.15.1, PostgreSQL 17, Docker 29.8, Compose v5.6, uv 0.13.1.

## Notas para el Capítulo 6

- Starlette avisa que usar `httpx` con `TestClient` está deprecado y sugiere `httpx2`: revisarlo en "Tests de endpoints con `httpx`".
- `debug=True` devuelve el traceback en los `500` (visto en el Capítulo 3): retomarlo en "Preparar la app para producción".
- Los `print` del `lifespan` se reemplazan por logs en "Logs: qué registrar y qué no".
- Material for MkDocs avisa que deja de tener soporte el 5 de mayo de 2027 (sucesor: Zensical); mencionarlo en el apéndice "Cómo está hecha esta guía".
- Páginas que pasan un poco de 200 renglones: python/03, 07, 08, 16, 18, 19, 21; fastapi/08, 10 (225); base-de-datos/05, 06, 07, 10.
