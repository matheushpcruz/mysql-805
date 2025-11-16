# Aula 10 – Percona XtraBackup e Replicação MySQL

## 1. Percona XtraBackup – Backup Hot (Online)

### 1.1. Instalação do Percona XtraBackup

```bash
# Documentação oficial
# https://docs.percona.com/percona-xtrabackup/8.4/apt-repo.html

# Baixar repositório Percona
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb

# Instalar pacote de repositório
dpkg -i percona-release_latest.generic_all.deb

# Corrigir dependências caso necessário
apt install -f

# Habilitar repositório do XtraBackup 8.4 LTS
percona-release enable pxb-84-lts

# Instalar Percona XtraBackup
apt install -y percona-xtrabackup-84
```

### 1.2. Configuração de Usuário e Permissões

```sql
-- Criar usuário específico para backup
CREATE USER xtrabackup@localhost IDENTIFIED BY 'aluno123';

-- Permissões essenciais para backup:
-- BACKUP_ADMIN: permissão para fazer backup (MySQL 8.0+)
-- PROCESS: ver processos em execução
-- RELOAD: executar FLUSH commands
-- LOCK TABLES: travar tabelas durante backup
-- REPLICATION CLIENT: ler posição dos binlogs
GRANT BACKUP_ADMIN, PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT 
ON *.* TO xtrabackup@localhost;

-- Permissões para ler informações do Performance Schema
GRANT SELECT ON performance_schema.log_status TO xtrabackup@localhost;
GRANT SELECT ON performance_schema.keyring_component_status TO xtrabackup@localhost;

-- Permissão para informações de replicação em grupo (se usar Group Replication)
GRANT SELECT ON performance_schema.replication_group_members TO xtrabackup@localhost;
```

### 1.3. Configuração Segura de Credenciais

```bash
# mysql_config_editor: armazena credenciais criptografadas
# Evita expor senha na linha de comando
mysql_config_editor set --host=localhost --user='xtrabackup' --password --login-path=xtrabackup
# Será solicitado a senha: aluno123
```

## 2. Backup Full (Completo)

### 2.1. Criar Backup Full

```bash
# Criar diretório para backups
mkdir -p /srv/bkps/full

# Backup full usando credenciais explícitas
xtrabackup --user=xtrabackup --password='aluno123' --backup --target-dir=/srv/bkps/full

# Limpar backup anterior
rm -rf /srv/bkps/full/*

# Backup full usando login-path (mais seguro)
# --backup: modo de backup
# --target-dir: diretório onde será salvo o backup
xtrabackup --login-path=xtrabackup --backup --target-dir=/srv/bkps/full

# Ver conteúdo do backup
cd /srv/bkps/full
```

### 2.2. Restore de Backup Full

```sql
-- Simular perda de dados
DROP DATABASE employees;
DROP DATABASE sakila;
```

```bash
# Voltar ao diretório home
cd ~/

# Parar MySQL para restore
systemctl stop mysql

# Remover dados antigos
rm -rf /var/lib/mysql/*

# FASE 1: PREPARAR O BACKUP
# Aplica logs de transação e torna backup consistente
# --prepare: aplica redo logs e desfaz transações não commitadas
xtrabackup --prepare --target-dir=/srv/bkps/full

# FASE 2: RESTAURAR OS DADOS
# Copia arquivos do backup para o datadir
# --copy-back: copia arquivos preservando estrutura
# --datadir: destino da restauração
xtrabackup --copy-back --target-dir=/srv/bkps/full --datadir=/var/lib/mysql

# Ajustar permissões dos arquivos restaurados
chown -R mysql: /var/lib/mysql

# Iniciar MySQL
systemctl restart mysql
```

### 2.3. Restore em Servidor Diferente

```bash
# SERVIDOR ORIGEM (db1)
# Preparar backup
xtrabackup --prepare --target-dir=/srv/bkps/full

# SERVIDOR DESTINO (db2)
# Parar MySQL
systemctl stop mysql

# Limpar dados
rm -rf /var/lib/mysql/*

# Ver IP do servidor destino
ip a  # inet 172.27.11.20

# SERVIDOR ORIGEM (db1)
# Copiar backup preparado para outro servidor via SCP
scp -r /srv/bkps/full/* root@172.27.11.20:/var/lib/mysql

# SERVIDOR DESTINO (db2)
# Ajustar permissões
chown -R mysql: /var/lib/mysql

# Iniciar MySQL
systemctl restart mysql
```

## 3. Backup Incremental

### 3.1. Estratégia de Backup Incremental

```bash
# Limpar backups anteriores
rm -rf /srv/bkps/full/*

# BACKUP FULL (base)
# Primeiro backup: captura estado completo do banco
xtrabackup --login-path=xtrabackup --backup --target-dir=/srv/bkps/full
```

```sql
-- Preparar dados para testar backup incremental
ALTER TABLE sales REMOVE PARTITIONING;
ALTER TABLE sales DROP PRIMARY KEY, ADD PRIMARY KEY (id);

-- Inserir dados após backup full
INSERT INTO sales (order_date, cpf, priority) VALUES 
('2027-01-10', '111.111.111-11', 'h'),
('2027-02-15', '222.222.222-22', 'l'),
('2027-03-20', '333.333.333-33', 'm'),
('2027-07-05', '444.444.444-44', 'c');
```

```bash
# BACKUP INCREMENTAL 1
# --incremental-basedir: aponta para backup anterior (full ou inc)
# Captura apenas mudanças desde o backup full
xtrabackup --login-path=xtrabackup --backup --target-dir=/srv/bkps/inc1 --incremental-basedir=/srv/bkps/full
```

```sql
-- Inserir mais dados após primeiro incremental
INSERT INTO sales (order_date, cpf, priority) VALUES 
('2028-01-10', '111.111.111-11', 'h'),
('2028-02-15', '222.222.222-22', 'l'),
('2028-03-20', '333.333.333-33', 'm'),
('2028-07-05', '444.444.444-44', 'c');
```

```bash
# BACKUP INCREMENTAL 2
# Baseia-se no incremental anterior (inc1)
# Captura apenas mudanças desde inc1
xtrabackup --login-path=xtrabackup --backup --target-dir=/srv/bkps/inc2 --incremental-basedir=/srv/bkps/inc1
```

### 3.2. Restore de Backups Incrementais

```bash
# FASE 1: PREPARAR BACKUP FULL
# --apply-log-only: aplica logs mas mantém possibilidade de aplicar incrementais
# NÃO executa rollback de transações não commitadas ainda
xtrabackup --prepare --apply-log-only --target-dir=/srv/bkps/full

# FASE 2: APLICAR PRIMEIRO INCREMENTAL
# Mescla mudanças do inc1 no backup full
xtrabackup --prepare --apply-log-only --target-dir=/srv/bkps/full --incremental-dir=/srv/bkps/inc1

# FASE 3: APLICAR SEGUNDO INCREMENTAL
# Mescla mudanças do inc2 no backup full
xtrabackup --prepare --apply-log-only --target-dir=/srv/bkps/full --incremental-dir=/srv/bkps/inc2

# FASE 4: PREPARAÇÃO FINAL
# Agora SEM --apply-log-only: executa rollback de transações não commitadas
# Deixa backup em estado consistente para restore
xtrabackup --prepare --target-dir=/srv/bkps/full
```

```sql
-- Simular perda de dados
DROP DATABASE ti;
```

```bash
# Parar MySQL
systemctl stop mysql

# Limpar datadir
rm -rf /var/lib/mysql/*

# Restaurar backup (já preparado com todos os incrementais)
xtrabackup --copy-back --target-dir=/srv/bkps/full --datadir=/var/lib/mysql

# Ajustar permissões
chown -R mysql: /var/lib/mysql

# Iniciar MySQL
systemctl start mysql
```

```sql
-- Verificar se dados estão presentes (anos 2027 e 2028)
SELECT DISTINCT YEAR(order_date) FROM sales ORDER BY YEAR(order_date);
```

## 4. Recuperação Point-in-Time com Binary Logs

### 4.1. Visualizar e Extrair Binary Logs

```sql
-- Inserir dados para testar recuperação
INSERT INTO sales (order_date, cpf, priority) VALUES 
('2029-01-10', '111.111.111-11', 'h'),
('2029-02-15', '222.222.222-22', 'l'),
('2029-03-20', '333.333.333-33', 'm'),
('2029-07-05', '444.444.444-44', 'c');
```

```bash
# Ir para diretório dos binlogs
cd /var/lib/mysql

# Visualizar conteúdo dos binary logs
# --idempotent: gera statements que podem ser executados múltiplas vezes
# -v: verbose (mostra valores dos campos)
mysqlbinlog --idempotent -v bin*
```

```sql
-- Simular erro: deletar dados acidentalmente
DELETE FROM sales WHERE YEAR(order_date) = '2029';
```

```bash
# Voltar ao home
cd ~/

# Extrair transações de período específico
# --start-datetime: início do período
# --stop-datetime: fim do período
# Útil para recuperar transações específicas
mysqlbinlog --idempotent --start-datetime='2025-11-14 18:51:38' --stop-datetime='2025-11-14 18:52:38' /var/lib/mysql/bin* > fatia.sql

# Aplicar transações extraídas
mysql -uroot -p < fatia.sql

# Extrair todas as transações de um dia
# Útil para auditoria ou recuperação de desastres
mysqlbinlog --idempotent --start-datetime='2025-11-14 00:00:00' --stop-datetime='2025-11-14 23:59:59' /var/lib/mysql/bin* > transacoes-2025-11-14.sql
```

## 5. Cenário Completo: Backup + Point-in-Time Recovery

### 5.1. Criar Backup Base

```bash
# Intervalo -> 20 minutos

# Limpar backups antigos
rm -rf /srv/bkps/*

# Criar backup full como ponto de partida
xtrabackup --login-path=xtrabackup --backup --target-dir=/srv/bkps/full
```

```sql
-- Após backup: criar tabelas de teste
USE employees;

CREATE TABLE salaries_teste AS SELECT * FROM salaries;
CREATE TABLE employees_teste AS SELECT * FROM employees;

-- Simular erro grave: TRUNCATE acidental
TRUNCATE TABLE salaries;
```

### 5.2. Extrair e Corrigir Binary Logs

```bash
# Extrair todas as transações do dia com detalhes máximos
# -vv: muito verbose (mostra todos os detalhes)
mysqlbinlog --idempotent --start-datetime='2025-11-14 00:00:00' --stop-datetime='2025-11-14 23:59:59' -vv /var/lib/mysql/bin* > /root/transacoes-erro.sql

# Identificar timestamp do erro no arquivo
# #251114 19:26:01 server id 1
# Este é o momento do TRUNCATE indesejado

# Parar MySQL para restore limpo
systemctl stop mysql

# Limpar datadir
rm -rf /var/lib/mysql/*

# Editar arquivo SQL e comentar comando problemático
# sed -i: edita arquivo in-place
# s/^truncate table salaries/# truncate table salaries/: comenta linha do TRUNCATE
sed -i 's/^truncate table salaries/# truncate table salaries/' transacoes-erro.sql
```

### 5.3. Restore Completo

```bash
# Preparar backup
xtrabackup --prepare --target-dir=/srv/bkps/full

# Restaurar backup
xtrabackup --copy-back --target-dir=/srv/bkps/full --datadir=/var/lib/mysql

# Ajustar permissões
chown -R mysql: /var/lib/mysql

# Iniciar MySQL
systemctl start mysql

# Aplicar binary logs corrigidos (sem o TRUNCATE)
# Recupera todas as transações APÓS o backup, exceto a problemática
mysql -uroot -p < /root/transacoes-erro.sql
```

## 6. Replicação MySQL

### 6.1. Configuração Inicial dos Servidores

```bash
# Editar configuração em ambos servidores
nano /etc/mysql/mysql.conf.d/mysqld.cnf

# DB1 (Master):
server-id = 1

# DB2 (Replica):
server-id = 2

# Reiniciar MySQL em ambos após alterar configuração
systemctl restart mysql
```

### 6.2. Preparar Master (DB1)

```sql
-- SERVIDOR MASTER (DB1)

-- Travar escrita para garantir consistência durante cópia inicial
FLUSH TABLES WITH READ LOCK;

-- Anotar posição atual do binary log
SHOW BINARY LOG STATUS;
-- Resultado exemplo:
-- +---------------+----------+--------------+------------------+-------------------+
-- | File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
-- +---------------+----------+--------------+------------------+-------------------+
-- | binlog.000064 |      158 |              |                  |                   |
-- +---------------+----------+--------------+------------------+-------------------+
```

```bash
# MANTER O LOCK ATIVO e executar em outro terminal:

# Parar MySQL (mantém lock até parar)
systemctl stop mysql

# SERVIDOR REPLICA (DB2)
# Limpar dados antigos
rm -rf /var/lib/mysql/*

# SERVIDOR MASTER (DB1)
# Copiar dados para replica
scp -r /var/lib/mysql/* root@172.27.11.20:/var/lib/mysql/

# SERVIDOR REPLICA (DB2)
# Ajustar permissões
chown -R mysql: /var/lib/mysql

# Iniciar MySQL no master
# No DB1:
systemctl start mysql
```

### 6.3. Configurar Usuário de Replicação

```sql
-- SERVIDOR MASTER (DB1)

-- Criar dados de teste
CREATE DATABASE replication;
USE replication;
CREATE TABLE teste(id INT NOT NULL PRIMARY KEY, valor VARCHAR(50));
INSERT INTO teste (id, valor) VALUES (1, 'replicacao funcionando');

-- Criar usuário para replicação
-- Permite apenas conexão do IP da replica
CREATE USER repl@'172.27.11.20' IDENTIFIED BY 'aluno123';

-- Permissões para replicação:
-- REPLICATION SLAVE: permite replica conectar e ler binlog
-- REPLICATION CLIENT: permite ver status de replicação
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO repl@'172.27.11.20';
```

### 6.4. Configurar Replica (DB2)

```sql
-- SERVIDOR REPLICA (DB2)

-- Configurar conexão com master
-- SOURCE_HOST: IP do servidor master
-- SOURCE_LOG_FILE: arquivo binlog anotado do SHOW BINARY LOG STATUS
-- SOURCE_LOG_POS: posição anotada do SHOW BINARY LOG STATUS
-- SOURCE_SSL: usar conexão SSL (recomendado)
-- SOURCE_USER: usuário criado para replicação
-- SOURCE_PASSWORD: senha do usuário
CHANGE REPLICATION SOURCE TO
SOURCE_HOST = '172.27.11.10',
SOURCE_LOG_FILE = 'binlog.000064',
SOURCE_LOG_POS = 158,
SOURCE_SSL = 1,
SOURCE_USER = 'repl',
SOURCE_PASSWORD = 'aluno123';

-- Ver configuração armazenada
SELECT * FROM mysql.slave_master_info;

-- Ver status da replicação (ainda não iniciada)
SHOW REPLICA STATUS\G;

-- Iniciar replicação
START REPLICA;

-- Verificar status (deve mostrar erros se houver problema)
SHOW REPLICA STATUS\G;
```

### 6.5. Resolver Problema de UUID Duplicado

```bash
# SERVIDOR REPLICA (DB2)

# Problema: MySQL usa UUID único por servidor
# Ao copiar dados, o auto.cnf (que contém UUID) é copiado também
# Isso causa conflito na replicação

# Ir para datadir
cd /var/lib/mysql

# Fazer backup do auto.cnf
mv /var/lib/mysql/auto.cnf /var/lib/mysql/auto.cnf.bkp

# Reiniciar MySQL: vai gerar novo auto.cnf com UUID único
systemctl restart mysql

# Verificar status da replicação
SHOW REPLICA STATUS\G;
```

### 6.6. Testar Replicação

```sql
-- SERVIDOR MASTER (DB1)
-- Inserir dados no master
INSERT INTO teste (id, valor) VALUES (2, 'teste de replicação');

-- SERVIDOR REPLICA (DB2)
-- Verificar se dados foram replicados automaticamente
SELECT * FROM replication.teste;
-- Deve mostrar os 2 registros
```

## 7. Replicação com XtraBackup (Sem Parar Master)

### 7.1. Backup do Master em Produção

```bash
# SERVIDOR MASTER (DB1)

# Limpar backups antigos
rm -rf /srv/bkps/*

# Criar backup HOT (sem parar MySQL)
# XtraBackup captura posição do binlog automaticamente
xtrabackup --login-path=xtrabackup --backup --target-dir=/srv/bkps/full
```

```sql
-- SERVIDOR MASTER (DB1)
-- Inserir dados DURANTE o backup (para testar replicação incremental)
INSERT INTO teste (id, valor) VALUES (3, 'teste de replicação com xtrabackup');
```

### 7.2. Restaurar Backup na Replica

```bash
# SERVIDOR MASTER (DB1)
# Preparar backup
xtrabackup --prepare --target-dir=/srv/bkps/full

# SERVIDOR REPLICA (DB2)
# Parar MySQL
systemctl stop mysql

# Limpar dados
rm -rf /var/lib/mysql/*

# SERVIDOR MASTER (DB1)
# Copiar backup preparado para replica
scp -r /srv/bkps/full/* root@172.27.11.20:/var/lib/mysql/

# SERVIDOR REPLICA (DB2)
# Ajustar permissões
chown -R mysql: /var/lib/mysql/
```

### 7.3. Configurar Replicação a partir do Backup

```sql
-- SERVIDOR MASTER (DB1)
-- Inserir dados APÓS o backup (para confirmar que replicação pegará)
INSERT INTO teste (id, valor) VALUES (4, 'teste de replicação com xtrabackup - Funcionando');
```

```bash
# SERVIDOR REPLICA (DB2)
# Ver informações do binlog capturadas pelo XtraBackup
cat /var/lib/mysql/xtrabackup_binlog_info
# Exemplo de conteúdo: binlog.000066    158
```

```sql
-- SERVIDOR REPLICA (DB2)

-- Iniciar MySQL
systemctl start mysql

-- Configurar replicação usando posição do xtrabackup_binlog_info
CHANGE REPLICATION SOURCE TO
SOURCE_HOST = '172.27.11.10',
SOURCE_LOG_FILE = 'binlog.000066',  -- do xtrabackup_binlog_info
SOURCE_LOG_POS = 158,                -- do xtrabackup_binlog_info
SOURCE_SSL = 1;

-- Verificar status
SHOW REPLICA STATUS\G;

-- Iniciar replicação com credenciais
-- Alternativa ao configurar USER/PASSWORD no CHANGE REPLICATION SOURCE
START REPLICA USER='repl' PASSWORD='aluno123';

-- Verificar se replicação está funcionando
-- Deve mostrar o registro ID 4 que foi inserido APÓS o backup
SELECT * FROM replication.teste;
```

## 8. Gestão de Binary Logs

### 8.1. Configurar Retenção de Binary Logs

```sql
-- Ver configuração atual de expiração de binlogs
-- Por padrão: 30 dias (2592000 segundos)
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';

-- Calcular 7 dias em segundos
SELECT 7*24*60*60;
-- Resultado: 604800

-- Configurar para expirar binlogs após 7 dias
SET GLOBAL binlog_expire_logs_seconds = 604800;
```

```bash
# Adicionar permanentemente no arquivo de configuração
nano /etc/mysql/mysql.conf.d/mysqld.cnf

# Adicionar linha:
binlog_expire_logs_seconds = 604800

# Reiniciar MySQL para aplicar
systemctl restart mysql
```

### 8.2. Remover Replicação

```sql
-- SERVIDOR REPLICA (DB2)

-- Parar replicação
STOP REPLICA;

-- Remover configuração de replicação completamente
-- Remove informações de master e reseta posição
RESET REPLICA ALL;
```

# Aula 10 – Percona XtraBackup e Replicação MySQL

## 1. Percona XtraBackup – Backup Hot (Online)

### 1.1. Instalação do Percona XtraBackup

```bash
# Documentação oficial
# https://docs.percona.com/percona-xtrabackup/8.4/apt-repo.html

# Baixar repositório Percona
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb

# Instalar pacote de repositório
dpkg -i percona-release_latest.generic_all.deb

# Corrigir dependências caso necessário
apt install -f

# Habilitar repositório do XtraBackup 8.4 LTS
percona-release enable pxb-84-lts

# Instalar Percona XtraBackup
apt install -y percona-xtrabackup-84
```

### 1.2. Configuração de Usuário e Permissões

```sql
-- Criar usuário específico para backup
CREATE USER xtrabackup@localhost IDENTIFIED BY 'aluno123';

-- Permissões essenciais para backup:
-- BACKUP_ADMIN: permissão para fazer backup (MySQL 8.0+)
-- PROCESS: ver processos em execução
-- RELOAD: executar FLUSH commands
-- LOCK TABLES: travar tabelas durante backup
-- REPLICATION CLIENT: ler posição dos binlogs
GRANT BACKUP_ADMIN, PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT 
ON *.* TO xtrabackup@localhost;

-- Permissões para ler informações do Performance Schema
GRANT SELECT ON performance_schema.log_status TO xtrabackup@localhost;
GRANT SELECT ON performance_schema.keyring_component_status TO xtrabackup@localhost;

-- Permissão para informações de replicação em grupo (se usar Group Replication)
GRANT SELECT ON performance_schema.replication_group_members TO xtrabackup@localhost;
```

### 1.3. Configuração Segura de Credenciais

```bash
# mysql_config_editor: armazena credenciais criptografadas
# Evita expor senha na linha de comando
mysql_config_editor set --host=localhost --user='xtrabackup' --password --login-path=xtrabackup
# Será solicitado a senha: aluno123
```

## 2. Backup Full (Completo)

### 2.1. Criar Backup Full

```bash
# Criar diretório para backups
mkdir -p /srv/bkps/full

# Backup full usando credenciais explícitas
xtrabackup --user=xtrabackup --password='aluno123' --backup --target-dir=/srv/bkps/full

# Limpar backup anterior
rm -rf /srv/bkps/full/*

# Backup full usando login-path (mais seguro)
# --backup: modo de backup
# --target-dir: diretório onde será salvo o backup
xtrabackup --login-path=xtrabackup --backup --target-dir=/srv/bkps/full

# Ver conteúdo do backup
cd /srv/bkps/full
```

### 2.2. Restore de Backup Full

```sql
-- Simular perda de dados
DROP DATABASE employees;
DROP DATABASE sakila;
```

```bash
# Voltar ao diretório home
cd ~/

# Parar MySQL para restore
systemctl stop mysql

# Remover dados antigos
rm -rf /var/lib/mysql/*

# FASE 1: PREPARAR O BACKUP
# Aplica logs de transação e torna backup consistente
# --prepare: aplica redo logs e desfaz transações não commitadas
xtrabackup --prepare --target-dir=/srv/bkps/full

# FASE 2: RESTAURAR OS DADOS
# Copia arquivos do backup para o datadir
# --copy-back: copia arquivos preservando estrutura
# --datadir: destino da restauração
xtrabackup --copy-back --target-dir=/srv/bkps/full --datadir=/var/lib/mysql

# Ajustar permissões dos arquivos restaurados
chown -R mysql: /var/lib/mysql

# Iniciar MySQL
systemctl restart mysql
```

### 2.3. Restore em Servidor Diferente

```bash
# SERVIDOR ORIGEM (db1)
# Preparar backup
xtrabackup --prepare --target-dir=/srv/bkps/full

# SERVIDOR DESTINO (db2)
# Parar MySQL
systemctl stop mysql

# Limpar dados
rm -rf /var/lib/mysql/*

# Ver IP do servidor destino
ip a  # inet 172.27.11.20

# SERVIDOR ORIGEM (db1)
# Copiar backup preparado para outro servidor via SCP
scp -r /srv/bkps/full/* root@172.27.11.20:/var/lib/mysql

# SERVIDOR DESTINO (db2)
# Ajustar permissões
chown -R mysql: /var/lib/mysql

# Iniciar MySQL
systemctl restart mysql
```

## 3. Backup Incremental

### 3.1. Estratégia de Backup Incremental

```bash
# Limpar backups anteriores
rm -rf /srv/bkps/full/*

# BACKUP FULL (base)
# Primeiro backup: captura estado completo do banco
xtrabackup --login-path=xtrabackup --backup --target-dir=/srv/bkps/full
```

```sql
-- Preparar dados para testar backup incremental
ALTER TABLE sales REMOVE PARTITIONING;
ALTER TABLE sales DROP PRIMARY KEY, ADD PRIMARY KEY (id);

-- Inserir dados após backup full
INSERT INTO sales (order_date, cpf, priority) VALUES 
('2027-01-10', '111.111.111-11', 'h'),
('2027-02-15', '222.222.222-22', 'l'),
('2027-03-20', '333.333.333-33', 'm'),
('2027-07-05', '444.444.444-44', 'c');
```

```bash
# BACKUP INCREMENTAL 1
# --incremental-basedir: aponta para backup anterior (full ou inc)
# Captura apenas mudanças desde o backup full
xtrabackup --login-path=xtrabackup --backup --target-dir=/srv/bkps/inc1 --incremental-basedir=/srv/bkps/full
```

```sql
-- Inserir mais dados após primeiro incremental
INSERT INTO sales (order_date, cpf, priority) VALUES 
('2028-01-10', '111.111.111-11', 'h'),
('2028-02-15', '222.222.222-22', 'l'),
('2028-03-20', '333.333.333-33', 'm'),
('2028-07-05', '444.444.444-44', 'c');
```

```bash
# BACKUP INCREMENTAL 2
# Baseia-se no incremental anterior (inc1)
# Captura apenas mudanças desde inc1
xtrabackup --login-path=xtrabackup --backup --target-dir=/srv/bkps/inc2 --incremental-basedir=/srv/bkps/inc1
```

### 3.2. Restore de Backups Incrementais

```bash
# FASE 1: PREPARAR BACKUP FULL
# --apply-log-only: aplica logs mas mantém possibilidade de aplicar incrementais
# NÃO executa rollback de transações não commitadas ainda
xtrabackup --prepare --apply-log-only --target-dir=/srv/bkps/full

# FASE 2: APLICAR PRIMEIRO INCREMENTAL
# Mescla mudanças do inc1 no backup full
xtrabackup --prepare --apply-log-only --target-dir=/srv/bkps/full --incremental-dir=/srv/bkps/inc1

# FASE 3: APLICAR SEGUNDO INCREMENTAL
# Mescla mudanças do inc2 no backup full
xtrabackup --prepare --apply-log-only --target-dir=/srv/bkps/full --incremental-dir=/srv/bkps/inc2

# FASE 4: PREPARAÇÃO FINAL
# Agora SEM --apply-log-only: executa rollback de transações não commitadas
# Deixa backup em estado consistente para restore
xtrabackup --prepare --target-dir=/srv/bkps/full
```

```sql
-- Simular perda de dados
DROP DATABASE ti;
```

```bash
# Parar MySQL
systemctl stop mysql

# Limpar datadir
rm -rf /var/lib/mysql/*

# Restaurar backup (já preparado com todos os incrementais)
xtrabackup --copy-back --target-dir=/srv/bkps/full --datadir=/var/lib/mysql

# Ajustar permissões
chown -R mysql: /var/lib/mysql

# Iniciar MySQL
systemctl start mysql
```

```sql
-- Verificar se dados estão presentes (anos 2027 e 2028)
SELECT DISTINCT YEAR(order_date) FROM sales ORDER BY YEAR(order_date);
```

## 4. Recuperação Point-in-Time com Binary Logs

### 4.1. Visualizar e Extrair Binary Logs

```sql
-- Inserir dados para testar recuperação
INSERT INTO sales (order_date, cpf, priority) VALUES 
('2029-01-10', '111.111.111-11', 'h'),
('2029-02-15', '222.222.222-22', 'l'),
('2029-03-20', '333.333.333-33', 'm'),
('2029-07-05', '444.444.444-44', 'c');
```

```bash
# Ir para diretório dos binlogs
cd /var/lib/mysql

# Visualizar conteúdo dos binary logs
# --idempotent: gera statements que podem ser executados múltiplas vezes
# -v: verbose (mostra valores dos campos)
mysqlbinlog --idempotent -v bin*
```

```sql
-- Simular erro: deletar dados acidentalmente
DELETE FROM sales WHERE YEAR(order_date) = '2029';
```

```bash
# Voltar ao home
cd ~/

# Extrair transações de período específico
# --start-datetime: início do período
# --stop-datetime: fim do período
# Útil para recuperar transações específicas
mysqlbinlog --idempotent --start-datetime='2025-11-14 18:51:38' --stop-datetime='2025-11-14 18:52:38' /var/lib/mysql/bin* > fatia.sql

# Aplicar transações extraídas
mysql -uroot -p < fatia.sql

# Extrair todas as transações de um dia
# Útil para auditoria ou recuperação de desastres
mysqlbinlog --idempotent --start-datetime='2025-11-14 00:00:00' --stop-datetime='2025-11-14 23:59:59' /var/lib/mysql/bin* > transacoes-2025-11-14.sql
```

## 5. Cenário Completo: Backup + Point-in-Time Recovery

### 5.1. Criar Backup Base

```bash
# Intervalo -> 20 minutos

# Limpar backups antigos
rm -rf /srv/bkps/*

# Criar backup full como ponto de partida
xtrabackup --login-path=xtrabackup --backup --target-dir=/srv/bkps/full
```

```sql
-- Após backup: criar tabelas de teste
USE employees;

CREATE TABLE salaries_teste AS SELECT * FROM salaries;
CREATE TABLE employees_teste AS SELECT * FROM employees;

-- Simular erro grave: TRUNCATE acidental
TRUNCATE TABLE salaries;
```

### 5.2. Extrair e Corrigir Binary Logs

```bash
# Extrair todas as transações do dia com detalhes máximos
# -vv: muito verbose (mostra todos os detalhes)
mysqlbinlog --idempotent --start-datetime='2025-11-14 00:00:00' --stop-datetime='2025-11-14 23:59:59' -vv /var/lib/mysql/bin* > /root/transacoes-erro.sql

# Identificar timestamp do erro no arquivo
# #251114 19:26:01 server id 1
# Este é o momento do TRUNCATE indesejado

# Parar MySQL para restore limpo
systemctl stop mysql

# Limpar datadir
rm -rf /var/lib/mysql/*

# Editar arquivo SQL e comentar comando problemático
# sed -i: edita arquivo in-place
# s/^truncate table salaries/# truncate table salaries/: comenta linha do TRUNCATE
sed -i 's/^truncate table salaries/# truncate table salaries/' transacoes-erro.sql
```

### 5.3. Restore Completo

```bash
# Preparar backup
xtrabackup --prepare --target-dir=/srv/bkps/full

# Restaurar backup
xtrabackup --copy-back --target-dir=/srv/bkps/full --datadir=/var/lib/mysql

# Ajustar permissões
chown -R mysql: /var/lib/mysql

# Iniciar MySQL
systemctl start mysql

# Aplicar binary logs corrigidos (sem o TRUNCATE)
# Recupera todas as transações APÓS o backup, exceto a problemática
mysql -uroot -p < /root/transacoes-erro.sql
```

## 6. Replicação MySQL

### 6.1. Configuração Inicial dos Servidores

```bash
# Editar configuração em ambos servidores
nano /etc/mysql/mysql.conf.d/mysqld.cnf

# DB1 (Master):
server-id = 1

# DB2 (Replica):
server-id = 2

# Reiniciar MySQL em ambos após alterar configuração
systemctl restart mysql
```

### 6.2. Preparar Master (DB1)

```sql
-- SERVIDOR MASTER (DB1)

-- Travar escrita para garantir consistência durante cópia inicial
FLUSH TABLES WITH READ LOCK;

-- Anotar posição atual do binary log
SHOW BINARY LOG STATUS;
-- Resultado exemplo:
-- +---------------+----------+--------------+------------------+-------------------+
-- | File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
-- +---------------+----------+--------------+------------------+-------------------+
-- | binlog.000064 |      158 |              |                  |                   |
-- +---------------+----------+--------------+------------------+-------------------+
```

```bash
# MANTER O LOCK ATIVO e executar em outro terminal:

# Parar MySQL (mantém lock até parar)
systemctl stop mysql

# SERVIDOR REPLICA (DB2)
# Limpar dados antigos
rm -rf /var/lib/mysql/*

# SERVIDOR MASTER (DB1)
# Copiar dados para replica
scp -r /var/lib/mysql/* root@172.27.11.20:/var/lib/mysql/

# SERVIDOR REPLICA (DB2)
# Ajustar permissões
chown -R mysql: /var/lib/mysql

# Iniciar MySQL no master
# No DB1:
systemctl start mysql
```

### 6.3. Configurar Usuário de Replicação

```sql
-- SERVIDOR MASTER (DB1)

-- Criar dados de teste
CREATE DATABASE replication;
USE replication;
CREATE TABLE teste(id INT NOT NULL PRIMARY KEY, valor VARCHAR(50));
INSERT INTO teste (id, valor) VALUES (1, 'replicacao funcionando');

-- Criar usuário para replicação
-- Permite apenas conexão do IP da replica
CREATE USER repl@'172.27.11.20' IDENTIFIED BY 'aluno123';

-- Permissões para replicação:
-- REPLICATION SLAVE: permite replica conectar e ler binlog
-- REPLICATION CLIENT: permite ver status de replicação
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO repl@'172.27.11.20';
```

### 6.4. Configurar Replica (DB2)

```sql
-- SERVIDOR REPLICA (DB2)

-- Configurar conexão com master
-- SOURCE_HOST: IP do servidor master
-- SOURCE_LOG_FILE: arquivo binlog anotado do SHOW BINARY LOG STATUS
-- SOURCE_LOG_POS: posição anotada do SHOW BINARY LOG STATUS
-- SOURCE_SSL: usar conexão SSL (recomendado)
-- SOURCE_USER: usuário criado para replicação
-- SOURCE_PASSWORD: senha do usuário
CHANGE REPLICATION SOURCE TO
SOURCE_HOST = '172.27.11.10',
SOURCE_LOG_FILE = 'binlog.000064',
SOURCE_LOG_POS = 158,
SOURCE_SSL = 1,
SOURCE_USER = 'repl',
SOURCE_PASSWORD = 'aluno123';

-- Ver configuração armazenada
SELECT * FROM mysql.slave_master_info;

-- Ver status da replicação (ainda não iniciada)
SHOW REPLICA STATUS\G;

-- Iniciar replicação
START REPLICA;

-- Verificar status (deve mostrar erros se houver problema)
SHOW REPLICA STATUS\G;
```

### 6.5. Resolver Problema de UUID Duplicado

```bash
# SERVIDOR REPLICA (DB2)

# Problema: MySQL usa UUID único por servidor
# Ao copiar dados, o auto.cnf (que contém UUID) é copiado também
# Isso causa conflito na replicação

# Ir para datadir
cd /var/lib/mysql

# Fazer backup do auto.cnf
mv /var/lib/mysql/auto.cnf /var/lib/mysql/auto.cnf.bkp

# Reiniciar MySQL: vai gerar novo auto.cnf com UUID único
systemctl restart mysql

# Verificar status da replicação
SHOW REPLICA STATUS\G;
```

### 6.6. Testar Replicação

```sql
-- SERVIDOR MASTER (DB1)
-- Inserir dados no master
INSERT INTO teste (id, valor) VALUES (2, 'teste de replicação');

-- SERVIDOR REPLICA (DB2)
-- Verificar se dados foram replicados automaticamente
SELECT * FROM replication.teste;
-- Deve mostrar os 2 registros
```

## 7. Replicação com XtraBackup (Sem Parar Master)

### 7.1. Backup do Master em Produção

```bash
# SERVIDOR MASTER (DB1)

# Limpar backups antigos
rm -rf /srv/bkps/*

# Criar backup HOT (sem parar MySQL)
# XtraBackup captura posição do binlog automaticamente
xtrabackup --login-path=xtrabackup --backup --target-dir=/srv/bkps/full
```

```sql
-- SERVIDOR MASTER (DB1)
-- Inserir dados DURANTE o backup (para testar replicação incremental)
INSERT INTO teste (id, valor) VALUES (3, 'teste de replicação com xtrabackup');
```

### 7.2. Restaurar Backup na Replica

```bash
# SERVIDOR MASTER (DB1)
# Preparar backup
xtrabackup --prepare --target-dir=/srv/bkps/full

# SERVIDOR REPLICA (DB2)
# Parar MySQL
systemctl stop mysql

# Limpar dados
rm -rf /var/lib/mysql/*

# SERVIDOR MASTER (DB1)
# Copiar backup preparado para replica
scp -r /srv/bkps/full/* root@172.27.11.20:/var/lib/mysql/

# SERVIDOR REPLICA (DB2)
# Ajustar permissões
chown -R mysql: /var/lib/mysql/
```

### 7.3. Configurar Replicação a partir do Backup

```sql
-- SERVIDOR MASTER (DB1)
-- Inserir dados APÓS o backup (para confirmar que replicação pegará)
INSERT INTO teste (id, valor) VALUES (4, 'teste de replicação com xtrabackup - Funcionando');
```

```bash
# SERVIDOR REPLICA (DB2)
# Ver informações do binlog capturadas pelo XtraBackup
cat /var/lib/mysql/xtrabackup_binlog_info
# Exemplo de conteúdo: binlog.000066    158
```

```sql
-- SERVIDOR REPLICA (DB2)

-- Iniciar MySQL
systemctl start mysql

-- Configurar replicação usando posição do xtrabackup_binlog_info
CHANGE REPLICATION SOURCE TO
SOURCE_HOST = '172.27.11.10',
SOURCE_LOG_FILE = 'binlog.000066',  -- do xtrabackup_binlog_info
SOURCE_LOG_POS = 158,                -- do xtrabackup_binlog_info
SOURCE_SSL = 1;

-- Verificar status
SHOW REPLICA STATUS\G;

-- Iniciar replicação com credenciais
-- Alternativa ao configurar USER/PASSWORD no CHANGE REPLICATION SOURCE
START REPLICA USER='repl' PASSWORD='aluno123';

-- Verificar se replicação está funcionando
-- Deve mostrar o registro ID 4 que foi inserido APÓS o backup
SELECT * FROM replication.teste;
```

## 8. Gestão de Binary Logs

### 8.1. Configurar Retenção de Binary Logs

```sql
-- Ver configuração atual de expiração de binlogs
-- Por padrão: 30 dias (2592000 segundos)
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';

-- Calcular 7 dias em segundos
SELECT 7*24*60*60;
-- Resultado: 604800

-- Configurar para expirar binlogs após 7 dias
SET GLOBAL binlog_expire_logs_seconds = 604800;
```

```bash
# Adicionar permanentemente no arquivo de configuração
nano /etc/mysql/mysql.conf.d/mysqld.cnf

# Adicionar linha:
binlog_expire_logs_seconds = 604800

# Reiniciar MySQL para aplicar
systemctl restart mysql
```

### 8.2. Remover Replicação

```sql
-- SERVIDOR REPLICA (DB2)

-- Parar replicação
STOP REPLICA;

-- Remover configuração de replicação completamente
-- Remove informações de master e reseta posição
RESET REPLICA ALL;
```

## Resumo das Ferramentas de Backup

| Ferramenta | Tipo | Hot Backup | Incremental | Velocidade | Uso Principal |
|------------|------|-----------|-------------|------------|---------------|
| **mysqldump** | Lógico | Sim (com --single-transaction) | Não | Lento | Backups pequenos, portabilidade |
| **mysqldump --tab** | Lógico | Sim | Não | Médio | Manipulação de dados, imports |
| **MyDumper/MyLoader** | Lógico | Sim | Não | Rápido (paralelo) | Databases grandes |
| **XtraBackup** | Físico | Sim | Sim | Muito rápido | Produção, backups grandes |
| **LVM Snapshot** | Físico | Sim | Não | Rápido | Ambientes virtualizados |
| **Cold Backup** | Físico | Não | Não | Muito rápido | Migração, clonagem |

## Boas Práticas

### Backup
1. **Estratégia 3-2-1**: 3 cópias, 2 mídias diferentes, 1 off-site
2. **Teste de restore regularmente**: backup não testado não é backup
3. **Monitore o tamanho dos backups**: espaço em disco é crítico
4. **Use compressão**: economiza espaço e largura de banda
5. **Automatize**: use cron jobs para backups regulares
6. **Documente**: mantenha runbooks de restore atualizados

### Replicação
1. **Monitore o lag de replicação**: use `SHOW REPLICA STATUS\G;`
2. **Configure alertas**: para quando replicação parar
3. **Use SSL**: proteja dados em trânsito
4. **Backups na replica**: não sobrecarrega master
5. **Teste failover**: simule falha do master periodicamente
6. **Mantenha binlogs por tempo suficiente**: permite recuperação de replicas atrasadas

# InnoDB Cluster – Alta Disponibilidade Automática

## 9. Conceitos do InnoDB Cluster

O **MySQL InnoDB Cluster** é uma solução de alta disponibilidade que combina:
- **Group Replication**: Replicação síncrona multi-master com detecção automática de falhas
- **MySQL Shell**: Ferramenta administrativa para gerenciar o cluster
- **MySQL Router**: Proxy inteligente que roteia conexões automaticamente

**Vantagens sobre replicação tradicional:**
- Failover automático (sem intervenção manual)
- Detecção automática de falhas
- Prevenção de split-brain
- Modo single-primary ou multi-primary
- Consistência garantida entre nós

## 10. Instalação e Configuração do Ambiente

### 10.1. Instalação do MySQL Shell

```bash
# Instalar MySQL Shell em todas as máquinas (db1, db2, db3)
# MySQL Shell: ferramenta para administrar e criar o InnoDB Cluster
apt-get install mysql-shell

# Conectar ao MySQL Shell como root (modo SQL)
mysqlsh -uroot -p

# Conectar ao MySQL Shell em modo JavaScript
# JS é usado para operações administrativas do cluster via API DBA
mysqlsh -uroot -p --js
```

### 10.2. Configuração de Resolução de Nomes

```bash
# Editar arquivo hosts para resolução de nomes local
# Evita necessidade de DNS configurado
nano /etc/hosts

# Adicionar mapeamento de IPs e hostnames
# Copiar essas linhas para /etc/hosts de TODAS as máquinas do cluster
172.27.11.10 db1 db1.local
172.27.11.20 db2 db2.local
172.27.11.30 db3 db3.local
172.27.11.40 haproxy haproxy.local
172.27.11.60 rhel-demo rhel-demo.local
```

## 11. Preparação das Instâncias MySQL

### 11.1. Configurar Primeira Instância (DB1)

```bash
# Conectar ao MySQL Shell em modo JavaScript
mysqlsh -uroot -p --js
```

```javascript
// Configurar instância local para participar do cluster
// Verifica e ajusta automaticamente:
// - Variáveis necessárias (server_id, binlog, gtid, etc)
// - Permissões de usuários
// - Configurações de Group Replication
dba.configureInstance('root@localhost:3306')

// Durante a execução, será apresentado um menu:
// [1] Create remotely usable account for 'root' with same grants and password
// [2] Create a new admin account for InnoDB Cluster with minimal required grants
// [3] Ignore and continue
// [4] Cancel

// ESCOLHA: Opção 2
// - Cria usuário administrador específico para o cluster
// - Nome do usuário: cadmin
// - Senha: 4linux

// Quando perguntado "Do you want to perform the required configuration changes?":
// Responda: y (yes)

// Quando perguntado "Do you want to restart the instance after configuring it?":
// Responda: y (yes) - MySQL será reiniciado automaticamente
```

## 12. Criação do Cluster

### 12.1. Criar Cluster no Nó Primário (DB1)

```bash
# Conectar com o usuário administrador do cluster
mysqlsh -ucadmin -p --js
```

```javascript
// Criar cluster com nome 'cluster_curso'
// Armazena referência do cluster na variável 'cluster'
var cluster = dba.createCluster('cluster_curso')

// Consultar status inicial do cluster
// Mostra: topologia, membros, modo (single/multi-primary), status
cluster.status();
```

## 13. Adicionar Membros ao Cluster

### 13.1. Preparar Segunda Instância (DB2)

```bash
# No servidor DB2, conectar ao MySQL Shell
mysqlsh -uroot -p --js
```

```javascript
// Configurar instância DB2 (mesmo processo do DB1)
dba.configureInstance('root@localhost:3306');
// Seguir mesmo procedimento: opção 2, criar cadmin, confirmar mudanças, reiniciar
```

### 13.2. Adicionar DB2 ao Cluster

```javascript
// No DB1 (onde o cluster foi criado), adicionar DB2
// cadmin@db2: usa usuário administrador no host db2
cluster.addInstance('cadmin@db2')

// Será perguntado sobre método de recuperação:
// [C] Clone: usa MySQL Clone Plugin (recomendado)
//     - Copia estado completo automaticamente
//     - Mais rápido e confiável
// [I] Incremental: usa binary logs
//     - Requer binlogs disponíveis
//     - Pode falhar se binlogs foram purgados

// ESCOLHA: C (Clone)

// Verificar se DB2 foi adicionado com sucesso
cluster.status()
// Deve mostrar DB2 como ONLINE
```

### 13.3. Adicionar Terceira Instância (DB3)

```bash
# No servidor DB3
mysqlsh -uroot -p --js
```

```javascript
// Configurar DB3
dba.configureInstance('root@localhost:3306');

// No DB1, adicionar DB3 ao cluster
cluster.addInstance('cadmin@db3')
// Escolher Clone (C)

// Verificar cluster completo
cluster.status()
// Deve mostrar 3 membros: db1, db2, db3
```

## 14. Gerenciamento do Cluster

### 14.1. Remover Instância

```javascript
// Remover um membro do cluster
// Útil para manutenção, upgrade ou remoção definitiva
cluster.removeInstance('cadmin@db2')

// Verificar remoção
cluster.status()
```

### 14.2. Reconectar ao Cluster Existente

```javascript
// Se desconectou do shell ou trocou de máquina
// Recuperar referência ao cluster existente
cluster = dba.getCluster()

// Verificar status atual
cluster.status()
```

### 14.3. Consultar Detalhes do Cluster

```javascript
// Status completo: mostra saúde, membros, replicação
cluster.status()

// Status detalhado: inclui informações de performance
cluster.status({extended: 1})

// Status com métricas de replicação
cluster.status({extended: 2})

// Descrever configuração do cluster
cluster.describe()
```

## 15. Modos de Operação do Cluster

### 15.1. Single-Primary Mode (Padrão)

```javascript
// Modo padrão: apenas um membro aceita escritas (PRIMARY)
// Outros membros são SECONDARY (somente leitura)
// Failover automático: se PRIMARY falha, um SECONDARY é promovido

// Definir manualmente qual instância será PRIMARY
cluster.setPrimaryInstance('db1:3306')

// Verificar mudança
cluster.status()
// PRIMARY: db1
// SECONDARY: db2, db3
```

### 15.2. Multi-Primary Mode

```javascript
// Modo avançado: TODOS os membros aceitam escritas
// Replicação síncrona entre todos os nós
// Útil para: distribuição de carga de escrita, aplicações distribuídas
// Cuidado: possibilidade de conflitos de escrita

// Alternar para modo multi-primary
cluster.switchToMultiPrimaryMode()

// Verificar mudança
cluster.status()
// Todos os membros aparecem como "R/W" (Read/Write)

// Voltar para single-primary (se necessário)
cluster.switchToSinglePrimaryMode()
// Ou especificar qual será o primary:
cluster.switchToSinglePrimaryMode('db1:3306')
```

## 16. MySQL Router – Proxy Inteligente

### 16.1. Instalação do Router

```bash
# Instalar MySQL Router em todas as máquinas
# Router: proxy que roteia conexões para membros do cluster
# - Detecta automaticamente PRIMARY e SECONDARY
# - Roteia escritas para PRIMARY
# - Balanceia leituras entre SECONDARY
# - Failover transparente para aplicação
apt-get install mysql-router

# Parar serviço padrão (será reconfigurado)
systemctl stop mysqlrouter.service
```

### 16.2. Bootstrap do Router

```bash
# Bootstrap: configura Router para conectar ao cluster
# --bootstrap: conecta ao cluster e descobre topologia
# --user: usuário do sistema operacional que rodará o Router

# Bootstrap do Router no DB1
mysqlrouter --bootstrap='cadmin@172.27.11.10' --user='mysqlrouter'

# Bootstrap do Router no DB2
mysqlrouter --bootstrap='cadmin@172.27.11.20' --user='mysqlrouter'

# Bootstrap do Router no DB3
mysqlrouter --bootstrap='cadmin@172.27.11.30' --user='mysqlrouter'

# Após bootstrap, o Router cria configuração automática em:
# /etc/mysqlrouter/mysqlrouter.conf

# Iniciar serviço do Router
systemctl start mysqlrouter.service

# Verificar status
systemctl status mysqlrouter.service

# Habilitar inicialização automática
systemctl enable mysqlrouter.service
```

### 16.3. Portas do MySQL Router

Após o bootstrap, o Router expõe portas para diferentes propósitos:

| Porta | Propósito | Roteamento |
|-------|-----------|-----------|
| **6446** | Read/Write (R/W) | Apenas PRIMARY - para escritas |
| **6447** | Read/Only (RO) | Round-robin entre SECONDARY - para leituras |
| **6448** | Read/Write (X Protocol) | PRIMARY via X Protocol |
| **6449** | Read/Only (X Protocol) | SECONDARY via X Protocol |

## 17. Configuração de Usuários para Aplicações

### 17.1. Criar Usuário da Aplicação

```sql
-- Conectar em qualquer membro do cluster (de preferência via Router)
-- Usuário criado será replicado automaticamente para todos os membros

-- Criar usuário permitindo conexão de qualquer host
CREATE USER aluno@'%' IDENTIFIED BY '4linux';

-- Permissões básicas: SELECT em todos os databases
GRANT SELECT ON *.* TO aluno@'%';

-- Permissão específica: UPDATE apenas na tabela employees.employees
GRANT UPDATE ON employees.employees TO aluno@'%';

-- Aplicar permissões
FLUSH PRIVILEGES;
```

### 17.2. Conectar Aplicação via Router

```bash
# Conexão para ESCRITA (porta 6446 - vai para PRIMARY)
mysql -ualuno -p4linux -h 172.27.11.10 -P 6446

# Conexão para LEITURA (porta 6447 - balanceada entre SECONDARY)
mysql -ualuno -p4linux -h 172.27.11.10 -P 6447

# Verificar em qual membro está conectado
SELECT @@hostname, @@port;
```

## 18. Teste de Failover Automático

### 18.1. Simular Falha do Primary

```bash
# No servidor PRIMARY (ex: db1)
# Parar MySQL forçadamente para simular falha
systemctl stop mysql

# Aguardar alguns segundos (detecção de falha)
sleep 10
```

```javascript
// Em outro membro do cluster (db2 ou db3)
mysqlsh -ucadmin -p --js

// Recuperar cluster
var cluster = dba.getCluster()

// Verificar status: um SECONDARY foi promovido a PRIMARY
cluster.status()

// Novo PRIMARY será eleito automaticamente
// Aplicações conectadas via Router continuam funcionando
```

### 18.2. Reintegrar Membro Recuperado

```bash
# No servidor que falhou (db1), iniciar MySQL
systemctl start mysql
```

```javascript
// O membro tentará se reintegrar automaticamente
// Verificar status
cluster.status()

// Se não reintegrar automaticamente:
cluster.rejoinInstance('cadmin@db1')

// db1 retorna como SECONDARY (em single-primary mode)
```

## 19. Monitoramento e Troubleshooting

### 19.1. Comandos de Diagnóstico

```javascript
// Status geral do cluster
cluster.status()

// Status detalhado com informações de performance
cluster.status({extended: 1})

// Verificar configuração do cluster
cluster.describe()

// Verificar quorum (maioria necessária para decisões)
cluster.status({extended: 2})

// Listar todos os routers conectados ao cluster
cluster.listRouters()

// Ver opções configuradas do cluster
cluster.options()
```

### 19.2. Logs Importantes

```bash
# Log do MySQL (erros de Group Replication)
tail -f /var/log/mysql/error.log

# Log do MySQL Router
tail -f /var/log/mysqlrouter/mysqlrouter.log

# Verificar status do Group Replication via SQL
mysql -uroot -p
SELECT * FROM performance_schema.replication_group_members;
```

### 19.3. Resolver Problemas Comuns

```javascript
// Cluster sem quorum (maioria dos membros offline)
// Forçar quorum no membro sobrevivente
var cluster = dba.getCluster()
cluster.forceQuorumUsingPartitionOf('cadmin@db2')

// Cluster inconsistente: rebootar do metadata
var cluster = dba.rebootClusterFromCompleteOutage()

// Remover metadata de membro morto
cluster.removeInstance('cadmin@db1', {force: true})
```



