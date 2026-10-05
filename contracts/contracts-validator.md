# Contrato — pdf-validator

Servicio interno (ADR-0004), sin ruta pública. Solo `pdf-main` lo llama.

## `POST /validate`

Clasifica el contenido subido (PDF o Markdown) y devuelve su checksum.

**Request:**
```json
{
  "filename": "informe.pdf",
  "content_base64": "JVBERi0xLjQKJcOkw7zDtsO4Cg=="
}
```

**Response `200 OK`:**
```json
{
  "original_format": "pdf",
  "checksum": "a94a8fe5ccb19ba61c4c0873d391e987982fbbd3"
}
```

`original_format` es siempre `"pdf"` o `"markdown"` — ningún otro valor.

**Response `400 Bad Request`** (contenido inválido — no es PDF ni Markdown, o supera el tamaño máximo): RFC 9457 (ADR-0001). El body no se parsea más allá del `detail` — no hace falta ninguna extensión particular.

**Cualquier otro código** (incluido un `422`, por ejemplo) se trata como `503` del lado de `pdf-main` — ver reglas generales.
