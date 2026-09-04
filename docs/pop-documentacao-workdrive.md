# POP de documentação e WorkDrive — Parte VII do Workflow v2

Criado em 19/08/2026 a partir do "Manual de Estruturação de Oportunidades Comerciais" enviado pelo usuário. Incorporado como **Parte VII** do `Workflow_Petronect_v2_Plano_Implementacao.docx` (agora 30 páginas).

Entregues junto: **`00_LEIA_ME_Fluxo.pdf`** (guia visual de 1 página, construído) e **`gabarito_nomes.txt`** (esqueletos de nome prontos). Ambos vão na raiz do modelo de pastas do WorkDrive e são copiados para toda oportunidade nova pela função FN-08.

**Escopo:** o POP é da área comercial inteira, não só Petronect. Nas oportunidades Petronect o `[COD_OPORTUNIDADE]` é o número de 10 dígitos do portal — o mesmo que nomeia o registro no CRM e a pasta no Zoho Mail.

## 1. Orçamentos por tipo de fornecimento (novo)

Pedido do usuário: ao decidir cotar, abrir orçamentos na oportunidade em função do tipo — instrumento, serviços, software ou soluções.

- Campo novo `Tipo_Fornecimento` em Negócios — **multisseleção**: Instrumento / Serviço / Software / Solução Integrada.
- Na transição **T3** (Aprovar para Cotação), a função **FN-07** abre **um Orçamento (Quotes) por tipo selecionado**, cada um já com a estrutura de custo do seu tipo. Quatro tipos marcados = quatro orçamentos, não um misturado.
- `Amount` do Negócio = soma dos orçamentos, calculada pelo sistema. Ninguém digita o total à mão.
- Orçamento de Solução Integrada não duplica o que já está nos outros: carrega só o que é próprio da integração.

Estrutura de custo por tipo (tabela completa no doc):

| Tipo | Composição | Risco a vigiar |
|---|---|---|
| Instrumento | produto origem + frete int'l + impostos + despachante/seguro | cotação vencida e variação cambial entre cotar e submeter |
| Serviço | HH por função × horas + deslocamento/diárias + encargos + mobilização | escopo de horas mal delimitado, reajuste em contrato longo |
| Software | licença (perpétua/subscrição) + implantação + treinamento + suporte anual | renovação e reajuste anual não previstos |
| Solução Integrada | soma dos anteriores + engenharia de integração + comissionamento + garantia | responsabilidade técnica pelo conjunto |

Campos novos no módulo Orçamentos: `Revisao` (inteiro) e `Motivo_Revisao` (picklist: Emissão inicial · Adendo do cliente · Correção interna · Alteração de escopo · Atualização de preço).

## 2. Arquitetura de pastas no WorkDrive

Criada automaticamente pela **FN-08** na transição T3, a partir de modelo de pasta do WorkDrive.

Raiz: `[COD_OPORTUNIDADE]_[CLIENTE]_[OBJETO_CURTO]`

| Pasta | Subpastas | Conteúdo |
|---|---|---|
| 01_DOC_CLIENTE | /Atual, /Historico | RFQ, edital, adendo, escopo, pedidos do cliente |
| 02_DOC_FORNECEDOR | **/\<PARCEIRO\>**/Atual, /Historico | catálogo, datasheet, certificado, cotação recebida |
| 03_PROPOSTA_TECNICA | /Atual, /Historico | memorial, arquitetura, matriz de atendimento |
| 04_PROPOSTA_COMERCIAL | /Atual, /Historico | precificação, orçamentos, minuta, condições |
| 05_CORRESPONDENCIAS_ATAS | /Atual, /Historico | atas, e-mails críticos, RFQ enviada, respostas, comprovantes |
| 06_POSVENDA_TRANSICAO | (única) | cronograma, PO final, passagem de bastão |

**Dois ajustes ao modelo original do usuário, com justificativa dada no doc:**
1. **02_DOC_FORNECEDOR ganhou nível por parceiro** — sem ele, catálogo de Nirmal e Microsensor se misturam e não há como dar acesso a um parceiro sem expor a cotação do outro (achado D6). A regra /Atual e /Historico continua valendo dentro da pasta de cada parceiro.
2. **Raiz segue o código da oportunidade** — é o número repetido em CRM, Mail e WorkDrive que permite reconstruir a história depois.

## 3. Nomenclatura e revisão

Padrão: `[COD_OPORTUNIDADE]_[CLIENTE]_[TIPO_DOC]_REV[XX].[ext]`

- Underline entre campos, sem espaço, sem acento, sem caractere especial.
- **REV com dois dígitos: REV00, REV01** — nunca REV0, rev1 ou REV-A. **Isto corrigiu a v2**, que usava REV A/REV B; `Sufixo_Revisao` agora é REV + 2 dígitos.
- Emissão inicial sempre REV00, direto em /Atual.
- Alteração: move o vigente para /Historico **antes** de o novo entrar (função **FN-11** faz automaticamente quando o documento sobe pelo CRM).
- /Atual contém **um único arquivo de cada tipo** — o vigente.
- Arquivo em /Historico nunca é editado nem apagado.
- Nada é enviado a cliente ou parceiro se não estiver em /Atual.

Tabela completa de códigos de tipo de documento no doc (RFQ_CLIENTE, EDITAL, ADENDO, RFQ_FORN, COTACAO, DATASHEET, CERT_FAB, PROP_TECNICA, MEMORIAL, MATRIZ_ATEND, PROP_COMERCIAL, PLAN_PRECO, ORC_*, ATA, EMAIL, COMPROV_SUBM, PO_FINAL).

### Resolução do achado F2 (versionamento ambíguo)

São **duas revisões distintas** que não podem se confundir:

| | Revisão do documento | Revisão da oportunidade |
|---|---|---|
| O que conta | cada documento tem seu contador, no nome do arquivo | quantos adendos do cliente a oportunidade sofreu |
| Onde vive | sufixo REVxx + campo `Revisao` no Orçamento | campo `Numero_Revisao` no Negócio |
| Quem incrementa | quem emite a nova versão | a transição T12, ao registrar adendo |
| Relação | adendo com impacto obriga reemitir os afetados, cada um subindo o próprio REV | não força reemissão de documento que o adendo não tocou |

`Motivo_Revisao` responde a pergunta que a auditoria de qualidade faz: por que este documento tem quatro revisões?

## 4. E-mail como evidência de auditoria de qualidade

Pedido explícito do usuário. Separa **o documento recebido** da **prova de que ele foi pedido**:

| O que | Onde vai |
|---|---|
| Anexo técnico recebido do parceiro | 02_DOC_FORNECEDOR/\<PARCEIRO\>/Atual |
| E-mail da RFQ enviada | 05_CORRESPONDENCIAS_ATAS/Atual |
| E-mail de resposta do parceiro | 05_CORRESPONDENCIAS_ATAS/Atual |
| Mensagem de sala da Petronect | 01_DOC_CLIENTE/Atual + cópia em 05 |
| Comprovante de submissão | 05_CORRESPONDENCIAS_ATAS/Atual |

**O arquivamento no Zoho Mail continua em paralelo e não é substituído** — públicos diferentes: quem procura correspondência vai ao Mail, quem audita qualidade vai ao WorkDrive e espera achar a evidência dentro da pasta da oportunidade, sem acesso à caixa de ninguém.

## 5. Conteúdo do 00_LEIA_ME_Fluxo.pdf (construído)

Não é fluxograma do processo — é **árvore de decisão** para responder em segundos "tenho um arquivo na mão, onde ele vai?". Cinco blocos:

1. **De onde veio o documento?** — 6 cartões (cliente / fornecedor / nós-técnico / nós-comercial / registro de conversa / oportunidade ganha) → pasta destino de cada um.
2. **Já existe em /Atual?** — Não → REV00. Sim → move o atual para /Historico, depois salva a REV seguinte. /Atual guarda sempre um único arquivo.
3. **Confira o nome** — 3 perguntas sim/não (começa com o código? underline sem acento? termina com REV e dois dígitos?) + o modelo geral.
4. **Faixa vermelha NUNCA FAÇA** — 6 proibições (termos ambíguos, área pessoal, apagar, editar histórico, enviar o que não está em /Atual, criar pasta fora do padrão).
5. **Em dúvida, não improvise** — pergunte; pasta fora do padrão quebra a trilha de auditoria da oportunidade inteira.

**Critério de aceite do guia:** entregar o PDF a alguém que nunca viu o processo, com três documentos diferentes, e pedir que arquive os três. Se errar algum, o guia é que precisa mudar — não a pessoa.

## 6. Novas funções Deluge (somam-se às FN-01 a FN-06)

- **FN-07** — abre orçamentos por tipo (transição T3)
- **FN-08** — cria a árvore no WorkDrive a partir do modelo, com os dois arquivos-guia (T3)
- **FN-09** — cria 02_DOC_FORNECEDOR/\<PARCEIRO\> com permissão isolada, ao criar Cotação de Parceiro
- **FN-10** — arquiva e-mail como evidência em 05 e anexos na pasta do parceiro
- **FN-11** — rotina de revisão: move o vigente para /Historico e grava o novo com o REV seguinte

## 7. Acréscimos ao plano de ondas

- **Onda 0** — nova **DP-06**: confirmar tabela de homem-hora e política de reajuste (sustentam os orçamentos de Serviço e Software)
- **Onda 1** — criar `Tipo_Fornecimento`; criar `Revisao` e `Motivo_Revisao` em Orçamentos
- **Onda 2** — configurar os modelos de orçamento por tipo
- **Onda 3** — criar o modelo de pasta no WorkDrive; escrever FN-07 a FN-11; publicar os dois arquivos-guia
- **Onda 4** — Rotina A passa a depositar anexo e e-mail nas pastas corretas com a nomenclatura
- **Onda 5** — treinar no POP; migrar as oportunidades **ativas** para a estrutura nova (encerradas ficam como estão)
