# Contrato — pdf-extractor

Dos contratos distintos, no uno — ver ADR-0004/0005 para por qué coexisten.

## 1. Producción: Redis Streams (asíncrono)

`pdf-extractor` no expone HTTP en el camino de producción. Es consumer de `queue:extraction`, con consumer group (`XREADGROUP`/`XACK`).

### Job — `queue:extraction`

Publicado por `pdf-main`:

```json
{
  "pdf_id": "665f1a2b3c4d5e6f7a8b9c0d",
  "filename": "informe.pdf",
  "content_base64": "JVBERi0xLjQKJcOkw7zDtsO4Cg=="
}
```

### Resultado — `queue:extraction-results`

Publicado por `pdf-extractor`. `status` es `"done"` o `"failed"` — nunca `"pending"` acá (ese estado solo existe del lado de `pdf-persistence`, antes de que llegue este resultado).

**Caso éxito:**
```json
{
  "pdf_id": "665f1a2b3c4d5e6f7a8b9c0d",
  "status": "done",
  "content": "# Informe\n\nTexto extraído en Markdown..."
}
```

**Caso error** (`error` es un objeto RFC 9457 completo, no un string — a diferencia de como se persiste más tarde, ver `persistence.md`):
```json
{
  "pdf_id": "665f1a2b3c4d5e6f7a8b9c0d",
  "status": "failed",
  "error": {
    "type": "about:blank",
    "title": "Processing failed",
    "status": 422,
    "detail": "El PDF está corrupto o no se pudo leer",
    "instance": "/api/pdfs"
  }
}
```

`content` y `error` son mutuamente excluyentes y ambos opcionales en el wire (`omitempty` del lado de `pdf-main`): si `status` es `"done"`, el mensaje trae `content` y no trae `error`; si es `"failed"`, al revés.

## 2. Benchmark del TP: `POST /extract` (síncrono — desvío explícito de ADR-0004)

Este contrato **no** pasa por la cola. Es un adaptador aparte sobre el mismo núcleo de extracción, agregado solo para el TP de carga — ver ADR-0004 §separación núcleo/transporte.

**Request:**
```json
{
  "content_base64": "JVBERi0xLjQKJcOkw7zDtsO4Cg=="
}
```

**Response `200 OK`:**
```json
{
  "content": "# Informe\n\nTexto extraído en Markdown...",
  "page_count": 292
}
```

Esta misma forma es la que `pdf-main` reexpone tal cual en su propio `POST /extract` público (ver `public-api.md`) — es tanto el contrato interno `pdf-main → pdf-extractor` como, sin transformación, el contrato público que evalúa la cátedra.

**Cualquier código distinto de `200`** se trata como `503` del lado de `pdf-main` — ver reglas generales. No hay un `400` documentado para este endpoint puntual; si `pdf-extractor` necesita rechazar un PDF inválido en este camino síncrono, es un caso a coordinar, no a inventar del lado del servicio.
