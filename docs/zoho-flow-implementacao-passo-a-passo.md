# Passo a passo — Automação EMAIL_CRM Petronect via Zoho Flow + Claude

Última atualização: 21/08/2026 (revisão 3 — confirmado conector nativo Zoho WorkDrive no Zoho Flow)

Fluxograma no Zoho Vani (espaço "Automação EMAIL_CRM - Fluxograma Petronect", equipe AUTIAM), já atualizado com as correções abaixo:
https://app.vanihq.com/edition/909221261/space/12454000000033002/zone/0a3b39cc-45f6-4510-ba23-2ac3f357e05b

## O que mudou nesta revisão

1. Criados 3 campos novos no CRM para controle (ver seção "Campos de controle criados").
2. Fechada a regra do item 6.3 (relatório de oportunidade Declinada) — estava incompleta na revisão anterior.
3. Fechada a regra para mensagem de sala chegando numa oportunidade ainda `Pendente Análise` (item 5.2).
4. Corrigido o item 5.5 ("registrar ciência") — não precisa de campo novo, basta concluir a tarefa urgente.
5. O fluxograma no Vani foi atualizado nos nós de criação de Negócio, criação de Tarefa, tarefa urgente e "ciência", refletindo os campos novos.

## Campos de controle criados no CRM (hoje, via API)

| Módulo | Campo | API name | Tipo | Uso |
|---|---|---|---|---|
| Negócios (Deals) | Origem Automação | `Origem_Automa_o` | Picklist: Zoho Flow / Rotina Cowork / Manual | Registra qual automação criou/processou o registro |
| Negócios (Deals) | Tarefa Triagem Criada | `Tarefa_Triagem_Criada` | Checkbox (boolean) | Impede que Zoho Flow e a rotina do Cowork criem a tarefa de triagem duas vezes para a mesma oportunidade |
| Tasks | Tipo Tarefa Petronect | `Tipo_Tarefa_Petronect` | Picklist: Triagem / Elaborar Proposta / Urgente - Sala | Classifica por que a tarefa foi criada, para filtrar/relatar depois |

**Nota sobre o api_name `Origem_Automa_o`**: o Zoho gerou esse nome automaticamente a partir do rótulo "Origem Automação" e engoliu os acentos de um jeito estranho (ficou sem o "ca" antes do "o"). A API do Zoho não permite renomear nem excluir campos — só dá pra corrigir isso pela interface (igual ao caso do campo obsoleto `Data_Limite_Petronect`, já documentado). Funciona normalmente, só o nome interno é feio. Se quiser, posso te passar o passo a passo pra recriar com um nome mais limpo mais tarde.

---

## Como ler o restante do documento

Cada etapa está marcada:
- **[ZOHO FLOW]** — condicional/ação nativa do Zoho Flow, sem IA. Determinística, roda sozinha, não depende de nenhuma sessão do Claude estar ativa.
- **[CLAUDE]** — exige julgamento ou geração de texto. Ou fica rodando aqui no Cowork (rotina agendada já ativa), ou é chamado de dentro do Zoho Flow via HTTP Module apontando para a Anthropic Messages API (API Key própria, cobrada por uso).

---

## 0. Pré-requisitos antes de montar o fluxo

1. Conectores nativos no Zoho Flow: Zoho Mail, Zoho CRM e **Zoho WorkDrive confirmado pelo usuário** (21/08/2026) — a ação de criar pasta por oportunidade pode ser feita nativamente, sem HTTP Module.
2. Criar **duas conexões de Zoho Mail dentro do Zoho Flow**: uma autenticada como `jbueno@autiam.com`, outra com o login do próprio `gmartinez@autiam.com` (resolve o problema de acesso — a delegação de caixa não aparece pela API/MCP, testei e confirmei, mas o Zoho Flow autentica cada conta separadamente).
3. Se for usar as etapas **[CLAUDE]** dentro do próprio Zoho Flow, gerar uma API Key da Anthropic em console.anthropic.com (cobrança separada, por uso).

## 1. Gatilho — dois fluxos idênticos, um por caixa

**[ZOHO FLOW]**
- App: Zoho Mail · Trigger: "New Email" na pasta Caixa de Entrada.
- Fluxo A: conexão `jbueno@autiam.com`, folderId `3068981000000008014`.
- Fluxo B: conexão `gmartinez@autiam.com`, Caixa de Entrada dele.
- Mesma lógica nos dois. Use "Reusable Flow"/subfluxo do Zoho Flow se disponível, para não duplicar a lógica.

## 2. Filtro de remetente

**[ZOHO FLOW]** — Condition step
- Remetente contém `petronect.com.br` **E** não contém `cadastropetrobras@`, `notificacao@`, `manutencao@`, `vocesabia@` **E** não é pessoa física conhecida `@petronect.com.br`.
- Se falso → encerra (ignora o e-mail).

## 3. Classificação por assunto

**[ZOHO FLOW]** — Router com 4 ramos
- "Nova Oportunidade": assunto contém `Criação de Oportunidade ID` OU `Oportunidade Publicada` OU `Oportunidade Prorrogada ID`
- "Sala": assunto começa com `SALA `
- "Relatório": assunto contém `Relatório Divulgado ID`
- "Outro": nenhum dos anteriores → ignora

---

## 4. Ramo "Nova Oportunidade"

**4.1 [CLAUDE] (só se necessário) — Extrair campos do texto livre**
Testar primeiro só com regex/extrator de texto nativo do Zoho Flow. Só acionar o Claude se o padrão não bater.

**4.2 [ZOHO FLOW] — Checar duplicidade**
Zoho CRM → Search Records, Negócios, por número extraído.
- Não encontrado → 4.3
- Encontrado → pula direto para 4.4 (a própria 4.4 agora checa o campo de controle antes de criar tarefa, então não duplica)

**4.3 [ZOHO FLOW] — Criar o Negócio** *(atualizado)*
Zoho CRM → Create Record, Negócios:
- Nome do Negócio = `<número> - <objeto>`
- Conta = PETRONECT (id `6670502000002997001`)
- Lead_Source = `Portal Eletrônico`
- Status_Petronect = `Pendente Análise`
- Data_Limite_Proposta = data extraída (se houver)
- **`Origem_Automa_o` = `Zoho Flow`** (campo novo — identifica quem criou o registro)

**4.4 [ZOHO FLOW] — Criar a tarefa de triagem, com trava de duplicidade** *(atualizado)*
- Se `Tarefa_Triagem_Criada` do Negócio = verdadeiro → **não faz nada**, pula para 4.5 (a tarefa já existe, criada por este fluxo ou pela rotina do Cowork).
- Se falso → Zoho CRM → Create Record, Tasks:
  - Subject = `Revisar oportunidade Petronect <número> - definir Cotado/Declinado`
  - What_Id = id do Negócio
  - Owner = Guilherme Martinez · Status = `Not Started` · Priority = `Normal`
  - Due_Date = hoje + 2 dias úteis
  - Description = referência à pasta de origem do e-mail
  - Send_Notification_Email = `true`
  - **`Tipo_Tarefa_Petronect` = `Triagem`** (campo novo)
  - Depois de criar → Update Record no Negócio: **`Tarefa_Triagem_Criada` = `verdadeiro`**

**4.5 [ZOHO FLOW] — Arquivar o e-mail**
Zoho Mail → confirmar/criar pasta `/Inbox/NEGOCIOS/PETRONECT/<número>` → mover mensagem → marcar como lida.

---

## 5. Ramo "Sala" (assunto `SALA <n>`)

**5.1 [ZOHO FLOW] — Localizar o negócio**
Zoho CRM → Search Records, Negócios, pelo número da sala.

**5.2 [ZOHO FLOW] — Checar Status_Petronect** *(fechado nesta revisão)*
- `Cotado` → 5.3
- `Declinado` → 5.6
- `Pendente Análise` → **decisão fechada**: trata como alerta. Cria a mesma tarefa de 4.4 antecipadamente (What_Id=negócio, Owner=Guilherme, `Tipo_Tarefa_Petronect`=`Triagem`, Priority=`High` em vez de `Normal` para sinalizar a urgência, Due_Date = hoje + 1 dia útil) e marca `Tarefa_Triagem_Criada`=verdadeiro, para que 4.4 não crie outra depois. Motivo: já chegou mensagem na sala antes da triagem acontecer, então a decisão Cotar/Declinar fica mais urgente.

**5.3 [ZOHO FLOW] — Criar tarefa urgente** *(atualizado)*
Zoho CRM → Create Record, Tasks: Subject = `Mensagem urgente - Sala <número>`, What_Id = negócio, Priority = `Highest`, Due_Date = hoje + 1 dia útil, Owner = responsável do negócio, **`Tipo_Tarefa_Petronect` = `Urgente - Sala`**.

**5.4 [CLAUDE] (opcional) — Avaliar se a mensagem pede resposta e redigir rascunho**
Só adicione se quiser respostas assistidas por IA. Sem essa etapa, a tarefa de 5.3 já obriga um humano a olhar e responder.

**5.5 [ZOHO FLOW] — Registrar "ciência"** *(corrigido — não precisa de campo novo)*
Guilherme conclui a tarefa urgente criada em 5.3 (`Status` = `Completed`). Isso já é o registro de que ele leu e reconheceu a mensagem — não é preciso criar campo/nota separado.

**5.6 [ZOHO FLOW] — Arquivar (caso Declinado)**
Zoho Mail → mover o e-mail para a pasta da oportunidade, sem criar tarefa.

---

## 6. Ramo "Relatório de Divulgação"

**6.1 [ZOHO FLOW] — Localizar negócio + checar status** (igual 5.1/5.2, sem o caso "Pendente Análise" — relatório de divulgação só existe para oportunidade já publicada/decidida)

**6.2 Se `Cotado`: [CLAUDE] — Resumir o relatório e notificar o usuário para leitura imediata**
Gera um resumo curto e manda como notificação de alta prioridade (e-mail/push), já que um fluxo em segundo plano não consegue literalmente "trazer a mensagem pra tela".

**6.3 Se `Declinado`: [ZOHO FLOW] — Arquivar, sem tarefa e sem notificação** *(fechado nesta revisão)*
**Decisão**: mesma lógica do item 5.6 (sala declinada) — Zoho Mail move o e-mail para a pasta da oportunidade, sem criar tarefa e sem notificar ninguém. Motivo: a oportunidade já foi declinada, então um relatório de divulgação dela não exige nenhuma ação — só fica arquivado para consulta futura, se precisar.

---

## 7. O que continua sendo feito pelo Claude, fora do Zoho Flow

- Extração de texto livre quando o padrão varia (4.1)
- Decidir se uma mensagem de sala pede resposta e redigir o rascunho (5.4)
- Resumir o relatório para leitura imediata (6.2)
- A rotina agendada que já roda aqui no Cowork (trigger `trig_01UgCozpX8hBhGxLWRAnBTXr`, a cada 3 horas) — **atualizada hoje** para também usar os 3 campos de controle novos, então já pode rodar em paralelo ao Zoho Flow sem duplicar oportunidade ou tarefa.

## 8. Riscos e observações (atualizado)

- **Duplicidade — mitigada, não 100% eliminada**: com `Origem_Automa_o` e `Tarefa_Triagem_Criada`, tanto o Zoho Flow quanto a rotina do Cowork checam o mesmo registro antes de criar Negócio/Tarefa, então rodar os dois ao mesmo tempo não deve mais gerar duplicata — exceto num caso raro de corrida (os dois sistemas processando o mesmíssimo e-mail no mesmíssimo segundo, antes de qualquer um marcar o campo). Ainda assim, recomendo testar o Zoho Flow isolado por alguns dias antes de considerá-lo definitivo.
- **Fechado nesta revisão**: Zoho Flow tem conector nativo para Zoho WorkDrive (confirmado pelo usuário em 21/08/2026) — a criação de pasta por oportunidade no fluxo do TeamInbox/WorkDrive pode usar a ação nativa "Create Folder" do conector, sem precisar de HTTP Module/API manual para essa parte.
- O api_name feio `Origem_Automa_o` (ver nota acima) é cosmético, não funcional.
