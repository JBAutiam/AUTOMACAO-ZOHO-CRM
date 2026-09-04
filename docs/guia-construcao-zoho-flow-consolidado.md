# Guia mestre de construção — Zoho Flow (pronto para montar)

**Criado em 25/08/2026, atualizado no mesmo dia conforme as decisões fecharam.** Este documento existe porque o usuário pediu que toda a automação passe a rodar dentro do Zoho (Zoho Flow + Zoho Drive), sem o Claude executando tarefas de produção — CRM e Zoho Mail devem funcionar standalone. Ele consolida a ordem de construção e aponta para o desenho detalhado de cada peça, que já está espalhado em outros documentos do projeto. **Não substitui esses documentos — é o roteiro de montagem que amarra todos.**

**Por que este documento existe e não a construção em si:** esta sessão (Cowork rodando na nuvem, sem o app desktop conectado) não tem nenhuma ferramenta de API para o Zoho Flow, nem para o Zoho WorkDrive. As flows anteriores (o esqueleto do Fluxo A, as 4 configurações do Passo 1) foram montadas por um Claude conectado ao computador do usuário, clicando na interface do Zoho via extensão Claude for Chrome — essa conexão não existe nesta sessão agendada. Ver seção final "Como isto é efetivamente construído".

---

## Ordem de construção recomendada

### 0. Pré-requisito — conexões (usuário reportou concluído em 25/08, não verificado nesta sessão)

Zoho Flow → Configurações → Conexões:
1. Zoho Mail — `jbueno@autiam.com`
2. Zoho Mail — `gmartinez@autiam.com`
3. Zoho CRM — org Autiam
4. Zoho WorkDrive

**Antes de montar qualquer fluxo, confirmar na interface que as 4 conexões aparecem como ativas.** Sem isso os gatilhos nem aparecem como opção.

### 1. Fluxo A — Nova oportunidade Petronect (jbueno@ e gmartinez@)

Desenho completo, passo a passo, em `claude/zoho-flow-implementacao-passo-a-passo.md`, seções 1 a 4 e 7–8. Continua válido — a única parte superada pela DP-05 é a ideia de rodar em paralelo com uma rotina agendada do Cowork; a lógica do fluxo em si (gatilho, filtro de remetente, classificação por assunto, checagem de duplicidade por `Numero_Petronect`, criação do Negócio, criação da tarefa de triagem, arquivamento do e-mail) é o que deve ser montado.

**Acrescentar a esse fluxo, no passo de criação/atualização do Negócio (item 4.3 do documento referenciado):** aplicar as tags fixas `PETRONECT`, `Portal` e `Oleo&Gas` na criação (ação nativa "Add Tag" do conector CRM, se existir no Zoho Flow, ou Update Record no campo Tags) — ver `claude/padrao-tags.md`.

**Atenção ao detalhe crítico já documentado:** a ação de CRM dentro do fluxo precisa ativar `apply_feature_execution` para a regra de validação de Data Limite Proposta rodar — sem isso o Flow grava `Cotado` sem prazo.

### 2. Fluxo B — Mensagem de Sala (SALA \<n\>)

Desenho completo em `claude/zoho-flow-implementacao-passo-a-passo.md`, seção 5, e em `claude/estado-automacao-petronect.md` ("Fluxo de sala / relatório"). Cobre os três casos: oportunidade Cotada (tarefa urgente 1 dia útil), Pendente Análise (antecipa a tarefa de triagem) e Declinada (arquiva sem tarefa).

### 3. Fluxo C — Relatório de Divulgação

Desenho completo na mesma seção 6 do documento de passo a passo. Cotada → resumir e notificar (única etapa que ainda chama o Claude via HTTP, para gerar o resumo). Declinada → arquivar, sem notificar.

### 4. Fluxo D — Pasta no WorkDrive + tags/rótulos (DP-08) — desenho fechado

- **Criação de pasta por oportunidade:** ação nativa "Create Folder" do conector WorkDrive, gatilha junto com a criação do Negócio no Fluxo A/E/F.
- **Aplicar tags — dois gatilhos** (ver `claude/padrao-tags.md`): (1) na criação da oportunidade, tags fixas; (2) toda vez que `Fornecedor_Parceiro` for definido/atualizado, a tag daquele parceiro.
- **Sincronização bidirecional confirmada:** mudança manual de tag/rótulo em qualquer uma das três plataformas (CRM, Mail, WorkDrive) precisa se propagar para as outras duas. **Cuidado técnico ao montar:** usar checagem de idempotência ou campo de controle para não entrar em loop (mudança em A aciona B, que aciona A de novo).
- **Regra fechada para o WorkDrive:** se o conector nativo do WorkDrive dentro do Zoho Flow não tiver ação de aplicar rótulo à pasta, **declinar essa parte específica — não vale a pena montar um contorno.** O resto do Fluxo D (criação de pasta, tags no CRM/Mail) segue normalmente independente disso. Confirmar isso é o primeiro passo prático ao montar este fluxo.
- **Taxonomia já pronta no CRM e no Mail** (feito em 25/08 nesta conversa): as 9 tags de parceiro (`NIRMAL`, `MICROSENSOR`, `TRUEDYNE`, `LIANGGU VALVE`, `JIWEI`, `HENAN QUANSHUN`, `ZHENXUAN`, `LESHAN`, `XHVAL`) mais `PETRONECT`, `Portal`, `Oleo&Gas` existem nos dois sistemas, e o backfill nos 172 negócios existentes já foi aplicado — falta só a lógica de aplicação/sincronização automática dentro do Flow, e replicar a taxonomia no WorkDrive (se suportado).

### 5. Fluxos E/F — vendas@ e contato@ (desenho praticamente fechado)

Destino, nomenclatura, status inicial, janela de dedup e o comportamento do botão na Camada 2 já fechados em `claude/criterios-deduplicacao-multi-canal.md`. Falta, antes de montar:
- Mecânica do sequencial `SSSS` do código `<CLIENTE> <AAMMSSSS>` (reinicia por mês ou é contagem corrida?) — **único ponto de decisão do usuário que ainda falta nesta frente.**
- Criar os campos novos no CRM (`Codigo_Oportunidade_Direta`, `Chave_Dedup_Origem`, `Possivel_Duplicata`) — via API, uma vez a mecânica acima decidida.

A lógica das 3 camadas de dedup (remetente+negócio aberto/45 dias / mesma empresa com sinalização+botão de criação / número em texto livre) está especificada e pronta para montar assim que o ponto acima fechar.

### 6. WorkDrive + TeamInbox — confirmado, não muda

Usuário confirmou em 25/08 que mantém o desenho já aprovado em 21/08 (WorkDrive + TeamInbox), sem trocar por Zoho Connect/Streams. Nenhuma ação adicional aqui.

### 7. Botão manual, no Zoho Mail e no Zoho CRM — desenho fechado em 25/08

**O que o botão faz, ao ser clicado num e-mail:**

- **Se a oportunidade correspondente já existe** (Fluxo automático já criou, ou foi criada antes por este mesmo botão): (1) lê o e-mail; (2) transfere o e-mail para a pasta (no Zoho Mail) com o nome da oportunidade; (3) marca o e-mail como lido; (4) abre a oportunidade correspondente no Zoho CRM; (5) abre a pasta com o nome da oportunidade no Zoho WorkDrive.
- **Se a oportunidade não existe ainda** (caso da Camada 2 do DP-07 — mesma empresa, remetente novo, sinalizado para revisão, e o responsável confirma que é de fato nova): o botão **primeiro gera o nome/código** (`<CLIENTE> <AAMMSSSS>`, próximo sequencial, grava em `Codigo_Oportunidade_Direta`) **e cria o Negócio** (status inicial "Pendente Análise") — e só depois segue com os mesmos 5 passos acima (ler, mover, marcar como lido, abrir CRM, abrir WorkDrive). É o mesmo botão nos dois casos; a única diferença é esse passo extra de criação quando ainda não existe oportunidade.

**Disponibilidade:** um único botão, visível e utilizável por todos os usuários (não é por usuário individual).

**Como construir (interface, não API):** botão customizado é configuração de layout — mesma categoria das 4 configurações do Passo 1. Vai precisar de um guia de cliques dedicado (caminho de tela em Zoho CRM: Personalização → Botões; em Zoho Mail: verificar se o produto permite botão customizado por e-mail/thread, ou se a ação equivalente é um atalho/regra). Pode ser escrito agora que o comportamento está fechado — falta só decidir se entra na mesma rodada de montagem dos Fluxos E/F ou depois.

---

## Como isto é efetivamente construído

Esta sessão de Cowork está rodando na nuvem (foi disparada por uma rotina agendada) e **não tem acesso ao computador do usuário nem a uma ferramenta de API para Zoho Flow ou Zoho WorkDrive**. Os únicos sistemas Zoho alcançáveis diretamente daqui são CRM e Mail (por isso a tarefa de tags da DP-08 pôde ser executada nesta conversa, mas a montagem dos fluxos e do botão, não).

As duas formas de montar o que está descrito acima:

1. **Manual, pela interface**, seguindo este guia e os documentos referenciados — o mesmo processo já usado com sucesso no Passo 1 e no início do Fluxo A (montagem do esqueleto).
2. **Por um Claude conectado ao computador do usuário** (Cowork rodando "no seu computador", não na nuvem, com o app desktop aberto) — nesse modo existe a extensão Claude for Chrome, que consegue clicar na interface do Zoho Flow e do WorkDrive como fez antes. Essa é a via mais rápida para replicar o que já foi feito no Fluxo A.

Este guia fica pronto para qualquer um dos dois caminhos.
