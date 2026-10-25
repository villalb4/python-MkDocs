# Plan: Guía de Python + FastAPI para principiantes

Va en el repo como `docs/_plan/PLAN.md`. Las casillas se tildan a medida que se termina cada página.

## Capítulo 0 · Introducción
- [x] Qué vas a aprender y en qué orden
- [x] Qué necesitás instalar (editor, terminal, Git, Docker)
- [x] Cómo leer las comparaciones (desplegables y pestañas)

## Capítulo 1 · Python básico
- [x] Qué es Python y cómo se ejecuta un script
- [x] Instalar y usar `uv` (sin Python manual)
- [x] Variables y tipos básicos (`int`, `float`, `str`, `bool`, `None`)
- [x] Strings y f-strings
- [x] Listas
- [x] Diccionarios
- [x] Condicionales (`if`, `elif`, `else`)
- [x] Bucles (`for`, `while`)
- [x] Funciones
- [x] Parámetros, valores por defecto y argumentos con nombre
- [x] Módulos e `import`
- [x] Paquetes y `__init__.py`
- [x] `__name__` y `__file__`
- [x] Rutas de archivos con `pathlib`
- [x] Variables de entorno (`os.environ`) y archivos `.env`
- [x] Manejo de errores (`try` / `except`)
- [x] Clases básicas
- [x] Type hints
- [x] `async` / `await` (intuición, sin profundizar)
- [x] Decoradores (qué hacen, sin escribirlos)
- [x] Entornos virtuales y por qué existen

## Capítulo 2 · FastAPI
- [x] Qué es FastAPI y qué es una API
- [x] Primera app: `FastAPI()` y la primera ruta
- [x] Servidor Uvicorn y recarga automática
- [x] Rutas con métodos HTTP (`GET`, `POST`, `PUT`, `DELETE`)
- [x] Parámetros de ruta y de query
- [x] Cuerpo de la petición (body) con JSON
- [x] Validación con Pydantic: modelos
- [x] Códigos de estado y respuestas de error
- [x] Documentación automática (`/docs`)
- [x] Organizar rutas con `APIRouter`
- [x] Inyección de dependencias con `Depends`
- [x] Configuración con `pydantic-settings`
- [x] CORS (qué es y por qué aparece)
- [x] `lifespan`: arranque y apagado de la app

## Capítulo 3 · Base de datos
- [x] Qué es una base de datos relacional
- [x] Tablas, filas y columnas
- [x] Qué es PostgreSQL
- [x] Qué es un ORM y qué es SQLAlchemy
- [x] Definir un modelo
- [x] Sesiones y conexión a la base
- [x] Crear, leer, actualizar y borrar (CRUD) con SQLAlchemy
- [x] Relaciones entre tablas (uno a muchos)
- [x] Migraciones con Alembic: qué son y por qué
- [x] Primera migración

## Capítulo 4 · Docker
- [x] Qué es un contenedor y qué es una imagen
- [x] Instalar Docker Desktop y comprobar
- [x] Levantar PostgreSQL con `docker run` (para entender qué pasa)
- [x] `docker-compose.yml`: qué es y para qué sirve
- [x] Volúmenes: dónde se guardan los datos
- [x] Variables de entorno en Compose
- [x] Dockerfile para la propia API
- [x] Levantar API y base juntas

## Capítulo 5 · Autenticación
- [x] Qué es autenticar y qué es autorizar
- [x] Por qué nunca guardar contraseñas en texto plano
- [x] Hashing de contraseñas (argon2 o bcrypt)
- [x] Modelo de usuario
- [x] Registro de usuarios
- [x] Login y verificación de contraseña
- [x] Qué es un JWT y cómo se usa
- [x] Proteger rutas con `Depends`
- [x] Roles y permisos básicos
- [x] Expiración de tokens y refresh tokens
- [x] Dónde guardar el token en el frontend (cookies httpOnly vs localStorage)
- [x] Recomendaciones de seguridad (HTTPS, rate limiting, secretos fuera del código)
- [x] Errores comunes de autenticación

## Capítulo 6 · Calidad y despliegue
- [x] Tests con pytest: primer test
- [x] Tests de endpoints con `httpx`
- [x] Linter y formateo con Ruff
- [x] Logs: qué registrar y qué no
- [x] Preparar la app para producción (variables, debug apagado)
- [x] Desplegar el backend (opciones y trade-offs)

## Apéndices
- [x] Glosario
- [x] Chuleta de comandos de `uv`, Docker y FastAPI
- [x] Errores comunes por capítulo
- [x] Cómo está hecha esta guía (MkDocs, GitHub Pages)

## Pendientes antes de arrancar
- [x] Crear el repo en GitHub y confirmar el nombre (`villalb4/python-MkDocs`)
- [x] Esqueleto: `mkdocs.yml`, `requirements.txt`, workflow de deploy, `docs/index.md`
- [ ] Activar GitHub Pages en el repo (Settings → Pages → Source: GitHub Actions)
