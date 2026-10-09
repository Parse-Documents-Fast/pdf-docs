# ADR-0006: pdf-infra gestiona la instancia de MongoDB; pdf-persistance la consume

**Estado:** Aceptado

## Contexto

`pdf-persistance` es el único servicio que accede a los datos y `pdf-main` es su único cliente (ADR-0004). Además, está pensado para escalar horizontalmente, con varias réplicas detrás de un mismo nombre de servicio.

Si cada réplica, o el compose de cada servicio, levantara su propia instancia de MongoDB, los datos quedarían repartidos entre instancias:

- **Duplicación:** un mismo PDF subido varias veces podría guardarse una vez por instancia. El índice único del checksum no lo evita, porque cada instancia tiene su propio índice.
- **Fragmentación:** una consulta solo vería los datos de la instancia a la que llegó.

Hasta ahora no estaba definido quién es dueño de esa instancia: el compose de `pdf-persistance` o `pdf-infra`. Hace falta una frontera explícita entre **operar** la base y **usar** la base.

## Decisión

### 1. `pdf-infra` opera la instancia de MongoDB

`pdf-infra` declara y mantiene la instancia: imagen con versión fijada, volumen, red, credenciales, límites de recursos y healthcheck. Hay **una sola instancia**, compartida por todas las réplicas de `pdf-persistance`.

Si el despliegue migra a k3s, la responsabilidad sigue en `pdf-infra`: cambia el formato del manifiesto, no la frontera.

### 2. `pdf-persistance` es dueño del modelo de datos

`pdf-persistance` define las colecciones, el esquema, los índices (incluido el índice único del checksum) y las consultas. Crea los índices de forma idempotente al arrancar: crear un índice con la misma definición que ya existe no tiene efecto, así que varias réplicas arrancando a la vez no se pisan.

Que el servicio cree sus propios índices evita que un cambio de esquema obligue a coordinar dos repos.

### 3. Contrato de conexión

`pdf-persistance` recibe la ubicación de la base por la variable de entorno `MONGO_URL` (credenciales incluidas), sin valor por defecto, y no asume host ni topología. Pasar de una instancia única a un replica set es un cambio de URL, sin tocar código (los servicios de apoyo son recursos adjuntos, factor IV de 12-factor).

### 4. Credenciales

Las credenciales se definen en `pdf-infra`, fuera del control de versiones (archivo `.env` ignorado por git), y se inyectan como variables de entorno. Ningún repo de servicio contiene credenciales reales.

### 5. Una sola instancia: escalan las réplicas, no la base

En esta etapa MongoDB no se escala horizontalmente. Escalan las réplicas de `pdf-persistance`, todas apuntando a la misma instancia. El replica set queda como evolución futura, posible gracias al contrato del punto 3.

### 6. Desarrollo aislado

El compose de `pdf-persistance` no declara un MongoDB en el stack integrado. Si hace falta uno para desarrollar el servicio de forma aislada, se define bajo un `profile` (`dev`) que no se activa por defecto.

### Ejemplo — servicio en `pdf-infra`

```yaml
services:
  mongo:
    image: mongo:<versión fija>
    volumes:
      - mongo_data:/data/db
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - fast_pdf_network

volumes:
  mongo_data:

networks:
  fast_pdf_network:
    external: true
```

## Consecuencias

**Positivas:**
- Hay una única fuente de datos, así que el índice único del checksum garantiza que N réplicas de `pdf-persistance` no dupliquen documentos.
- La frontera es clara: `pdf-infra` decide cómo se ejecuta la base y `pdf-persistance` decide qué se guarda. Cambiar la versión de MongoDB o su topología no toca el código del servicio.
- Las credenciales viven en un solo lugar.

**Negativas / a tener en cuenta:**
- Una única instancia es un punto único de falla y un límite de capacidad de escritura. Es aceptable con el volumen actual y debe revisarse si crece.
- En el stack integrado, `pdf-persistance` depende de que `pdf-infra` haya levantado MongoDB. Desarrollar el servicio solo requiere activar el profile `dev`.
- Como el servicio crea los índices al arrancar, cambiar las opciones de un índice existente (misma clave, distintas opciones) falla en lugar de reemplazarlo, y requiere una migración manual.

## Alternativas descartadas

- **Una instancia de MongoDB por réplica o por servicio:** es exactamente la duplicación y fragmentación de datos que motiva este ADR.
- **Declarar MongoDB en el compose de `pdf-persistance`:** acopla el ciclo de vida de los datos al del servicio. Recrear o escalar el servicio podría afectar la base, y deja ambiguo quién la opera.
- **Replica set o sharding desde ahora:** agrega complejidad operativa sin una necesidad con la carga actual. El contrato por `MONGO_URL` permite migrar más adelante sin cambiar el servicio.

## Referencias

- ADR-0004: `pdf-persistance` es el único punto de contacto con la persistencia y `pdf-main` su único cliente.
- ADR-0005: el contenido persistido es Markdown, el formato canónico.
