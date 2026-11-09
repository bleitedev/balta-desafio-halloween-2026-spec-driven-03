# HTTP API Contract: Encurtador de Links (`001-url-shortener`)

**Date**: 2026-11-09  
**Resource Endpoint File**: `src/UrlShortener.Api/Endpoints/LinkEndpoints.cs` (um arquivo de endpoints por recurso)  
**Error Format**: RFC 9457 / RFC 7807 `application/problem+json` (`ProblemDetails` / `HttpValidationProblemDetails`) com mensagens em inglês.

---

## 1. Create Shortened Link (`FR-001`, `FR-002`, `FR-003`, `FR-004`)

Cria um novo link curto a partir de uma URL de destino válida, gerando um código único de 7 caracteres alfanuméricos sem caracteres ambíguos e uma chave de gerenciamento (cujo valor em texto puro é retornado apenas nesta resposta e armazenado exclusivamente como hash SHA-256).

- **Method & Path**: `POST /links`
- **Content-Type**: `application/json`

### Request Body (`CreateLinkRequest`)

```json
{
  "originalUrl": "https://balta.io/carreiras/desenvolvedor-dotnet"
}
```

### Responses

#### `201 Created`
- **Headers**: `Location: /links/k9m2x7p`
- **Body (`CreateLinkResponse`)**:

```json
{
  "code": "k9m2x7p",
  "shortUrl": "http://localhost:5000/k9m2x7p",
  "originalUrl": "https://balta.io/carreiras/desenvolvedor-dotnet",
  "clickCount": 0,
  "createdAtUtc": "2026-11-09T13:00:00Z",
  "managementKey": "mkey_8f4b2e9c1a7d6f3b5e0c9a2d4f6b8e1c"
}
```

#### `400 Bad Request` (`application/problem+json`)
Retornado sempre que `originalUrl` for nula/vazia, ultrapassar 2048 caracteres ou não for uma URI absoluta `http`/`https`.

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "originalUrl": [
      "The destination URL must be a valid absolute HTTP or HTTPS URL."
    ]
  }
}
```

---

## 2. Redirect by Short Code (`FR-004`, `FR-005`, `FR-006`, `FR-008`)

Busca o link curto pelo código de 7 caracteres (*case-insensitive*), incrementa o contador `ClickCount` em `+1` via update atômico no banco de dados e redireciona temporariamente o visitante para a URL original.

- **Method & Path**: `GET /{code}`
- **Route Parameter**: `code` (string, 7 caracteres alfanuméricos, *case-insensitive*)

### Responses

#### `302 Found` (Temporary Redirect)
- **Headers**:
  - `Location: https://balta.io/carreiras/desenvolvedor-dotnet`
  - `Cache-Control: no-store, no-cache`

#### `404 Not Found` (`application/problem+json`)
Retornado quando o código informado não existe no sistema.

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.5",
  "title": "Not Found",
  "status": 404,
  "detail": "Short link not found."
}
```

---

## 3. Get Link Statistics (`FR-007`, `FR-008`)

Consulta os dados e o total acumulado de acessos (`clickCount`) de um link curto pelo seu código (*case-insensitive*), sem alterar o contador de cliques e sem expor a chave de gerenciamento ou seu hash.

- **Method & Path**: `GET /links/{code}`
- **Route Parameter**: `code` (string, 7 caracteres alfanuméricos, *case-insensitive*)

### Responses

#### `200 OK`
- **Body (`LinkStatsResponse`)**:

```json
{
  "code": "k9m2x7p",
  "originalUrl": "https://balta.io/carreiras/desenvolvedor-dotnet",
  "clickCount": 3,
  "createdAtUtc": "2026-11-09T13:00:00Z"
}
```

#### `404 Not Found` (`application/problem+json`)
Retornado quando o código informado não existe no sistema.

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.5",
  "title": "Not Found",
  "status": 404,
  "detail": "Short link not found."
}
```

