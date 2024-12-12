******************************************************************************** Elastich Search Kurulum **************************************************************************************************

version: '3.7'

services:
  # Elasticsearch servisi
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.7.1
    environment:
      - node.name=es01
      - cluster.name=docker-cluster
      - discovery.type=single-node
      - ELASTIC_PASSWORD=changeme
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es_data:/usr/share/elasticsearch/data
    networks:
      - elk
    ports:
      - "9200:9200"  # Elasticsearch API port

  # Kibana servisi
  kibana:
    image: docker.elastic.co/kibana/kibana:8.7.1
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"  # Kibana UI port
    networks:
      - elk
    depends_on:
      - elasticsearch

  # Logstash servisi
  logstash:
    image: docker.elastic.co/logstash/logstash:8.7.1
    environment:
      - LOGSTASH_START=1
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5044:5044"  # Logstash Beats input port
    networks:
      - elk
    depends_on:
      - elasticsearch

  # Filebeat servisi
  filebeat:
    image: docker.elastic.co/beats/filebeat:8.7.1
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - LOGSTASH_HOSTS=logstash:5044
    volumes:
      - /var/lib/docker/containers:/hostfs/var/lib/docker/containers:ro
      - /var/log:/hostfs/var/log:ro
      - /proc:/hostfs/proc:ro
      - /sys:/hostfs/sys:ro
      - /etc:/hostfs/etc:ro
    networks:
      - elk
    depends_on:
      - logstash

networks:
  elk:
    driver: bridge

volumes:
  es_data:
    driver: local




----------------------------------------
logstash/pipeline/logstash.conf :


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



---------------------------------------

/etc/filebeat/filebeat.yml: 

filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/*.log

output.logstash:
  hosts: ["logstash:5044"]


