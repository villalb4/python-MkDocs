# `async` / `await`

!!! info "En esta página"

    **Qué vas a aprender:** la idea detrás de `async` y `await`, por qué hacen que una API atienda más pedidos, y las tres reglas que necesitás para usarlos en FastAPI. Sin profundizar: solo la intuición.

    **Qué necesitás:** [Type hints](18-type-hints.md).

## El problema: esperar sin hacer nada

Gran parte del tiempo, un backend **espera**: a que responda la base de datos, a que llegue una respuesta de otro servicio, a que se lea un archivo. Mientras espera, el procesador no hace nada útil.

Pensá en un mozo de restaurante. Uno malo toma el pedido de la mesa 1, va a la cocina y **se queda parado** al lado hasta que el plato está listo; recién después atiende la mesa 2. Uno bueno toma el pedido de la mesa 1, lo deja en la cocina y, **mientras se cocina**, atiende la mesa 2 y la 3. Cuando un plato está listo, lo lleva.

`async` y `await` le permiten a Python ser el mozo bueno: mientras una tarea espera, avanza con otra.

## La versión que espera

Este script simula tres consultas a una base de datos, cada una tarda 1 segundo (`time.sleep(1)` frena el programa durante 1 segundo):

```python title="sincronico.py"
import time


def buscar(dato):
    print(f"Buscando {dato}...")
    time.sleep(1)  # simula esperar a la base de datos
    print(f"Listo {dato}")
    return dato.upper()


inicio = time.perf_counter()
resultados = [buscar("usuario"), buscar("tareas"), buscar("permisos")]
print(resultados)
print(f"Tardó {time.perf_counter() - inicio:.1f} segundos")
```

```text
Buscando usuario...
Listo usuario
Buscando tareas...
Listo tareas
Buscando permisos...
Listo permisos
['USUARIO', 'TAREAS', 'PERMISOS']
Tardó 3.0 segundos
```

Una consulta por vez: 3 segundos en total, casi todo esperando. (`time.perf_counter()` es un cronómetro: restando dos lecturas sabés cuánto pasó.)

## La versión asincrónica

```python title="asincronico.py"
import asyncio
import time


async def buscar(dato):
    print(f"Buscando {dato}...")
    await asyncio.sleep(1)  # simula esperar a la base de datos
    print(f"Listo {dato}")
    return dato.upper()


async def main():
    inicio = time.perf_counter()
    resultados = await asyncio.gather(buscar("usuario"), buscar("tareas"), buscar("permisos"))
    print(resultados)
    print(f"Tardó {time.perf_counter() - inicio:.1f} segundos")


asyncio.run(main())
```

```text
Buscando usuario...
Buscando tareas...
Buscando permisos...
Listo usuario
Listo tareas
Listo permisos
['USUARIO', 'TAREAS', 'PERMISOS']
Tardó 1.0 segundos
```

Las tres esperas ocurrieron **al mismo tiempo**: 1 segundo en total. Qué cambió:

- `async def` define una función **asincrónica**: una que puede pausarse mientras espera.
- `await` marca el punto de espera: "acá esperá el resultado, y mientras tanto dejá que avancen otras tareas".
- `asyncio.sleep` es la versión asincrónica de `time.sleep`: espera sin frenar todo el programa.
- `asyncio.gather(...)` lanza varias tareas juntas y espera a que terminen todas.
- `asyncio.run(main())` arranca todo. **En FastAPI no lo vas a escribir**: FastAPI lo hace por vos.

!!! note "No es más rápido, es esperar mejor"

    `async` no hace que una cuenta se calcule más rápido. Sirve cuando el tiempo se va en **esperar**. En una API eso es lo normal: mientras la consulta de un usuario espera a la base de datos, el servidor ya está atendiendo el pedido de otro.

## Las tres reglas que necesitás

**1. Para obtener el resultado de una función `async`, usá `await`.**

```python
async def buscar_usuario(usuario_id):
    await asyncio.sleep(0.5)
    return {"id": usuario_id, "nombre": "Ana"}


async def main():
    usuario = await buscar_usuario(1)
    print(usuario)  # {'id': 1, 'nombre': 'Ana'}
```

Si te olvidás el `await`, no obtenés el diccionario sino un objeto que representa "una tarea que nunca se ejecutó", y Python te avisa:

```text
RuntimeWarning: coroutine 'buscar_usuario' was never awaited
<coroutine object buscar_usuario at 0x106628c40>
```

**2. `await` solo se puede usar dentro de una función `async def`.**

En una función común da `SyntaxError: 'await' outside async function`.

**3. Dentro de una función `async`, no uses cosas que frenan todo.**

`time.sleep()` o una librería que no es asincrónica bloquean el programa entero, y el mozo vuelve a quedarse parado en la cocina. Por eso en esta guía vas a usar versiones asincrónicas: `asyncpg` para PostgreSQL y SQLAlchemy en modo async.

## Cómo se ve en FastAPI

Vas a escribir rutas así:

```python
@app.get("/tareas/{tarea_id}")
async def obtener_tarea(tarea_id: int):
    tarea = await buscar_tarea_en_la_base(tarea_id)
    return tarea
```

La ruta es `async def` y usa `await` para esperar a la base de datos. FastAPI se encarga de ejecutarla y de atender otros pedidos mientras tanto. (El `@app.get` de arriba es un **decorador**, el tema de la próxima página.)

FastAPI también acepta rutas con `def` común, y las ejecuta de forma que no bloqueen al resto. Usá `async def` cuando adentro hagas `await`.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Es casi idéntico a JavaScript:

    | JavaScript | Python |
    | --- | --- |
    | `async function buscar() { ... }` | `async def buscar():` |
    | `await buscar()` | `await buscar()` |
    | `Promise.all([a(), b()])` | `asyncio.gather(a(), b())` |
    | `new Promise(r => setTimeout(r, 1000))` | `asyncio.sleep(1)` |
    | Una `Promise` sin `await` | Una *coroutine* sin `await` |

    Una diferencia: en Node todo el ecosistema es asincrónico desde siempre. En Python conviven librerías sincrónicas y asincrónicas, y tenés que elegir las asincrónicas cuando trabajás con `async`.

??? example "Ver comparación en Django ⬇️"

    Django es sincrónico en su origen: cada pedido ocupa un *worker* hasta terminar. Las versiones recientes aceptan views `async def`, pero el ORM clásico es sincrónico. FastAPI, en cambio, se diseñó desde el principio para trabajar con `async`.

??? example "Ver comparación en Spring Boot ⬇️"

    Spring MVC tradicional usa un hilo por pedido, que queda bloqueado mientras espera a la base de datos. La alternativa asincrónica es Spring WebFlux, con `Mono` y `Flux`. `async`/`await` en Python logra lo mismo que WebFlux, pero el código se lee de arriba hacia abajo como si fuera sincrónico, sin encadenar `.map()` ni `.flatMap()`.

## Mini ejercicio

¿En qué orden se muestran los mensajes? ¿Y la lista final?

```python title="ejercicio.py"
import asyncio


async def preparar(plato, minutos):
    await asyncio.sleep(minutos / 10)
    print(f"{plato} listo")
    return plato


async def main():
    platos = await asyncio.gather(preparar("Pizza", 3), preparar("Ensalada", 1), preparar("Sopa", 2))
    print(platos)


asyncio.run(main())
```

??? success "Ver solución"

    ```text
    Ensalada listo
    Sopa listo
    Pizza listo
    ['Pizza', 'Ensalada', 'Sopa']
    ```

    Los tres platos se preparan a la vez, así que termina primero el que menos espera (Ensalada, 0,1 segundos) y último la Pizza (0,3). Pero `gather` devuelve los resultados **en el orden en que los pediste**, no en el que terminaron.

---

**Siguiente:** [Decoradores →](20-decoradores.md)
