# Feature Specification: Encurtador de Links (URL Shortener)

**Feature Branch**: `001-url-shortener`

**Created**: 2026-11-07

**Status**: Draft

**Input**: User description: "Um encurtador de links. O criador informa uma URL longa e recebe um link curto para compartilhar. Quem clica no link curto é redirecionado ao destino, e cada acesso é contado. O criador consegue consultar quantos acessos o link dele recebeu. Comportamentos esperados: o código do link é gerado automaticamente com 7 caracteres alfanuméricos; o código é único em todo o sistema; a URL de destino precisa ser http ou https e ter no máximo 2048 caracteres; o contador nunca diminui; os links continuam válidos depois que a aplicação reinicia. Fora de escopo: cadastro e login de usuários, domínio personalizado, relatório geográfico, edição do destino de um link já criado."

## Clarifications

### Session 2026-11-09

- Q: O código curto de 7 caracteres alfanuméricos deve diferenciar letras maiúsculas de minúsculas (case-sensitive) ou ignorar a caixa (case-insensitive)? (FR-004) → A: Case-insensitive (`Abc1234` e `abc1234` representam o mesmo link; qualquer variação de caixa redireciona/consulta o mesmo destino).
- Q: Quando uma mesma URL de destino for enviada para encurtamento mais de uma vez, o sistema deve gerar um novo código curto independente ou reutilizar o código curto já existente? (FR-001) → A: Gerar novo código (cada solicitação cria um novo registro com código de 7 caracteres próprio e contador zerado).
- Q: O redirecionamento do link curto deve ser temporário (não armazenado em cache pelo navegador) para garantir que cliques repetidos de um mesmo visitante sejam sempre contabilizados no servidor? (FR-005) → A: Redirecionamento temporário (sem cache no cliente), garantindo que todo clique passe pelo servidor e seja contabilizado.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Criar um Link Curto a partir de uma URL Longa (Priority: P1)

O criador informa uma URL de destino longa e válida e recebe um link curto contendo um código único gerado automaticamente com 7 caracteres alfanuméricos, pronto para ser compartilhado.

**Why this priority**: É a funcionalidade central que origina os links curtos no sistema; sem a criação de links, o redirecionamento e a contagem de acessos não podem existir.

**Independent Test**: Pode ser testado de forma independente enviando uma URL válida (`http` ou `https` com até 2048 caracteres) e verificando que o sistema retorna um link curto com um código alfanumérico de 7 caracteres e rejeita URLs inválidas.

**Acceptance Scenarios**:

1. **Given** que o criador possui uma URL válida iniciada por `http://` ou `https://` com até 2048 caracteres, **When** ele solicita o encurtamento da URL, **Then** o sistema gera um código único de exatamente 7 caracteres alfanuméricos sem caracteres ambíguos (`abcdefghjkmnpqrstuvwxyz23456789`, excluindo `0`, `o`, `1`, `l`, `i`), persiste o vínculo com contador inicial zero e hash SHA-256 da chave de gerenciamento, e retorna os dados do link curto criado.
2. **Given** que o criador informa uma URL com esquema diferente de `http` ou `https` (como `ftp://`, `javascript:` ou texto sem esquema), **When** ele solicita o encurtamento, **Then** o sistema rejeita a solicitação informando erro de validação na URL de destino.
3. **Given** que o criador informa uma URL com mais de 2048 caracteres ou vazia, **When** ele solicita o encurtamento, **Then** o sistema rejeita a solicitação informando erro de validação quanto ao tamanho ou obrigatoriedade da URL.

---

### User Story 2 - Redirecionar Visitante e Contabilizar Acesso (Priority: P1)

Quando qualquer pessoa acessa o link curto utilizando seu código de 7 caracteres, o sistema redireciona imediatamente o visitante para a URL de destino original e registra +1 no contador de acessos daquele link.

**Why this priority**: Entrega o valor principal para quem recebe o link compartilhado (chegar ao destino original) e alimenta a métrica de acessos de forma contínua.

**Independent Test**: Pode ser testado acessando um código previamente cadastrado e verificando que o visitante é encaminhado para a URL original, que acessos sucessivos incrementam a contagem e que códigos inexistentes retornam aviso de recurso não encontrado.

**Acceptance Scenarios**:

1. **Given** um link curto previamente criado e ativo no sistema, **When** um visitante acessa esse link pelo seu código de 7 caracteres, **Then** o visitante é redirecionado para a URL de destino correspondente e o contador de acessos do link aumenta em 1 unidade.
2. **Given** um link curto que já recebeu acessos e foi persistido, **When** a aplicação é reiniciada e um visitante acessa novamente o link curto, **Then** o redirecionamento continua funcionando normalmente e a contagem soma-se ao valor anterior sem perda de dados.
3. **Given** um código de link curto que não existe no sistema, **When** um visitante tenta acessá-lo, **Then** o sistema informa que o link solicitado não foi encontrado.

---

### User Story 3 - Consultar Quantidade de Acessos de um Link (Priority: P2)

O criador consulta um link curto pelo seu código de 7 caracteres para visualizar quantos acessos aquele link recebeu até o momento, sem que a própria consulta altere o contador de cliques.

**Why this priority**: Permite ao criador acompanhar o engajamento e o alcance do link compartilhado, completando o ciclo de valor do produto.

**Independent Test**: Pode ser testado consultando as informações de um código existente antes e depois de acessos de redirecionamento e confirmando que o número exibido reflete exatamente os acessos realizados.

**Acceptance Scenarios**:

1. **Given** um link curto recém-criado que ainda não foi utilizado para redirecionamento, **When** o criador consulta as estatísticas desse código, **Then** o sistema exibe a URL de destino, o código curto e o total de acessos igual a `0`.
2. **Given** um link curto que recebeu `N` acessos de redirecionamento, **When** o criador consulta as estatísticas desse código repetidas vezes, **Then** o sistema informa o total exato de `N` acessos sem incrementar nem reduzir o contador durante a consulta.
3. **Given** um código que não corresponde a nenhum link cadastrado, **When** o criador consulta as estatísticas desse código, **Then** o sistema informa que o link não foi encontrado.

---

### Edge Cases

- **URL no limite exato de tamanho**: O que acontece quando a URL de destino possui exatamente 2048 caracteres? O sistema deve aceitá-la normalmente; com 2049 caracteres, deve rejeitá-la com erro de validação.
- **Colisão na geração de código**: Como o sistema lida caso o gerador automático produza um código de 7 caracteres que já exista na base? O sistema deve detectar a colisão de forma transparente e gerar um novo código até garantir unicidade antes de concluir o cadastro.
- **Concorrência de acessos e invariância do contador**: Quando múltiplos visitantes clicam no mesmo link simultaneamente, todos os cliques devem ser somados e o contador nunca deve diminuir sob nenhuma circunstância.
- **Diferenciação de código e caracteres inválidos**: O código curto é insensível a maiúsculas e minúsculas (case-insensitive), portanto variações como `Abc1234` e `abc1234` referenciam o mesmo link. Caso alguém consulte ou tente redirecionar um código inexistente ou fora do padrão, o sistema não deve falhar inesperadamente, respondendo de forma clara que o link não foi encontrado ou é inválido.
- **Reinicialização do sistema**: Links criados e seus respectivos totais de acessos devem permanecer armazenados de forma durável após reinicializações do serviço.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema DEVE permitir a criação de um link curto a partir de uma URL de destino informada pelo usuário, sem exigir autenticação ou cadastro prévio, gerando sempre um novo registro com código curto independente e contador zerado mesmo quando a mesma URL de destino já tiver sido encurtada anteriormente.
- **FR-002**: O sistema DEVE validar que a URL de destino informada não seja vazia, possua formato de URI absoluta válida, utilize exclusivamente o esquema `http` ou `https` e tenha comprimento máximo de 2048 caracteres.
- **FR-003**: O sistema DEVE rejeitar solicitações de criação com URL inválida e retornar mensagens de erro de validação claras em inglês.
- **FR-004**: O sistema DEVE gerar automaticamente, para cada link criado, um código identificador único em todo o sistema composto exatamente por 7 caracteres alfanuméricos sem caracteres ambíguos (`abcdefghjkmnpqrstuvwxyz23456789`, excluindo `0`, `o`, `1`, `l`, `i`), tratado de forma insensível a maiúsculas e minúsculas (case-insensitive) tanto na garantia de unicidade quanto nas buscas de redirecionamento e consulta.
- **FR-005**: O sistema DEVE redirecionar qualquer solicitação de acesso a um código existente para a sua respectiva URL de destino original utilizando redirecionamento temporário (não armazenado em cache pelo cliente), garantindo que acessos repetidos de um mesmo visitante passem sempre pelo servidor para contabilização.
- **FR-006**: O sistema DEVE incrementar em `1` o contador de acessos do link a cada redirecionamento realizado com sucesso, garantindo que o contador inicie em `0` na criação e nunca diminua (monotônico crescente).
- **FR-007**: O sistema DEVE permitir a consulta das informações de um link curto a partir de seu código (incluindo código, URL de destino, data de criação e total de acessos acumulado), sem que essa consulta incremente o contador de acessos.
- **FR-008**: O sistema DEVE informar claramente que o recurso não foi encontrado quando uma tentativa de redirecionamento ou de consulta de estatísticas referenciar um código inexistente.
- **FR-009**: O sistema DEVE armazenar os links criados e seus respectivos contadores de acesso de maneira durável, mantendo todos os links e contagens íntegros após reinicializações da aplicação.

### Key Entities *(include if feature involves data)*

- **ShortenedLink (Link Encurtado)**: Representa o vínculo permanente entre um código curto e uma URL longa de destino.
  - **Code (Código)**: Identificador único de 7 caracteres alfanuméricos sem caracteres ambíguos (`abcdefghjkmnpqrstuvwxyz23456789`), gerado automaticamente e tratado de forma *case-insensitive*.
  - **OriginalUrl (URL de Destino)**: Endereço completo (`http` ou `https`, até 2048 caracteres) para o qual o visitante será redirecionado.
  - **ClickCount (Total de Acessos)**: Número inteiro não negativo (iniciado em `0`, apenas crescente) que contabiliza quantas vezes o link curto foi acionado para redirecionamento.
  - **ManagementKeyHash (Hash da Chave de Gerenciamento)**: Hash criptográfico SHA-256 da chave de gerenciamento emitida na criação do link (nunca armazenada em texto puro).
  - **CreatedAtUtc (Data de Criação UTC)**: Momento exato em UTC em que o link curto foi gerado.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% das URLs válidas (`http`/`https` com até 2048 caracteres) submetidas resultam na geração imediata de um código alfanumérico único de 7 caracteres.
- **SC-002**: 100% das tentativas de criação com URLs fora do padrão (sem esquema `http`/`https`, malformadas ou com mais de 2048 caracteres) são bloqueadas antes da persistência com retorno explícito dos motivos de invalidação.
- **SC-003**: 100% dos cliques válidos em links curtos redirecionam o visitante para a URL original exata e refletem o incremento correspondente de `+1` na consulta de acessos subsequente.
- **SC-004**: 100% dos links criados e seus contadores de acesso permanecem disponíveis e inalterados após a parada e reinicialização completa da aplicação.

## Assumptions

- Cada solicitação de encurtamento gera um novo código curto independente de 7 caracteres alfanuméricos, mesmo que a mesma URL de destino já tenha sido encurtada anteriormente por outro criador, permitindo que cada pessoa acompanhe a contagem exclusiva do link que compartilhou.
- Como não há cadastro nem login de usuários, qualquer pessoa que possua o código de 7 caracteres pode tanto utilizá-lo para redirecionamento quanto consultar a contagem de acessos daquele código.
- Links criados não possuem data de expiração automática e não podem ter sua URL de destino editada ou seu contador zerado após a criação.

## Out of Scope

- Cadastro, autenticação e login de usuários.
- Configuração de domínio personalizado ou escolha manual de apelido (slug customizado) para o link curto.
- Relatórios geográficos, identificação de dispositivo/navegador ou histórico detalhado por clique (apenas o total acumulado é mantido).
- Edição da URL de destino de um link já criado ou redução/reinicialização manual do contador de acessos.

