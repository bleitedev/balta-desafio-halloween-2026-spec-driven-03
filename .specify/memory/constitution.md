# Encurtador Constitution

## Core Principles

### I. Acesso a Dados Mediado por Serviço de Aplicação (Application Service Mediation)
Os endpoints de Minimal APIs NUNCA DEVEM (MUST NOT) acessar o banco de dados ou o `DbContext`
diretamente. Toda interação de leitura ou escrita de dados a partir da camada HTTP DEVE (MUST)
ocorrer exclusivamente por meio de um serviço de aplicação dedicado.

**Racional**: Garantir o desacoplamento entre a camada de transporte HTTP e a camada de
persistência, facilitando testes, manutenção e evolução independente do contrato da API.

### II. Isolamento da Regra de Negócio (Domain Isolation from HTTP)
Nenhuma regra de negócio DEVE (MUST NOT) residir na definição do endpoint ou no handler HTTP.
Os handlers HTTP DEVEM (MUST) limitar-se a receber a requisição, acionar o serviço de aplicação
correspondente e mapear o resultado para a resposta HTTP apropriada.

**Racional**: Manter a lógica de domínio pura, reutilizável e testável sem dependência do
pipeline do ASP.NET Core ou do contexto HTTP.

### III. Persistência Agnóstica com EF Core (Provider-Agnostic EF Core Persistence)
Toda a camada de persistência DEVE (MUST) ser implementada utilizando Entity Framework Core com
provider facilmente substituível por configuração. Consultas em SQL puro (raw SQL) NÃO DEVEM
(MUST NOT) ser espalhadas pela aplicação; todo acesso e mapeamento de dados DEVE (MUST) utilizar
LINQ e as abstrações do EF Core concentradas na camada de infraestrutura/persistência.

**Racional**: Permitir a troca de banco de dados entre ambientes (ex.: desenvolvimento, testes e
produção) sem reescrita de consultas ou acoplamento a dialetos SQL específicos.

### IV. Testes Unitários Obrigatórios para Regras de Negócio (Mandatory xUnit Unit Testing)
Toda regra de negócio implementada no domínio ou nos serviços de aplicação DEVE (MUST) possuir
cobertura de testes unitários automatizados utilizando o framework xUnit, validando cenários de
sucesso, casos de borda e falhas esperadas.

**Racional**: Assegurar que o comportamento central do sistema seja verificado de forma rápida,
determinística e isolada de infraestrutura externa.

### V. Testes de Integração de Caminho Feliz por Endpoint (Endpoint Happy-Path Integration Testing)
Todo endpoint exposto pela API DEVE (MUST) possuir ao menos um teste de integração automatizado
que valide o seu caminho feliz (happy path) de ponta a ponta no pipeline HTTP.

**Racional**: Garantir que o roteamento, a serialização, a injeção de dependência e a integração
entre camadas funcionem corretamente para todos os contratos expostos.

### VI. Padronização de Erros de Validação com ProblemDetails (Standardized Validation ProblemDetails)
Qualquer erro de validação de entrada DEVE (MUST) retornar invariavelmente o status HTTP `400 Bad Request`
com o corpo formatado no padrão RFC 9457 / RFC 7807 (`ProblemDetails` / `HttpValidationProblemDetails`).
Respostas de erro em formatos arbitrários ou não padronizados NÃO DEVEM (MUST NOT) ser utilizadas.

**Racional**: Oferecer um contrato de erro previsível, estruturado e interoperável para todos os
consumidores da API.

### VII. Controle Estrito de Dependências (Strict Dependency Justification)
Nenhum pacote NuGet ou dependência externa nova DEVE (MUST NOT) ser adicionado ao projeto sem
uma justificativa técnica explícita documentada previamente no plano de implementação (`plan.md`).
Recursos nativos do .NET 10 e do ASP.NET Core DEVEM (MUST) ter preferência.

**Racional**: Evitar inchamento do projeto (bloat), reduzir superfície de ataque de segurança e
minimizar custos de manutenção e atualização de pacotes de terceiros.

### VIII. Código e Mensagens de Erro em Inglês (English Naming and Error Messages)
Todos os nomes de classes, interfaces, métodos, propriedades, variáveis, namespaces e arquivos de
código-fonte, bem como todas as mensagens de erro e validação retornadas pela API, DEVEM (MUST)
ser escritos em inglês.

**Racional**: Manter consistência linguística com as bibliotecas da plataforma .NET e com os
padrões internacionais de desenvolvimento de software.

### IX. Commits Semânticos Rastreáveis (Traceable Conventional Commits)
Toda mensagem de commit DEVE (MUST) seguir estritamente o formato `tipo(escopo): descrição <ID-REQUISITO>`
(ou referência equivalente ao identificador do requisito/tarefa no escopo ou descrição), associando
explicitamente a alteração ao requisito correspondente da especificação.

**Racional**: Garantir rastreabilidade ponta a ponta entre os requisitos especificados e o
histórico de alterações no controle de versão.

## Stack Tecnológica e Restrições Arquiteturais

- **Plataforma e Linguagem**: .NET 10 com C# e ASP.NET Core Minimal APIs.
- **Persistência**: Entity Framework Core com configuração desacoplada do provider de banco de dados.
- **Testes**: xUnit para testes unitários e de integração (`WebApplicationFactory` para endpoints).
- **Contratos de Erro**: Uso obrigatório de `ProblemDetails` nativo do ASP.NET Core para falhas de validação (HTTP 400).
- **Dependências**: Proibição de bibliotecas externas sem registro e aprovação prévia na seção de decisões/complexidade do `plan.md`.

## Fluxo de Desenvolvimento e Qualidade

- **Organização de Código**: Nomes de arquivos e tipos exclusivamente em inglês; separação clara entre Endpoints (camada HTTP), Application Services / Domain (regras de negócio) e Infrastructure/Data (EF Core).
- **Critérios de Aceite (Quality Gates)**: Nenhuma tarefa ou PR pode ser considerada concluída sem:
  1. Testes unitários em xUnit cobrindo todas as regras de negócio adicionadas ou alteradas.
  2. Ao menos um teste de integração de caminho feliz para cada endpoint criado ou modificado.
  3. Verificação de que endpoints não referenciam `DbContext` ou SQL diretamente e não contêm lógica de negócio.
  4. Respostas de validação retornando `400 Bad Request` com `ProblemDetails` e mensagens em inglês.
- **Controle de Versão**: Commits atômicos no padrão `tipo(escopo): descrição` contendo o identificador do requisito (ex.: `feat(shortener): create short url endpoint (FR-001)`).

## Governance

Esta Constituição sobrepõe-se a quaisquer outras práticas ou convenções informais do projeto.
Todas as especificações (`spec.md`), planos de arquitetura (`plan.md`), listas de tarefas (`tasks.md`)
e revisões de código DEVEM (MUST) verificar e atestar conformidade com os princípios aqui definidos.

- **Procedimento de Emenda**: Alterações nesta Constituição exigem documentação explícita da motivação,
  atualização deste arquivo via fluxo `/speckit-constitution` e revisão de impacto nos artefatos dependentes.
- **Política de Versionamento**: Esta Constituição segue Versionamento Semântico (`MAJOR.MINOR.PATCH`):
  - `MAJOR`: Remoção ou redefinição incompatível de princípios ou regras de governança.
  - `MINOR`: Adição de novos princípios ou expansão material das diretrizes existentes.
  - `PATCH`: Esclarecimentos de redação, correções tipográficas ou refinamentos não semânticos.
- **Revisão de Conformidade**: Qualquer violação aos princípios (como adição de pacote sem justificativa
  no plano ou acesso direto ao banco em endpoint) bloqueia a aprovação da implementação até ser corrigida.

**Version**: 1.0.0 | **Ratified**: 2026-11-07 | **Last Amended**: 2026-11-07
