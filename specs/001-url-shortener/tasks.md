---
description: "Task list for Encurtador de Links (URL Shortener) implementation"
---

# Tasks: Encurtador de Links (URL Shortener)

**Input**: Design documents from `/specs/001-url-shortener/`

**Prerequisites**: [plan.md](./plan.md) (required), [spec.md](./spec.md) (required for user stories), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/http-api.md](./contracts/http-api.md)

**Tests**: Required by Project Constitution (Principle IV: mandatory xUnit unit tests for all business rules; Principle V: mandatory happy-path integration tests for every endpoint).

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Exact file paths are included in every description

## Path Conventions

- **API Project**: `src/UrlShortener.Api/`
- **Test Project**: `tests/UrlShortener.Tests/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization, solution layout, and justified NuGet dependencies

- [X] T001 Create .NET 10 solution and project structure (`UrlShortener.sln`, `src/UrlShortener.Api/UrlShortener.Api.csproj`, `tests/UrlShortener.Tests/UrlShortener.Tests.csproj`)
- [X] T002 Add justified EF Core NuGet packages (`Microsoft.EntityFrameworkCore`, `Microsoft.EntityFrameworkCore.Sqlite`, `Microsoft.EntityFrameworkCore.Design`) in `src/UrlShortener.Api/UrlShortener.Api.csproj`
- [X] T003 [P] Add justified test NuGet packages (`xunit`, `xunit.runner.visualstudio`, `Microsoft.NET.Test.Sdk`, `Microsoft.AspNetCore.Mvc.Testing`) and project reference in `tests/UrlShortener.Tests/UrlShortener.Tests.csproj`
- [X] T004 [P] Configure SQLite connection string (`ConnectionStrings:DefaultConnection` = `"Data Source=urlshortener.db"`) in `src/UrlShortener.Api/appsettings.json`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core domain entity, EF Core persistence with unique index, contracts, and DI configuration that MUST be complete before user stories

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [X] T005 Create `ShortenedLink` domain entity enforcing field constraints (`Id` Guid required Primary Key, `Code` string required fixed length `7` normalized lowercase, `OriginalUrl` string required max length `2048`, `ClickCount` long required default `0` non-negative `>= 0`, `ManagementKeyHash` string required length `64` SHA-256 hex digest, `CreatedAtUtc` DateTimeOffset required) in `src/UrlShortener.Api/Domain/ShortenedLink.cs`
- [X] T006 Create EF Core configuration `ShortenedLinkConfiguration` defining Primary Key `PK_ShortenedLinks`, Unique Index `IX_ShortenedLinks_Code` (`HasIndex(x => x.Code).IsUnique()`), `Code` fixed length `7` required, `OriginalUrl` max length `2048` required, `ManagementKeyHash` fixed length `64` required, and `ClickCount` default `0` in `src/UrlShortener.Api/Data/Configurations/ShortenedLinkConfiguration.cs`
- [X] T007 Implement `AppDbContext` exposing `DbSet<ShortenedLink> ShortenedLinks` and applying entity configurations from assembly in `src/UrlShortener.Api/Data/AppDbContext.cs`
- [X] T008 Create initial EF Core SQLite migration (`InitialCreate`) for `ShortenedLinks` table and `IX_ShortenedLinks_Code` unique index in `src/UrlShortener.Api/Data/Migrations/`
- [X] T009 [P] Create DTO records (`CreateLinkRequest`, `CreateLinkResponse`, `LinkStatsResponse`) and `LinkServiceResult<T>` outcome wrapper in `src/UrlShortener.Api/Application/LinkContracts.cs` and `src/UrlShortener.Api/Application/LinkServiceResult.cs`
- [X] T010 Define `ILinkService` application service interface (`CreateAsync`, `ResolveAndIncrementClickAsync`, `GetStatsAsync`) in `src/UrlShortener.Api/Application/ILinkService.cs`
- [X] T011 Configure DI container (`AppDbContext` with SQLite provider, `ILinkService`/`LinkService`, `AddProblemDetails`), automatic database migration on startup, endpoint mapping, and `public partial class Program` in `src/UrlShortener.Api/Program.cs`

**Checkpoint**: Foundation ready - user story implementation can now begin

---

## Phase 3: User Story 1 - Criar um Link Curto a partir de uma URL Longa (Priority: P1) 🎯 MVP

**Goal**: Permitir que o criador envie uma URL longa válida (`http`/`https`, até 2048 caracteres) e receba um link curto único de 7 caracteres alfanuméricos sem caracteres ambíguos e uma chave de gerenciamento cujo valor é persistido exclusivamente como hash SHA-256.

**Independent Test**: Enviar `POST /links` com uma URL válida e verificar retorno `201 Created` com código de 7 caracteres do alfabeto `abcdefghjkmnpqrstuvwxyz23456789`, `clickCount = 0` e `managementKey` (armazenada como hash SHA-256 no banco); enviar URLs inválidas (vazia, > 2048 caracteres, esquema `ftp://`) e verificar retorno `400 Bad Request` com `ProblemDetails` e mensagens em inglês.

### Tests for User Story 1 ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [X] T012 [P] [US1] Write xUnit unit tests for `LinkService.CreateAsync` covering URL validation (`"The destination URL is required."`, `"The destination URL must not exceed 2048 characters."` for 2049 chars vs valid 2048 chars, `"The destination URL must be a valid absolute HTTP or HTTPS URL."`), 7-char unambiguous alphabet `abcdefghjkmnpqrstuvwxyz23456789`, collision retry, independent code generation for duplicate URLs (`FR-001`), and SHA-256 `ManagementKeyHash` (length `64`, never storing plaintext `ManagementKey`) in `tests/UrlShortener.Tests/Unit/LinkServiceTests.cs`
- [X] T013 [P] [US1] Write integration tests for `POST /links` happy path (`201 Created`) and validation errors (`400 Bad Request` returning `ProblemDetails` with English messages) using `WebApplicationFactory<Program>` in `tests/UrlShortener.Tests/Integration/LinkEndpointsTests.cs`

### Implementation for User Story 1

- [X] T014 [US1] Implement `CreateAsync` in `LinkService` with URL validation (`Required`, max length `2048`, absolute `http`/`https` URI), 7-char code generator using `RandomNumberGenerator` over `abcdefghjkmnpqrstuvwxyz23456789`, pre-insert uniqueness check + `DbUpdateException` unique index retry, and SHA-256 hashing (`SHA256.HashData`) of `ManagementKey` in `src/UrlShortener.Api/Application/LinkService.cs`
- [X] T015 [US1] Implement `POST /links` endpoint in `LinkEndpoints` delegating to `ILinkService.CreateAsync` and returning `201 Created` or `400 Bad Request` (`TypedResults.ValidationProblem`) in `src/UrlShortener.Api/Endpoints/LinkEndpoints.cs`

**Checkpoint**: User Story 1 is fully functional and independently testable (`POST /links` + unit and integration tests passing)

---

## Phase 4: User Story 2 - Redirecionar Visitante e Contabilizar Acesso (Priority: P1)

**Goal**: Redirecionar qualquer visitante que acesse `GET /{code}` (de forma *case-insensitive*) para a URL original com redirecionamento temporário sem cache no cliente (`302 Found`) e incrementar `ClickCount` em `+1` via update atômico no banco de dados.

**Independent Test**: Criar um link curto, acessar `GET /{code}` (inclusive alternando maiúsculas/minúsculas no código) e verificar resposta `302 Found` apontando para `OriginalUrl` com `Cache-Control: no-store, no-cache`, incremento atômico do contador no banco e retorno `404 Not Found` para códigos inexistentes.

### Tests for User Story 2 ⚠️

- [X] T016 [P] [US2] Write xUnit unit tests for `LinkService.ResolveAndIncrementClickAsync` verifying case-insensitive lookup (`Abc1234` matches `abc1234`), atomic increment (`ClickCount` increases by `+1` per call and never decreases), and not-found result for unknown codes in `tests/UrlShortener.Tests/Unit/LinkServiceTests.cs`
- [X] T017 [P] [US2] Write integration tests for `GET /{code}` happy path (`302 Found` with `Location` header and `Cache-Control: no-store, no-cache`), case-insensitive redirect, persistence durability across scopes (`FR-009`), and `404 Not Found` (`ProblemDetails` with `"Short link not found."`) in `tests/UrlShortener.Tests/Integration/LinkEndpointsTests.cs`

### Implementation for User Story 2

- [X] T018 [US2] Implement `ResolveAndIncrementClickAsync` in `LinkService` normalizing `code` via `ToLowerInvariant()`, executing atomic database update via `ExecuteUpdateAsync(s => s.SetProperty(x => x.ClickCount, x => x.ClickCount + 1))`, and returning the target `OriginalUrl` in `src/UrlShortener.Api/Application/LinkService.cs`
- [X] T019 [US2] Implement `GET /{code}` endpoint in `LinkEndpoints` delegating to `ILinkService.ResolveAndIncrementClickAsync`, setting `Cache-Control: no-store, no-cache`, and returning `302 Found` redirect or `404 Not Found` `ProblemDetails` (`"Short link not found."`) in `src/UrlShortener.Api/Endpoints/LinkEndpoints.cs`

**Checkpoint**: User Stories 1 AND 2 work independently and together (link creation + redirect with atomic click counting)

---

## Phase 5: User Story 3 - Consultar Quantidade de Acessos de um Link (Priority: P2)

**Goal**: Permitir que o criador consulte `GET /links/{code}` (de forma *case-insensitive*) para visualizar `code`, `originalUrl`, `clickCount` e `createdAtUtc` sem alterar o contador de acessos e sem expor `ManagementKeyHash`.

**Independent Test**: Criar um link (`clickCount: 0`), realizar `N` redirecionamentos em `GET /{code}`, consultar `GET /links/{code}` repetidas vezes e confirmar retorno `200 OK` com `clickCount == N` inalterado pela consulta e `404 Not Found` para código inexistente.

### Tests for User Story 3 ⚠️

- [X] T020 [P] [US3] Write xUnit unit tests for `LinkService.GetStatsAsync` verifying case-insensitive code lookup, accurate `ClickCount` reporting after multiple redirects, zero side effects on `ClickCount` during stats queries, and not-found handling in `tests/UrlShortener.Tests/Unit/LinkServiceTests.cs`
- [X] T021 [P] [US3] Write integration tests for `GET /links/{code}` happy path (`200 OK` returning `LinkStatsResponse` with accurate `clickCount` before and after redirects) and `404 Not Found` (`ProblemDetails` with `"Short link not found."`) in `tests/UrlShortener.Tests/Integration/LinkEndpointsTests.cs`

### Implementation for User Story 3

- [X] T022 [US3] Implement `GetStatsAsync` in `LinkService` querying `ShortenedLinks.AsNoTracking()` by normalized lowercase `code` and projecting to `LinkStatsResponse` without modifying `ClickCount` in `src/UrlShortener.Api/Application/LinkService.cs`
- [X] T023 [US3] Implement `GET /links/{code}` endpoint in `LinkEndpoints` delegating to `ILinkService.GetStatsAsync` and returning `200 OK` (`LinkStatsResponse`) or `404 Not Found` `ProblemDetails` (`"Short link not found."`) in `src/UrlShortener.Api/Endpoints/LinkEndpoints.cs`

**Checkpoint**: All user stories (US1, US2, US3) are independently functional and covered by unit and integration tests

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Final architectural verification and end-to-end validation against quickstart guide

- [X] T024 Audit `src/UrlShortener.Api/Endpoints/LinkEndpoints.cs` and `src/UrlShortener.Api/Application/LinkService.cs` against Constitution Principles I–IX (no `AppDbContext` in endpoints, no business logic in HTTP handlers, English identifiers and error messages, `ProblemDetails` on all 400/404 responses)
- [X] T025 Run full test suite (`dotnet test`) in `UrlShortener.sln` and verify all scenarios in `specs/001-url-shortener/quickstart.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately (`T001` → `T002`, `T003`, `T004`).
- **Foundational (Phase 2)**: Depends on Setup completion (`T005` → `T006` → `T007` → `T008`, `T009`, `T010` → `T011`) — **BLOCKS all user stories**.
- **User Story 1 (Phase 3 - P1 🎯 MVP)**: Depends on Foundational (Phase 2).
- **User Story 2 (Phase 4 - P1)**: Depends on Foundational (Phase 2); integrates with links created by US1 in end-to-end HTTP flows.
- **User Story 3 (Phase 5 - P2)**: Depends on Foundational (Phase 2); inspects `ClickCount` accumulated by US2.
- **Polish (Phase 6)**: Depends on completion of Phases 3, 4, and 5.

### User Story Completion Graph

```text
Phase 1: Setup (T001–T004)
   │
   ▼
Phase 2: Foundational (T005–T011)
   │
   ├──► Phase 3: US1 - Criar Link Curto (P1 🎯 MVP) (T012–T015)
   │       │
   │       ├──► Phase 4: US2 - Redirecionar e Contar Acesso (P1) (T016–T019)
   │       │       │
   │       └───────┴──► Phase 5: US3 - Consultar Acessos (P2) (T020–T023)
   │                       │
   └───────────────────────┴──► Phase 6: Polish & Validation (T024–T025)
```

### Parallel Opportunities

- **Phase 1**: `T003` (`UrlShortener.Tests.csproj`) and `T004` (`appsettings.json`) can run in parallel with `T002`.
- **Phase 2**: `T009` (`LinkContracts.cs` & `LinkServiceResult.cs`) and `T010` (`ILinkService.cs`) can run in parallel with `T006`–`T008` once `T005` (`ShortenedLink.cs`) is created.
- **Within User Story Phases (US1, US2, US3)**: Unit test authoring (`T012`, `T016`, `T020` in `tests/UrlShortener.Tests/Unit/LinkServiceTests.cs`) and integration test authoring (`T013`, `T017`, `T021` in `tests/UrlShortener.Tests/Integration/LinkEndpointsTests.cs`) operate on separate files and can be written in parallel before implementing the corresponding service method and endpoint.

---

## Parallel Example: User Story 1

```bash
# Write Unit and Integration tests for User Story 1 in parallel:
Task T012: "Write xUnit unit tests for LinkService.CreateAsync in tests/UrlShortener.Tests/Unit/LinkServiceTests.cs"
Task T013: "Write integration tests for POST /links in tests/UrlShortener.Tests/Integration/LinkEndpointsTests.cs"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete **Phase 1: Setup** (`T001`–`T004`).
2. Complete **Phase 2: Foundational** (`T005`–`T011`).
3. Complete **Phase 3: User Story 1** (`T012`–`T015`).
4. **STOP and VALIDATE**: Run `dotnet test` to verify `POST /links` creation, validation `ProblemDetails`, unambiguous 7-char code generation, and SHA-256 `ManagementKeyHash`.

### Incremental Delivery

1. **Setup + Foundational** → Solution, EF Core SQLite schema with unique index, and DI ready.
2. **Add User Story 1 (`POST /links`)** → Links can be shortened and validated (MVP).
3. **Add User Story 2 (`GET /{code}`)** → Visitors are redirected and clicks are atomically counted.
4. **Add User Story 3 (`GET /links/{code}`)** → Creators can query click statistics at any time.
5. **Polish & Validation** → Final audit and `dotnet test` verification.

