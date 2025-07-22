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
> Força o MySQL a gravar todos os dados em disco e bloqueia para leitura — usado em backup.

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