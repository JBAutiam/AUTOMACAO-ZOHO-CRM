# AUTOMACAO-ZOHO-CRM

Documentação da automação que lê oportunidades do portal **Petronect** e as processa no **Zoho CRM** — triagem de e-mail, decisão Cotar/Declinar, handoff de proposta e coleta do resultado do certame.

**Responsável técnico:** Jorge Bueno — jbueno@autiam.com (DP-04)
**Motor da automação:** Zoho Flow. O Claude é serviço chamado por HTTP de dentro do Flow, apenas para o interpretativo (DP-05)

---

## Por onde começar

| Se você quer… | Leia |
|---|---|
| Entender o estado do projeto | [`docs/indice-e-estado-verificado.md`](docs/indice-e-estado-verificado.md) |
| O desenho-alvo completo | [`docs/workflow-petronect-v2.md`](docs/workflow-petronect-v2.md) |
| O que está em produção hoje | [`docs/estado-automacao-petronect.md`](docs/estado-automacao-petronect.md) |
| Coletar o resultado de um certame cotado | [`docs/fluxo-pos-proposta-petronect.md`](docs/fluxo-pos-proposta-petronect.md) |
| Saber o que o CRM tem de fato | [`estado-crm/snapshot.md`](estado-crm/snapshot.md) |

---

## Estrutura

```
docs/          Documentação do processo e dos planos de implantação
estado-crm/    Snapshot dos módulos e campos, conferido direto na API
```

### `docs/`

| Arquivo | O que é |
|---|---|
| `indice-e-estado-verificado.md` | **Porta de entrada.** Decisões fechadas, estado do CRM, mapa dos documentos |
| `workflow-petronect-v2.md` | Desenho-alvo oficial: 14 estados, 17 transições, modelo de dados, plano em 6 ondas |
| `fluxo-pos-proposta-petronect.md` | Coleta do mapa de resultado: portal → WorkDrive → CRM → tarefa de recurso |
| `estado-automacao-petronect.md` | O que roda hoje em produção |
| `analise-critica-workflow-petronect.md` | 28 achados sobre a v1 |
| `criterios-deduplicacao-multi-canal.md` | DP-07 — trava de duplicidade em toda caixa |
| `padrao-tags.md` | DP-08 — taxonomia sincronizada CRM / Mail / WorkDrive |
| `pop-documentacao-workdrive.md` | Padrão de pastas e nomes |
| `diario-implantacao.md` | Registro cronológico do que foi executado |
| `decisoes-pendentes-checklist.md` | Decisões que travam a construção |
| `guia-construcao-zoho-flow-consolidado.md` | Guia de montagem no Zoho Flow |
| `zoho-flow-implementacao-passo-a-passo.md` | Passo a passo de 20/08 — **superado pela DP-05** |
| `integracao-petronect-api-deluge.md` | Notas de integração com o portal |
| `carga-triagem-agosto-2026.md` | Carga de triagem de agosto |
| `fluxo-petronect-crm.mermaid` | Fluxograma multi-caixa com camadas de dedup |

### `estado-crm/`

Dump conferido na API em 03/09/2026. A ideia é que a defasagem entre documentação e realidade vire um `diff`, em vez de depender de alguém lembrar de atualizar o texto.

| Arquivo | Conteúdo |
|---|---|
| `snapshot.md` | Leitura humana: módulos, contagens, lacunas, armadilhas da API |
| `modulos.json` | Módulos padrão e customizados |
| `campos-deals.json` | Campos customizados de Negócios + campos de sistema relevantes |
| `campos-petronect-itens.json` | Módulo de itens do certame |
| `campos-petronect-anexos.json` | Módulo de anexos |
| `campos-petronect-logs.json` | Módulo de log |

---

## Estado em 03/09/2026

- Os três módulos `Petronect_*` existem desde 27/08 e **nunca executaram** — 0 registros em `Petronect_Itens`, e nenhum dos 33 negócios cotados tem `Ultima_Sync_Petronect`, `Vencedor_Final` ou `Valor_Homologado` preenchidos.
- Faltam três peças para o fluxo pós-proposta rodar: módulo `Petronect_Propostas` (preço por proponente por item), campos de recurso em Negócios, e lookup de `Petronect_Logs` para Negócios.
- Cinco decisões abertas travam a construção — ver a seção 5 de `docs/fluxo-pos-proposta-petronect.md`.

---

## Convenções

- **Nada de credencial neste repositório.** Sem token, sem senha de portal, sem chave de API. Se algum documento precisar referenciar um segredo, referencia o *nome* da variável, nunca o valor.
- **Documento defasado é pior que documento ausente.** Ao mudar o CRM, regere o snapshot em `estado-crm/` no mesmo commit.
- **Decisões são numeradas** (`DP-01`, `DP-02`, …) e nunca reindexadas — outros documentos as referenciam pelo número.
# AUTOMACAO-ZOHO-CRM
Autoamcoes no ZOho CRM para registro de contatos, cotas e oportunidades.
