# Aula 06 - Cluster

# InnoDB Cluster

## 1. Instalação e Conexão com MySQL Shell

```bash
# Instala o MySQL Shell (ferramenta para administrar e criar o InnoDB Cluster) nas três máquinas principais (db1, db2, db3)
apt-get install mysql-shell

# Conecta-se ao MySQL Shell como root, pedindo a senha do usuário
mysqlsh -uroot -p

# Conecta-se ao MySQL Shell já no modo JavaScript (utilizado para operações do cluster)
mysqlsh -uroot -p --js
```


## 2. Configuração do /etc/hosts

```bash
nano /etc/hosts

# Adiciona o mapeamento dos nomes e IPs de cada máquina. Isso evita a necessidade de um DNS nessa etapa.
# 172.27.11.10 db1 db1.local
# 172.27.11.20 db2 db2.local
# 172.27.11.30 db3 db3.local
# 172.27.11.40 haproxy haproxy.local
# 172.27.11.60 rhel-demo rhel-demo.local
```

## 3. Preparando a Instância

```bash
# Conecta ao shell JS do MySQL
mysqlsh -uroot -p --js

# Inicia configuração da instância local para cluster (permissões, usuários, configuração automática de variáveis)
dba.configureInstance('root@localhost:3306')
```
- **Durante o comando acima:** será perguntado se quer criar um usuário administrador para o cluster. Escolha:
  - Opção 2: Cria o usuário "cadmin" com a senha "4linux"
  - Responda "y" para aplicar alterações e reiniciar quando solicitado


## 4. Criação e Inicialização do Cluster

```bash
# Conecta-se no MySQL Shell usando o novo admin
mysqlsh -ucadmin -p --js

# Cria o cluster chamado 'cluster_curso'
var cluster = dba.createCluster('cluster_curso')

# Consulta o status do cluster
cluster.status();
```

## 5. Adicionando Membros ao Cluster

```bash
# No DB2, conecte-se usando JS
mysql -uroot -p --js
dba.configureInstance('root@localhost:3306');
# Adiciona a máquina db2 ao cluster criado anteriormente
cluster.addInstance('cadmin@db2')
# Quando solicitado, escolha 'CLONE' para sincronizar totalmente a instância

# Consulta o status do cluster novamente para verificar se db2 entrou corretamente
cluster.status()
```


## 6. Removendo Instância

```js
# Remove a instância 'db2' do cluster, caso precise excluir um nó por manutenção ou problemas
cluster.removeInstance('cadmin@db2')
```

## 7. Consultando o Estado/Status do Cluster

```js
# Recupera um cluster existente
cluster = dba.getCluster()
# Mostra o status e detalhes dos membros do cluster
cluster.status()
```

## 8. Escolhendo Primário e Modo Multi-Primário

```js
# Define a instância primária (master) do cluster
cluster.setPrimaryInstance('db1:3306')

# Habilita o modo multi-primary (multi-master), permitindo escritas em mais de um nó simultaneamente
cluster.switchToMultiPrimaryMode()
```

## 9. Instalação e Configuração do MySQL Router

```bash
# Instala o MySQL Router, que serve como proxy inteligente para o cluster
apt-get install mysql-router

# Interrompe o serviço do Router para reconfiguração
systemctl stop mysqlrouter.service

# Realiza o bootstrap do Router usando as credenciais do cluster (repita em cada nó)
mysqlrouter --bootstrap='cadmin@172.27.11.10' --user='mysqlrouter'
mysqlrouter --bootstrap='cadmin@172.27.11.20' --user='mysqlrouter'
mysqlrouter --bootstrap='cadmin@172.27.11.20' --user='mysqlrouter'
```

## 10. Usuário da Aplicação e Teste

```sql
# Criação de usuário para uso em aplicações e concessão das permissões necessárias
create user aluno@'%' identified by '4linux';
grant select on *.* to aluno@'%';
grant update on employees.employees to aluno@'%';
```

```bash
# Testa a conexão do novo usuário (aluno) ao cluster através do MySQL Router
mysql -ualuno -p -h db2 -P 6450
```

## 11. Alta Disponibilidade: HAProxy e Keepalived

```bash
# Sobe o balanceador HAProxy usando Vagrant
vagrant up hraproxy

# Instala o Keepalived para gerenciamento de IPs flutuantes na alta disponibilidade (VRRP)
apt-get install keepalived

# Abre o arquivo de configuração para customizar as definições dos IPs virtuais
nano /etc/keepalived/keepalived.conf
```

- **Configuração do Keepalived:**  
  (Exemplo para cada servidor; ajuste o "priority" para o master e para os backups)

  - **DB1 (MASTER):**
    ```conf
    vrrp_instance VI_1 {
      state MASTER
      interface eth0
      virtual_router_id 51
      priority 101
      advert_int 1
      virtual_ipaddress {
         172.27.11.70
      }
    }
    ```

  - **DB2 (BACKUP):** `priority 100`
  - **DB3 (BACKUP):** `priority 99`

```bash
# Reinicia o serviço para aplicar as configurações
systemctl restart keepalived
```