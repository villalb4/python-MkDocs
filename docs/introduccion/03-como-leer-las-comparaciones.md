# Cómo leer las comparaciones

!!! info "En esta página"

    **Qué vas a aprender:** cómo funcionan los desplegables y las pestañas que vas a ver en toda la guía.

    **Qué necesitás:** [Qué necesitás instalar](02-que-necesitas-instalar.md).

## Primero Python, después las comparaciones

Cada página explica el concepto en Python o FastAPI. Si ya programaste en **Express** (Node), **Django** o **Spring Boot**, al final de la explicación vas a encontrar desplegables cerrados con el equivalente en esos stacks.

Si nunca usaste ninguno, **no los abras**: no te hacen falta para seguir.

## Un ejemplo de verdad

Esta es una API mínima en FastAPI que responde `{"mensaje": "hola"}` cuando visitás `/`:

```python title="main.py"
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def saludar():
    return {"mensaje": "hola"}
```

Todavía no tenés que entender cada línea; eso llega en el Capítulo 2. Lo importante ahora es ver cómo se presentan las comparaciones.

### Desplegables, uno por stack

Cuando la comparación necesita explicación, hay un desplegable por stack. Hacé clic en el título para abrirlo:

??? example "Ver comparación en Express ⬇️"

    En Express también se registra una función por ruta y método:

    ```javascript
    const express = require('express')
    const app = express()

    app.get('/', (req, res) => res.json({ mensaje: 'hola' }))

    app.listen(8000)
    ```

    La diferencia: en Express llamás a `res.json(...)`; en FastAPI simplemente **devolvés** un diccionario y FastAPI lo convierte en JSON.

??? example "Ver comparación en Django ⬇️"

    En Django la función (la *view*) y la ruta se definen en archivos separados:

    ```python
    # views.py
    from django.http import JsonResponse

    def saludar(request):
        return JsonResponse({"mensaje": "hola"})

    # urls.py
    from django.urls import path
    from . import views

    urlpatterns = [path("", views.saludar)]
    ```

    En FastAPI la ruta va pegada a la función con `@app.get("/")`, en el mismo lugar.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring Boot la ruta también va pegada al método, con una anotación:

    ```java
    @RestController
    public class SaludoController {

        @GetMapping("/")
        public Map<String, String> saludar() {
            return Map.of("mensaje", "hola");
        }
    }
    ```

    `@app.get("/")` en FastAPI cumple el mismo rol que `@GetMapping("/")`, y en los dos casos devolvés un mapa que se convierte en JSON.

### Pestañas, cuando es solo código

Si la comparación es solo código equivalente, sin mucho que explicar, va en un único desplegable con pestañas adentro:

??? example "Ver la misma ruta en otros stacks ⬇️"

    === "Express"

        ```javascript
        app.get('/', (req, res) => res.json({ mensaje: 'hola' }))
        ```

    === "Django"

        ```python
        def saludar(request):
            return JsonResponse({"mensaje": "hola"})
        ```

    === "Spring Boot"

        ```java
        @GetMapping("/")
        public Map<String, String> saludar() {
            return Map.of("mensaje", "hola");
        }
        ```

## Otras cajas que vas a ver

!!! note "Nota"

    Información extra que ayuda pero no es imprescindible.

!!! warning "Cuidado"

    Algo que suele salir mal o que conviene no hacer.

=== "macOS / Linux"

    Las pestañas por sistema operativo aparecen cuando un comando cambia según dónde lo ejecutes.

=== "Windows (PowerShell)"

    Elegí la tuya y las demás pestañas de la página cambian solas a la misma opción.

## Mini ejercicio

En la API de ejemplo, ¿qué línea define que la función responde en la ruta `/`? ¿Qué devuelve la función?

??? success "Ver solución"

    La línea `@app.get("/")`, que está justo arriba de la función. Le dice a FastAPI: "cuando llegue un `GET` a `/`, ejecutá esta función".

    La función devuelve el diccionario `{"mensaje": "hola"}`, y FastAPI lo envía al navegador como JSON.

---

**Siguiente:** [Qué es Python y cómo se ejecuta un script →](../python/01-que-es-python.md)
