# Por qué nunca guardar contraseñas en texto plano

!!! info "En esta página"

    **Qué vas a aprender:** por qué guardar las contraseñas tal cual es peligroso, qué es un **hash**, por qué un hash común no alcanza, y qué es la **sal**. Todo con ejemplos usando solo la biblioteca estándar.

    **Qué necesitás:** [Qué es autenticar y qué es autorizar](01-autenticar-y-autorizar.md).

## Las bases de datos se filtran

Para comprobar la contraseña de un usuario en el login, la API necesita guardar *algo*. La idea más simple sería una columna `contraseña` con el texto tal cual: `123456`. Es un error grave, por una razón: **las bases de datos se filtran**. No es una posibilidad remota; pasa todo el tiempo, incluso en empresas enormes:

- Un backup que quedó en un lugar público.
- Un error de programación que permite leer datos ajenos (como la inyección SQL del Capítulo 3).
- Alguien con acceso interno que copia la base.

Si las contraseñas están en texto plano, quien tenga la copia las ve todas. Y como mucha gente **repite la misma contraseña** en varios sitios, el daño no se queda en tu app: con el email y la contraseña, el atacante prueba entrar al correo, al banco o a las redes de cada usuario.

La regla es absoluta: **la API nunca debe poder saber la contraseña de un usuario**. Ni siquiera vos, mirando la base.

## ¿Y encriptarlas?

Encriptar es reversible: con la **clave**, se recupera el texto original. Eso suena bien hasta que pensás que la clave tiene que estar en algún lado del servidor para poder usarla... y si se filtra la base, la clave puede filtrarse con ella. Para contraseñas no se encripta: se usa un **hash**.

## Qué es un hash

Una **función de hash** transforma un texto en otro de largo fijo, de forma que:

- El **mismo texto** da **siempre el mismo** resultado.
- Un cambio mínimo en el texto da un resultado **completamente distinto**.
- **No se puede volver atrás**: a partir del hash no hay forma de calcular el texto original.

Python trae varias en el módulo `hashlib`. Probá con SHA-256:

```python title="hash_simple.py"
import hashlib


def hashear(texto: str) -> str:
    return hashlib.sha256(texto.encode()).hexdigest()


print(hashear("mi-clave-secreta"))
print(hashear("mi-clave-secreta"))  # mismo texto, mismo hash
print(hashear("mi-clave-secretA"))  # un carácter distinto, hash totalmente distinto
```

```text
13b3bb0b09bf9861fe598e50191c74f0e3cf1fbe1302718028441a890424feb1
13b3bb0b09bf9861fe598e50191c74f0e3cf1fbe1302718028441a890424feb1
72fb03f56d0d3ee61a2534f4d26dc13488d27b62ce4419dbbad1773c1e536eae
```

`.encode()` convierte el texto en bytes (las funciones de hash trabajan con bytes) y `.hexdigest()` devuelve el resultado como texto legible.

Con esto, la idea es: en el **registro** se guarda el hash de la contraseña, no la contraseña. En el **login**, se calcula el hash de lo que escribió el usuario y se compara con el guardado. Si coinciden, la contraseña es correcta, y la API nunca guardó el texto original.

## Por qué un hash común no alcanza

SHA-256 es excelente para muchas cosas, pero para contraseñas tiene dos problemas.

**1. Contraseñas iguales dan hashes iguales.** Mirá lo que pasa si dos usuarios usan la misma:

```python title="iguales.py"
import hashlib

usuarios = {
    "ana@ejemplo.com": "123456",
    "juan@ejemplo.com": "mi-perro-firulais",
    "luz@ejemplo.com": "123456",
}

for email, clave in usuarios.items():
    print(email, hashlib.sha256(clave.encode()).hexdigest()[:16])
```

```text
ana@ejemplo.com 8d969eef6ecad3c2
juan@ejemplo.com 3bb555c8f1a938de
luz@ejemplo.com 8d969eef6ecad3c2
```

Quien vea la base sabe que Ana y Luz tienen la misma contraseña. Peor: existen listas enormes, ya calculadas, con los hashes de millones de contraseñas comunes (se las llama *rainbow tables*). Si `8d969eef...` está en esa lista, la contraseña queda al descubierto al instante.

**2. Es demasiado rápido.** SHA-256 está diseñado para ser veloz:

```python title="velocidad.py"
import hashlib
import time

inicio = time.perf_counter()
for numero in range(1_000_000):
    hashlib.sha256(str(numero).encode()).hexdigest()
segundos = time.perf_counter() - inicio

print(f"Un millón de hashes en {segundos:.2f} segundos")
```

```text
Un millón de hashes en 0.25 segundos
```

Eso es en Python, en una computadora común. Con hardware especializado se calculan **miles de millones por segundo**. Probar todas las contraseñas posibles de 8 caracteres deja de ser imposible.

## La sal

Para el primer problema existe la **sal** (*salt*): un texto aleatorio distinto para cada usuario, que se mezcla con la contraseña antes de calcular el hash. Se guarda junto al hash (no es secreta).

```python title="sal.py"
import hashlib
import secrets


def hashear_con_sal(clave: str) -> tuple[str, str]:
    sal = secrets.token_hex(16)
    hash_ = hashlib.sha256((sal + clave).encode()).hexdigest()
    return sal, hash_


for _ in range(2):
    sal, hash_ = hashear_con_sal("123456")
    print(f"sal={sal[:8]}...  hash={hash_[:16]}...")
```

```text
sal=59e48fdd...  hash=e26ac3f8f72bc5e0...
sal=3181169c...  hash=8e071eb661bf32b9...
```

La misma contraseña da hashes distintos (los tuyos van a ser otros, porque la sal es aleatoria). Las *rainbow tables* dejan de servir, y ya no se ve quién comparte contraseña. El módulo `secrets` genera valores aleatorios seguros, aptos para estos usos; para seguridad, nunca uses `random`.

Pero la sal no resuelve el segundo problema: SHA-256 sigue siendo igual de rápido.

!!! warning "No armes tu propio sistema"

    Este código es para **entender** las ideas, no para usarlo. Para contraseñas existen algoritmos diseñados a propósito para ser **lentos** y que ya incluyen la sal, como **argon2** y **bcrypt**. Inventar un esquema propio con SHA-256 es uno de los errores de seguridad más comunes. En la próxima página vas a usar argon2 a través de una librería.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Node existe el mismo módulo para hashes generales (`crypto.createHash("sha256")`), y el mismo consejo aplica: para contraseñas se usan librerías como `bcrypt` o `argon2`, nunca `crypto.createHash` a mano.

??? example "Ver comparación en Django ⬇️"

    Django resuelve esto por vos: `User.set_password()` guarda un hash con sal y un algoritmo lento (PBKDF2 por defecto, y se puede configurar argon2), y `check_password()` lo verifica. Si mirás la columna `password` de `auth_user`, vas a ver algo como `pbkdf2_sha256$...`, nunca la contraseña.

??? example "Ver comparación en Spring Boot ⬇️"

    Spring Security trae `PasswordEncoder`, con implementaciones como `BCryptPasswordEncoder` y `Argon2PasswordEncoder`. El `NoOpPasswordEncoder`, que guarda el texto plano, existe solo para pruebas y está marcado como obsoleto, justamente por lo que explica esta página.

## Mini ejercicio

Una base filtrada tiene este hash SHA-256, sin sal: `8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92`. Escribí un script que pruebe las contraseñas `password`, `qwerty`, `123456` y `admin`, y averigüe cuál es.

??? success "Ver solución"

    ```python title="ejercicio.py"
    import hashlib

    hash_guardado = "8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92"

    for intento in ["password", "qwerty", "123456", "admin"]:
        if hashlib.sha256(intento.encode()).hexdigest() == hash_guardado:
            print(f"La contraseña es: {intento}")
    ```

    ```text
    La contraseña es: 123456
    ```

    Acabás de hacer un ataque de diccionario: probar contraseñas comunes hasta encontrar una que dé el mismo hash. Con una lista de millones de contraseñas y la velocidad de SHA-256, es cuestión de segundos. Por eso hacen falta la sal y un algoritmo lento.

---

**Siguiente:** [Hashing de contraseñas (argon2 o bcrypt) →](03-hashing-argon2.md)
