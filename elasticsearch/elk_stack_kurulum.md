********* ELK-Stack Kurulumu ************

# Prequests: 
Proje kurulumuna geçmeden önce Docker ve Docker compose yapısının kurulu olduğundan emin olun.

# Dosya Yapısı
Kurulum için bir elk klasörü oluşturun. Daha sonra klasör ieçrisine docker-compose.yml dosyasını oluşturun. Ardından elasticsearh, filebeat, kibana ve logstash klasörlerini oluşturun. her klasör içerisine config klasörlerini oluşturun. Sadece logstash klasörü içerisinde config ile beraber pipeline klasörü oluşturun. Her klasörde config kalsörleri içerisine elasticsearch.yml, filebeat.yml, logsatsh.yml, kibana.yml dosyalarını ekleyin. Logstashteki pipeline klasörü altına ise logstash.conf dosyasını ekleyin.


elk
  | docker-compose.yaml
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


#Docker-compose.yaml

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
  # Filebeat servisi
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







#elasticsearch.yml

cluster.name: "elasticsearch"
network.host: localhost



#filebeat.yml

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

#kibana.yml

server.name: kibana
server.host: 0.0.0.0
elasticsearch.hosts: [ http://elasticsearch:9200 ]

#logstash.yml
http.host: 0.0.0.0
xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch:9200"]

#logstash.conf

input {
  beats {
    port => 5044
  }
}

filter {
  # Burada, gelen logları işleyebilirsiniz
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
  }
}




