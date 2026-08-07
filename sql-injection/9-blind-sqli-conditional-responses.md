## Lab: Blind SQL injection with conditional responses

**Vulnerabilidade:** Blind SQL Injection (via cookie, conditional responses)

**Payload/técnica:**
Injeção no cookie `TrackingId`, usando busca binária manual caractere por caractere:

```sql
TrackingId=<valor_real>' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'administrator'), N, 1) > 'X'--
```

Onde `N` é a posição do caractere testado (1, 2, 3...) e `X` o valor de comparação. Processo:
1. Confirmar TrackingId válido (do próprio cookie da sessão) no lado esquerdo do `AND`, pra não invalidar a condição inteira.
2. Para cada posição, usar `>`/`<` para ir dividindo o intervalo de caracteres possíveis ao meio (busca binária), até restar 1-2 candidatos, e então confirmar com `=`.
3. Repetir para cada posição até encontrar uma posição "vazia" (ex: `SUBSTRING(..., 21, 1) < '0'`), indicando o fim da senha.

Senha extraída: `i9e5h4w8yd86ksd6qycl` (20 caracteres)

**Raciocínio:**
Diferente dos labs de UNION attack, aqui a query não retorna dados visíveis — a aplicação só muda de comportamento ("Welcome back" aparece ou não) dependendo se a condição injetada é verdadeira ou falsa. Isso caracteriza SQL injection **cega** (blind).

Para extrair dados sem enxergá-los diretamente, usei o princípio de busca binária: em vez de testar cada letra possível uma por uma (força bruta linear), comparei o valor ASCII/lexicográfico do caractere com um ponto médio (`> 'm'`), e fui reduzindo o intervalo de possibilidades pela metade a cada resposta, até isolar o caractere exato.

Pontos de atenção que precisei corrigir no processo:
- O lado esquerdo do `AND` precisa ser uma condição **verdadeira** (usar o TrackingId real da própria sessão, não um valor inventado), já que `AND` exige ambos os lados verdadeiros — usar um valor falso ali invalidaria o teste inteiro, sempre retornando falso independente do resultado real.
- Esquecer de fechar a aspa simples antes do `--` (ex: `= 'h--` em vez de `= 'h'--`) quebra comparações de igualdade (`=`), porque o SQL interpreta o `--` como parte da string comparada. Isso não afetava tanto os testes com `>`/`<` (porque a comparação lexicográfica já decidia pelo primeiro caractere), mas quebrava totalmente o `=`.
- Para descobrir o tamanho da senha, usei o mesmo princípio: testar `SUBSTRING` em posições crescentes até uma retornar "vazio" (string vazia, menor que `'0'` em comparação lexicográfica), indicando que a senha já tinha terminado antes daquela posição.
