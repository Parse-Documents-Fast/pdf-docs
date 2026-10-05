# Contratos entre servicios

Esta carpeta documenta, en JSON concreto, el contrato real que `pdf-main` espera de cada servicio downstream — generado a partir de `internal/dto` e `internal/clients` de `pdf-main`, que es la fuente de verdad (único servicio que habla HTTP con los demás). Cada archivo es el contrato de un servicio.

## Reglas generales, válidas para todos

- **Content-Type**: toda request con body va `application/json`. Requests sin body (`GET`, `DELETE`) no llevan ese header.
- **Binarios**: siempre base64 dentro de un campo string del JSON (ADR-0002) — nunca multipart, nunca binario crudo en el body.
- **Solo los status codes documentados en cada contrato tienen significado especial.** Cualquier otro código que devuelva un servicio — aunque sea un 4xx de negocio legítimo no listado acá — `pdf-main` lo trata igual que un 503: lo mapea a `ErrDownstream` y el cliente final recibe "Service unavailable". Si un servicio necesita comunicar un caso nuevo, hay que sumarlo acá primero, coordinado con `pdf-main` — no alcanza con que el servicio lo devuelva por su cuenta.
- **Circuit breaker**: solo errores de transporte (timeout, conexión rechazada) y respuestas `5xx` cuentan como falla y pueden abrir el breaker. Un `4xx` documentado (sea el esperado o no) nunca abre el breaker — se trata como una respuesta válida del servicio, no como una falla de infraestructura.
- **Errores**: todo error expuesto sigue RFC 9457 (ADR-0001), salvo donde se aclare lo contrario.

## Servicios

- [`validator.md`](./validator.md) — `pdf-validator`
- [`converter.md`](./converter.md) — `pdf-converter`
- [`extractor.md`](./extractor.md) — `pdf-extractor` (síncrono + cola)
- [`persistence.md`](./persistence.md) — `pdf-persistence`
- [`public-api.md`](./public-api.md) — API pública de `pdf-main` (lo que consumen `pdf-web`/CLI)
