
# EFK-Stack Kurulumu
## Açıklama
Bu kurulumda belirli bir konteynır üzerinden fluentd ile log toplayıp bunun elasticsearch üzerinden görüntülenmesi için bir kurulum dosyasından bahsedeceğiz.

## Gereksinimler

Proje kurulumuna geçmeden önce **Docker** ve **Docker Compose** kurulumlarının tamamlanmış olduğundan emin olun.

---

## Dosya Yapısı

Kurulum için bir `efk` klasörü oluşturun. Daha sonra klasör içerisine `docker-compose.yml` dosyasını oluşturun. Ardından aşağıdaki yapıya uygun olarak klasörleri ve yapılandırma dosyalarını ekleyin:

```
efk
  | docker-compose.yml
  | fluentd
           | conf
                 | fluent.conf

```

---

## Docker Compose Yapılandırması

`docker-compose.yml` dosyasını şu şekilde oluşturun:

```yaml
version: '3.8'

services:
  fluentd:
    image: fluent/fluentd-kubernetes-daemonset:v1.14.6-debian-elasticsearch7-1.0
    container_name: fluentd
    ports:
      - "24224:24224"
      - "24224:24224/udp"
    volumes:
      - ./fluentd/conf:/fluentd/etc
    environment:
      - FLUENTD_CONF=fluent.conf
    restart: always

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.13.1
    container_name: elasticsearch
    environment:
      - "discovery.type=single-node"
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:7.13.1
    container_name: kibana
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

```

---

## Yapılandırma Dosyaları


### Fluentd KOnfigürasyon Dosyasının Yapılandırılması (`fluent.conf`)

```plaintext
<source>
  @type forward
  @id forward_input
  port 24224
  bind 0.0.0.0
</source>

<filter *-logs>
  @type parser
  format json
  key_name log
  reserve_data true
</filter>

<match *-logs>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix my-container
  flush_interval 5s
</match>

```

---

## Kurulum Adımları

1. `efk` klasörüne yukarıdaki dosya ve klasörleri ekleyin.
2. Terminalde `efk` klasörüne gidin.
3. Aşağıdaki komutu çalıştırarak servisleri başlatın:

   ```bash
   docker-compose up -d
   ```

4. Tüm servislerin çalıştığını doğrulamak için aşağıdaki komutları kullanabilirsiniz:

   ```bash
   docker ps
   ```

5. Kibana'ya tarayıcı üzerinden erişmek için şu URL'yi kullanın:

   ```
   http://localhost:5601
   ```

---

## Belirli Bir Konteynırın Loglarını Toplama

Fluentd ile belirli bir konteynır loglarını toplamak için şu yöntemlerden birini kullancağız.

### Docker Compose Up

Eğer log toplayacağımız dosyayı çalıştırırken docker-compose.yaml dosyasını kullanıyorsak docker-compose.yaml dosyasına bazı eklemeler yapmalıyız.

docker-compose.yaml(log toplanacak uygulamanın compose dosyası) içerisine aşağıdaki kısım eklenmeli.
```
version: '3.8'

services:
  webapp:
    build: .
    container_name: my-container
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: "{{.Name}}-logs"
    
 
    
```


### Docker Container Run

Eğer docker container run komutu ile konteynır çalıştırıyorsak aşağıdaki komutu kullanabiliriz.

```bash
   docker compose up -d  --log-driver=fluentd   --log-opt tag="{{.Name}}-logs"   --name ${container-name} ${image-name}
```


Bu adımları izleyerek tüm logları veya belirli bir konteynır loglarını toplamak için EFK Stack yapısını başarıyla kurabilirsiniz!
