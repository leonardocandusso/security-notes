# SQL injection UNION attack, determining the number of columns returned by the query

**Categoria:** SQL injection UNION attacks → Determining the number of columns required
**Dificuldade:** Apprentice
**Link:** https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns

## Objetivo
Primeiro passo de um ataque UNION: descobrir quantas colunas a query original retorna, requisito pra qualquer `UNION SELECT` funcionar (o número de colunas do `SELECT` injetado precisa bater exatamente com o da query original).

## Reconhecimento
- Mesmo filtro de categoria vulnerável dos labs anteriores.
- Duas técnicas possíveis: `ORDER BY` incremental ou `UNION SELECT NULL` incremental.

## Exploração
Usei a técnica de `ORDER BY`, incrementando o número até dar erro:
```
category=Gifts' ORDER BY 1--   → OK
category=Gifts' ORDER BY 2--   → OK
category=Gifts' ORDER BY 3--   → erro (coluna não existe)
```
Erro no `ORDER BY 3` indica que a query tem exatamente **2 colunas**.

## Payload final
```
category=Gifts' UNION SELECT NULL,NULL--
```
(retorna sem erro, confirmando as 2 colunas)

## Por que funcionou
`ORDER BY` referencia colunas pela posição numérica. Quando o número ultrapassa a quantidade real de colunas, o banco retorna erro — isso permite descobrir a contagem sem precisar de acesso ao schema. Uma vez confirmado, um `UNION SELECT` com o mesmo número de `NULL`s executa sem erro de sintaxe.

## Mitigação
- Mesma defesa raiz: prepared statements eliminam esse vetor por completo, já que o input nunca é interpretado como parte da estrutura da query.
- Mensagens de erro genéricas em produção (não expor detalhes de erro SQL ao usuário final) — dificulta esse tipo de enumeração, ainda que não resolva a causa raiz.
