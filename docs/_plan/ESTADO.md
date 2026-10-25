# Estado de la guía (para retomar el trabajo)

Excluido del sitio publicado, igual que `PLAN.md`.

## Avance

- Capítulos 0 a 6 y los 4 apéndices: completos (80 páginas; todas las casillas de contenido tildadas en `PLAN.md`).
- La última página (`apendices/04-como-esta-hecha.md`) termina con "Volver al inicio". La cadena de "Siguiente" se revisó entera (80 páginas, coincide con el `nav:`).
- Repo en GitHub: `git@github.com:villalb4/python-MkDocs.git` (`site_url: https://villalb4.github.io/python-MkDocs/`).
- Falta: activar Pages en el repo (Settings → Pages → Source: GitHub Actions).
- El build del workflow se simuló en un entorno limpio (`pip install -r requirements.txt` + `mkdocs build --strict`): anda con MkDocs 1.6.1 y Material 9.8.0. `requirements.txt` no fija versión.

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

## Cambios al código de la API en el Capítulo 6

Sobre el estado del Capítulo 5 (descripto arriba):

- `pyproject.toml`: grupo `dev` con `pytest` y `ruff`; `[tool.pytest]` con `pythonpath = ["."]` y `testpaths = ["tests"]`; `[tool.ruff]` con `line-length = 120` y `extend-exclude = ["migraciones"]`.
- `tests/`: `test_seguridad.py` (3), `test_tokens.py` (4), `conftest.py` (fixture `cliente`: cambia `URL_BASE_DATOS` a la base `tareas_test` antes de importar la app, recrea las tablas con `create_all` + `motor.dispose()`, vacía `intentos_por_ip`, `with TestClient(app)`), `test_endpoints.py` (4), `test_tareas.py` (helper `iniciar_sesion` + 2), `test_logs.py` (`caplog`, 1). Total: `14 passed`.
- `app/seguridad.py`: `datetime.UTC` en vez de `timezone.utc` (arreglo de Ruff, UP017).
- `app/routers/tareas.py`: el decorador de `borrar_tarea` partido en tres renglones (`ruff format`).
- Logs: `basicConfig` en `app/main.py` con `nivel_log`; el SQL se activa con `logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)` si `debug` (el motor ya no tiene `echo`); loggers en `main.py` (arranque/apagado), `routers/auth.py` (registro, login fallido con email, cuenta desactivada, login correcto) y `dependencias.py` (429).
- `app/config.py`: sin `clave_api`; `url_base_datos: SecretStr`, `clave_jwt: SecretStr = Field(min_length=32)`, `nivel_log` (`str` en la página; `Literal[...]` en la solución del ejercicio). `.get_secret_value()` en `db.py`, `migraciones/env.py` y `seguridad.py`.
- `docker-compose.prod.yml` (db y api sin `ports`, `restart: unless-stopped`, `command` con `alembic upgrade head` y `--forwarded-allow-ips='*'`, servicio `proxy` con `caddy:2`) y `Caddyfile` (`localhost { reverse_proxy api:8000 }`).

Versiones nuevas: pytest 9.1.1, Ruff 0.17.2 (sus reglas por defecto incluyen `UP`, `I`, `B`, `RUF`, `LOG`, etc.), httpx2 2.16 (viene con `fastapi[standard]`), Caddy 2.11.

## Notas para los apéndices

- `docker run --env-file .env` no saca las comillas: `NOMBRE_APP="Tareas (desarrollo)"` llega con comillas (Compose sí las saca). Afecta al ejemplo de `docker/07-dockerfile.md`; va en "Errores comunes por capítulo".
- `docker rm` sin `-v` deja el volumen anónimo de PostgreSQL (la chuleta recomienda `docker rm -v`). `alembic revision --autogenerate` sin cambios genera una migración vacía.
- `DEBUG=false docker compose up` no cambia el `env_file` (solo la interpolación del YAML); `environment:` sí lo pisa.
- Material for MkDocs deja de tener soporte el 5 de mayo de 2027 (sucesor: Zensical): mencionarlo en "Cómo está hecha esta guía". `mkdocs build` muestra un aviso sobre MkDocs 2 (se calla con `NO_MKDOCS_2_WARNING=1`); no es un warning del build.
- Errores reales del Capítulo 6 que pueden ir a "Errores comunes": `ModuleNotFoundError: No module named 'app'` (falta `pythonpath`), `attached to a different loop` (falta `dispose()`), `KeyError: 'access_token'` (límite de intentos en los tests), `database "tareas_test" does not exist`, el SQL duplicado con `echo` + `basicConfig`, `ValueError: Unknown level: 'info'`, todos detrás del proxy con la misma IP.

## Notas del Capítulo 5 ya resueltas en el Capítulo 6

- El aviso de `httpx` con `TestClient`: no aparece con `fastapi[standard]`, que trae `httpx2` (explicado en `calidad-y-despliegue/02`).
- El traceback con `debug=True` en los `500`: en `calidad-y-despliegue/05`.
- Los `print` del `lifespan` reemplazados por logs: en `calidad-y-despliegue/04`.

## Largo de las páginas

Páginas que pasan un poco de 200 renglones: python/03, 07, 08, 16, 18, 19, 21; fastapi/08, 10 (225); base-de-datos/05, 06, 07, 10; calidad-y-despliegue/01 (204).
