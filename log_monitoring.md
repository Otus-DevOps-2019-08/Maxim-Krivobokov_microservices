### HW 16 Monitoring - 1
plan:
* Prometheus: запуск, конфигурация, знакомство с Web UI
* Мониторинг состояния микросервисов
* Сбор метрик хоста с использованием экспортера

### подготовка окружения

* будем использовать ВМ в GCP

* prometheus использует tcp порт 9090 и puma использует 9292. 

* создадим правила файервола для gcp
```
gcloud compute firewall-rules create prometheus-default --allow tcp:9090
gcloud compute firewall-rules create puma-default --allow tcp:9292
```


* удаляем старую ВМ, создаем новую, подключаемся к ней

```
docker-machine rm -f docker-host3

docker-machine create --driver google \
--google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/
images/family/ubuntu-1604-lts \
--google-machine-type n1-standard-1 \
--google-zone europe-west1-b \
docker-host

eval $(docker-machine env docker-host)

```

* Prometheus будет запущен внутри контейнера

* для начала воспользуемся готовым образом dockerHub. 

```
# открыли порты 9090, запуск в detached режиме
docker run --rm -p 9090:9090 -d --name prometheus prometheus:v2.1.0
```

* узнаем адрес докер-хоста
```
docker-machine ip docker-host
```

* открыли в браузере этот ip : 9090. работает

* выбрана метрика prometheus_build_info. output: {branch="HEAD",goversion="go1.9.2",instance="localhost:9090",job="prometheus",revision="85f23d82a045d103ea7f3c89a91fba4a93e6367a",version="2.1.0"} 1
=======================
<b>название метрики </b> (prometheus_build_info) - идентификатор собранной информации.

<b>лейбл </b> (branch , goversion, instance, job, revision, version)- добавляет метаданных метрике, уточняет ее.
Использование лейблов дает нам возможность не ограничиваться лишь одним названием метрик для идентификации получаемой
информации. Лейблы содержаться в {} скобках и представлены наборами "ключ=значение".


Значение метрики - численное значение метрики, либо NaN, если значение недоступно

======================

* раздел Targets - представляют собой системы или процессы, за
которыми следит Prometheus. Помним, что Prometheus является
pull системой, поэтому он постоянно делает HTTP запросы на
имеющиеся у него адреса (endpoints). Посмотрим текущий список
целей

* активна цель localhost:9090/metrics. перейдя по адресу host:port/metrics можно посмотреть на информацию, собираемую прометиусом.

* тестовый контейнер prometheus сотановили

### переупорядочим структуру директорий репозитория

* создана папка docker, в нее перенесены docker-monolith, gitlab-ci, src, reddit

* создана папка monitoring/prometheus

* dockerfile, берет образ прометиуса, копирует конфиг с хостовой машины

```
FROM prom/prometheus:v2.1.0
ADD prometheus.yml /etc/prometheus/
```
* создан prometheus.yml
```
---
    global:
      scrape_interval: '5s'
      # c какой частотой собирать метрики
    
    scrape_configs:
      - job_name: 'prometheus'  # job-ы объединяют таргеты (эндпоинты), выполняющие одинаковые функции, в группы.
        static_configs:
          - targets:
            - 'localhost:9090' # адреса для сбора метрик (эндпоинты)
    
      - job_name: 'ui'
        static_configs:
          - targets:
            - 'ui:9292'
    
      - job_name: 'comment'
        static_configs:
          - targets:
            - 'comment:9292'


```


* собираем образ
```
export USER_NAME=username
$ docker build -t $USER_NAME/prometheus .
```

* собираем образы max89k/ui, max89k/post, max89k/comment c помощью скриптов docker_build.sh, лежащимив папках src/<microservice_name>

* редактируем doccer/docker-compose.yml
  * удаляем директивы build (образы уже созданы)
  * добавляем сервис prometheus и его data volume. Прописали ему обе сети и алиас.
```
...
prometheus:
     image: ${USERNAME}/prometheus
     ports:
         - '9090:9090'
     volumes:
         - prometheus_data:/prometheus
    networks:
       front_net:
         aliases:
             - prometheus
       back_net:
          aliases:
             - prometheus
     command:
         - '--config.file=/etc/prometheus/prometheus.yml'
         - '--storage.tsdb.path=/prometheus'
         - '--storage.tsdb.retention=1d'


 volumes:
  post_db:
  prometheus_data:
```

* запустили инфраструктуру

```
docker-compose up -d
```
* reddit и prometeus работают

* в меню targets видим три позиции, все в стостоянии UP

Healthcheck-и представляют собой проверки того, что
наш сервис здоров и работает в ожидаемом режиме. В
нашем случае healthcheck выполняется внутри кода
микросервиса и выполняет проверку того, что все
сервисы, от которых зависит его работа, ему доступны.
Если требуемые для его работы сервисы здоровы, то
healthcheck проверка возвращает status = 1, что
соответсвует тому, что сам сервис здоров.
Если один из нужных ему сервисов нездоров или
недоступен, то проверка вернет status = 0.

* ui_health - состояние сервиса UI. вводим над кнопкой Execute


* Попробуем остановить сервис post на некоторое
время и проверим, как изменится статус ui сервиса,
который зависим от post.
``` 
docker-compose stop post

Stopping my_project_post_1 ... done
```
* healthcheck "упал", видно на графике

* ищем проблемуб смотрим статусы сервисов
```
ui_health_post_availability
ui_healt_comment_availability

```

* видим, что сервис post подвел. Исправим 
```
docker-compose start post
```

### Exporters

Экспортер похож на вспомогательного агента для
сбора метрик.
В ситуациях, когда мы не можем реализовать
отдачу метрик Prometheus в коде приложения, мы
можем использовать экспортер, который будет
транслировать метрики приложения или системы в
формате доступном для чтения Prometheus.

Программа, которая делает метрики доступными
для сбора Prometheus
• Дает возможность конвертировать метрики в
нужный для Prometheus формат
• Используется когда нельзя поменять код
приложения
• Примеры: PostgreSQL, RabbitMQ, Nginx, Node
exporter, cAdvisor

* воспользуемся  Node exporter для сбора инфрормации о ВМ docker-host и отправке Прометиусу.

* запускать его будем тоже в контейнере. Определим сервис в docker/docker-compose.yml, не забыть прописать сети
```
services:
...
  node-exporter:
    image: prom/node-exporter:v0.15.2
    user: root
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points="^/(sys|proc|dev|host|etc)($$|/)"'
    networks: 
      - front_net
      - back_net
```

* добавим job  в конфиг прометиуса
```
scrape_configs:
...
- job_name: 'node'
  static_configs:
  - targets:
  - 'node-exporter:9100'
```

* создание docker образа для promethius.
```
monitoring/prometheus $ docker build -t $USER_NAME/prometheus .
```

* запуск. теперь можно смотреть загрузку ЦП по параметру node_load1 
```
docker-compose up -d
```

* загрузить ЦП и посмотреть результат
```
docker-machine ssh docker-host
 yes > /dev/null
```


* загрузка образов на docker hub
```
docker login
docker push max89k/ui
docker push max89k/comment
docker push max89k/post
docker push max89k/prometheus

```

### ссылка на docker hub

https://hub.docker.com/u/max89k
