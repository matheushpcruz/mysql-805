# Comandos Realizados na Aula 04 – Transações

### 1. Storage Engines e Online DDL

```sql
ALTER TABLE produtos ADD COLUMN estoque INT DEFAULT 0, ALGORITHM=INPLACE, LOCK=NONE;
```
> Adiciona a coluna `estoque` com valor padrão 0 à tabela `produtos` sem bloquear leitura nem escrita.

```sql
ALTER TABLE produtos ADD COLUMN categoria VARCHAR(50), ALGORITHM=INPLACE, LOCK=SHARED;
```
> Adiciona a coluna `categoria` e permite leituras durante a alteração, mas bloqueia escritas.

```sql
ALTER TABLE produtos ADD COLUMN preco NUMERIC(15,2), ALGORITHM=INPLACE, LOCK=SHARED;
```
> Adiciona a coluna `preco` com precisão decimal; permite leituras durante a operação.

```sql
ALTER TABLE produtos ROW_FORMAT=compact, ALGORITHM=inplace, LOCK=none;
```
> Altera o formato de linha da tabela para `COMPACT`, sem bloquear leitura nem escrita.

```sql
ALTER TABLE produtos ADD KEY idx_nome(nome), ALGORITHM=inplace, LOCK=exclusive;
```
> Cria um índice na coluna `nome`, usando operação inplace. Bloqueia leitura e escrita durante a criação do índice.

```sql
ALTER TABLE produtos DROP COLUMN id;
```
> Remove a coluna `id` da tabela `produtos`.

```sql
ALTER TABLE produtos ADD COLUMN id INT AUTO_INCREMENT PRIMARY KEY;
```
> Adiciona uma nova coluna `id` como chave primária com incremento automático.


### 2. Formatos de Linha do InnoDB

```sql
CREATE DATABASE row_format;
```
> Cria um banco de dados chamado `row_format`.

```sql
USE row_format;
```
> Define que os próximos comandos serão executados dentro do banco `row_format`.

```sql
CREATE TABLE compacta(id INT AUTO_INCREMENT PRIMARY KEY, dados TEXT) ENGINE=InnoDB ROW_FORMAT=COMPACT;
```
> Cria uma tabela `compacta` usando o formato de linha `COMPACT`.

```sql
CREATE TABLE dinamica(id INT AUTO_INCREMENT PRIMARY KEY, dados TEXT) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;
```
> Cria uma tabela `dinamica` usando o formato `DYNAMIC`, que armazena grandes campos fora da página principal.

```sql
CREATE TABLE comprimida(id INT AUTO_INCREMENT PRIMARY KEY, dados TEXT) ENGINE=InnoDB ROW_FORMAT=COMPRESSED;
```
> Cria uma tabela `comprimida` usando compressão para reduzir o espaço em disco.

```sql
INSERT INTO compacta(dados) SELECT REPEAT('A', 10000);
INSERT INTO dinamica(dados) SELECT REPEAT('A', 10000);
INSERT INTO comprimida(dados) SELECT REPEAT('A', 10000);
```
> Insere uma string de 10.000 caracteres para testar o uso de espaço em cada formato.

```sql
SELECT table_schema, table_name, ROUND(data_length/1024/1024,2) AS data_mb,
       ROUND(index_length/1024/1024,2) AS index_mb, row_format
FROM information_schema.tables
WHERE table_name IN ('compacta','dinamica','comprimida');
```
> Consulta informações de tamanho e formato das tabelas criadas.

```sql
SHOW TABLE STATUS FROM row_format;
```
> Mostra o status geral de todas as tabelas no banco `row_format`.

```sql
SHOW TABLE STATUS FROM row_format WHERE Name IN ('compacta','dinamica','comprimida')\G;
```
> Mostra detalhes das tabelas selecionadas, formatando a saída em colunas verticais.


###  **No Shell Linux**

```bash
du -hsc /var/lib/mysql/row_format/*
```
> Mostra o uso de disco de cada tabela armazenada fisicamente no diretório do MySQL.

```bash
cp /var/lib/mysql/row_format/compacta.ibd /tmp/compacta.ibd
```
> Copia o arquivo de dados da tabela `compacta` para `/tmp`.

```bash
innochecksum --verbose /tmp/compacta.ibd --log '/dev/stdout'
```
> Verifica a integridade do arquivo `.ibd` da tabela `compacta`.

----------

###  3. Transações e Locks

```sql
BEGIN;  -- ou START TRANSACTION;
```
> Inicia uma transação manualmente.

```sql
COMMIT;
```
> Finaliza a transação e aplica as alterações.

```sql
ROLLBACK;
```
> Cancela a transação e desfaz as alterações feitas.

```sql
SET TRANSACTION ISOLATION LEVEL ...;
```
> Define o nível de isolamento da transação (ex: `READ UNCOMMITTED`, `READ COMMITTED`).

```sql
SHOW VARIABLES LIKE '%transaction_isolation%';
```
> Exibe o nível de isolamento de transações configurado no servidor.

```sql
LOCK TABLE cientistas READ;
```
> Bloqueia a tabela `cientistas` para leitura; impede escrita por outras sessões.

```sql
LOCK TABLE cientistas WRITE;
```
> Bloqueia a tabela para escrita; impede leituras e escritas por outras sessões.

```sql
UNLOCK TABLES;
```
> Libera todos os locks aplicados manualmente.

```sql
FLUSH TABLES WITH READ LOCK;
```
> Força o MySQL a gravar todos os dados em disco e bloqueia para leitura (usado em backup).

```sql
SELECT * FROM cientistas WHERE id = 2 FOR UPDATE;
```
> Seleciona a linha com `id = 2` e bloqueia para escrita por outras transações até `COMMIT` ou `ROLLBACK`.

```sql
SELECT * FROM cientistas WHERE id = 2 FOR SHARE;
```
> Seleciona a linha com `id = 2` e bloqueia apenas contra escrita, permitindo outras leituras.

```sql
SELECT * FROM information_schema.innodb_trx;
```
> Mostra informações sobre as transações InnoDB atualmente em execução.


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



### 4. Usuários e Segurança

```sql
CREATE USER observador@'%' IDENTIFIED BY 'viewer';
```
> Cria um usuário `observador` com acesso de qualquer host (`%`), senha `viewer`.

```sql
CREATE USER observador@'localhost' IDENTIFIED BY 'viewer';
```
> Cria o mesmo usuário, mas restrito ao host local.

```sql
CREATE USER observador@'172.27.11.1' IDENTIFIED BY 'viewer';
```
> Cria o mesmo usuário, mas acessível apenas a partir do IP `172.27.11.1`.

```sql
GRANT SELECT ON *.* TO observador@localhost;
```
> Concede permissão de leitura em todas as bases/tabelas para o usuário local.

```sql
GRANT SELECT ON ti.* TO observador@'172.27.11.1';
```
> Concede leitura apenas no banco `ti` para o IP específico.

```sql
SHOW GRANTS FOR observador@localhost;
```

> Exibe todas as permissões atuais do usuário `observador` local.

```sql
REVOKE SELECT ON *.* FROM observador@localhost;
```
> Revoga o privilégio de leitura em todas as bases do usuário local.

```sql
CREATE ROLE alunos;
```
> Cria uma role chamada `alunos`, usada para agrupar permissões.

```sql
GRANT SELECT ON ti.cientistas TO alunos;
```
> Concede permissão de leitura na tabela `cientistas` para a role `alunos`.

```sql
GRANT SELECT ON ti.reconhecimentos TO alunos;
```
> Concede leitura na tabela `reconhecimentos` para a role `alunos`.

```sql
SET ROLE alunos;
```
> Ativa temporariamente a role `alunos` na sessão atual.

```sql
SET DEFAULT ROLE alunos TO 'observador'@'172.27.11.1';
```
> Define que a role `alunos` será automaticamente ativada para o usuário especificado ao se conectar.


## 5. SSL e Tunelamento

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