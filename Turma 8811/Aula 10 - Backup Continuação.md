# Aula 10 – Backup no MySQL: Snapshot LVM e Restauração PITR com binlogs.

## 1. Configuração do Snapshot LVM

### Preparando o volume para MySQL

```bash
# Listar discos disponíveis
fdisk -l

# Criar nova partição LVM no disco /dev/sdc
fdisk /dev/sdc
# Comandos dentro do fdisk:
# n   -> nova partição
# t   -> alterar tipo para 8e (Linux LVM)
# w   -> salvar e sair
```

```bash
# Instalar pacotes necessários
apt-get install lvm2 xfsprogs

# Criar volume físico
pvcreate /dev/sdc1
pvs

# Criar volume group
vgcreate mysql /dev/sdc1
vgs

# Criar volume lógico
lvcreate -n 'mysql-data' -L '1g' mysql
lvs

# Formatar em XFS
mkfs.xfs -L 'mysql-data' /dev/mysql/mysql-data

# Montar e mover dados do MySQL
mkdir /srv/mysql
systemctl stop mysql
mount /dev/mysql/mysql-data /srv/mysql/
cp -r /var/lib/mysql/* /srv/mysql/
chown -R mysql: /srv/mysql/
```

Editar `/etc/mysql/my.cnf` para atualizar `datadir`:

```ini
datadir = /srv/mysql
```

```bash
systemctl start mysql
```

---

## 2. Criando Snapshot LVM

```bash
# Criar snapshot (exemplo 1G de tamanho)
lvcreate --size 1G --snapshot --name mysql-snap /dev/mysql/mysql-data

# Montar snapshot para backup
mkdir /mnt/mysql-snap
mount /dev/mysql/mysql-snap /mnt/mysql-snap

# Fazer backup com XtraBackup a partir do snapshot
xtrabackup --backup --target-dir=/backups/snap-$(date +%F) --datadir=/mnt/mysql-snap

# Desmontar e remover snapshot
umount /mnt/mysql-snap
lvremove -f /dev/mysql/mysql-snap
```

---

## 3. Restauração PITR (Point in Time Recovery) com binlogs

```bash
# Preparar backup full
xtrabackup --prepare --apply-log-only --target-dir=/backups/full/2025-08-01

# Aplicar backup incremental
xtrabackup --prepare --target-dir=/backups/full/2025-08-01 --incremental-dir=/backups/inc/2025-08-02

# Restaurar arquivos no datadir
systemctl stop mysql
rm -rf /srv/mysql/*
xtrabackup --copy-back --target-dir=/backups/full/2025-08-01
chown -R mysql:mysql /srv/mysql
```

Aqui está o trecho corrigido e formatado de forma mais clara:

---

**Aplicar binlogs até o ponto desejado:**

```bash
mysqlbinlog --start-datetime="2025-08-01 14:00:00" -v /var/lib/mysql/mysql-bin.000123 | mysql -uroot -p
```

**Gerar fatia parcial de dados:**

```bash
mysqlbinlog --start-position=1203 --stop-datetime="2025-07-30 14:35:00" \
  /var/lib/mysql/mysql-bin.000012 > /tmp/binlog_replay.sql
```

**Aplicar o arquivo gerado no banco:**

```bash
mysql -uroot -p < /tmp/binlog_replay.sql
```

Segue o texto corrigido e formatado:

---

## 4. Extender o LVM do disco

Verificar os grupos de volume disponíveis:

```bash
vgs
```

Exemplo de saída:

```
VG     #PV #LV #SN Attr   VSize     VFree
mysql    1   1   0 wz--n- <10.00g   6.00g
```

---

**Visualizar snapshots existentes e volumes de origem:**

```bash
lvs -o +origin
```

**Remover snapshots ativos antes de estender:**

```bash
lvremove /dev/mysql/mysql-snap
```

---

**Adicionar 1 GB ao volume lógico:**

```bash
lvextend -L +1G /dev/mysql/mysql-data
```

**Ou usar todo o espaço livre do VG:**

```bash
lvextend -l +100%FREE /dev/mysql/mysql-data
```

---

### Redimensionar o sistema de arquivos

Se estiver usando **XFS**:

```bash
xfs_growfs /srv/mysql
```

Se estiver usando **EXT4**:

```bash
resize2fs /dev/mysql/mysql-data
```

---

### Verificações

**Verificar o uso do volume:**

```bash
df -h /srv/mysql
```

**Verificar o tamanho do LV:**

```bash
lvs
```

---

### Garantir montagem automática no `/etc/fstab`

Adicionar uma entrada no `/etc/fstab`:

```
/dev/mapper/mysql-mysql--data      /srv/mysql    xfs    defaults  0 0
/dev/mapper/mysql-mysql--restored  /srv/restore  xfs    defaults  0 0
```