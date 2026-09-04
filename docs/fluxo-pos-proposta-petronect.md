# Fluxo pós-proposta Petronect — coleta do mapa de resultado

**Versão 1.0 — 03/09/2026.** Especificação do fluxo que vai ao portal da Petronect com o número de uma oportunidade cotada, colhe o mapa de propostas, arquiva os arquivos no WorkDrive, grava no CRM o preço de cada proponente por item e abre a tarefa de decisão de recurso.

Estado do CRM abaixo **conferido direto na API em 03/09/2026**, não copiado de documento anterior.

---

## 1. Estado verificado — o que já existe e o que está vazio

A estrutura de sincronização com o portal já foi criada (27/08/2026) e **nunca foi usada**. Isso muda o desenho: não é construir do zero, é preencher uma lacuna específica.

### Já existe em Negócios (Deals)

| Campo | Tipo | Serve para |
|---|---|---|
| `Numero_Petronect` | texto | Número do certame (10 dígitos). Igual a `Deal_Name` nos registros atuais |
| `Fase_Petronect` | picklist | `Publicada` · `Em Cotacao` · `Proposta Enviada` · **`Em Disputa`** · **`Em Analise`** · **`Homologada`** · `Cancelada` · **`Fracassada`** |
| `Vencedor_Final` | texto | Razão social do vencedor |
| `CNPJ_Vencedor` | texto | CNPJ do vencedor |
| `Valor_Homologado` | moeda | Valor homologado do certame |
| `Ultima_Sync_Petronect` | data/hora | Carimbo da última coleta bem-sucedida |
| `Sync_Erro` | texto | Última falha de coleta |
| `URL_Pasta_WorkDrive` | site | Pasta da oportunidade |
| `Pasta_WorkDrive_Criada` | booleano | — |
| `Anexos_Sincronizados` | data/hora | Carimbo da última coleta de anexos |
| `Qtd_Anexos` | inteiro | — |
| `Codigo_Oportunidade_Direta` | texto | Código alternativo de acesso |

### Já existem 3 módulos customizados

- **`Petronect_Itens`** — 1 registro por item do certame. Campos: `Chave_Item` (chave única), `Negocio` (lookup → Deals), `Numero_Certame`, `Item_Number`, `Codigo_Material`, `Descricao_Especificacao`, `Familia`, `Quantidade`, `Unidade`, **`Preco_Autiam`**, `Palavra_Chave_Match`, `Interesse_Autiam`. **0 registros hoje.**
- **`Petronect_Anexos`** — registro de arquivo. Campos: `Chave_Anexo` (única), `Negocio`, `Numero_Certame`, `Nome_Arquivo`, `URL_Anexo`, `Tipo_Anexo`, `Tamanho`, `Data_Coleta`.
- **`Petronect_Logs`** — `Funcao`, `Nivel` (INFO/WARN/ERROR), `Mensagem`, `Contexto`, `Data_Hora`. **Sem lookup para Negócio** — ver lacuna 3.

### O que está vazio hoje

Consulta aos 28 negócios `Status_Petronect = Cotado`: **nenhum** tem `Vencedor_Final`, `Valor_Homologado`, `Qtd_Anexos` ou `Anexos_Sincronizados` preenchidos. `Fase_Petronect` está nula em 27 dos 28 (só o 7004644291 tem `Em Cotacao`). `URL_Pasta_WorkDrive` existe em 4. `Petronect_Itens` está zerado.

Ou seja: **a coleta pós-proposta nunca rodou uma vez.**

---

## 2. Três lacunas que travam o fluxo

### Lacuna 1 — não há onde gravar o preço do concorrente (bloqueante)

`Petronect_Itens` tem `Preco_Autiam` e só. O pedido é o preço de **cada proponente** para **cada item** — isso é N × M, e não cabe num campo do item nem num campo do negócio.

**Solução: criar o módulo `Petronect_Propostas`.** Um registro = um proponente, um item, um preço.

| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| `Name` | texto | sim | `<certame>-<item>-<CNPJ>` |
| `Chave_Proposta` | texto **único** | sim | Mesma composição do Name. É a trava de idempotência |
| `Negocio` | lookup → Deals | sim | Related list "Propostas do Certame" |
| `Item` | lookup → Petronect_Itens | sim | |
| `Numero_Certame` | texto | sim | Redundante de propósito, para filtrar sem join |
| `Item_Number` | inteiro | sim | |
| `Proponente_Razao_Social` | texto | sim | Como aparece no portal, sem normalizar |
| `Proponente_CNPJ` | texto | não | Vazio se o portal não expuser |
| `Preco_Unitario` | moeda | sim | **> 0** (ver regra R2) |
| `Quantidade_Ofertada` | double | não | |
| `Preco_Total_Item` | moeda | não | |
| `Moeda` | picklist | sim | BRL · USD · EUR |
| `Classificacao_Item` | inteiro | não | Colocação no item, se o portal exibir |
| `Vencedor_do_Item` | booleano | sim | Default falso |
| `Somos_Nos` | booleano | sim | Verdadeiro quando o CNPJ é o da Autiam |
| `Fonte_Coleta` | picklist | sim | `Mapa de Propostas` · `Ata` · `Relatório de Divulgação` · `Manual` |
| `Data_Coleta` | data/hora | sim | |
| `Conferido_Humano` | booleano | sim | Default **falso** — nada vindo do portal entra como conferido |

### Lacuna 2 — não há campos de recurso

Adicionar em Negócios:

| Campo | Tipo | Observação |
|---|---|---|
| `Prazo_Recurso_Ate` | data/hora | Lido do edital/ata. **Não inventar** — se não achar, deixa vazio e sinaliza |
| `Decisao_Recurso` | picklist | `-None-` · `Não avaliado` · `Sem recurso` · `Recurso a interpor` · `Recurso interposto` · `Recurso julgado` |
| `Justificativa_Recurso` | textarea | Obrigatória quando `Decisao_Recurso` ≠ `Não avaliado` (regra de validação) |
| `Delta_Para_Vencedor_Pct` | double | (nosso preço ÷ preço vencedor − 1) × 100, calculado no fecho da coleta |
| `Motivo_Perda` | picklist | `Preço` · `Técnico/Especificação` · `Habilitação/Documentação` · `Prazo de entrega` · `Desclassificado` · `Não identificado` |

### Lacuna 3 — o log não aponta para o negócio

`Petronect_Logs` não tem lookup para Deals, então uma falha de coleta não aparece no registro. **Adicionar `Negocio` (lookup → Deals)** e `Numero_Certame` (texto). Sem isso, `Sync_Erro` no negócio é a única pista e ela é sobrescrita a cada tentativa.

---

## 3. O fluxo, etapa por etapa

Gatilho: usuário escolhe uma oportunidade e aciona (botão no CRM, ou lote das que estão em `Proposta Enviada` com data-limite vencida).

### E0 — Pré-condições (aborta se falhar)

- Negócio existe, `Status_Petronect = Cotado`, `Numero_Petronect` com 10 dígitos.
- `Data_Limite_Proposta` já passou. Coletar mapa antes da abertura não faz sentido e pode ler dado parcial.
- Já existe coleta? Se `Ultima_Sync_Petronect` < 24h e `Fase_Petronect = Homologada`, não repete — só reabre sob confirmação explícita.

### E1 — Acesso ao portal

O portal **não tem API confirmada** (DP-01 segue aberta), então o acesso é por navegador.

> **Regra dura: eu não digito credencial.** O login na Petronect é feito por você, no navegador, antes de me passar o comando. Eu opero a sessão já autenticada. Se a sessão cair no meio, eu paro, gravo WARN no log e devolvo a etapa — não tento reautenticar.

Se a Claude in Chrome estiver fora do ar (histórico de timeout em `list_connected_browsers`), a alternativa é você exportar o mapa do portal e me entregar o arquivo — o fluxo continua da E4 sem alteração.

### E2 — Localizar o certame

Buscar por `Numero_Petronect`. Se não achar, tentar `Codigo_Oportunidade_Direta`. Não encontrado nos dois: grava `Sync_Erro = "certame não localizado"`, log ERROR, encerra sem tocar em nada.

### E3 — Solicitar a pesquisa e capturar o resultado

Disparar a consulta do mapa de propostas / ata / relatório de divulgação. Capturar, na ordem:

1. **Cabeçalho** — situação do certame, unidade requisitante, data da abertura, vencedor, valor homologado.
2. **Itens** — número, código de material, descrição, quantidade, unidade.
3. **Mapa de propostas** — para cada item, a lista de proponentes com preço.
4. **Anexos** — ata, mapa, relatório, comunicados.

Tudo capturado vai para um pacote intermediário antes de escrever no CRM. **Nada é gravado parcialmente**: ou a etapa E6 escreve o conjunto, ou não escreve nada.

### E4 — Arquivar no WorkDrive

- Pasta da oportunidade: se `URL_Pasta_WorkDrive` estiver vazia, cria com o padrão do POP (`pop-documentacao-workdrive.md`); grava a URL e marca `Pasta_WorkDrive_Criada`.
- Subpasta `05_Resultado`.
- Sobe cada arquivo. Nome: `<certame>_<tipo>_<AAAA-MM-DD>.<ext>`.
- Para cada arquivo, um registro em `Petronect_Anexos` com `Chave_Anexo` = `<certame>|<tipo>|<hash do nome>`. Chave já existente → não duplica, atualiza `Data_Coleta`.
- Atualiza `Qtd_Anexos` e `Anexos_Sincronizados`.
- Aplica as tags da DP-08 na pasta (`PETRONECT`, `Portal`, `Oleo&Gas` + parceiro).

### E5 — Gravar os itens

Upsert em `Petronect_Itens` por `Chave_Item` = `<certame>|<item_number>`. Nunca duplica item.
`Preco_Autiam` **não é tocado por esta rotina** — é dado nosso, não do portal.

### E6 — Gravar as propostas dos concorrentes

Para cada par (item, proponente):

- **R1 — só grava se houver preço.** Registro com preço nulo/vazio é descartado.
- **R2 — só grava se `Preco_Unitario` > 0.** Zerado significa "não cotou este item" ou desclassificado; não é proposta.
- **R3 — upsert por `Chave_Proposta`.** Recoleta atualiza, não duplica. Esta é a trava que evita repetir o incidente de 23/08.
- **R4 — `Conferido_Humano = falso`** em tudo que veio do portal.
- **R5 — campo vazio em vez de campo errado.** Não deu para ler o CNPJ? Deixa vazio. Nunca inferir a partir da razão social.
- **R6 — `Somos_Nos`** marcado comparando o CNPJ com o da Autiam. Se o CNPJ não veio, comparar razão social e gravar WARN.

Fecho da etapa, no negócio:

- `Vencedor_Final`, `CNPJ_Vencedor`, `Valor_Homologado` a partir do cabeçalho.
- `Fase_Petronect` conforme a situação lida: `Em Analise` · `Homologada` · `Fracassada` · `Cancelada`.
- `Delta_Para_Vencedor_Pct` calculado a partir das propostas gravadas.
- `Ultima_Sync_Petronect` = agora; `Sync_Erro` limpo.
- `Stage` **não é alterado por esta rotina** — encerrar negócio é decisão humana, coerente com a rotina C da v2.

### E7 — Tarefa de decisão de recurso

Criada sempre que a coleta fecha com `Fase_Petronect` em `Em Analise`, `Homologada` ou `Fracassada` **e** `Somos_Nos = verdadeiro` em pelo menos uma proposta gravada.

| Campo | Valor |
|---|---|
| Assunto | `RECURSO? — <certame> — decidir até <prazo>` |
| Responsável | **a definir** (ver DP-10) |
| Vencimento | `Prazo_Recurso_Ate`. Se vazio: **hoje + 1 dia útil**, com o assunto prefixado `[PRAZO NÃO CONFIRMADO]` |
| Prioridade | Alta |
| `Tipo_Tarefa_Petronect` | `Decisão de Recurso` (valor novo na picklist) |
| Descrição | Vencedor, valor homologado, nosso preço, `Delta_Para_Vencedor_Pct`, link da pasta `05_Resultado`, link da lista de propostas |

Fechar a tarefa exige `Decisao_Recurso` e `Justificativa_Recurso` preenchidos — regra de validação, não convenção.

### E8 — Registro

Um registro em `Petronect_Logs` por execução: `Funcao = "coleta_pos_proposta"`, nível, contagens (itens lidos, propostas gravadas, descartadas por R1/R2, anexos), duração, e o `Negocio` no novo lookup.

---

## 4. Onde cada peça roda

Coerente com a **DP-05** (motor = Zoho Flow; Claude é serviço HTTP chamado de dentro do Flow, só para o interpretativo):

| Etapa | Executor | Por quê |
|---|---|---|
| E0 pré-condições | Zoho Flow | É condição, logo é do Zoho |
| E1–E3 portal | Claude (navegador) | Portal sem API, leitura de tela e de PDF |
| E4 WorkDrive | Zoho Flow | Determinístico |
| E5–E6 escrita no CRM | Zoho Flow | Escrita transacional com upsert por chave |
| E7 tarefa | Regra de workflow nativa | Determinístico |
| E8 log | Função Deluge | — |

O ponto de contato é um payload JSON que o Claude devolve ao Flow. O Flow valida antes de escrever: se faltar campo obrigatório, ele **rejeita o lote inteiro** e grava ERROR — não escreve pela metade.

---

## 5. O que precisa ser decidido antes de construir

| # | Decisão | Por que trava |
|---|---|---|
| **DP-01** | A Petronect tem API para fornecedor? | Já estava aberta. Se tiver, E1–E3 mudam de natureza e ficam muito mais confiáveis |
| **DP-09** | Prazo padrão de recurso | O prazo real vem do edital de cada certame. Precisa de um default para quando a leitura falhar — e de quem confirma que ele está certo. Não vou fixar prazo legal por conta própria |
| **DP-10** | Quem decide o recurso | O fluxo atual manda proposta para `gmartinez@`. Recurso é decisão de outra natureza — pode ser você |
| **DP-11** | Guardar proposta de concorrente é aceitável? | Dado é público na ata, mas vale a confirmação de que fica no CRM e quem enxerga |
| **DP-12** | Escopo da coleta | Só o item que cotamos, ou todos os itens do certame? Todos dá inteligência de mercado e custa mais coleta |

---

## 6. Ordem de construção

1. Criar `Petronect_Propostas` (módulo + 17 campos + unicidade em `Chave_Proposta` + related list em Negócios).
2. Criar os 5 campos de recurso em Negócios e o valor `Decisão de Recurso` em `Tipo_Tarefa_Petronect`.
3. Adicionar `Negocio` e `Numero_Certame` em `Petronect_Logs`.
4. Regra de validação: fechar tarefa de recurso exige decisão + justificativa.
5. Montar o Flow com um certame só, escolhido a dedo, e conferir registro a registro antes de soltar em lote.
6. Só depois: botão no CRM e execução em lote.

> Lembrete da lição 4 do projeto: a API do Zoho **cria** campo, não altera. Unicidade, validação e picklist são interface. Os passos 1–4 são trabalho manual de configuração, não chamada de API.

---

## 7. Teste de aceite

Pegar um certame encerrado e reconstruir, sem abrir planilha nem perguntar a ninguém: quantos proponentes cotaram cada item, com que preço, quem venceu, por quanto perdemos, quais arquivos sustentam isso, quem decidiu sobre recurso e com que justificativa — e rodar a coleta duas vezes seguidas sem gerar um único registro duplicado.
