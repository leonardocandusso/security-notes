# SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

**Categoria:** Retrieving hidden data
**Dificuldade:** Apprentice
**Link:** https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data

## Objetivo
Uma loja online filtra produtos por categoria via parâmetro na URL. O objetivo é manipular a query SQL por trás desse filtro para retornar produtos que normalmente ficam escondidos (ex: fora de catálogo/estoque).

## Reconhecimento
- A aplicação tem um filtro de categoria que reflete o valor escolhido na URL, algo como `/filter?category=Gifts`.
- Isso sugere que o valor de `category` é usado diretamente numa cláusula `WHERE` da query SQL no backend — candidato natural a SQL injection.

## Exploração
- Testei inserir uma aspa simples (`'`) no valor do parâmetro pra ver se quebrava a query — se aparecer erro de sintaxe SQL ou comportamento anômalo, confirma a vulnerabilidade.
- Com a injeção confirmada, o objetivo é fazer a condição do `WHERE` ser sempre verdadeira, trazendo todos os produtos (inclusive os escondidos).

## Payload final
```
category=Gifts'--
```

## Por que funcionou
O `--` comenta o restante da query original, removendo qualquer condição adicional que normalmente restringiria os resultados (como checar se o produto está "released" ou visível). Como resultado, a query passa a retornar todas as linhas da tabela de produtos, incluindo os itens ocultos.

## Mitigação
- Uso de **prepared statements / queries parametrizadas** em vez de concatenar input do usuário diretamente na query.
- Validação e allowlist de valores esperados para o parâmetro `category` (já que é um conjunto fixo e conhecido de categorias).
