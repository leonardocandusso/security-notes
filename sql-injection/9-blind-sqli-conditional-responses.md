# SQL injection attack, querying the database type and version on MySQL and Microsoft

**Categoria:** Examining the database
**Dificuldade:** Practitioner
**Link:** https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft

## Objetivo
Descobrir o tipo e a versão do banco de dados por trás da aplicação, usando um ataque UNION.

## Reconhecimento
- Reaproveitei a técnica de labs anteriores: confirmar número de colunas via `ORDER BY` incremental.
- `ORDER BY 2--` → OK (200)
- Confirmado: a query original tem 2 colunas.

## Exploração
Sabendo que é MySQL/Microsoft (título do lab já indicava), usei `@@VERSION` — função nativa que retorna a string de versão do banco.

Testei nas duas posições possíveis:

`category=Gifts' UNION SELECT @@VERSION,NULL-- -`
→ Retornou 200 OK, mas sem resolver o lab e sem o valor da versão aparecendo corretamente na página.

`category=Gifts' UNION SELECT NULL,@@VERSION-- -`
→ Retornou 200 OK e resolveu o lab (class='academyLabBanner is-solved' na resposta).

## Descoberta extra (comportamento de tipo no MySQL)
Diferente de outros bancos (Postgres, por exemplo), o MySQL é mais tolerante com incompatibilidade de tipo — em vez de estourar um erro de sintaxe/conversão quando uma string é colocada numa coluna numérica, ele pode aceitar silenciosamente (com truncamento/conversão implícita) sem gerar erro 500. Isso significa que "não deu erro" não é garantia de que a posição está certa — é preciso confirmar se o dado realmente apareceu certo na resposta, não só se a request "passou".

**Alerta de metodologia:** depois que o lab é resolvido uma vez, o estado fica salvo na sessão (cookie) — testes seguintes, mesmo com payload errado, continuam mostrando is-solved. Pra validar uma hipótese de forma limpa, é preciso testar em sessão nova (aba anônima / sessão sem cookie).

## Payload final (o que efetivamente resolveu)
`category=Gifts' UNION SELECT NULL,@@VERSION-- -`

## Por que funcionou
A 2ª coluna da query original é a que aceita/exibe texto corretamente na interface. @@VERSION retorna a string de versão do MySQL, que aparece refletida na resposta quando colocada na coluna certa.

## Mitigação
- Prepared statements / queries parametrizadas — elimina o vetor por completo.
- Não expor mensagens de erro de banco de dados em produção (aqui nem precisou de erro pra vazar informação, mas em outros cenários isso amplia muito a superfície de ataque).
- Desabilitar funções de sistema (@@VERSION, etc.) para o usuário de banco usado pela aplicação, quando possível — reduz a informação disponível mesmo se a injeção existir.
