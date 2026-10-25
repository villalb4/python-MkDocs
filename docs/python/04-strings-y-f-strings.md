# Strings y f-strings

!!! info "En esta página"

    **Qué vas a aprender:** cómo escribir textos (`str`), cómo meter variables adentro con f-strings y los métodos más útiles para limpiarlos.

    **Qué necesitás:** [Variables y tipos básicos](03-variables-y-tipos.md).

## Comillas simples, dobles y triples

Un string es un texto entre comillas. Simples o dobles da igual; elegí unas y usalas siempre. En esta guía usamos dobles.

```python title="comillas.py"
saludo = "Hola"
otro = 'Hola'
print(saludo == otro)  # True

frase = "Es un lugar 'tranquilo'"
print(frase)  # Es un lugar 'tranquilo'

mensaje = """Primera línea
Segunda línea"""
print(mensaje)
```

- Si el texto tiene comillas simples adentro, envolvelo con dobles (y al revés).
- Con **comillas triples** (`"""`) el texto puede ocupar varias líneas.

## Unir textos con `+`

```python title="unir.py"
nombre = "Ana"
edad = 30

print("Hola, " + nombre)  # Hola, Ana
print("Tenés " + str(edad) + " años")  # Tenés 30 años
```

Funciona, pero tiene dos problemas: hay que convertir los números con `str()` y, con varias variables, se vuelve difícil de leer. Para eso existen las f-strings.

## f-strings: variables adentro del texto

Poné una `f` antes de las comillas y escribí entre llaves `{}` lo que quieras insertar:

```python title="fstrings.py"
nombre = "Ana"
edad = 30

print(f"Hola, {nombre}. Tenés {edad} años.")
print(f"El año que viene vas a tener {edad + 1}.")
```

```text
Hola, Ana. Tenés 30 años.
El año que viene vas a tener 31.
```

- Adentro de las llaves podés poner variables o cualquier expresión, como `edad + 1`.
- No hace falta `str()`: la f-string convierte sola.
- Si te olvidás la `f`, las llaves se muestran tal cual: `"Hola, {nombre}"`.

Las f-strings son la forma recomendada de armar textos en Python. Las vas a usar todo el tiempo, por ejemplo para mensajes de error de la API.

## Dar formato a números

Después de la expresión, con `:` podés indicar cómo mostrarla:

```python title="formato.py"
precio = 1234.5
porcentaje = 0.1875

print(f"Precio: ${precio:.2f}")  # Precio: $1234.50
print(f"Precio: ${precio:,.2f}")  # Precio: $1,234.50
print(f"Descuento: {porcentaje:.1%}")  # Descuento: 18.8%

total = 4500
print(f"{total=}")  # total=4500
```

| Formato | Qué hace |
| --- | --- |
| `:.2f` | Dos decimales. |
| `:,` | Separa los miles con coma. |
| `:.1%` | Multiplica por 100 y agrega `%`, con un decimal. |
| `=` | Muestra el nombre de la variable y su valor. Muy útil para depurar. |

!!! note "Separador de miles"

    `:,` usa el formato en inglés: coma para los miles y punto para los decimales. Para mostrar `$1.234,50` hay que hacer un paso extra; en una API casi nunca hace falta, porque el formato lo resuelve el frontend.

## Métodos útiles

Los strings traen **métodos**: funciones que se llaman con un punto después del texto.

```python title="metodos.py"
email = "  Ana@Ejemplo.COM  "

limpio = email.strip().lower()
print(limpio)  # ana@ejemplo.com
print(email)  # "  Ana@Ejemplo.COM  " (no cambió)

print(limpio.upper())  # ANA@EJEMPLO.COM
print(limpio.replace("ejemplo", "empresa"))  # ana@empresa.com
print(limpio.startswith("ana"))  # True
print("@" in limpio)  # True
print(len(limpio))  # 15
```

| Método | Qué hace |
| --- | --- |
| `.strip()` | Saca los espacios del principio y del final. |
| `.lower()` / `.upper()` | Pasa todo a minúsculas / mayúsculas. |
| `.title()` | Pone en mayúscula la primera letra de cada palabra. |
| `.replace(a, b)` | Reemplaza `a` por `b`. |
| `.startswith(x)` | `True` si empieza con `x`. |
| `x in texto` | `True` si `x` aparece en el texto. |
| `len(texto)` | Cantidad de caracteres (es una función, no un método). |

Dos cosas importantes:

- Los métodos **no modifican** el string original: devuelven uno nuevo. Por eso `email` sigue igual y guardamos el resultado en `limpio`.
- Se pueden **encadenar**: `email.strip().lower()` primero saca espacios y al resultado lo pasa a minúsculas.

## Caracteres especiales

La barra invertida `\` arma caracteres que no se pueden escribir directamente:

```python title="especiales.py"
print("Línea 1\nLínea 2")  # \n: salto de línea
print("Columna\tOtra")  # \t: tabulación
print("Dijo: \"hola\"")  # \": comilla dentro del texto
```

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Las f-strings son el equivalente de los *template literals* de JavaScript:

    | JavaScript | Python |
    | --- | --- |
    | `` `Hola, ${nombre}` `` | `f"Hola, {nombre}"` |
    | `texto.trim()` | `texto.strip()` |
    | `texto.toLowerCase()` | `texto.lower()` |
    | `texto.includes("@")` | `"@" in texto` |
    | `texto.length` | `len(texto)` |
    | `precio.toFixed(2)` | `f"{precio:.2f}"` |

??? example "Ver comparación en Django ⬇️"

    Es Python, así que es igual. Ojo con no confundir las f-strings con las plantillas de Django: `{{ nombre }}` en un template HTML es otra cosa y la procesa Django, no Python.

??? example "Ver comparación en Spring Boot ⬇️"

    En Java se suele usar `String.format` o concatenar con `+`. Las f-strings son más directas:

    | Java | Python |
    | --- | --- |
    | `String.format("Hola, %s", nombre)` | `f"Hola, {nombre}"` |
    | `String.format("%.2f", precio)` | `f"{precio:.2f}"` |
    | `texto.trim()` | `texto.strip()` |
    | `texto.contains("@")` | `"@" in texto` |
    | `texto.length()` | `len(texto)` |

    Igual que en Java, los strings de Python son **inmutables**: los métodos devuelven un string nuevo.

## Mini ejercicio

Completá el script para que muestre exactamente `2 x Teclado Mecánico: $90,000`.

```python title="ejercicio.py"
producto = "  teclado mecánico "
precio = 45000
cantidad = 2

# Tu código acá
```

??? success "Ver solución"

    ```python
    producto = "  teclado mecánico "
    precio = 45000
    cantidad = 2

    nombre = producto.strip().title()
    total = precio * cantidad
    print(f"{cantidad} x {nombre}: ${total:,}")
    ```

    `.strip()` saca los espacios, `.title()` pone las mayúsculas y `:,` agrega el separador de miles.

---

**Siguiente:** [Listas →](05-listas.md)
