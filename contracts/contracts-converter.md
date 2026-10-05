# Contrato — pdf-converter

Servicio interno (ADR-0004/0005), sin ruta pública. Solo descarga: Markdown → PDF. Siempre HTTP síncrono — sin contrato de cola/ingesta.

## `POST /convert`

**Request:**
```json
{
  "content": "# Título\n\nTexto en Markdown..."
}
```

**Response `200 OK`:**
```json
{
  "content_base64": "JVBERi0xLjQKJcOkw7zDtsO4Cg==",
  "mime_type": "application/pdf"
}
```

**Cualquier otro código** (no solo 5xx — `pdf-main` no distingue ningún 4xx particular para este endpoint) se trata como `503` — ver reglas generales. Si `pdf-converter` necesita reportar, por ejemplo, "Markdown inválido para convertir", hay que coordinar con `pdf-main` un código y un caso explícito antes de implementarlo — hoy no existe ese camino en el cliente.
