# CORS (qué es y por qué aparece)

!!! info "En esta página"

    **Qué vas a aprender:** qué es CORS, por qué tu API funciona con `curl` pero falla cuando la llama una página web, y cómo habilitar los orígenes que correspondan con `CORSMiddleware`.

    **Qué necesitás:** [Configuración con `pydantic-settings`](12-pydantic-settings.md).

## El error que tarde o temprano vas a ver

Tu API funciona perfecto con `curl` y desde `/docs`. Pero el día que un frontend (una página hecha con React, Vue o JavaScript común) intente usarla, no va a mostrar nada, y en la consola del navegador va a aparecer esto:

```text
Access to fetch at 'http://127.0.0.1:8000/tareas' from origin 'http://localhost:5500'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present
on the requested resource.
```

Para entender por qué, hace falta una idea: el **origen**.

## Qué es un origen

El **origen** de una página es la combinación de tres cosas de su URL: el protocolo, el dominio y el puerto.

| URL | Origen |
| --- | --- |
| `http://localhost:5500/index.html` | `http://localhost:5500` |
| `http://127.0.0.1:8000/tareas` | `http://127.0.0.1:8000` |
| `https://mitienda.com/productos` | `https://mitienda.com` (puerto 443 implícito) |

Si cambia **cualquiera** de las tres, es otro origen. Ojo: `http://localhost:5500` y `http://127.0.0.1:5500` apuntan a la misma computadora, pero para el navegador son orígenes distintos.

## Por qué el navegador bloquea

Por seguridad, los navegadores aplican la **política del mismo origen**: el JavaScript de una página solo puede leer respuestas de **su propio origen**. Si no existiera, cualquier sitio que visites podría pedirle datos a tu banco o a tu correo usando tu sesión abierta, y leerlos.

**CORS** (*Cross-Origin Resource Sharing*) es la forma de hacer una excepción controlada: el **servidor** le avisa al navegador, con encabezados en la respuesta, qué otros orígenes tienen permiso. El encabezado clave es `Access-Control-Allow-Origin`. Si la respuesta no lo trae (o trae otro origen), el navegador **no le entrega** la respuesta al JavaScript de la página.

Tres cosas que confunden al principio:

- **Lo aplica el navegador, no tu API.** Por eso `curl`, Python o una app de celular no tienen este problema. Y `/docs` tampoco, porque está servida desde el mismo origen que la API.
- **El pedido sí llega al servidor.** En la terminal vas a ver `"GET /tareas HTTP/1.1" 200 OK`. Lo que se bloquea es la lectura de la respuesta en el navegador.
- **Se arregla en el servidor**, no en el frontend.

### El *preflight*

Para pedidos que pueden modificar algo (como un `POST` con JSON), el navegador primero manda una petición de consulta con el método `OPTIONS`, llamada ***preflight***: "¿puedo mandarte un `POST` con `Content-Type: application/json` desde este origen?". Solo si el servidor responde que sí, manda el pedido real. `CORSMiddleware` responde estas consultas por vos.

## Habilitar CORS en FastAPI

FastAPI trae un **middleware** para esto. Un middleware es código que se ejecuta en **todos** los pedidos, antes y después de tus rutas; este agrega los encabezados de CORS a cada respuesta.

Primero, la lista de orígenes permitidos va en la configuración, no en el código, porque cambia según el entorno (en desarrollo, `localhost`; en producción, el dominio de tu frontend). Agregá el campo a `Configuracion` en `app/config.py`:

```python
origenes_permitidos: list[str] = []
```

En el `.env`, una lista se escribe en formato JSON:

```bash title=".env"
ORIGENES_PERMITIDOS=["http://localhost:5500"]
```

Y en `app/main.py`, agregá el middleware:

```python title="app/main.py"
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import obtener_configuracion
from app.routers import tareas, usuarios

configuracion = obtener_configuracion()

app = FastAPI(
    title=configuracion.nombre_app,
    description="Una API para organizar tareas pendientes.",
    version="0.1.0",
    debug=configuracion.debug,
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

- **`allow_origins`**: la lista de orígenes que pueden llamar a la API desde un navegador.
- **`allow_methods=["*"]`**: permite todos los métodos (`GET`, `POST`, etc.).
- **`allow_headers=["*"]`**: permite cualquier encabezado, como `Content-Type` o tu `X-Clave`.

Como cambiaste el `.env`, reiniciá el servidor (o guardá un `.py`). Ahora la respuesta trae el permiso:

```bash
curl -i -H "Origin: http://localhost:5500" http://127.0.0.1:8000/tareas
```

```http
HTTP/1.1 200 OK
access-control-allow-origin: http://localhost:5500
```

Con `-H "Origin: ..."` simulás lo que manda el navegador. Si probás con un origen que no está en la lista, la respuesta llega igual pero **sin** ese encabezado, y un navegador la bloquearía.

!!! warning "No uses `allow_origins=["*"]` a la ligera"

    `"*"` permite **cualquier** origen. Es cómodo para probar, pero en producción poné solo los orígenes reales de tu frontend. Y si más adelante usás cookies (`allow_credentials=True`), `"*"` directamente no está permitido.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Express se usa el paquete `cors`, también como middleware:

    ```javascript
    const cors = require("cors")
    app.use(cors({ origin: ["http://localhost:5500"] }))
    ```

    Es exactamente la misma idea que `CORSMiddleware` con `allow_origins`.

??? example "Ver comparación en Django ⬇️"

    En Django se instala `django-cors-headers`, se agrega `CorsMiddleware` a `MIDDLEWARE` y se configura `CORS_ALLOWED_ORIGINS = ["http://localhost:5500"]` en `settings.py`. En FastAPI viene incluido, y la lista sale de tu configuración.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring se usa `@CrossOrigin(origins = "http://localhost:5500")` en un controller, o una configuración global con `WebMvcConfigurer` y `addCorsMappings`. `app.add_middleware(CORSMiddleware, ...)` equivale a la configuración global.

## Mini ejercicio

Tenés un frontend que se abre en `http://127.0.0.1:5500` y otro en `http://localhost:5500`. Con la configuración de esta página, solo el segundo funciona. Hacé que funcionen los dos y comprobalo con `curl`.

??? success "Ver solución"

    En el `.env`, agregá el segundo origen a la lista:

    ```bash title=".env"
    ORIGENES_PERMITIDOS=["http://localhost:5500", "http://127.0.0.1:5500"]
    ```

    Reiniciá el servidor y probá:

    ```bash
    curl -i -H "Origin: http://127.0.0.1:5500" http://127.0.0.1:8000/tareas
    ```

    La respuesta ahora trae `access-control-allow-origin: http://127.0.0.1:5500`. No hizo falta tocar el código: por eso conviene tener los orígenes en la configuración.

---

**Siguiente:** [`lifespan`: arranque y apagado de la app →](14-lifespan.md)
