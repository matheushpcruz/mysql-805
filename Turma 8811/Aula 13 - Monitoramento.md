# Aula 13 — Monitoramento do MySQL

## **1. ZABBIX**

### **Instalação do Zabbix na VM `monitor`**
```bash
wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.0+debian12_all.deb  
# Baixa o pacote de configuração do repositório oficial do Zabbix

dpkg -i zabbix-release_latest_7.0+debian12_all.deb  
# Instala o pacote de configuração do repositório no sistema

apt update  
# Atualiza a lista de pacotes disponíveis no repositório

apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent  
# Instala o servidor Zabbix, a interface web, módulos do Apache e o agente
```


### **Instalar MySQL server para o Zabbix**
```bash
wget https://dev.mysql.com/get/mysql-apt-config_0.8.34-1_all.deb  
# Baixa o pacote de configuração do repositório oficial MySQL

dpkg -i mysql-apt-config_0.8.34-1_all.deb  
# Instala o repositório MySQL

apt-get install -y mysql-community-server  
# Instala o servidor MySQL
```


### **Criar banco e usuário no MySQL para o Zabbix**
```sql
mysql -uroot -p  
# Acessa o MySQL com o usuário root

CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;  
# Cria o banco de dados “zabbix” com codificação UTF-8

CREATE USER zabbix@localhost IDENTIFIED BY 'password';  
# Cria um usuário chamado “zabbix” com senha “password”

GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;  
# Dá permissão total de acesso ao banco “zabbix”

SET GLOBAL log_bin_trust_function_creators = 1;  
# Permite criar funções armazenadas sem restrições (necessário para importação)
```



### **Importar o schema do Zabbix**
```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | \
mysql --default-character-set=utf8mb4 -uzabbix -p zabbix  
# Descompacta e importa as tabelas e dados iniciais do Zabbix para o banco
```



### **Ajustar configuração e iniciar serviços na VM `monitor`**
Editar `/etc/zabbix/zabbix_server.conf`:
```
DBPassword=password  
# Define a senha do usuário do banco no arquivo de configuração
```
Depois:
```bash
systemctl restart zabbix-server zabbix-agent apache2  
# Reinicia os serviços do Zabbix Server, Agente e Apache

systemctl enable zabbix-server zabbix-agent apache2  
# Configura para iniciar automaticamente ao ligar a máquina
```


### **Configurar Agente na VM `db1`**
```bash
wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.0+debian12_all.deb  
# Baixa o repositório oficial do Zabbix

dpkg -i zabbix-release_latest_7.0+debian12_all.deb  
# Instala repositório

apt update  
# Atualiza lista de pacotes

apt install zabbix-agent2  
# Instala agente de monitoramento Zabbix v2
```

Editar `/etc/zabbix/zabbix_agent2.conf`:
```
Server=172.27.11.50     # IP do servidor Zabbix
ServerActive=172.27.11.50
Hostname=DB1            # Nome do host que aparecerá no Zabbix
```

***

### **Criar usuário mínimo para monitoramento no MySQL**
```sql
CREATE USER 'zbx_monitor'@'localhost' IDENTIFIED BY 'password';  
# Cria usuário de monitoramento

GRANT USAGE, REPLICATION CLIENT, PROCESS, SHOW DATABASES, SHOW VIEW ON *.* TO 'zbx_monitor'@'localhost';  
# Dá permissões mínimas necessárias para coleta de métricas
```


## **2. NAGIOS**

### **Instalação e configuração**
```bash
apt-get update  
# Atualiza lista de pacotes

apt-get install -y autoconf gcc make wget unzip apache2 apache2-utils php libgd-dev openssl libssl-dev  
# Instala dependências para compilar o Nagios e configurar o Apache
```

```bash
cd /tmp  
# Vai para pasta temporária

wget -O nagioscore.tar.gz https://github.com/NagiosEnterprises/nagioscore/archive/nagios-4.4.14.tar.gz  
# Baixa o código-fonte do Nagios

tar xzf nagioscore.tar.gz  
# Descompacta

cd nagioscore-nagios-4.4.14/  
# Entra na pasta do código

./configure --with-httpd-conf=/etc/apache2/sites-enabled  
# Configura compilação com integração ao Apache

make all  
# Compila o Nagios
```

```bash
make install-groups-users  
# Cria grupo/usuário nagios

usermod -a -G nagios www-data  
# Dá permissão ao Apache para acessar arquivos do Nagios
```

```bash
make install               # Instala binários e arquivos HTML  
make install-daemoninit    # Instala serviço Nagios  
make install-commandmode   # Habilita comandos externos  
make install-config        # Copia arquivos de configuração de exemplo  
make install-webconf       # Configura Apache para Nagios
```

```bash
a2enmod rewrite  
a2enmod cgi  
# Ativa módulos necessários no Apache

iptables -I INPUT -p tcp --dport 80 -j ACCEPT  
# Libera acesso HTTP

apt-get install -y iptables-persistent  
# Salva regras de firewall permanentemente
```

```bash
htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin  
# Cria usuário para acessar interface web do Nagios
```

```bash
systemctl restart apache2  
systemctl start nagios  
# Inicia Apache e Nagios
```


### **Monitorar MySQL no Nagios**
```bash
apt install nagios-plugins-standard  
# Instala plugins padrões do Nagios

cp /usr/lib/nagios/plugins/check_* /usr/local/nagios/libexec  
# Copia scripts de verificação para pasta executável do Nagios
```

Criar `db1.cfg` (define host e serviço):
```cfg
define host {
    use                 linux-server
    host_name           db1
    alias               Servidor MySQL
    address             172.27.11.10
    max_check_attempts  3
}

define service {
    host_name           db1
    service_description MySQL - Check Status
    check_command       check_mysql!zbx_monitor!password!mysql
    max_check_attempts  3
    check_period        24x7
}
```

Adicionar comando no `commands.cfg`:
```cfg
define command {
    command_name check_mysql
    command_line $USER1$/check_mysql -H $HOSTADDRESS$ -u $ARG1$ -p $ARG2$ -d $ARG3$
}
```

```bash
/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg  
# Valida se a configuração do Nagios está correta

systemctl restart nagios  
# Reinicia Nagios para aplicar mudanças
```


## **3. PMM (Percona Monitoring and Management)**

### **Parar serviços Zabbix e Nagios para liberar recursos**
```bash
systemctl stop apache2  
systemctl stop zabbix-server.service  
systemctl stop nagios  
# Evita sobrecarga de CPU/memória na VM de monitoramento
```



### **Instalar Docker e subir PMM Server**
```bash
apt-get update && apt-get install -y ca-certificates curl  
# Atualiza pacotes e instala utilitários

install -m 0755 -d /etc/apt/keyrings  
# Cria pasta para chaves do repositório

curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc  
# Baixa chave GPG do Docker

chmod a+r /etc/apt/keyrings/docker.asc  
# Dá permissão de leitura
```

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
tee /etc/apt/sources.list.d/docker.list > /dev/null  
# Adiciona repositório oficial do Docker
```

```bash
apt-get update  
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin  
# Instala Docker Engine e plugins
```

```bash
docker run -d -p 8443:8443 --name pmm-server percona/pmm-server:latest  
# Baixa e executa o container do PMM Server (porta 8443)
```


### **Instalar PMM Client na VM `db1`**
```bash
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb  
# Baixa pacote de configuração da Percona

sudo dpkg -i percona-release_latest.generic_all.deb  
# Instala o repositório Percona
```

```bash
sudo percona-release enable pmm3-client release  
# Ativa repositório do PMM Client (versão 3)
```

```bash
sudo apt update  
sudo apt install -y pmm-client  
# Instala agente cliente
```

Criar usuário MySQL:
```sql
CREATE USER 'pmm'@'127.0.0.1' IDENTIFIED BY 'password' WITH MAX_USER_CONNECTIONS 10;  
GRANT SELECT, PROCESS, REPLICATION CLIENT, RELOAD, BACKUP_ADMIN ON *.* TO 'pmm'@'127.0.0.1';  
# Permissões para coleta de métricas
```

Conectar cliente ao server:
```bash
pmm-admin config --server-insecure-tls --server-url=https://admin:admin@172.27.11.50:8443  
# Configura cliente para apontar para o servidor PMM

pmm-admin add mysql --query-source=perfschema --username=pmm --password=password  
# Adiciona serviço MySQL para monitoramento
```
