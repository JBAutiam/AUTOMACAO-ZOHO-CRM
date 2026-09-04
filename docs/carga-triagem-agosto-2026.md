# Carga da triagem Petronect no CRM — agosto/2026

Executada em 23/08/2026 a partir da planilha `Triagem_Oportunidades_Petronect_Final.xlsx` devolvida preenchida pelo usuário.

## O que a planilha trouxe

195 linhas → **192 oportunidades únicas** (a planilha voltou muito maior do que saiu: a equipe completou com o histórico). Três números apareciam duplicados na própria planilha e foram consolidados num registro só, com as observações concatenadas: `7004623953`, `7004624452`, `7004628644`.

Distribuição das decisões: **29 Cotar**, **161 Declinar**, 4 em branco, 1 "Encerrado".

## Decisões tomadas com o usuário

| Questão | Decisão |
|---|---|
| 133 oportunidades da planilha não existiam no CRM | Criar todas |
| Linha `7004628723` marcada como "Encerrado" (valor fora do picklist) | Gravar como **Declinado** |
| Fornecedor `ZHENXUAN` na `7004640078`, inexistente no CRM | É o mesmo que ZHENCHAO → registro renomeado para **ZHENXUAN** (id `6670502000003293003`) |
| 17 oportunidades no CRM ausentes da planilha | Não mexer — permanecem como estavam |

As 4 linhas sem decisão (`7004590806`, `7004595434`, `7004596772`, `7004617932`) ficaram como **Pendente Análise**.

## O que foi gravado

- **59 registros atualizados** (os que já existiam).
- **133 registros criados**.
- Campos gravados: `Status_Petronect`, `Data_Limite_Proposta`, `Fornecedor_Parceiro`, `Amount`, `Description` (objeto + observações + marca `[Triagem Petronect ago/2026]`), `Lead_Source` = Portal Eletrônico.
- Nos registros **novos**: `Stage` = **Cancelado** quando Declinado, **Prospeccao** quando Cotado. Nos registros que já existiam o Stage não foi alterado — há inconsistência a resolver se o usuário quiser padronizar.
- Tags **PETRONECT**, **Portal** e **Oleo&Gas** aplicadas a todos os registros da conta, em modo aditivo (`over_write: false`), preservando tags como NIRMAL.

## Estado final verificado

**224 oportunidades** na conta PETRONECT — mais que as 209 previstas. A diferença de 15 são registros criados por outro processo entre as sessões, com nomes descritivos (ex.: `7004644291 - TRANSMISSOR DE PRESS`, `7004643929 - REGULADOR DE PRESSÃO`), todos em Pendente Análise. Não foram tocados além das tags.

**28 oportunidades com Status_Petronect = Cotado**, todas com Data Limite Proposta preenchida — são as que devem gerar tarefa de proposta para o Guilherme quando a regra de workflow estiver configurada.

## Armadilhas encontradas (para a próxima carga)

1. **`postAddTags`**: o parâmetro `ids` é uma *string* com array JSON e **não aceita espaço após as vírgulas**. Com espaço a API responde `expected_data_type: bigint`. Gerar com `json.dumps(lista, separators=(',',':'))`.
2. **Não confiar em mapeamentos Deal_Name → id montados a partir da ordem da resposta de `createRecords`**. Numa das execuções o mapeamento saiu trocado e gerou ~35 ids inexistentes. Sempre reconsultar os ids reais via `searchRecords` antes de operações subsequentes.
3. `searchRecords` pagina em 200; conferir `info.more_records` e buscar a página seguinte.
