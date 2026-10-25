# Qué vas a aprender y en qué orden

!!! info "En esta página"

    **Qué vas a aprender:** el recorrido completo de la guía y por qué está ordenado así.

    **Qué necesitás:** nada. Es la primera página.

## La idea general

Un **backend** es el programa que corre en un servidor, recibe pedidos (por ejemplo, "dame la lista de tareas") y devuelve respuestas, casi siempre en formato JSON. Para armar uno necesitás varias piezas, y cada una se apoya en la anterior:

1. **Python**: el lenguaje. Sin él no se entiende nada de lo que sigue.
2. **FastAPI**: la librería que convierte funciones de Python en rutas de una API.
3. **Base de datos**: dónde se guardan los datos para que no se pierdan al reiniciar.
4. **Docker**: una forma de levantar la base de datos (y la API) igual en cualquier máquina.
5. **Autenticación**: saber quién hace cada pedido y qué tiene permitido.
6. **Calidad y despliegue**: tests, orden en el código y publicar la API.

Por eso la guía va en ese orden. Saltear capítulos se puede, pero vas a encontrar palabras que se explicaron antes.

## Qué herramientas vas a usar

No hace falta que sepas qué es cada una todavía; las vas a ir conociendo de a una.

| Herramienta | Para qué sirve |
| --- | --- |
| `uv` | Instala Python y las librerías de cada proyecto. |
| FastAPI | Define las rutas de la API. |
| Uvicorn | El servidor que ejecuta la app de FastAPI. |
| Pydantic | Valida que los datos que llegan tengan la forma correcta. |
| SQLAlchemy | Habla con la base de datos desde Python. |
| Alembic | Lleva el historial de cambios de la base de datos. |
| PostgreSQL | La base de datos. |
| Docker Compose | Levanta varios servicios (API y base) con un solo comando. |
| pytest | Ejecuta los tests. |
| Ruff | Revisa y formatea el código. |

## Cómo conviene leer cada página

- Leé el bloque del principio: te dice si te falta alguna página previa.
- Escribí el código a mano en vez de copiarlo. Se aprende mucho más.
- Hacé el mini ejercicio del final antes de mirar la solución.

## Mini ejercicio

Sin mirar arriba: ¿en qué orden van estas piezas y por qué? *Docker, Python, Autenticación, FastAPI, Base de datos.*

??? success "Ver solución"

    **Python → FastAPI → Base de datos → Docker → Autenticación.**

    FastAPI está escrito en Python, así que primero va el lenguaje. Con FastAPI ya podés tener una API, pero los datos se pierden al reiniciar: por eso sigue la base de datos. Docker sirve para levantar esa base fácilmente. La autenticación necesita usuarios guardados en la base, así que va después.

---

**Siguiente:** [Qué necesitás instalar →](02-que-necesitas-instalar.md)
