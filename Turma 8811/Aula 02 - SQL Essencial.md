# Comandos Realizados na Aula 02 – SQL Essencial

## 1. Criação de banco e tabelas

```sql
CREATE DATABASE ti;
USE ti;

CREATE TABLE cientistas (
  id INT PRIMARY KEY AUTO_INCREMENT,
  nome VARCHAR(100),
  email VARCHAR(50),
  reconhecimento VARCHAR(50),
  nascimento DATE,
  status ENUM('ativo', 'inativo') DEFAULT 'ativo',
  interesses SET('computação', 'matematica', 'engenharia', 'fisica'),
  premiado BIT(1),
  biografia TEXT,
  criado_em DATETIME,
  foto BLOB
);
```

Cria um novo banco de dados chamado `ti` e define que os próximos comandos serão executados dentro desse banco.

Em seguida, é criada a tabela `cientistas`, com os seguintes campos:

-   **`id`**: identificador único com auto incremento.
    
-   **`nome`**, **`email`**, **`biografia`**: campos de texto.
    
-   **`reconhecimento`**: campo de texto que será posteriormente normalizado para outra tabela.
    
-   **`nascimento`**: armazena a data de nascimento.
    
-   **`status`**: pode conter os valores `'ativo'` ou `'inativo'` (ENUM).
    
-   **`interesses`**: permite múltiplos valores, como `'computação'`, `'matemática'`, `'engenharia'` ou `'física'` (SET).
    
-   **`premiado`**: armazena 0 ou 1 (valor binário), indicando se o cientista foi premiado.
    
-   **`criado_em`**: registra a data e hora de criação do registro.
    
-   **`foto`**: armazena imagens em formato binário (BLOB).

---

## 2. Inserção de dados

```sql
-- Inserção individual
INSERT INTO cientistas(nome, email, reconhecimento, nascimento, status, interesses, premiado, biografia, criado_em)
VALUES ('Dennis Ritchie', 'dennis@ritchie.c', 'C', '1941-09-09', 'ativo', 'computação,engenharia', b'1', 'Criador da linguagem C e do UNIX.', NOW());

-- Inserção em lote
INSERT INTO cientistas (nome, email, reconhecimento, nascimento)
VALUES
('Ada Lovelace', 'ada@lovelace.com', 'Potencial Computador', '1815-12-10'),
('Bill Gates', 'bill@gates.win', 'Microsoft', '1955-10-28'),
('Margaret Hamilton', 'margaret@hamilton.nasa', 'Software da Apollo', '1936-08-17');
```
-   Insere um novo cientista na tabela.
    
-   Insere vários cientistas de uma vez só.

---

## 3. Manipulação de dados

Altera dados de uma linha específica, por exemplo:
-   Atualiza interesses.
-   Muda o status de premiado.
-   Adiciona biografia.

```sql
-- Atualização de campos
UPDATE cientistas SET interesses = 'matematica,fisica' WHERE id = 2;
UPDATE cientistas SET premiado = b'1' WHERE nascimento BETWEEN '1900-01-01' AND '1956-01-01';
```

Retorna os nomes e e-mails dos cientistas com ID de 1 a 3:

```sql
-- Consulta simples
SELECT nome, email FROM cientistas WHERE id BETWEEN 1 AND 3;
```
Filtra por um conjunto específico de IDs:

```sql
-- Subquery
SELECT * FROM cientistas WHERE id IN (SELECT id FROM cientistas WHERE nascimento > '1950-01-01');
```

---

## 4. Manipulação de arquivos

Copia uma imagem do host para o diretório onde o MySQL pode acessá-la:
```bash
# Terminal
cp /vagrant/files/foto_cientista.jpeg /var/lib/mysql-files/
```

Carrega o conteúdo da imagem no campo `foto` do cientista com ID 1:
```sql
-- Inserção de imagem
UPDATE cientistas SET foto = LOAD_FILE('/var/lib/mysql-files/foto_cientista.jpeg') WHERE id = 1;
```

Mostra o tamanho (em bytes) do campo `foto`, útil para confirmar se a imagem foi inserida corretamente:
```sql
-- Ver tamanho da imagem
SELECT LENGTH(foto) FROM cientistas WHERE id = 1;
```

---

## 5. Criação de usuário e permissões

Cria um usuário chamado `aluno`, com senha `4linux`, que pode acessar o banco de qualquer IP:
```sql
CREATE USER 'aluno'@'%' IDENTIFIED BY '4linux';
```

Concede permissão de leitura (SELECT) em todas as tabelas de todos os bancos para esse usuário:
```sql
GRANT SELECT ON *.* TO 'aluno'@'%';
```

---

## 6. Criação de tabelas auxiliares e relacionamento

Cria uma nova tabela separada para armazenar reconhecimentos, ligando com o cientista via `cientista_id`:

```sql
CREATE TABLE reconhecimentos (
  id INT PRIMARY KEY AUTO_INCREMENT,
  cientista_id INT,
  item VARCHAR(100)
);
```

Adiciona dados nessa nova tabela, um cientista pode ter vários reconhecimentos (1:N):
```sql
INSERT INTO reconhecimentos (cientista_id, item)
VALUES (1, 'C'), (1, 'Unix'), (2, 'Potencial Computador');
```

Cria a relação oficial (chave estrangeira) entre as tabelas:
```sql
-- Adicionando chave estrangeira
ALTER TABLE reconhecimentos ADD CONSTRAINT fk_cientista_id FOREIGN KEY (cientista_id) REFERENCES cientistas(id);
```

Remove a coluna `reconhecimento`, que agora está em uma tabela separada (normalização):
```sql
-- Remoção de coluna (após normalização)
ALTER TABLE cientistas DROP COLUMN reconhecimento;
```

---

## 7. Importação de arquivos CSV

Lê o conteúdo de um arquivo CSV e insere os dados diretamente na tabela `musicas`.

Campos e parâmetros usados:

-   `FIELDS TERMINATED BY ','`: separador entre colunas.    
-   `OPTIONALLY ENCLOSED BY '"'`: campos podem estar entre aspas.
-   `LINES TERMINATED BY '\n'`: separador de linhas. 
-   `IGNORE 1 LINES`: ignora o cabeçalho.


```sql
CREATE TABLE musicas (
  id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  musica VARCHAR(100),
  artista VARCHAR(50),
  ano CHAR(4)
);

LOAD DATA INFILE '/var/lib/mysql-files/musicas.csv'
INTO TABLE musicas
FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 LINES (musica, artista, ano);
```




---

## 8. Exportação de dados para arquivo

Exporta todos os dados da tabela `cientistas` para um arquivo CSV no servidor:
```sql
SELECT * FROM cientistas
INTO OUTFILE '/var/lib/mysql-files/cientistas.csv'
FIELDS TERMINATED BY ',' ENCLOSED BY '"'
LINES TERMINATED BY '\n';
```

---

## **9. Funções e formatação**

Formata a data de nascimento no formato brasileiro (dia/mês/ano):

```sql
-- Formatação de data
SELECT id, nome, DATE_FORMAT(nascimento, '%d/%m/%Y') FROM cientistas;
```

Substitui o domínio do e-mail de um cientista específico:

```sql
-- Correção de domínio de e-mail com REPLACE
SELECT email, REPLACE(email, '@gates.win', '@gmail.com') AS email_corrigido FROM cientistas WHERE id = 3;
```

Retorna o cientista mais velho e o mais novo (datas):

```sql
-- Mínimos e máximos
SELECT MIN(nascimento), MAX(nascimento), MAX(premiado), MIN(premiado) FROM cientistas;
```

