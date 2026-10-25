# Hashing de contraseñas (argon2 o bcrypt)

!!! info "En esta página"

    **Qué vas a aprender:** qué son argon2 y bcrypt, por qué son los algoritmos correctos para contraseñas, y cómo usarlos en tu API con la librería `pwdlib` para hashear y verificar contraseñas.

    **Qué necesitás:** [Por qué nunca guardar contraseñas en texto plano](02-contrasenas-texto-plano.md).

## Algoritmos hechos para contraseñas

En la página anterior viste los dos problemas de usar SHA-256 para contraseñas: hashes iguales para contraseñas iguales, y una velocidad que facilita probar millones de opciones. Los algoritmos diseñados para contraseñas resuelven los dos:

- **Incluyen la sal** automáticamente: cada hash usa una sal aleatoria propia, guardada dentro del mismo texto del hash.
- **Son lentos a propósito**, y la lentitud es configurable. Para tu API, tardar una fracción de segundo en un login no se nota. Para un atacante que quiere probar miles de millones de contraseñas, es la diferencia entre horas y siglos.

Los dos más usados son:

- **argon2**: el más moderno, ganador de una competencia internacional de algoritmos para contraseñas en 2015. Además de tiempo, consume **memoria**, lo que encarece los ataques con hardware especializado. Es la recomendación actual.
- **bcrypt**: más viejo (1999), muy probado y todavía seguro. Tiene un límite: solo usa los primeros 72 bytes de la contraseña.

En esta guía vas a usar **argon2**.

## La librería `pwdlib`

**`pwdlib`** es una librería que da una interfaz simple para hashear y verificar contraseñas, con argon2 o bcrypt por debajo. Instalala en tu proyecto, con el extra de argon2:

```bash
uv add "pwdlib[argon2]"
```

## El módulo de seguridad

Creá `app/seguridad.py`. Acá van a vivir todas las funciones de seguridad del capítulo:

```python title="app/seguridad.py"
from pwdlib import PasswordHash

hasheador = PasswordHash.recommended()


def hashear_contrasena(contrasena: str) -> str:
    return hasheador.hash(contrasena)


def verificar_contrasena(contrasena: str, hash_guardado: str) -> bool:
    return hasheador.verify(contrasena, hash_guardado)
```

- **`PasswordHash.recommended()`** crea un hasheador con la configuración recomendada por la librería: argon2 con parámetros seguros.
- **`hashear_contrasena`** se usa en el **registro**: recibe la contraseña y devuelve el hash, que es lo que se guarda en la base.
- **`verificar_contrasena`** se usa en el **login**: recibe lo que escribió el usuario y el hash guardado, y devuelve `True` o `False`.

Fijate que la verificación **no** compara hashes con `==`. Como cada hash tiene su sal, hashear la misma contraseña dos veces da resultados distintos. `verify` lee la sal y los parámetros que están dentro del hash guardado, recalcula con ellos y compara.

## Probarlo

En la raíz del proyecto, creá este script:

```python title="probar_hash.py"
from app.seguridad import hashear_contrasena, verificar_contrasena

hash_1 = hashear_contrasena("mi-clave-secreta")
hash_2 = hashear_contrasena("mi-clave-secreta")

print(hash_1)
print(hash_2)
print(hash_1 == hash_2)  # False: cada hash tiene su propia sal

print(verificar_contrasena("mi-clave-secreta", hash_1))  # True
print(verificar_contrasena("otra-clave", hash_1))  # False
```

Al ejecutarlo con `uv run probar_hash.py`:

```text
$argon2id$v=19$m=65536,t=3,p=4$2jQDcJHssZa9qpXe/++H4w$0JbGP2kdxJjADT2tIWstTjW4ExpPWLoFe5HYJv6cob8
$argon2id$v=19$m=65536,t=3,p=4$lGcXdB9PuWNNG0ygGBIEVg$LU/u5wcqwWnsyLS7q6nKQaM+M/+8nWE3NPkyD1Q5lgI
False
True
False
```

(Los hashes que veas van a ser otros: la sal es aleatoria.)

## Qué hay dentro de un hash de argon2

El texto del hash guarda todo lo necesario para verificarlo, separado por `$`:

```text
$argon2id$v=19$m=65536,t=3,p=4$2jQDcJHssZa9qpXe/++H4w$0JbGP2kdxJjADT2tIWstTjW4ExpPWLoFe5HYJv6cob8
 ───┬──── ─┬── ───────┬─────── ──────────┬─────────── ─────────────────┬─────────────────────────
    │      │          │                  │                             └── el hash propiamente dicho
    │      │          │                  └── la sal (aleatoria, distinta en cada hash)
    │      │          └── parámetros: memoria (64 MB), iteraciones, paralelismo
    │      └── versión del algoritmo
    └── algoritmo: argon2id
```

Por eso alcanza con **una sola columna** en la base para guardarlo: el algoritmo, los parámetros y la sal viajan juntos. Y si en el futuro querés subir los parámetros, los hashes viejos se siguen pudiendo verificar, porque cada uno dice con qué parámetros se creó.

## Lento a propósito

```python title="lento.py"
import time

from app.seguridad import hashear_contrasena

inicio = time.perf_counter()
for _ in range(10):
    hashear_contrasena("mi-clave-secreta")
segundos = time.perf_counter() - inicio

print(f"10 hashes con argon2 en {segundos:.2f} segundos")
```

```text
10 hashes con argon2 en 0.26 segundos
```

Unos 26 milésimos de segundo por hash. Compará con la página anterior: SHA-256 hacía un millón en 0,25 segundos. argon2 es unas **cien mil veces más lento**, y además cada intento ocupa 64 MB de memoria. Para un login, 26 milésimas no se notan; para un ataque, es una muralla.

!!! note "¿Y si quiero bcrypt?"

    Se instala con `uv add "pwdlib[bcrypt]"` y se crea el hasheador con `PasswordHash((BcryptHasher(),))`, importando `BcryptHasher` de `pwdlib.hashers.bcrypt`. El resto del código no cambia. Sus hashes empiezan con `$2b$12$...`, donde `12` es el "costo", que define cuán lento es.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Node se usan los paquetes `argon2` o `bcrypt`, con la misma idea:

    ```javascript
    const hash = await argon2.hash(contrasena)
    const ok = await argon2.verify(hash, contrasena)
    ```

    Fijate que en Node el orden de los argumentos de `verify` es al revés: primero el hash.

??? example "Ver comparación en Django ⬇️"

    Django hace esto por vos con `make_password()` y `check_password()`. Por defecto usa PBKDF2, pero se puede poner argon2 primero en `PASSWORD_HASHERS`, y Django actualiza los hashes viejos solo, cuando el usuario inicia sesión. `pwdlib` tiene algo parecido, `verify_and_update`, para cuando cambian los parámetros.

??? example "Ver comparación en Spring Boot ⬇️"

    Equivale a un `PasswordEncoder` de Spring Security: `encoder.encode(contrasena)` para hashear y `encoder.matches(contrasena, hash)` para verificar. `Argon2PasswordEncoder` y `BCryptPasswordEncoder` son las implementaciones equivalentes.

## Mini ejercicio

Modificá `lento.py` para estimar cuánto tardaría un atacante en probar **un millón** de contraseñas con argon2, a partir del tiempo que mediste para 10.

??? success "Ver solución"

    ```python
    por_hash = segundos / 10
    horas = por_hash * 1_000_000 / 3600
    print(f"Un millón de intentos: unas {horas:.0f} horas")
    ```

    Con 26 milésimas por hash da algo como `Un millón de intentos: unas 7 horas`, contra un cuarto de segundo con SHA-256. Y un millón de contraseñas es poco: los diccionarios que usan los atacantes tienen miles de millones. Un atacante con hardware dedicado iría más rápido que tu computadora, pero argon2, al exigir memoria por cada intento, le hace la vida mucho más difícil.

---

**Siguiente:** [Modelo de usuario →](04-modelo-de-usuario.md)
