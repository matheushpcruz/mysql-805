# Comandos Realizados na Aula 03 – Múltiplas Instâncias

## 1. Arquitetura Física e Lógica do MySQL

### Diretórios do MySQL, do Sistema Linux e Arquivos de Configuração

-   `/etc/`  
    > Diretório principal que contém os arquivos de configuração do sistema.
    
-   `/etc/my.cnf`  
    > Arquivo principal de configuração do MySQL, que pode ser um link simbólico para outro arquivo.
    
-   `/etc/mysql/my.cnf`  
    > Arquivo alternativo de configuração do MySQL.
    
-   `/etc/mysql/mysql.conf.d/mysqld.cnf`  
    > Arquivo com configurações específicas para o processo `mysqld`.
    
-   `~/.my.cnf`  
    > Arquivo de configuração pessoal do usuário para o cliente MySQL.
    
-   `~/.mylogin.cnf`  
    > Arquivo criptografado utilizado para armazenamento seguro das credenciais de acesso.
    
-   `/home/`  
    > Diretórios pessoais dos usuários do sistema.
    
-   `/sbin/`  
    > Binários administrativos essenciais ao sistema.
    
-   `/tmp/`  
    > Diretório destinado a arquivos temporários.
    
-   `/usr/`  
    > Diretório que contém programas e bibliotecas do sistema.
    
-   `/usr/share/mysql-8.4`  
    > Diretório com arquivos de fontes e recursos do MySQL versão 8.4.
    
-   `/var/`  
    > Diretórios para dados variáveis, incluindo logs e bancos de dados.
    
-   `/var/lib/mysql`  
    > Local padrão para armazenamento dos dados do MySQL.
    
-   `/var/log/`  
    > Diretório onde são armazenados os arquivos de log do sistema e do MySQL.
    

**Hierarquia para busca do arquivo de configuração do MySQL:**  
A aplicação busca os arquivos nesta ordem: `/etc/mysql/`, seguido pelos diretórios `conf.d/` e `mysql.conf.d`.


## 2. Criação e Gerenciamento de Múltiplas Instâncias MySQL

```bash
mkdir /tmp/mysql/
mkdir /tmp/mysql2
```
> São criados diretórios temporários para armazenamento dos dados das duas instâncias MySQL que serão configuradas.

```bash
mysqld --initialize --datadir='/tmp/mysql/' --user='mysql' --log-error='/tmp/mysql/error.log'
```

> Inicializa o diretório de dados da primeira instância, criando os arquivos necessários e gerando uma senha temporária.
> O processo é executado com o usuário do sistema `mysql`, e os erros são registrados no arquivo especificado.

```bash
mysqld --initialize --datadir='/tmp/mysql2' --user='mysql' --log-error='/tmp/mysql2/error.log'
```

> Realiza a mesma inicialização para a segunda instância MySQL.

```bash
mysqld --daemonize --datadir='/tmp/mysql/' --user='mysql' --port='3307' --socket='/tmp/mysql/mysql.sock' --pid-file='/tmp/mysql/mysql.pid' --log-error
```
> Inicia a primeira instância do MySQL em modo daemon (executando em segundo plano), definindo diretório de dados, usuário do sistema, porta customizada (3307), arquivos de socket e PID personalizados, e arquivo para logs de erro.

```bash
nano /tmp/mysql/my.cnf
nano /etc/mysql/mysql.conf.d/mysqld.cnf
nano /etc/systemd/system/mysql-tmp.service
```

> Edita os arquivos de configuração específicos para as instâncias e o serviço systemd para gerenciar o ciclo de vida do servidor MySQL temporário.

```ini
[mysqld]
daemonize
user=mysql
port=3307
datadir=/tmp/mysql
log-error=/tmp/mysql/error.log
socket=/tmp/mysql/mysql.sock
pid-file=/tmp/mysql/mysql.pid
```

```bash
mysqld --defaults-file=/tmp/mysql/my.cnf
```

> Inicia o servidor MySQL carregando as configurações do arquivo personalizado informado.


```ini
[mysqld2]
user            = mysql
port            = 3307
datadir         = /tmp/mysql
log-error       = /tmp/mysql/error.log
socket          = /tmp/mysql/mysql.sock
pid-file        = /tmp/mysql/mysql.pid

[mysqld3]
user            = mysql
port            = 3308
datadir         = /tmp/mysql2
log-error       = /tmp/mysql2/error.log
socket          = /tmp/mysql2/mysql.sock
pid-file        = /tmp/mysql2/mysql.pid
```

```bash
mysqld_multi --defaults-file=/etc/mysql/mysql.conf.d/mysqld.cnf start 2
mysqld_multi --defaults-file=/etc/mysql/mysql.conf.d/mysqld.cnf start 3
```

> Comando para iniciar as instâncias número 2 e 3, conforme configuradas no arquivo de configuração especificado. O utilitário `mysqld_multi` permite gerenciar múltiplas instâncias MySQL em um único servidor.

```bash
mysqld_multi --defaults-file=/etc/mysql/mysql.conf.d/mysqld.cnf stop 2
mysqld_multi --defaults-file=/etc/mysql/mysql.conf.d/mysqld.cnf stop 3
```

> Comandos para encerrar as instâncias número 2 e 3.

```bash
mysqladmin --socket='/tmp/mysql/mysql.sock' -uroot -p'4linux' shutdown
```

> Utilitário para enviar comando de desligamento ao servidor MySQL em execução, utilizando autenticação pelo socket e usuário root com senha.


## 3. Acesso e Gerenciamento de Usuários

```bash
mysql -uroot -p -P3307 --socket=/tmp/mysql/mysql.sock
```

> Estabelece conexão com a instância MySQL na porta 3307, utilizando o usuário root, solicitando a senha e definindo o socket específico.

```bash
mysql -uroot -p -P 3307 -h 127.0.0.1
```

> Conexão ao MySQL pela porta 3307, utilizando o IP local (localhost), com autenticação por senha.

```bash
cat /tmp/mysql/error.log | grep password
cat error.log | grep password

```
> Busca por mensagens relacionadas a senhas nos arquivos de log, útil para localizar senhas temporárias ou erros de autenticação.


## 4. Acessos Criptografados com `--login-path`

```bash
mysql_config_editor set --host=172.27.11.10 --user='aluno' --password --login-path=db1 --port=3306
```

> Configura um perfil de conexão seguro e criptografado, identificado por `db1`, definindo host, usuário, porta e solicitando senha que será armazenada criptografada.

```bash
mysql --login-path=db1
```

> Realiza conexão ao MySQL utilizando as credenciais armazenadas no perfil `db1`, eliminando a necessidade de digitar a senha manualmente.

```bash
mysql_config_editor print --all
```

Exibe todos os perfis de conexão configurados no `mysql_config_editor`, com os dados de acesso (exceto as senhas, que são armazenadas de forma segura e não exibidas).

