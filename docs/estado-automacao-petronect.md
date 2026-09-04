# Estado da automação Petronect → Zoho CRM

Última atualização: **23/08/2026**

> **Leia primeiro:** duas decisões fecharam a arquitetura em 23/08/2026 e tornam obsoleto tudo o que este projeto dizia antes sobre "rotina agendada". Ver a seção *Decisões de arquitetura fechadas*.

## Decisões de arquitetura fechadas (23/08/2026)

| # | Decisão | Conteúdo |
|---|---|---|
| **DP-04** | Responsável técnico da automação | **Jorge Bueno — jbueno@autiam.com** |
| **DP-05** | Motor da automação | **Zoho Flow.** Toda automação roda dentro do Zoho. Rotinas agendadas externas **não escrevem no CRM**. O Claude entra apenas como serviço chamado de dentro do Zoho Flow, por requisição HTTP, nos passos que exigem leitura de linguagem natural (classificar e-mail, extrair prazo de edital). |

Regra prática derivada: se a decisão pode ser escrita como condição, é do Zoho (Blueprint, regra de workflow, validação, Deluge). Se depende de interpretar linguagem, o Zoho Flow chama o Claude e grava o resultado marcado como não conferido.

## Incidente de 23/08/2026 — rotina agendada descontrolada

A rotina agendada `trig_01UgCozpX8hBhGxLWRAnBTXr`, com verificação de duplicidade fraca, rodou e:

- recriou **34 oportunidades que já existiam** no CRM (base foi de 126 → 259 registros);
- criou um registro literalmente chamado **`None`**;
- reescreveu `Lead_Source` de volta para `Portal Eletrônico` em registros já corrigidos.

**Contenção e correção aplicadas no mesmo dia:**

1. Rotina **desativada** (`enabled = false`) e renomeada para `[DESATIVADA] Petronect — substituída pelo Zoho Flow`.
2. O prompt da rotina foi **substituído** por uma instrução de não fazer nada: se ela for disparada por engano, não cria, não atualiza e não exclui nenhum registro.
3. Os 34 duplicados + o registro `None` foram **excluídos** (35 exclusões, com aprovação do usuário).
4. `Lead_Source` regravado em **224 registros** para `PORTAIS ELETRONICOS`.

**Lição registrada:** o número Petronect pode estar embutido em qualquer posição do nome (`"7 X PSVs 7004613925"`). A verificação de duplicidade tem que **extrair os 10 dígitos de qualquer lugar do nome**, nunca comparar a string inteira. Segunda lição: nunca confiar em uma lista de IDs capturada no início de uma operação em lote — reconsultar por critério no final.

## Conexões ativas

- **Zoho Mail** via MCP (`ZOHO_MAIL_MCP`) — conta `jbueno@autiam.com`, accountId `3068981000000008002`, caixa de entrada folderId `3068981000000008014`.
- **Zoho CRM** via MCP — org `Autiam`, zgid `879084350`, Zoho One **Enterprise**.

## Módulo e conta

O módulo usado para oportunidades é o **Negócios (Deals)**. Não existe módulo customizado "Oportunidade". Todas as oportunidades Petronect ficam vinculadas à conta **PETRONECT** (id `6670502000002997001`).

## Estado da base em 23/08/2026 (verificado por COQL)

| Métrica | Valor |
|---|---|
| Total de registros no módulo Negócios | **245** |
| Nomes distintos | **245** (sem duplicatas — pronto para receber a restrição de unicidade) |
| `Lead_Source = PORTAIS ELETRONICOS` | **224** |
| `Lead_Source = PORTAIS ELETRONICO` ou `Portal Eletrônico` | **0** |
| `Status_Petronect = Declinado` | 160 — **todos** com `Stage = Cancelado` |
| `Status_Petronect = Cotado` | 28 — **todos** com `Data_Limite_Proposta` preenchida |
| `Status_Petronect = Pendente Análise` | 37 |

Colisão de nome resolvida para permitir a unicidade: dois negócios distintos chamados `RETROFIT VRU COOL SORPTION`, de contas diferentes, foram renomeados (não excluídos):

- `6670502000000729006` → `RETROFIT VRU COOL SORPTION - RUFF RIBEIRAO PRETO`
- `6670502000000730001` → `RETROFIT VRU COOL SORPTION - REDEPETRO`

## Campos no módulo Negócios

| Campo | API name | Tipo | Uso |
|---|---|---|---|
| Status Petronect | `Status_Petronect` | Picklist: Pendente Análise / Cotado / Declinado | Decisão do usuário sobre cotar ou declinar |
| Data Limite Proposta | `Data_Limite_Proposta` | Data | Prazo do edital para envio da proposta |
| Fornecedor Parceiro | `Fornecedor_Parceiro` | Lookup → Vendors | Parceiro da Autiam na oportunidade |
| Origem Automação | `Origem_Automa_o` | Picklist | **Campo oficial e único de procedência.** A v2 **NÃO** cria outro campo para isso — decisão do usuário em 23/08/2026. Backfill pendente nos registros antigos. |
| Tarefa Triagem Criada | `Tarefa_Triagem_Criada` | Checkbox | Trava de duplicidade da tarefa de triagem |

Campo obsoleto **Data Limite Petronect** (`Data_Limite_Petronect`), vazio, substituído por Data Limite Proposta — remover do layout pela interface.

**Limitação da API que rege todo o resto:** a API do Zoho CRM tem `createFields` (criar campo novo) mas **não tem `updateFields`**. Não é possível, por API: renomear opção de picklist existente, marcar campo como único, criar regra de validação, criar regra de workflow. Tudo isso é interface.

**Sobre o campo Stage/Estágio:** a API aceita apenas os rótulos **em português** do pipeline "Standard" do org — `getFields` retorna os nomes em inglês, mas gravar com eles falha com `MAPPING_MISMATCH`. Valores confirmados: `Prospeccao`, `Cancelado`, `Negociação/Revisão`, `Perda fechada para a concorrência`, `Perda fechada`, `Ganho fechado`, `Qualificação`.

## Campo no módulo Tasks

| Campo | API name | Tipo | Uso |
|---|---|---|---|
| Tipo Tarefa Petronect | `Tipo_Tarefa_Petronect` | Picklist: Triagem / Elaborar Proposta / Urgente - Sala | Classifica por que a tarefa foi criada automaticamente |

Campos padrão confirmados: `Subject`, `Due_Date`, `Who_Id`, `What_Id`, `Status` (criação = `Not Started`), `Priority` (`Normal`, `High`, `Highest`, `Low`, `Lowest`), `Description`, `Send_Notification_Email`, `Remind_At`, `Owner`.

## Passo 1 da implantação — 4 configurações de interface (documento entregue)

Documento operacional completo, com caminho de tela, parâmetros e checklist de conferência: **`Guia_Implantacao_Passo1_Zoho_CRM.docx`**, entregue ao usuário em 23/08/2026.

| # | Configuração | Onde | Estado |
|---|---|---|---|
| 1 | Renomear a opção `PORTAIS ELETRONICO` → **`PORTAIS ELETRONICOS`** na lista de Fonte de Cliente Potencial | Personalização → Campos | **Pendente — prioridade máxima.** Os 224 registros já têm o valor gravado, mas ele ainda não existe na lista; até renomear, filtros e relatórios não encontram os registros |
| 2 | Marcar **Nome do Negócio (`Deal_Name`) como único** | Personalização → Campos | Pendente (pré-condição já atendida: base sem duplicatas) |
| 3 | **Regra de validação**: exigir Data Limite Proposta quando `Status_Petronect = Cotado`, inclusive em chamadas de API | Personalização → Regras de Validação | Pendente (nenhum registro histórico viola a regra) |
| 4 | **Regra de workflow**: `Status_Petronect = Declinado` → `Stage = Cancelado` | Automação → Regras de Workflow | Pendente (os 160 declinados já foram ajustados por API) |

### Regra de nomenclatura que torna a unicidade eficaz

O nome do negócio de origem Petronect é **exclusivamente o número de 10 dígitos**. Nada de descrição, tag ou sufixo — a unicidade compara a string inteira, então `7004613925` e `7004613925 - REGULADOR` passariam os dois. Os registros que ainda carregam descrição no nome (ex. `7004643929 - REGULADOR DE PRESSÃO`) precisam ser renomeados para só o número antes de a automação entrar em produção.

## Outros pendentes de interface (menor prioridade)

1. Remover o campo Data Limite Petronect do layout.
2. Regra de layout: mostrar Data Limite Proposta apenas quando Status Petronect = Cotado.
3. Regra de workflow: `Status_Petronect = Cotado` → criar tarefa para Gabriel Martinez (`gmartinez@autiam.com`), assunto `Elaborar proposta - ${Deals.Nome Negócio}`, prazo = Data Limite Proposta, prioridade Alta.
4. Regra de workflow "rede de segurança": ao criar registro com Conta = PETRONECT e `Tarefa_Triagem_Criada` = falso, criar tarefa de revisão.
5. Esvaziar as 7 pastas vazias que ficaram no Lixo do Zoho Mail após a consolidação.

## Estrutura de pastas no Zoho Mail (consolidada em 17/08/2026)

Árvore única e definitiva: **`/Inbox/NEGOCIOS/PETRONECT`** (folderId `3068981000001429002`).

- **90 pastas de oportunidade** (nome = número Petronect, algumas com sufixo descritivo antigo, ex. `7004606740 - 152 PIT`) + pastas de status `PEDIDOS`, `PERDIDOS`, `POSTADOS` = 93 subpastas, **147 e-mails**.
- A árvore temporária `/Inbox/PETRONECT` foi consolidada aqui e removida.
- Convenção: **uma pasta por oportunidade, nomeada com o número**. Pastas antigas com sufixo descritivo e a classificação por status são informação do usuário e não devem ser renomeadas nem movidas.
- Observação de API: `deleteFolder` **não** exclui a pasta — move para `/Trash`.
- As oportunidades criadas a partir da planilha de triagem não têm e-mail correspondente, logo não têm pasta — não é necessário criar.

## Regras de classificação dos e-mails

- `Criação de Oportunidade ID <n>`, `Oportunidade Publicada`, `Oportunidade Prorrogada ID <n>` → nova oportunidade
- `SALA <n> ...` → mensagem de sala de colaboração, não gera oportunidade
- `Relatório Divulgado ID <n>` → relatório, não gera oportunidade
- Remetentes `cadastropetrobras@`, `notificacao@`, `manutencao@`, `vocesabia@` e pessoas físicas `@petronect.com.br` → administrativo, ignorar

## Fluxo de sala / relatório — desenhado, a implementar no Zoho Flow

- Sala de oportunidade **Cotada** → tarefa urgente (Priority Highest, `Tipo_Tarefa_Petronect = Urgente - Sala`), vencimento 1 dia útil; a "ciência" do usuário é concluir essa tarefa.
- Sala de oportunidade **Pendente Análise** → cria antecipadamente a tarefa de triagem (Priority High) e marca `Tarefa_Triagem_Criada = verdadeiro`.
- Sala de oportunidade **Declinada** → arquivar o e-mail na pasta da oportunidade, sem tarefa.
- Relatório de divulgação de oportunidade **Cotada** → resumir e notificar para leitura imediata.
- Relatório de divulgação de oportunidade **Declinada** → arquivar, sem tarefa.

Caso tratado manualmente em 17/08/2026: e-mail "SALA 7004641195", oportunidade estava Pendente Análise, usuário decidiu mudar para Cotado. Negócio `6670502000003273032` → Cotado; tarefa `6670502000003300001` criada, prioridade Highest, vencimento 18/08/2026.

## Triagem — planilha processada em 19/08/2026

`Triagem_Oportunidades_Petronect_3.xlsx`, 110 linhas com decisão preenchida.

- **24 negócios existentes atualizados** (6 Cotar + 18 Declinar): status, fornecedor, valor, data limite e observação.
- **35 oportunidades criadas** por decisão explícita do usuário, inclusive as 30 declinadas, "para termos o registro e reuniões com o parceiro".
- Em nenhuma delas foi criada tarefa "Elaborar proposta": os prazos do edital já haviam vencido (triagem retroativa), e criar tarefa com prazo vencido só polui a lista.
- **Restam 51 linhas sem decisão** — aguardando o usuário. Quando devolvida, repetir: checar existência no CRM → atualizar ou criar → nunca gerar tarefa com prazo retroativo vencido.

## Fornecedores / fabricantes parceiros

| Fornecedor | id | Cadastro |
|---|---|---|
| NIRMAL | `6670502000002314187` | completo |
| MICROSENSOR | `6670502000002314123` | completo |
| TRUEDYNE | `6670502000002314338` | completo |
| LIANGGU VALVE | `6670502000002314348` | completo |
| JIWEI | `6670502000003135009` | só nome |
| HENAN QUANSHUN | `6670502000003293002` | só nome |
| ZHENCHAO | `6670502000003293003` | só nome |
| LESHAN | `6670502000003293004` | só nome |

Os dados cadastrais dos quatro últimos serão preenchidos nos módulos **Contas** e **Contatos**, não em Fornecedores.

## Multi-caixa (gmartinez@autiam.com) e WorkDrive

- A Petronect também envia para `gmartinez@autiam.com`. A delegação de caixa configurada no admin do Zoho Mail **não aparece pela API/MCP** (testado com `getMailAccounts` e `getAllFolders` autenticados como jbueno). Este ambiente não consegue ler a caixa do Gabriel diretamente.
- Caminho viável: **Zoho Flow**, que autentica cada conta de e-mail separadamente — uma conexão por caixa.
- Não existe "Zoho Teams". O que o usuário quer é **Zoho WorkDrive** (pastas) + **Zoho TeamInbox** (colaboração), ambos sem conector MCP — vão via Zoho Flow.
- **Confirmado em 21/08/2026**: o Zoho Flow tem conector nativo para WorkDrive; a criação de pasta por oportunidade usa a ação nativa, sem HTTP Module manual.

## Próximos passos, em ordem

| Passo | O que é | Onde | Depende de |
|---|---|---|---|
| **1** | As 4 configurações de interface acima | Zoho CRM → Configurações | — |
| 2 | Blueprint do módulo Negócios | Zoho CRM → Blueprint | Passo 1 |
| 3 | Fluxo A — e-mail de nova oportunidade vira negócio | Zoho Flow | Passos 1 e 2 |
| 4 | Fluxo B — mensagem sobre oportunidade existente | Zoho Flow | Fluxo A |
| 5 | Fluxo C — relatório de divulgação | Zoho Flow | Fluxo A |
| 6 | Fluxo D — pastas no WorkDrive e arquivamento do e-mail de RFQ | Zoho Flow + WorkDrive | Fluxo A |
| 7 | Fluxo E — vigilância de prazos e alertas | Zoho Flow | Fluxos A–D |

## Fluxograma Zoho Vani

Espaço "Automação EMAIL_CRM - Fluxograma Petronect" (team AUTIAM, edition_id `909221261`, space_id `12454000000033002`), zona "Fluxo Petronect" (zone_id `0a3b39cc-45f6-4510-ba23-2ac3f357e05b`):
https://app.vanihq.com/edition/909221261/space/12454000000033002/zone/0a3b39cc-45f6-4510-ba23-2ac3f357e05b

24 nós / 28 conexões. **Atenção:** o fluxograma ainda mostra nós prefixados `[CLAUDE]` como se fossem motor. Após DP-05, esses nós representam chamadas HTTP feitas *de dentro* do Zoho Flow — o diagrama precisa ser reetiquetado.
