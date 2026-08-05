# SQL injection UNION attack, finding a column containing text

**Categoria:** SQL injection UNION attacks → Finding columns with a useful data type
**Dificuldade:** Apprentice
**Link:** https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text

## Objetivo
Com o número de colunas já confirmado (2), descobrir qual delas aceita dado do tipo texto/string — necessário pra depois extrair dados como username/password nessa posição.

## Reconhecimento
- Continuação direta do lab anterior: já sei que a query tem 2 colunas.
- Nem toda coluna aceita string (pode ser INT, DATE etc.) — preciso testar cada posição substituindo `NULL` por um valor de texto.

## Exploração
Testei substituir cada `NULL` por uma string, uma posição de cada vez:
```
category=Gifts' UNION SELECT 'a',NULL--   → erro (1ª coluna não é texto)
category=Gifts' UNION SELECT NULL,'a'--   → OK (2ª coluna aceita texto)
```

## Payload final
```
category=Gifts' UNION SELECT NULL,'a'--
```

## Por que funcionou
Quando o tipo de dado da string não é compatível com o tipo da coluna original (ex: tentar colocar texto numa coluna INT), o banco gera erro de conversão de tipo. Testando posição por posição, identifico exatamente qual coluna aceita texto — essa é a que vou usar para exibir dados extraídos nos próximos labs.

## Mitigação
- Mesma raiz: prepared statements.
- Princípio de menor privilégio na conta de banco usada pela aplicação (não que resolva SQLi em si, mas limita o estrago em caso de exploração).
