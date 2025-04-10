# Postgresql ve Barman ile Veri Yedekleme 

* 2 adet sanal makine üzerinde kurulumu gerçekleştireceğim. İlk sanal makinede postgresql, ikincide barman kurulumu olacak. O anki aşamanın hangi sanal makinede olacağı parantez içinde verilecek.

* 1 = postgresql
* 2 = barman

## Genel Şema
```
 +----------------------+                   +----------------------+
|                      |                   |                      |
|   PostgreSQL Server   |                   |     Barman Server    |
|      (postgres)       |                   |      (barman)        |
|                      |                   |                      |
+----------+-----------+                   +-----------+----------+
           |                                                   |
           |                                                   |
           | SSH: postgres → barman (Yedekleme için)           |
           | archive_command: rsync ile WAL dosyaları gönderir |
           +-------------------------------------------------->|
           |                                                   |
           |                                                   |
           | SSH: barman → postgres (Geri yükleme için)         |
           | barman recover sırasında rsync ile dosyaları kopyalar|
           |<--------------------------------------------------+
           |                                                   |
+----------+-----------+                   +-----------+----------+
|                      |                   |                      |
|  postgresql.conf     |                   |  /etc/barman.d/      |
|  pg_hba.conf         |                   |  postgresql.conf     |
|                      |                   |                      |
+----------------------+                   +----------------------+
```

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
    CREATE ROLE barman WITH LOGIN REPLICATION PASSWORD 'your-password';
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

* Barman tarafında dosyalara erişim sağlanması için postgresql dosyalarının konumlarına gerekli izinleri vermeliyiz.

 ```bash
    sudo chown -R postgres:postgres /var/lib/postgresql/<postgresql-version>/main
    sudo chmod -R 700 /var/lib/postgresql/<postgresql-version>/main
 ```


## Barman Kurulumu (2)


* 
```bash
    sudo apt update
    sudo apt install barman
```

## PostgreSQL Client Kurulumu (2)
* Barman üzerinde postgresql client kuruyoruz.
```bash
sudo apt update
sudo apt install postgresql-client
```

## Barman Üzerinde Konfigürasyon Dosyalarının Güncellenmesi (2)

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

## Alınan Backupları Listeleme (2)

* Almış olduğumuz backupları aşağıdaki komut ile listeleyebiliriz.

```bash
    sudo barman list-backup postgresql
```

## Backuptan Geri Yükleme (2)

* Backuptan postgresql e geri veri yüklemek için şu komutu çalıştır:

* Alınan backuptan geri veri yüklemek için aşağıdaki komutu çalıştır. Backup-directory-name e /var/lib/barman/postgresql/base altındaki klaösrden erişebilirsin. Klasör adları = backup adı

```bash
    sudo barman recover --remote-ssh-command "ssh postgres@<Postgresql-server-ip>" postgresql <backup-direktory-name> /var/lib/postgresql/<postgresql-version>/main
```
* Bundan sonra PostgreSQL makinesinde postgresql servisini başlat.
```bash
     sudo systemctl start postgresql
```
