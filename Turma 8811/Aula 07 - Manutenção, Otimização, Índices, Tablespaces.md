# Aula 07 – Manutenção, Otimização, Índices, Tablespaces e Benchmark no MySQL


### 1. Otimização, Estatísticas e Fragmentação

```sql
-- Criação de tabelas MyISAM e InnoDB a partir de uma já existente (clone)
CREATE TABLE salaries_myisam AS SELECT * FROM salaries;
ALTER TABLE salaries_myisam ENGINE=MyISAM;

CREATE TABLE salaries_innodb AS SELECT * FROM salaries;

-- Atualizar estatísticas pós-alterações
ANALYZE TABLE salaries_innodb;
ANALYZE TABLE salaries_myisam;

-- Atualizar estatísticas MyISAM: fora do MySQL
cd /var/lib/mysql/employees
myisamchk --analyze salaries_myisam

-- Consultar fragmentação de tabelas do servidor
SELECT
  table_schema,
  table_name,
  engine,
  ROUND(data_length / 1024 / 1024, 2) AS data_mb,
  ROUND(index_length / 1024 / 1024, 2) AS index_mb,
  ROUND(data_free / 1024 / 1024, 2) AS data_free_mb,
  ROUND((data_free / data_length) * 100, 2) AS fragmentation_pct
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
  AND data_free > 0
  AND data_length > 0
ORDER BY fragmentation_pct DESC
LIMIT 20;

-- Fragmentação das tabelas salaries_
SELECT
  table_schema,
  table_name,
  engine,
  ROUND(data_length / 1024 / 1024, 2) AS data_mb,
  ROUND(index_length / 1024 / 1024, 2) AS index_mb,
  ROUND(data_free / 1024 / 1024, 2) AS data_free_mb,
  ROUND((data_free / data_length) * 100, 2) AS fragmentation_pct
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
  AND table_name LIKE 'salaries_%'
ORDER BY fragmentation_pct DESC
LIMIT 20;
```

### 2. Manutenção de Dados

```sql
-- Atualizando e removendo grandes volumes de linhas
UPDATE salaries_innodb SET salary = 1 WHERE emp_no BETWEEN 200000 AND 354000;
DELETE FROM salaries_innodb WHERE emp_no BETWEEN 200000 AND 354000;
DELETE FROM salaries_myisam WHERE emp_no BETWEEN 200000 AND 354000;
```

### 3. Criação de Índices

```sql
CREATE INDEX idx_salaries_innodb ON salaries_innodb (salary, from_date, to_date);
CREATE INDEX idx_salaries_myisam ON salaries_myisam (salary, from_date, to_date);

-- Para tabela sales (após importação)
CREATE INDEX price ON sales(price);
CREATE INDEX order_date ON sales(order_date);
CREATE INDEX country ON sales(country(3));
CREATE INDEX region ON sales(region);
```

### 4. Analise de Query e EXPLAIN

```sql
-- Analisando plano de execução de SELECT
EXPLAIN SELECT id, order_date, price FROM sales WHERE price  /tmp/a.sql
mysql -uroot -p -D curso < a.sql
cp /var/lib/mysql/curso/a.ibd /var/lib/mysql/ti/
```

### 5. Recuperação InnoDB em Caso de Corrupção

```ini
# Adicionar ao /etc/mysql/my.cnf:
innodb_force_recovery = 3
```
```bash
systemctl restart mysql
systemctl stop mysql
systemctl status mysql
```

### 6. Benchmark: Carga e Stress Test

```bash
# mysqlslap: simulação simples de múltiplas conexões e inserts aleatórios
mysqlslap --concurrency=20 --iterations=100 --number-int-cols=5 --number-char-cols=5 --auto-generate-sql --commit=3 -v --user=root -p

# sysbench: simulação OLTP para banco ti
sysbench oltp_read_write --table-size=10000 --tables=20 --mysql-db=ti --mysql-user=root --mysql-password=4linux prepare
sysbench oltp_read_write --table-size=10000 --tables=20 --mysql-db=ti --mysql-user=root --mysql-password=4linux --time=30 --threads=10 run
sysbench oltp_read_write --table-size=10000 --tables=20 --mysql-db=ti --mysql-user=root --mysql-password=4linux cleanup
```
