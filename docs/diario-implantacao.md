# Diário de implantação — automação Petronect

Registro cronológico do que foi efetivamente executado. Complementa `indice-e-estado-verificado.md` (estado) e `workflow-petronect-v2.md` (desenho).

Responsável: Jorge Bueno — jbueno@autiam.com (DP-04).

---

## 23/08/2026

| Ação | Escopo | Resultado |
|---|---|---|
| Incidente: rotina agendada recriou registros | 126 → 259 registros | 34 duplicatas + 1 registro `None` |
| Rotina desativada e neutralizada | `trig_01UgCozpX8hBhGxLWRAnBTXr` | `enabled = false`, prompt trocado por instrução de não fazer nada |
| Exclusão dos registros indevidos | 35 exclusões | Base volta ao normal |
| `Lead_Source` → `PORTAIS ELETRONICOS` | 224 registros | Zero com grafia antiga |
| `Status_Petronect = Declinado` → `Stage = Cancelado` | 160 registros | 160 de 160 |
| Colisão de nome resolvida | 2 registros | `RETROFIT VRU COOL SORPTION` desambiguado pelo nome da conta |
| V2 corrigida | Documento | Zoho Flow como motor; `Origem_Registro` removido; DP-04 e DP-05 decididas |
| Guia de Implantação Passo 1 | Documento | 4 configurações de interface, com checklist |

## 25/08/2026 — manhã (API)

| Ação | Escopo | Resultado |
|---|---|---|
| Padronização do nome do negócio | **15 registros** | `Deal_Name` passou a ser **apenas o número Petronect de 10 dígitos**; o objeto foi movido para o campo `Description`, com nota de rastreabilidade |
| Conferência | COQL | **0 registros** de origem Petronect com descrição no nome |

IDs renomeados: `6670502000003341014`, `6670502000003342001` a `6670502000003342013`, `6670502000003365001`.

## 25/08/2026 — PASSO 1 CONCLUÍDO (interface + API)

Executado por navegador (extensão Claude for Chrome) e por API. **As quatro configurações estão ativas e testadas.**

### Configuração 1 — opção da lista renomeada ✅

Personalização → Módulos e Campos → Negócios → Campos → Fonte de Cliente potencial → Editar propriedades.
A opção `PORTAIS ELETRONICO` foi **renomeada** para `PORTAIS ELETRONICOS` (o Zoho avisou "a alteração será atualizada em todos os locais utilizados"). Conferido depois: **224 registros** continuam com o valor, agora dentro da lista. Filtros e relatórios voltam a funcionar.

### Configuração 2 — unicidade ✅ (por caminho alternativo)

**Descoberta que muda o desenho:** `Deal_Name` é **campo definido pelo sistema**. O Zoho recusa alterar suas propriedades — a tela mostra *"As propriedades dos campos definidos pelo sistema não podem ser modificadas"*. Marcar o Nome do Negócio como único **é impossível**, por interface e por API.

Solução aplicada, que entrega o mesmo efeito:

| Item | Valor |
|---|---|
| Campo criado | **Número Petronect** — `Numero_Petronect` |
| Tipo | Texto, 20 caracteres |
| Restrição | **Único, sem diferenciar maiúsculas** |
| id do campo | `6670502000003393001` |
| Backfill | **224 registros** preenchidos com o número de 10 dígitos |

**Teste real executado:** tentativa de criar um negócio com `Numero_Petronect = 7004613925` (já existente) foi **recusada** com `DUPLICATE_DATA`, e a resposta ainda devolve o id do registro que já tem o número (`6670502000002997004`) — o Zoho Flow pode usar isso para decidir entre criar e atualizar, sem consulta extra.

**Consequência para o desenho:** a chave de idempotência do Fluxo A passa a ser `Numero_Petronect`, não `Deal_Name`. O nome do negócio continua sendo o número por convenção, mas quem protege é o campo novo.

### Configuração 3 — regra de validação ✅

Negócios → Regras de validação → campo **Data Limite Proposta**.

| Parâmetro | Valor |
|---|---|
| Executar regra | Quando os critérios forem atendidos |
| Validar em | Salvar apenas |
| Critério do campo | Data Limite Proposta **está vazio** |
| Aplicada a | Negócios com **Status Petronect É Cotado** |
| Preferência | Interromper com erro |
| Mensagem | "Informe a Data Limite da Proposta. Toda oportunidade Cotada precisa de prazo." |

**Teste real executado:** criação por API de um negócio com `Status_Petronect = Cotado` e sem data foi **recusada** com `VALIDATION_RULE_FAILED` e a mensagem acima.

> **Detalhe crítico para o Zoho Flow:** a regra só é aplicada em chamadas de API quando a requisição pede explicitamente `apply_feature_execution: [{"name": "criteria_validation_rule"}]`. Sem esse parâmetro, a validação é ignorada. **A ação de CRM dentro do Fluxo A precisa estar configurada para executar as regras de validação**, senão o Flow grava Cotado sem prazo. A unicidade do `Numero_Petronect`, ao contrário, vale sempre.

### Configuração 4 — regra de workflow ✅

Automação → Regras de Fluxo de Trabalho → **"Declinado encerra a oportunidade"**.

| Parâmetro | Valor |
|---|---|
| Módulo | Negócios |
| Executar em | Criar ou editar |
| Repetição | Repetir sempre que o negócio for editado |
| Condição | Status Petronect **É** Declinado |
| Ação imediata | Atualização de campo **"Fase para Cancelado"** → Estágio = Cancelado |
| Status | Ativa |

## 25/08/2026 — Zoho Flow: iniciado e bloqueado

- Zoho Flow acessível na org Autiam, workspace vazio até hoje.
- Criado o fluxo **"Fluxo A - Petronect nova oportunidade para o CRM"** (pasta My Flows), com descrição registrando DP-05 e o responsável. **O fluxo está DESLIGADO e sem gatilho** — é um esqueleto, não roda nada.
- Gatilho escolhido: Zoho Mail → **"Email matching search received"**.
- **Bloqueio:** o gatilho exige uma *conexão* Zoho Flow ↔ Zoho Mail. O botão "Conectar" abre a tela de consentimento OAuth em **uma janela separada do navegador**, fora do grupo de abas que a automação controla. Não é possível ver nem completar essa tela por automação.

**Como desbloquear (feito uma vez, por pessoa):** criar as conexões manualmente em Zoho Flow → Configurações → Conexões:

1. **Zoho Mail** — caixa `jbueno@autiam.com`
2. **Zoho Mail** — caixa `gmartinez@autiam.com` (segunda conexão; é assim que se contorna a limitação da delegação, que não aparece por API)
3. **Zoho CRM** — org Autiam
4. **Zoho WorkDrive** — para o Fluxo D

Com as conexões criadas, o resto do Fluxo A pode ser montado por automação.

## Estado do CRM após o Passo 1

| Métrica | Valor |
|---|---|
| Registros no módulo Negócios | 245 |
| `Lead_Source = PORTAIS ELETRONICOS` | 224 — agora dentro da lista de opções |
| `Numero_Petronect` preenchido | 224, todos únicos |
| Nomes de negócio Petronect com descrição | 0 |
| Declinados com `Stage = Cancelado` | 160 de 160 |
| Cotados com `Data_Limite_Proposta` | 28 de 28 |
| Regras de validação ativas | 1 |
| Regras de workflow ativas | 2 (a nova + a "Big Deal Rule" pré-existente) |

## Pendências que continuam abertas

- **Conexões do Zoho Flow** — usuário reportou em 25/08 (via chat) que as conexões OneAuth foram criadas. **Não verificado neste ambiente** — falta confirmar na interface do Zoho Flow se o gatilho do Fluxo A passou a funcionar e testar com um e-mail real antes de considerar o bloqueio resolvido.
- **Ação de CRM do Fluxo A** precisa habilitar a execução das regras de validação (ver o detalhe crítico na Configuração 3).
- `Origem_Automa_o` vazia na maior parte dos registros — backfill não feito por falta de critério confiável para separar, retroativamente, o que veio de automação, o que veio da planilha e o que foi manual. Definir a regra antes de preencher.
- Campo obsoleto `Data_Limite_Petronect` ainda no layout.
- 51 linhas da planilha de triagem sem decisão do usuário.
- DP-01 (API da Petronect), DP-02 (alçadas), DP-03 (prazo do parceiro), DP-06 (homem-hora).
- Blueprint do módulo Negócios (Passo 2) não iniciado.

## 25/08/2026 — Novo pedido do usuário: ampliar escopo (pendente decisão)

Pedido recebido nesta conversa, com pontos que precisam de decisão do usuário antes de qualquer construção:

1. **Ampliar a criação de oportunidades para todas as caixas ativas do Zoho Mail**, incluindo `vendas@autiam.com` e `contato@autiam.com`, não só a Petronect. Isso alinha com a descrição original do projeto ("qualquer email de cliente parceiro e leads"), mas amplia bastante o escopo hoje desenhado (só Petronect). Falta definir: qual módulo recebe esses registros (Leads vs. Negócios), critério de classificação por caixa, e sobretudo —
2. **Chave de deduplicação para as caixas novas.** Fora da Petronect não existe um "número da oportunidade" para usar como chave única (a chave atual, `Numero_Petronect`, é específica do portal). Especificação em `criterios-deduplicacao-multi-canal.md` (DP-07) — 3 camadas (remetente+negócio aberto, mesma empresa com sinalização para revisão humana, número citado em texto livre). **Ainda não construído.**
3. **Armazenamento único dos e-mails (Zoho Connect ou Streams), organizável por todos.** A arquitetura fechada em 21/08/2026 já usa **Zoho WorkDrive** (pasta por oportunidade) + **Zoho TeamInbox** para esse mesmo objetivo — nenhum dos dois tem conector MCP neste ambiente, ambos passam pelo Zoho Flow. Zoho Connect/Streams é um produto diferente (intranet/colaboração), sem conector disponível aqui e fora do desenho aprovado. **Ainda sem decisão do usuário** entre substituir WorkDrive+TeamInbox por Connect, ou manter o que já foi decidido.
4. **Botões no Zoho Mail e no Zoho CRM, para todos os usuários.** Mesma limitação já documentada nas 4 configurações do Passo 1: customização de botão/layout é **interface, não API**. Este ambiente não pode criar o botão diretamente — precisa de um guia passo a passo (como o `Guia_Implantacao_Passo1_Zoho_CRM.docx`) para execução manual. **Ainda não construído.**
5. **Atualizar toda a documentação e o fluxograma no Zoho Vani.** Documentos texto (`indice-e-estado-verificado.md`, `criterios-deduplicacao-multi-canal.md`, `padrao-tags.md`) e o fluxograma local (`fluxo-petronect-crm.mermaid`) já atualizados em 25/08. **O fluxograma no Zoho Vani (ferramenta externa) segue pendente** — melhor consolidar numa única atualização quando os pontos 1–4 acima estiverem decididos.

**Conflito de política já sinalizado:** DP-05 (23/08/2026) determina que toda automação de escrita no CRM roda dentro do Zoho Flow, nunca por rotina agendada externa. Os itens 1 e 2 acima — a lógica de criação de oportunidade e a checagem de dedup rodando continuamente sobre e-mail de produção — continuam reservados ao Zoho Flow; não foram e não serão executados a partir desta sessão.

## 25/08/2026 — DP-08 (padrão de tags): configuração executada no CRM e no Mail

Pedido do usuário nesta conversa: "implementar estas configurações nos aplicativos CRM, Zoho Mail e Zoho Drive e informar quando finalizar" — referente ao padrão de tags/rótulos (DP-08, ver `padrao-tags.md`).

**Distinção importante em relação ao item anterior:** isto é configuração pontual de taxonomia (criar tags/labels e corrigir o backfill de registros já existentes), no mesmo padrão do Passo 1 (23–25/08) — não é ligar uma automação contínua de leitura de e-mail e escrita de negócio. Por isso foi executado diretamente nesta sessão; a criação/dedup de oportunidades (item anterior) continua de fora, reservada ao Zoho Flow por causa da DP-05.

### O que foi feito

**1. Descoberta: existe um 9º parceiro não documentado.** `getRecords` no módulo Vendors retornou **9 fornecedores**, não 8: além de NIRMAL, MICROSENSOR, TRUEDYNE, LIANGGU VALVE, JIWEI, HENAN QUANSHUN, ZHENXUAN, LESHAN — existe também **XHVAL** (id `6670502000003325048`), já em uso no negócio "OBRA VRU ALE GRU" (id `6670502000003347002`). Incorporado em tudo abaixo.

**2. Tags criadas no CRM (módulo Deals)** — as que faltavam, das 9: `TRUEDYNE`, `LIANGGU VALVE`, `JIWEI`, `HENAN QUANSHUN`, `ZHENXUAN`, `LESHAN`, `XHVAL`. (`NIRMAL` e `MICROSENSOR` já existiam.)

**3. Labels criados no Zoho Mail** (conta `jbueno@autiam.com`) — os mesmos 7 que faltavam. (`NIRMAL`, `MICROSENSOR`, `PETRONECT`, `Portal`, `Oleo&Gas` já existiam como labels — o Mail já estava alinhado nesses cinco antes deste pedido.)

**4. Backfill de tags de parceiro em Negócios já existentes** — consulta COQL encontrou **172 negócios** com `Fornecedor_Parceiro` preenchido. Tag aplicada por parceiro, em modo aditivo (`over_write: false`, preservando as tags já existentes em cada registro):

| Parceiro | Negócios tageados |
|---|---|
| NIRMAL | 101 |
| MICROSENSOR | 68 |
| TRUEDYNE | 1 |
| ZHENXUAN | 1 |
| XHVAL | 1 |
| **Total** | **172** |

**Conferido depois** via `getTags`: `associated_record_count` bate com o esperado em cada tag (NIRMAL, MICROSENSOR, TRUEDYNE, ZHENXUAN, XHVAL todos com contagem consistente; LIANGGU VALVE, JIWEI, HENAN QUANSHUN, LESHAN criadas com 0 registros — nenhum negócio ativo usa esses 4 parceiros hoje, tag fica pronta para quando algum usar). Nenhum negócio foi criado, atualizado em outros campos, ou excluído — só o campo Tags recebeu adição.

### O que **não** foi feito (e por quê)

- **Aplicar os labels às mensagens de e-mail já arquivadas por oportunidade.** Os labels foram criados (taxonomia pronta), mas rotular as ~147 mensagens já arquivadas em `/Inbox/NEGOCIOS/PETRONECT/<número>` por oportunidade/parceiro é um backfill à parte, maior, ainda não executado. Fica para quando o usuário confirmar que quer esse backfill retroativo (o Zoho Flow, quando estiver ligado, passa a aplicar o label em todo e-mail novo).
- **Zoho WorkDrive (rótulos).** Este ambiente **não tem conector/ferramenta para o Zoho WorkDrive** — só Zoho Mail e Zoho CRM estão disponíveis aqui via MCP. Aplicar rótulos nas pastas do WorkDrive precisa ser feito pela interface (manual) ou de dentro do Zoho Flow, quando montado. Nada foi alterado no WorkDrive.
- **Sincronização automática entre as três plataformas.** O que existe hoje é o estado atual sincronizado manualmente, uma vez. A regra "toda mudança futura se propaga" (ver `padrao-tags.md`) ainda depende do Zoho Flow.
