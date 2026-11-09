# Implementation Plan: Encurtador de Links (URL Shortener)

**Branch**: `001-url-shortener` | **Date**: 2026-11-09 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-url-shortener/spec.md`

## Summary

Implementação de uma API de encurtamento de links em **.NET 10 com C# e ASP.NET Core Minimal APIs**, organizada com um arquivo de endpoints por recurso (`LinkEndpoints.cs`) e uma camada de aplicação centralizada em `LinkService` (dona de todas as regras de negócio). A persistência utiliza **EF Core com SQLite** nesta versão, desenhada de forma desacoplada do provider para permitir troca direta para **PostgreSQL** futuramente.

O desenho arquitetural consolida quatro decisões técnicas fundamentais:
1. **Unicidade do código em dupla camada**: checagem prévia no `LinkService` combinada com índice único (`UNIQUE INDEX`) no banco de dados para blindar contra condições de corrida sob concorrência.
2. **Incremento do contador por update atômico**: uso de `ExecuteUpdateAsync` do EF Core (`SET ClickCount = ClickCount + 1`) diretamente no banco, nunca lendo e reescrevendo o contador na memória da aplicação.
3. **Chave de gerenciamento armazenada como hash**: geração de `ManagementKey` retornada apenas uma vez na criação e persistida exclusivamente como hash criptográfico SHA-256 (`ManagementKeyHash`), nunca em texto puro.
4. **Alfabeto do código sem caracteres ambíguos**: geração de códigos de 7 caracteres usando o conjunto alfanumérico de 31 caracteres `abcdefghjkmnpqrstuvwxyz23456789` (excluindo `0`, `o`, `1`, `l`, `i`), com busca *case-insensitive*.

## Technical Context

**Language/Version**: C# (.NET 10 / `net10.0`)

**Primary Dependencies**:
- `Microsoft.EntityFrameworkCore` (10.0.x) — ORM e abstração de persistência agnóstica de banco de dados (Princípio III).
- `Microsoft.EntityFrameworkCore.Sqlite` (10.0.x) — Provider EF Core para SQLite nesta versão, permitindo persistência durável em disco e troca futura para PostgreSQL (`Npgsql.EntityFrameworkCore.PostgreSQL`) via configuração (Princípio III).
- `Microsoft.EntityFrameworkCore.Design` (10.0.x) — Ferramental de geração de migrações do EF Core em tempo de desenvolvimento.

**Storage**: SQLite via EF Core nesta versão (provider trocável para PostgreSQL via configuração de injeção de dependência, sem SQL manual espalhado pela aplicação).

**Testing**:
- `xunit`, `xunit.runner.visualstudio`, `Microsoft.NET.Test.Sdk` — Framework de testes unitários e de integração exigido pelo Princípio IV da Constituição.
- `Microsoft.AspNetCore.Mvc.Testing` (10.0.x) — Hospedagem em memória (`WebApplicationFactory<Program>`) para testes de integração de caminho feliz dos endpoints Minimal API exigidos pelo Princípio V da Constituição.

**Target Platform**: Cross-platform (.NET 10 runtime — Windows / Linux / Containers)

**Project Type**: `web-service` (ASP.NET Core Minimal API)

**Performance Goals**: Redirecionamento (`GET /{code}`) e criação (`POST /links`) com baixa latência (< 50ms localmente) e zero perda de contagem em cliques simultâneos graças ao update atômico no banco.

**Constraints**:
- Endpoints nunca acessam `AppDbContext` diretamente; toda operação passa por `ILinkService` / `LinkService`.
- Nenhuma regra de negócio vive nos handlers HTTP de `LinkEndpoints.cs`.
- Erros de validação sempre retornam HTTP `400 Bad Request` com `ProblemDetails` (`application/problem+json`) e mensagens em inglês.
- Nomes de classes, arquivos, métodos, propriedades e mensagens de erro sempre em inglês.

**Scale/Scope**: 1 entidade de domínio (`ShortenedLink`), 1 serviço de aplicação (`LinkService`), 1 arquivo de endpoints para o recurso (`LinkEndpoints.cs` expondo 3 rotas: `POST /links`, `GET /{code}`, `GET /links/{code}`), suíte de testes unitários e de integração.

## Key Architectural Decisions & Justifications

| Decision | Justification |
|----------|---------------|
| **1. Unicidade do código garantida por índice único no banco + checagem no serviço** | Em ambientes concorrentes, apenas consultar o banco antes de inserir (`SELECT` seguido de `INSERT` no `LinkService`) permite que duas requisições paralelas gerem o mesmo código de 7 caracteres ao mesmo tempo (condição de corrida TOCTOU). A checagem no `LinkService` evita colisões rotineiras rapidamente, enquanto o índice único (`HasIndex(x => x.Code).IsUnique()`) no banco garante integridade transacional absoluta, provocando retentativa segura em caso de colisão concorrente. |
| **2. Contador incrementado por update atômico (`ExecuteUpdateAsync`)** | Ler a entidade na aplicação, fazer `link.ClickCount++` e salvar (`SaveChangesAsync`) sobrescreve incrementos concorrentes (*lost update*) e poderia violar a invariante de que nenhum acesso é perdido. Com `ExecuteUpdateAsync(s => s.SetProperty(x => x.ClickCount, x => x.ClickCount + 1))`, o EF Core gera uma instrução atômica no banco sem espalhar SQL puro pela aplicação, garantindo que o contador nunca diminua nem perca cliques simultâneos. |
| **3. Chave de gerenciamento armazenada como hash (SHA-256), nunca em texto puro** | A chave de gerenciamento emitida na criação do link é uma credencial sensível. Armazená-la em texto puro exporia o controle de todos os links caso o arquivo SQLite ou um backup do banco fosse lido indevidamente. Persistir apenas o hash SHA-256 (`ManagementKeyHash` via `SHA256.HashData`) protege o segredo em repouso sem exigir dependências externas. |
| **4. Alfabeto do código sem caracteres ambíguos (`abcdefghjkmnpqrstuvwxyz23456789`)** | A remoção de `0`, `o`, `1`, `l` e `i` evita erros humanos de leitura e digitação ao compartilhar links curtos, mantendo 31 caracteres alfanuméricos válidos ($31^7 \approx 27,5$ bilhões de combinações) e compatibilidade total com buscas *case-insensitive* (`ToLowerInvariant()`). |

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status (Pre-Phase 0) | Status (Post-Phase 1) | Verification / Evidence |
|-----------|----------------------|-----------------------|-------------------------|
| **I. Acesso a Dados Mediado por Serviço de Aplicação** | PASS | PASS | `LinkEndpoints.cs` injeta apenas `ILinkService`; `AppDbContext` é consumido exclusivamente por `LinkService`. |
| **II. Isolamento da Regra de Negócio** | PASS | PASS | Validação de URL, geração de código sem caracteres ambíguos, hash de chave e regras de redirecionamento/incremento vivem em `LinkService`. `LinkEndpoints.cs` apenas mapeia DTOs e status HTTP. |
| **III. Persistência Agnóstica com EF Core** | PASS | PASS | `AppDbContext` e `ShortenedLinkConfiguration` usam EF Core Fluent API e LINQ (`ExecuteUpdateAsync`), com SQLite configurado de forma isolada e pronta para troca por PostgreSQL. Zero SQL espalhado pela aplicação. |
| **IV. Testes Unitários Obrigatórios (xUnit)** | PASS | PASS | Projeto `UrlShortener.Tests` planejado com `LinkServiceTests.cs` cobrindo todas as regras de negócio no `LinkService`. |
| **V. Testes de Integração de Caminho Feliz por Endpoint** | PASS | PASS | Projeto `UrlShortener.Tests` planejado com `LinkEndpointsTests.cs` usando `WebApplicationFactory<Program>` para os 3 endpoints (`POST /links`, `GET /{code}`, `GET /links/{code}`). |
| **VI. Padronização de Erros de Validação (`ProblemDetails`)** | PASS | PASS | Erros de validação do `LinkService` são convertidos em `TypedResults.ValidationProblem(...)` retornando `400 Bad Request` com `application/problem+json`. |
| **VII. Controle Estrito de Dependências** | PASS | PASS | Todas as dependências NuGet (`EF Core Sqlite/Design`, `xUnit`, `Mvc.Testing`) estão justificadas na seção *Technical Context* deste plano; zero pacotes supérfluos. |
| **VIII. Código e Mensagens de Erro em Inglês** | PASS | PASS | Nomes de projetos, classes, arquivos, propriedades e mensagens de erro de validação/not-found definidos integralmente em inglês em `data-model.md` e `contracts/http-api.md`. |
| **IX. Commits Semânticos Rastreáveis** | PASS | PASS | Fluxo de commits seguirá `tipo(escopo): descrição (ID-REQUISITO)`. |

## Project Structure

### Documentation (this feature)

```text
specs/001-url-shortener/
├── plan.md              # Este arquivo (saída do comando /speckit-plan)
├── research.md          # Decisões técnicas e alternativas avaliadas (Phase 0)
├── data-model.md        # Modelo de dados, índices e regras de validação (Phase 1)
├── quickstart.md        # Guia de execução e validação ponta a ponta (Phase 1)
├── contracts/
│   └── http-api.md      # Contratos HTTP dos endpoints Minimal API (Phase 1)
└── tasks.md             # Lista de tarefas (Phase 2 - gerado por /speckit-tasks)
```

### Source Code (repository root)

```text
UrlShortener.sln
src/
└── UrlShortener.Api/
    ├── UrlShortener.Api.csproj
    ├── Program.cs
    ├── appsettings.json
    ├── Domain/
    │   └── ShortenedLink.cs
    ├── Application/
    │   ├── ILinkService.cs
    │   ├── LinkService.cs
    │   ├── LinkContracts.cs
    │   └── LinkServiceResult.cs
    ├── Data/
    │   ├── AppDbContext.cs
    │   ├── Configurations/
    │   │   └── ShortenedLinkConfiguration.cs
    │   └── Migrations/
    └── Endpoints/
        └── LinkEndpoints.cs
tests/
└── UrlShortener.Tests/
    ├── UrlShortener.Tests.csproj
    ├── Unit/
    │   └── LinkServiceTests.cs
    └── Integration/
        └── LinkEndpointsTests.cs
```

**Structure Decision**: Estrutura enxuta em solução .NET 10 com um projeto principal `src/UrlShortener.Api` (organizado internamente nas pastas `Domain`, `Application`, `Data` e `Endpoints` com um único arquivo `LinkEndpoints.cs` para o recurso de links) e um projeto de testes `tests/UrlShortener.Tests` separado em `Unit/` e `Integration/`. Essa estrutura evita complexidade desnecessária de múltiplos `.csproj` enquanto mantém separação rigorosa de responsabilidades conforme os Princípios I, II e III da Constituição.

## Complexity Tracking

Nenhuma violação à Constituição. Todos os 9 princípios são integralmente atendidos pela arquitetura proposta.
