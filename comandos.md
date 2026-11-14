# Comandos executados no projeto

Registro dos comandos rodados ao longo do projeto de replicação PostgreSQL (primário + réplica), com seus respectivos comentários.

## Sumário

1. [Criação do banco de dados](#1-criação-do-banco-de-dados)
2. [Testes do banco primário](#2-testes-do-banco-primário)
3. [Testes da réplica](#3-testes-da-réplica)
4. [Inserção de dados no banco primário](#4-inserção-de-dados-no-banco-primário)
5. [Teste da inserção na réplica](#5-teste-da-inserção-na-réplica)
6. [Criação da role `app`](#6-criação-da-role-app)
7. [Criação das conexões como `app`](#7-criação-das-conexões-como-app)
8. [Verificação das conexões ativas](#8-verificação-das-conexões-ativas)
9. [Confirmação da quarta conexão](#9-confirmação-da-quarta-conexão)
10. [Modificação da role com timeouts](#10-modificação-da-role-com-timeouts)

---

## 1. Criação do banco de dados

Criação do banco de dados usando o arquivo `docker compose`:

```bash
docker compose up -d
```

---

## 2. Testes do banco primário

Criação de uma tabela de teste e inserção de um registro:

```bash
docker compose exec primary psql -U postgres -c "CREATE TABLE teste (id serial PRIMARY KEY, msg text); INSERT INTO teste (msg) VALUES ('ola');"
```

Verificação se o servidor está em modo de recuperação (no primário, o esperado é `false`):

```sql
SELECT pg_is_in_recovery();
```

---

## 3. Testes da réplica

Verificação do modo de recuperação e leitura da tabela criada no primário:

```bash
docker compose exec replica psql -U postgres -c "SELECT pg_is_in_recovery();" -c "SELECT * FROM teste;"
```

Verificação do status da replicação, executada no primário:

```bash
docker compose exec primary psql -U postgres -c "SELECT client_addr, state, sync_state FROM pg_stat_replication;"
```

Verificação do modo de recuperação (na réplica, o esperado é `true`):

```sql
SELECT pg_is_in_recovery();
```

---

## 4. Inserção de dados no banco primário

Criação da tabela `clientes` e inserção de 1.000 registros:

```sql
CREATE TABLE clientes (id serial PRIMARY KEY, nome text, email text);

INSERT INTO clientes (nome, email)
SELECT 'cliente ' || g, 'c' || g || '@loja.com'
FROM generate_series(1, 1000) g;
```

Criação da tabela `eventos`:

```sql
CREATE TABLE eventos (id bigserial PRIMARY KEY, ts timestamptz DEFAULT now());
```

---

## 5. Teste da inserção na réplica

Contagem dos registros de `clientes` na réplica:

```sql
SELECT count(*) FROM clientes;
```

---

## 6. Criação da role `app`

Criação da role `app` com limite de 3 conexões e sem timeout:

```sql
CREATE ROLE app LOGIN PASSWORD 'app' CONNECTION LIMIT 3;
```

---

## 7. Criação das conexões como `app`

Abertura de conexões com a role `app`, mantidas ocupadas por 300 segundos:

```bash
docker compose exec primary psql -U app -d postgres -c "SELECT pg_sleep(300);"
```

---

## 8. Verificação das conexões ativas

Listagem das conexões ativas da role `app` e o tempo de execução de cada uma:

```sql
SELECT pid, usename, state, now() - query_start AS tempo
FROM pg_stat_activity
WHERE usename = 'app';
```

---

## 9. Confirmação da quarta conexão

Encerramento das conexões da role `app`:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE usename = 'app';
```

---

## 10. Modificação da role com timeouts

Alteração da role `app`, agora com timeouts:

```sql
ALTER ROLE app SET statement_timeout = '3s';
ALTER ROLE app SET idle_session_timeout = '5s';
```
