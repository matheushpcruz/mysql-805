# Aula 11 - Replicação

# 1. Replicação via Binlog

### **No DB1 (Master)**

**Criar usuário de replicação**

```sql
CREATE USER 'repl'@"172.27.11.20" IDENTIFIED BY '4linux'; -- Usuário usado pelo DB2
```

**Criar base e tabela para teste**

```sql
CREATE DATABASE replicacao;
USE replicacao;
CREATE TABLE replica_teste (
    id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    texto VARCHAR(100)
);
INSERT INTO replica_teste (texto) VALUES ('Este é um dado replicado');
```

**Backup com XtraBackup**

```bash
xtrabackup --login-path=xtrabackup --backup --target-dir=/backup/backup_replica
xtrabackup --login-path=xtrabackup --target-dir=/backup/backup_replica --prepare
```

**Configuração no `/etc/mysql/my.cnf` (Master)**

```ini
server_id = 1
log-bin   = /var/lib/mysql/mysql-bin
```

---

### **No DB2 (Slave)**

**Configuração no `/etc/mysql/my.cnf` (Slave)**

```ini
server_id = 2
relay-log = /var/lib/mysql/relay-bin
```

**Preparar o ambiente para receber o backup**

```bash
systemctl stop mysql
rm -rf /var/lib/mysql/*
```

**Copiar o backup do DB1 para o DB2**

```bash
scp -r /backup/backup_replica/* root@172.27.11.20:/var/lib/mysql/
chown -R mysql: /var/lib/mysql
systemctl restart mysql
```

**Obter informações do binlog do backup**

```bash
cat xtrabackup_binlog_info
cat /var/lib/mysql/xtrabackup_binlog_info
```

**Configurar replicação no Slave**

```sql
RESET REPLICA ALL; -- Limpa configuração anterior

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='172.27.11.10',
  SOURCE_LOG_FILE='mysql-bin.000001', -- Obtido do xtrabackup_binlog_info
  SOURCE_LOG_POS=607,                 -- Obtido do xtrabackup_binlog_info
  SOURCE_USER='repl',
  SOURCE_PASSWORD='4linux';

START REPLICA USER='repl' PASSWORD='4linux';
SHOW REPLICA STATUS\G; -- Confere o status da replicação
```

---

## 2. Replicação com GTID

### **Configuração no DB1 (Master)**

```ini
[mysqld]
server-id=1
gtid_mode=ON
enforce-gtid-consistency=ON
log-bin=mysql-bin
binlog-format=ROW
binlog_expire_logs_seconds=604800
```

### **Configuração no DB2 (Slave)**

```ini
[mysqld]
server-id=2
gtid_mode=ON
enforce-gtid-consistency=ON
relay_log=relay-bin
log-bin=mysql-bin
binlog-format=ROW
read_only=ON
```


**Backup com XtraBackup**

```bash
xtrabackup --login-path=xtrabackup --backup --target-dir=/backup/backup_replica
xtrabackup --login-path=xtrabackup --target-dir=/backup/backup_replica --prepare
```

**Preparar o ambiente para receber o backup**

```bash
systemctl stop mysql
rm -rf /var/lib/mysql/*
```

**Copiar o backup do DB1 para o DB2**

```bash
scp -r /backup/backup_replica/* root@172.27.11.20:/var/lib/mysql/
chown -R mysql: /var/lib/mysql
systemctl restart mysql
```

**Obter informações do binlog do backup**

```bash
cat xtrabackup_binlog_info
cat /var/lib/mysql/xtrabackup_binlog_info
```

**Configurar replicação no Slave**

```sql
RESET REPLICA ALL; -- Limpa configuração anterior
```

**Configurar replicação usando GTID (Slave)**

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='172.27.11.10',
  SOURCE_PORT=3306,
  SOURCE_AUTO_POSITION=1;

START REPLICA USER='repl' PASSWORD='4linux';
SHOW REPLICA STATUS\G; -- Confere o status da replicação
```