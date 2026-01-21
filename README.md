# Домашнее задание к занятию «ELK»

### Задание 1. Elasticsearch 

Установите и запустите Elasticsearch, после чего поменяйте параметр cluster_name на случайный. 

*Приведите скриншот команды 'curl -X GET 'localhost:9200/_cluster/health?pretty', сделанной на сервере с установленным Elasticsearch. Где будет виден нестандартный cluster_name*.

### Решение 1.

Запустил контейнер elasticsearch через docker compose, подставив файл конфигурации на свой:

```YAML
services:
    elasticsearch:
    image: elasticsearch:7.17.9
    container_name: mia-elastic
    environment:
      - xpack.security.enabled=false
      - discovery.type=single-node
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
      - ./elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
    ports:
      - 9200:9200
```

![Запрос health?pretty на адрес elasticsearch](https://github.com/murtazinilyas/11-03-ELK/blob/main/scshots/11.03-1.png)

---

### Задание 2. Kibana

Установите и запустите Kibana.

*Приведите скриншот интерфейса Kibana на странице http://<ip вашего сервера>:5601/app/dev_tools#/console, где будет выполнен запрос GET /_cluster/health?pretty*.

### Решение 2.

Также запустил контейнер kibana:

```YAML
services:
    elasticsearch:
    image: elasticsearch:7.17.9
    container_name: mia-elastic
    environment:
      - xpack.security.enabled=false
      - discovery.type=single-node
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
      - ./elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
    ports:
      - 9200:9200
 
  kibana:
    container_name: mia-kib
    image: kibana:7.17.9
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - 5601:5601
    depends_on:
      - elasticsearch
```

![Результат запроса GET /_cluster/health?pretty в kibana](https://github.com/murtazinilyas/11-03-ELK/blob/main/scshots/11.03-2.png)

---

### Задание 3. Logstash

Установите и запустите Logstash и Nginx. С помощью Logstash отправьте access-лог Nginx в Elasticsearch. 

*Приведите скриншот интерфейса Kibana, на котором видны логи Nginx.*

### Решение 3.

Запустил контейнер logstash, добавил в контейнер файл логов nginx с хоста (предварительно перед этим открыв доступ для чтения всем пользователям для файла логов nginx):

```YAML
  logstash:
    image: logstash:7.17.9
    container_name: mia_logs
    environment:
      XPACK_MONITORING_ENABLED: "false"
      ES_HOST: "elasticsearch:9200"
    ports:
      - "5044:5044/udp"
    volumes:
      - ./logstash/pipelines.yml:/usr/share/logstash/config/pipelines.yml
      - ./logstash/pipelines:/usr/share/logstash/config/pipelines
      - /var/log/nginx/access.log:/var/log/nginx/access.log:ro
    depends_on:
      - elasticsearch
```

Пайплайн для logstash:

```
input {
    file {
        path => "/var/log/nginx/access.log"
        start_position => "beginning"
    }
}
filter {
    grok {
        match => { "message" => "%{IPORHOST:remote_ip} - %{DATA:user_name}\[%{HTTPDATE:access_time}\] \"%{WORD:http_method} %{DATA:url} HTTP/%{NUMBER:http_version}\"%{NUMBER:response_code} %{NUMBER:body_sent_bytes}\"%{DATA:referrer}\" \"%{DATA:agent}\"" }
    }
    mutate {
    remove_field => [ "host" ]
    }
}
output {
    elasticsearch {
        hosts => [ "${ES_HOST}" ]
        data_stream => "true"
    }
}
```

Результат:

![Логи nginx в kibana, отправленные через logstash](https://github.com/murtazinilyas/11-03-ELK/blob/main/scshots/11.03-3.png)

---

### Задание 4. Filebeat. 

Установите и запустите Filebeat. Переключите поставку логов Nginx с Logstash на Filebeat. 

*Приведите скриншот интерфейса Kibana, на котором видны логи Nginx, которые были отправлены через Filebeat.*

### Решение 4.

Также запустил контейнер filebeat, c добавлением в контейнер файла логов nginx с хоста:

```YAML
  filebeat:
    image: docker.elastic.co/beats/filebeat:7.17.9
    container_name: mia-fb
    command: --strict.perms=false
    user: root
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml
      - /var/lib/docker:/var/lib/docker:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/log/nginx/access.log:/var/log/nginx/access.log:ro
```

filebeat.yml:

```YAML
filebeat.inputs:
- type: log
  paths:
    - '/var/log/nginx/access.log'

output.elasticsearch:
  hosts: ["elasticsearch:9200"]

logging.json: true
logging.metrics.enabled: false
```

Результат:

![Логи nginx в kibana, отправленные через filebeat](https://github.com/murtazinilyas/11-03-ELK/blob/main/scshots/11.03-4.png)
