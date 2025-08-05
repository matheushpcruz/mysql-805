# Aula 06 – Análise e Manutenção

### 1. Importação de base de testes

```bash
# Instala o git
apt-get install git

# Clona bases de demonstração
git clone https://github.com/datacharmer/test_db.git

# Importa a base employees
mysql -uroot -p < test_db/employees.sql

# Importa a base sakila – schema e dados
cd sakila/
mysql -uroot -p < sakila-mv-schema.sql
mysql -uroot -p < sakila-mv-data.sql
```

### 2. Comandos SHOW e Inspeção de Objetos

```sql
-- Lista bancos de dados
SHOW DATABASES;

-- Mostra o comando de criação de um banco
SHOW CREATE DATABASE employees;

-- Lista procedures e mostra criação de procedure
SHOW PROCEDURE STATUS;
SHOW CREATE PROCEDURE sakila.rewards_report;

-- Lista funções e mostra criação de função
SHOW FUNCTION STATUS;
SHOW CREATE FUNCTION nome_funcao;

-- Lista triggers e mostra criação de um trigger específico
SHOW TRIGGERS FROM sakila\G;
SHOW CREATE TRIGGER sakila.payment_date;

-- Detalhe das tabelas e índices
SHOW TABLE STATUS FROM employees\G;
SHOW INDEXES FROM employees.departments;

-- Visualiza permissões e usuários
SHOW GRANTS FOR root@localhost\G;
SHOW GRANTS FOR aluno@'%'\G;
SHOW CREATE USER root@localhost;

-- Trabalha com views
SHOW CREATE VIEW sakila.staff_list;
SELECT * FROM information_schema.views;
```

### 3. Logs, Variáveis, Status e Diagnóstico

```sql
-- Lista arquivos de log binário, eventos e conexões
SHOW BINARY LOGS;
SHOW BINLOG EVENTS IN 'binlog.000033' LIMIT 50;
SHOW FULL PROCESSLIST;

-- Exibe e filtra variáveis do servidor
SHOW VARIABLES\G;
SHOW VARIABLES LIKE '%max_con%'\G;

-- Operações administrativas temporárias
SET sql_log_bin = OFF;           -- Desativa log binário nesta sessão (grandes cargas)
SET foreign_key_checks = OFF;    -- Desativa checagem de chave estrangeira (importações)
```

### 4. Estatísticas do Servidor e Queries de Diagnóstico

```sql
-- Status global, por sessão e por thread
SHOW GLOBAL STATUS;
SHOW STATUS;
SHOW GLOBAL STATUS LIKE '%thread%';

-- Reset estatísticas, medir desempenho de queries específicas
FLUSH STATUS;
SELECT * FROM employees.employees WHERE first_name LIKE 'An%';
SHOW STATUS LIKE 'Handler%';

-- Detalhe do motor de armazenamento InnoDB e Performance Schema
SHOW ENGINE INNODB STATUS\G;
SHOW ENGINE PERFORMANCE_SCHEMA STATUS;
```

### 5. Queries Information_schema para Auditoria e Análise

```sql
-- Tamanho dos bancos
SELECT
  table_schema AS database_name,
  ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS total_size_mb,
  ROUND(SUM(data_length) / 1024 / 1024, 2) AS data_size_mb,
  ROUND(SUM(index_length) / 1024 / 1024, 2) AS index_size_mb,
  COUNT(*) AS number_of_tables
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql', 'performance_schema', 'information_schema', 'sys')
GROUP BY table_schema
ORDER BY total_size_mb DESC;

-- Tamanho das tabelas
SELECT
  table_schema,
  table_name,
  ROUND(data_length / 1024 / 1024, 2) AS data_mb,
  ROUND(index_length / 1024 / 1024, 2) AS index_mb,
  ROUND((data_length + index_length) / 1024 / 1024, 2) AS total_mb
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql', 'performance_schema', 'information_schema', 'sys')
ORDER BY total_mb DESC;

-- Mapeamento de colunas "grandes"
SELECT table_schema, table_name, column_name, data_type
FROM information_schema.columns
WHERE data_type IN('text', 'blob', 'mediumtext', 'long_text')
  AND table_schema NOT IN('information_schema','mysql','sys','performance_schema')
LIMIT 10\G;

-- Tabelas sem chave primária
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
  AND table_type = 'BASE TABLE'
  AND table_name NOT IN (
    SELECT DISTINCT table_name
    FROM information_schema.statistics
    WHERE index_name = 'PRIMARY'
  );

-- Colunas nulas por schema
SELECT table_schema, table_name, column_name, is_nullable
FROM information_schema.columns
WHERE is_nullable = 'YES'
  AND table_schema NOT IN ('mysql','information_schema','performance_schema','sys');

-- Contagem de objetos por banco
SELECT table_schema, COUNT(*) AS total_objetos,
       SUM(table_type = 'BASE TABLE') AS tabelas,
       SUM(table_type = 'VIEW') AS views
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
GROUP BY table_schema\G;
```

### 6. Diagnóstico SYS

```sql
-- Resumo de operações de I/O por host/tipo/usuário
SELECT * FROM sys.host_summary_by_file_io;
SELECT * FROM sys.host_summary_by_file_io_type;
SELECT * FROM sys.user_summary_by_file_io;
SELECT * FROM sys.user_summary_by_file_io_type;

-- Locks e uso de memória
SELECT * FROM sys.innodb_lock_waits\G;
SELECT * FROM sys.memory_by_thread_by_current_bytes WHERE user = 'root@localhost'\G;
SELECT * FROM sys.schema_unused_indexes\G;
CALL sys.ps_truncate_all_tables(FALSE);    -- Limpa tabelas do Performance Schema (pré-testes)
```

### 7. Performance Schema – Instrumentação

```sql
-- Instrumentos ativos e configuração
SELECT name, enabled, timed FROM performance_schema.setup_instruments WHERE enabled = 'YES';
UPDATE performance_schema.setup_instruments SET ENABLED = 'NO' WHERE name LIKE 'wait/io/file/%';
UPDATE performance_schema.setup_consumers SET ENABLED = 'YES' WHERE NAME = 'events_waits_current';

-- Objetos e uso de memória por evento
SELECT * FROM performance_schema.setup_objects;
SELECT event_name, count_alloc counter, sum_number_of_bytes_alloc allocated, 
       low_number_of_bytes_used minimum, high_number_of_bytes_used maximum
FROM performance_schema.memory_summary_by_user_by_event_name
WHERE event_name LIKE '%sort%' AND user = 'root';

-- Locks detalhados
SELECT * FROM performance_schema.data_lock_waits\G;
SELECT thread_id, event_id, index_name, lock_type, lock_mode, lock_status, lock_data
FROM performance_schema.data_locks WHERE object_name ='employees';
```

### 8. Análise do Ambiente Linux

```bash
# Visualização de processos e recursos em tempo real
top
htop
btop

# Estatísticas do sistema (CPU/memória/discos)
vmstat 1 10         # A cada 1s, até 10 amostras
free -h             # Memória
df -h               # Espaço em disco

# Ferramentas de diagnóstico de disco/I/O
apt-get install iotop sysstat
iostat 1 10

apt-get install hdparm
hdparm --direct -t -T /dev/sda1
```

### 9. Testes de Desempenho de Disco

```bash
dd if=/dev/zero of=/test bs=256M count=1 oflag=dsync
dd if=/dev/zero of=teste bs=1M count=1024 conv=fdatasync
dd if=/dev/zero of=teste bs=1G count=1 conv=fdatasync
```
**Exemplo de resultado:**
```
1073741824 bytes (1,1 GB, 1,0 GiB) copied, 3.47 s, 309 MB/s
1073741824 bytes (1,1 GB, 1,0 GiB) copied, 6.47 s, 166 MB/s
```