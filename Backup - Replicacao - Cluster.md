**A aula do dia 15/08 abordou os capítulos sobre Backup, Replicação e Cluster.**

### Abaixo, alguns comandos utilizados para backup:

**Instalar o XtraBackup:**

1. Seguir as instruções na documentação oficial:  
   [Percona XtraBackup 8.0 - APT Repository](https://docs.percona.com/percona-xtrabackup/8.0/apt-repo.html)

2. Baixar o pacote do repositório Percona:
   ```bash
   wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb
   ```

3. Instalar a dependência `curl`:
   ```bash
   apt-get install curl
   ```

4. Instalar o pacote baixado:
   ```bash
   dpkg -i percona-release_latest.$(lsb_release -sc)_all.deb
   ```

5. Habilitar o repositório:
   ```bash
   percona-release enable-only tools release
   ```

6. Atualizar os pacotes e instalar o XtraBackup:
   ```bash
   apt update && apt install percona-xtrabackup-80
   ```

**Criar o usuário do XtraBackup no banco de dados:**

```sql
CREATE USER 'xtrabackup'@'localhost' IDENTIFIED BY 'aluno123';
GRANT BACKUP_ADMIN, PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'xtrabackup'@'localhost';
GRANT SELECT ON performance_schema.log_status TO 'xtrabackup'@'localhost';
GRANT SELECT ON performance_schema.keyring_component_status TO 'xtrabackup'@'localhost';
GRANT SELECT ON performance_schema.replication_group_members to xtrabackup@localhost;
```

**Criar um diretório de backup (por exemplo, `/backup/`):**

```bash
mkdir /backup/
```

**Executar backup completo:**

```bash
xtrabackup --user='xtrabackup' --password='aluno123' --backup --target-dir=/backups/full
```

**Backup incremental:**

```bash
xtrabackup --user='xtrabackup' --password='aluno123' --backup --target-dir='/backups/incremental' --incremental-basedir='/backups/full'
```

**Restore do backup completo:**

1. Preparar o backup:
   ```bash
   xtrabackup --prepare --target-dir='/backups/full'
   ```

2. Parar o serviço MySQL:
   ```bash
   systemctl stop mysql
   ```

3. Limpar o diretório de dados:
   ```bash
   rm -rf /var/lib/mysql/*
   ```

4. Restaurar o backup:
   ```bash
   xtrabackup --copy-back --target-dir='/backups/full' --datadir=/var/lib/mysql/
   ```

5. Ajustar as permissões:
   ```bash
   chown -R mysql:mysql /var/lib/mysql
   ```

6. Iniciar o serviço MySQL:
   ```bash
   systemctl start mysql
   ```

**Restore de backup incremental:**

1. Preparar o backup incremental:
   ```bash
   xtrabackup --prepare --target-dir='/backups/full' --incremental-dir='/backups/incremental'
   ```

2. Parar o serviço MySQL:
   ```bash
   systemctl stop mysql
   ```

3. Limpar o diretório de dados:
   ```bash
   rm -rf /var/lib/mysql/*
   ```

4. Restaurar o backup:
   ```bash
   xtrabackup --copy-back --target-dir='/backups/full' --datadir=/var/lib/mysql/
   ```

5. Ajustar as permissões:
   ```bash
   chown -R mysql:mysql /var/lib/mysql
   ```

6. Iniciar o serviço MySQL:
   ```bash
   systemctl start mysql
   ```



### Alguns comandos utilizados para replicação como exemplo:

1. Alterar o arquivo `/etc/mysql/my.cnf`:
   - `server_id=1` para o servidor master.
   - `binlog_expire_logs_seconds=604800` para definir o tempo em segundos para expirar o log (neste caso, 7 dias) **ou** `expire_logs_days=7` para especificar a quantidade de dias do binlog.

2. No servidor master, criar o usuário de replicação:
   ```sql
   CREATE USER replicador@'172.27.11.20';
   GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO replicador@'172.27.11.20';
   SHOW MASTER STATUS;
   ```

3. No servidor slave:
   - Faça um backup do diretório `/var/lib/mysql` **ou**, se já tiver um backup, apenas limpe o diretório com: `rm -rf /var/lib/mysql`.
   - No servidor master, copie os arquivos via SCP para o servidor slave:
     ```bash
     scp -r /var/lib/mysql/* root@172.27.11.20:/var/lib/mysql/
     ```

4. No arquivo de configuração do servidor slave (`/etc/mysql/my.cnf`):
   - Ajuste o ID do servidor e ative o modo somente leitura (`read_only`):
     ```
     [mysqld]
     server_id=2
     read_only=1
     ```
   - Ajuste as permissões do diretório MySQL:
     ```bash
     chown mysql:mysql -R /var/lib/mysql/
     ```
   - Ajuste ou remova o arquivo `auto.cnf` no servidor slave:
     ```bash
     mv /var/lib/mysql/auto.cnf /var/lib/mysql/auto.cnf.bkp
     ```

5. Subir o serviço do MySQL em ambos os servidores (master e slave):
   ```bash
   systemctl start mysql
   ```

6. No servidor master, desbloquear as tabelas:
   ```sql
   UNLOCK TABLES;
   ```

7. No servidor slave, configurar o servidor master:
   ```sql
   CHANGE MASTER TO 
   MASTER_HOST='172.27.11.10',
   MASTER_PORT=3306,
   MASTER_USER='replicador',
   MASTER_PASSWORD='aluno123',
   MASTER_LOG_FILE='binlog.000035',
   MASTER_LOG_POS=850;
   ```

   - `MASTER_LOG_FILE`: Último binlog recebido.
   - `MASTER_LOG_POS`: Posição do último binlog.

8. Iniciar a replicação:
   ```sql
   START SLAVE;
   ```



## **Comandos utilizados para Cluster:**

1. **Preparação para inicializar o cluster:**
   - Reinstalar o MySQL nos servidores `db2` e `db3`.
   - Remover todos os bancos de dados ou tabelas no `db1` que não possuem chave primária.

2. **Editar o arquivo `/etc/hosts`** em cada servidor (`db1`, `db2`, `db3`, `haproxy`):
   ```
   # Configuração da rede:
   172.27.11.10    db1     db1.local
   172.27.11.20    db2     db2.local
   172.27.11.30    db3     db3.local
   172.27.11.40    haproxy haproxy.local
   ```

3. **Instalar o MySQL Shell** nos três bancos de dados (`db1`, `db2`, `db3`):
   ```bash
   apt-get install mysql-shell
   ```

4. **No servidor HAProxy**, instalar o MySQL Router:
   ```bash
   apt-get install mysql-router
   ```

5. **No `db1`, acessar o MySQL Shell** da seguinte forma:
   ```bash
   mysqlsh
   ```

6. **Configurar uma instância local**:
   ```js
   dba.configureLocalInstance("root@localhost:3306");
   ```

7. **Criar um usuário administrador do cluster**.

8. **Conectar ao cluster**:
   ```js
   shell.connect('clusteradmin@db1:3306');
   ```

9. **Criar o cluster**:
   ```js
   cluster = dba.createCluster('ha');
   ```

10. **Verificar a configuração** utilizando o comando:
    ```js
    cluster.status();
    ```

11. **No `db2` e `db3`**, acessar o MySQL Shell, configurar uma instância local e criar o mesmo usuário criado no `db1` para que se comuniquem:
    ```bash
    mysqlsh
    dba.configureLocalInstance("root@localhost:3306");
    ```

12. **No `db1`, acessar o MySQL Shell novamente**:
    ```bash
    mysqlsh
    ```

13. **Conectar na instância**:
    ```js
    shell.connect('clusteradmin@db1:3306');
    ```

14. **Obter o cluster**:
    ```js
    cluster = dba.getCluster();
    ```

15. **Adicionar as outras instâncias**:
    ```js
    cluster.addInstance('clusteradmin@db2:3306'); 
    cluster.addInstance('clusteradmin@db3:3306');
    ```

16. **Na etapa de seleção do método de recuperação**, escolha "C" para clonar os ambientes e realizar a réplica:
    ```
    Please select a recovery method [C]lone/[I]ncremental recovery/[A]bort (default Clone): C
    ```

**Comandos para o MySQL Router:**

1. **No servidor HAProxy (`172.27.11.40`)**:
   ```bash
   apt-get -y install mysql-router
   ```

2. **Configurar a conexão com o cluster admin**:
   ```bash
   mysqlrouter --bootstrap=clusteradmin@db1:3306 --directory myrouter --user=root
   ```

3. **Após a configuração do MySQL Router**:
   ```bash
   # MySQL Router configured for the InnoDB Cluster 'ha'

   After this MySQL Router has been started with the generated configuration

       $ mysqlrouter -c /root/myrouter/mysqlrouter.conf

   InnoDB Cluster 'ha' can be reached by connecting to:

   ## MySQL Classic protocol

   - Read/Write Connections: localhost:6446
   - Read/Only Connections:  localhost:6447

   ## MySQL X protocol

   - Read/Write Connections: localhost:6448
   - Read/Only Connections:  localhost:6449
   ```

4. **Executar o script de inicialização do MySQL Router**:
   ```bash
   myrouter/start.sh
   ```

5. **Após isso, já será possível acessar o cluster** utilizando o MySQL Router nas portas `6446` (Read/Write) e `6447` (Read-Only).
