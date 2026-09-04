# Workflow Petronect v2 — processo corrigido e plano de implementação

Criado em 19/08/2026. Entregue ao usuário como `Workflow_Petronect_v2_Plano_Implementacao.docx` (24 páginas).

**Substitui** o "WORKFLOW PETRONECT.docx" (v1) enviado pelo usuário. Incorpora os 28 achados de `claude/analise-critica-workflow-petronect.md`. Este é agora o **desenho-alvo oficial** do projeto; `claude/estado-automacao-petronect.md` descreve o que está em produção hoje.

## Fato decisivo confirmado

`getOrganization` retornou `paid_type: "zohooneenterprise"` — **Zoho One Enterprise**, confirmado pelo usuário. Isso libera Blueprint, módulos customizados, funções Deluge agendadas, processos de aprovação, regras de validação/layout, WorkDrive, Flow e Cliq. `getModules` confirmou **Blueprint suportado no módulo Deals**.

Consequência de arquitetura: quase toda a governança que a v1 queria construir com automação já existe nativa no Zoho — e melhor, porque o Blueprint grava trilha de auditoria sozinho.

**Princípio de divisão adotado:** Zoho faz o determinístico (estados, prazos, obrigatoriedade, alçada, tarefas, alertas, cobrança, cálculo, relatórios, auditoria). Claude faz só o interpretativo (entender e-mail em linguagem livre, ler edital em PDF, interagir com o portal que não tem API). Regra de ouro: se a decisão pode ser escrita como condição, é do Zoho.

## Arquitetura da v2

**Blueprint no módulo Negócios**, campo `Situacao_Petronect`, critério de entrada Conta = PETRONECT (não afeta os demais negócios do CRM). Os Hold Points da v1 viraram transições de Blueprint.

**14 estados:** Captada → Em Triagem → (Declinada | Em Especificação) → Aguardando Cotação Parceiro → Em Precificação → Em Aprovação Interna → Pronta para Submissão → Proposta Enviada → (Em Revisão por Adendo | Encerrada-Ganha | Encerrada-Perdida). Mais: Aguardando Esclarecimento Petrobras (exceção, salva estado de origem) e **Encerrada - Sem Proposta** (estado novo que obriga nomear por que a oportunidade travou — permite medir perda por inação/prazo/parceiro).

**17 transições (T1–T17)**, cada uma com: quem pode executar, campos obrigatórios na transição, ações posteriores. Status_Petronect e Stage passam a ser atualizados pelo Blueprint, não à mão.

**SLA sempre relativo à `Data_Limite_Proposta`** (percentual do prazo consumido: alerta 50%, escalonamento 80%, crítico 90%) — nunca em dias fixos, porque edital de 30 dias e de 3 dias não têm a mesma régua.

**9 fases:** 1 Captação (auto) · 2 Triagem/Go-No-Go · 3 Especificação e RFQ · 4 Retorno dos parceiros (inclui o caminho "parceiro não cotou") · 5 Precificação/aprovação/submissão · 6 Adendo com triagem de impacto · 7 **Encerramento (nova)** · 8 Mensagens de SALA (exceção) · 9 Esclarecimento Petrobras (exceção).

## Modelo de dados especificado

- **Ajustes no que existe:** Deal_Name único; adicionar `Portal Eletrônico` à picklist Lead_Source; remover Data_Limite_Petronect do layout; Fornecedor_Parceiro passa a significar o parceiro vencedor.
- **~45 campos novos em Negócios**, agrupados: controle de processo (Situacao_Petronect, Objeto_Resumido, Aderencia_Linha, Origem_Registro, ID_Mensagem_Origem, Link_WorkDrive…), prazos e exigências (Data_Limite_Esclarecimento, Exige_Visita_Tecnica, Dias_Para_Prazo, Faixa_Criticidade…), decisão e desfecho (Motivo_Declinio, Motivo_Perda, Concorrente_Vencedor, Impacto_Adendo…), custo landed (Moeda_Origem, Taxa_Cambio_Utilizada, Data_Taxa_Cambio, 4 custos, Custo_Total_Landed, Margem_Percentual), conformidade (Tipo_Oferta, Doc_Fabricante_Anexado, Numero_Revisao, Sufixo_Revisao), premissa (4 campos), e **13 carimbos de auditoria `AT_*`**.
- **3 módulos customizados novos:** `Cotações de Parceiro` (N por oportunidade — resolve o lookup único; módulo e não subformulário porque subform não dispara workflow, e a cobrança automática depende disso), `Log de Auditoria Petronect` (somente leitura, escrita só por função/API), `Documentos de Habilitação` (validade + alerta).
- Picklists completas definidas (Motivo do Declínio com 9 valores, Motivo da Perda com 7, etc.).

## Configuração nativa especificada

10 regras de validação (VR-01 a VR-10), 8 regras de layout (LR-01 a LR-08), 10 regras de workflow (WF-01 a WF-10), processo de aprovação AP-01 por faixa de valor, 6 funções Deluge (FN-01 a FN-06, três delas agendadas diariamente), 9 relatórios/painel, e regras de acesso no WorkDrive.

## Automação Claude (só 5 rotinas)

- **A** — triagem de e-mails (a cada 2h): classifica, cria em Captada, nunca decide Cotar/Declinar, idempotente pelo número, prorrogação atualiza em vez de duplicar, pasta por prefixo, falha não consome o e-mail.
- **B** — mensagens de SALA, roteadas pelo estado da oportunidade.
- **C** — relatório de resultado (não encerra sozinha; encerramento é transição humana).
- **D** — leitura do edital (resultado sempre marcado "não conferido"; campo vazio em vez de campo errado).
- **E** — health-check diário (transforma ausência em alerta).

## Auditoria — 5 camadas

1. Histórico do Blueprint (transições, autor, tempo em cada estado) — nativo
2. Histórico de campo nos 4 campos sensíveis — nativo
3. Carimbos `AT_*` em campo filtrável (porque histórico nativo não entra em relatório)
4. Log de Auditoria (eventos de automação — catálogo de 29 eventos com origem e status)
5. Anexos e notas (evidência documental)

**Teste de aceite:** pegar uma oportunidade ao acaso e reconstruir quem decidiu cotar e quando, quais parceiros foram consultados e o que responderam, qual câmbio foi usado e em que data, quem aprovou, quem submeteu, e por que ganhou/perdeu — sem abrir planilha nem perguntar a ninguém.

## Plano em 6 ondas

- **Onda 0** — 5 decisões pendentes (ver abaixo)
- **Onda 1** — estrutura de dados (campos, 3 módulos, backfill de Origem_Registro nos 111 registros)
- **Onda 2** — Blueprint, validações, layouts, workflows, backfill de Situacao_Petronect, **desativar geradores redundantes de tarefa**
- **Onda 3** — aprovação, funções Deluge, escrita no Log, permissões WorkDrive
- **Onda 4** — virada das rotinas Claude, desligar a rotina agendada atual, 5 dias de observação
- **Onda 5** — relatórios, treinamento, triagem das 51 linhas pendentes, Produtos, revisão após 30 dias

## Decisões pendentes do usuário (bloqueiam a Onda 1)

| # | Decisão | Sugestão dada |
|---|---|---|
| DP-01 | A Petronect oferece API para fornecedores? | Confirmar formalmente; começar com download humano do edital |
| DP-02 | Faixas de alçada e margem mínima | R$ 250 mil e R$ 1 mi como ponto de partida |
| DP-03 | Prazo padrão de retorno do parceiro | 5 dias úteis, ajustável por parceiro |
| DP-04 | Quem é o responsável técnico das automações | — |
| DP-05 | Manter a rotina atual durante a virada? | Não — corte único, paralelo gera duplicidade |

## Pontos de atenção para sessões futuras

- **Stage aceita apenas os rótulos em português** (Prospeccao, Cancelado, Negociação/Revisão, Perda fechada, Perda fechada para a concorrência, Ganho fechado, Qualificação). `getFields` devolve os nomes em inglês, mas gravar com eles falha com `MAPPING_MISMATCH`.
- A rotina agendada `trig_01UgCozpX8hBhGxLWRAnBTXr` continua ativa e **precisa ser desligada na Onda 4**, não antes.
- Existem hoje 3 geradores da mesma tarefa de triagem; a v2 mantém só a regra de workflow nativa (WF-01).
