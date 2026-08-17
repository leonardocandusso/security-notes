## Lab: Blind SQL injection with time delays and information retrieval

**Vulnerabilidade:** Blind SQL Injection (via cookie, time-based - PostgreSQL)

**Payload/técnica:**
Injeção no cookie `TrackingId`, usando `pg_sleep()` como sinal de verdadeiro/falso, medindo o tempo de resposta em vez de conteúdo ou erro.

Payload de fingerprinting do DBMS (testado antes de qualquer extração):
```sql
' AND (SELECT 1 FROM pg_sleep(10))=1--
```

Payload final por posição/caractere:
```sql
' AND (SELECT CASE WHEN (SUBSTRING(password,1,1) > 'm') THEN (SELECT 1 FROM pg_sleep(10)) ELSE 1 END FROM users WHERE username='administrator')=1--
```

Automatizado com **Burp Intruder** (Sniper, payload type "Brute forcer", character set `0123456789abcdefghijklmnopqrstuvwxyz`), reduzindo o delay para ~3 segundos por tentativa para acelerar o processo, e identificando o candidato correto pelo tempo de resposta (>3s = verdadeiro).

Senha extraída: `z9w72kaax5zucjl243nm` (20 caracteres)

**Raciocínio:**
Esse é o terceiro "sabor" de blind SQLi (depois de conditional responses e conditional errors): usado quando a aplicação trata erros de forma tão graciosa que nem muda o conteúdo nem expõe qualquer erro — o único sinal que resta, inevitavelmente, é o **tempo de resposta**, já que a query é processada de forma síncrona.

Passos de raciocínio e depuração:
1. **Fingerprinting do DBMS em blind puro (sem erro nem conteúdo como sinal):** testei a sintaxe de delay de cada banco (`SLEEP()` MySQL, `WAITFOR DELAY` SQL Server, `pg_sleep()` PostgreSQL, `dbms_lock.sleep()` Oracle) isoladamente, sem `IF`/`CASE`, apenas observando qual delay realmente ocorria.
2. **Erro de tipo com `pg_sleep()` direto no `AND`:** `pg_sleep()` retorna tipo `void` (nenhum valor útil), o que quebra a sintaxe ao tentar usá-la diretamente como parte de uma condição booleana. Resolvido envolvendo a chamada numa subquery `(SELECT 1 FROM pg_sleep(N))`, aproveitando que no PostgreSQL certas funções podem ser usadas no lugar de uma tabela no `FROM` — a subquery então "converte" o efeito colateral do sleep num valor numérico simples (`1`), comparável e sem conflito de tipo.
3. **Estrutura combinada:** mesma lógica de `CASE WHEN condição THEN gatilho ELSE alternativa END` dos labs anteriores, trocando o gatilho de erro (`1/0`) pelo gatilho de tempo (`SELECT 1 FROM pg_sleep(N)`). O `ELSE` precisa devolver o mesmo tipo do `THEN` (número), para não reintroduzir o problema de tipos incompatíveis já visto no Oracle.
4. **Otimização de tempo total:** com o custo por tentativa agora sendo literalmente segundos de espera (não apenas latência de rede), reduzir o valor do delay (de 10s para ~3s) tem impacto direto e proporcional no tempo total de extração — importante ao multiplicar por até 36 tentativas × 20 posições.
