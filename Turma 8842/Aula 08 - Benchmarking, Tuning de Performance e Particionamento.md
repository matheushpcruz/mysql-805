# Aula 08 – Tuning e Particionamento

## 1. Ferramentas de Benchmark

### 1.1. MySQLSlap

```bash
# Teste de carga: 20 conexões simultâneas, 100 repetições
# Gera tabelas e queries automaticamente para simular carga real
mysqlslap --concurrency=20 --iterations=100 --number-int-cols=5 --number-char-cols=5 --auto-generate-sql --commit=3 -uroot -p

# Criar database para testes
CREATE DATABASE mysqlslap;

# Teste de query específica: 50 conexões executando a mesma consulta
mysqlslap -uroot -p --concurrency=50 --iterations=100 --query="SELECT * FROM employees.employees;"
```

### 1.2. Sysbench

```bash
# Instalar ferramenta de benchmark
apt install sysbench

# Teste de CPU: calcula números primos até 20.000
sysbench cpu --cpu-max-prime=20000 run

# Ir para diretório de testes
cd /srv/storage/

# Preparar arquivos de 2GB para teste de disco
sysbench fileio --file-total-size=2G prepare

# Executar teste de I/O: leitura e escrita aleatórias
sysbench fileio --file-total-size=2G --file-test-mode=rndrw run

# Limpar arquivos de teste
sysbench fileio --file-total-size=2G cleanup

# Criar database e usuário para testes OLTP
CREATE DATABASE sysbench;
CREATE USER sysbench@localhost IDENTIFIED BY 'aluno123';
GRANT ALL ON sysbench.* TO sysbench@localhost;

# Preparar 10 tabelas com 100k registros cada
sysbench oltp_read_write --table-size=100000 --tables=10 --mysql-db=sysbench --mysql-user=sysbench --mysql-password=aluno123 prepare

# Executar teste: 2 threads por 30 segundos
# Simula transações reais (INSERT, UPDATE, SELECT, DELETE)
sysbench oltp_read_write --table-size=100000 --tables=10 --mysql-db=sysbench --mysql-user=sysbench --mysql-password=aluno123 --threads=2 --time=30 run

# Limpar dados de teste
sysbench oltp_read_write --table-size=100000 --tables=10 --mysql-db=sysbench --mysql-user=sysbench --mysql-password=aluno123 cleanup
```

## 2. Análise de Performance com Performance Schema

```sql
-- Top 5 eventos que mais consomem tempo (identifica gargalos)
SELECT event_name, count_star, sum_timer_wait 
FROM events_waits_summary_global_by_event_name 
ORDER BY sum_timer_wait DESC 
LIMIT 5;

-- Visão simplificada de latências (sys schema)
SELECT * FROM waits_global_by_latency LIMIT 5;
```

## 3. Monitoramento de Variáveis e Status

### 3.1. Cache de Tabelas

```sql
-- Quantas vezes tabelas foram abertas (alto = cache insuficiente)
SHOW STATUS LIKE 'Opened_tables';

-- Tamanho atual do cache de tabelas abertas
SHOW VARIABLES LIKE 'table_open_cache';
```

### 3.2. Threads e Conexões

```sql
-- Estatísticas de threads (criadas, em cache, conectadas, rodando)
SHOW STATUS LIKE 'threads_%';

-- Total de tentativas de conexão ao servidor
SHOW STATUS LIKE 'Connections%';

-- Quantidade de threads mantidas em cache (reutilização)
SHOW VARIABLES LIKE 'thread_cache_size';
```

### 3.3. Variáveis InnoDB

```sql
-- Todas as configurações do storage engine InnoDB
SHOW VARIABLES LIKE 'innodb_%';
```

## 4. Tuning de Configuração MySQL

### 4.1. Configurações Recomendadas

```ini
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
log-error       = /var/log/mysql/error.log

# Permite usar diretórios externos para tablespaces
innodb_directories='/srv/storage/;/data/;'

# Modo de recuperação forçada (usar apenas em emergências)
# Valores: 1-6 (quanto maior, mais agressivo)
#innodb_force_recovery = 3

# Buffer pool: cache de dados e índices na memória
# Recomendado: 70-80% da RAM total disponível
innodb_buffer_pool_size = 3G

# Número máximo de conexões simultâneas permitidas
max_connections = 2000
```

### 4.2. Ajuste do Vagrant (quando aplicável)

```ruby
# Aumentar memória da VM para 4GB
'db1' => {'ip' => '10', 'memory' => 4096, 'box' => 'debian', 'script' => 'debian.sh'}
```

### 4.3. Cálculo do InnoDB Log Size

```sql
-- Ver configurações atuais dos logs de transação
SHOW VARIABLES LIKE 'innodb_log_%';

-- Ver quanto foi escrito nos logs (em bytes)
SHOW GLOBAL STATUS LIKE 'Innodb_os_log_written';

-- Aguardar 1 minuto para medir crescimento
SELECT SLEEP(60);

-- Verificar novamente o valor
SHOW GLOBAL STATUS LIKE 'Innodb_os_log_written';

-- Calcular MB escritos por minuto (diferença entre valores)
SELECT (418677760 - 343081472) / 1024 / 1024 AS MB_per_min;

-- Calcular tamanho ideal: log deve caber 1 hora de escrita
-- Formula: MB_por_min * 60 / 2 (dois arquivos de log)
SELECT 72 * 60 / 2;
```

## 5. MySQLTuner – Diagnóstico Automatizado

```bash
# Baixar ferramenta de análise de configuração
git clone https://github.com/major/MySQLTuner-perl.git
cd MySQLTuner-perl/

# Executar análise: sugere ajustes baseado no uso real
perl mysqltuner.pl
```

## 6. Tuning do Sistema Operacional

```bash
# Reduzir uso de swap (troca para disco)
# 10 = usar swap apenas quando necessário (padrão é 60)
# Mantém mais dados do MySQL em RAM
sysctl -w vm.swappiness=10
```

## 7. Particionamento de Tabelas

### 7.1. Criação de Tabela para Particionamento

```sql
USE ti;

DROP TABLE sales;

-- Criar tabela simples para demonstração
CREATE TABLE sales (
    id INT NOT NULL AUTO_INCREMENT,
    order_date DATE NOT NULL,
    cpf VARCHAR(14),
    priority CHAR(1),
    PRIMARY KEY(id)
);

-- Inserir dados de exemplo
INSERT INTO sales (order_date, cpf, priority) VALUES
('2025-01-10', '111.111.111-11', 'h'),
('2025-02-15', '222.222.222-22', 'l'),
('2025-03-20', '333.333.333-33', 'm'),
('2025-07-05', '444.444.444-44', 'c');

-- Inserir com IDs específicos para testar partições
INSERT INTO sales (id, order_date, cpf, priority) VALUES 
(1000, '2025-01-10', '111.111.111-11', 'h'),
(1001, '2025-02-15', '222.222.222-22', 'l'),
(1002, '2025-03-20', '333.333.333-33', 'm'),
(1003, '2025-07-05', '444.444.444-44', 'c');
```

### 7.2. Particionamento por RANGE

```sql
-- Dividir tabela em partições por faixa de ID
-- p0: IDs < 1000, p1: IDs 1000-1999, pmax: IDs >= 2000
ALTER TABLE sales PARTITION BY RANGE(id) (
    PARTITION p0 VALUES LESS THAN (1000),
    PARTITION p1 VALUES LESS THAN (2000),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- Ver quantos registros em cada partição
SELECT PARTITION_NAME, TABLE_ROWS 
FROM information_schema.PARTITIONS 
WHERE TABLE_NAME = 'sales';

-- Inserir mais dados para popular diferentes partições
INSERT INTO sales (id, order_date, cpf, priority) VALUES 
(2000, '2025-01-10', '111.111.111-11', 'h'),
(2001, '2025-02-15', '222.222.222-22', 'l'),
(2002, '2025-03-20', '333.333.333-33', 'm'),
(3003, '2025-07-05', '444.444.444-44', 'c');

-- Ver plano de execução: MySQL só acessa partições necessárias
-- Partition pruning: otimização que ignora partições irrelevantes
EXPLAIN SELECT * FROM sales WHERE id > 1000;  # Ignora p0
EXPLAIN SELECT * FROM sales WHERE id < 1000;  # Usa só p0
EXPLAIN SELECT * FROM sales WHERE id > 2000;  # Usa só pmax
```

### 7.3. Reorganização de Partições

```sql
-- Dividir partição pmax em duas novas partições
-- p2: IDs 2000-2999, p3: IDs >= 3000
ALTER TABLE sales REORGANIZE PARTITION pmax INTO (
    PARTITION p2 VALUES LESS THAN (3000),
    PARTITION p3 VALUES LESS THAN MAXVALUE
);

-- Deletar partição p0 e todos os seus dados
ALTER TABLE sales DROP PARTITION p0;

-- Remover particionamento: volta a ser tabela normal
ALTER TABLE sales REMOVE PARTITIONING;
```

### 7.4. Particionamento por LIST (baseado em data)

```sql
-- Chave primária deve incluir coluna usada no particionamento
ALTER TABLE sales DROP PRIMARY KEY, ADD PRIMARY KEY (id, order_date);

-- Particionar por trimestre: extrai mês da data
-- p1: Jan-Mar, p2: Abr-Jun, p3: Jul-Dez
ALTER TABLE sales PARTITION BY LIST (MONTH(order_date)) (
    PARTITION p1 VALUES IN(1,2,3),
    PARTITION p2 VALUES IN(4,5,6),
    PARTITION p3 VALUES IN(7,8,9,10,11,12)
);

-- Ver plano: MySQL acessa só a partição do trimestre correto
EXPLAIN SELECT * FROM sales WHERE order_date = '2025-01-10';  # p1
EXPLAIN SELECT * FROM sales WHERE order_date = '2025-04-10';  # p2
EXPLAIN SELECT * FROM sales WHERE order_date = '2025-07-10';  # p3

-- Ver detalhes completos de todas as partições
SELECT PARTITION_NAME, TABLE_ROWS, PARTITION_EXPRESSION, PARTITION_DESCRIPTION 
FROM information_schema.PARTITIONS 
WHERE TABLE_NAME = 'sales';

-- Remover particionamento
ALTER TABLE sales REMOVE PARTITIONING;
```

