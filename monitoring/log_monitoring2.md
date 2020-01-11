### Homework 17. Monitoring -2

* создана ветка monitoring-2

* запущен docker-host в GCE, IP 35.205.40.222

* созданы правила файерволла для портов: 9090, 9093, 8080, 9292, 2376, 3000.

* docker-compose.yml разделен на две части - основной запускает микросервисы Реддит, второй docker-compose-monitoring.yml запускает контейнеры с приложениями для мониторинга. Запускается командой
```
docker-compose -f docker-compose-monitoring.yml up -d
```

* добавим контейнер с cAdvisor, для слежения за состоянием докер-хоста и контейнеров
* в *мониторинг.юмл добавлен сервис cadvisor

* добавили информацию о нем в конифг Prometheus - monitoring/prometheus.yml

* docker- образ прометиуса пересобран, сервисы докер-компоуза перезапущены

* cAdvisor UI имеет Web UI по адресу <IP>:8080

* Добавил инструмент Grafana
  * Добавил новый сервис в docker-compose-monitoring.yml, прописав ему обе сети (front и back)
  * логин-пароль передаются через environment переменные
  * графана работает на порту 3000

* в GUI графаны добавил источник данных - Prometheus server (http://prometheus:9090)

* загрузил с оф.сайта готовых дашбордов Docker and system monitoring. Его json файл лежит в monitoring/grafana/dashboards/DockerMonitoring.json

* в конфиг прометиуса (monitoring/prometheus/prometheus.yml) добавлена информация о post-сервисе. для сбора с него метрик. Образ Prometheus пересоздан, инфраструктура перезапущена. В реддит добавлено несколько постов и комментов


* в Grafana создан dashboard UI_Service_monitoring с несколькими графиками
   * счетчик частоты запросов к интерфейсу  rate (ui_request_count)
   * частота запросо с ошибочным кодом возврата rate (ui_request_count{http_status=~[45].*}[1m])
   * добавлена гистограмма с выводом  95 перцентили времени отклика сервера на http запрос 
* дашборд сохранен и экспортирован в monitoring/grafana/dashboards/UI_service_monitoring.json

* созданы дашборды для бизнес логики, считающие частоту написание постов и комментов. Экспортирован в monitoring/grafana/dashboards/Business_logic_monitoring.json

#### alerting

* создан докер-образ max89k/alertmanager, из базового prom/alertmanager:v0.14.0 ; загружающий конфиг файл config.yml. 

* конфиг файл создает веб-хук для моего канала slack

*  в compose- файл мониторинга добавлен сервис alertmanager. Сам алертменеджер работает на порту 9093

* создан файл monitoring/prometheus/alerts.yml . В нем определены условия, при которых должен срабатывать алерт. 

* Алерт будет срабатывать в ситуации, когда одна из наблюдаемых систем (endpoint) недоступна для сбора метрик (в этом случае метрика up с
лейблом instance равным имени данного эндпоинта будет равна нулю).

* конфиг prometheus.yml обновлен, добавлена информация об alertmanagers
* опция копирования alerts.yml добавлена в докерфайл для образа prometheus. Образ пересоздан. Инфраструктура докер-компоуза перезапущена

* алертменеджер работает

* все образы выложены в докер-хаб

https://hub.docker.com/u/max89k

