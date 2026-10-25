# Servidor Uvicorn y recarga automática

!!! info "En esta página"

    **Qué vas a aprender:** qué hace el servidor Uvicorn, qué pasa por debajo de `fastapi dev`, cómo funciona la recarga automática y cómo cambiar el puerto o levantar la app en modo producción.

    **Qué necesitás:** [Primera app: `FastAPI()` y la primera ruta](02-primera-app.md).

## La app y el servidor son dos cosas distintas

En la página anterior viste que `uv run main.py` no levanta la API. Es porque hay dos piezas:

- **Tu app** (`app = FastAPI()`): sabe qué hacer con cada petición, pero no sabe escuchar la red.
- **El servidor** (**Uvicorn**): abre un **puerto** en tu computadora, espera peticiones HTTP, se las pasa a tu app como llamadas de Python y devuelve la respuesta al cliente.

Un **puerto** es un número que identifica a un programa que escucha en la red. Por eso la URL es `http://127.0.0.1:8000`: `127.0.0.1` es "esta misma computadora" (también se puede escribir `localhost`) y `8000` es el puerto de Uvicorn.

FastAPI y Uvicorn se hablan con un estándar llamado **ASGI**. No hace falta que sepas cómo funciona: alcanza con saber que cualquier servidor ASGI puede ejecutar una app de FastAPI, y Uvicorn es el más usado.

## Qué hace `fastapi dev`

`fastapi dev main.py` es un atajo: busca la variable `app` en `main.py` y arranca Uvicorn con una configuración cómoda para desarrollar. Lo mismo se puede hacer llamando a Uvicorn directamente:

```bash
uv run uvicorn main:app --reload
```

```text
INFO:     Will watch for changes in these directories: ['/Users/ana/tareas-api']
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [48913] using WatchFiles
INFO:     Application startup complete.
```

`main:app` se lee "**módulo** `main`, **variable** `app`": es el mismo formato que en un `from main import app`. Si tu app estuviera en `app/principal.py` en una variable `api`, sería `app.principal:api`.

Vas a ver este comando en muchos tutoriales y en la documentación de despliegue. En esta guía usamos `fastapi dev` por ser más corto, pero los dos hacen lo mismo.

## La recarga automática

`--reload` (que `fastapi dev` activa solo) hace que Uvicorn **vigile los archivos** del proyecto. Cada vez que guardás un cambio, reinicia la app:

```text
WARNING:  WatchFiles detected changes in 'main.py'. Reloading...
INFO:     Application startup complete.
```

Así no tenés que apagar y prender el servidor con cada modificación. Dos cosas a tener en cuenta:

- **Reiniciar borra la memoria.** Todo lo que guardaste en variables (como la lista `tareas`, si la modificaste) vuelve a su estado inicial.
- **Si guardás con un error, la app no arranca.** En la terminal aparece el traceback, y el navegador se queda esperando o muestra un error de conexión:

    ```text
      File "/Users/ana/tareas-api/main.py", line 23
        return "1.0"(
                    ^
    SyntaxError: '(' was never closed
    ```

    No hace falta reiniciar nada: corregí el archivo, guardalo y el servidor se recarga solo.

!!! tip "Mirá siempre la terminal del servidor"

    Si el navegador no responde o algo no anda, el motivo casi siempre está en la terminal donde corre el servidor. Leé el traceback de abajo hacia arriba, como aprendiste en el Capítulo 1.

## Cambiar el puerto

Si el puerto 8000 está ocupado (por ejemplo, porque te quedó otro servidor prendido en otra terminal), Uvicorn no arranca:

```text
ERROR:    [Errno 48] error while attempting to bind on address ('127.0.0.1', 8000): address already in use
```

El número del error puede variar según el sistema. Tenés dos opciones: apagar el otro servidor, o usar otro puerto:

```bash
uv run fastapi dev main.py --port 8001
```

y entrar a `http://127.0.0.1:8001`. Con Uvicorn directo, la opción es la misma: `--port 8001`.

## Modo producción: `fastapi run`

Para cuando la API se publica de verdad existe `fastapi run`:

```bash
uv run fastapi run main.py
```

```text
 ⚡️ Starting FastAPI in production mode

 🌐 Server started at http://0.0.0.0:8000
```

Las diferencias con `fastapi dev`:

| | `fastapi dev` | `fastapi run` |
| --- | --- | --- |
| Recarga automática | Sí | No |
| Escucha en | `127.0.0.1` (solo tu computadora) | `0.0.0.0` (cualquier conexión) |
| Para qué | Desarrollar | Producción y contenedores |

`0.0.0.0` significa "aceptá conexiones desde cualquier lado", no solo desde tu propia computadora. Lo vas a necesitar en el Capítulo 4, cuando la API corra dentro de Docker. Para desarrollar, usá siempre `fastapi dev`: la recarga consume recursos y no tiene sentido en un servidor real.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Node, el servidor HTTP viene incluido y Express lo prende con `app.listen(8000)`. En Python, el servidor es un programa aparte (Uvicorn) que carga tu app.

    | Node | FastAPI |
    | --- | --- |
    | `app.listen(8000)` | `uvicorn main:app --port 8000` |
    | `nodemon app.js` o `node --watch app.js` | `fastapi dev main.py` / `uvicorn --reload` |
    | `node app.js` (producción) | `fastapi run main.py` |

??? example "Ver comparación en Django ⬇️"

    `fastapi dev` cumple el rol de `python manage.py runserver`: servidor de desarrollo con recarga automática. Y así como en producción Django se ejecuta con Gunicorn o Uvicorn (no con `runserver`), FastAPI se ejecuta con `fastapi run` o Uvicorn sin `--reload`. De hecho, Uvicorn también puede servir Django en modo ASGI.

??? example "Ver comparación en Spring Boot ⬇️"

    Spring Boot trae un servidor embebido (Tomcat, por defecto) que arranca solo con la aplicación. En FastAPI, Uvicorn ocupa ese lugar, pero se ejecuta como un comando aparte.

    | Spring Boot | FastAPI |
    | --- | --- |
    | Tomcat embebido | Uvicorn |
    | `server.port=8001` | `--port 8001` |
    | Spring DevTools (reinicio automático) | `--reload` / `fastapi dev` |
    | `java -jar app.jar` | `fastapi run main.py` |

## Mini ejercicio

1. Levantá tu API con Uvicorn directamente, en el puerto `9000` y con recarga automática.
2. Con el servidor prendido, cambiá el mensaje de la ruta `/` y comprobá en `http://127.0.0.1:9000` que se actualizó sin reiniciar.

??? success "Ver solución"

    ```bash
    uv run uvicorn main:app --reload --port 9000
    ```

    Al guardar `main.py`, la terminal muestra `WatchFiles detected changes in 'main.py'. Reloading...` y, al recargar el navegador, aparece el mensaje nuevo. El orden de las opciones (`--reload`, `--port`) no importa.

---

**Siguiente:** [Rutas con métodos HTTP →](04-metodos-http.md)
