# Padrão de tags — oportunidade, e-mails e pasta do WorkDrive

**Decisão DP-08 (25/08/2026), pedido do usuário nesta conversa, refinado na sequência.** Toda oportunidade criada, todo e-mail relacionado a ela e a pasta correspondente no WorkDrive devem carregar o mesmo conjunto de tags: **PETRONECT, PORTAL, OLEO&GAS**, e o **nome de cada parceiro fornecedor envolvido** (MICROSENSOR, NIRMAL, ou qualquer outro cadastrado no módulo Vendors). Taxonomia e backfill no CRM/Mail já executados em 25/08 (ver `diario-implantacao.md`) — a lógica automática de aplicação/sincronização ainda não foi construída no Zoho Flow.

## Múltiplos parceiros — confirmado

Quando mais de um parceiro for consultado na mesma oportunidade, **todas as tags de parceiro entram, uma por parceiro** — não só a última. Ex.: RFQ enviada para NIRMAL e MICROSENSOR na mesma oportunidade → a oportunidade carrega as duas tags, `NIRMAL` e `MICROSENSOR`, simultaneamente.

Importante distinguir de outro assunto já registrado no projeto: o campo `Fornecedor_Parceiro` no Negócio é um **lookup único** (só guarda um parceiro por vez) — isso é uma limitação diferente, do modelo de dados de cotação, já apontada no achado B4 da `analise-critica-workflow-petronect.md`, que recomenda um módulo `Cotações de Parceiro` para rastrear preço/prazo por parceiro. **A tag não depende dessa correção.** O campo nativo "Tags" do Zoho CRM já aceita múltiplos valores por registro — então dá para aplicar `NIRMAL` e `MICROSENSOR` como tags da oportunidade mesmo enquanto `Fornecedor_Parceiro` só guarda um lookup. As duas coisas evoluem em paralelo, sem uma depender da outra.

## Onde aplicar

1. **Zoho CRM — Negócio (Deal), campo Tags nativo** (multivalorado). Taxonomia completa já criada (9 tags de parceiro + PETRONECT/Portal/Oleo&Gas) e aplicada aos 172 negócios com `Fornecedor_Parceiro` conhecido (25/08).
2. **Zoho Mail — Labels nas mensagens da oportunidade**, mesmo conjunto de valores. Taxonomia completa já criada na caixa `jbueno@autiam.com` (25/08) — falta aplicar aos e-mails já arquivados (backfill retroativo) e ligar a aplicação automática no Zoho Flow.
3. **Zoho WorkDrive — pasta raiz da oportunidade** (`[COD_OPORTUNIDADE]_[CLIENTE]_[OBJETO_CURTO]`, ver `pop-documentacao-workdrive.md`). Usuário confirmou que o WorkDrive tem rótulos nativos — mas **decisão de 25/08 (item 7 abaixo): se o conector do WorkDrive dentro do Zoho Flow não tiver ação para aplicar rótulo à pasta, essa parte fica de fora — não vale a pena montar um contorno.** Verificar isso é o primeiro passo prático ao montar o Fluxo D.

## Quando aplicar a tag — confirmado em 25/08

**Dois momentos disparam a aplicação de tag:**
1. **Na criação da oportunidade** — as tags fixas (`PETRONECT`/`PORTAL` quando vem de portal, `OLEO&GAS`).
2. **Na definição do parceiro** — assim que `Fornecedor_Parceiro` for preenchido ou atualizado (cada vez que um parceiro novo entra na negociação, não só quando o vencedor for definido), aplica a tag daquele parceiro. Se mais de um parceiro for envolvido ao longo do processo, cada definição adiciona sua própria tag (acumulativo, nunca substitui).

## Regra de sincronização — confirmada e ampliada em 25/08: bidirecional entre as três plataformas

**Toda inclusão ou remoção de tag/rótulo, feita manualmente em qualquer uma das três plataformas (CRM, Zoho Mail, Zoho WorkDrive), deve se propagar automaticamente para as outras duas.** Não é mais "CRM como única fonte da verdade" — é sincronização nos dois sentidos, em qualquer direção que a mudança se origine.

**Nota técnica para quem for montar isso no Zoho Flow (risco de loop):** sincronização em três sentidos precisa de uma trava para não entrar em ciclo — CRM muda tag → aplica no Mail → gatilho "label mudou" no Mail dispara → tenta aplicar de volta no CRM → gatilho "tag mudou" no CRM dispara de novo, e por aí vai. Cada fluxo de sincronização precisa checar, antes de escrever, se o valor já está igual ao de destino (idempotência), e/ou usar um campo de controle (parecido com `Tarefa_Triagem_Criada`) para marcar "já sincronizado nesta rodada" e evitar disparo repetido. Detalhe de implementação a resolver ao montar, não uma decisão de negócio pendente.

## Origem do valor de cada tag

- **PETRONECT / PORTAL** — fixos hoje, porque a única fonte em produção é o portal eletrônico Petronect. Quando outro portal entrar (ver DP-07, `criterios-deduplicacao-multi-canal.md`), a tag do portal específico muda (ex.: `COMPRASNET`), mas `PORTAL` continua genérica para toda oportunidade vinda de portal eletrônico. Oportunidades de `vendas@`/`contato@` sem portal (cliente direto) não recebem `PETRONECT` nem `PORTAL` — só as tags que fizerem sentido (setor, parceiro).
- **OLEO&GAS** — fixo, setor de atuação da conta.
- **Nome do(s) parceiro(s)** — dinâmico, uma tag por parceiro envolvido na oportunidade, cadastrado no módulo Vendors (9 hoje: NIRMAL, MICROSENSOR, TRUEDYNE, LIANGGU VALVE, JIWEI, HENAN QUANSHUN, ZHENXUAN, LESHAN, XHVAL).

## Resolvido em 25/08/2026

- ~~Quando aplicar a tag de parceiro?~~ **Fechado: na criação da oportunidade (tags fixas) e na definição de cada parceiro (tag do parceiro), não só quando o vencedor for definido.**
- ~~Sincronização manual fora do CRM volta automaticamente, ou fica de fora?~~ **Fechado: sincronização bidirecional entre as três plataformas** (cuidado técnico de loop anotado acima).
- ~~E se o WorkDrive não suportar aplicar rótulo via Zoho Flow?~~ **Fechado: declinar essa parte específica, não perder tempo com contorno.** O resto (folder creation, tags no CRM/Mail) segue normalmente.

## Ainda em aberto

- Confirmar, ao montar o Fluxo D, se o conector do WorkDrive no Zoho Flow tem ação nativa de aplicar rótulo à pasta — decide se essa parte entra ou é declinada (regra já fechada acima).
