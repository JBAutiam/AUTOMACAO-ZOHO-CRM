# Análise crítica do "WORKFLOW PETRONECT.docx" (19/08/2026)

Documento analisado: **Workflow Completo: Processo de Cotação Autiam via Petronect (Governança, Audit Trail e Loops de Revisão)** — 21 passos em 6 fases, enviado pelo usuário em 19/08/2026. Entregue ao usuário como `Analise_Critica_Workflow_Petronect.docx` (9 páginas, 28 achados).

Esta é a versão de referência dos achados. O workflow proposto pelo usuário é o desenho-alvo; o doc `claude/estado-automacao-petronect.md` descreve o que já existe em produção. **Os dois divergem em vários pontos — a lista abaixo é a ponte entre eles.**

## Estrutura do workflow proposto (resumo)

- **Fase 1** (auto): gatilho por e-mail OU agendamento 2h → Zoho Flow autentica na Petronect, cria Deal, extrai itens, baixa editais, cria pastas no WorkDrive espelhadas no Mail.
- **Fase 2**: IA lê PDFs, salva Resumo do Certame → **Hold Point 1** (Go/No-Go, tarefa "Cotar ou Declinar").
- **Fase 3**: precificação interna ou RFQ a parceiros (pastas DOCS FORNECEDOR/<nome>, draft de e-mail) → **Hold Point 2** (revisar e enviar RFQ).
- **Fase 4**: loop condicional de dúvida técnica → status "Aguardando Esclarecimento Petrobras"; saídas: Petrobras responde, ou humano assume premissa registrada.
- **Fase 5**: retorno dos parceiros → **Hold Point 3** (elaborar proposta), pastas PROPOSTA TECNICA/COMERCIAL, submissão no portal, status "Enviada".
- **Fase 6**: loop de adendo pós-submissão → reabre Deal ("Revisão Necessária Petrobras"), retorna obrigatoriamente ao Passo 13, versionamento REV.

## Achados (28)

### A — Bloqueadores técnicos (Alta)

- **A1. A Fase 1 pressupõe API da Petronect que provavelmente não existe.** As notificações instruem acessar o portal com usuário e senha; o edital não vem anexado no e-mail. Sem API, os Passos 2–6 e o Passo 9 (IA lê PDFs) caem juntos. Opções: RPA autenticado, ou download do edital como passo humano no Hold Point 1 (recomendado para começar).
- **A2. "A IA lê os PDFs" não tem motor definido.** Zoho Flow não faz isso nativamente. Falta definir motor, custo e — principalmente — o fallback quando a extração falha. Resumo errado é pior que resumo nenhum, porque o Hold Point 1 decide em cima dele. Marcar sempre como "não conferido" + link para o PDF de origem.
- **A3. Dois motores disputando o mesmo gatilho.** O doc nomeia Zoho Flow; já existe rotina agendada a cada 3h fazendo parte do mesmo trabalho (trigger `trig_01UgCozpX8hBhGxLWRAnBTXr`). Escolher um e desligar o outro no mesmo dia — paralelo = oportunidade duplicada.

### B — Conflitos com o que já está implementado

- **B1 (Alta). Três geradores da mesma tarefa de triagem**: Passo 11 do doc + rotina agendada + regra "rede de segurança" planejada. Eleger só a regra de workflow nativa do CRM (pega também cadastro manual — caso das 35 criadas em 19/08).
- **B2 (Alta). Gatilho "Opção A ou B" sem idempotência.** Três assuntos diferentes da Petronect podem se referir à mesma oportunidade. Fixar o número Petronect como chave única e checar antes de criar; idealmente marcar Deal_Name como único.
- **B3 (Alta). Status citados não existem no CRM.** "Aguardando Esclarecimento Petrobras", "Revisão Necessária Petrobras", "Enviada" não estão em `Status_Petronect` (3 valores) nem em Stage. Separar eixos: Status Petronect = decisão comercial; Stage = funil; criar terceiro campo "Situação Operacional" para os estados de processo.
- **B4 (Alta). `Fornecedor_Parceiro` é lookup único, mas o doc prevê múltiplos parceiros.** Criar subform ou módulo "Cotações de Parceiro": fornecedor, data RFQ, data retorno, preço, prazo, moeda, situação. Resolve também C8.
- **B5 (Média). "Número da Proposta" na nomenclatura conflita com a chave real** (número da oportunidade, usado em tudo, inclusive nos e-mails com parceiros). Padronizar `RFQ <oportunidade> _ <Parceiro> [_REV x]`.
- **B6 (Média). Espelhamento automático colide com as 90 pastas legadas** (sufixos descritivos como "7004606740 - 152 PIT", pastas dentro de PERDIDOS). Busca de pasta tem que ser por prefixo e recursiva; nunca renomear/mover pasta existente.
- **B7 (Média). "Pop-up na tela" não é recurso do Zoho CRM.** Trocar por tarefa prioridade Muito Elevada + alerta e-mail + Zoho Cliq.

### C — Fluxos faltantes (o bloco mais crítico)

- **C1 (Alta). O caminho mais frequente não existe no fluxo: o parceiro não cota.** Doc vai do Passo 15 ao 17 como se a resposta fosse certa; na planilha de triagem, a maioria dos 48 declínios é "Nirmal/Microsensor não cotou". Criar prazo de resposta do parceiro, cobrança em D+X e saída "Parceiro não cotou/recusou" → Declinado com motivo, sem passar pelos Hold Points 2 e 3.
- **C2 (Alta). Mensagens de SALA (inbound) não têm lugar.** A Fase 4 só cobre dúvida iniciada pela Autiam. Caso real 7004641195 (17/08) foi tratado manualmente. Rotear por status: Cotada → tarefa urgente 1 dia útil + ciência; Declinada → arquivar; Pendente → antecipar Hold Point 1.
- **C3 (Alta). O funil nunca fecha — não há fluxo para o resultado do certame.** Workflow termina em "Enviada" e só se move por adendo. Sem ganho/perda não há taxa de conversão, forecast nem insumo para as campanhas de marketing que são o objetivo do projeto. Criar Fase 7 (Encerramento) a partir do "Relatório Divulgado ID <n>", hoje ignorado pela rotina. **Fluxo faltante mais caro do documento.**
- **C4 (Média). Prorrogação de prazo sem tratamento.** "Oportunidade Prorrogada ID <n>" deve atualizar `Data_Limite_Proposta` do Negócio existente, não criar registro novo.
- **C5 (Alta). Visita técnica obrigatória não aparece.** Evidência: e-mail "Solicitação de Agendamento de Visita Técnica — Licitação Nº 7004613226", com CPF dos profissionais para portaria. Perder a visita = desclassificação. Extrair "exige visita? data limite" no Passo 9 e criar Hold Point condicional com prazo próprio.
- **C6 (Alta). Exigência de comprovação "IDÊNTICO NÃO" fora do fluxo.** A Autiam quase sempre oferta equivalente (Nirmal/Microsensor/Truedyne no lugar da marca de referência), e a Petrobras exige documento oficial do fabricante. Virar item de checklist bloqueante do Hold Point 3.
- **C7 (Média). Cadastro/habilitação (CRCC) fora do escopo.** Remetentes administrativos hoje são "ignorar", mas por ali chegam avisos de vencimento de cadastro que barram participação. Separar "ignorar" (newsletter) de "rotear" (cadastro/documentação).
- **C8 (Média). Não há passo de comparação e seleção entre parceiros**, nem registro da justificativa da escolha.

### D — Lacunas de governança

- **D1 (Alta). Nenhum Hold Point tem prazo, alerta ou escalonamento.** O prazo do edital corre durante a pausa → falha silenciosa. Amarrar à `Data_Limite_Proposta`: alerta a 50%, escalonamento a 80%, crítico a 90%. Nota: o prazo para pedir esclarecimento encerra ANTES do prazo da proposta — são duas datas e o fluxo só conhece uma.
- **D2 (Alta). Não existe alçada de aprovação por valor.** Propostas de R$ 6.900 (7004620843) e R$ 3.000.000 (7004607160) seguem o mesmo rito com o mesmo decisor. Criar faixas (sugestão inicial: > R$ 250 mil exige coordenação) via processo de aprovação nativo do Zoho.
- **D3 (Média). "Inside Sales" é o único papel — não há RACI** nem substituto definido. Primeira ausência trava a operação inteira, porque ele decide em todos os Hold Points.
- **D4 (Média). Audit trail sem modelo de dados.** Sete menções a "[Audit Trail]: Sistema registra…" sem dizer onde grava. Especificar os campos de data/hora no módulo Negócios + ligar histórico nativo em Status_Petronect e Data_Limite_Proposta.
- **D5 (Média). Sem tratamento de erro nem monitoramento.** O modo de falha mais provável de uma automação é parar em silêncio. Definir retentativa, destino do que falhou e resumo periódico de execução.
- **D6 (Média). Preço de parceiro e proposta sem controle de acesso** no WorkDrive — risco de um parceiro ver preço do outro. Definir permissões ANTES de criar a estrutura.

### E — Regras de negócio ausentes (Média)

- **E1. Câmbio e validade da cotação do parceiro não são tratados.** Parceiros cotam em moeda estrangeira; Passo 19 resume a "preenchimento de planilhas". Registrar moeda, taxa, data da taxa e validade; na Fase 6, cotação vencida exige recotação.
- **E2. Custos de importação fora da precificação** (II, frete internacional, despachante, seguro, nacionalização). Há trabalho recorrente sobre NCM/alíquota por e-mail, fora de qualquer controle de prazo.
- **E3. Motivo do declínio não estruturado.** Criar picklist "Motivo do Declínio" obrigatório quando Status = Declinado. Opções já observadas: parceiro não cotou, fora da linha, fora de range/especificação, prazo, preço não competitivo, perdemos para concorrente.
- **E4. Itens cotados não entram no CRM — não haverá histórico de preço.** A Autiam recota as mesmas famílias (transmissores de pressão, PSVs, reguladoras). Avaliar módulos Produtos/Cotações antes que a base cresça.

### F — Ambiguidades e correções pontuais

- **F1 (Média). Todo adendo obriga refazer a cotação inteira** (Passo 21, "RETORNA OBRIGATORIAMENTE ao Passo 13"). Inserir triagem de impacto: altera especificação/quantidade/condição comercial? Parceiros respondem em dias/semanas — refazer por adendo irrelevante pode custar a submissão.
- **F2 (Média). Versionamento REV ambíguo** — letra por documento ou por oportunidade? Amarrar ao Negócio: o adendo incrementa um contador e todos os documentos daquele ciclo carregam a letra.
- **F3 (Média). Fonte de verdade WorkDrive vs Mail não definida.** Recomendação: WorkDrive = repositório de trabalho; pasta do Mail = arquivo do e-mail original apenas.
- **F4 (Baixa). Passos 2 a 6 comprimidos em uma linha** — cinco pontos de falha independentes escondidos, justo num documento de governança.
- **F5 (Baixa). Não se define quando a oportunidade NÃO deve virar Negócio.** Filtro de aderência por família de produto antes do Hold Point 1, criando o Deal já com sugestão de Declinar.

## O que o documento acerta (preservar)

Hold Points nos pontos irreversíveis (decidir cotar, enviar RFQ, submeter); intenção de audit trail por transição; ciclo de revisão da Fase 6 (edital muda depois do envio — problema real e normalmente ignorado); saída B do loop de esclarecimento (assumir premissa registrada quando não há tempo hábil — maduro); espelhamento de estrutura entre repositório e e-mail.

## Sequência recomendada ao usuário

1. Responder **A1** (existe API?) e **A3** (qual motor) — sem isso, implementar a Fase 1 é risco puro.
2. Resolver o modelo de dados: **B3, B4, E3** — mudanças estruturais ficam mais caras a cada registro (já são 111).
3. Fechar os fluxos faltantes de maior impacto: **C1, C3, C2**.
4. Só então Hold Points com prazos (**D1**) e alçada (**D2**).
5. Melhoria contínua: **E4, C8, F5**.
