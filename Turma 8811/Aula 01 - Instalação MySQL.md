
# Comandos realizados durante a Aula 01 de Instalação do MySQL

## **Instalação do MySQL no Debian/Ubuntu**

### Instalação via repositório oficial:

```bash
wget https://dev.mysql.com/get/mysql-apt-config_0.8.34-1_all.deb
```
> Baixa o pacote de configuração do repositório APT da MySQL diretamente do site oficial.
---
```bash
dpkg -i mysql-apt-config_0.8.34-1_all.deb
```
> Instala esse pacote `.deb`, adicionando os repositórios MySQL ao sistema.
---
```bash
apt-get update
```
> Atualiza a lista de pacotes disponíveis, incluindo os do repositório MySQL recém-adicionado.
---
```bash
apt-get install mysql-community-server
```
> Instala o servidor MySQL a partir do repositório oficial.

---

### Instalação via repositório manual:

```bash
apt-key adv --keyserver pgp.mit.edu --recv-keys A8D3785C
```

> Importa a chave GPG usada para verificar a autenticidade dos pacotes da MySQL, em caso de erro de verificação.

---

```bash
cd /etc/apt/sources.list.d/
```
> Entra no diretório onde ficam os arquivos de repositórios personalizados.

```bash
cat > mysql.list <<EOF
deb http://repo.mysql.com/apt/debian/ bookworm mysql-8.4-lts
deb http://repo.mysql.com/apt/debian/ bookworm mysql-tools
EOF
```
> Cria manualmente um arquivo de repositório apontando para o MySQL 8.4 e ferramentas complementares.

```bash
apt-get update
apt-get install mysql-community-server
```
> Atualiza os repositórios e instala o MySQL (mesma função dos comandos anteriores).

---

### Instalação via arquivos `.deb`:

```bash
wget https://dev.mysql.com/get/Downloads/MySQL-8.4/mysql-server_8.4.5-1debian12_amd64.deb-bundle.tar
```
> Baixa o bundle com todos os `.deb` necessários para instalar o MySQL manualmente.

```bash
mkdir mysql_install
cp mysql-server_8.4.5-1debian12_amd64.deb-bundle.tar mysql_install/
```
> Cria uma pasta de instalação e copia o arquivo para lá (opcional, por organização).

```bash
tar -xvf mysql-server_8.4.5-1debian12_amd64.deb-bundle.tar
```
>Extrai o conteúdo do bundle com todos os pacotes `.deb`.

```bash
dpkg -i mysql-{common,community-client-plugins,community-client-core,community-client,client,community-server-core,community-server,server}_*.deb
```
> Instala manualmente todos os pacotes necessários do MySQL, na ordem correta.

```bash
apt-get -f install
```
> Corrige eventuais dependências pendentes após a instalação manual.

---

## **Instalação do MySQL no RHEL/Red Hat/CentOS**

### **Instalação via repositório `.rpm`:**

```bash
wget https://dev.mysql.com/get/mysql84-community-release-el9-2.noarch.rpm
```
> Baixa o pacote do repositório MySQL para EL9 (Red Hat/CentOS 9+).

```bash
rpm -i mysql84-community-release-el9-2.noarch.rpm
```
> Instala esse pacote, que configura o repositório MySQL no sistema.

```bash
yum install mysql-community-server
```
> Instala o servidor MySQL a partir do repositório configurado.


---

### Instalação via repositório manual:

```bash
rpm --import http://repo.mysql.com/RPM-GPG-KEY-mysql-2022
```
> Importa a chave usada para verificar a autenticidade dos pacotes da MySQL, em caso de erro de verificação.

```bash 
cat > /etc/yum.repos.d/mysql-community.repo <<EOF
[mysql80-community]
name=MySQL 8.4 Community Server
baseurl=https://repo.mysql.com/yum/mysql-8.4-community/el/9/\$basearch/
enabled=1
gpgcheck=1
gpgkey=https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
EOF
```
> Cria manualmente um arquivo de repositório apontando para o MySQL 8.4 e ferramentas complementares.

```bash
yum install -y mysql-community-server
```
>  instala o MySQL (mesma função dos comandos anteriores).

---

### **Inicialização e configuração:**

```bash
systemctl start mysqld
```
> Inicia o serviço do MySQL.

```bash
tail /var/log/mysqld.log | grep password
```
> Extrai a senha root temporária gerada automaticamente (útil na primeira inicialização).

```bash
mysql_secure_installation
```
> Roda o script de configuração segura: define senha root, remove usuários anônimos, desativa login remoto do root, etc.

---

### **Criação de senha root (exemplo):**

```sql
ALTER USER root@localhost IDENTIFIED BY '4LinuX@2025';
```
> Altera (ou define) a senha do usuário `root` local para `'4LinuX@2025'`.

---

### **Instalação via pacote `.rpm` completo:**

```bash
wget https://dev.mysql.com/get/Downloads/MySQL-8.4/mysql-8.4.5-1.el9.x86_64.rpm-bundle.tar
```
> Baixa o bundle com todos os `.rpm` do MySQL.

```bash
mkdir mysql-debug-rpms
```
> Cria um diretório para mover os pacotes de debug (opcional).

```bash
tar -xvf mysql-8.4.5-1.el9.x86_64.rpm-bundle.tar
```
> Extrai todos os `.rpm` do bundle.

```bash
mv *debug*.rpm mysql-debug-rpms/
```
> Move os pacotes de debug para fora, para evitar instalação desnecessária.

```bash
yum install mysql-community-{server,client,client-plugins,icu-data-files,common,libs}-*
```
> Instala os principais pacotes do MySQL manualmente, na ordem necessária.

---

## **Comandos de Acesso e Criação de Usuários**


```bash
mysql -uroot -p
```
> Conecta ao MySQL com o usuário `root`, solicitando a senha em seguida.  
> Use `-h <host>` para especificar o endereço IP ou hostname em conexões remotas.  
> Use `-P <porta>` para especificar a porta de conexão (caso não seja a padrão `3306`).

```bash
CTRL+D
```
> Sai do MySQL (ou qualquer shell).

```sql
CREATE USER aluno@'%' IDENTIFIED BY '4linux';
```
> Cria um usuário `aluno`, acessível de qualquer IP (`'%'`), com senha `4linux`.

```sql
GRANT SELECT ON *.* TO aluno@'%';
```
> Concede ao usuário `aluno` permissão de leitura (SELECT) em todas as bases.
