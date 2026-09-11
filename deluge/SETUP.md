# SETUP — instalação das funções Deluge

Ordem obrigatória. Cada passo depende do anterior.

**Por que Deluge e não Zoho Flow:** a Etapa 1 cria sete pastas numa chamada só, com laço e tratamento de erro. No Flow isso vira sete ações encadeadas, sem laço e sem transação. A exceção é a Etapa 4, que precisa de um gatilho de chegada de e-mail — isso o Deluge do CRM não tem, então essa parte roda no Flow chamando a função.

---

## Passo 0 — o que eu preciso de você

Três coisas. Sem elas nada roda.

| # | O que | Onde pegar |
|---|---|---|
| 1 | **Conexão `workdrive_conn`** | Zoho CRM → Configuração → Desenvolvedor → Conexões → Criar. Serviço: Zoho WorkDrive (OAuth). Escopos: `WorkDrive.files.ALL` e `WorkDrive.team.READ` |
| 2 | **`WD_PASTA_PETRONECT`** — o `folder_id` da pasta `/NEGOCIOS/PETRONECT` no WorkDrive | Abra a pasta no WorkDrive; o id é o trecho depois de `/folder/` na URL |
| 3 | **`CNPJ_AUTIAM`** — só dígitos | É o que faz `Somos_Nos` funcionar na Etapa 3 |

Os itens 2 e 3 viram **variáveis de organização**: Configuração → Desenvolvedor → Variáveis → Nova, tipo Texto, nomes exatamente `WD_PASTA_PETRONECT` e `CNPJ_AUTIAM`.

> A conexão é OAuth e abre uma tela de consentimento em janela separada — eu não consigo completar essa tela por automação, e não digito credencial. Esse passo é seu.

---

## Passo 1 — campos e módulo no CRM

A API do Zoho **cria** campo mas não altera. Unicidade, validação e picklist são interface. Os passos abaixo que dizem "interface" não têm atalho.

### 1.1 — Valores novos na picklist `Tipo_Tarefa_Petronect` (módulo Tarefas)

Interface: Personalização → Módulos e Campos → Tarefas → `Tipo Tarefa Petronect` → adicionar:

- `Acao Petrobras`
- `Decisao de Recurso`

### 1.2 — Lookup em `Petronect_Logs`

Hoje o log não aponta para o negócio, então uma falha de coleta não aparece no registro. Criar:

| Campo | Tipo |
|---|---|
| `Negocio` | Lookup → Negócios |

### 1.3 — Módulo `Petronect_Propostas` (Etapa 3)

É o que falta para guardar preço **por proponente por item** — hoje `Petronect_Itens` só tem `Preco_Autiam`. Criar o módulo e os campos:

| Campo | Tipo | Obrigatório |
|---|---|---|
| `Name` | Texto | sim |
| `Chave_Proposta` | Texto, **único** | sim |
| `Negocio` | Lookup → Negócios | sim |
| `Numero_Certame` | Texto | sim |
| `Item_Number` | Inteiro | sim |
| `Proponente_Razao_Social` | Texto | sim |
| `Proponente_CNPJ` | Texto | não |
| `Preco_Unitario` | Moeda | sim |
| `Moeda` | Picklist: BRL · USD · EUR | sim |
| `Vencedor_do_Item` | Caixa de seleção | sim |
| `Somos_Nos` | Caixa de seleção | sim |
| `Fonte_Coleta` | Picklist: Mapa de Propostas · Ata · Relatório de Divulgação · Manual | sim |
| `Data_Coleta` | Data/hora | sim |
| `Conferido_Humano` | Caixa de seleção, padrão **desmarcado** | sim |

**A unicidade de `Chave_Proposta` não é opcional.** É ela que faz a recoleta atualizar em vez de duplicar — a mesma trava que faltou no incidente de 23/08.

### 1.4 — Campos de recurso em Negócios

| Campo | Tipo |
|---|---|
| `Prazo_Recurso_Ate` | Data/hora |
| `Decisao_Recurso` | Picklist: Não avaliado · Sem recurso · Recurso a interpor · Recurso interposto · Recurso julgado |
| `Justificativa_Recurso` | Área de texto |
| `Delta_Para_Vencedor_Pct` | Decimal |
| `Motivo_Perda` | Picklist: Preço · Técnico/Especificação · Habilitação/Documentação · Prazo de entrega · Desclassificado · Não identificado |

---

## Passo 2 — criar as funções

Configuração → Desenvolvedor → Funções → Nova Função, categoria **Automação**, para cada arquivo deste diretório. Cole o conteúdo inteiro; as funções auxiliares (`FN90` a `FN99`) vão junto no primeiro arquivo que as declara, então **crie nesta ordem**:

1. `FN01_estrutura_pastas.dg` — traz `FN96`, `FN97`, `FN98`, `FN99`
2. `FN02_arquivar_email.dg` — traz `FN92`, `FN93`, `FN94`, `FN95`
3. `FN03_resultado_licitacao.dg` — traz `FN90`, `FN91`

Se o editor reclamar de função duplicada, é porque uma auxiliar já existe — apague a cópia e mantenha a primeira.

---

## Passo 3 — regra de workflow (Etapa 1)

Automação → Regras de Fluxo de Trabalho → Nova.

| Parâmetro | Valor |
|---|---|
| Nome | `WF-20 Cotado abre estrutura de pastas` |
| Módulo | Negócios |
| Executar em | Criar ou editar |
| Repetição | Sempre que o negócio for editado |
| Condição | `Status Petronect` **É** `Cotado` |
| Ação | Função → `FN01_estrutura_pastas` |
| Argumento | `dealId` = `${Deals.Deal Id}` |

Não precisa condicionar a `Pasta_WorkDrive_Criada = falso`: a função é idempotente e reaproveita o que já existe.

---

## Passo 4 — função agendada (Etapa 3)

Configuração → Desenvolvedor → Funções → `FN03_resultado_licitacao` → Agendar.

| Parâmetro | Valor |
|---|---|
| Frequência | Diária |
| Horário | 07:00 |
| Argumentos | nenhum |

Ela mesma varre os cotados cuja `Data_Limite_Proposta` já passou e que ainda não têm `Ultima_Sync_Petronect`.

---

## Passo 5 — Zoho Flow (Etapa 4)

Um fluxo por caixa — `jbueno@` e `gmartinez@`.

| Etapa | Configuração |
|---|---|
| Gatilho | Zoho Mail → novo e-mail |
| Filtro | Remetente contém `petronect.com.br` |
| Ação | Custom Function → `FN02_arquivar_email` |

Argumentos a mapear:

```
assunto    = Subject
remetente  = From
corpo      = Content (texto puro)
anexos     = JSON [{"nome": <nome>, "url": <download url>}]
messageId  = MessageId
```

O `anexos` é o único que exige montagem: use a ação de listar anexos do Zoho Mail e componha o JSON antes de chamar a função.

---

## Passo 6 — teste, um certame só

Não solte em lote antes disso.

1. Pegue um negócio cotado **sem** pasta — hoje o `7004639291` está nessa condição (`Pasta_WorkDrive_Criada = falso`).
2. Reedite o Status para `Cotado` para disparar a WF-20.
3. Confira no WorkDrive: pasta com o número e as 6 subpastas.
4. Rode de novo. **Tem que continuar com 6 subpastas, não 12.** Se duplicar, a idempotência quebrou — pare e me chame.
5. Veja o registro em `Petronect_Logs` com `Funcao = FN01` e o `Negocio` preenchido.

---

## O que fica faltando depois disso

A **Etapa 2** e a parte de download da **Etapa 3** dependem da DP-01 — a Petronect não tem API confirmada para fornecedor. O ponto único de contato com o portal é a função `FN91_coletar_do_portal`, no fim do `FN03_resultado_licitacao.dg`, com o contrato de saída já definido. Enquanto ela devolver `ok=false`, a FN03 registra WARN e não grava nada errado.

Os três caminhos para resolver estão listados no cabeçalho da FN91. O mais barato é o segundo: abrir dois ou três e-mails `Relatório Divulgado ID <n>` já arquivados e ver se o mapa de propostas vem anexado. Se vier, a Etapa 3 inteira roda sem tocar no portal.
