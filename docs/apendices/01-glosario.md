# Glosario

!!! info "En esta página"

    **Qué vas a aprender:** nada nuevo. Es una página de referencia: los términos de la guía, agrupados por tema, con una definición corta y un enlace a la página donde se explican.

    **Qué necesitás:** nada. Volvé acá cada vez que una palabra no te suene.

Cada término lleva a la página donde aparece explicado por primera vez. Si buscás uno en particular, usá el buscador de arriba.

## Python y herramientas

- **[Script](../python/01-que-es-python.md)**: un archivo `.py` con instrucciones que Python ejecuta de arriba hacia abajo.
- **[uv](../python/02-instalar-uv.md)**: la herramienta que instala Python, crea el entorno virtual y maneja las dependencias del proyecto.
- **[Variable](../python/03-variables-y-tipos.md)**: un nombre que apunta a un valor. Los tipos básicos son `int`, `float`, `str`, `bool` y `None`.
- **[f-string](../python/04-strings-y-f-strings.md)**: un texto con `f` adelante, que reemplaza lo que está entre `{}` por su valor.
- **[Lista](../python/05-listas.md)**: una secuencia ordenada de valores, entre `[]`.
- **[Diccionario](../python/06-diccionarios.md)**: pares de clave y valor, entre `{}`. Es lo que se convierte en un objeto JSON.
- **[Función](../python/09-funciones.md)**: un bloque de código con nombre, que recibe **parámetros** y devuelve un valor con `return`.
- **[Argumento con nombre](../python/10-parametros.md)**: pasar un valor indicando a qué parámetro va: `crear(titulo="Pan")`.
- **[Módulo](../python/11-modulos-e-import.md)**: un archivo `.py` que se puede importar desde otro con `import`.
- **[Paquete](../python/12-paquetes.md)**: una carpeta de módulos, con un `__init__.py`. En la guía, `app/`.
- **[`__name__`](../python/13-name-y-file.md)**: el nombre del módulo actual; vale `"__main__"` si el archivo se ejecuta directamente.
- **[`pathlib`](../python/14-pathlib.md)**: el módulo para trabajar con rutas de archivos sin pelearse con las barras de cada sistema.
- **[Variable de entorno](../python/15-variables-de-entorno.md)**: un valor que el sistema le pasa al programa. El archivo **`.env`** las guarda para desarrollo.
- **[Excepción](../python/16-manejo-de-errores.md)**: un error que interrumpe el programa, salvo que se atrape con `try` / `except`. El **traceback** muestra dónde ocurrió.
- **[Clase](../python/17-clases-basicas.md)**: un molde para crear objetos con datos (atributos) y funciones (métodos).
- **[Type hint](../python/18-type-hints.md)**: una anotación del tipo esperado, como `titulo: str`. FastAPI y Pydantic las usan para validar.
- **[`async` / `await`](../python/19-async-await.md)**: la forma de escribir código que espera (a la red, a la base) sin bloquear al resto.
- **[Decorador](../python/20-decoradores.md)**: una línea con `@` encima de una función, que le agrega comportamiento. Por ejemplo, `@app.get("/")`.
- **[Entorno virtual](../python/21-entornos-virtuales.md)**: la carpeta `.venv` con las librerías de un proyecto, separadas de las de otros. Se describen en **`pyproject.toml`** y las versiones exactas quedan en **`uv.lock`**.

## HTTP y APIs

- **[API](../fastapi/01-que-es-fastapi.md)**: un programa que otros programas usan mandándole pedidos. Una API web responde por HTTP, normalmente con JSON.
- **[Ruta](../fastapi/02-primera-app.md)** (o *endpoint*): una dirección de la API, como `/tareas`, junto con el método que atiende.
- **[Método HTTP](../fastapi/04-metodos-http.md)**: el tipo de pedido: `GET` (leer), `POST` (crear), `PUT` (reemplazar), `DELETE` (borrar).
- **[Parámetro de ruta y de query](../fastapi/05-parametros.md)**: datos en la URL: `/tareas/3` (de ruta) y `/tareas?limite=5` (de query).
- **[Body](../fastapi/06-body-json.md)**: el cuerpo del pedido, donde viajan los datos (en la guía, en JSON).
- **[Código de estado](../fastapi/08-codigos-de-estado.md)**: el número que resume la respuesta: `200` bien, `201` creado, `404` no existe, `422` datos inválidos, `500` error del servidor.
- **[Encabezado](../fastapi/06-body-json.md)** (*header*): un dato extra del pedido o la respuesta, como `Authorization` o `Content-Type`.
- **[CORS](../fastapi/13-cors.md)**: la regla del navegador que bloquea pedidos a otro origen, salvo que la API los permita.
- **[HTTPS](../autenticacion/12-recomendaciones-seguridad.md)**: HTTP cifrado. Sin él, contraseñas y tokens viajan a la vista.

## FastAPI

- **[FastAPI](../fastapi/01-que-es-fastapi.md)**: el framework de la guía: definís rutas con funciones y type hints, y se encarga de validar, convertir a JSON y documentar.
- **[Uvicorn](../fastapi/03-uvicorn.md)**: el servidor que ejecuta la app. Habla con FastAPI con un estándar llamado **ASGI**. `fastapi dev` lo levanta con **recarga automática**; `fastapi run`, sin ella.
- **[Pydantic](../fastapi/07-pydantic-modelos.md)**: la librería que valida datos con clases. Un **esquema** (`TareaCrear`, `UsuarioRespuesta`) define qué entra y qué sale.
- **[`/docs`](../fastapi/09-documentacion.md)**: la documentación interactiva (Swagger UI), generada a partir del formato estándar **OpenAPI**.
- **[`APIRouter`](../fastapi/10-apirouter.md)**: un grupo de rutas en su propio archivo, que después se suma a la app.
- **[Dependencia](../fastapi/11-depends.md)**: una función que FastAPI ejecuta antes de la ruta, pedida con `Depends`. Sirve para la sesión de la base, el usuario actual o la paginación.
- **[pydantic-settings](../fastapi/12-pydantic-settings.md)**: la librería que lee la configuración de variables de entorno y del `.env`, y la valida.
- **[Middleware](../fastapi/13-cors.md)**: código que se ejecuta en todos los pedidos, antes y después de las rutas, como el de CORS.
- **[`lifespan`](../fastapi/14-lifespan.md)**: el código que se ejecuta al arrancar y al apagar la app.

## Base de datos

- **[Base de datos relacional](../base-de-datos/01-que-es-una-base-de-datos.md)**: datos guardados en tablas que se relacionan entre sí, consultados con **SQL**.
- **[Tabla, fila y columna](../base-de-datos/02-tablas-filas-columnas.md)**: la tabla es el tipo de dato (tareas), cada fila es un registro y cada columna, un campo. La **clave primaria** (`id`) identifica cada fila.
- **[PostgreSQL](../base-de-datos/03-que-es-postgresql.md)**: la base de datos de la guía.
- **[ORM](../base-de-datos/04-orm-sqlalchemy.md)**: una librería que traduce clases de Python a tablas y objetos a filas. En la guía, **SQLAlchemy**, con **asyncpg** para hablar con PostgreSQL.
- **[Modelo](../base-de-datos/05-definir-un-modelo.md)**: la clase de SQLAlchemy que describe una tabla (`Tarea`, `Usuario`). No confundir con el esquema de Pydantic.
- **[Sesión y motor](../base-de-datos/06-sesiones-y-conexion.md)**: el motor maneja las conexiones con la base; la sesión agrupa las operaciones de un pedido hasta el `commit`.
- **[CRUD](../base-de-datos/07-crud.md)**: crear, leer, actualizar y borrar, las cuatro operaciones básicas.
- **[Clave foránea](../base-de-datos/08-relaciones.md)**: una columna que apunta a la clave primaria de otra tabla, como `usuario_id`. Arma las relaciones **uno a muchos**.
- **[Migración](../base-de-datos/09-migraciones-que-son.md)**: un archivo con un cambio de la estructura de la base, aplicado en orden con **Alembic**.

## Docker

- **[Imagen y contenedor](../docker/01-contenedores-e-imagenes.md)**: la imagen es el paquete con todo lo necesario para correr algo; el contenedor, una copia de la imagen en ejecución. Se descargan de un **registro**, como Docker Hub.
- **[Puerto publicado](../docker/03-postgres-docker-run.md)**: `-p 5432:5432` conecta un puerto de tu máquina con uno del contenedor.
- **[Docker Compose](../docker/04-docker-compose.md)**: el archivo `docker-compose.yml`, que describe varios contenedores (**servicios**) y los levanta juntos.
- **[Volumen](../docker/05-volumenes.md)**: un espacio donde los datos sobreviven aunque se borre el contenedor.
- **[Dockerfile](../docker/07-dockerfile.md)**: la receta para construir la imagen de tu API. El **`.dockerignore`** dice qué no copiar.
- **[Healthcheck](../docker/08-api-y-base-juntas.md)**: un comando que dice si un servicio está listo, para que los demás lo esperen.

## Autenticación

- **[Autenticar y autorizar](../autenticacion/01-autenticar-y-autorizar.md)**: autenticar es saber quién sos (si falla, `401`); autorizar, qué podés hacer (si falla, `403`).
- **[Hash](../autenticacion/02-contrasenas-texto-plano.md)**: un resumen de un texto que no se puede revertir. La **sal** hace que dos contraseñas iguales den hashes distintos.
- **[argon2](../autenticacion/03-hashing-argon2.md)**: el algoritmo de hash para contraseñas de la guía, lento a propósito. Lo usa la librería **pwdlib**.
- **[JWT](../autenticacion/07-jwt.md)**: un token con datos (como el id del usuario) y una **firma** que prueba que lo emitió tu API. Se manda como `Authorization: Bearer <token>`.
- **[Rol](../autenticacion/09-roles.md)**: lo que un usuario puede hacer, como `usuario` o `admin`.
- **[Token de acceso y de refresco](../autenticacion/10-refresh-tokens.md)**: el de acceso dura poco y se usa en cada pedido; el de refresco dura más y solo sirve para pedir uno nuevo.
- **[Cookie `httpOnly`](../autenticacion/11-donde-guardar-token.md)**: una cookie que el JavaScript de la página no puede leer, lo que la protege del **XSS**. `SameSite` la protege del **CSRF**.
- **[Rate limiting](../autenticacion/12-recomendaciones-seguridad.md)**: limitar cuántos pedidos se aceptan en un tiempo, como el límite de intentos de login (`429`).

## Calidad y despliegue

- **[Test](../calidad-y-despliegue/01-pytest-primer-test.md)**: código que prueba código, con `assert`. Se corre con **pytest**. pytest se instala como **dependencia de desarrollo** (`--dev`).
- **[Fixture](../calidad-y-despliegue/02-tests-de-endpoints.md)**: una función de pytest que prepara lo que necesita un test, y que el test pide como parámetro. Las compartidas van en `conftest.py`.
- **[`TestClient`](../calidad-y-despliegue/02-tests-de-endpoints.md)**: el cliente que manda pedidos a la app sin levantar el servidor. Usa **httpx2**, la continuación de **httpx**.
- **[Linter y formateador](../calidad-y-despliegue/03-ruff.md)**: el linter busca errores sin ejecutar el código; el formateador ordena su forma. **Ruff** hace las dos cosas.
- **[Log](../calidad-y-despliegue/04-logs.md)**: un mensaje con fecha, origen y **nivel** (`DEBUG`, `INFO`, `WARNING`, `ERROR`), escrito con el módulo `logging`.
- **[Producción](../calidad-y-despliegue/05-preparar-para-produccion.md)**: la API publicada, con `debug` apagado. **`SecretStr`** evita que los secretos aparezcan en logs.
- **[Desplegar](../calidad-y-despliegue/06-desplegar.md)**: publicar la API. Puede ir en una **plataforma de contenedores** o en un **VPS** detrás de un **proxy inverso** (como Caddy), que se encarga de HTTPS.

## Equivalencias

??? example "Ver equivalencias con Express ⬇️"

    | En esta guía | En Express |
    | --- | --- |
    | `uv`, `pyproject.toml`, `uv.lock` | `npm`, `package.json`, `package-lock.json` |
    | Uvicorn | El servidor HTTP de Node (`app.listen`) |
    | `APIRouter` | `express.Router()` |
    | Dependencias (`Depends`) | Middlewares por ruta |
    | Pydantic | zod o joi |
    | SQLAlchemy y Alembic | Prisma o Sequelize, con sus migraciones |
    | pytest y `TestClient` | Jest y supertest |
    | Ruff | ESLint y Prettier |

??? example "Ver equivalencias con Django ⬇️"

    | En esta guía | En Django |
    | --- | --- |
    | Ruta (función con `@app.get`) | *View*, más su entrada en `urls.py` |
    | Esquema de Pydantic | *Serializer* de DRF |
    | Modelo de SQLAlchemy | Modelo de Django (`models.Model`) |
    | Migraciones de Alembic | `makemigrations` y `migrate` |
    | pydantic-settings | `settings.py` |
    | `fastapi dev` | `python manage.py runserver` |
    | pytest con una fixture `cliente` | `TestCase` con `self.client` |

??? example "Ver equivalencias con Spring Boot ⬇️"

    | En esta guía | En Spring Boot |
    | --- | --- |
    | `uv` y `pyproject.toml` | Maven o Gradle, y `pom.xml` |
    | Ruta (`@app.get`) | Método con `@GetMapping` en un `@RestController` |
    | Dependencias (`Depends`) | Inyección de dependencias con `@Autowired` o por constructor |
    | Esquema de Pydantic | DTO con anotaciones de validación |
    | Modelo de SQLAlchemy | Entidad JPA (`@Entity`) |
    | Alembic | Flyway o Liquibase |
    | pytest | JUnit |
    | `logging` | SLF4J con Logback |

## Mini ejercicio

Sin mirar el glosario, explicá con tus palabras la diferencia entre estos tres pares, que en la guía se usan mucho y es fácil mezclar: **modelo** y **esquema**; **imagen** y **contenedor**; **token de acceso** y **token de refresco**.

??? success "Ver solución"

    - **Modelo y esquema:** el modelo (SQLAlchemy) describe cómo se guarda un dato en la base, con todas sus columnas, incluido `hash_contrasena`. El esquema (Pydantic) describe qué datos entran o salen de la API. Por eso hay un `Usuario` (modelo) y un `UsuarioRespuesta` (esquema) sin el hash.
    - **Imagen y contenedor:** la imagen es el paquete, que no cambia; el contenedor es una copia de la imagen funcionando. De una imagen podés levantar muchos contenedores.
    - **Token de acceso y de refresco:** el de acceso va en cada pedido y vence en minutos; el de refresco vive en una cookie `httpOnly`, dura días y solo sirve para conseguir un token de acceso nuevo en `/auth/refrescar`.

---

**Siguiente:** [Chuleta de comandos de `uv`, Docker y FastAPI →](02-chuleta-de-comandos.md)
