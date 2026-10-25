# Recomendaciones de seguridad (HTTPS, rate limiting, secretos fuera del código)

!!! info "En esta página"

    **Qué vas a aprender:** las protecciones que rodean a la autenticación y que no conviene dejar para después: HTTPS, limitar los intentos de login, cuidar los secretos y otras buenas prácticas para cuando la API salga a producción.

    **Qué necesitás:** [Dónde guardar el token en el frontend](11-donde-guardar-token.md).

## HTTPS, siempre

Todo lo que armaste en este capítulo (contraseñas en el login, tokens en cada pedido, cookies) viaja por la red. Con **HTTP** común, viaja en texto plano: cualquiera en el camino (la red Wi-Fi de un bar, un proveedor, un equipo comprometido) puede leerlo y copiarlo. Con **HTTPS**, la conexión está cifrada.

En producción, **HTTPS no es opcional**. La buena noticia es que casi nunca lo configura tu código:

- Las plataformas donde se publican APIs (las vas a ver en el Capítulo 6) suelen dar HTTPS automáticamente.
- Si administrás tu propio servidor, se pone delante de la API un **proxy inverso** (como Nginx o Caddy) que maneja los certificados, y tu API sigue hablando HTTP solo dentro del servidor.

Por eso la cookie de refresco tiene `secure=True`: si algún día alguien accede por HTTP, el navegador directamente no la manda.

## Limitar los intentos de login

El login es la puerta de entrada, y un atacante puede probar contraseñas sin parar. argon2 hace que cada intento sea lento, pero no impide probar miles por hora. Por eso se limita **cuántos intentos** se aceptan en un período: es lo que se llama ***rate limiting***.

Una versión simple, por dirección IP, como dependencia. Agregala al final de `app/dependencias.py` (y sumá `import time` y `Request` a los imports):

```python title="app/dependencias.py (agregar al final)"
MAXIMO_INTENTOS = 5
VENTANA_SEGUNDOS = 60
intentos_por_ip: dict[str, list[float]] = {}


def limitar_intentos(request: Request) -> None:
    ip = request.client.host if request.client else "desconocida"
    ahora = time.monotonic()
    recientes = [t for t in intentos_por_ip.get(ip, []) if ahora - t < VENTANA_SEGUNDOS]
    if len(recientes) >= MAXIMO_INTENTOS:
        espera = int(VENTANA_SEGUNDOS - (ahora - recientes[0])) + 1
        raise HTTPException(
            status_code=status.HTTP_429_TOO_MANY_REQUESTS,
            detail="Demasiados intentos. Probá de nuevo en un rato.",
            headers={"Retry-After": str(espera)},
        )
    recientes.append(ahora)
    intentos_por_ip[ip] = recientes
```

- **`Request`** es el objeto con todos los datos del pedido; `request.client.host` es la IP de quien lo hace.
- Por cada IP se guarda una lista con los momentos de sus intentos. Se descartan los de hace más de 60 segundos y, si quedan 5 o más, se responde **`429 Too Many Requests`**.
- **`Retry-After`** es el encabezado estándar que le dice al cliente cuántos segundos esperar.
- `time.monotonic()` es un reloj que siempre avanza, aunque cambie la hora del sistema: es el adecuado para medir intervalos.

Aplicalo al login en `app/routers/auth.py` (importando `limitar_intentos`):

```python
@router.post("/login", response_model=Token, dependencies=[Depends(limitar_intentos)])
```

Probalo con siete intentos seguidos con una contraseña equivocada:

```text
intento 1 -> {"detail":"Email o contraseña incorrectos"} [401]
...
intento 5 -> {"detail":"Email o contraseña incorrectos"} [401]
intento 6 -> {"detail":"Demasiados intentos. Probá de nuevo en un rato."} [429]
intento 7 -> {"detail":"Demasiados intentos. Probá de nuevo en un rato."} [429]
```

Incluso con la contraseña **correcta**, durante ese minuto la respuesta es `429`. Es a propósito: si no, el atacante sabría que acertó.

!!! warning "Esto es una versión para aprender"

    El diccionario vive en la memoria del proceso: se borra al reiniciar, no se comparte entre varias copias de la API, y crece con cada IP nueva. En producción, el límite se pone con un almacenamiento compartido (por ejemplo, Redis), con una librería dedicada, o directamente en el proxy inverso o la plataforma, que suelen ofrecerlo.

## Los secretos, fuera del código

Ya lo venís haciendo desde el Capítulo 2, pero en autenticación es crítico. Tu API tiene tres secretos: la **contraseña de la base**, la **clave de los JWT** y, antes, la `CLAVE_API`. Con la clave de los JWT, cualquiera puede fabricar un token de administrador.

- **Nunca en el código ni en Git.** Van en el `.env` (que está en el `.gitignore`) o en las variables de entorno de la plataforma donde publiques. Podés comprobar que Git ignora el archivo con `git check-ignore -v .env`.
- **Distintos en cada entorno.** La clave de desarrollo no es la de producción.
- **Largos y aleatorios.** Generados con `secrets.token_hex(32)`, nunca inventados.
- **Si se filtra uno, se cambia.** Si subiste un secreto a Git por error, borrar el archivo no alcanza: queda en el historial. Hay que generar uno nuevo. Al cambiar la clave de los JWT, todos los tokens emitidos dejan de valer y todos tienen que volver a iniciar sesión.

## Otras buenas prácticas

| Práctica | Por qué |
| --- | --- |
| `DEBUG=false` en producción | Con `debug` activado, un error `500` devuelve el traceback completo (lo viste en el Capítulo 3). |
| No registrar contraseñas ni tokens en los logs | Los logs se guardan, se copian y los leen más personas que la base. |
| CORS con orígenes concretos | Nunca `"*"`, y menos con credenciales. |
| Validar todo lo que entra | Pydantic ya lo hace; mantené los esquemas estrictos (como `UsuarioCrear`, sin `rol`). |
| Mensajes de error genéricos en el login | No revelar qué emails existen. |
| Actualizar las dependencias | Las librerías publican arreglos de seguridad; `uv lock --upgrade` las actualiza. |
| Usuario de base con permisos mínimos | En producción, el usuario de la API no necesita poder borrar la base entera. |

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    En Express, el rate limiting se suele hacer con `express-rate-limit` (con un almacenamiento en Redis para varias instancias), y varias cabeceras de seguridad con `helmet`. La lista de buenas prácticas es la misma: HTTPS delante de la app, secretos en variables de entorno, errores sin detalles en producción.

??? example "Ver comparación en Django ⬇️"

    Django trae varias de estas protecciones: `DEBUG = False` y `ALLOWED_HOSTS` en producción, `SECURE_SSL_REDIRECT` y `SECURE_HSTS_SECONDS` para HTTPS, y `python manage.py check --deploy` revisa la configuración. En DRF, el *throttling* (`AnonRateThrottle`, `UserRateThrottle`) cumple el rol de `limitar_intentos`.

??? example "Ver comparación en Spring Boot ⬇️"

    Spring Security agrega por defecto varias cabeceras de seguridad y permite exigir HTTPS con `requiresSecure()`. Para rate limiting se usan librerías como Bucket4j o la configuración del gateway. Los secretos se inyectan con variables de entorno o con herramientas como Spring Cloud Config o un gestor de secretos.

## Mini ejercicio

El registro también se puede abusar: un bot podría crear miles de cuentas. Aplicale el mismo límite a `POST /auth/registro` y comprobalo creando seis cuentas seguidas.

??? success "Ver solución"

    ```python
    @router.post(
        "/registro",
        response_model=UsuarioRespuesta,
        status_code=status.HTTP_201_CREATED,
        dependencies=[Depends(limitar_intentos)],
    )
    ```

    Las primeras cinco cuentas se crean (`201`) y la sexta recibe `429`. Fijate un detalle: el login y el registro **comparten** el mismo contador por IP, porque usan el mismo diccionario. Para tener límites separados, la dependencia tendría que recibir un nombre o crear un diccionario por ruta; las librerías de rate limiting ya traen esa flexibilidad.

---

**Siguiente:** [Errores comunes de autenticación →](13-errores-comunes.md)
