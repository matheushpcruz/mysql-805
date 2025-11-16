## **Aula 07 – Manutenção, Otimização, Índices e Tablespaces no MySQL**

### **1. Criação de Tablespaces**

```sql
CREATE TABLESPACE meuspace ADD DATAFILE 'meuspace.ibd';

USE online_ddl;
ALTER TABLE produtos TABLESPACE meuspace;

USE ti;
ALTER TABLE cientistas TABLESPACE meuspace;

USE employees;
CREATE TABLE salaries_externo TABLESPACE meuspace AS SELECT * FROM salaries;

CREATE TABLE `cientistas` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `nome` VARCHAR(100) DEFAULT NULL,
  `email` VARCHAR(50) DEFAULT NULL,
  `nascimento` DATE DEFAULT NULL,
  `status` ENUM('ativo','inativo') DEFAULT 'ativo',
  `interesses` SET('computação','matemática','engenharia','física') DEFAULT NULL,
  `premiado` BIT(1) DEFAULT NULL,
  `biografia` TEXT,
  `criado_em` DATETIME DEFAULT NULL,
  `foto` MEDIUMBLOB,
  PRIMARY KEY (`id`),
  KEY `idx_nascimento` (`nascimento`)
) TABLESPACE `meuspace`;
```

```sql
SELECT * FROM information_schema.innodb_tablespaces\G;

SELECT a.NAME AS space_name, b.NAME AS table_name 
FROM information_schema.innodb_tablespaces a, information_schema.innodb_tables b 
WHERE a.SPACE = b.SPACE AND a.NAME = 'meuspace';

ALTER TABLE produtos TABLESPACE innodb_file_per_table;
ALTER TABLE ti.cientistas TABLESPACE innodb_file_per_table;
ALTER TABLE employees.salaries_externo TABLESPACE innodb_file_per_table;

DROP TABLESPACE meuspace;
```

---

### **2. Tabelas Externas**

**Adição de um novo disco no VirtualBox:**

```
Configurações > Armazenamento > Controladora SATA > Adicionar Disco Rígido
```

Criar disco de 5 GB

**Montar o disco no sistema:**

```bash
fdisk -l
apt-get install lvm2 xfsprogs
fdisk /dev/sdb
# Criar partição primária
mkfs.ext4 /dev/sdb1
mkdir /srv/storage
mount /dev/sdb1 /srv/storage/
df -h
nano /etc/fstab
# Adicionar linha:
# /dev/sdb1 /srv/storage ext4 defaults,noatime 0 2
```

**Configuração do MySQL:**

```bash
blkid
# /srv/storage:
# UUID=8194960c-8c2c-4208-9855-1f45403007bc /srv/storage ext4 defaults,noatime 0 2

show variables like '%innodb_directories%';
nano /etc/mysql/mysql.conf.d/mysqld.cnf
innodb_directories=/srv/storage/
chown -R mysql: /srv/storage/
systemctl restart mysql
```

**Criação de tabela externa:**

```sql
ALTER TABLE salaries_externo DATA DIRECTORY = '/srv/storage/';
DROP TABLE salaries_externo;

CREATE TABLE `salaries_externa` (
  `emp_no` INT NOT NULL,
  `salary` INT NOT NULL,
  `from_date` DATE NOT NULL,
  `to_date` DATE NOT NULL,
  PRIMARY KEY (`emp_no`,`from_date`)
) ENGINE=InnoDB DATA DIRECTORY = '/srv/storage/';

INSERT INTO salaries_externa SELECT * FROM salaries;
```

**Criação de tablespace externa:**

```sql
CREATE TABLESPACE extspace ADD DATAFILE '/srv/storage/employees/extspace.ibd';
SELECT * FROM information_schema.innodb_datafiles;
ALTER TABLE salaries TABLESPACE innodb_file_per_table;
```

---

### **3. InnoDB Corrompido**

```bash
dd if=/dev/zero of=/var/lib/mysql/ti/reconhecimentos.ibd bs=1 count=1024 seek=16384 conv=notrunc
systemctl restart mysql
tail -100 /var/log/mysql/error.log

nano /etc/mysql/mysql.conf.d/mysqld.cnf
innodb_force_recovery = 3
systemctl restart mysql
```

**Exportação e recuperação:**

```bash
mysqldump -uroot -p --compact --databases ti --tables reconhecimentos > /tmp/a.sql
# Remover constraints de foreign key do dump
mysql -uroot -p < /tmp/a.sql
```

**Restauração do tablespace:**

```bash
rm /var/lib/mysql/ti/reconhecimentos.ibd
cp /var/lib/mysql/curso/reconhecimentos.ibd /var/lib/mysql/ti/
chown -R mysql: /var/lib/mysql/ti/
systemctl restart mysql

ALTER TABLE reconhecimentos DISCARD TABLESPACE;
ALTER TABLE reconhecimentos IMPORT TABLESPACE;
DROP TABLE curso.reconhecimentos;
```

---

### **4. Criação de Índices**

```bash
wget http://eforexcel.com/wp/wp-content/uploads/2017/07/1500000%20Sales%20Records.zip
apt install unzip
unzip 1500000\ Sales\ Records.zip
mv 1500000\ Sales\ Records.csv /var/lib/mysql-files/sales.csv
```

**Criação da tabela e importação de dados:**

```sql
USE ti;

CREATE TABLE IF NOT EXISTS sales (
  id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  region VARCHAR(100) NOT NULL DEFAULT '',
  country VARCHAR(50) NOT NULL DEFAULT '',
  type VARCHAR(20) NOT NULL DEFAULT '',
  channel ENUM('Online', 'Offline') NOT NULL DEFAULT 'Online',
  priority CHAR(1) NOT NULL DEFAULT 'M',
  order_date DATE NULL,
  oid INT UNSIGNED NOT NULL DEFAULT 0,
  ship_date DATE NULL,
  sold SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  price FLOAT(10,2) UNSIGNED NOT NULL DEFAULT 0.0,
  cost FLOAT(10,2) UNSIGNED NOT NULL DEFAULT 0.0,
  total_revenue FLOAT(10,2) UNSIGNED NOT NULL DEFAULT 0.0,
  total_cost FLOAT(10,2) UNSIGNED NOT NULL DEFAULT 0.0,
  total_profit FLOAT(10,2) UNSIGNED NOT NULL DEFAULT 0.0
);

LOAD DATA INFILE '/var/lib/mysql-files/sales.csv'
INTO TABLE sales
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\r\n'
IGNORE 1 LINES
(region, country, type, channel, priority, @odate, oid, @sdate, sold, price, cost, total_revenue, total_cost, total_profit)
SET order_date = STR_TO_DATE(@odate, '%m/%d/%Y'),
    ship_date = STR_TO_DATE(@sdate, '%m/%d/%Y');
```

**Criação e teste de índices:**

```sql
CREATE INDEX price ON sales (price);
CREATE INDEX country ON sales (country(3));
CREATE INDEX order_date ON sales (order_date, price);
ALTER TABLE sales ADD KEY (total_cost);

EXPLAIN SELECT id, order_date, price FROM sales WHERE price < 10 LIMIT 10;
EXPLAIN SELECT order_date, price FROM sales WHERE order_date BETWEEN '2012-01-01' AND '2012-02-01' AND price < 100;
EXPLAIN SELECT * FROM cientistas c JOIN reconhecimentos r ON r.cientista_id = c.id;
EXPLAIN SELECT order_date, price FROM sales FORCE INDEX (price) WHERE order_date BETWEEN '2012-01-01' AND '2012-02-01' AND price < 100 AND country LIKE 'The%';
```
