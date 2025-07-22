# Comandos Realizados na Aula 05 – Gerenciamento de Usuários e Segurança

## 1. Transações Ativas e Verificação de Locks

```sql
START TRANSACTION;
```
> Inicia uma transação manualmente.

```sql
SELECT * FROM cientistas WHERE id = 5 FOR UPDATE;
```
> Seleciona o registro com `id = 5` na tabela `cientistas` e bloqueia a linha para atualização por outras sessões até o fim da transação.

```sql
SELECT * FROM performance_schema.data_locks\G;
```
> Lista todos os locks atualmente ativos no banco de dados, exibindo o resultado formatado verticalmente.

```sql
select * from performance_schema.data_locks\G;
SELECT DISTINCT
    dl.OBJECT_SCHEMA,
    dl.OBJECT_NAME,
    dl.LOCK_TYPE,
    dl.LOCK_MODE,
    dl.LOCK_DATA,
    t.PROCESSLIST_USER AS user,
    t.PROCESSLIST_HOST AS host,
    t.PROCESSLIST_ID AS thread_id,
    t.PROCESSLIST_INFO AS query
FROM
    performance_schema.data_locks dl
JOIN
    performance_schema.threads t ON dl.THREAD_ID = t.THREAD_ID;
SELECT
    rdl.OBJECT_SCHEMA AS waiting_schema,
    rdl.OBJECT_NAME AS waiting_table,
    rdl.INDEX_NAME AS waiting_index,
    rdl.LOCK_TYPE AS waiting_lock_type,
    rdl.LOCK_MODE AS waiting_lock_mode,
    rdl.LOCK_DATA AS waiting_row,
    wt.PROCESSLIST_ID AS waiting_process_id,
    wt.PROCESSLIST_USER AS waiting_user,
    wt.PROCESSLIST_HOST AS waiting_host,
    wt.PROCESSLIST_INFO AS waiting_query,
    bdl.OBJECT_SCHEMA AS blocking_schema,
    bdl.OBJECT_NAME AS blocking_table,
    bdl.INDEX_NAME AS blocking_index,
    bdl.LOCK_TYPE AS blocking_lock_type,
    bdl.LOCK_MODE AS blocking_lock_mode,
    bdl.LOCK_DATA AS blocking_row,
    bt.PROCESSLIST_ID AS blocking_process_id,
    bt.PROCESSLIST_USER AS blocking_user,
    bt.PROCESSLIST_HOST AS blocking_host,
    bt.PROCESSLIST_INFO AS blocking_query
FROM
    performance_schema.data_lock_waits dw
    JOIN performance_schema.data_locks rdl ON dw.REQUESTING_ENGINE_LOCK_ID = rdl.ENGINE_LOCK_ID
    JOIN performance_schema.data_locks bdl ON dw.BLOCKING_ENGINE_LOCK_ID = bdl.ENGINE_LOCK_ID
    JOIN performance_schema.threads wt ON rdl.THREAD_ID = wt.THREAD_ID
    JOIN performance_schema.threads bt ON bdl.THREAD_ID = bt.THREAD_ID\G;
```

> Consulta completa que mostra relações entre processos que estão esperando por locks e aqueles que estão bloqueando, útil para identificar deadlocks.


## 2. Permissões de Usuários e Segurança Granular

```sql
SHOW CREATE USER aluno;
```
> Mostra o comando necessário para recriar o usuário `aluno`, incluindo seus atributos e autenticação.

```sql
GRANT SELECT ON *.* TO `aluno`@`%`;
```
> Concede permissão de leitura em todas as tabelas e bancos ao usuário `aluno`.

```sql
GRANT INSERT, UPDATE ON ti.cientistas TO aluno@`%`;
```
> Permite que `aluno` insira e atualize registros na tabela `ti.cientistas`.

```sql
GRANT DELETE ON ti.cientistas TO aluno@`%`;
```
> Permite exclusão de registros na tabela `ti.cientistas`.

```sql
GRANT LOCK TABLES ON ti.* TO aluno@`%`;
```
> Permite que o usuário execute comandos de travamento de tabelas no banco `ti`.

```sql
LOCK TABLES cientistas READ;
```
> Trava a tabela `cientistas` para leitura apenas (bloqueando escrita por outros).

```sql
GRANT CREATE, ALTER ON ti.* TO aluno@`%`;
```
> Permite criar e alterar objetos (como tabelas) no banco `ti`.

```sql
ALTER TABLE cientistas ADD COLUMN site VARCHAR(100);
```
> Adiciona uma nova coluna `site` do tipo `VARCHAR(100)` à tabela `cientistas`.

```sql
REVOKE CREATE, ALTER ON ti.* FROM aluno@`%`;
```
> Remove as permissões de criação e alteração no banco `ti`.

```sql
REVOKE LOCK TABLES ON `ti`.* FROM `aluno`@`%`;
```
> Revoga a permissão de travar tabelas do banco `ti`.

```sql
REVOKE INSERT, UPDATE, DELETE ON `ti`.`cientistas` FROM `aluno`@`%`;
```
> Remove as permissões de inserção, atualização e exclusão na tabela `ti.cientistas`.

```sql
GRANT CREATE TEMPORARY TABLES ON ti.* TO aluno@`%`;
```
> Permite criação de tabelas temporárias no banco `ti`.



## 3. SSL e Tunelamento

```bash
cd /var/lib/mysql
ls -ll | grep pem
```
> Acessa o diretório de dados e lista os arquivos de certificados SSL (`.pem`).

```bash
nano /etc/mysql/my.cnf
```
> Abre o arquivo de configuração principal do MySQL.

```ini
require_secure_transport = ON
ssl-ca=/var/lib/mysql/ca.pem  
ssl-key=/var/lib/mysql/server-key.pem  
ssl-cert=/var/lib/mysql/server-cert.pem
```

> Ativa a exigência de SSL e define os caminhos dos certificados e chave do servidor.

```sql
CREATE USER criptografo@`%` REQUIRE X509;
```
> Cria um usuário que só pode se autenticar com certificado SSL válido (X.509).

```sql
GRANT SELECT ON ti.* TO criptografo@`%`;
```
> Concede permissão de leitura ao usuário `criptografo`.

```bash
mysql  -u  criptografo  --ssl-key=/var/lib/mysql/client-key.pem  --ssl-cert=/var/lib/mysql/client-cert.pem  --ssl-ca=/var/lib/mysql/ca.pem
```
> Conecta ao MySQL usando SSL com os certificados do cliente.

```bash
mkdir /root/ssl
cp /var/lib/mysql/client-*.pem /root/ssl/
cp /var/lib/mysql/ca.pem /root/ssl/
scp -r /root/ssl/* root@172.27.11.20:/root/ssl/
```
> Prepara e transfere os certificados SSL para outro servidor.

```bash
nano /etc/mysql/my.cnf
bind-address=127.0.0.1
```
> Restringe o MySQL a aceitar conexões apenas da máquina local (localhost).

```bash
mysql  -u  criptografo  -h  172.27.11.10  --ssl-key=/root/ssl/client-key.pem  --ssl-cert=/root/ssl/client-cert.pem  --ssl-ca=/root/ssl/ca.pem
```
> Conecta-se ao servidor DB1 (172.27.11.10), a partir do DB2 (172.27.11.20), utilizando autenticação por certificados SSL.

**TUNELAMENTO**
```bash
ssh -f -N -L 127.0.0.1:3307:127.0.0.1:3306 root@172.27.11.10
```
> Cria um túnel SSH da porta 3307 local para a 3306 remota, via `ssh`.

```bash
mysql -uroot -p -h 127.0.0.1 -P3307
```
> Conecta ao MySQL via túnel SSH.

```bash
ps -ef | grep ssh
kill 1223
```

> Verifica os processos `ssh` ativos e encerra o túnel pelo PID (neste caso, 1223).


## 4. Instalação de Plugins de Acesso

```bash
cd /usr/lib/mysql/plugin
```
> Acessa o diretório onde os plugins estão armazenados.

```sql
INSTALL PLUGIN auth_socket SONAME 'auth_socket.so';
```
> Instala o plugin `auth_socket` para autenticação via SO.

```sql
UNINSTALL PLUGIN auth_socket;
```
> Remove o plugin `auth_socket`.

```ini
plugin-load=auth_socket.so
```
> Linha no `my.cnf` que carrega o plugin `auth_socket` no início.

```sql
CREATE USER vagrant@localhost IDENTIFIED WITH auth_socket;
```

> Cria um usuário que se autentica automaticamente pelo sistema.

```sql
GRANT SELECT ON ti.* TO vagrant@localhost;
```

> Permite que `vagrant` consulte todas as tabelas do banco `ti`.

```sql
INSTALL PLUGIN validate_password SONAME 'validate_password.so';
SHOW VARIABLES LIKE '%validate_password%';
```
> Instala e verifica variáveis do plugin que valida a força da senha.

```sql
CREATE USER teste@localhost IDENTIFIED BY 'Senh@F0rt3';
```
> Cria um usuário com senha forte, compatível com a política de validação.

```sql
SET GLOBAL validate_password.policy = 2;
SET GLOBAL validate_password_length = 5;
SET GLOBAL validate_password_mixed_case_count = 0;
SET GLOBAL validate_password_number_count = 0;
SET GLOBAL validate_password_special_char_count = 0;

```
> Configura a política de senha para exigência mínima de segurança.

```sql
UNINSTALL PLUGIN validate_password;
```
> Remove o plugin de validação de senha.

----------

## 5. Consultas Lentas e Otimização

```ini
slow-query-log = ON
long-query-time = 0.2
slow-query-log-file = /var/log/mysql/slow.log
```
> Configura o MySQL para registrar queries que demoram mais que 0.2 segundos.

```bash
mysqldumpslow -s 'c' -t 5 /var/log/mysql/slow.log
```
> Mostra as 5 queries mais frequentes.

```bash
mysqldumpslow -s 't' -t 5 /var/log/mysql/slow.log
```
> Mostra as 5 queries mais lentas por tempo.

```bash
mysqldumpslow -s 'r' -t 5 /var/log/mysql/slow.log
```
> Mostra as 5 queries que retornaram mais linhas.

