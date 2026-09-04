# Critérios de deduplicação multi-canal — todas as caixas, todos os portais

**Decisão DP-07 (25/08/2026), confirmada pelo usuário nesta conversa:** toda automação de criação de oportunidade, em **qualquer** caixa de e-mail ativa da Autiam — hoje `jbueno@autiam.com`, `gmartinez@autiam.com`, `vendas@autiam.com`, `contato@autiam.com`, e qualquer caixa que passe a existir no futuro — tem que passar por esta trava antes de criar um registro no CRM. Não existe caixa isenta da checagem, hoje ou depois.

Este documento é a especificação da trava. Ainda **não foi construído** em lugar nenhum (nem Zoho Flow, nem CRM) — é o desenho a ser usado quando os Fluxos para `vendas@` e `contato@` forem montados no Zoho Flow, mantendo a DP-05 (motor único = Zoho Flow, nunca rotina agendada externa).

## Por que isso importa

O incidente de 23/08/2026 (ver `estado-automacao-petronect.md`) aconteceu porque a checagem de duplicidade comparava o nome do negócio inteiro, e não uma chave estável — resultado: 34 oportunidades recriadas, um registro `None`, 126 registros com `Lead_Source` reescrito. Ampliar a criação automática de oportunidades para mais caixas, incluindo caixas que recebem e-mail de clientes diversos (não só Petronect), multiplica esse risco se a chave de dedup não for definida com o mesmo rigor.

## Três situações diferentes

### 1. E-mail vem de um portal com número de oportunidade (ex.: Petronect)

- Chave forte: campo dedicado e único no CRM (hoje `Numero_Petronect`; restrição de unicidade ativa desde 25/08).
- Antes de criar: `Search Records` no CRM por esse número. Achou → não cria, atualiza/anexa ao existente. Não achou → cria.
- Essa checagem já é global (contra o CRM inteiro, não só "a caixa que processou"), então funciona mesmo que o mesmo e-mail chegue em várias caixas ao mesmo tempo — é só isso que já protege `jbueno@` e `gmartinez@` hoje.
- Se entrar outro portal com número de processo no futuro (ComprasNet, BEC, etc.), replicar o mesmo padrão: um campo próprio, único, com o número extraído do assunto/corpo.

### 2. Cliente direto sem portal — inclui prospecção fria (o caso de `vendas@` e `contato@`) — **decisão fechada em 25/08**

**Confirmado pelo usuário:** `vendas@` e `contato@` recebem tanto cliente que já compra da Autiam quanto prospecção fria (empresa nova). Nos dois casos, **cria-se a oportunidade direto no módulo Negócios** — sem passar por Leads — com **status inicial equivalente a "Pendente Análise"** (campo `Status_Petronect`, rótulo em processo de renomear para **"Status da Oportunidade"** — ver `decisoes-pendentes-checklist.md`, item 14), aguardando o usuário decidir Cotar ou Declinar. A partir dessa decisão, segue o mesmo fluxo já desenhado para as demais oportunidades (tarefa de triagem, depois Cotado ↔ tarefa de proposta / Declinado ↔ Estágio Cancelado).

Não existe um número de portal para essas oportunidades, então a Autiam cria o próprio identificador:

**Convenção de nome:** `<NOME_CLIENTE> <AAMMSSSS>` — exemplo dado pelo usuário: `DELL 26080010`. Decompondo: `DELL` = nome do cliente solicitante; `26` = ano (2026); `08` = mês (agosto, informativo — não reinicia o contador); `0010` = sequencial da oportunidade não-portal (4 dígitos, zero-padded).

**Mecânica do sequencial — fechada em 25/08: reinício anual.** O `SSSS` reinicia uma vez por ano (em janeiro volta a `0001`), e **não** reinicia a cada mês — o `MM` no código é só informativo (mês de criação), não é fronteira de reinício. Exemplo: depois de `26080010` (10ª de 2026, criada em agosto), a próxima em setembro/2026 é `26090011` (11ª de 2026, segue contando); a primeira de janeiro/2027 volta a `27010001`.

**Implementação:** como o reinício é anual, dá para usar o **Auto Number nativo do Zoho CRM** com prefixo composto pelo ano (ex.: prefixo `{{ano}}`, reiniciado manualmente uma vez por ano) **ou** uma função Deluge simples que verifica o ano da última oportunidade criada e decide se reinicia — mais simples que a hipótese de reinício mensal descartada.

**Campo criado:** `Codigo_Oportunidade_Direta` (texto, único — mesma solução já usada para `Numero_Petronect`, já que `Deal_Name` é campo de sistema e não pode virar único). `Deal_Name` segue a convenção `<CLIENTE> <AAMMSSSS>` por convenção visual; quem protege contra duplicata é o campo novo.

**Importante — o identificador não substitui a checagem de dedup, só nomeia o que já foi decidido como novo.** Como o código só é atribuído no momento da criação (não vem pronto no e-mail, ao contrário do número Petronect), ele não serve para *detectar* duplicata antes de criar — para isso continuam valendo as camadas abaixo (remetente, empresa, número citado em texto livre). O código serve para identificar/nomear a oportunidade depois que a checagem de dedup já decidiu que é caso novo.

**Camada 1 — Remetente exato + negócio aberto, dentro de uma janela de dias.** **Janela confirmada pelo usuário em 25/08: 45 dias.**
O que essa camada resolve: evitar que um e-mail de reforço/continuação do mesmo assunto (ex.: cliente escreve de novo "e aí, já tem a cotação?" sobre o mesmo pedido) seja tratado como oportunidade nova. Buscar Negócios vinculados ao mesmo e-mail de remetente (via Contato) que estejam em estágio aberto (não Cancelado/Ganho/Perda) e criados nos últimos **45 dias**. Achou dentro da janela → não cria negócio novo, trata como mensagem sobre a oportunidade existente (mesma lógica do ramo "Sala"). Não achou (ou achou, mas fora da janela) → segue para a Camada 2/3, tratando como possível oportunidade nova.

**Camada 2 — Mesma empresa (Conta), sem negócio aberto do mesmo remetente — fechado em 25/08: criação por confirmação humana, via botão.**
Se o remetente é novo mas o domínio do e-mail bate com uma Conta já cadastrada, e essa conta tem negócio(s) aberto(s) recente(s), a automação **não cria o negócio sozinha**. Ela gera uma tarefa/notificação "Revisar possível duplicata — e-mail de `<remetente>` parece com `<negócio existente>`" para o responsável. **Se, ao revisar, for de fato uma oportunidade nova**, o responsável usa o **mesmo botão** do item 8 (`guia-construcao-zoho-flow-consolidado.md`) — só que, neste caso, antes de mover o e-mail e abrir CRM/WorkDrive, o botão **primeiro gera o nome/código da oportunidade** (convenção `<CLIENTE> <AAMMSSSS>`, próximo sequencial anual, campo `Codigo_Oportunidade_Direta`) e cria o Negócio com status inicial "Pendente Análise" — só depois disso segue com ler/mover/marcar como lido/abrir CRM/abrir pasta do WorkDrive, exatamente como no caso de oportunidade já existente.

**Camada 3 — Número de oportunidade citado em texto livre**, mesmo fora de portal formal (ex.: cliente escreve "referente ao pedido 12345" ou "cotação COT-2026-081" no corpo do e-mail). Se um número for identificável, tratar como Camada 1 (chave forte), independente de empresa/remetente. Extrair esse número de texto livre é a única parte desta trava que pode precisar do [CLAUDE] (mesmo padrão já usado para outros campos de texto livre); tudo o resto é busca + condição, determinístico, cabe inteiro no Zoho Flow.

## Campos novos necessários (proposta — ainda não criados no CRM)

| Campo | Tipo | Uso |
|---|---|---|
| `Codigo_Oportunidade_Direta` | Texto, único | Identificador `<CLIENTE> <AAMMSSSS>` das oportunidades sem portal (vendas@/contato@), sequencial reinicia por ano |
| `Chave_Dedup_Origem` | Texto | Guarda o valor usado na checagem (número extraído, ou `remetente:email@dominio.com`) — auditoria e revisão manual |
| `Possivel_Duplicata` | Checkbox | Marca os casos de Camada 2, para a tarefa de revisão e para o botão saber que aquele registro está pendente de confirmação |

Reaproveita `Origem_Automa_o` (já existente) para registrar qual fluxo/mecanismo criou o registro (inclusive "Manual - Botão"), e `Status_Petronect`/"Status da Oportunidade" (já existente) para a decisão Cotar/Declinar — não cria campo novo para nenhum dos dois.

## Quem executa o quê

Toda a lógica é determinística (busca + condição), cabe inteira no Zoho Flow, sem violar a DP-05. A única exceção é a extração de número de texto livre na Camada 3, que pode chamar o Claude via HTTP de dentro do Flow — igual ao padrão já usado no resto do desenho. A criação por botão (Camada 2) é ação humana, disparando um Instant Flow/Webhook — ver `guia-construcao-zoho-flow-consolidado.md`, item 7.

## Em aberto

Nenhuma decisão de negócio pendente nesta frente. Falta só criar os campos no CRM (via API, `createFields`) e montar a lógica no Zoho Flow.

## Resolvido em 25/08/2026

- ~~Camada 2 deve criar direto em Negócios, ou em Leads primeiro?~~ **Fechado: direto em Negócios**, tanto para prospecção fria quanto cliente recorrente, com status inicial "Pendente Análise".
- ~~`vendas@`/`contato@` recebem só cliente recorrente ou também prospecção fria?~~ **Fechado: os dois.**
- ~~Janela de dias da Camada 1?~~ **Fechado: 45 dias.**
- ~~Camada 2 cria automaticamente e só sinaliza, ou espera confirmação humana?~~ **Fechado: espera confirmação — criação via botão**, não automática.
- ~~Esse botão também cria a oportunidade quando ela não existe, ou a criação é por outro caminho?~~ **Fechado: é o mesmo botão** — quando a oportunidade não existe ainda, ele gera o nome/código primeiro, cria o Negócio, e só depois segue com o resto da ação (mover e-mail, marcar como lido, abrir CRM, abrir WorkDrive).
- ~~Mecânica do sequencial `SSSS`?~~ **Fechado: reinício anual** (não mensal). `MM` no código é só informativo.
