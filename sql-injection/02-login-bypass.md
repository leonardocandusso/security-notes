# SQL injection vulnerability allowing login bypass

**Categoria:** Subverting application logic
**Dificuldade:** Apprentice
**Link:** https://portswigger.net/web-security/sql-injection/lab-login-bypass

## Objetivo
Explorar SQL injection no formulário de login para autenticar como o usuário `administrator` sem saber a senha real.

## Reconhecimento
- Formulário de login padrão (campos `username` e `password`).
- A lógica de autenticação no backend provavelmente monta uma query do tipo:
  `SELECT * FROM users WHERE username = 'INPUT_USER' AND password = 'INPUT_PASS'`

## Exploração
- Testei uma aspa simples no campo de usuário pra confirmar a injeção (gerou erro interno / comportamento inesperado).
- A estratégia: fazer a query autenticar sem precisar da senha correta, comentando a checagem de senha.

## Payload final
```
Username: administrator'--
Password: (qualquer valor)
```

## Por que funcionou
O `--` comenta a parte `AND password = 'INPUT_PASS'` da query. A condição resultante fica só `WHERE username = 'administrator'`, que é verdadeira — logando como administrator independente da senha enviada.

## Mitigação
- Prepared statements (novamente, é a defesa raiz contra qualquer SQLi).
- Nunca construir queries de autenticação por concatenação de string.
- Rate limiting / lockout de tentativas de login como camada extra de defesa.
