# Aula 09 – Particionamento Avançado e Backup

## 1. Particionamento por RANGE COLUMNS (Datas)

### 1.1. Preparação da Chave Primária

```sql
-- A chave primária deve incluir a coluna usada no particionamento
ALTER TABLE sales DROP PRIMARY KEY, ADD PRIMARY KEY (id, order_date);
```

### 1.2. Criação de Partições por Trimestre

```sql
-- Particionar por faixas de datas (trimestres)
-- Cada partição contém dados de um trimestre específico
ALTER TABLE sales PARTITION BY RANGE COLUMNS(order_date) (
    PARTITION p2024_q1 VALUES LESS THAN ('2024-04-01'),
    PARTITION p2024_q2 VALUES LESS THAN ('2024-07-01'),
    PARTITION p2024_q3 VALUES LESS THAN ('2024-10-01'),
    PARTITION p2024_q4 VALUES LESS THAN ('2025-01-01'),
    PARTITION p2025_q1 VALUES LESS THAN ('2025-04-01'),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- Testar partition pruning: MySQL acessa só a partição necessária
EXPLAIN SELECT * FROM sales WHERE order_date = '2024-01-10';  # p2024_q1
EXPLAIN SELECT * FROM sales WHERE order_date = '2024-06-10';  # p2024_q2

-- Remover particionamento
ALTER TABLE sales REMOVE PARTITIONING;
```

## 2. Particionamento por HASH

### 2.1. HASH Baseado em Função

```sql
-- Distribui dados entre 4 partições baseado no ano
-- HASH: algoritmo de distribuição automática e uniforme
ALTER TABLE sales PARTITION BY HASH(YEAR(order_date)) PARTITIONS 4;

-- Ver informações detalhadas das partições criadas
SELECT PARTITION_NAME, TABLE_ROWS, PARTITION_EXPRESSION, PARTITION_DESCRIPTION 
FROM information_schema.partitions 
WHERE table_name = 'sales';

-- Remover particionamento
ALTER TABLE sales REMOVE PARTITIONING;
```

## 3. Particionamento por KEY

### 3.1. Preparação e Criação

```sql
-- KEY: usa algoritmo interno do MySQL para distribuir dados
-- Mais eficiente que HASH para chaves primárias

-- Ajustar chave primária para apenas ID
ALTER TABLE sales DROP PRIMARY KEY, ADD PRIMARY KEY (id);

-- Particionar usando chave primária: distribui em 4 partições
ALTER TABLE sales PARTITION BY KEY (id) PARTITIONS 4;

-- Remover particionamento
ALTER TABLE sales REMOVE PARTITIONING;
```

### 3.2. KEY com Chave Composta

```sql
-- Particionar usando múltiplas colunas
ALTER TABLE sales DROP PRIMARY KEY, ADD PRIMARY KEY (id, cpf);

-- KEY pode usar várias colunas para calcular distribuição
ALTER TABLE sales PARTITION BY KEY (id) PARTITIONS 4;
```

## 4. Manutenção de Partições

### 4.1. Operações de Otimização

```sql
-- Analisar todas as partições: atualiza estatísticas
-- Ajuda o otimizador a escolher melhor plano de execução
ALTER TABLE sales ANALYZE PARTITION ALL;

-- Otimizar partição específica: desfragmenta e reorganiza
-- Libera espaço não utilizado e melhora performance
ALTER TABLE sales OPTIMIZE PARTITION p0;

-- Reconstruir partição: recreia completamente a partição
-- Útil após muitas operações de DELETE/UPDATE
ALTER TABLE sales REBUILD PARTITION p0;
```

## 5. Backup e Restore com mysqldump

### 5.1. Preparação do Ambiente

```bash
# Limpar databases de teste antigos
DROP DATABASE employees;
DROP DATABASE sakila;

# Instalar Git para clonar repositórios
apt install git

# Baixar database de exemplo (employees)
git clone https://github.com/datacharmer/test_db.git
cd test_db/

# Importar database employees
mysql -uroot -p < employees.sql

# Importar database sakila (schema + dados)
cd sakila/
mysql -uroot -p < sakila-mv-schema.sql
mysql -uroot -p < sakila-mv-data.sql
```

### 5.2. Backup Completo

```sql
-- Backup de todos os databases
-- --compact: remove comentários e comandos extras
-- --quick: não carrega toda tabela na memória
-- --single-transaction: backup consistente sem lock
mysqldump -uroot -p --compact --quick --single-transaction --all-databases > curso.sql
```

### 5.3. Backup Específico com Rotinas

```sql
-- Backup de database específico incluindo stored procedures/triggers
-- -B (--databases): inclui comando CREATE DATABASE
-- --triggers: inclui triggers
-- --routines: inclui stored procedures e functions
mysqldump -uroot -p --compact --quick --single-transaction --triggers --routines -B employees > employees.sql

-- Testar restore: deletar e recriar
DROP DATABASE employees;
```

### 5.4. Restore com Controle de Foreign Keys

```sql
-- Desabilitar verificação de foreign keys (sessão)
SET FOREIGN_KEY_CHECKS = 0;

-- Desabilitar globalmente (afeta todas as conexões)
SET GLOBAL FOREIGN_KEY_CHECKS = 0;

-- Restaurar backup
mysql -uroot -p < employees.sql

-- Reabilitar verificação de foreign keys
SET GLOBAL FOREIGN_KEY_CHECKS = 1;
```

## 6. Backup Compactado

### 6.1. Backup com GZIP

```sql
-- Criar backup compactado com data no nome
-- $(date +%Y-%m-%d): adiciona timestamp ao nome do arquivo
mysqldump -uroot -p --compact --quick --single-transaction --triggers --routines -B employees | gzip > employees-$(date +%Y-%m-%d).sql.gz

-- Descompactar arquivo específico
gzip -d employees-2025-11-13.sql.gz

-- Testar restore de arquivo compactado
DROP DATABASE employees;
SET GLOBAL FOREIGN_KEY_CHECKS = 0;

-- zcat: descompacta e envia direto para mysql (sem arquivo intermediário)
zcat employees-2025-11-13.sql.gz | mysql -uroot -p

SET GLOBAL FOREIGN_KEY_CHECKS = 1;
```

### 6.2. Backup com BZIP2 (Maior Compressão)

```sql
-- Instalar ferramentas de compressão paralela
apt-get install pigz lbzip2

-- Criar backup com bzip2 (melhor compressão que gzip)
-- -9: nível máximo de compressão
mysqldump -uroot -p --compact --quick --single-transaction --triggers --routines -B employees | bzip2 -9 > employees-$(date +%Y-%m-%d).sql.bz2
```

## 7. Backup em Arquivos Separados (--tab)

### 7.1. Preparação do Diretório

```sql
-- Criar diretório para arquivos de backup
mkdir /var/lib/mysql-files/employees

-- Ajustar permissões: MySQL precisa escrever aqui
chown -R mysql: /var/lib/mysql-files/
```

### 7.2. Backup em Formato Texto

```sql
-- --tab: cria 2 arquivos por tabela:
-- .sql: estrutura da tabela (CREATE TABLE)
-- .txt: dados em formato CSV (separado por TAB)
mysqldump -uroot -p --tab=/var/lib/mysql-files/employees/ employees
```

### 7.3. Restore de Backup --tab

```sql
-- Deletar e recriar database
DROP DATABASE employees;
CREATE DATABASE employees;

-- Restaurar estrutura: executar todos os .sql
-- -f: continua mesmo se houver erros
cat /var/lib/mysql-files/employees/*.sql | mysql -f employees -uroot -p

-- Restaurar dados: importar todos os .txt
-- mysqlimport: ferramenta específica para arquivos de dados
SET GLOBAL FOREIGN_KEY_CHECKS = 0;
mysqlimport employees /var/lib/mysql-files/employees/*.txt -uroot -p
SET GLOBAL FOREIGN_KEY_CHECKS = 1;
```

## 8. MyDumper – Backup Paralelo de Alta Performance

### 8.1. Instalação

```bash
# Adicionar chave GPG do repositório
wget -qO- 'https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x1D357EA7D10C9320371BDD0279EA15C0E82E34BA&exact=on' | sudo tee /etc/apt/keyrings/mydumper.asc

# Adicionar repositório do MyDumper
echo "deb [signed-by=/etc/apt/keyrings/mydumper.asc] https://mydumper.github.io/mydumper/repo/apt/debian $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/mydumper.list

# Instalar MyDumper e MyLoader
apt-get update && apt-get install mydumper
```

### 8.2. Configuração de Usuário

```sql
-- Criar usuário específico para backup com permissões necessárias
CREATE USER 'mydumper'@'%' IDENTIFIED BY 'aluno123';

-- Permissões mínimas requeridas:
-- SELECT: ler dados
-- LOCK TABLES: garantir consistência
-- SHOW VIEW: backup de views
-- EVENT: backup de eventos
-- TRIGGER: backup de triggers
-- REPLICATION CLIENT: informações de binlog
-- BACKUP_ADMIN: operações de backup (MySQL 8.0+)
-- RELOAD: flush tables
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER, REPLICATION CLIENT, BACKUP_ADMIN, RELOAD 
ON *.* TO 'mydumper'@'%';
```

### 8.3. Backup com MyDumper

```bash
# Backup single-thread (padrão)
# -B: especifica database
# -o: diretório de saída
mydumper -u mydumper -p aluno123 -h localhost -P 3306 -B employees -o /tmp/employees

# Ver arquivos criados
cd /tmp/employees/

# Backup paralelo com 2 threads
# --threads: número de conexões paralelas (acelera o backup)
# Cada thread processa tabelas diferentes simultaneamente
mydumper -u mydumper -p aluno123 -h localhost -P 3306 -B employees --threads=2 -o /tmp/employees
```

### 8.4. Restore com MyLoader

```bash
# Preparar para restore
DROP DATABASE employees;
SET GLOBAL FOREIGN_KEY_CHECKS = 0;

# Restaurar com MyLoader
# -d: diretório com backup
# --threads: paralelização do restore
# --verbose 3: log detalhado (níveis: 1=info, 2=debug, 3=trace)
myloader -u root -p aluno123 -h localhost -P 3306 -d /tmp/employees/ --threads 2 --verbose 3
```

## 9. Cópia Física de Dados (Cold Backup)

### 9.1. Preparação dos Servidores

```bash
# Iniciar segunda VM (db2)
vagrant up db2

# Instalar rsync em ambos servidores
apt-get install rsync

# Parar MySQL em db1 (origem)
# No db1:
systemctl stop mysql

# Parar MySQL em db2 (destino)
# No db2:
systemctl stop mysql
```

### 9.2. Cópia com Rsync

```bash
# Limpar dados antigos no destino
# No db2:
rm -rf /var/lib/mysql/*

# Ver IP do servidor de origem
# No db1:
ip a

# Copiar dados via rsync (mantém permissões e atributos)
# -a: archive mode (preserva tudo)
# -v: verbose
# No db1:
rsync -av /var/lib/mysql/* root@172.27.11.20:/var/lib/mysql/

# Iniciar MySQL no servidor de origem
systemctl start mysql
```

### 9.3. Cópia com SCP (Alternativa)

```bash
# Preparar servidor destino
# No DB2:
systemctl stop mysql
rm -rf /var/lib/mysql/*

# Copiar via SCP
# -r: recursivo (copia diretórios)
# No db1:
scp -r /var/lib/mysql/* root@172.27.11.20:/var/lib/mysql/

# Ajustar permissões no destino
# No db2:
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql
```

## 10. LVM Snapshots para Backup

### 10.1. Preparação do Disco e LVM

```bash
# Desligar VM
vagrant halt db1

# No VirtualBox:
# Configurações > Armazenamento > Adicionar Disco > Criar > 10GB > Finalizar

# Iniciar VM
vagrant up db1

# Instalar ferramentas LVM e sistema de arquivos
apt-get install lvm2 xfsprogs

# Particionar novo disco
fdisk /dev/sdc
```

### 10.2. Criação de Partição LVM

```bash
# Criar nova partição
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-20971519, default 2048): [ENTER]
Last sector (2048-20971519, default 20971519): [ENTER]

Created a new partition 1 of type 'Linux' and of size 10 GiB.

# Alterar tipo para LVM
Command (m for help): t
Hex code or alias (type L to list all): 8e
Changed type of partition 'Linux' to 'Linux LVM'.

# Salvar e sair
Command (m for help): w
```

### 10.3. Configuração LVM

```bash
# Criar Physical Volume (PV)
pvcreate /dev/sdc1

# Criar Volume Group (VG) chamado 'mysql'
vgcreate mysql /dev/sdc1

# Criar Logical Volume (LV) de 3GB para dados
lvcreate -n 'mysql-data' -L '3g' mysql

# Formatar com XFS e adicionar label
mkfs.xfs -L 'mysql-data' /dev/mysql/mysql-data

# Criar ponto de montagem
mkdir /srv/mysql

# Parar MySQL para mover dados
systemctl stop mysql

# Montar novo volume
mount /dev/mysql/mysql-data /srv/mysql/

# Copiar dados do MySQL
cp -r /var/lib/mysql/* /srv/mysql/

# Ajustar permissões
chown -R mysql: /srv/mysql/

# Alterar configuração do MySQL
nano /etc/mysql/mysql.conf.d/mysqld.cnf
# Adicionar/alterar linha:
datadir = /srv/mysql
```

### 10.4. Fluxo Completo de Snapshot

```bash
# FASE 1: CRIAR BACKUP
# =====================
# 1. Criar snapshot do volume de dados ativo (MySQL rodando)
lvcreate --size '3g' --snapshot --name 'mysql-snap' /dev/mysql/mysql-data

# 2. Fazer backup do snapshot (compactado)
dd if=/dev/mysql/mysql-snap | gzip > /root/mysql-snap.gz

# FASE 2: RESTAURAR BACKUP
# =========================
# 3. Criar novo volume para restore
lvcreate -n 'mysql-restored' -L '3g' mysql

# 4. Restaurar dados do backup
gzip -d -c /root/mysql-snap.gz | dd of=/dev/mysql/mysql-restored

# 5. Preparar ambiente para usar volume restaurado
mkdir /srv/restore
systemctl stop mysql
umount /srv/mysql  # Se houver volume montado

# 6. Montar volume restaurado
mount -o nouuid /dev/mysql/mysql-restored /srv/restore/

# 7. Alterar configuração do MySQL
nano /etc/mysql/mysql.conf.d/mysqld.cnf
# datadir = /srv/restore

# 8. Ajustar permissões e iniciar
chown -R mysql: /srv/restore/
systemctl start mysql

# FASE 3: RETORNAR AO NORMAL (OPCIONAL)
# =======================================
# 9. Voltar para configuração original
nano /etc/mysql/mysql.conf.d/mysqld.cnf
# datadir = /var/lib/mysql
systemctl restart mysql
```