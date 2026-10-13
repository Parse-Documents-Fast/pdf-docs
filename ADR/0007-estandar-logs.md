# ADR-0007: Estándar de logs de aplicación (formato y mensajes)

**Estado:** Aceptado

## Contexto

El sistema tiene varios servicios en Go y Python, y cada uno escribe sus propios logs. `docker compose logs`  mezcla las líneas de todos los servicios en un solo flujo. Si cada servicio usa su propio formato, no se pueden seguir los eventos de un mismo PDF a través de los servicios ni distinguir un fallo real de una entrada inválida.

Hace falta un contrato de logs único, independiente del lenguaje, igual que ya existe uno para los errores HTTP (ADR-0001) y para los DTOs (ADR-0002).

Quedan fuera de alcance OpenTelemetry y las trazas distribuidas: el objetivo es un formato simple y sin dependencias nuevas.

## Decisión

### 1. Formato: un objeto JSON por línea, a stdout

- Cada evento es **una sola línea** con un objeto JSON, escrita a stdout. El servicio no escribe archivos de log: el entorno recoge stdout (factor XI de 12-factor).
- Los nombres de los campos van en `snake_case`, igual que en los DTOs (ADR-0002), sin importar el lenguaje del servicio.

### 2. Campos obligatorios

| Campo | Tipo | Descripción |
|---|---|---|
| `time` | string | Marca de tiempo RFC 3339 **en UTC**, por ejemplo `2026-10-13T14:02:11.482Z` |
| `level` | string | `DEBUG`, `INFO`, `WARN` o `ERROR` |
| `service` | string | Nombre del servicio (`pdf-main`, `pdf-validator`, ...), fijo durante toda la vida del proceso |
| `msg` | string | Descripción corta y estable del evento (ver sección 4) |

Cualquier otro dato va en campos adicionales, nunca dentro de `msg`.

### 3. Campos adicionales estándar

| Campo | Cuándo se usa |
|---|---|
| `pdf_id` | Siempre que exista. Es la clave de correlación entre servicios (ADR-0004) |
| `checksum` | Cuando todavía no existe `pdf_id` (por ejemplo, durante la validación) |
| `size_bytes` | Tamaño de un PDF o de un mensaje |
| `duration_ms` | Duración de una operación |
| `http_method`, `http_path`, `http_status` | Peticiones HTTP, entrantes o salientes |
| `peer_service` | Servicio con el que se estaba hablando |
| `stream`, `consumer_group`, `consumer_name` | Eventos de Redis Streams |
| `reason` | Motivo de un rechazo de entrada |
| `error` | Texto del error, cuando una operación falló |

Un mismo concepto siempre usa el mismo nombre de campo. Se pueden agregar campos nuevos, pero no redefinir los de esta tabla.

### 4. Reglas para `msg`

1. Va **en inglés**, igual que el código y los identificadores. La documentación sigue en español.
2. Minúsculas, sin punto final y sin emojis.
3. Una frase corta en pasado que dice qué ocurrió: `pdf received`, `validation failed`.
4. **Es estable:** el mismo evento siempre usa el mismo `msg`. Los datos variables (IDs, tamaños, nombres) van en campos y nunca se interpolan en el texto. Así se puede contar y filtrar por `msg`.
5. Un evento es una línea. Los stack traces van dentro del campo `error`, con los saltos de línea escapados por JSON.

### 5. Niveles

| Nivel | Se usa para | Ejemplo |
|---|---|---|
| `DEBUG` | Detalle para diagnosticar. Apagado por defecto | rama de decisión elegida |
| `INFO` | Eventos normales del flujo | `pdf received`, `service started` |
| `WARN` | Algo inesperado que el servicio resuelve por sí mismo | entrada inválida rechazada, mensaje enviado a la DLQ |
| `ERROR` | Una operación falló y no se pudo resolver | `downstream call failed` tras los reintentos |

Regla práctica: una entrada inválida es `WARN` (el servicio funcionó bien al rechazarla). Un fallo del propio servicio o de una dependencia es `ERROR`.

### 6. Registrar o devolver, no ambas

Un error se registra en el punto donde se **maneja** (el handler que lo traduce a una respuesta RFC 9457, o el consumer que decide reintentar o enviar a la DLQ), no en cada capa por la que pasa. Si una capa devuelve el error hacia arriba, no lo registra además. Así un mismo fallo no aparece varias veces.

### 7. Qué no se registra

- **Contenido del PDF:** ni los bytes, ni el texto extraído, ni el Markdown generado, ni siquiera en `DEBUG` (restricción de RAM y privacidad, ADR-0002). Sí se pueden registrar `checksum`, `size_bytes` y `duration_ms`.
- **Credenciales:** contraseñas, tokens ni URLs de conexión con usuario y clave, como `MONGO_URL` (ADR-0006).

### 8. Configuración

El nivel se controla con la variable de entorno `LOG_LEVEL` (`debug`, `info`, `warn` o `error`, sin distinguir mayúsculas). El valor por defecto es `info`. Si el valor es inválido, el servicio usa `info` y emite un `WARN` al arrancar. Cambiar el nivel requiere reiniciar el contenedor, no recompilar.

### 9. Eventos comunes

Cuando aplique, los servicios usan estos `msg`, para que el mismo evento se llame igual en todos:

| `msg` | Nivel | Campos |
|---|---|---|
| `service started` | `INFO` | `http_addr`, `log_level` |
| `http request completed` | `INFO` (2xx y 3xx), `WARN` (4xx), `ERROR` (5xx) | `http_method`, `http_path`, `http_status`, `duration_ms` |
| `message received` | `INFO` | `stream`, `consumer_group`, `consumer_name`, `pdf_id` |
| `message acknowledged` | `DEBUG` | `stream`, `pdf_id` |
| `message sent to dlq` | `WARN` | `stream`, `pdf_id`, `reason` |
| `downstream call failed` | `ERROR` | `peer_service`, `http_status`, `duration_ms`, `error` |

### Ejemplo — salida

```json
{"time":"2026-10-13T14:02:11.482Z","level":"INFO","service":"pdf-main","msg":"pdf received","pdf_id":"665f1a2b3c4d5e6f7a8b9c0d","size_bytes":183422}
{"time":"2026-10-13T14:02:11.530Z","level":"WARN","service":"pdf-validator","msg":"validation failed","checksum":"a94a8fe5ccb19ba61c4c0873d391e987982fbbd3","reason":"invalid magic bytes"}
{"time":"2026-10-13T14:02:13.611Z","level":"ERROR","service":"pdf-main","msg":"downstream call failed","peer_service":"pdf-persistance","http_status":503,"duration_ms":2004,"error":"service unavailable"}
```

### Ejemplo — implementación en Go (`log/slog`, sin dependencias)

```go
level := new(slog.LevelVar) // INFO por defecto
if err := level.UnmarshalText([]byte(os.Getenv("LOG_LEVEL"))); err != nil {
	level.Set(slog.LevelInfo) // valor vacío o inválido: INFO (y WARN al arrancar si era inválido)
}

handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
	Level: level,
	ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
		if len(groups) == 0 && a.Key == slog.TimeKey {
			a.Value = slog.TimeValue(a.Value.Time().UTC()) // time siempre en UTC
		}
		return a
	},
})
logger := slog.New(handler).With("service", "pdf-main")

logger.Info("pdf received", "pdf_id", id, "size_bytes", n)
logger.Error("downstream call failed", "peer_service", "pdf-persistance", "error", err)
```

### Ejemplo — implementación en Python (`logging`, sin dependencias)

```python
import json, logging, os, sys, time


class JsonFormatter(logging.Formatter):
    def __init__(self, service: str):
        super().__init__()
        self.service = service

    def format(self, record: logging.LogRecord) -> str:
        entry = {
            "time": time.strftime("%Y-%m-%dT%H:%M:%S", time.gmtime(record.created))
                    + f".{int(record.msecs):03d}Z",
            "level": "WARN" if record.levelname == "WARNING" else record.levelname,
            "service": self.service,
            "msg": record.getMessage(),
        }
        entry.update(getattr(record, "fields", {}))
        if record.exc_info:
            entry["error"] = self.formatException(record.exc_info)
        return json.dumps(entry, ensure_ascii=False)


level = os.getenv("LOG_LEVEL", "INFO").upper()
if level not in ("DEBUG", "INFO", "WARN", "WARNING", "ERROR"):
    level = "INFO"  # y emitir un WARN al arrancar

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JsonFormatter("pdf-validator"))
logging.basicConfig(level=level, handlers=[handler])

logger = logging.getLogger(__name__)
logger.info("pdf received", extra={"fields": {"pdf_id": pdf_id, "size_bytes": n}})
```

## Consecuencias

**Positivas:**
- Un mismo PDF se sigue a través de todos los servicios filtrando por `pdf_id`, sin importar el lenguaje que lo registró. Por ejemplo: `docker compose logs --no-log-prefix | jq 'select(.pdf_id == "...")'`.
- Contar eventos o errores es trivial, porque `msg` es estable y los datos variables están en campos.
- No requiere bibliotecas externas: `log/slog` y `logging` están en la biblioteca estándar de cada lenguaje.
- Las marcas de tiempo en UTC se pueden ordenar entre servicios sin conversiones.

**Negativas / a tener en cuenta:**
- El JSON es menos legible a simple vista durante el desarrollo. Hay que usar `jq` o aceptar leer las líneas crudas.
- Los servicios en Python necesitan el `JsonFormatter` de unas veinte líneas. Hoy se copia en cada servicio, porque no hay una biblioteca compartida entre repos.
- La estabilidad de `msg` y los nombres de campos depende de la disciplina del equipo: ninguna herramienta la verifica. Renombrar un campo existente rompe las consultas que lo usen, mientras que agregar campos es gratis.
- Registrar una línea `http request completed` por petición aumenta el volumen de logs bajo carga. Con `LOG_LEVEL=warn` esas líneas se omiten, salvo las de 4xx y 5xx.

## Alternativas descartadas

- **Texto `clave=valor` (logfmt):** es más legible, pero cada lenguaje lo formatea y escapa de manera distinta, y filtrar requiere parsers a medida. JSON se procesa igual desde cualquier lenguaje y herramienta.
- **Mensajes de texto libre con los datos interpolados** (`"PDF 665f... recibido, 183422 bytes"`): es lo más rápido de escribir, pero obliga a usar expresiones regulares para filtrar o contar y cambia el texto en cada ocurrencia.
- **Bibliotecas de terceros** (`zap`, `structlog`): ofrecen más funciones, pero suman dependencias en cada repo para algo que la biblioteca estándar ya resuelve.
- **Mensajes en español:** coherentes con la documentación, pero el código, los identificadores y los campos están en inglés. Mezclar idiomas dentro de una misma línea de log empeora la búsqueda.
- **OpenTelemetry:** queda fuera del alcance de este proyecto. Si más adelante se adopta, los campos `pdf_id` y `service` se mapean directamente a sus atributos.

## Referencias

- ADR-0001: formato de errores HTTP (RFC 9457). El cuerpo de la respuesta es para el cliente y el log es para quien opera el servicio. Ambos pueden coincidir en `http_status`, pero no se reemplazan.
- ADR-0002: `snake_case` en el contrato y restricción de no manejar el contenido del PDF fuera de RAM.
- ADR-0004: `pdf_id` como clave de correlación entre servicios y nombres de los streams.
- ADR-0006: `MONGO_URL` contiene credenciales y no se registra.
