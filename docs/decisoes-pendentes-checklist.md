# Checklist de decisões pendentes — bloqueiam fechar o desenho e montar o Zoho Flow

Criado em 25/08/2026, a pedido do usuário, consolidando tudo que está espalhado em `criterios-deduplicacao-multi-canal.md`, `padrao-tags.md`, `guia-construcao-zoho-flow-consolidado.md`, `integracao-petronect-api-deluge.md` e `workflow-petronect-v2.md`. **Atualizado no mesmo dia conforme as respostas chegaram.**

## Resolvidas em 25/08/2026

1. ~~`vendas@`/`contato@` recebem só cliente recorrente, ou também prospecção fria?~~ **Fechado: os dois.**
2. ~~Destino do registro: Leads ou Negócios direto?~~ **Fechado: direto em Negócios**, status inicial "Pendente Análise". Convenção de nome: `<CLIENTE> <AAMMSSSS>` (ex.: `DELL 26080010`), campo novo `Codigo_Oportunidade_Direta` (texto único).
3. ~~Janela de dias da Camada 1 de dedup?~~ **Fechado: 45 dias.**
3b. ~~Mecânica do sequencial `SSSS`?~~ **Fechado: reinício anual** (não mensal) — `MM` no código é só informativo, não é fronteira de reinício.
4. ~~Zoho Connect/Streams substitui o WorkDrive+TeamInbox?~~ **Fechado: mantém WorkDrive + TeamInbox.**
5. ~~Cadência de aplicação da tag de parceiro?~~ **Fechado: na criação (tags fixas) e na definição de cada parceiro (tag do parceiro).**
6. ~~Sincronização manual fora do CRM volta automaticamente, ou fica de fora?~~ **Fechado: sincronização bidirecional entre CRM, Mail e WorkDrive.**
7. ~~E se o conector do WorkDrive não permitir aplicar rótulo?~~ **Fechado: declinar essa parte específica.**
8. ~~O que o botão faz?~~ **Fechado: ler → mover para a pasta da oportunidade → marcar como lido → abrir no CRM → abrir a pasta no WorkDrive.**
8b. ~~O botão também cria a oportunidade quando ela não existe?~~ **Fechado: sim, é o mesmo botão** — quando não existe, gera o nome/código primeiro e cria o Negócio, depois segue com o resto.
9. ~~Botão por usuário ou único?~~ **Fechado: um único botão, disponível a todos os usuários.**
14. ~~Renomear `Status_Petronect`?~~ **Fechado: novo rótulo "Status da Oportunidade"** (api_name continua `Status_Petronect`). **Ação pendente de execução** — ajuste de interface, não executável por API nesta sessão.

## Verificação técnica (não é decisão do usuário, resolve só ao montar)

7b. Confirmar dentro do Zoho Flow se o conector nativo do WorkDrive tem ação de aplicar rótulo à pasta (decide se a parte de rótulo no WorkDrive entra ou é declinada, conforme a regra já fechada no item 7).

## DP-01 — API do Petronect: ainda em aberto, com um alerta importante

**10.** O documento anexado em 25/08 ("PERGUNTA 10") **não é a documentação real da API do Petronect** — é a resposta de outro assistente de IA, que o próprio texto admite ter inventado uma tabela de campos genéricos por falta da informação real, pedindo para o usuário cruzar com o arquivo verdadeiro. Ver `integracao-petronect-api-deluge.md` para o achado completo e o esqueleto de função Deluge já preparado (com todos os `TODO` de nome de campo/endpoint reais). **DP-01 continua sem resposta:** falta a documentação real da API (portal de desenvolvedor, Swagger/OpenAPI, coleção Postman, manual com nomes de campo reais, ou credenciais/sandbox).

## Adiadas a pedido do usuário (25/08) — não tratar agora

11. **DP-02** — Faixas de alçada e margem mínima para aprovação.
12. **DP-03** — Prazo padrão de retorno do parceiro (RFQ).
13. **DP-06** — Tabela de homem-hora e política de reajuste.

Ficam para a etapa de propostas automáticas, mais adiante no projeto.

## Estado geral

Todas as decisões de negócio das frentes novas (Fluxos A–F, tags DP-08, botão) estão fechadas. Resta: a verificação técnica 7b (resolve só ao montar), a DP-01 (precisa da documentação real da API do Petronect, ainda não fornecida), e as itens 11–13, explicitamente adiadas para depois.
