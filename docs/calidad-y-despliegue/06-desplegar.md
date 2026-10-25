# Desplegar el backend

!!! info "En esta página"

    **Qué vas a aprender:** qué opciones hay para publicar la API y qué gana y qué pierde cada una, qué pasos son iguales en cualquier opción, y cómo se ve un despliegue en un servidor propio con Docker Compose y un proxy con HTTPS, que vas a poder probar en tu máquina.

    **Qué necesitás:** [Preparar la app para producción](05-preparar-para-produccion.md) y [API y base juntas](../docker/08-api-y-base-juntas.md).

## Qué es desplegar

**Desplegar** es poner la API en una computadora que esté prendida todo el tiempo, conectada a internet, con una dirección fija (un dominio) y HTTPS, para que el frontend y los usuarios la puedan usar. Tu imagen de Docker ya tiene todo lo que la API necesita; lo que cambia entre una opción y otra es **quién se ocupa del resto**: el servidor, la base, los certificados, las actualizaciones.

## Las opciones

| Opción | Cómo es | A favor | En contra |
| --- | --- | --- | --- |
| **Plataforma de contenedores** (por ejemplo Google Cloud Run, Fly.io o Render) | Le das tu imagen o tu repositorio y la plataforma la ejecuta. | Poco mantenimiento: HTTPS, reinicios, logs y escalado vienen hechos. | Menos control, y el costo crece con el uso. Cada plataforma tiene sus propias reglas. |
| **Servidor propio** (un VPS) con Docker Compose | Alquilás una máquina virtual con Linux y levantás vos los contenedores. | Control total y un costo fijo y predecible. Es lo que ya sabés hacer. | Todo es tuyo: actualizaciones del sistema, seguridad, copias de la base. |
| **Funciones** (*serverless*) | El código se ejecuta solo cuando llega un pedido. | Muy barato con poco tráfico. | Arranques lentos, límites de tiempo, y no encaja bien con conexiones a la base ni con estado en memoria. |

Para la base de datos la elección es parecida: una **base gestionada** (el proveedor se ocupa de las copias de seguridad, las actualizaciones y la disponibilidad) o un **contenedor de PostgreSQL** en tu servidor, donde todo eso queda a tu cargo. Para una primera API, una plataforma de contenedores con una base gestionada es lo más tranquilo. Un VPS es la opción más barata a la larga, y la que más enseña.

## Lo mismo en cualquier opción

- **La imagen de Docker** del Capítulo 4, que se construye igual en todos lados.
- **Las variables de entorno de producción** de la [página anterior](05-preparar-para-produccion.md#las-variables-en-produccion).
- **Las migraciones**, aplicadas con `alembic upgrade head` antes de que la versión nueva atienda pedidos. Se puede correr a mano o en el arranque, como en el mini ejercicio de [API y base juntas](../docker/08-api-y-base-juntas.md).
- **Un health check:** casi todas las plataformas preguntan periódicamente si la app está viva. Tu ruta `/salud` sirve para eso, porque además comprueba la base.
- **HTTPS**, que lo da la plataforma o un proxy delante de la API.
- **Los logs a la salida estándar**, como ya hace `logging`: las plataformas los juntan y te los muestran.

## Un servidor propio con Compose y Caddy

En un servidor, delante de la API se pone un **proxy inverso**: un programa que recibe todos los pedidos de internet, se encarga de HTTPS, y se los pasa a la API. Vas a usar **Caddy**, que consigue y renueva los certificados HTTPS solo. Creá un `Caddyfile` en la carpeta del proyecto:

```text title="Caddyfile"
localhost {
	reverse_proxy api:8000
}
```

Dice "lo que llegue para `localhost`, pasáselo al servicio `api`, puerto 8000". Y un Compose aparte para producción:

```yaml title="docker-compose.prod.yml"
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Falta POSTGRES_PASSWORD en el .env}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - datos_db:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 2s
      timeout: 5s
      retries: 10
    restart: unless-stopped

  api:
    build: .
    command: sh -c "alembic upgrade head && fastapi run app/main.py --port 8000 --forwarded-allow-ips='*'"
    env_file: .env
    environment:
      URL_BASE_DATOS: postgresql+asyncpg://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  proxy:
    image: caddy:2
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - datos_caddy:/data
    depends_on:
      - api
    restart: unless-stopped

volumes:
  datos_db:
  datos_caddy:
```

Las diferencias con el `docker-compose.yml` de desarrollo:

- **Solo el proxy publica puertos** (80 y 443). La base y la API no tienen `ports:`: desde afuera no se puede llegar a ellas, solo a través de Caddy, por la red interna de Compose.
- **`command`** aplica las migraciones y después arranca la API. `alembic upgrade head` no hace nada si la base ya está al día, así que se puede correr en cada arranque.
- **`restart: unless-stopped`** vuelve a levantar un contenedor si se cae, o si se reinicia el servidor.
- **`datos_caddy`** guarda los certificados, para no pedirlos de nuevo en cada reinicio.
- **`--forwarded-allow-ips`** se explica más abajo.

Si todavía tenés levantado el Compose de desarrollo, bajalo con `docker compose down`, y levantá este:

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

Con `localhost`, Caddy crea su propio certificado, que tu sistema no conoce. Por eso `curl https://localhost/salud` falla con `SSL certificate problem: unable to get local issuer certificate`, y para probar hay que agregar `-k` ("no verifiques el certificado"):

```bash
curl -k https://localhost/salud
```

```json
{"base_de_datos":"ok"}
```

Y si alguien entra por HTTP, Caddy lo manda a HTTPS: `curl -i http://localhost/salud` responde `308 Permanent Redirect` con `Location: https://localhost/salud`.

En un servidor real, `localhost` se reemplaza por tu dominio (por ejemplo, `api.ejemplo.com`), que tiene que apuntar a la IP del servidor. Caddy pide entonces un certificado de verdad a Let's Encrypt, sin que hagas nada más, y el `-k` deja de hacer falta.

## Detrás de un proxy: la IP real

Sin `--forwarded-allow-ips`, la API cree que todos los pedidos vienen de Caddy. Los logs lo muestran:

```text
api-1  | INFO:     172.20.0.4:48734 - "POST /auth/login HTTP/1.1" 401 Unauthorized
api-1  | 2026-10-25 05:09:57,741 WARNING app.dependencias: Demasiados intentos desde 172.20.0.4
```

`172.20.0.4` es la IP del contenedor de Caddy. El [límite de intentos](../autenticacion/12-recomendaciones-seguridad.md) cuenta por IP, así que **todos los usuarios comparten un mismo límite**: cinco logins fallidos de cualquiera bloquean el login de todos durante un minuto.

Caddy le avisa a la API la IP real del cliente en un encabezado, `X-Forwarded-For`. Uvicorn solo lo lee si viene de una IP en la que confía, y por defecto confía solo en `127.0.0.1`. `--forwarded-allow-ips='*'` le dice que confíe en cualquiera. Con eso, el log muestra la IP del cliente (en esta prueba, `192.168.65.1`, la de tu máquina vista desde Docker):

```text
api-1  | 2026-10-25 05:10:17,621 WARNING app.dependencias: Demasiados intentos desde 192.168.65.1
```

!!! warning "`'*'` solo si la API no está expuesta"

    Confiar en cualquiera es seguro acá porque a la API solo se llega a través de Caddy, que reemplaza el `X-Forwarded-For` que manda el cliente. Si la API estuviera publicada directamente, cualquiera podría mandar `X-Forwarded-For: 1.2.3.4` y la API le creería (el log diría `1.2.3.4`), y así se saltearía el límite de intentos cambiando de "IP" en cada pedido.

## Comparaciones

??? example "Ver comparación en Express ⬇️"

    El despliegue es el mismo: una imagen de Docker, las mismas opciones de plataforma y un proxy delante. El equivalente de `--forwarded-allow-ips` en Express es `app.set("trust proxy", 1)`, y si falta, `express-rate-limit` tiene exactamente el mismo problema de ver a todos con la IP del proxy.

??? example "Ver comparación en Django ⬇️"

    Django se despliega con Gunicorn (o Uvicorn) detrás de Nginx o Caddy. Además, hay que ejecutar `python manage.py collectstatic` para los archivos estáticos y `migrate` en cada versión nueva, el equivalente de `alembic upgrade head`. Detrás de un proxy, la IP real también hay que leerla de `X-Forwarded-For`.

??? example "Ver comparación en Spring Boot ⬇️"

    Spring Boot genera un `.jar` que trae su propio servidor, y se despliega igual: en una imagen de Docker, en una plataforma o en un VPS detrás de un proxy. Las migraciones las aplican Flyway o Liquibase solos al arrancar, y la IP real se activa con `server.forward-headers-strategy=native`.

## Mini ejercicio

Con el Compose de producción levantado, reiniciá la API con `docker compose -f docker-compose.prod.yml restart api` y mirá sus logs con `docker compose -f docker-compose.prod.yml logs api`. ¿Se volvió a aplicar la migración? ¿Por qué no hay problema en correr `alembic upgrade head` en cada arranque?

??? success "Ver solución"

    No se vuelve a aplicar. En el primer arranque, los logs muestran `Running upgrade  -> ..., crear tablas`; en los siguientes, solo aparecen `Context impl PostgresqlImpl.` y `Will assume transactional DDL.`, sin ningún `Running upgrade`.

    Alembic guarda en la tabla `alembic_version` qué migración tiene aplicada la base. `upgrade head` compara esa versión con la última que existe, y si son iguales no hace nada. Por eso se puede correr en cada arranque: solo hace algo cuando hay una migración nueva.

    Cuando termines, bajá todo con `docker compose -f docker-compose.prod.yml down`. Si además querés borrar los datos y los certificados, agregá `-v`.

---

¡Terminaste el Capítulo 6, y con él la guía! Tu API tiene tests, código prolijo, logs útiles, una configuración lista para producción, y sabés cómo publicarla. Los apéndices quedan como referencia para cuando los necesites.

**Siguiente:** [Apéndices · Glosario →](../apendices/01-glosario.md)
