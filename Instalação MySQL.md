**Instalação do MySQL:**

### Forma Automática pelos Repositórios:

1. Acesse o link: [https://dev.mysql.com/downloads/repo/apt/](https://dev.mysql.com/downloads/repo/apt/)
2. Obtenha o link para download do arquivo `.deb`.
3. No servidor, execute:
   ```bash
   wget https://dev.mysql.com/get/mysql-apt-config_0.8.32-1_all.deb
   ```
4. Instale o pacote:
   ```bash
   dpkg -i mysql-apt-config_0.8.32-1_all.deb
   ```
5. Durante a instalação, escolha a opção desejada. O mais comum é trocar de `8.4 LTS` para `8.0`, que são as versões mais utilizadas.
6. Atualize a lista de pacotes:
   ```bash
   apt-get update
   ```
7. Instale o MySQL:
   ```bash
   apt-get install mysql-community-server
   ```

### Instalação Manual:

1. Adicione as chaves:
   ```bash
   apt-key adv --keyserver pgp.mit.edu --recv-keys A8D3785C
   ```
2. Gere o arquivo do repositório:
   ```bash
   cat > /etc/apt/sources.list.d/mysql.list <<EOF
   deb http://repo.mysql.com/apt/debian/ bookworm mysql-8.0
   deb http://repo.mysql.com/apt/debian/ bookworm mysql-tools
   EOF
   ```
3. Atualize a lista de pacotes:
   ```bash
   apt-get update
   ```
4. Instale o MySQL:
   ```bash
   apt-get install mysql-community-server
   ```

### Instalação Não Interativa:

Se preferir uma instalação não interativa, siga os passos abaixo:

1. Defina as senhas root com o comando:
   ```bash
   debconf-set-selections <<< "mysql-community-server mysql-community-server/root-pass password mypassword"
   debconf-set-selections <<< "mysql-community-server mysql-community-server/re-root-pass password mypassword"
   ```
2. Instale o MySQL sem interação:
   ```bash
   DEBIAN_FRONTEND=noninteractive apt-get -y install mysql-community-server
   ```

### Habilitar e Iniciar o Serviço MySQL:

Como utilizamos o Debian, o serviço é iniciado automaticamente após a instalação. Porém, se necessário, siga os passos abaixo para habilitar e iniciar o serviço:

```bash
systemctl enable --now mysql
```

Desta forma, o serviço será habilitado e inicializado automaticamente.
