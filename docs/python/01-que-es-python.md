# Qué es Python y cómo se ejecuta un script

!!! info "En esta página"

    **Qué vas a aprender:** qué es Python, qué es un script y en qué orden se ejecuta su código.

    **Qué necesitás:** [Cómo leer las comparaciones](../introduccion/03-como-leer-las-comparaciones.md).

## Python es un lenguaje interpretado

Python es un lenguaje de programación. Escribís instrucciones en un archivo de texto con extensión `.py` y un programa llamado **intérprete** las lee y las ejecuta.

"Interpretado" quiere decir que no hay un paso previo de compilación: le pasás el archivo al intérprete y lo ejecuta en el momento, renglón por renglón.

A ese archivo `.py` que ejecutás directamente se lo llama **script**.

## Tu primer script

Este es un script completo:

```python title="hola.py"
print("Hola, mundo")
print("Esto es Python")
```

`print(...)` es una **función** que muestra en la terminal lo que le pases entre paréntesis. El texto va entre comillas para que Python sepa que es texto y no una instrucción.

Para ejecutarlo, en la terminal le pedís al intérprete que lo corra:

```bash
uv run hola.py
```

Y la terminal muestra:

```text
Hola, mundo
Esto es Python
```

!!! note "Todavía no lo vas a poder ejecutar"

    `uv` es la herramienta que instala Python por vos, y la instalás en la [próxima página](02-instalar-uv.md). Por ahora quedate con la idea: **un archivo `.py` + el intérprete = el programa corriendo**.

    En otras guías vas a ver `python hola.py`. Es lo mismo: `uv run` termina llamando al intérprete de Python.

## El código se ejecuta de arriba hacia abajo

Python lee el script desde el primer renglón hasta el último, en orden. Si un renglón falla, el script se detiene ahí y lo que sigue **no** se ejecuta.

```python title="orden.py"
print("Paso 1")
print("Paso 2")
print(nombre)
print("Paso 4")
```

En el tercer renglón usamos `nombre`, pero nunca dijimos qué es. Al ejecutarlo:

```text
Paso 1
Paso 2
Traceback (most recent call last):
  File "orden.py", line 3, in <module>
    print(nombre)
          ^^^^^^
NameError: name 'nombre' is not defined
```

Fijate en tres cosas:

- `Paso 1` y `Paso 2` **sí** se mostraron: esos renglones se ejecutaron antes del error.
- `Paso 4` **no** aparece: el script se cortó en el renglón 3.
- El mensaje de error (el *traceback*) te dice el archivo, el renglón (`line 3`) y qué pasó (`NameError`). Leelo siempre de abajo hacia arriba: la última línea es la más importante.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Node también es un intérprete: `uv run hola.py` es el equivalente de `node hola.js`. Y un script de Node también se ejecuta de arriba hacia abajo y se corta en el primer error no manejado.

    ```javascript
    // hola.js
    console.log("Hola, mundo")
    ```

    `print(...)` cumple el rol de `console.log(...)`.

??? example "Ver comparación en Django ⬇️"

    Django **es** Python, así que todo lo de esta página aplica igual. De hecho, cuando ejecutás `python manage.py runserver`, estás corriendo un script de Python: `manage.py`.

??? example "Ver comparación en Spring Boot ⬇️"

    Java es compilado: primero `javac` traduce el código a *bytecode* (archivos `.class`) y después la JVM lo ejecuta. En Python no hay paso de compilación visible: le pasás el `.py` al intérprete y listo.

    Tampoco hace falta una clase con un método `main`. El script entero es el "main": Python ejecuta lo que encuentra, de arriba hacia abajo.

    ```java
    public class Hola {
        public static void main(String[] args) {
            System.out.println("Hola, mundo");
        }
    }
    ```

## Mini ejercicio

¿Qué muestra este script en la terminal? Pensalo antes de abrir la solución.

```python title="ejercicio.py"
print("Arranca el script")
print(2 + 3)
print("2 + 3")
```

??? success "Ver solución"

    ```text
    Arranca el script
    5
    2 + 3
    ```

    Sin comillas, `2 + 3` es una cuenta y Python muestra el resultado. Con comillas, `"2 + 3"` es texto y se muestra tal cual.

---

**Siguiente:** [Instalar y usar `uv` →](02-instalar-uv.md)
