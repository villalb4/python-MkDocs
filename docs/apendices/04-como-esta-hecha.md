# Cómo está hecha esta guía

!!! info "En esta página"

    **Qué vas a aprender:** con qué herramientas está hecha esta guía, cómo se escriben sus bloques (avisos, desplegables, pestañas), cómo verla en tu máquina y cómo se publica en GitHub Pages. Te sirve para armar la documentación de tu propia API.

    **Qué necesitás:** saber usar `uv` ([Instalar y usar `uv`](../python/02-instalar-uv.md)). Para publicar, una cuenta de GitHub.

## Las piezas

- **Markdown:** cada página es un archivo `.md`, texto con marcas simples (`#` para títulos, `**negrita**`, listas con `-`).
- **MkDocs:** un programa en Python que convierte esos archivos en un sitio web.
- **Material for MkDocs:** el tema que le da el aspecto, el buscador, el modo oscuro y los bloques especiales.
- **GitHub Pages:** el servicio de GitHub que publica el sitio, gratis para repositorios públicos.
- **GitHub Actions:** construye y publica el sitio solo, cada vez que se suben cambios.

## Los archivos

```text
mkdocs.yml                     # configuración del sitio
requirements.txt               # mkdocs-material (para el servidor de GitHub)
.github/workflows/deploy.yml   # cómo se construye y publica
docs/
├── index.md                   # la portada
├── python/
│   ├── 01-que-es-python.md
│   └── ...
├── ...
└── _plan/                     # notas de trabajo, que no se publican
```

Los archivos llevan un número adelante (`01-`, `02-`) para que el orden se vea en el disco. El orden del sitio, en cambio, lo define la sección `nav:` de `mkdocs.yml`.

## `mkdocs.yml`

Estas son las partes principales del de esta guía:

```yaml title="mkdocs.yml (partes)"
site_name: Python + FastAPI desde cero

# El plan de trabajo vive en docs/_plan/ pero no se publica.
exclude_docs: |
  _plan/

theme:
  name: material
  language: es

markdown_extensions:
  - admonition
  - pymdownx.details
  - pymdownx.superfences
  - pymdownx.tabbed:
      alternate_style: true

nav:
  - Inicio: index.md
  - "0 · Introducción":
      - Qué vas a aprender: introduccion/01-que-vas-a-aprender.md
```

- **`exclude_docs`** deja afuera carpetas que están en `docs/` pero no tienen que publicarse.
- **`theme`** elige Material, en español. El archivo completo agrega la paleta de colores (clara y oscura) y algunas funciones del tema, como el botón para copiar el código.
- **`markdown_extensions`** activa los bloques especiales de la sección siguiente. La guía usa unas cuantas más, como el resaltado del código.
- **`nav`** es el menú: cada entrada es un título y su archivo.

## Los bloques de la guía

Un **aviso**, como el "En esta página" de arriba, empieza con `!!!`, el tipo y un título. El contenido va con cuatro espacios de sangría:

```markdown
!!! info "En esta página"

    **Qué vas a aprender:** cómo se ven los bloques de la guía.
```

Un **desplegable** cerrado, como el de las comparaciones, es igual, pero con `???`:

````markdown
??? example "Ver comparación en Express ⬇️"

    ```javascript
    app.get('/', (req, res) => res.json({ ok: true }))
    ```
````

Las **pestañas** se escriben con `===`, una por pestaña:

````markdown
=== "macOS / Linux"

    ```bash
    source .venv/bin/activate
    ```

=== "Windows (PowerShell)"

    ```powershell
    .venv\Scripts\Activate.ps1
    ```
````

Y las teclas, como <kbd>Ctrl</kbd> + <kbd>C</kbd>, se escriben con HTML: `<kbd>Ctrl</kbd>`.

## Verla en tu máquina

MkDocs se instala como herramienta con `uv`, con Material incluido:

```bash
uv tool install mkdocs --with mkdocs-material
```

Desde la carpeta donde está `mkdocs.yml`:

```bash
mkdocs serve
```

```text
INFO    -  [16:56:28] Watching paths for changes: 'docs', 'mkdocs.yml'
INFO    -  [16:56:28] Serving on http://127.0.0.1:8000/
```

Abrí `http://127.0.0.1:8000`: cada vez que guardás un archivo, la página se actualiza sola, como la recarga de `fastapi dev`.

Para generar el sitio final, en la carpeta `site/` (que no se sube a Git):

```bash
mkdocs build --strict
```

Con `--strict`, cualquier aviso corta el build. Por ejemplo, un enlace a una página que no existe:

```text
WARNING -  Doc file '01-primera-pagina.md' contains a link '02-segunda-pagina.md', but the target is not found among documentation files.

Aborted with 1 warnings in strict mode!
```

Sin `--strict`, el mismo enlace roto da el aviso pero el sitio se genera igual, y el error llega publicado. Por eso el servidor de GitHub construye siempre con `--strict`.

## Publicarla en GitHub Pages

El archivo `.github/workflows/deploy.yml` le dice a GitHub Actions qué hacer en cada `push` a `main`:

1. Instalar Python 3.12 y lo que dice `requirements.txt` (`pip install -r requirements.txt`).
2. Construir el sitio con `mkdocs build --strict`. Si hay un aviso, se corta acá y no se publica nada.
3. Subir la carpeta `site/` y publicarla en GitHub Pages.

Para activarlo, una sola vez, en el repositorio de GitHub: **Settings → Pages → Source: GitHub Actions**. Después de cada `push`, la pestaña **Actions** muestra si el build anduvo, y el sitio queda en `https://<usuario>.github.io/<repositorio>/`.

## El futuro: Zensical

Al construir, Material avisa que **llega al fin de su vida útil el 5 de mayo de 2027**: hasta esa fecha recibe arreglos críticos y de seguridad, y después no. Su sucesor es **Zensical**, hecho por el mismo equipo, que se puede probar con el mismo `mkdocs.yml`. El aviso se calla con la variable `NO_MKDOCS_2_WARNING=1`; no es un error del build, y `--strict` no lo cuenta.

Para esta guía no cambia nada todavía, pero antes de esa fecha conviene probar la migración.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En el mundo de JavaScript, el equivalente más usado es **Docusaurus** (de Meta) o **VitePress**: también arman un sitio a partir de archivos Markdown, con un archivo de configuración y el menú lateral. Se publican igual, con GitHub Actions y GitHub Pages.

??? example "Ver comparación en Django ⬇️"

    La documentación de Django (y de Python mismo) está hecha con **Sphinx**, que usa reStructuredText (o Markdown con un plugin) y suele publicarse en **Read the Docs**. MkDocs es más simple de configurar, por eso muchos proyectos nuevos, como FastAPI, lo eligieron.

??? example "Ver comparación en Spring Boot ⬇️"

    En Java, la documentación de proyectos suele escribirse en **AsciiDoc** con Asciidoctor, que es lo que usa la propia documentación de Spring. Para documentar la API en sí, Spring REST Docs genera fragmentos a partir de los tests; en FastAPI eso lo cubre `/docs`.

## Mini ejercicio

En tu sitio, creás `docs/02-segunda-pagina.md` pero te olvidás de agregarla al `nav:`. ¿`mkdocs build --strict` falla? ¿La página se publica?

??? success "Ver solución"

    No falla: el build termina bien, solo con un mensaje informativo (no un aviso):

    ```text
    INFO    -  The following pages exist in the docs directory, but are not included in the "nav" configuration:
      - 02-segunda-pagina.md
    ```

    Y la página **sí** se publica, en `/02-segunda-pagina/`, pero no aparece en el menú: nadie la va a encontrar si no tiene el enlace. Por eso, al agregar una página, hay que acordarse de las dos cosas: el archivo y su entrada en `nav:`.

---

¡Terminaste la guía! Empezaste sin saber Python y llegaste a una API con base de datos, autenticación, tests y lista para publicar. Lo que sigue es practicar: armá tu propia API, con un tema que te interese, y volvé a estas páginas cada vez que lo necesites.

**Volver al inicio:** [Python + FastAPI desde cero](../index.md)
