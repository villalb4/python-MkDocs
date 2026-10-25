# Rutas de archivos con `pathlib`

!!! info "En esta página"

    **Qué vas a aprender:** cómo armar rutas de archivos que funcionen en cualquier sistema operativo, cómo ubicar archivos a partir de `__file__` y cómo leer y escribir archivos de texto.

    **Qué necesitás:** [`__name__` y `__file__`](13-name-y-file.md).

## Por qué no armar rutas a mano

Una ruta es un texto como `datos/tareas.txt`. Podrías armarla uniendo strings, pero tiene problemas:

- macOS y Linux separan carpetas con `/`; Windows, con `\`.
- Es fácil olvidarse o duplicar una barra: `"datos" + "tareas.txt"` da `datostareas.txt`.

El módulo `pathlib`, de la biblioteca estándar, resuelve esto con el tipo `Path`.

## Crear y unir rutas

```python title="rutas.py"
from pathlib import Path

carpeta = Path("datos")
archivo = carpeta / "tareas.txt"
print(archivo)  # datos/tareas.txt
```

El operador `/` **une** partes de una ruta. No es una división: con `Path`, la barra significa "entrar en". Además usa el separador correcto según el sistema: en Windows, ese mismo código muestra `datos\tareas.txt`.

## Partes de una ruta

```python
ruta = Path("datos/reportes/ventas-2026.csv")

print(ruta.name)  # ventas-2026.csv
print(ruta.stem)  # ventas-2026
print(ruta.suffix)  # .csv
print(ruta.parent)  # datos/reportes
print(ruta.parent.parent)  # datos
```

| Atributo | Qué devuelve |
| --- | --- |
| `.name` | El nombre del archivo con extensión. |
| `.stem` | El nombre sin la extensión. |
| `.suffix` | La extensión, con el punto. |
| `.parent` | La carpeta que lo contiene (otro `Path`). |

## Ubicar archivos a partir de `__file__`

En la página anterior viste que una ruta relativa como `datos/tareas.txt` se busca desde la **carpeta de trabajo**, que cambia según desde dónde ejecutes. La solución es partir de `__file__`, que siempre apunta al archivo de código:

```text
proyecto/
├── datos/
│   └── tareas.txt
└── scripts/
    └── rutas.py
```

```python title="scripts/rutas.py"
from pathlib import Path

ESTE_ARCHIVO = Path(__file__).resolve()
CARPETA_SCRIPTS = ESTE_ARCHIVO.parent
BASE_DIR = CARPETA_SCRIPTS.parent

print(ESTE_ARCHIVO)  # /Users/ana/proyecto/scripts/rutas.py
print(CARPETA_SCRIPTS)  # /Users/ana/proyecto/scripts
print(BASE_DIR)  # /Users/ana/proyecto
print(BASE_DIR / "datos" / "tareas.txt")  # /Users/ana/proyecto/datos/tareas.txt
```

- `Path(__file__)` convierte la ruta del archivo en un `Path`.
- `.resolve()` la vuelve **absoluta** (completa, desde la raíz del disco) y limpia cosas como `..`.
- Con `.parent` subís de carpeta tantas veces como haga falta.

Ejecutá esto desde `proyecto/` o desde `proyecto/scripts/`: el resultado es el mismo. Ese es todo el truco.

!!! tip "El patrón `BASE_DIR`"

    Definir una constante `BASE_DIR` con la carpeta raíz del proyecto, y armar todas las rutas a partir de ella, es una costumbre muy extendida. La vas a usar en el Capítulo 2 para encontrar el archivo `.env`.

Si necesitás la carpeta de trabajo, la versión de `pathlib` de `os.getcwd()` es `Path.cwd()`.

## Leer y escribir archivos de texto

`Path` también sabe trabajar con el archivo al que apunta:

```python title="archivos.py"
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent
carpeta = BASE_DIR / "datos"
archivo = carpeta / "notas.txt"

print(carpeta.exists())  # False (la primera vez)

carpeta.mkdir(parents=True, exist_ok=True)
archivo.write_text("Comprar pan\nEstudiar Python\n", encoding="utf-8")

print(archivo.exists())  # True
print(archivo.is_file())  # True

contenido = archivo.read_text(encoding="utf-8")
print(contenido.splitlines())  # ['Comprar pan', 'Estudiar Python']
```

| Método | Qué hace |
| --- | --- |
| `.exists()` | `True` si la ruta existe. |
| `.is_file()` / `.is_dir()` | `True` si es un archivo / una carpeta. |
| `.mkdir(parents=True, exist_ok=True)` | Crea la carpeta (y las intermedias); no falla si ya existe. |
| `.write_text(texto, encoding="utf-8")` | Escribe el texto en el archivo (si existía, lo **reemplaza**). |
| `.read_text(encoding="utf-8")` | Devuelve todo el contenido como un string. |
| `.glob("*.txt")` | Recorre los archivos de la carpeta que coinciden con el patrón. |

!!! warning "Indicá siempre el encoding"

    Sin él, Python usa la codificación por defecto del sistema, que en Windows puede no ser UTF-8, y las tildes y la ñ se rompen. Ponerlo siempre te ahorra el problema.

Si el archivo no existe, `read_text` da `FileNotFoundError: [Errno 2] No such file or directory: 'no-existe.txt'`. En la página de manejo de errores vas a ver cómo atraparlo.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    | Node | Python |
    | --- | --- |
    | `path.join(dir, "datos", "x.txt")` | `dir / "datos" / "x.txt"` |
    | `path.basename(p)` | `p.name` |
    | `path.extname(p)` | `p.suffix` |
    | `path.dirname(p)` | `p.parent` |
    | `path.resolve(__dirname, "..")` | `Path(__file__).resolve().parent.parent` |
    | `fs.readFileSync(p, "utf8")` | `p.read_text(encoding="utf-8")` |
    | `fs.existsSync(p)` | `p.exists()` |

    En Node las rutas son strings y las funciones están sueltas en `path` y `fs`; en Python un `Path` es un objeto que trae todo eso como métodos.

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. Ahora podés leer completa la línea de `settings.py`:

    ```python
    BASE_DIR = Path(__file__).resolve().parent.parent
    ```

    `settings.py` está en `proyecto/proyecto/settings.py`: un `.parent` llega a la carpeta interna y el segundo, a la raíz. Después se usa en cosas como `BASE_DIR / "db.sqlite3"`.

??? example "Ver comparación en Spring Boot ⬇️"

    `pathlib.Path` es casi idéntico a `java.nio.file.Path`:

    | Java | Python |
    | --- | --- |
    | `Path.of("datos", "x.txt")` | `Path("datos") / "x.txt"` |
    | `p.getFileName()` | `p.name` |
    | `p.getParent()` | `p.parent` |
    | `p.toAbsolutePath().normalize()` | `p.resolve()` |
    | `Files.readString(p)` | `p.read_text(encoding="utf-8")` |
    | `Files.exists(p)` | `p.exists()` |

    En Spring Boot los archivos de configuración se buscan en el *classpath*; en Python se ubican con rutas a partir de `__file__`.

## Mini ejercicio

Tenés esta estructura. En `app/config.py`, definí `BASE_DIR` apuntando a `proyecto/` y `ARCHIVO_ENV` apuntando a `proyecto/.env`, de forma que funcione sin importar desde dónde se ejecute.

```text
proyecto/
├── .env
└── app/
    └── config.py
```

??? success "Ver solución"

    ```python title="app/config.py"
    from pathlib import Path

    BASE_DIR = Path(__file__).resolve().parent.parent
    ARCHIVO_ENV = BASE_DIR / ".env"
    ```

    `Path(__file__).resolve()` es `proyecto/app/config.py`; el primer `.parent` es `proyecto/app` y el segundo, `proyecto`. Desde ahí, `/ ".env"` arma la ruta al archivo.

---

**Siguiente:** [Variables de entorno y archivos `.env` →](15-variables-de-entorno.md)
