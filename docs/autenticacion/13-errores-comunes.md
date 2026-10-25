# Errores comunes de autenticación

!!! info "En esta página"

    **Qué vas a aprender:** a reconocer y resolver los errores más frecuentes al trabajar con autenticación: los que dan `401` sin motivo aparente, los que hacen vencer tokens antes de tiempo y los que filtran datos sin que te des cuenta.

    **Qué necesitás:** haber recorrido el capítulo hasta [Recomendaciones de seguridad](12-recomendaciones-seguridad.md).

Cada error tiene el síntoma (lo que ves), la causa y la solución. Todos los mensajes son los reales de la API que armaste en el capítulo.

## 1. El hash de la contraseña aparece en una respuesta

**Síntoma:** una ruta devuelve el usuario con un campo de más:

```json
{"email":"juan@ejemplo.com","rol":"usuario","hash_contrasena":"$argon2id$v=19$m=65536,t=3,p=4$hWecIrhH...","id":8,"nombre":"Juan","activo":true}
```

**Causa:** la ruta devuelve el objeto `Usuario` de SQLAlchemy **sin `response_model`**. Sin un modelo de respuesta que filtre, FastAPI convierte el objeto entero, con todas sus columnas.

**Solución:** toda ruta que devuelva un usuario tiene que tener `response_model=UsuarioRespuesta` (o un esquema que no incluya el hash). Es el error más peligroso de esta lista porque **no da ningún error**: todo parece funcionar.

## 2. `401 Not authenticated` con el token en la mano

**Síntoma:** mandás el token y la respuesta es `{"detail":"Not authenticated"}`.

**Causa:** el encabezado no tiene el formato que espera `OAuth2PasswordBearer`. Todos estos fallan:

| Encabezado | Problema |
| --- | --- |
| `Authorization: Token eyJ...` | Tiene que decir `Bearer`. |
| `Authorization: eyJ...` | Falta `Bearer`. |
| `Authorization: BearereyJ...` | Falta el espacio. |

**Solución:** exactamente `Authorization: Bearer <token>`, con un espacio. (`bearer` en minúsculas sí funciona.)

## 3. `401 No se pudo validar el token`

Este mensaje es de tu dependencia `obtener_usuario_actual`: el encabezado llegó bien, pero el token no sirve. Las causas más comunes:

- **El token venció.** Pasaron más de `minutos_token` desde el login. Renovalo con `/auth/refrescar` o volvé a iniciar sesión.
- **Mandaste el token de refresco** en vez del de acceso. Cada uno sirve para lo suyo.
- **Copiaste el token con comillas.** Al copiarlo del JSON es fácil llevarse las `"` del principio y el final: `Bearer "eyJ..."` falla.
- **Cambiaste `CLAVE_JWT`.** Los tokens firmados con la clave anterior dejan de valer. Es lo esperado: hay que volver a iniciar sesión.
- **La cuenta está desactivada** o el usuario se borró.

## 4. El token vence al instante

**Síntoma:** iniciás sesión, y el token recién creado ya da `401`. Si lo mirás con PyJWT, el error es `ExpiredSignatureError: Signature has expired`.

**Causa:** el vencimiento se calculó con `datetime.now()` **sin zona horaria**. PyJWT interpreta esa hora como si fuera UTC. En Argentina, que está 3 horas detrás de UTC, un token "de 30 minutos" queda vencido hace 2 horas y media.

**Solución:** usá siempre `datetime.now(timezone.utc)`, como en `crear_token`.

## 5. `InvalidSubjectError: Subject must be a string`

**Causa:** pusiste el `id` del usuario en `sub` como número (`{"sub": 4}`). PyJWT exige que `sub` sea texto.

**Solución:** `{"sub": str(usuario_id)}` al crearlo, e `int(datos["sub"])` al leerlo.

## 6. El login responde `422` con `Field required`

**Causa:** mandaste los datos como JSON, pero el login espera un **formulario** (`username` y `password`), por `OAuth2PasswordRequestForm`.

**Solución:** con `curl`, `-d "username=...&password=..."` sin el encabezado de JSON. Desde JavaScript, `Content-Type: application/x-www-form-urlencoded`. O usá el formulario de `/docs`.

## 7. "La contraseña es correcta" pero el login falla

Causas típicas:

- **Comparar hashes con `==`.** `hashear_contrasena(clave) == hash_guardado` da siempre `False`, porque cada hash tiene su propia sal. Usá `verificar_contrasena`.
- **Mayúsculas en el email.** Si el registro guardó `Ana@Ejemplo.com` tal cual y el login busca `ana@ejemplo.com`, no lo encuentra. Pasá el email a minúsculas en los **dos** lugares.
- **El límite de intentos.** Después de 5 intentos fallidos en un minuto, incluso la contraseña correcta recibe `429`. Esperá lo que dice `Retry-After`.

## 8. La cookie de refresco no llega

**Síntoma:** `/auth/refrescar` responde `401` aunque el login guardó la cookie.

Revisá:

- **El `path`.** La cookie tiene `path="/auth"`: solo se manda a rutas que empiezan con `/auth`.
- **`SameSite` y dominios distintos.** Con `samesite="strict"`, si el frontend y la API están en sitios diferentes, el navegador no la manda.
- **`secure=True` sin HTTPS.** Fuera de `localhost`, una cookie `secure` solo viaja por HTTPS.
- **`fetch` desde otro origen sin `credentials: "include"`** (y la API sin `allow_credentials=True` en CORS).

## 9. `403` donde esperabas `401` (o al revés)

Volvé a la primera página del capítulo: `401` es "no sé quién sos" y `403` es "sé quién sos, pero no podés". Si un usuario común recibe `401` en una ruta de administración, probablemente el problema no es su rol, sino su token. Si recibe `403`, el token está bien.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    Son los mismos errores en cualquier stack: el `Bearer` mal escrito, el token vencido por una hora mal calculada, el hash que se escapa al devolver el objeto del ORM (en Node, el clásico `res.json(usuario)` con la contraseña adentro) y la cookie que no llega por `SameSite` o `credentials`.

??? example "Ver comparación en Django ⬇️"

    En Django algunos de estos errores no aparecen, porque la autenticación viene armada: `check_password` evita comparar hashes a mano, y los serializers de DRF obligan a declarar qué campos salen. Pero con JWT (simplejwt) aparecen los mismos problemas de encabezados, vencimientos y cookies.

??? example "Ver comparación en Spring Boot ⬇️"

    En Spring, el equivalente del error 1 es devolver la entidad `Usuario` y que Jackson incluya el campo `password`; se evita con DTOs o `@JsonIgnore`. Y los problemas de zona horaria al calcular vencimientos se evitan usando `Instant` en vez de `LocalDateTime`, la misma idea que `timezone.utc`.

## Mini ejercicio

Una compañera te dice: "Hago login en `/docs`, copio el `access_token` de la respuesta, lo pego en el encabezado de Postman como `Authorization: Bearer "eyJhbGciOi..."` y siempre me da `401 No se pudo validar el token`. El token es nuevo, lo acabo de generar". ¿Qué está pasando?

??? success "Ver solución"

    Copió el token **con las comillas** del JSON. El encabezado tiene el formato correcto (`Bearer` + espacio), así que `OAuth2PasswordBearer` lo acepta y le pasa el texto a tu dependencia... pero ese texto empieza y termina con `"`, y PyJWT no lo reconoce como un token válido. Por eso el mensaje es `No se pudo validar el token` y no `Not authenticated`.

    La solución es pegar solo los caracteres del token, sin comillas. Y fijate cómo el **mensaje** ayuda a diagnosticar: `Not authenticated` apunta al formato del encabezado; `No se pudo validar el token`, al token en sí.

---

¡Terminaste el Capítulo 5! Tu API tiene registro, login, tokens con renovación, rutas protegidas, roles y las protecciones básicas de seguridad. En el último capítulo vas a escribir tests, ordenar el código y prepararla para publicarla.

**Siguiente:** [Capítulo 6 · Tests con pytest: primer test →](../calidad-y-despliegue/01-pytest-primer-test.md)
