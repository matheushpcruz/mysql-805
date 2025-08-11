# Aula 08 – Tuning e Particionamento

### 1. Análise de Espera e Consultas no Performance Schema

```sql
-- Eventos que mais consomem tempo
SELECT EVENT_NAME, count_star, sum_timer_wait
FROM events_waits_summary_global_by_event_name
ORDER BY sum_timer_wait DESC
LIMIT 10;

-- Consultas mais frequentes
SELECT DIGEST_TEXT, COUNT_STAR, SUM_TIMER_WAIT
FROM performance_schema.events_statements_summary_by_digest
ORDER BY COUNT_STAR DESC
LIMIT 50;

-- Consultas mais demoradas
SELECT DIGEST_TEXT, COUNT_STAR, SUM_TIMER_WAIT
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 50;

-- Tabelas mais acessadas
SELECT
  OBJECT_SCHEMA AS database_name,
  OBJECT_NAME AS table_name,
  SUM(COUNT_READ + COUNT_WRITE + COUNT_FETCH + COUNT_INSERT + COUNT_UPDATE + COUNT_DELETE) AS total_accesses,
  SUM(SUM_TIMER_WAIT) / 1000000000000 AS total_wait_seconds
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'information_schema', 'sys')
GROUP BY OBJECT_SCHEMA, OBJECT_NAME
ORDER BY total_accesses DESC;

-- Acesso detalhado às esperas globais
SELECT EVENT_NAME, count_star, sum_timer_wait, (SUM_TIMER_WAIT / 1000000000000) AS total_wait_seconds
FROM events_waits_summary_global_by_event_name
ORDER BY sum_timer_wait DESC
LIMIT 10;

SELECT * FROM waits_global_by_latency;
```

### 2. Comandos para Tuning & Variáveis

```sql
SHOW STATUS LIKE 'Opened_tables';
SHOW STATUS LIKE 'table_open_cache';
SHOW VARIABLES LIKE 'table_open_cache';
```

```ini
# Exemplos de configuração no my.cnf (ou em runtime):

innodb_buffer_pool_size = 8G    # Recomenda-se 70-80% da RAM disponível
innodb_read_io_threads = 4      # Em geral: metade da quantidade de CPUs
innodb_write_io_threads = 4
```

### 3. MySQLTuner – Diagnóstico do Ambiente

```bash
git clone https://github.com/major/MySQLTuner-perl.git
cd MySQLTuner-perl/
perl mysqltuner.pl
```
> Permite realizar uma varredura rápida na configuração do MySQL, sugerindo ajustes automáticos.

### 4. Tuning do Sistema Operacional (Linux)

```bash
# Ajuste do swappiness (menos troca para swap/disco, mais memória para o MySQL)
sysctl -w vm.swappiness=10
echo 'vm.swappiness=10' >> /etc/sysctl.conf
```

### 5. Gerenciamento de Discos Externos para o MySQL

```bash
lsblk -f                         # Listar discos e partições
mkfs.ext4 /dev/sdb               # Formatar disco externo para ext4
mount /dev/sdb /data/            # Montar disco em /data
mkdir /data/mysql                # Criar diretório para os dados
chown -R mysql: /data/           # Ajustar permissões para o MySQL

systemctl stop mysql
mv /var/lib/mysql/* /data/       # Migrar os dados para o novo disco
chown -R mysql: /data/
nano /etc/mysql/my.cnf           # Ajustar datadir = /data

systemctl daemon-reload
systemctl start mysql

# Ajustar /etc/fstab para automontagem após reboot
blkid /dev/sdb                   # Verificar UUID
nano /etc/fstab                  # Exemplo:
# UUID=1890a3db-... /var/lib/mysql ext4 defaults,noatime 0 2
```

### 6. Benchmark de Performance

```bash
sysbench oltp_read_write --table-size=10000 --tables=5 --mysql-db=ti --mysql-user=root --mysql-password=4linux prepare
sysbench oltp_read_write --table-size=10000 --tables=5 --mysql-db=ti --mysql-user=root --mysql-password=4linux --time=30 --threads=10 run
```

### 7. Operações e Particionamento de Tabelas

```sql
USE ti;
DROP TABLE sales;
CREATE TABLE sales(
 id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
 order_date DATE NOT NULL,
 cpf VARCHAR(14),
 priority CHAR(1)
);

INSERT INTO sales (order_date, cpf, priority)
VALUES ('2025-01-10', '111.111.111-11', 'h'),
       ('2025-02-15', '222.222.222-22', 'l'),
       ('2025-03-20', '333.333.333-33', 'm'),
       ('2025-07-05', '444.444.444-44', 'c');

-- Análise do plano de execução
EXPLAIN SELECT * FROM sales WHERE id > 1000;
EXPLAIN SELECT * FROM sales WHERE id  sakila.sql
mysqldump -uroot -p --compact --quick --single-transaction --databases sakila --no-create-db --no-create-info --no-data --triggers --routines > sakila_objects.sql
mysqldump -uroot -p -h 172.27.11.10 --databases employees > employees.sql

mysql -uroot -p < employees.sql
```