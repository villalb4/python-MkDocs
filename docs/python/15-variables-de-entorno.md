# Variables de entorno y archivos `.env`

!!! info "En esta página"

    **Qué vas a aprender:** qué son las variables de entorno, cómo leerlas desde Python con `os.environ` y cómo usar un archivo `.env` para no escribir configuración ni secretos en el código.

    **Qué necesitás:** [Rutas de archivos con `pathlib`](14-pathlib.md).

## Configuración fuera del código

Tu API va a necesitar datos que **cambian según dónde corre**: la dirección de la base de datos, el puerto, si está en modo de pruebas. Y algunos son **secretos**, como contraseñas o claves para firmar tokens.

Escribirlos directamente en el código es mala idea:

- Para cambiarlos tendrías que modificar el código.
- Si subís el código a GitHub, los secretos quedan expuestos.

La solución es pasarlos desde afuera con **variables de entorno**: pares `NOMBRE=valor` que el sistema operativo le entrega a cada programa que se ejecuta.

## Leer variables con `os.environ`

El módulo `os` tiene `os.environ`, que funciona como un diccionario con todas las variables de entorno:

```python title="leer.py"
import os

print(os.environ["HOME"])  # /Users/ana (en Windows, probá con "USERPROFILE")
```

Como con cualquier diccionario, si la variable no existe, los corchetes dan `KeyError` y `.get()` te deja poner un valor por defecto:

```python title="config.py"
import os

nombre = os.environ.get("APP_NOMBRE", "Mi API")
puerto = int(os.environ.get("APP_PUERTO", "8000"))

print(f"{nombre} en el puerto {puerto}")
```

Si lo ejecutás tal cual, muestra `Mi API en el puerto 8000`: como no definiste las variables, usa los valores por defecto.

**Regla práctica:** usá `.get()` con un valor por defecto para lo opcional (nombre, puerto) y corchetes para lo **obligatorio** (una clave secreta). Si falta un secreto, es mejor que el programa se corte de entrada con un `KeyError` claro a que arranque mal configurado.

## Definir variables en la terminal

=== "macOS / Linux"

    Para un solo comando, ponelas adelante:

    ```bash
    APP_NOMBRE="Tareas API" APP_PUERTO=9000 uv run config.py
    ```

    Para toda la sesión de la terminal, usá `export`:

    ```bash
    export APP_NOMBRE="Tareas API"
    uv run config.py
    ```

=== "Windows (PowerShell)"

    Definilas con `$env:` y quedan para toda la sesión de la terminal:

    ```powershell
    $env:APP_NOMBRE = "Tareas API"
    $env:APP_PUERTO = "9000"
    uv run config.py
    ```

Con las dos variables definidas, el script muestra `Tareas API en el puerto 9000`. Al cerrar la terminal, las variables desaparecen.

## Siempre llegan como texto

Las variables de entorno son **siempre strings**. Por eso en `config.py` convertimos el puerto con `int(...)`. Con los valores de verdadero/falso hay una trampa:

```python title="bools.py"
import os

debug_mal = bool(os.environ.get("APP_DEBUG", "false"))
print(debug_mal)  # True (!)

debug = os.environ.get("APP_DEBUG", "false").lower() in ("1", "true", "yes")
print(debug)  # False
```

`bool("false")` da `True`, porque `"false"` es un texto **no vacío**, y en la página de condicionales viste que eso cuenta como verdadero. Hay que comparar el texto.

!!! tip "No vas a hacer esto a mano"

    En el Capítulo 2 vas a usar `pydantic-settings`, que lee las variables, las convierte al tipo correcto (incluidos los `bool`) y te avisa si falta alguna obligatoria. Pero hace exactamente esto por debajo, y conviene entenderlo.

## Archivos `.env`

Definir diez variables a mano cada vez que abrís la terminal es tedioso. Lo habitual es guardarlas en un archivo llamado `.env` en la raíz del proyecto:

```bash title=".env"
# Configuración local: NO subir a Git
APP_NOMBRE="Tareas API"
APP_PUERTO=9000
CLAVE_SECRETA=cambiame-por-algo-largo-y-aleatorio
```

- Un par `NOMBRE=valor` por renglón, sin espacios alrededor del `=`.
- Los renglones que empiezan con `#` son comentarios.
- Si el valor tiene espacios, va **entre comillas**.

Python no lee el `.env` solo. Con `uv` se lo pasás con `--env-file`:

```bash
uv run --env-file .env config.py
```

Ahora muestra `Tareas API en el puerto 9000` sin definir nada en la terminal. Si una variable ya estaba definida en la terminal, esa gana sobre la del archivo.

!!! warning "Comillas para valores con espacios"

    Si escribís `APP_NOMBRE=Tareas API` sin comillas, `uv` avisa `warning: Failed to parse environment file` y **no carga ninguna variable** del archivo, ni siquiera las que estaban bien. Si ves ese aviso, buscá un valor con espacios.

## El `.env` nunca va a Git

El `.env` tiene secretos, así que **no se sube al repositorio**. Agregalo al archivo `.gitignore`:

```text title=".gitignore"
.env
```

Para que otras personas sepan qué variables hacen falta, se sube en cambio un `.env.example` con los mismos nombres y valores de ejemplo (nunca los reales):

```bash title=".env.example"
APP_NOMBRE="Tareas API"
APP_PUERTO=8000
CLAVE_SECRETA=poné-acá-tu-clave
```

Quien clona el proyecto copia `.env.example` a `.env` y completa sus valores.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | Node | Python |
    | --- | --- |
    | `process.env.APP_PUERTO` | `os.environ["APP_PUERTO"]` |
    | `process.env.APP_PUERTO ?? "8000"` | `os.environ.get("APP_PUERTO", "8000")` |
    | `node --env-file=.env app.js` | `uv run --env-file .env app.py` |
    | `require("dotenv").config()` | `pydantic-settings` (Capítulo 2) |

    Una diferencia: `process.env.NO_EXISTE` da `undefined` sin avisar; `os.environ["NO_EXISTE"]` da `KeyError`. Y en los dos casos los valores son siempre strings.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. El `settings.py` que genera Django trae `SECRET_KEY` y `DEBUG = True` escritos en el código; en producción se reemplazan por `os.environ["DJANGO_SECRET_KEY"]` y algo como `os.environ.get("DJANGO_DEBUG") == "1"`. Es el mismo patrón que viste en esta página.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring Boot la configuración va en `application.properties` y se puede pisar con variables de entorno (`SERVER_PORT=9000` reemplaza a `server.port`). En Python no hay un archivo de configuración estándar: la fuente de verdad son las variables de entorno, y el `.env` es solo una comodidad para desarrollo.

    | Spring Boot | Python |
    | --- | --- |
    | `@Value("${app.nombre:Mi API}")` | `os.environ.get("APP_NOMBRE", "Mi API")` |
    | `System.getenv("APP_PUERTO")` | `os.environ.get("APP_PUERTO")` |
    | `@ConfigurationProperties` | `pydantic-settings` (Capítulo 2) |

## Mini ejercicio

Escribí un script que lea tres variables: `APP_NOMBRE` (por defecto `"Mi API"`), `MAX_TAREAS` (un número, por defecto `100`) y `APP_DEBUG` (`True` solo si vale `"true"`, por defecto `False`). Después creá un `.env` que las defina y ejecutalo con y sin `--env-file`.

??? success "Ver solución"

    ```python title="ejercicio.py"
    import os

    nombre = os.environ.get("APP_NOMBRE", "Mi API")
    max_tareas = int(os.environ.get("MAX_TAREAS", "100"))
    modo_debug = os.environ.get("APP_DEBUG", "false").lower() == "true"

    print(f"{nombre}: hasta {max_tareas} tareas, debug={modo_debug}")
    ```

    ```bash title=".env"
    APP_NOMBRE="Agenda"
    MAX_TAREAS=50
    APP_DEBUG=true
    ```

    - `uv run ejercicio.py` → `Mi API: hasta 100 tareas, debug=False`
    - `uv run --env-file .env ejercicio.py` → `Agenda: hasta 50 tareas, debug=True`

---

**Siguiente:** [Manejo de errores →](16-manejo-de-errores.md)
