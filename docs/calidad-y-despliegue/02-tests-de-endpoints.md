# Tests de endpoints con `httpx`

!!! info "En esta página"

    **Qué vas a aprender:** a probar las rutas de tu API sin levantar el servidor, con el `TestClient` de FastAPI; a usar una base de datos aparte para los tests; y a preparar lo que necesitan los tests con una *fixture* de pytest.

    **Qué necesitás:** [Tests con pytest: primer test](01-pytest-primer-test.md), y la base de Compose levantada (`docker compose up -d db`).

## `TestClient`: pedidos sin servidor

En la página anterior probaste funciones sueltas. Ahora vas a probar la API entera: mandar un pedido a `/auth/registro` y comprobar el código de estado y el JSON, igual que hacías con `curl`, pero desde un test.

Para eso, FastAPI trae **`TestClient`**. Le pasás tu `app` y te da un cliente con los métodos de siempre (`get`, `post`, `put`, `delete`). Los pedidos no pasan por la red: van directo a la app, en el mismo proceso. No hace falta Uvicorn.

Por dentro, `TestClient` usa **httpx**, una librería para hacer pedidos HTTP. Hoy, en realidad, usa **`httpx2`**, la continuación de httpx que mantiene el equipo de Pydantic. Viene con `fastapi[standard]`, así que no tenés que instalar nada.

!!! warning "Si ves un aviso sobre `httpx`"

    Si instalaste `fastapi` sin `[standard]` y agregaste `httpx` a mano (como dicen muchos tutoriales), pytest termina con este aviso:

    ```text
    StarletteDeprecationWarning: Using `httpx` with `starlette.testclient` is deprecated; install `httpx2` instead.
    ```

    Los tests funcionan igual, pero se arregla con `uv add --dev httpx2`.

## Una base de datos para los tests

Los tests van a registrar usuarios y crear tareas. Si usaran tu base de desarrollo, la llenarían de datos de prueba, y un test podría fallar por datos que dejó otro. Por eso usan **su propia base**, que se vacía antes de cada test. Creala dentro del mismo PostgreSQL de Compose:

```bash
docker compose exec db createdb -U tareas tareas_test
```

## La fixture `cliente`

Cada test necesita lo mismo: la base vacía y un `TestClient` listo. En pytest, eso se escribe una sola vez como una **fixture**: una función marcada con `@pytest.fixture` que prepara algo. Un test la pide **poniendo su nombre como parámetro**, y pytest se la pasa ya preparada.

Las fixtures que usan varios archivos de tests van en `tests/conftest.py`. pytest lo lee solo, sin importarlo:

```python title="tests/conftest.py"
import asyncio
import os

# Los tests usan su propia base, para no tocar los datos de desarrollo.
# Tiene que ir antes de importar la app, que lee la configuración al arrancar.
os.environ["URL_BASE_DATOS"] = "postgresql+asyncpg://tareas:tareas@localhost:5432/tareas_test"

import pytest
from fastapi.testclient import TestClient

from app import modelos  # noqa: F401  (registra las tablas en Base)
from app.db import Base, motor
from app.dependencias import intentos_por_ip
from app.main import app


async def recrear_tablas():
    async with motor.begin() as conexion:
        await conexion.run_sync(Base.metadata.drop_all)
        await conexion.run_sync(Base.metadata.create_all)
    await motor.dispose()


@pytest.fixture
def cliente():
    asyncio.run(recrear_tablas())
    intentos_por_ip.clear()
    with TestClient(app) as cliente:
        yield cliente
```

Paso por paso:

- **`os.environ[...]`** cambia la URL de la base antes de importar la app. pydantic-settings le da prioridad a las variables de entorno sobre el `.env`, así que la app arranca apuntando a `tareas_test`. (Si tu base usa otro puerto, cambialo acá también.)
- **`recrear_tablas`** borra todas las tablas y las vuelve a crear vacías. Es la misma `create_all` del Capítulo 3: para los tests no hacen falta las migraciones. `asyncio.run` la ejecuta desde una función común.
- **`intentos_por_ip.clear()`** vacía el contador del [límite de intentos](../autenticacion/12-recomendaciones-seguridad.md). Sin esto, como los tests hacen muchos logins en pocos segundos, a partir del sexto la API responde `429` y los tests fallan con `KeyError: 'access_token'`.
- **`with TestClient(app)`** ejecuta el `lifespan` al entrar y al salir, igual que un servidor de verdad.
- **`yield`** entrega el cliente al test. Cuando el test termina, la fixture sigue después del `yield`: sale del `with` y se apaga la app.

!!! note "Los dos `dispose()`"

    Cada `TestClient` (y cada `asyncio.run`) usa su propio *event loop*, y una conexión de asyncpg no se puede usar desde otro loop. Por eso las conexiones se cierran con `motor.dispose()` al terminar `recrear_tablas` y en el apagado del `lifespan`. Si te olvidás de alguno, los tests fallan con un `RuntimeError` que termina en `attached to a different loop`.

## Los primeros tests de endpoints

```python title="tests/test_endpoints.py"
def test_salud(cliente):
    respuesta = cliente.get("/salud")
    assert respuesta.status_code == 200
    assert respuesta.json() == {"base_de_datos": "ok"}


def test_registro(cliente):
    datos = {"nombre": "Ana", "email": "Ana@Ejemplo.com", "contrasena": "una-clave-larga"}
    respuesta = cliente.post("/auth/registro", json=datos)
    assert respuesta.status_code == 201
    usuario = respuesta.json()
    assert usuario["email"] == "ana@ejemplo.com"
    assert usuario["rol"] == "usuario"
    assert "contrasena" not in usuario
    assert "hash_contrasena" not in usuario


def test_tareas_sin_token(cliente):
    respuesta = cliente.get("/tareas")
    assert respuesta.status_code == 401
```

`json=` manda el diccionario como cuerpo JSON. `respuesta.status_code` es el código y `respuesta.json()` es el cuerpo ya convertido a diccionario. Fijate en las dos últimas líneas de `test_registro`: comprueban el [error más peligroso del capítulo anterior](../autenticacion/13-errores-comunes.md), que el hash se filtre en la respuesta. Ahora, si alguien lo rompe, un test lo avisa.

## Tests con login

Para las rutas protegidas, el test tiene que registrarse e iniciar sesión primero. Como lo van a hacer todos, conviene una función auxiliar. Su nombre no empieza con `test_`, así que pytest no la corre como test:

```python title="tests/test_tareas.py"
def iniciar_sesion(cliente, email):
    datos = {"nombre": "Prueba", "email": email, "contrasena": "una-clave-larga"}
    cliente.post("/auth/registro", json=datos)
    respuesta = cliente.post("/auth/login", data={"username": email, "password": "una-clave-larga"})
    token = respuesta.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}


def test_crear_y_listar_tareas(cliente):
    encabezados = iniciar_sesion(cliente, "ana@ejemplo.com")
    respuesta = cliente.post("/tareas", json={"titulo": "Comprar pan"}, headers=encabezados)
    assert respuesta.status_code == 201
    assert respuesta.json()["hecha"] is False

    respuesta = cliente.get("/tareas", headers=encabezados)
    assert [tarea["titulo"] for tarea in respuesta.json()] == ["Comprar pan"]


def test_no_se_ven_tareas_ajenas(cliente):
    de_ana = iniciar_sesion(cliente, "ana@ejemplo.com")
    de_juan = iniciar_sesion(cliente, "juan@ejemplo.com")
    tarea = cliente.post("/tareas", json={"titulo": "Secreta"}, headers=de_ana).json()

    respuesta = cliente.get(f"/tareas/{tarea['id']}", headers=de_juan)
    assert respuesta.status_code == 404
```

El login usa `data=` en vez de `json=`, porque espera un formulario. `uv run pytest` ahora corre los tests de las dos páginas: `12 passed`. Si te olvidaste de crear la base, falla con `database "tareas_test" does not exist`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Node se usa **supertest**, que cumple el papel de `TestClient`: recibe la app de Express y manda pedidos sin levantar el servidor.

    ```javascript
    const res = await request(app).post("/auth/registro").send(datos)
    expect(res.status).toBe(201)
    ```

    Las fixtures de pytest se reemplazan con `beforeEach`, y la base de tests se configura a mano, igual que acá.

??? example "Ver comparación en Django ⬇️"

    Django resuelve la base de tests solo: `python manage.py test` crea una base `test_<nombre>`, aplica las migraciones, y envuelve cada test en una transacción que deshace al final. El equivalente de `TestClient` es `self.client` (o `APIClient` en DRF): `self.client.post("/auth/registro/", datos, format="json")`.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring se usa `@SpringBootTest` con **MockMvc** (o `WebTestClient`), que manda pedidos a la app sin un servidor real: `mockMvc.perform(post("/auth/registro").content(json))`. La base de tests se suele configurar con un perfil aparte (`application-test.properties`) o con Testcontainers, que levanta un PostgreSQL en Docker solo para los tests.

## Mini ejercicio

Agregá a `tests/test_endpoints.py` un test `test_registro_con_email_repetido` que registre dos veces el mismo email y compruebe que la segunda respuesta es `409`, con el mensaje de error de la API.

??? success "Ver solución"

    ```python
    def test_registro_con_email_repetido(cliente):
        datos = {"nombre": "Ana", "email": "ana@ejemplo.com", "contrasena": "una-clave-larga"}
        cliente.post("/auth/registro", json=datos)
        respuesta = cliente.post("/auth/registro", json=datos)
        assert respuesta.status_code == 409
        assert respuesta.json() == {"detail": "Ya existe una cuenta con ese email"}
    ```

    El test pasa aunque `test_registro` ya haya registrado a `ana@ejemplo.com`, porque la fixture vacía la base antes de cada test. Ese es el sentido de recrear las tablas: cada test empieza de cero y no depende del orden en que se corren. `uv run pytest` tiene que mostrar `13 passed`.

---

**Siguiente:** [Linter y formateo con Ruff →](03-ruff.md)
