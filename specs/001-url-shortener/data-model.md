# Phase 1 Data Model: Encurtador de Links (`001-url-shortener`)

**Date**: 2026-11-09  
**Feature**: [spec.md](./spec.md) | **Research**: [research.md](./research.md)

## Entities

### `ShortenedLink`

Representa um link encurtado persistido pelo sistema, contendo o código curto único, a URL original de destino, o contador monotônico de acessos e o hash da chave de gerenciamento.

| Field | C# Type | Persistence Constraints | Description |
|-------|---------|-------------------------|-------------|
| `Id` | `Guid` | Primary Key (`PK_ShortenedLinks`), required | Identificador interno único do registro. |
| `Code` | `string` | Required, fixed length `7`, Unique Index (`IX_ShortenedLinks_Code`), case-insensitive (stored normalized in lowercase) | Código curto de 7 caracteres alfanuméricos sem caracteres ambíguos (`abcdefghjkmnpqrstuvwxyz23456789`). |
| `OriginalUrl` | `string` | Required, max length `2048` | URL absoluta de destino (`http://` ou `https://`). |
| `ClickCount` | `long` | Required, default `0`, non-negative (`>= 0`) | Total acumulado de redirecionamentos realizados. Atualizado exclusivamente via update atômico (`ClickCount + 1`). |
| `ManagementKeyHash` | `string` | Required, length `64` (SHA-256 hex digest) | Hash criptográfico SHA-256 da chave de gerenciamento gerada na criação (nunca armazenada em texto puro). |
| `CreatedAtUtc` | `DateTimeOffset` | Required | Data e hora UTC da criação do link curto. |

## Indexes & Database Integrity Rules

1. **Primary Key**: `Id` (`Guid`).
2. **Unique Index on `Code`**:
   - Nome: `IX_ShortenedLinks_Code`
   - Propriedade: `IsUnique(true)`
   - Garantia de unicidade sob concorrência: O `LinkService` normaliza `Code` para minúsculas antes de consultar ou persistir. Caso duas requisições simultâneas tentem inserir o mesmo `Code`, o índice único do banco rejeita a duplicata (`DbUpdateException`), permitindo ao `LinkService` capturar a colisão e gerar um novo código de forma segura.
3. **Atomic Counter Invariant**:
   - `ClickCount` inicia sempre em `0` na criação.
   - Nenhum método de domínio ou aplicação permite atribuir um valor arbitrário ou decrementar `ClickCount`.
   - O incremento ocorre no banco via `ExecuteUpdateAsync(s => s.SetProperty(x => x.ClickCount, x => x.ClickCount + 1))`.

## Validation Rules (Domain / `LinkService`)

Todas as mensagens de erro de validação são emitidas em **inglês** (Princípio VIII da Constituição) e retornadas na camada HTTP como `400 Bad Request` no formato `ProblemDetails` (Princípio VI da Constituição):

- **`OriginalUrl`**:
  - Não pode ser nula, vazia ou conter apenas espaços: `"The destination URL is required."`
  - Deve ter no máximo 2048 caracteres: `"The destination URL must not exceed 2048 characters."`
  - Deve ser uma URI absoluta válida com esquema `http` ou `https`: `"The destination URL must be a valid absolute HTTP or HTTPS URL."`
- **`Code` (Geração)**:
  - Gerado internamente pelo `LinkService` com exatamente 7 caracteres pertencentes ao alfabeto sem ambiguidades `abcdefghjkmnpqrstuvwxyz23456789`.
- **`ManagementKey` (Geração e Hash)**:
  - Gerada internamente como token aleatório seguro, devolvida apenas no DTO de resposta da criação (`CreateLinkResponse`) e persistida na entidade `ShortenedLink` apenas como `ManagementKeyHash` (SHA-256 hex de 64 caracteres).

## State Transitions

O ciclo de vida de `ShortenedLink` é linear e imutável quanto ao destino:

```text
[Novo Pedido de Encurtamento]
        │
        ▼ (Validação OK + Código 7 chars único + Hash da ManagementKey)
┌───────────────────────────┐
│ Criado (ClickCount = 0)   │
└─────────────┬─────────────┘
              │
              ▼ (Cada acesso de redirecionamento: Update Atômico ClickCount = ClickCount + 1)
┌───────────────────────────┐
│ Ativo (ClickCount = N)    │◄───┐
└─────────────┬─────────────┘    │ Redirecionamento (+1 atômico)
              └──────────────────┘ (Consultas de estatísticas não alteram o estado)
```

