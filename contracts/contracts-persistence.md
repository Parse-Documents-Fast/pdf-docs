# Contrato — pdf-persistence

Servicio interno (ADR-0004), sin ruta pública. Solo `pdf-main` lo llama.

## Forma del registro (`PersistRecord`)

Es la forma que devuelven `create`/`get`/`list`/`find-by-checksum`/`update`. **`content` y `error` siempre están presentes en el JSON — como `null` cuando no aplican, nunca ausentes.** Esto es distinto a como viajan en otros contratos (ver nota de `omitempty` al final).

```json
{
  "id": "665f1a2b3c4d5e6f7a8b9c0d",
  "title": "Informe mensual",
  "original_format": "pdf",
  "checksum": "a94a8fe5ccb19ba61c4c0873d391e987982fbbd3",
  "status": "pending",
  "content": null,
  "error": null,
  "created_at": "2026-10-05T14:30:00Z"
}
```

`status` es uno de `"pending"`, `"done"`, `"failed"`.

## `POST /documents` — crear

**Request** (ejemplo PDF: se crea `pending`, sin contenido todavía — nótese que acá `content` sí está *ausente*, no `null`):
```json
{
  "title": "Informe mensual",
  "original_format": "pdf",
  "checksum": "a94a8fe5ccb19ba61c4c0873d391e987982fbbd3",
  "status": "pending"
}
```

**Request** (ejemplo Markdown: se persiste `done` directo, con contenido ya incluido):
```json
{
  "title": "Notas",
  "original_format": "markdown",
  "checksum": "b3a8d9...",
  "status": "done",
  "content": "# Notas\n\nTexto..."
}
```

**Response `201 Created`:** un `PersistRecord` (arriba).

**Response `409 Conflict`** (duplicado por checksum): el body no se parsea — `pdf-main` lo mapea a "duplicado" sin leer nada del contenido. Este caso es una red de seguridad ante condición de carrera; el duplicado normal ya se detecta antes, vía `by-checksum` (siguiente endpoint), que sí devuelve el registro existente completo con su `id`.

## `GET /documents/by-checksum?checksum=...` — detección de duplicados

**Response `200 OK`:** un `PersistRecord` si existe.
**Response `404 Not Found`:** no existe — body vacío, no se parsea.

## `GET /documents/{id}`

**Response `200 OK`:** un `PersistRecord`.
**Response `404 Not Found`:** RFC 9457.

## `GET /documents` — listar

**Response `200 OK`:** array de `PersistRecord`, `[]` si no hay ninguno (nunca `null`).

## `PATCH /documents/{id}` — actualizar (resultado de extracción)

Usado cuando llega un resultado de `queue:extraction-results`.

**Request** (caso éxito — `error` *ausente*, no `null`):
```json
{
  "status": "done",
  "content": "# Informe\n\nTexto extraído..."
}
```

**Request** (caso error — `content` *ausente*; `error` es un **string**, el `detail` del RFC 9457 original, no el objeto completo):
```json
{
  "status": "failed",
  "error": "El PDF está corrupto o no se pudo leer"
}
```

**Response `200 OK`:** el `PersistRecord` ya actualizado.
**Response `404 Not Found`:** RFC 9457.

## Nota sobre `omitempty` — leela antes de implementar en otro lenguaje

Los campos opcionales se comportan distinto según el contrato, y la diferencia importa:

- En **requests** (`POST /documents`, `PATCH /documents/{id}`), un campo opcional sin valor **no viaja en el JSON** — la clave no existe. Así están `PersistCreateRequest`/`PersistUpdateRequest` del lado de `pdf-main`.
- En **responses** (`PersistRecord`), `content` y `error` **siempre viajan**, como `null` cuando no hay valor — nunca ausentes.

Si implementás `pdf-persistence` en un lenguaje donde "ausente" y "`null`" se tratan distinto al deserializar (por ejemplo, un modelo Pydantic con `Optional[str] = None` por default trata ambos casos igual al leer, pero no necesariamente al escribir si no se configura explícitamente `exclude_none`), confirmá que tu serialización de salida respeta esto: los *requests* que recibís pueden venir sin la clave, pero lo que vos devolvés como *response* tiene que incluirla siempre, en `null` si no hay valor.
