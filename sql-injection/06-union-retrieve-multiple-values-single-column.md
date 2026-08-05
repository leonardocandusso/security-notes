# SQL injection UNION attack, retrieving multiple values in a single column

**Categoria:** Retrieving multiple values within a single column
**Dificuldade:** Apprentice
**Link:** https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column

## Objetivo
Cenário parecido com o anterior, mas dessa vez a query original só retorna **1 coluna** no total (não 2) — o que exige extrair username e password concatenados nessa única coluna disponível.

## Reconhecimento
- Repeti o processo de descoberta de colunas (`ORDER BY` incremental) e, dessa vez, deu erro já no `ORDER BY 2` → só **1 coluna**.

## Exploração
Com apenas 1 coluna, não tem escolha: username e password precisam ser concatenados na mesma posição, com um separador pra conseguir distinguir os dois valores depois na resposta.

## Payload final
```
category=Gifts' UNION SELECT username || '~' || password FROM users--
```

## Por que funcionou
Mesmo princípio de UNION attack, mas adaptado à restrição de 1 coluna só. A concatenação com um caractere delimitador (`~`, que dificilmente aparece nos dados reais) permite separar visualmente os dois campos na resposta renderizada, mesmo estando tecnicamente numa única coluna.

## Mitigação
- Prepared statements.
- Também vale reforçar: quanto mais colunas/menos colunas a query original expõe, mais o atacante precisa se adaptar — mas isso é só um obstáculo a mais, não uma defesa real. A correção estrutural é sempre a mesma.
