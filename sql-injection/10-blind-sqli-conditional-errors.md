## Lab: Blind SQL injection with conditional errors

**Vulnerabilidade:** Blind SQL Injection (via cookie, conditional errors - Oracle)

**Payload/técnica:**
Injeção no cookie `TrackingId`, usando `CASE WHEN` para forçar um erro de banco (divisão por zero) apenas quando a condição testada é verdadeira. Payload final por posição/caractere:

```sql
' AND (SELECT CASE WHEN (SUBSTR(password, N, 1) = 'X') THEN TO_CHAR(1/0) ELSE 'a' END FROM users WHERE username = 'administrator')='a'--
```

Onde `N` é a posição do caractere e `X` o candidato testado. Automatizado com **Burp Intruder** (modo Sniper, payload type "Brute forcer", character set `0123456789abcdefghijklmnopqrstuvwxyz`), testando os 36 candidatos de uma posição por vez e identificando qual gerou HTTP 500 (erro) em vez de 200 (normal).

Senha extraída: `ll1v0wep5reel7jo95n9` (20 caracteres)

**Raciocínio:**
Diferente do lab anterior (conditional responses), aqui a aplicação não muda de conteúdo dependendo do resultado da query — ela sempre responde igual, então não existe um sinal "natural" (tipo Welcome back) pra observar. A solução foi criar um sinal artificial: usar `CASE WHEN condição THEN 1/0 ELSE 'a' END`, forçando uma divisão por zero (erro garantido) apenas quando a condição interna é verdadeira. Erro 500 = verdadeiro; resposta normal = falso.

Vários bugs estruturais precisaram ser depurados, isolando uma variável de cada vez:
1. **Nome da tabela/coluna com case sensitivity de valores:** o enunciado citava `administrator` em minúsculo, mas eu testei `'Administrator'` — a comparação de string é case-sensitive, então o filtro nunca encontrava a linha, e qualquer operação sobre a subquery vazia gerava erro (mascarando completamente o teste real).
2. **Filtro dentro vs. fora do CASE:** colocar `username = 'administrator' AND ...` dentro do `WHEN` faz a subquery rodar o CASE para todas as linhas da tabela (retornando múltiplos valores em vez de um só). A correção foi mover o filtro para um `WHERE` fora do CASE, na cláusula `FROM users WHERE username = 'administrator'`, deixando dentro do `WHEN` só a condição que realmente varia.
3. **`SUBSTRING` vs `SUBSTR`:** esse banco é Oracle, que usa `SUBSTR` (não `SUBSTRING`).
4. **Inconsistência de tipos entre os ramos do CASE (bug mais sutil):** com `THEN 1/0 ELSE 'a'`, o Oracle detecta que um ramo é numérico (`1/0`) e tenta forçar o outro ramo (`'a'`) para o mesmo tipo, gerando erro de conversão (`ORA-01722: invalid number`) mesmo quando a condição era falsa. Isso fazia a query dar erro sempre, nos dois sentidos de comparação (`>` e `<`), mascarando totalmente o resultado real. A correção foi envolver o gatilho de erro em `TO_CHAR(1/0)`, garantindo que ambos os ramos "prometam" retornar texto — o cálculo de `1/0` ainda estoura erro quando executado, mas não há mais conflito de tipo entre os ramos.

Por fim, automatizei a extração usando o Burp Intruder em vez de busca binária manual (que usei no lab anterior). Como o Intruder testa todos os candidatos de uma posição automaticamente e sem custo manual extra, compensou mais usar comparação de igualdade (`=`) testando os 36 caracteres possíveis de uma vez, em vez de replicar a lógica de divisão binária (`>`/`<`), que só valia a pena quando cada tentativa exigia esforço manual.
