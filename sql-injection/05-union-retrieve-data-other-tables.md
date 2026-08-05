# SQL injection UNION attack, retrieving data from other tables

**Categoria:** Using a SQL injection UNION attack to retrieve interesting data
**Dificuldade:** Apprentice
**Link:** https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables

## Objetivo
Combinar as técnicas dos labs anteriores (número de colunas + coluna com texto) para extrair dados de uma tabela completamente diferente da usada na query original — nesse caso, a tabela `users`, com colunas `username` e `password`.

## Reconhecimento
- Já sei: 2 colunas, sendo a 2ª a que aceita texto.
- O enunciado do lab informa diretamente o nome da tabela (`users`) e das colunas (`username`, `password`) — em um cenário real, essa informação viria de enumeração do schema (`information_schema.tables` / `information_schema.columns`).

## Exploração
Como só uma coluna aceita texto, mas preciso trazer dois valores (usuário e senha), concatenei os dois numa única coluna usando `||` (padrão de concatenação em bancos como PostgreSQL/Oracle).

## Payload final
```
category=Gifts' UNION SELECT NULL,username || ':' || password FROM users--
```

## Por que funcionou
O `UNION SELECT` junta os resultados da minha query arbitrária aos resultados da query original, desde que o número de colunas e os tipos sejam compatíveis. Ao consultar `users` em vez da tabela de produtos, "sequestro" a funcionalidade legítima do filtro pra vazar dados de uma tabela sensível completamente não relacionada. Concatenar username e password na mesma coluna contorna a limitação de só ter 1 coluna compatível com texto.

## Mitigação
- Prepared statements (raiz do problema).
- Separação de privilégios: a conta de banco usada pela aplicação de e-commerce idealmente não deveria nem ter permissão de leitura na tabela `users` de autenticação, se estiverem em domínios de dados diferentes.
