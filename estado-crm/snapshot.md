# Snapshot do CRM — 03/09/2026

Tirado direto da API do Zoho CRM (`getModules`, `getFields`, COQL), não copiado de documento anterior. Serve de linha de base: um `diff` contra o próximo snapshot mostra o que mudou no CRM sem depender de ninguém lembrar de atualizar a documentação.

## Módulos

| Módulo | Tipo | Blueprint | Observação |
|---|---|---|---|
| `Deals` | padrão | sim | Casa dos negócios Petronect |
| `Tasks` | padrão | sim | `Tipo_Tarefa_Petronect` |
| `Vendors` | padrão | sim | 9 parceiros fornecedores |
| `Petronect_Itens` | customizado | sim | Criado 27/08. **0 registros** |
| `Petronect_Anexos` | customizado | sim | Criado 27/08 |
| `Petronect_Logs` | customizado | sim | Criado 27/08. Sem lookup para Deals |

## Negócios com `Status_Petronect = Cotado`

**33 registros** — a documentação do projeto (`docs/indice-e-estado-verificado.md`, de 23/08) diz 28. Divergência esperada: entraram novos desde então.

| Indicador | Valor |
|---|---|
| Com `Ultima_Sync_Petronect` preenchida | **0 de 33** |
| Com `Vencedor_Final` preenchido | **0 de 33** |
| Com `Valor_Homologado` preenchido | **0 de 33** |
| Com `Fase_Petronect` preenchida | **1 de 33** (7004644291 = `Em Cotacao`) |
| Com `URL_Pasta_WorkDrive` | **6 de 33** |
| Com `Qtd_Anexos` | **0 de 33** |

**Leitura:** a estrutura de sincronização com o portal existe desde 27/08 e nunca executou uma única vez. O fluxo pós-proposta (`docs/fluxo-pos-proposta-petronect.md`) preenche exatamente esses campos.

## Lacunas registradas

1. **Sem módulo para proposta de concorrente.** `Petronect_Itens.Preco_Autiam` guarda só o nosso preço. Preço por proponente por item é N×M e não cabe. Proposta: módulo `Petronect_Propostas`.
2. **Sem campos de recurso** em Negócios (`Prazo_Recurso_Ate`, `Decisao_Recurso`, `Justificativa_Recurso`, `Delta_Para_Vencedor_Pct`, `Motivo_Perda`).
3. **`Petronect_Logs` não aponta para o negócio** — falha de coleta não aparece no registro.

## Como regerar

Os JSONs desta pasta vêm de:

```
getModules(status=visible)
getFields(module=Deals)
getFields(module=Petronect_Itens)
getFields(module=Petronect_Anexos)
getFields(module=Petronect_Logs)
COQL: select ... from Deals where Status_Petronect = 'Cotado' limit 200
```

## Armadilhas confirmadas da API (mantidas aqui para não redescobrir)

- **Não existe `updateFields`.** A API cria campo, não altera. Picklist, unicidade, validação e workflow são interface.
- **`Stage` só aceita os rótulos em português** (`Prospeccao`, `Cancelado`, `Negociação/Revisão`, `Ganho fechado`, `Perda fechada`, `Perda fechada para a concorrência`, `Qualificação`). `getFields` devolve os nomes em inglês, mas gravar com eles falha com `MAPPING_MISMATCH`.
- **COQL exige `where`** em toda consulta — `select ... from X limit 5` sem cláusula devolve `SYNTAX_ERROR: missing clause`. Use `where id is not null`.
- **COQL `count(id)`** só funciona acompanhado de `group by` com a mesma coluna no `select`. Para contagem simples, traga os ids e conte as linhas.
- **`getModules`** aceita `status` em `visible` / `user_hidden` / `system_hidden` / `scheduled_for_deletion` — não `active`.
