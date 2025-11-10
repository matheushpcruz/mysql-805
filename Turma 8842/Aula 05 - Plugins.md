
## 1. Instalação de Plugins de Acesso

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

## 2. Consultas Lentas e Otimização

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


### 3. Importação de base de testes

```bash
# Instala o git
apt-get install git

# Clona bases de demonstração
git clone https://github.com/datacharmer/test_db.git

# Importa a base employees
mysql -uroot -p < test_db/employees.sql

# Importa a base sakila – schema e dados
cd sakila/
mysql -uroot -p < sakila-mv-schema.sql
mysql -uroot -p < sakila-mv-data.sql
```

### 4. Comandos SHOW e Inspeção de Objetos

```sql
-- Lista bancos de dados
SHOW DATABASES;

-- Mostra o comando de criação de um banco
SHOW CREATE DATABASE employees;

-- Lista procedures e mostra criação de procedure
SHOW PROCEDURE STATUS;
SHOW CREATE PROCEDURE sakila.rewards_report;

-- Lista funções e mostra criação de função
SHOW FUNCTION STATUS;
SHOW CREATE FUNCTION nome_funcao;

-- Lista triggers e mostra criação de um trigger específico
SHOW TRIGGERS FROM sakila\G;
SHOW CREATE TRIGGER sakila.payment_date;

-- Detalhe das tabelas e índices
SHOW TABLE STATUS FROM employees\G;
SHOW INDEXES FROM employees.departments;

-- Visualiza permissões e usuários
SHOW GRANTS FOR root@localhost\G;
SHOW GRANTS FOR aluno@'%'\G;
SHOW CREATE USER root@localhost;

-- Trabalha com views
SHOW CREATE VIEW sakila.staff_list;
SELECT * FROM information_schema.views;
```

### 5. Logs, Variáveis, Status e Diagnóstico

```sql
-- Lista arquivos de log binário, eventos e conexões
SHOW BINARY LOGS;
SHOW BINLOG EVENTS IN 'binlog.000033' LIMIT 50;
SHOW FULL PROCESSLIST;

-- Exibe e filtra variáveis do servidor
SHOW VARIABLES\G;
SHOW VARIABLES LIKE '%max_con%'\G;

-- Operações administrativas temporárias
SET sql_log_bin = OFF;           -- Desativa log binário nesta sessão (grandes cargas)
SET foreign_key_checks = OFF;    -- Desativa checagem de chave estrangeira (importações)
```

