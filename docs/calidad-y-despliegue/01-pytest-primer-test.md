# Tests con pytest: primer test

!!! info "En esta página"

    **Qué vas a aprender:** qué es un test automático, cómo instalar pytest como dependencia de desarrollo, cómo escribir y correr tus primeros tests sobre las funciones de seguridad de la API, cómo leer un test que falla y cómo comprobar que algo tira un error con `pytest.raises`.

    **Qué necesitás:** la API del [Capítulo 5](../autenticacion/13-errores-comunes.md), con su `.env`. Para esta página no hace falta tener la base de datos levantada.

## Qué es un test

Hasta ahora, para saber si algo funcionaba, lo probabas a mano: levantabas el servidor, mandabas un pedido con `curl` o desde `/docs`, y mirabas la respuesta. Funciona, pero cada vez que cambiás algo tendrías que repetir todas esas pruebas, y nadie lo hace.

Un **test automático** es código que prueba tu código. Llama a una función con ciertos datos y comprueba que el resultado sea el esperado. Una vez escrito, lo corrés con un comando, todas las veces que quieras, y en segundos sabés si rompiste algo.

En Python, la herramienta más usada para esto es **pytest**.

## Instalar pytest

pytest se usa mientras desarrollás, pero tu API no lo necesita para funcionar. Por eso se instala como **dependencia de desarrollo**, con `--dev`:

```bash
uv add --dev pytest
```

uv lo anota en `pyproject.toml` en una sección aparte, `[dependency-groups]`, bajo `dev`. Es la razón del `--no-dev` del [Dockerfile](../docker/07-dockerfile.md): la imagen que va a producción no incluye pytest ni ninguna otra herramienta de desarrollo.

## El primer test

Los tests van en una carpeta `tests/`, al lado de `app/`. Creá `tests/test_seguridad.py`:

```python title="tests/test_seguridad.py"
from app.seguridad import hashear_contrasena, verificar_contrasena


def test_el_hash_no_es_la_contrasena():
    hash_guardado = hashear_contrasena("una-clave-larga")
    assert hash_guardado != "una-clave-larga"


def test_verificar_contrasena_correcta():
    hash_guardado = hashear_contrasena("una-clave-larga")
    assert verificar_contrasena("una-clave-larga", hash_guardado)


def test_verificar_contrasena_incorrecta():
    hash_guardado = hashear_contrasena("una-clave-larga")
    assert not verificar_contrasena("otra-clave", hash_guardado)
```

Cada test es una función común, sin nada especial:

- **El nombre empieza con `test_`.** Así pytest sabe que es un test. Lo mismo vale para el archivo: `test_seguridad.py`.
- **`assert` comprueba algo.** Si lo que sigue es verdadero, no pasa nada. Si es falso, el test falla.
- **Un test, una cosa.** El nombre dice qué se prueba. Cuando uno falle, el nombre ya te cuenta qué se rompió.

## Correr los tests

```bash
uv run pytest
```

La primera vez, falla antes de llegar a los tests:

```text
E   ModuleNotFoundError: No module named 'app'
```

pytest no encuentra el paquete `app`, porque no busca en la carpeta del proyecto. Se arregla diciéndole dónde buscar. Agregá esto al final de `pyproject.toml`:

```toml title="pyproject.toml (agregar al final)"
[tool.pytest]
pythonpath = ["."]
testpaths = ["tests"]
```

`pythonpath` agrega la carpeta del proyecto a los lugares donde Python busca módulos, y `testpaths` le dice a pytest que los tests están en `tests/`. Corré de nuevo:

```text
============================= test session starts ==============================
platform darwin -- Python 3.12.15, pytest-9.1.1, pluggy-1.7.0
rootdir: .../tareas-api
configfile: pyproject.toml
testpaths: tests
plugins: anyio-4.15.1
collected 3 items

tests/test_seguridad.py ...                                              [100%]

============================== 3 passed in 0.29s ===============================
```

Cada punto es un test que pasó. Con `uv run pytest -v` ves una línea por test, con su nombre y `PASSED`.

!!! note "Los tests leen el `.env`"

    `app/seguridad.py` usa la configuración, así que importarlo lee el `.env`. Si falta (o le faltan variables obligatorias), los tests no llegan a correr y ves el `ValidationError` de pydantic-settings: `3 validation errors for Configuracion`.

## Cuando un test falla

Para ver cómo se ve un test que falla, cambiá a propósito el `!=` del primero por `==` y corré `uv run pytest`:

```text
tests/test_seguridad.py F..                                              [100%]

=================================== FAILURES ===================================
_______________________ test_el_hash_no_es_la_contrasena _______________________

    def test_el_hash_no_es_la_contrasena():
        hash_guardado = hashear_contrasena("una-clave-larga")
>       assert hash_guardado == "una-clave-larga"
E       AssertionError: assert '$argon2id$v=...BvtaFmTY9Vfig' == 'una-clave-larga'
E         
E         - una-clave-larga
E         + $argon2id$v=19$m=65536,t=3,p=4$fPoRwRuQjG4SLhxGW63laQ$j47ZtN1HZEzCOGcMqYSkp0y9JIYAPXBvtaFmTY9Vfig

tests/test_seguridad.py:6: AssertionError
=========================== short test summary info ============================
FAILED tests/test_seguridad.py::test_el_hash_no_es_la_contrasena - AssertionE...
========================= 1 failed, 2 passed in 0.28s ==========================
```

La `F` marca el test que falló. La `>` señala la línea, y las líneas con `E` muestran los dos valores que se compararon: lo que esperabas (`-`) y lo que salió (`+`). Con un `assert` común, pytest te muestra todo eso solo. Volvé a poner el `!=` antes de seguir.

## Comprobar que algo da error: `pytest.raises`

A veces lo correcto es que una función **tire un error**. Por ejemplo, leer un token vencido. Para eso está `pytest.raises`. Creá `tests/test_tokens.py`:

```python title="tests/test_tokens.py"
from datetime import timedelta

import jwt
import pytest

from app.seguridad import crear_token, leer_token


def test_leer_un_token_recien_creado():
    token = crear_token(7, "acceso", timedelta(minutes=5))
    datos = leer_token(token)
    assert datos["sub"] == "7"
    assert datos["tipo"] == "acceso"


def test_un_token_vencido_da_error():
    token = crear_token(7, "acceso", timedelta(seconds=-1))
    with pytest.raises(jwt.ExpiredSignatureError):
        leer_token(token)
```

Con una duración negativa, el token nace vencido. El test pasa solo si el código dentro del `with` tira `ExpiredSignatureError`. Si no tira nada, falla con `Failed: DID NOT RAISE ExpiredSignatureError`. Y para correr solo algunos tests, `-k` filtra por nombre: `uv run pytest -k vencido` corre únicamente el segundo.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Node se usan Jest o Vitest. La idea es la misma, pero los tests van dentro de `test()` y se comprueba con `expect`:

    ```javascript
    test("verificar contraseña incorrecta", async () => {
      const hash = await argon2.hash("una-clave-larga")
      expect(await argon2.verify(hash, "otra-clave")).toBe(false)
    })
    ```

    `expect(() => ...).toThrow()` es el equivalente de `pytest.raises`. Y Jest también se instala como dependencia de desarrollo (`npm install --save-dev jest`).

??? example "Ver comparación en Django ⬇️"

    Django trae su propio sistema de tests, basado en `unittest`: los tests son métodos de una clase que hereda de `TestCase`, se comprueba con `self.assertEqual(...)` o `self.assertRaises(...)`, y se corren con `python manage.py test`. Muchos proyectos de Django usan pytest igual, con el plugin `pytest-django`, para escribir tests como funciones y con `assert` común.

??? example "Ver comparación en Spring Boot ⬇️"

    En Java se usa JUnit 5: cada test es un método con `@Test`, se comprueba con `assertEquals(...)` o `assertTrue(...)`, y `assertThrows(ExpiredJwtException.class, () -> ...)` equivale a `pytest.raises`. Se corren con `./mvnw test`, y las dependencias de test se declaran con `<scope>test</scope>`, la misma idea que `--dev`.

## Mini ejercicio

Agregá dos tests a `tests/test_tokens.py`:

1. `test_token_de_refresco`: que un token creado con `crear_token_refresco(3)` se lea con `sub` igual a `"3"` y `tipo` igual a `"refresco"`.
2. `test_token_alterado_da_error`: que un token de acceso al que le cortás el último carácter (`token[:-1]`) dé error al leerlo.

??? success "Ver solución"

    ```python
    from app.seguridad import crear_token, crear_token_acceso, crear_token_refresco, leer_token


    def test_token_de_refresco():
        datos = leer_token(crear_token_refresco(3))
        assert datos["sub"] == "3"
        assert datos["tipo"] == "refresco"


    def test_token_alterado_da_error():
        token = crear_token_acceso(3)
        with pytest.raises(jwt.InvalidTokenError):
            leer_token(token[:-1])
    ```

    En el segundo conviene usar `jwt.InvalidTokenError`, la clase madre de todos los errores de token de PyJWT. Al cortar el último carácter, casi siempre el error es `DecodeError` (la firma ya no se puede decodificar), pero a veces es `InvalidSignatureError`. Con la clase madre, el test acepta cualquiera de los dos. `uv run pytest` tiene que mostrar `7 passed`.

---

**Siguiente:** [Tests de endpoints con `httpx` →](02-tests-de-endpoints.md)
