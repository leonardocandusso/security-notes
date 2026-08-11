## Lab: Visible error-based SQL injection

**Vulnerabilidade:** Error-based SQL injection (via cookie, verbose error messages - PostgreSQL)

**Payload/técnica:**
Injeção no cookie `TrackingId`, usando `CAST()` para forçar erro de conversão de tipo (texto → int), fazendo o banco vazar o valor real dentro da própria mensagem de erro.

Payload final:
```sql
x' AND CAST((SELECT password FROM users LIMIT 1) AS int)=1--
```

Processo:
1. Confirmar o DBMS em contexto blind: testando sintaxes específicas de cada banco (`@@version`, `version()`, `v$version`) e observando qual não gera erro de sintaxe. Confirmado PostgreSQL.
2. Testar `CAST('a' AS int)` para validar que a técnica de erro de conversão funciona e vaza o valor testado na mensagem.
3. Tentar `CAST((SELECT password FROM users WHERE username = 'administrator') AS int)` — funcionava logicamente, mas sempre dava erro de "unterminated string literal".
4. Descobrir, por eliminação, que o cookie `TrackingId` tem um limite de ~60 caracteres — tudo além disso é truncado pela aplicação, quebrando a sintaxe da query.
5. Reduzir o payload ao mínimo: trocar o TrackingId real por um caractere único (`x`), remover `WHERE username = 'administrator'` (muito longo) e usar `LIMIT 1` para garantir que a subquery retorne uma única linha (evitando erro de "more than one row").
6. Validar que a linha retornada por `LIMIT 1` (sem `ORDER BY`) era de fato a do `administrator`, testando `username` antes de confiar no `password`.

Senha extraída: `qdkhdqc7qt65oge0jvh4`

**Raciocínio:**
Esse foi o lab mais trabalhoso até agora, com múltiplas camadas de depuração:

- **`CAST` como técnica de vazamento:** ao tentar converter um dado incompatível (texto → int), o PostgreSQL inclui o próprio valor problemático na mensagem de erro (`invalid input syntax for type integer: "valor"`), transformando um cenário blind em um vazamento direto — sem precisar de busca binária caractere por caractere.
- **`AND` exige booleano dos dois lados:** o `CAST(...)` sozinho retorna um valor (int), não uma condição verdadeiro/falso, então precisei comparar o resultado com algo (`=1`) só para satisfazer a sintaxe do `AND` — mesmo sabendo que essa comparação nunca seria de fato alcançada, porque o `CAST` já falha antes.
- **Limite de tamanho do campo:** o sintoma (erro sempre "unterminated string", cortando sempre na mesma posição da query, independente do conteúdo) foi a pista para suspeitar de truncamento em vez de erro de sintaxe. Confirmei enviando payloads de teste com caracteres fáceis de contar, até identificar o ponto exato de corte.
- **Orçamento de caracteres:** com o limite apertado, precisei economizar em cada parte do payload — trocar o TrackingId real por um único caractere, evitar `WHERE username = 'administrator'` (muito longo) em favor de `LIMIT 1` (mais curto, mas sem garantia de qual linha vem primeiro sem `ORDER BY`).
- **Trade-off tamanho vs. confiabilidade:** `LIMIT 1` sem `ORDER BY` não garante que a linha retornada seja a do administrator (depende da ordem física dos dados no banco, não é uma regra lógica). Funcionou nesse caso, mas foi validado explicitamente (testando o `username` retornado) em vez de assumido às cegas — uma alternativa mais robusta seria `ORDER BY username LIMIT 1`, ao custo de mais caracteres.
