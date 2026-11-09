# Phase 0 Research: Encurtador de Links (`001-url-shortener`)

**Date**: 2026-11-09  
**Feature**: [spec.md](./spec.md)

## Architectural & Technical Decisions

### 1. Unicidade do Código Curto (Serviço + Índice Único no Banco)

- **Decision**: Garantir a unicidade do código curto de 7 caracteres em duas camadas complementares:
  1. Verificação prévia no `LinkService` durante a geração do código (com retentativa automática caso o código já exista).
  2. Índice único (`UNIQUE INDEX`) configurado via EF Core na coluna `Code` (`HasIndex(x => x.Code).IsUnique()`), com normalização canônica (lowercase) para assegurar unicidade *case-insensitive* e tratamento de exceção de violação de chave única em cenários de concorrência.
- **Rationale**: Apenas checar a existência no serviço (`SELECT` antes de `INSERT`) está sujeito a condições de corrida (*race conditions* / TOCTOU) quando duas requisições concorrentes geram o mesmo código simultaneamente. O índice único no banco de dados atua como barreira transacional definitiva de integridade, enquanto a checagem no serviço resolve colisões comuns sem precisar abortar a transação na maioria das vezes.
- **Alternatives considered**:
  - *Apenas checagem em memória/serviço sem índice único*: Rejeitado porque permite códigos duplicados sob requisições paralelas.
  - *Sequência numérica auto-incrementada convertida em Base62*: Rejeitado porque expõe a ordem sequencial previsível de criação dos links e acopla a geração do código ao gerador de IDs do banco.

---

### 2. Incremento do Contador de Acessos via Update Atômico

- **Decision**: Incrementar o contador `ClickCount` exclusivamente por meio de uma instrução de atualização atômica no banco de dados via EF Core LINQ (`ExecuteUpdateAsync(s => s.SetProperty(x => x.ClickCount, x => x.ClickCount + 1))`), nunca lendo a entidade para a memória da aplicação para somar e reescrever (`entity.ClickCount++` + `SaveChangesAsync`).
- **Rationale**: O padrão *read-modify-write* na aplicação sofre de *lost updates* (atualizações perdidas) quando dois ou mais cliques ocorrem ao mesmo tempo para o mesmo código curto (ambas as threads leem `N` e gravam `N + 1` em vez de `N + 2`). O `ExecuteUpdateAsync` do EF Core delega o incremento atômico diretamente ao motor de armazenamento (`SET ClickCount = ClickCount + 1`), preservando a invariante de que o contador nunca diminui nem perde cliques concorrentes, além de respeitar o Princípio III da Constituição (sem SQL manual espalhado pela aplicação).
- **Alternatives considered**:
  - *Carregar a entidade com `FindAsync`, fazer `link.ClickCount++` e `SaveChangesAsync`*: Rejeitado por perda de cliques em concorrência e overhead desnecessário de round-trip.
  - *SQL puro (`ExecuteSqlRawAsync`)*: Rejeitado pelo Princípio III da Constituição ("sem SQL espalhado pela aplicação, provider trocável"), já que `ExecuteUpdateAsync` é tipado, agnóstico de provider e traduzido nativamente pelo EF Core tanto para SQLite quanto para PostgreSQL.

---

### 3. Chave de Gerenciamento Armazenada como Hash (Nunca em Texto Puro)

- **Decision**: Gerar uma chave secreta de gerenciamento (`ManagementKey`) criptograficamente segura no momento da criação do link, retorná-la apenas uma única vez na resposta de criação ao criador e persistir no banco de dados exclusivamente o seu hash SHA-256 (`ManagementKeyHash`), nunca o valor em texto puro.
- **Rationale**: Em um sistema sem cadastro/login de usuários, a chave de gerenciamento representa a credencial de posse do criador sobre aquele link. Armazená-la em texto puro exporia todos os links caso o arquivo do banco (SQLite) ou um backup (PostgreSQL) fosse comprometido. Utilizando `SHA256.HashData` nativo do .NET (`System.Security.Cryptography`), o segredo original nunca é gravado em disco.
- **Alternatives considered**:
  - *Armazenar a chave em texto puro no banco*: Rejeitado por falha grave de segurança (exposição imediata de credenciais em caso de vazamento de leitura do banco).
  - *Criptografia simétrica reversível*: Rejeitado porque o servidor não precisa recuperar a chave original após a criação, apenas verificar a correspondência do hash quando necessário.

---

### 4. Alfabeto do Código Curto sem Caracteres Ambíguos

- **Decision**: Gerar o código de 7 caracteres utilizando `RandomNumberGenerator` sobre o alfabeto alfanumérico não ambíguo e normalizado em caixa baixa: `abcdefghjkmnpqrstuvwxyz23456789` (31 caracteres), excluindo visualmente ambíguos (`0`, `o`, `1`, `l`, `i`). Nas buscas e redirecionamentos, o código recebido é normalizado (`ToLowerInvariant()`) para garantir comportamento *case-insensitive*.
- **Rationale**: Links curtos são frequentemente compartilhados em imagens, apresentações, mensagens rápidas ou lidos em fontes onde `0` (zero) e `O`/`o` (letra O), ou `1` (um), `l` (ele minúsculo) e `I` (i maiúsculo) são indistinguíveis. Com 31 caracteres não ambíguos em 7 posições, o sistema mantém $31^7 = 27.512.614.111$ (~27,5 bilhões) de combinações únicas, eliminando erros de digitação humana e respeitando simultaneamente a regra de 7 caracteres alfanuméricos (`FR-004`) e a insensibilidade a maiúsculas/minúsculas.
- **Alternatives considered**:
  - *Alfabeto alfanumérico completo com `0, o, 1, l, i`*: Rejeitado por causar erros frequentes de leitura e transcrição pelo usuário final.
  - *Apenas dígitos numéricos (`0-9`)*: Rejeitado por reduzir drasticamente o espaço de códigos ($10^7 = 10$ milhões), aumentando a taxa de colisão.

---

### 5. Persistência com EF Core e Provider Trocável (SQLite → PostgreSQL)

- **Decision**: Concentrar o `AppDbContext` e o mapeamento Fluent API (`IEntityTypeConfiguration<ShortenedLink>`) na camada de infraestrutura (`Data/`), utilizando tipos padrão compatíveis entre SQLite e PostgreSQL e configurando o provider na inicialização (`Program.cs` / `DependencyInjection`) a partir da configuração (`ConnectionStrings:DefaultConnection` e `DatabaseProvider`). Nesta versão, o provider ativo é `Microsoft.EntityFrameworkCore.Sqlite`.
- **Rationale**: Cumpre o Princípio III da Constituição. O uso exclusivo de LINQ, `ExecuteUpdateAsync` e mapeamento via Fluent API sem tipos proprietários de banco permite trocar `UseSqlite(...)` por `UseNpgsql(...)` futuramente sem alterar nenhuma linha do `LinkService` ou dos endpoints.
- **Alternatives considered**:
  - *Dapper ou ADO.NET com queries SQL manuais*: Rejeitado diretamente pela Constituição (Princípio III).
  - *Banco em memória (`UseInMemoryDatabase`)*: Rejeitado porque viola `FR-009` ("os links continuam válidos depois que a aplicação reinicia") e não respeita restrições reais de índice único relacional.

