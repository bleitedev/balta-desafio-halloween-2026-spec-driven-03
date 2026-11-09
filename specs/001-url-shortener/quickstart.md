# Quickstart & Validation Guide: Encurtador de Links (`001-url-shortener`)

**Date**: 2026-11-09  
**Spec**: [spec.md](./spec.md) | **Data Model**: [data-model.md](./data-model.md) | **Contract**: [contracts/http-api.md](./contracts/http-api.md)

## Prerequisites

- .NET 10 SDK (`dotnet --version` >= `10.0.100`)
- SQLite (embutido via provider `Microsoft.EntityFrameworkCore.Sqlite`, sem necessidade de servidor externo em desenvolvimento/testes)

## Build & Automated Tests

Execute a partir da raiz do repositório:

```powershell
# Restaurar pacotes e compilar a solução
dotnet build

# Executar todos os testes unitários (regras de negócio em LinkService) e de integração (endpoints)
dotnet test
```

**Expected Test Outcomes**:
- Todos os testes unitários em `UrlShortener.Tests/Unit/` passam (validação de URL `http`/`https`, limite de 2048 caracteres, alfabeto sem caracteres ambíguos de 7 posições, unicidade e retentativa de código, hash SHA-256 da chave de gerenciamento nunca persistida em texto puro, incremento atômico do contador).
- Todos os testes de integração em `UrlShortener.Tests/Integration/` passam (`POST /links`, `GET /{code}` e `GET /links/{code}`, além de validação `400 Bad Request` com `ProblemDetails`).

## Running the API Locally

```powershell
dotnet run --project src/UrlShortener.Api
```

O banco de dados SQLite (`urlshortener.db`) aplica as migrações automaticamente na inicialização, garantindo que os links criados continuem válidos após reiniciar a aplicação (`FR-009`).

## End-to-End Validation Scenarios

### Scenario 1: Criar um Link Curto (`POST /links`)

```powershell
curl -i -X POST http://localhost:5000/links `
  -H "Content-Type: application/json" `
  -d '{"originalUrl": "https://balta.io"}'
```

- **Resultado esperado**: Status `201 Created`, retorno JSON contendo `code` de 7 caracteres do alfabeto sem ambiguidades (`abcdefghjkmnpqrstuvwxyz23456789`), `clickCount: 0` e `managementKey` gerada.

### Scenario 2: Erro de Validação com `ProblemDetails` (`POST /links`)

```powershell
curl -i -X POST http://localhost:5000/links `
  -H "Content-Type: application/json" `
  -d '{"originalUrl": "ftp://invalid-scheme.com"}'
```

- **Resultado esperado**: Status `400 Bad Request`, cabeçalho `Content-Type: application/problem+json` e mensagem de erro em inglês conforme especificado em [contracts/http-api.md](./contracts/http-api.md).

### Scenario 3: Redirecionar e Incrementar Contador Atomicamente (`GET /{code}`)

```powershell
# Usando o código em letras maiúsculas ou minúsculas (case-insensitive)
curl -i http://localhost:5000/k9m2x7p
```

- **Resultado esperado**: Status `302 Found`, cabeçalho `Location: https://balta.io` e incremento imediato de `+1` no banco de dados.

### Scenario 4: Consultar Estatísticas de Acessos (`GET /links/{code}`)

```powershell
curl -i http://localhost:5000/links/k9m2x7p
```

- **Resultado esperado**: Status `200 OK` com `clickCount` refletindo o total exato de redirecionamentos realizados, sem que a própria consulta altere o valor.

