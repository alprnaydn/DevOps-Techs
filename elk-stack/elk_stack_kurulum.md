# ELK-Stack Kurulumu

## Gereksinimler

Proje kurulumuna geçmeden önce **Docker** ve **Docker Compose** kurulumlarının tamamlanmış olduğundan emin olun.

---

## Dosya Yapısı

Kurulum için bir `elk` klasörü oluşturun. Daha sonra klasör içerisine `docker-compose.yml` dosyasını oluşturun. Ardından aşağıdaki yapıya uygun olarak klasörleri ve yapılandırma dosyalarını ekleyin:

```
elk
  | docker-compose.yml
  | elasticsearch
                | config
                        | elasticsearch.yml
  | filebeat
            | config
                    | filebeat.yml
  | kibana
           | config
                    | kibana.yml
  | logstash
           | config
                    | logstash.yml
           | pipeline
                    | logstash.conf
```

---

## Docker Compose Yapılandırması

`docker-compose.yml` dosyasını şu şekilde oluşturun:

```yaml
version: '3.8'
services:
  elasticsearch:
    image: elasticsearch:7.17.0
    container_name: elasticsearch
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    environment:
      - discovery.type=single-node
      - http.host=0.0.0.0
      - transport.host=0.0.0.0
      - xpack.security.enabled=false
      - xpack.monitoring.enabled=false
      - cluster.name=elasticsearch
      - bootstrap.memory_lock=true
    networks:
      - elk

  logstash:
    image: logstash:7.17.0
    container_name: logstash
    ports:
      - "5044:5044"
      - "9600:9600"
    volumes:
      - ./logstash/pipeline/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: kibana:7.17.0
    container_name: kibana
    ports:
      - "5601:5601"
    volumes:
      - ./kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml
    networks:
      - elk
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:7.17.0
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/config/filebeat.yml:/usr/share/filebeat/filebeat.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/log:/var/log:ro
    networks:
      - elk
    depends_on:
      - logstash

volumes:
  elasticsearch-data:
    driver: local

networks:
  elk:
    driver: bridge
```

---

## Yapılandırma Dosyaları

### Elasticsearch Yapılandırması (`elasticsearch.yml`)

```yaml
cluster.name: "elasticsearch"
network.host: localhost
```

### Filebeat Yapılandırması (`filebeat.yml`)

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/*.log
      - /var/lib/docker/containers/*/*.log

output.logstash:
  hosts: ["logstash:5044"]

setup.kibana:
  host: "kibana:5601"
```

### Kibana Yapılandırması (`kibana.yml`)

```yaml
server.name: kibana
server.host: 0.0.0.0
elasticsearch.hosts: [ http://elasticsearch:9200 ]
```

### Logstash Yapılandırması (`logstash.yml`)

```yaml
http.host: 0.0.0.0
xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch:9200"]
```

### Logstash Pipeline Yapılandırması (`logstash.conf`)

```plaintext
input {
  beats {
    port => 5044
  }
}

filter {

}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
  }
}
```

---

## Kurulum Adımları

1. `elk` klasörüne yukarıdaki dosya ve klasörleri ekleyin.
2. Terminalde `elk` klasörüne gidin.
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
Eğer tüm logları toplamak yerine belirli bir konteynırın loglarını toplamak istersek bunun için bir takım düzenlemeler yapmamız gerekir.
ContainerId'yi **/var/lib/docker/containers** altında oluşturulan klasör adları ile aynı olacak şekilde yazın. Kısaltılmış id kullanımında log toplama işlemi başarısız olacaktır. 


filebeat.yml:
   ```
   filebeat.inputs:
  - type: container
    enabled: true
    paths:
      - /var/lib/docker/containers/containerId/*.log # ContainerId kısmını loglarını toplamak istediğiniz konteynırın id'si ile değiştirin. 
    processors:
      - add_docker_metadata:
          host: "unix:///var/run/docker.sock"

    output.logstash:
      hosts: ["<logstash_host>:5044"]  # Logstash'in çalıştığı host ve port

    setup.kibana:
      host: "kibana:5601"

   ```
logstash.conf:
   ```
   input {
      beats {
        port => 5044
      }
    }
    
    filter {
      
      if [container][name] == "${container_name}" {  # Sadece ${container_name} container'ını filtrele. ${container_name} kısmını loglarını toplayacağınız konteynırın adı ile değiştirin. 
        mutate {
          add_field => { "app" => "${container_name}" }
        }
      }
    }
    
    output {
      elasticsearch {
        hosts => ["<elasticsearch_host>:9200"]  # Elasticsearch'in çalıştığı host ve port
        index => "${container_name}-logs-%{+yyyy.MM.dd}"   # ${container_name} kısmını loglarını toplayacağınız konteynırın adı ile değiştirin. 
      }
      stdout {
        codec => rubydebug
      }
    }


   ```



Bu adımları izleyerek tüm logları veya belirli bir konteynır loglarını toplamak için ELK Stack yapısını başarıyla kurabilirsiniz!
