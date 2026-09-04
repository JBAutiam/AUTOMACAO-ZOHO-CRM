# Índice da automação + estado verificado do CRM

**Última verificação: 23/08/2026, 14h40 (decisões e execuções atualizadas em 25/08/2026).** Este é o documento de entrada do projeto — leia-o primeiro. Publicado também como página consultável: https://claude.ai/code/artifact/f9b0ca63-2958-47ac-b53b-7d6848d95d8c

Os números abaixo foram **conferidos direto na API do Zoho**, não copiados dos outros documentos.

## Decisões fechadas

| # | Data | Decisão | Conteúdo |
|---|---|---|---|
| **DP-04** | 23/08 | Responsável técnico da automação | **Jorge Bueno — jbueno@autiam.com** |
| **DP-05** | 23/08 | Motor da automação | **Zoho Flow.** Sempre automação dentro do Zoho, nunca rotina agendada externa. O Claude é serviço chamado por HTTP de dentro do Flow, só para leitura de linguagem natural. |
| **DP-07** | 25/08 | Deduplicação multi-canal, obrigatória em toda caixa | Toda automação de criação de oportunidade, em **qualquer** caixa ativa da Autiam (hoje `jbueno@`, `gmartinez@`, `vendas@`, `contato@`, e futuras) precisa passar pela trava de deduplicação antes de criar registro no CRM. Especificação completa em `claude/criterios-deduplicacao-multi-canal.md`. Ainda não construído. |
| **DP-08** | 25/08 | Padrão de tags, multi-parceiro, sincronizado | Oportunidade, e-mail e pasta do WorkDrive carregam as mesmas tags: `PETRONECT`, `PORTAL`, `OLEO&GAS` e **uma tag por parceiro fornecedor** envolvido (múltiplas permitidas). Especificação completa em `claude/padrao-tags.md`. **Taxonomia e backfill no CRM/Mail executados em 25/08** — ver `diario-implantacao.md`. |

Isso resolve os conflitos 1 e 2 listados nas versões anteriores deste índice: **motor único = Zoho Flow**, e **campo de procedência único = `Origem_Automa_o`** (a v2 foi corrigida para não criar `Origem_Registro`).

> Numeração: DP-06 (tabela de homem-hora) segue pendente — ver `workflow-petronect-v2.md`. O salto de DP-05 para DP-07 é proposital, para não reindexar decisões já referenciadas em outros documentos.

## Estado real do CRM

| Métrica | Valor |
|---|---|
| Total de registros no módulo Negócios | **245** |
| Nomes distintos | **245** — sem duplicatas |
| Oportunidades com `Lead_Source = PORTAIS ELETRONICOS` | **224** |
| Registros com a grafia antiga (`PORTAIS ELETRONICO` / `Portal Eletrônico`) | **0** |
| Status: Pendente Análise / Cotado / Declinado | 37 / 28 / 160 |
| Declinados com `Stage = Cancelado` | 160 de 160 |
| Cotados com `Data_Limite_Proposta` preenchida | 28 de 28 |
| Campos customizados em Negócios | 6 (1 obsoleto) |
| Fornecedores no módulo Vendors | **9** (não 8 — `XHVAL` descoberto em 25/08, não estava documentado) |

**Campos customizados em Deals:** `Data_Limite_Proposta`, `Status_Petronect`, `Fornecedor_Parceiro`, `Data_Limite_Petronect` (obsoleto), `Origem_Automa_o`, `Tarefa_Triagem_Criada`. Em Tasks: `Tipo_Tarefa_Petronect`. **Propostos, ainda não criados** (DP-07): `Chave_Dedup_Origem`, `Possivel_Duplicata`.

**Tags no CRM (Deals) — atualizado 25/08:** todas as 9 tags de parceiro existem (`NIRMAL`, `MICROSENSOR`, `TRUEDYNE`, `LIANGGU VALVE`, `JIWEI`, `HENAN QUANSHUN`, `ZHENXUAN`, `LESHAN`, `XHVAL`), mais `PETRONECT`, `Portal`, `Oleo&Gas`. Backfill aplicado aos 172 negócios com `Fornecedor_Parceiro` preenchido (101 NIRMAL, 68 MICROSENSOR, 1 cada TRUEDYNE/ZHENXUAN/XHVAL) — contagens conferidas via `getTags` depois da escrita. **Labels espelhados no Zoho Mail** (mesmos 9 nomes de parceiro + PETRONECT/Portal/Oleo&Gas). **Ainda falta:** aplicar os labels às mensagens já arquivadas (backfill retroativo, não feito) e replicar em rótulos do WorkDrive (sem conector disponível nesta sessão).

**Nada da especificação da v2 foi criado ainda** — nem Blueprint, nem os campos novos, nem os módulos novos, nem regras, nem funções Deluge. O Passo 1 (4 configurações de interface) foi **concluído em 25/08** (ver `diario-implantacao.md`).

## Incidente de 23/08/2026 e sua correção

A rotina agendada recriou **34 oportunidades existentes**, criou um registro chamado **`None`** e reescreveu `Lead_Source` de volta. A base foi de 126 para 259 registros.

| Ação | Resultado |
|---|---|
| Rotina desativada e neutralizada | `trig_01UgCozpX8hBhGxLWRAnBTXr` → `enabled = false`, renomeada `[DESATIVADA]`, prompt substituído por "não fazer nada" |
| 34 duplicados + `None` excluídos | 35 exclusões, com aprovação do usuário |
| `Lead_Source` regravado | 224 registros → `PORTAIS ELETRONICOS` |
| Declinados alinhados | 160 registros → `Stage = Cancelado` |
| Colisão de nome resolvida | `RETROFIT VRU COOL SORPTION` (2 contas distintas) renomeado com o nome da conta em cada um |

Esse incidente é a origem direta da DP-05 e da DP-07 — qualquer expansão da automação (mais caixas, mais fontes) reabre exatamente esse risco se a chave de dedup não for definida com o mesmo rigor antes de entrar em produção. A configuração de tags (DP-08) executada em 25/08 é de natureza diferente — taxonomia e backfill pontual, sem lógica de criação/dedup de registro — por isso foi executada diretamente nesta sessão sem contrariar a DP-05.

## Correção de grafia — decisão do usuário

O usuário determinou **`PORTAIS ELETRONICOS`** (plural, sem acento). Os 224 registros já estão com esse valor gravado, e a opção já foi renomeada na picklist (Passo 1, Configuração 1, concluída 25/08).

## Mapa dos documentos

| Documento | Onde | Situação |
|---|---|---|
| **Guia de Implantação — Passo 1** (4 configurações de interface + checklist) | .docx entregue 23/08 | **Concluído 25/08** |
| Workflow Petronect v2 — Plano de Implementação (31 pág) | `claude/workflow-petronect-v2.md` + .docx entregue 23/08 | **MESTRE** — corrigido: Zoho Flow como motor, sem `Origem_Registro`, DP-04 e DP-05 decididos |
| Critérios de deduplicação multi-canal (DP-07) | `claude/criterios-deduplicacao-multi-canal.md` | **Novo, 25/08** — desenho, ainda não construído |
| Padrão de tags (DP-08) | `claude/padrao-tags.md` | **Novo, 25/08** — taxonomia e backfill no CRM/Mail **executados**; WorkDrive e labels retroativos em e-mail ainda pendentes |
| Estado da automação Petronect | `claude/estado-automacao-petronect.md` | Atualizado 23/08 |
| Diário de implantação | `claude/diario-implantacao.md` | Atualizado 25/08 — registro cronológico do executado |
| POP de documentação e WorkDrive (Parte VII do v2) | `claude/pop-documentacao-workdrive.md` | Vigente |
| `00_LEIA_ME_Fluxo.pdf` + `gabarito_nomes.txt` | arquivos entregues | Prontos para uso |
| Análise Crítica do Workflow (28 achados) | `claude/analise-critica-workflow-petronect.md` + .docx | Referência |
| Passo a passo Zoho Flow (20/08) | `claude/zoho-flow-implementacao-passo-a-passo.md` | **Superado pela DP-05** — a parte que admite Flow e rotina agendada em paralelo não vale mais |
| WORKFLOW PETRONECT.docx (v1 do usuário) | enviado pelo usuário | Superado pela v2 |
| Triagem_Oportunidades_Petronect_3.xlsx | devolvida preenchida | 51 linhas ainda sem decisão |
| Fluxograma local | `fluxo-petronect-crm.mermaid` | Atualizado 25/08 — multi-caixa + camadas de dedup |
| Fluxograma no Zoho Vani | espaço "Automação EMAIL_CRM", equipe AUTIAM | **Ainda pendente** — precisa reetiquetar os nós `[CLAUDE]` como chamadas HTTP de dentro do Flow, e incorporar multi-caixa + dedup (DP-07), tags multi-parceiro sincronizadas (DP-08) e os botões manuais quando decididos |

## O que falta

**Dados ainda abertos:**

- `Origem_Automa_o` vazia na maior parte dos registros — backfill pendente
- 51 linhas de triagem sem decisão do usuário
- Campos da DP-07 (`Chave_Dedup_Origem`, `Possivel_Duplicata`) — a criar após aprovação da janela de dias e do destino (Leads vs. Negócios) para a Camada 2
- Labels retroativos em e-mails já arquivados (DP-08) — taxonomia pronta, backfill nas ~147 mensagens ainda não feito
- Rótulos no WorkDrive (DP-08) — sem conector disponível nesta sessão; precisa Zoho Flow ou interface manual
- Sincronização automática entre CRM/Mail/WorkDrive quando uma tag mudar (DP-08) — ainda sem desenho, ver `padrao-tags.md`

**Decisões ainda pendentes do usuário:** DP-01 API da Petronect · DP-02 alçadas · DP-03 prazo de retorno do parceiro · DP-06 tabela de homem-hora · destino Leads-vs-Negócios para `vendas@`/`contato@` (ver `criterios-deduplicacao-multi-canal.md`) · Zoho Connect/Streams vs. WorkDrive+TeamInbox · botões manuais no Mail/CRM · cadência de reaplicação da tag de parceiro (ver `padrao-tags.md`).

**Implantação da v2:** Passo 1 concluído; passos 2–7 (Blueprint e Fluxos A–E no Zoho Flow) não iniciados. Fluxos C e D (vendas@, contato@) ainda nem desenhados no Zoho Flow — dependem da DP-07 estar implementável (campos criados) antes de ligar o gatilho.

## Lições registradas

1. **A checagem de duplicidade não pode ser por nome exato.** Extrair os 10 dígitos de qualquer posição do `Deal_Name` e comparar por eles. Foi assim que a duplicata passou em 19/08.
2. **Nunca confiar em um snapshot de IDs no início de uma operação em lote.** Reconsultar por critério ao final (`where Lead_Source != '<valor novo>'`), porque registros novos aparecem durante o trabalho.
3. **Automação sem trava no banco é questão de tempo.** O incidente de 23/08 só foi possível porque `Deal_Name` aceita duplicata. A regra de negócio tem que morar no CRM, não só no código que escreve nele.
4. **A API do Zoho CRM não altera campos existentes.** Existe `createFields`, não existe `updateFields`. Toda mudança de picklist, unicidade, validação ou workflow é interface.
5. **Deduplicação sem número de referência (caixas fora da Petronect) não pode decidir sozinha.** Quando a chave é fraca (mesma empresa, remetente novo), a automação cria e sinaliza para revisão humana — nunca funde/descarta automaticamente. Mesma lição do incidente de 23/08, aplicada ao caso sem número de portal.
6. **Cadastro de referência (Vendors) pode estar desatualizado na documentação sem ninguém perceber.** O projeto listava 8 parceiros havia dias; a consulta direta ao módulo Vendors em 25/08 encontrou um 9º (`XHVAL`), já em uso em pelo menos um negócio. Antes de qualquer backfill em lote, conferir a lista de referência direto na fonte, não copiar de um documento anterior.
