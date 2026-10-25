# Guía de Python + FastAPI para principiantes

Este repo es una documentación paso a paso, hecha con **MkDocs + Material for MkDocs** y publicada en **GitHub Pages**.
El plan completo, con capítulos y casillas, está en `docs/_plan/PLAN.md`. Leelo antes de escribir cualquier página.

## Decisiones (no cambiar sin preguntar)

- **Idioma:** español rioplatense, con voseo ("abrí", "ejecutá", "tenés").
- **Público:** principiantes. No se asume Python previo. Sí se asume que saben usar una terminal.
- **Alcance:** genérico. No está atado a ninguna app concreta. Los ejemplos son de propósito general (tareas, usuarios, productos).
- **Stacks para comparar:** solo Express (Node), Django y Spring Boot. Ninguno más.
- **Herramientas del stack enseñado:** `uv`, FastAPI, Uvicorn, Pydantic, pydantic-settings, SQLAlchemy 2 (async) + asyncpg, Alembic, PostgreSQL, Docker Compose, pytest, Ruff. Para autenticación: PyJWT + pwdlib (argon2).

## Reglas de escritura

- **Una página = una sola idea.** Si una página pasa de unos 150-200 renglones, se parte en dos.
- Cada página arranca con un bloque corto:
  - Qué vas a aprender.
  - Qué necesitás (páginas previas).
- Explicar primero el concepto en Python o FastAPI; después las comparaciones.
- Las comparaciones van **ocultas en desplegables cerrados**, una por stack:

  ```markdown
  ??? example "Ver comparación en Express ⬇️"

      ```javascript
      app.get('/', (req, res) => res.json({ ok: true }))
      ```

  ??? example "Ver comparación en Django ⬇️"

      ...

  ??? example "Ver comparación en Spring Boot ⬇️"

      ...
  ```

  Si el contenido es solo código equivalente en los tres, se puede usar un único desplegable con pestañas adentro (`=== "Express"`).
- Cada página termina con un mini ejercicio (con la solución en un desplegable `??? success "Ver solución"`) y un enlace a la página siguiente.
- Todo código que aparezca tiene que correr de verdad. Probarlo antes de dar la página por terminada.
- Comandos con variantes por sistema: pestañas `=== "macOS / Linux"` y `=== "Windows (PowerShell)"`.

## Estructura del repo

```
mkdocs.yml
requirements.txt          # mkdocs-material
.github/workflows/deploy.yml
docs/
├── index.md
├── introduccion/
├── python/
├── fastapi/
├── base-de-datos/
├── docker/
├── autenticacion/
├── calidad-y-despliegue/
├── apendices/
└── _plan/PLAN.md          # excluido del sitio publicado
```

Nombres de archivo: `NN-tema-en-kebab.md` (por ejemplo `python/03-variables-y-tipos.md`), para que el orden se vea en el disco. La navegación se define explícitamente en `nav:` de `mkdocs.yml`.

## Extensiones de Markdown necesarias en `mkdocs.yml`

`admonition`, `pymdownx.details`, `pymdownx.superfences`, `pymdownx.tabbed` (con `alternate_style: true`), `pymdownx.highlight`, `pymdownx.inlinehilite`, `pymdownx.snippets`, `attr_list`, `md_in_html`, `toc` (con `permalink: true`).

Features del tema recomendadas: `navigation.sections`, `navigation.expand`, `navigation.footer`, `navigation.top`, `toc.follow`, `content.code.copy`, `search.suggest`, `search.highlight`. Paleta con modo claro y oscuro. `language: es`.

## Flujo de trabajo

1. Una página por vez, en el orden del plan.
2. Verificar que el ejemplo corre y que `mkdocs build --strict` no tira warnings.
3. Tildar la casilla en `docs/_plan/PLAN.md`.
4. Al cerrar un capítulo, revisar enlaces y el `nav:`.

## Comandos

```bash
uv tool install mkdocs --with mkdocs-material   # o: pip install -r requirements.txt
mkdocs serve          # vista previa en http://127.0.0.1:8000
mkdocs build --strict # lo mismo que corre el deploy
```
