# Contrato — API pública de pdf-main

Lo que consumen `pdf-web` y el CLI. Única superficie HTTP pública del sistema (detrás de Traefik).

## `PdfSummary` — forma resumida

```json
{
  "id": "665f1a2b3c4d5e6f7a8b9c0d",
  "title": "Informe mensual",
  "original_format": "pdf",
  "checksum": "a94a8fe5ccb19ba61c4c0873d391e987982fbbd3",
  "status": "pending",
  "created_at": "2026-10-05T14:30:00Z"
}
```

## `PdfDocument` — forma completa (`PdfSummary` + contenido)

```json
{
  "id": "665f1a2b3c4d5e6f7a8b9c0d",
  "title": "Informe mensual",
  "original_format": "pdf",
  "checksum": "a94a8fe5ccb19ba61c4c0873d391e987982fbbd3",
  "status": "done",
  "created_at": "2026-10-05T14:30:00Z",
  "content": "# Informe\n\nTexto extraído...",
  "error": null
}
```

`content`/`error` siempre presentes (`null` si no aplica) — mismo comportamiento que `PersistRecord`, ver `persistence.md`.

## Endpoints

- **`POST /api/pdfs`** → `200` con `PdfSummary` (`status: "pending"` para PDF, `"done"` para Markdown). `400` inválido, `409` duplicado (con `existing_id` como extensión RFC 9457), `503` si algún downstream falla.
- **`GET /api/pdfs`** → `200`, array de `PdfSummary`.
- **`GET /api/pdfs/{id}`** → `200` con `PdfDocument`. `404` si no existe.
- **`DELETE /api/pdfs/{id}`** → `204`. `404` si no existe.
- **`GET /api/pdfs/{id}/download?format=markdown|pdf`** → el archivo. `format` default `markdown`. `409` si `status` es `pending` (`ErrNotReady`), `422` si `status` es `failed` (`ErrFailed`), `400` si `format` es inválido.
- **`POST /extract`** (benchmark del TP, no forma parte del flujo normal) → `200` con `{"content": "...", "page_count": N}`. Ver `extractor.md` — es el mismo shape que el contrato interno `pdf-main → pdf-extractor`, reexpuesto sin cambios.

Todos los errores, RFC 9457 (ADR-0001).
