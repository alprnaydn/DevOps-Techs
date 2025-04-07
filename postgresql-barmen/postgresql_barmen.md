# Postgresql ve Barman ile Veri Yedekleme 

* 2 adet sanal makine üzerinde kurulumu gerçekleştireceğim. İlk sanal makinede postgresql, ikincide barman kurulumu olacak. O anki aşamanın hangi sanala makinede olacağı parantez içinde verilecek.

* 1 = postgresql
* 2 = barman

## Postgresql Kurulumu (1)

* Verilerimizi saklamak için postgresql kurulumu gerçekleştiriyoruz.

* 
```bash
    sudo apt update
    sudo apt install postgresql postgresql-contrib
```



## Barman User Oluşturma (1)

* Barman erişimi için ve kullanımı için postgresql üzerinde user oluşturmalıyız.

```sql
    CREATE USER barman WITH SUPERUSER PASSWORD '123456';
    GRANT USAGE ON SCHEMA pg_catalog TO barman;
    GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO barman;
    GRANT EXECUTE ON FUNCTION pg_catalog.set_config(text, text, boolean) TO barman;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO barman;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_start_backup(text, boolean, boolean) TO barman;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_stop_backup(boolean, boolean) TO barman;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO barman;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO barman;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO barman;
    GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO barman;
    GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO barman;
    GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO barman;
    ALTER USER barman SET statement_timeout TO 0;
```

## Postgresql Ayarlarını Değiştirme (1)

* Postgresql için konfigürasyon dosyalarını düzenlemeliyiz.

* etc/postgresql/{postgresql_version}/main/postgresql.conf
```conf
    #------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

    listen_addresses = '*'                                                    
    archive_mode = on
    archive_command = 'rsync -a %p barman@192.168.122.93:/var/lib/barman/postgresql/incoming/%f'
    wal_level = replica
   
```

* etc/postgresql/{postgresql_version}/main/pg_hba.conf

```conf
    # Allow Barman connections
    host    all             barman          <Barman-server-ip>/32    md5
    # Allow WAL streaming
    host    replication     barman          <Barman-server-ip>/32    md5
```


## Postgresql i yeniden başlatma (1)

* Değişikliklerden sonra postgresqli yeniden başlatmalıyız.

```bash
    systemctl restart postgresql
```

## Dosyalara erişim için gerekli izinleri vermek (1)

* Barman tarafında dosyalara erişim sağlanamsı için postgresql dosylarının konumlarına gerkli izinleri vermeliyiz.

 ```bash
    sudo chown -R postgres:postgres /var/lib/postgresql/<postgresql-version>/main
    sudo chmod -R 700 /var/lib/postgresql/<postgresql-version>/main
 ```

## Postgresql Kurulumu (2)

* Verilerimizi saklamak için postgresql kurulumu gerçekleştiriyoruz.

```bash
    sudo apt update
    sudo apt install postgresql postgresql-contrib
```

## Barman Kurulumu (2)


* 
```bash
    sudo apt update
    sudo apt install barman
```


## Barman Üzerinde Konfigürason Dosyalarının Güncellenmesi (2)

* Barman makinesinde /etc/barman.conf dosyasında şu eklemeyi yapmalıyız:

```conf
    [postgresql]
    description =  "PostgreSQL Sunucusu"
    conninfo = host=<Postgresql-server-ip> user=postgres password=123456 dbname=postgres
    backup_method = rsync
```

* Barman makinesinde /etc/barman.d/postgresql dosyası yoksa oluşturup varsa direkt şu eklemeyi yap:

```conf
    [postgresql]
    description = "PostgreSQL Server"
    conninfo = host=<Postgresql-server-ip> user=barman dbname=postgres password=<barman-user-password>
    streaming_conninfo = host=<Postgresql-server-ip> user=barman dbname=postgres password=<barman-user-password>
    backup_method = rsync
    ssh_command = ssh postgres@<Postgresql-server-ip>
    archiver = on

```

* Barman makinesinde /etc/barman.d/pg.conf dosyasına şu eklemeyi yap:

```conf
    [postgresql]
    description = "PostgreSQL Server"
    conninfo = host=<Postgresql-server-ip> user=barman dbname=postgres password=<barman-user-password>
    streaming_conninfo = host=<Postgresql-server-ip> user=barman dbname=postgres password=<barman-user-password>
    backup_method = rsync
    ssh_command = ssh postgres@<Postgresql-server-ip>
    archiver = on

```

## SSH Bağlantısını Sağlama

* Ssh bağlantısı sırasında password istememesi lazım bunun için karşılıklı ssh-keygen oluşturup karşı tarafa kopyalıyoruz:

(1)
```bash
    # Generate SSH key if you haven't already
    # Press Enter end of the command
    sudo -u postgres ssh-keygen -t rsa

    # Copy the SSH key to Barman server
    sudo -u barman ssh-copy-id barman@<Barman-server-ip>
```


(2)
```bash
    # Generate SSH key if you haven't already
    # Press Enter end of the command
    sudo -u barman ssh-keygen -t rsa

    # Copy the SSH key to PostgreSQL server
    sudo -u barman ssh-copy-id postgresql@<Postgresql-server-ip>
```

## Backup İçin Gerekli Klasörlerin OLuşturulması

```bash
    sudo mkdir -p /var/lib/barman/postgresql
    sudo mkdir -p /var/lib/barman/postgresql/incoming
```

## Sistem Kontrolü (2)

* Sistemin doğru çalışıp çalışmadığını kontrol etmek için şu komutu çalıştır:

```bash
    barman check postgresql
```

## Backup İşlemi (2)

* Backup işlemini gerçekleştirmek için şu komutu çalıştır:

```bash
    sudo barman backup postgresql
```

## Backuptan Geri Yükleme (2)

* Backuptan postgresql e geri veri yüklemek için şu komutu çalıştır:

* Alınan backuptan geri veri yüklemek için aşağıdaki komutu çalıştır. Backup-directory-name e /var/lib/barman/postgresql/base altındaki klaösrden erişebilirsin. Klasör adları = backup adı

```bash
    sudo barman recover --remote-ssh-command "ssh postgres@<POstgresql-server-ip>" postgresql <backup-direktory-name> /var/lib/postgresql/<postgresql-version>/main
```
