# Aula 09 – Backup no MySQL: mysqldump, mydumper, rsync/scp e xtrabackup

### 1. Backup Lógico com mysqldump

```bash
# Backup apenas dos dados (sem comandos CREATE DATABASE)
mysqldump -uroot -p employees > employees.sql

# Backup da base com CREATE DATABASE incluído
mysqldump -uroot -p --databases employees > employees.sql

# Backup somente da estrutura (sem dados)
mysqldump -uroot -p --databases employees --no-data > employees_estrutura.sql

# Backup somente dos dados (sem estrutura)
mysqldump -uroot -p --databases employees --no-create-info > employees_dados.sql

# Backup com lock nas tabelas da sessão (impede gravações enquanto faz o dump)
mysqldump -uroot -p --databases employees --lock-tables > employees.sql

# Lock em todas as tabelas, inclusive para leitura
mysqldump -uroot -p --databases employees --lock-all-tables > employees.sql

# Backup compactado usando gzip
mysqldump -uroot -p --databases employees | gzip > employees-$(date +%Y-%m-%d).sql.gz

# Compactando com pigz (paralelo, mais rápido que gzip tradicional)
mysqldump -uroot -p --databases employees | pigz -9 > employees2-$(date +%Y-%m-%d).sql.gz

# Descompactar backup gzip para arquivo
zcat employees-2025-07-31.sql.gz > backup.sql

# Restauração de backup compactado direto para o MySQL
zcat employees-2025-07-31.sql.gz | mysql -uroot -p

# Backup de tabela específica
mysqldump -uroot -p employees salaries > tabela_salaries.sql
```

### 2. Restauração com mysql

```bash
# Restauração da estrutura
mysql -uroot -p  funcao_triggers.sql

# Exportar views manualmente
mysql -uroot -p  views.sql
```

### 6. Restauração com myloader

```bash
myloader -u root -p "4linux" -d /root/backup_exemplo/bkp_sakila/ --threads=5 --queries-per-transaction=10000 --verbose=3
```

### 7. Backup a frio com rsync e scp

```bash
# No servidor de destino
systemctl stop mysql
rm -rf /var/lib/mysql/*

# Copia os arquivos do banco do servidor 1 para 2 via scp
scp -r /var/lib/mysql/* root@172.27.11.20:/var/lib/mysql/

# Ajusta permissões no destino
chown -R mysql: /var/lib/mysql
systemctl start mysql

# Alternativa com rsync para sincronização eficiente
systemctl stop mysql
rm -rf /var/lib/mysql

rsync -av /var/lib/mysql root@172.27.11.20:/var/lib/
```

### 8. Instalação e Configuração do xtrabackup

```bash
dpkg -i percona-release_latest.bookworm_all.deb
apt-get install -f

# Criar usuário para backup
CREATE USER xtrabackup@localhost IDENTIFIED BY 'hot-backup';

GRANT BACKUP_ADMIN, PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO xtrabackup@localhost;
GRANT SELECT ON performance_schema.* TO xtrabackup@localhost;
```

### 9. Backup com xtrabackup

```bash
# Backup completo
xtrabackup --user='xtrabackup' --password='hot-backup' --backup --target-dir='/backup/full'

# Usando login path para armazenar credenciais
mysql_config_editor set --host=localhost --user=xtrabackup --password --login-path=xtrabackup
xtrabackup --login-path=xtrabackup --backup --target-dir='/backup/full'

# Preparar o backup para aplicação
xtrabackup --prepare --target-dir=/backup/full/

# Restaurar o backup
xtrabackup --copy-back --target-dir=/backup/full/

# Ajustar permissões pós-restauração
chown -R mysql: /var/lib/mysql/
systemctl start mysql
```

### 10. Backups incrementais com xtrabackup

```bash
# Backup incremental baseado no backup completo
xtrabackup --backup --target-dir=/backup/inc1 --incremental-basedir=/backup/full

# Preparar backup incremental para aplicar no backup completo
xtrabackup --prepare --apply-log-only --target-dir=/backup/full --incremental-dir=/backup/inc1

# Inserir dados para simulação (exemplo)
for X in $(seq 1 500); do mysql curso -e "INSERT INTO seeds (seed) VALUES ('$RANDOM')"; done

# Aplicar segundo backup incremental
xtrabackup --prepare --apply-log-only --target-dir=/backup/full --incremental-dir=/backup/inc2
```

### 11. Automação de Backup com Script Bash (Exemplo)

```bash
#!/bin/bash

BACKUP_DIR="/backups/full"
CURRENT_DATE=$(date +"%Y-%m-%d")
ULTIMOS_7_DIAS=$(date -d "6 days ago" +"%Y-%m-%d")
DAILY_BACKUP_DIR="${BACKUP_DIR}/${CURRENT_DATE}"

mkdir -p "$DAILY_BACKUP_DIR"

xtrabackup --login-path=xtrabackup --backup --target-dir="$DAILY_BACKUP_DIR" > "${DAILY_BACKUP_DIR}.log" 2>&1

# Limpar backups com mais de 7 dias
for dir in "$BACKUP_DIR"/*; do
  dir_name=$(basename "$dir")
  if [[ "$dir_name" < "$ULTIMOS_7_DIAS" ]]; then
    rm -rf "$dir"
  fi
done
```