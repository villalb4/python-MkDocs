# `lifespan`: arranque y apagado de la app

!!! info "En esta página"

    **Qué vas a aprender:** cómo ejecutar código una sola vez cuando la app arranca y cuando se apaga, usando `lifespan`. Como ejemplo, vas a hacer que las tareas se guarden en un archivo y sobrevivan a los reinicios.

    **Qué necesitás:** [CORS](13-cors.md), [`async` / `await`](../python/19-async-await.md) y [Rutas con `pathlib`](../python/14-pathlib.md).

## Código que corre una sola vez

Tus rutas se ejecutan **en cada pedido**. Pero hay tareas que tienen que hacerse **una sola vez**:

- **Al arrancar:** conectarse a la base de datos, cargar datos iniciales, verificar que un servicio externo responde.
- **Al apagar:** cerrar esas conexiones, guardar lo que esté pendiente.

FastAPI llama a esa vida de la app, desde que arranca hasta que se apaga, ***lifespan***, y te deja engancharte en los dos extremos.

## La función `lifespan`

Tiene una forma particular, que se lee en tres partes:

```python
from contextlib import asynccontextmanager


@asynccontextmanager
async def lifespan(app: FastAPI):
    print("Arrancando...")      # 1. Antes de atender pedidos
    yield                       # 2. Acá la app atiende pedidos
    print("Apagando...")        # 3. Cuando el servidor se detiene
```

- Lo que está **antes de `yield`** se ejecuta al arrancar, antes del primer pedido.
- **`yield`** es el punto donde la función "se pausa" mientras la app está prendida, que puede ser durante horas.
- Lo que está **después de `yield`** se ejecuta cuando el servidor se apaga.

`@asynccontextmanager` (de la biblioteca estándar) es el decorador que convierte esa función en algo que FastAPI sabe usar. No hace falta entender cómo funciona por dentro: con respetar la forma alcanza. Después se la pasás a la app con `FastAPI(lifespan=lifespan)`.

## Ejemplo: que las tareas sobrevivan a los reinicios

Hasta ahora, cada vez que el servidor se reinicia, la lista `tareas` vuelve a su estado inicial. Con `lifespan` podés cargarla de un archivo al arrancar y guardarla al apagar. Así queda `app/main.py`:

```python title="app/main.py"
import json
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import BASE_DIR, obtener_configuracion
from app.routers import tareas, usuarios

configuracion = obtener_configuracion()
ARCHIVO_DATOS = BASE_DIR / "tareas.json"


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Arranque: se ejecuta una vez, antes de atender pedidos
    if ARCHIVO_DATOS.exists():
        guardadas = json.loads(ARCHIVO_DATOS.read_text(encoding="utf-8"))
        tareas.tareas.clear()
        tareas.tareas.extend(guardadas)
        print(f"Se cargaron {len(guardadas)} tareas de {ARCHIVO_DATOS.name}")

    yield

    # Apagado: se ejecuta una vez, cuando el servidor se detiene
    ARCHIVO_DATOS.write_text(json.dumps(tareas.tareas, ensure_ascii=False, indent=2), encoding="utf-8")
    print(f"Se guardaron {len(tareas.tareas)} tareas en {ARCHIVO_DATOS.name}")


app = FastAPI(
    title=configuracion.nombre_app,
    description="Una API para organizar tareas pendientes.",
    version="0.1.0",
    debug=configuracion.debug,
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=configuracion.origenes_permitidos,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(tareas.router)
app.include_router(usuarios.router)
```

Lo nuevo:

- **`json`** (biblioteca estándar) convierte entre texto JSON y datos de Python: `json.loads(texto)` lee, `json.dumps(datos)` escribe. `indent=2` lo deja legible y `ensure_ascii=False` guarda las tildes tal cual.
- **`tareas.tareas`** es la lista `tareas` del módulo `app/routers/tareas.py`. Se vacía y se vuelve a llenar con `clear()` y `extend()`, en vez de reemplazarla, para que el router siga usando la misma lista.
- **`BASE_DIR`** se importa de `app/config.py`, así el archivo se guarda en la raíz del proyecto sin importar desde dónde ejecutes.

## Probarlo

Levantá el servidor, creá una tarea y apagalo con <kbd>Ctrl</kbd> + <kbd>C</kbd>. En la terminal vas a ver:

```text
INFO:     Shutting down
INFO:     Waiting for application shutdown.
Se guardaron 4 tareas en tareas.json
INFO:     Application shutdown complete.
```

Aparece un `tareas.json` en la raíz del proyecto. Volvé a levantar el servidor:

```text
INFO:     Waiting for application startup.
Se cargaron 4 tareas de tareas.json
INFO:     Application startup complete.
```

Y `GET /tareas` devuelve las cuatro, incluida la que creaste. Con la recarga automática pasa lo mismo: al guardar un `.py`, se ejecuta el apagado (guarda) y después el arranque (carga), así que tus datos ya no se pierden mientras programás.

Agregá `tareas.json` al `.gitignore`: son datos, no código.

!!! note "Un archivo no es una base de datos"

    Esto sirve para entender `lifespan`, pero tiene límites serios: si el servidor se corta de golpe, lo que no se guardó se pierde, y no funcionaría bien con varios servidores a la vez. En el Capítulo 3 vas a reemplazar la lista y el archivo por PostgreSQL. Y `lifespan` va a seguir ahí, para crear la conexión a la base al arrancar y cerrarla al apagar.

!!! tip "`print` por ahora"

    Usamos `print` para ver qué pasa. En el Capítulo 6 vas a cambiarlo por *logs*, la forma correcta de dejar registro en una app de verdad. En código viejo también vas a ver `@app.on_event("startup")`: es la forma anterior de hacer esto, y hoy está desaconsejada.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Express el arranque es simplemente el código que corre antes de `app.listen()`, y para el apagado se escucha la señal del sistema:

    ```javascript
    await cargarDatos()
    const server = app.listen(8000)

    process.on("SIGINT", async () => {
      await guardarDatos()
      server.close()
    })
    ```

    `lifespan` junta las dos cosas en un solo lugar, y FastAPI se encarga de llamarlas en el momento justo.

??? example "Ver comparación en Django ⬇️"

    Lo más parecido en Django es el método `ready()` de un `AppConfig`, que corre al arrancar. Para el apagado no hay un mecanismo estándar en el modo clásico; con ASGI existen los eventos *lifespan*, que son justamente lo que FastAPI expone con esta función.

??? example "Ver comparación en Spring Boot ⬇️"

    | Spring Boot | FastAPI |
    | --- | --- |
    | `@PostConstruct` o `ApplicationRunner` | Código antes del `yield` |
    | `@PreDestroy` | Código después del `yield` |

    En Spring esto se reparte entre los beans; en FastAPI hay una única función para toda la app.

## Mini ejercicio

Hacé que el nombre del archivo de datos se pueda cambiar desde la configuración, con un campo `archivo_datos: str = "tareas.json"`. Probalo poniendo `ARCHIVO_DATOS=datos-desarrollo.json` en el `.env`.

??? success "Ver solución"

    En `Configuracion` (`app/config.py`), agregá:

    ```python
    archivo_datos: str = "tareas.json"
    ```

    En `app/main.py`, cambiá la constante para que use la configuración:

    ```python
    ARCHIVO_DATOS = BASE_DIR / configuracion.archivo_datos
    ```

    Con `ARCHIVO_DATOS=datos-desarrollo.json` en el `.env`, al apagar el servidor vas a ver `Se guardaron 3 tareas en datos-desarrollo.json`, y el archivo aparece en la raíz del proyecto.

---

¡Terminaste el Capítulo 2! Tenés una API completa con validación, documentación, configuración y CORS. Lo único que le falta es una base de datos de verdad.

**Siguiente:** [Capítulo 3 · Qué es una base de datos relacional →](../base-de-datos/01-que-es-una-base-de-datos.md)
