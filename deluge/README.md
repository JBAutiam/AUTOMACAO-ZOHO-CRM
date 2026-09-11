# `deluge/` — código da automação de pastas e resultado

Criado em 11/09/2026 a partir do pedido de 4 etapas. Instalação passo a passo em [`SETUP.md`](SETUP.md).

## Por que este pacote existe

A reclamação de origem: *oportunidades decididas como Cotar não estão salvando automaticamente especificação técnica, inspeção etc.*

Diagnóstico conferido na API em 11/09/2026: **a pasta-raiz já está sendo criada** — 4 dos 5 negócios cotados mais recentes têm `URL_Pasta_WorkDrive` preenchida. O que nunca existiu é a **subestrutura**. Sem as 6 subpastas, não há onde o documento cair, e ele não cai.

Então a Etapa 1 não é construir do zero: é acrescentar as subpastas e garantir que a raiz exista quando falhou.

## Os arquivos

| Arquivo | Etapa | Estado |
|---|---|---|
| `FN01_estrutura_pastas.dg` | 1 — abrir a pasta e as 6 subpastas ao virar Cotado | **Pronto para instalar** |
| `FN02_arquivar_email.dg` | 4 (+ parte da 2) — rotear e-mail e anexos, criar tarefa imediata | **Pronto para instalar** |
| `FN03_resultado_licitacao.dg` | 3 — D+1, resultado e concorrentes | **Pronto, menos a fonte do portal** |
| `SETUP.md` | — | Conexões, campos, regras, teste |

## As 6 subpastas

Nomes fixos. As FN02 e FN03 gravam por esses nomes — mudar um quebra o roteamento.

```
<numero da oportunidade>/
├── 1 - DOC CLIENTE            downloads da consulta ao Petronect
├── 2 - DOC FORNECEDOR         propostas dos nossos fornecedores, todas as revisões
├── 3 - PROPOSTA               planilha de cálculo, proposta comercial e técnica
├── 4 - RESULTADO DA LICITACAO abertura do processo, proponentes e valores
├── 5 - CONCORRENTES           uma subpasta por concorrente, com proposta e preços
└── 6 - COMUNICADOS CLIENTE    sala de colaboração e comunicados que exigem ação
```

## O que roda e o que não roda

**Roda hoje**, assim que a conexão do WorkDrive existir: Etapa 1 inteira, Etapa 4 inteira, e todo o encanamento da Etapa 3 — pastas por concorrente, gravação de preço por item, cálculo do delta para o vencedor, tarefa de recurso.

**Não roda**: puxar documentos do My Petronect (Etapa 2, e o download da Etapa 3). Não é complexidade — é que **o portal não tem API confirmada para fornecedor**. É a DP-01, aberta desde 19/08/2026.

Isso está isolado numa função só, `FN91_coletar_do_portal`, com contrato de saída definido. Enquanto ela devolver `ok=false`, a FN03 registra WARN e **não grava nada** — em vez de gravar dado inventado.

> Não escrevi endpoint fictício ali de propósito. Já aconteceu neste projeto: o documento "PERGUNTA 10" trazia uma tabela de campos que a própria IA que a produziu admitiu ter inventado. Construir em cima daquilo grava dado errado em silêncio. Ver [`../docs/integracao-petronect-api-deluge.md`](../docs/integracao-petronect-api-deluge.md).

## Princípios respeitados

- **DP-05** — motor é o Zoho. Nada aqui depende de rotina agendada externa.
- **Idempotência** — rodar duas vezes não duplica pasta, arquivo nem proposta. A trava de `Chave_Proposta` é a mesma lição do incidente de 23/08.
- **Campo vazio em vez de campo errado** — CNPJ que não deu para ler fica vazio, nunca inferido da razão social.
- **Falha não fica em silêncio** — toda função grava em `Petronect_Logs` e marca `Sync_Erro` no negócio.
- **`Stage` não é alterado por automação** — encerrar negócio é decisão humana.
- **Prazo de recurso não é inventado** — sem o prazo lido do edital, a tarefa sai com `[PRAZO NAO CONFIRMADO]` e vencimento de 1 dia útil.
