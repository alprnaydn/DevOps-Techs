# Fluentd ve Timescaledb ile Veri Toplama ve Depolama

Bu dosyamızda fluentd ve postgresql extensionı olan timescaledb kullanarak belirli bir konteynır için veri toplama ve depolama işlemlerini gerçekleştireceğiz.

## Gereksinimler

- Docker
- Docker Compose

## Kurulum

### Dosya Yapısı


```
fluentd-timescaledb-stack
        | docker-compose.yml
        | init.sql
        | Dockerfile
        | fluentd
            | conf
                | fluent.conf

```

### Docker Compose Dosyası

Aşağıdaki `docker-compose.yml` dosyasını oluşturun:

```yaml

version: '3.8'
services:
  fluentd:
    build: .
    ports:
      - "24224:24224"
      - "24224:24224/udp"
      - "24220:24220"
    volumes:
      - ./fluentd/conf/fluent.conf:/fluentd/etc/fluent.conf
    environment:
      FLUENTD_CONF: fluent.conf
    networks:
      - postgres
    depends_on:
      postgres:
        condition: service_healthy
    restart: always

  timescaledb:
    image: timescale/timescaledb:latest-pg14
    container_name: timescaledb
    environment:
      POSTGRES_DB: your_database_name
      POSTGRES_USER: your_username
      POSTGRES_PASSWORD: your_password
    ports:
      - "5432:5432"
    volumes:
      - postgres:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U your_username -d your_database_name"]
      interval: 10s
      timeout: 5s
      retries: 5

  your-app:
    image: your-image
    ports:
      - your-port:your-port
    links:
      - fluentd
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: your-app-tag

networks:
  postgres:
    driver: bridge

volumes:
  postgres:

```


### Fluentd Konfigürasyonu

fluent image dosyasının oluşturulması için Dockerfile dosyasını oluşturun ve aşağıdaki içeriği ekleyin.

```Dockerfile

FROM fluent/fluentd:v1.14-1

USER root

RUN apk add --no-cache \
    build-base \
    ruby-dev \
    postgresql-dev \
    postgresql-client \
    curl \
    netcat-openbsd

RUN gem install bundler -v 2.4.22 && \
    gem install rake -v 13.0.6 && \
    gem install zeitwerk -v 2.6.18 && \
    gem install pg -v '~> 1.1' --no-document && \
    gem install fluent-plugin-sql && \
    gem install activerecord -v '~> 6.1'


RUN apk del build-base ruby-dev

USER fluent


```


fluentd.conf dosyasını oluşturun ve aşağıdaki içeriği ekleyin. fluentd.conf dosyası loglar üzerinde işlemler yapmayı sağlar.

```

<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<filter **>
  @type record_transformer
  enable_ruby
  <record>
    time ${Time.now.utc.iso8601}
  </record>
</filter>

<filter docker.**>
  @type parser
  format json 
  key_name log
  reserve_data true
</filter>

<match *.*>
  @type sql
  host timescaledb
  port 5432
  database your_daatabase_name
  username your_username
  password your_password
  adapter postgresql
  <table>
    table your_table_name
    column_mapping container_id:container_id, container_name:container_name, source:source, log:message, time:timestamp
  </table>
  
  debug true
  log_level debug
</match>

<label @FLUENT_LOG>
  <match fluent.*>
    @type stdout
  </match>
</label>

```


### init.sql Konfigürasyonu

init.sql dosyasını oluşturun ve aşağıdaki içeriği ekleyin. init.sql dosyası timescaledb veritabanınızda tablo oluşturmayı sağlar.

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

SET ROLE fluentd_user;

DROP TABLE IF EXISTS your_table_name;
CREATE TABLE your_table_name (
  id SERIAL,
  container_id VARCHAR(255),
  container_name VARCHAR(255),
  source VARCHAR(255),
  message TEXT,
  timestamp TIMESTAMP PRIMARY KEY
);
-- BU fonksiyon ile 1 haftalık chunk'lar oluşturulacak.
SELECT create_hypertable('your_table_name', 'timestamp',chunk_time_interval => INTERVAL '1 week');

-- Yapı şöyle olacak:
-- 1- Tabloyu 1 haftalık chunk'lar halinde saklayacağım.
-- 2- 3 ayı dolduran datalar silinecek.
-- 3- Silme işlemleri gibi fonksiyonlar günde 1 defa ve verilerin silinmesinin kullanıcıları etkilemeyeceği bir saatte yapılacak.

SELECT add_retention_policy('your_table_name', 
    INTERVAL '3 months',
    if_not_exists => true,
    schedule_interval => INTERVAL '1 week',
    initial_start => date_trunc('day', now()) + INTERVAL '25 hours'
);



-- Lütfen verdiğiniz izinleri kontrol edin.
-- Grant permissions
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO fluentd_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO fluentd_user;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA _timescaledb_functions TO fluentd_user;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA _timescaledb_internal TO fluentd_user;

```

## Kullanım

1. Proje dizinine gidin.
2. `docker-compose up -d` komutunu çalıştırın.
3. Uygulamanızın loglarının fluentd'e gönderilmesini sağlayın.(Örneğin projeniz bir web sitesi ise siteyi açın ve yenileyerek log oluşmasını sağlayın.)
4. Timescaledb veritabanına bağlanın ve `your_table_name` tablosunu kontrol edin.

## Timescaledb Veritabanına Bağlanma

Timescaledb veritabanına bağlanmak için aşağıdaki komutu kullanabilirsiniz:

```bash
docker exec -it <Container_Adı> psql -d <Database_Adı> -U <Kullanıcı_Adı>
```

Timescaledb'e bağlandıktan sonra aşağıdaki komut yapısı ile tablonuzu kontrol edip içerisindeki datayı çekebilirsiniz:

```bash
\dt
```
```sql
SELECT * FROM your_table_name
```

## Birden Fazla Konteynır için Log Toplama

Birdan fazla konteynır için log toplamak ve bunları farklı tablolarda timescaledbde depolamak için bazı değişiklikler yapmamız gerekiyor. Bu düzenlemeleri yaptıktan sonra yukarıda belirtilen şekilde timescaledbye bağlanıp logları görebilirsiniz.

### docker-compose.yaml

```yaml
version: '3.8'
services:
  fluentd:
    build: .
    ports:
      - "24224:24224"
      - "24224:24224/udp"
      - "24220:24220"
    volumes:
      - ./fluentd/conf/fluent.conf:/fluentd/etc/fluent.conf
    environment:
      FLUENTD_CONF: fluent.conf
    networks:
      - postgres
    depends_on:
      postgres:
        condition: service_healthy
    restart: always

  timescaledb:
    image: timescale/timescaledb:latest-pg14
    container_name: timescaledb
    environment:
      POSTGRES_DB: your_database_name
      POSTGRES_USER: your_username
      POSTGRES_PASSWORD: your_password
    ports:
      - "5432:5432"
    volumes:
      - postgres:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U your_username -d your_database_name"]
      interval: 10s
      timeout: 5s
      retries: 5

  your-app-1:
    image: your-image-1
    ports:
      - your-port:your-port-1
    links:
      - fluentd
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: your-app-tag-1

  your-app-2:
    image: your-image
    ports:
      - your-port:your-port-2
    links:
      - fluentd
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: your-app-tag-2

networks:
  postgres:
    driver: bridge

volumes:
  postgres:


```

### init.sql 

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

SET ROLE fluentd_user;

DROP TABLE IF EXISTS your_table_1;
CREATE TABLE your_table_1 (
  id SERIAL,
  container_id VARCHAR(255),
  container_name VARCHAR(255),
  source VARCHAR(255),
  message TEXT,
  timestamp TIMESTAMP PRIMARY KEY
);
SELECT create_hypertable('your_table_1', 'timestamp',chunk_time_interval => INTERVAL '1 week');

DROP TABLE IF EXISTS your_table_2;
CREATE TABLE your_table_2 (
  id SERIAL,
  container_id VARCHAR(255),
  container_name VARCHAR(255),
  source VARCHAR(255),
  message TEXT,
  timestamp TIMESTAMP PRIMARY KEY
);
SELECT create_hypertable('your_table_2', 'timestamp',chunk_time_interval => INTERVAL '1 week');



DROP TABLE IF EXISTS your_table_3;
CREATE TABLE your_table_3 (
  id SERIAL,
  container_id VARCHAR(255),
  container_name VARCHAR(255),
  source VARCHAR(255),
  message TEXT,
  timestamp TIMESTAMP PRIMARY KEY
);
SELECT create_hypertable('your_table_3', 'timestamp',chunk_time_interval => INTERVAL '1 week');


-- Yapı şöyle olacak:
-- 1- Tabloyu 1 haftalık chunk'lar halinde saklayacağım.
-- 2- 3 ayı dolduran datalar silinecek.
-- 3- Silme işlemleri gibi fonksiyonlar günde 1 defa ve verilerin silinmesinin kullanıcıları etkilemeyeceği bir saatte yapılacak.

SELECT add_retention_policy('your_table_1', 
    INTERVAL '3 months',
    if_not_exists => true,
    schedule_interval => INTERVAL '1 week',
    initial_start => date_trunc('day', now()) + INTERVAL '25 hours'
);

SELECT add_retention_policy('your_table_2', 
    INTERVAL '3 months',
    if_not_exists => true,
    schedule_interval => INTERVAL '1 week',
    initial_start => date_trunc('day', now()) + INTERVAL '25 hours'
);

SELECT add_retention_policy('your_table_3', 
    INTERVAL '3 months',
    if_not_exists => true,
    schedule_interval => INTERVAL '1 week',
    initial_start => date_trunc('day', now()) + INTERVAL '25 hours'
);



-- Lütfen verdiğiniz izinleri kontrol edin.
-- Grant permissions
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO fluentd_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO fluentd_user;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA _timescaledb_functions TO fluentd_user;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA _timescaledb_internal TO fluentd_user;

```

### fluent.conf

```conf
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<filter **>
  @type record_transformer
  enable_ruby
  <record>
    time ${Time.now.utc.iso8601}
  </record>
</filter>

<filter docker.**>
  @type parser
  format json 
  key_name log
  reserve_data true
</filter>

<match *.*>
  @type sql
  host timescaledb
  port 5432
  database your_daatabase_name
  username your_postgres_username
  password your_password
  adapter postgresql
  <table your-tag-1>
    table your_table_1
    column_mapping container_id:container_id, container_name:container_name, source:source, log:message, time:timestamp
  </table>

  <table your-tag-2>
    table your_table_2
    column_mapping container_id:container_id, container_name:container_name, source:source, log:message, time:timestamp
  </table>

  <table>
    table your_table_3
    column_mapping container_id:container_id, container_name:container_name, source:source, log:message, time:timestamp
  </table>

  debug true
  log_level debug
</match>

<label @FLUENT_LOG>
  <match fluent.*>
    @type stdout
  </match>
</label>

```


## Katkıda Bulunma

Katkılarınızı bekliyoruz! Lütfen bir sorunuz veya isteğiniz bir özellik varsa belirtiniz.

