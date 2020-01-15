# Maxim-Krivobokov_microservices
Maxim-Krivobokov microservices repository

### подготовка репозитория
* создан шаблон для PR ./.github/PULL_REQUEST_TEMPLATE.md
* в своем канале slack настроил подписку на все коммиты из этого репозитория
````
/github subscribe Otus-DevOps-2019-08/Maxim-Krivobokov_microservices commits:all
````
* делаем интеграцию с тестировщиков play-travis
  * папка play-travis ,  в ней файл test.py

  * в корне репозитория файл .travis.yml c настройками 

### Homework - Docker -1 & 2

* Используется ветка docker-2

Что проделано:
* устанавлен Docker - 19.03.4 ; docker-compose 1.24.1; docker-machine 0.16.0
* настроен запуск команд докера без root прав, путем добавления текущего пользователя в группу docker
````
sudo usermod -aG docker <username>
````
* после установки запускаем тестовый контейнер
````
docker run hello-world # запуск тестового контейнера
````

Что произошло?
• docker client запросил у docker engine запуск
container из image hello-world
• docker engine не нашел image hello-world
локально и скачал его с Docker Hub
• docker engine создал и запустил container из
image hello-world и передал docker client вывод
stdout контейнера

* проверяем свойство контейнера - после его остановки все стирается. 
````
docker run -it ubuntu:16.04 /bin/bash
root@8d0234c50f77:/# echo 'Hello world!' > /tmp/file
root@8d0234c50f77:/# exit
>docker run -it ubuntu:16.04 /bin/bash
root@4e727649fb85:/# cat /tmp/file
cat: /tmp/file: No such file or directory
root@4e727649fb85:/# exit
````
*  ищем в истории контейнер с file1
````
docker ps -a --format "table {{.ID}}\t{{.Image}}\t{{.CreatedAt}}\t{{.Names}}"
````
* запоминаем его container_id
````
docker start <u_container_id>
docker attach <u_container_id> # присоединение терминала к контейнеру
````
  * теперь можем посмотреть на /tmp/file1

* docker exec Запускает новый процесс внутри контейнера
````
docker exec -it <u_container_id> bash

ps axf
````
* docker commit Создает image из контейнера
````
docker commit <u_container_id> yourname/ubuntu-tmp-file

> docker images
````

### H/W
• Для сдачи домашнего задания, сохранил
вывод команды docker images в файл docker-monolith/
docker-1.log и закоммитить в репозиторий

### H/W additional * 
 - Сравнить вывод двух следующих команд
>docker inspect <u_container_id>
>docker inspect <u_image_id>
 - На основе вывода команд объяснить чем отличается контейнер от образа. Объяснение дописаны в файл docker- monolith/docker-1.log

 #### краткие объяснения разницы между остановленным контейнером и образом 
 * выдержка из docker-1.log

 Oстановленный контейнер отличается от образа тем, что он сохраняет все изменения в параметрах настройки (напр, IP адрес),
  файлы в файловой системе, метаданные - все "незамороженные" слои. 
 - В выводе команды docker inspect <container_id> можем увидеть разделы host_config, Network_settings, где и сохранен IP адрес. 
 - в выводе docker inspect <image_id> можем найти перечисление слоев. В описании контейнера это не нужно, т.к он ссылается на образ. 
 
 * docker kill - мгновенная остановка контейнера
 ````
>docker ps -q
8d0234c50f77
>docker kill $(docker ps -q) # id контейнера как переменная, результат команды ps -q
8d0234c50f77
 ````

 * удаляем все ненужное командами  docker rm && rmi
 ````
docker rm $(docker ps -a -q) # удалит все незапущенные контейнеры
docker rmi $(docker images -q) # удаление обьразов
 ````

 ### Docker контейнеры
Используем Докер в связке с облачной инфраструктурой Google. 

 * создан новый проект в GCP  с ID  docker-258609
 * установил Gcloud SDK (уже стоял)
````
 gcloud init

 # new configuration
````
*  project - docker-258609

* разрешаем доступ к профилю гугла
````
gcloud auth application-default login
````
* Credentials saved to file: [/home/maxim/.config/gcloud/application_default_credentials.json]


### использование docker machine
````
export GOOGLE_PROJECT=docker-258609
````

* создаем ВМ в GCP
````
docker-machine create --driver google \
--google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-
os-cloud/global/images/family/ubuntu-1604-lts \
--google-machine-type n1-standard-1 \
--google-zone europe-west1-b \
docker-host
````
* в результате создается ВМ docker-host, указанного типа и в указанном регионе, на базе образа ubuntu-1604, и на него ставиться Docker

* check list - просмтор списка запущенных ВМ с докером
````
docker-machine ls 
````
* переключение на работу с docker-host 
  * до настройки использования docker без sudo, была какая-то несуразица, контейнеры не создавались на ВМ в GCP, а только на локальной машине

````
eval $(docker-machine env docker-host)
````

### Повторяем демо из лекции, используя docker-host на GCP
* подробное описание в log.md, коммитить его не стал
* Что выяснили

  * PID namespace. С помощью команды pstree можно увидеть взаимосвязь процессов в контейнере и на хосте, и схему присовения PID

  * net namespace. у машины - хоста интерфейс docker0, через него контейнеры общаются с внешним миром. 
    * используя ifconfig, смотрим настройки сети в контейнерах. В контейнере есть сетевое солединения eth0, адрес которого в подсети у docker0 хоста

    * у хоста создаются интерфейсы veth<id> для каждого контейнера свой, они создаются при запуске контейнера, и убиваются при выходе из него (остановки)

  * демо user namespace. на хосте, смотрим вывод команды docker info. Параметр Docker root dir -  в нем хранятся все файлы, в т.ч. и volume


  * пример с docker run tehbilly
    * запускаем контейнер с утилитой htop

````
docker-host
docker run --rm -it tehbilly/htop

````
  *  Это образ с утилитой htop, она показывает нагрузку и список процессов. Процесс в этом namespace всего один  с PID 1- сама утилита htop


   * Затем запустим то же самое с пространством имен хостовой машины
````
docker run --rm -it --pid host tehbilly/htop
````
Результат: 
запустившись вне namespac-а, и используя PID хоста, утилита видит все системные процессы хоста. Есть доступ ко всем процессам, т.к контейнер запущен с параметром --pid host, то есть вне PID namespace. Таким образом из контейнера можно увидеть процессы  в том числе и самого контейнера с этой утилитой, и других контейнеров. 


### закончили с  демо, возврат к домашней работе

 * созданы ./docker-monolith dockerfile, mongod.conf, db_config, start.sh
   * докерфайл описывает команды при создании образа
   * mongod.conf и db_config содержат конффигурцию и определение переменных для MongoDB
   * start.sh запускает процессы mongod и демон puma
 * работаем в папке docker-monolith
 * в докерфайл добавляем
 ````

FROM ubuntu:16.04

RUN apt-get update                 # команды bash, выполняемые в контейнере в момент подготовки образа
RUN apt-get install -y mongodb-server ruby-full ruby-dev build-essential git
RUN gem install bundler

RUN git clone -b monolith https://github.com/express42/reddit.git
COPY mongod.conf /etc/mongod.conf  # копирование файлов с хоста в контейнер
COPY db_config /reddit/db_config
COPY start.sh /start.sh
RUN cd /reddit && bundle install
RUN chmod 0777 /start.sh
CMD ["/start.sh"]                  # запуск скрипта при запуске контейнера из готового образа
 ```` 
* собираем свой образ, используя команду docker build

````
docker build -t reddit:latest .   #внимание на точку в конце, она указывает на путь до докер-контекста
# флаг -т задает тег для собранного образа

docker images -a # просмотр списка образов, ключ -a  - просмотр также  промежуточных
````
* запуск контейнера
 * возникла проблема: 

Запуск контейнера не происходит на удаленной машине, а только на локальной
Решение:

* Разрешаем докеру выполнение без рута путем добавления текущего пользователя в группу docker
````
sudo usermod -aG docker maxim
````
   * перезагружаем сервис (а лучше весь ПК)
   * команда docker-machine env <hostname> выводит список переменных для подключения

   * используем eval для записи этих переменных, 
````
eval "$(docker machine env docker-host)"
````
* после этого все команды docker будут адресованы удаленной машине docker-host
на ней и нужно создать image reddit,  а не на локальной. Но докерфайл и вся папка докер-монолит - находится на локальной


* проверка результата
docker run --name reddit -d --network=host reddit:latest

* пытаемся открыть в браузере реддит
не открывается, т.к в GCP нет правила файервола для порта 9292
* Добавляем правило файервола
````
gcloud compute firewall-rules create reddit-app \
--allow tcp:9292 \
--target-tags=docker-machine \
--description="Allow PUMA connections" \
--direction=INGRESS
````
* теперь все доступно

### Docker HUB

* ресгитрация (совпадает с аккаунтом докер)

* аутентификация в терминале
````
docker login

````
* отмечаем тег для загрузки в докер хаб
````
docker tag reddit:latest max89k/otus-reddit:1.0

docker images 
# можно увидеть два образа с одинковым ID, один называется reddit, другой - max89k/otus-reddit
````

* загрузка образа на докер хаб
````
docker push <your-login>/otus-reddit:1.0
````
* замечание: так как образ грузится с облака GCP, а не через мой ростелеком, скорость upload впечатляющая

* теперь вопользуемся образом на локальной машине, он скачается с докер хаба
  * замечание: после ошибочных запусков, подтираем лишние контейнеры и образы на лок. машине, используя docker rm, docker rmi

````
docker run --name reddit -d -p 9292:9292 max89k/otus-reddit:1.0 # ключ -d - detached, запуск в фоновом режиме
# ключ -р 9292:9292 - назнгачить порт контейнера порту хоста
````
* приложение запустилось, работает на 127.0.0.1:9292
  * так как запустили docker run без ключа -it а с -d, не перешли в контейнер, он в фоновом режиме

### Проверка контейнера
* изучение логов. После каждого действия на "сайте" видим все обращения в БД, и пр.
````
docker logs reddit -f #Выход по Ctrl-C
````
* запустить в контейнере процесс bash, зайти в него (ключ -it)
````
docker exec -it reddit bash
# зашли в контейнер
ps aux 
kill 21 # убили процесс puma, контейнер упал
````

* перезапуск контейнера (приложение будет работать, все ранее написанные записи сохранятся)
````
docker start reddit
````

* остановим и удалим контейнер , затем запустим заново, без запуска приложения
````
docker stop reddit && docker rm reddit

docker run --name reddit --rm -it max89k/otus-reddit:1.0 bash # нет ключа -p 9292:9292, порт не назначен, приложение не заработает
#в контейнере проверяем список процессов
ps aux
# видим пустоту, ни пума, ни монгодб не запущены
````
* подробная информация об образе
````
docker inspect <your-login>/otus-reddit:1.0

#фрагмент
docker inspect <your-login>/otus-reddit:1.0 -f '{{.ContainerConfig.Cmd}}'
````
* заново запускаем приложение, c нужными параметрами
````
docker run --name reddit -d -p 9292:9292 <your-login>/otus-reddit:1.0
• docker exec -it reddit bash
• mkdir /test1234
• touch /test1234/testfile
• rmdir /opt
• exit
````
* просмотр последних событий (изменений) в контейнере. Список добавленных, удаленных, измененных файлов
````
docker diff reddit
````
* проверяем. что после остановки и удаления контейнера никаких изменения не останется
````
docker stop reddit && docker rm reddit
• docker run --name reddit --rm -it <your-login>/otus-reddit:1.0 bash

ls / #никаких изменений, папка opt на месте, папки test1234 нет
````

### Дополнительное задание - в процессе



### HW docker-3 . Docker-образы, микросервисы. 

План: 

* Разбить наше приложение на несколько компонентов
* Запустить наше микросервисное приложение


1. Подготовка. 
 * подключение к ранее созданному Docker -хосту в облаке google
 * тот хост давно "умер", создаем новый. Имя docker-host3 

 ````
 export GOOGLE_PROJECT=docker-123456

$ docker-machine create --driver google \
 --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
 --google-machine-type n1-standard-1 \
 --google-zone europe-west1-b \
 docker-host3

docker-machine ls

eval $(docker-machine env docker-host3)
 ````

* распаковка архива с приложением в репозиторий, в папку src. 
```
wget https://github.com/express42/reddit/archive/microservices.zipzip \
  && unzip microservices.zip && rm microservices.zip && mv reddit microservices src
```
 
2. Новая структура приложения. Три компонента - post-py (написание постов), comment (комментарии), ui (веб-интерфейс)

* создадим dockerfil-ы для каждого компонента

* post-py/dockerfile пришлось немного переделать, иначе установка компонентов через pip проваливалась. 
````
FROM python:3.6.0-alpine

WORKDIR /app
ADD . /app


RUN apk add --no-cache --virtual build-deps gcc musl-dev \
     && pip install -r /app/requirements.txt

ENV POST_DATABASE_HOST post_db 
ENV POST_DATABASE posts 

CMD ["python3", "post_app.py"]

````

comment/dockerfile
````
FROM ruby:2.2
RUN apt-get update -qq && apt-get install -y build-essential
ENV APP_HOME /app
RUN mkdir $APP_HOME
WORKDIR $APP_HOME
ADD Gemfile* $APP_HOME/
RUN bundle install
COPY . $APP_HOME
ENV COMMENT_DATABASE_HOST comment_db
ENV COMMENT_DATABASE comments
CMD ["puma"]
````

ui/dockerfile
````
FROM ruby:2.2
RUN apt-get update -qq && apt-get install -y build-essential
ENV APP_HOME /app
RUN mkdir $APP_HOME
WORKDIR $APP_HOME
ADD Gemfile* $APP_HOME/
RUN bundle install
ADD . $APP_HOME
ENV POST_SERVICE_HOST post
ENV POST_SERVICE_PORT 5000
ENV COMMENT_SERVICE_HOST comment
ENV COMMENT_SERVICE_PORT 9292
CMD ["puma"]
````

3.Создание образов.
 *  Скачиваем образ для контейнера для mongodb

````
docker pull mongo:latest
````

* сборка образов с компонентами
````
docker build -t max89k/post:1.1 ./post-py
# post 1.0 работал некорректно, поэтому сразу заменен на 1.1

docker build -t max89k/comment:1.0 ./comment
docker build -t max89k/ui:1.0 ./ui
````

* почему сборка ui началась со второго шага - потому что ui и comment создаются на базе ruby:2.2, и первый шаг у них одинаковый (установка build essential ) был выполнен при билде comment, и соответствующий промежуточный слой сохранился в системе. 

4.  запуск контейнеров

* создание специальной bridge-сети для приложения (для того, чтобы можно было использовать сетевые алиасы)
````
docker network create reddit
````

* запуск контейнеров с указанием алиасов
````
docker run -d --network=reddit \
--network-alias=post_db --network-alias=comment_db mongo:latest
docker run -d --network=reddit \
--network-alias=post max89k/post:1.1
docker run -d --network=reddit \
--network-alias=comment max89k/comment:1.0
docker run -d --network=reddit \
-p 9292:9292 max89k/ui:1.0
````
* Сетевые алиасы могут быть использованы для сетевых
соединений, как доменные имена

* перейдя по адресу 34.77.65.110:9292, можно проверить работоспособность приложения

5. Уменьшение размеров образов

* просмотр свойств образов 
````
docker images

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
max89k/post         1.1                 79afd7298ff4        44 minutes ago      198MB
max89k/ui           1.0                 a09e98dd9faf        3 hours ago         784MB
max89k/comment      1.0                 6786004877fa        3 hours ago         781MB
mongo               latest              a0e2e64ac939        10 days ago         364MB
ubuntu              16.04               c6a43cd4801e        10 days ago         123MB
ruby                2.2                 6c8e6f9667b2        20 months ago       715MB
python              3.6.0-alpine        cb178ebbf0f2        2 years ago         88.6MB
````

* размер образа UI можно уменьшить, изменив dockerfile

````
FROM ubuntu:16.04
RUN apt-get update \
    && apt-get install -y ruby-full ruby-dev build-essential \
    && gem install bundler --no-ri --no-rdoc

ENV APP_HOME /app 
RUN mkdir $APP_HOME 
WORKDIR $APP_HOME 
COPY Gemfile* $APP_HOME/
RUN bundle install
COPY . $APP_HOME
ENV POST_SERVICE_HOST post
ENV POST_SERVICE_PORT 5000
ENV COMMENT_SERVICE_HOST comment
ENV COMMENT_SERVICE_PORT 9292
CMD ["puma"]

````

* самое "вредное" для размера образа - установка пакетов отедблыми командами. Нужно поместить в один RUN как можно больше действий по установке. т.к 1 RUn - 1 слой (одно кэширование)

* также встроенный в VScode hadolint советует заменять ADD на COPY, указывать определенную версию пакета, и удалять списки apt-get после установки

* ссылка на Best Practices: https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#sort-multi-line-arguments

6. присоединение Volume для хранения БД постов. 

* удалив-стерев (kill) контейнеры и создав их заново, потеряем все старые посты

* создадим docker volume
````
docker volume create reddit_db
````

* убив контейнеры, запускаем заново. Для контейнера с БД MongoDB указываем параметр -v - использование Volume
````
docker run -d --network=reddit --network-alias=post_db \
--network-alias=comment_db -v reddit_db:/data/db mongo:latest
docker run -d --network=reddit \
--network-alias=post max89k/post:1.1
docker run -d --network=reddit \
--network-alias=comment max89k/comment:1.0
docker run -d --network=reddit \
-p 9292:9292 max89k/ui:2.0
````

* можно проверить, что после стирания контейнеров, и создания новых, старые посты сохраняются, т.к Data Volume не зависит от стирания присоединенного к ней контейнера. 



## Homework Docker-4 : networks, docker compose
## Отчет HW4

* создал ветку docker-4

* подключился к машине docker-host3 в своем докер-проекте в GCP 

````
docker-machine ls
NAME           ACTIVE   DRIVER   STATE     URL                      SWARM   DOCKER     ERRORS

docker-host3   *        google   Running   tcp://34.77.54.33:2376           v19.03.5 

eval $(docker-machine env docker-host3)
````
### Host & none

* запустил контейнер на базе joffotron/docker-net-tools с сетью none. Ifconfig показывает наличие loopback, и больше ничего.

* запустил из того же образа контейнер с  сетью host
   * видим сетевые интерфейсы br-cebe293f72df (172.18.0.1), docker0 (172.17.0.1), ens4 (10.132.0.3)

* сравним с выводом команды docker-machine ssh docker-host3 ifconfig. Вывод такoй же. 


* запустим несколько контейнеров с nginx (3 штуки). Команда docker ps показывает, что "жив" только первый. Это происходит, т.к сеть типа host предоставляет всем контейнерам один сетевой namespace, один адрес, и несколько nginx не могут сосуществовать в нем, слушая один и тот же порт. 

* подключился по ssh на docker-host3
```
docker-machine ssh docker-host3
```
* команда ln -s /var/run/docker/netns /var/run/netns пробрасывает симлинку из списка net-namespaces docker-a в область, доступную для утилиты ip

* запускаем утилиту ip netns - видим namespace default

* запустим три контейнера с сетями none (2шт) и host, посмотрим результат
```
docker run --network host -d nginx && docker run --network none -d nginx && docker run --network none -d nginx

docker ps

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
f21101d7783c        nginx               "nginx -g 'daemon of…"   6 seconds ago       Up 5 seconds                            determined_golick
755219d02c08        nginx               "nginx -g 'daemon of…"   8 seconds ago       Up 6 seconds                            serene_feistel
14540dd7e677        nginx               "nginx -g 'daemon of…"   9 seconds ago       Up 7 seconds                            elegant_kowalevski
```

* в данном случае все контейнеры запустились. 

* перешел на docker-host3 по ssh, проверил список net-namespace. Создались еще два (для каждой сети none)
```
sudo ip netns
dbdd21c2aef4
aaaf1cafa3ad
default
```

* команда ip netns exec <net_namespace_name> hostnamectl - выводит сетевое имя для устройства в заданном сетевом namespace

```
Static hostname: docker-host3
         Icon name: computer-vm
           Chassis: vm
        Machine ID: 8edc9b12ad0f16bb20b2af9ab6fe8ebb
           Boot ID: aaef559f9fab40c491cb7aa9f368f9bf
    Virtualization: kvm
  Operating System: Ubuntu 16.04.6 LTS
            Kernel: Linux 4.15.0-1050-gcp
      Architecture: x86-64
```


### Bridge network driver
* cheatsheet
````
Usage:  docker network COMMAND

Manage networks

Commands:
  connect     Connect a container to a network
  create      Create a network
  disconnect  Disconnect a container from a network
  inspect     Display detailed information on one or more networks
  ls          List networks
  prune       Remove all unused networks
  rm          Remove one or more networks
````

* создание bridge-сети в docker (уже была такая с прошлого д/з, удалил и создал заново)


````
docker network create reddit --driver bridge

Error response from daemon: network with name reddit already exists

docker network rm reddit

docker network create reddit --driver bridge
````

* запустили контейнеры для приложения reddit

````
docker run -d --network=reddit mongo:latest
docker run -d --network=reddit max89k/post:1.0
docker run -d --network=reddit max89k/comment:1.0
docker run -d --network=reddit -p 9292:9292 max89k/ui:1.0
````

* после запуска видим, что приложение не работает должным образом. Нет связи между микросервисами. Они ссылаются друг на друга по dns-именам. которые прописаны в dockerfile через ENV ( переменные окружения). Встроенный в докер DNS не знает об этих именах

* решение проблемы - присвоение контейнерам имен либо сетевых алиасов. Это можно сделать при запуске.

* убьем старые, создадим новые

````
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
bc272d5d0630ac5027c74eabed88b832901b26a665240d8e7b52afde730c1bf2
docker run -d --network=reddit --network-alias=post max89k/post:1.1
a67650bc95d1ebd9f746c8c551d419c96ec05f56e60898ceaf91be4505803174
docker run -d --network=reddit --network-alias=comment max89k/comment:1.0
15adc861934d213c34995c0dc213a9769d45ba6ee19c5f4cf301bfa1e16cb23c
docker run -d --network=reddit -p 9292:9292 max89k/ui:1.0
b222cfecd1f9f864d5f4547a0d7bf985a64dd20bb271e10219bfccb0de8e93e9
````

* теперь все работает корректно

#### запуск reddit в двух bridge-сетях

UI, comment, post - находятся в сети front_net (10.0.1.0/24)

DB, comment, post - находятся в сети back_net (10.0.2.0/24)

UI и DB не имеют доступа друг к другу

* создание сетей
````
docker kill $(docker ps -q)

docker network create back_net --subnet=10.0.2.0/24
docker network create front_net --subnet=10.0.1.0/24
````

* запуск новых контейнеров в новых сетях
````
docker run -d --network=front_net -p 9292:9292 --name ui  max89k/ui:1.0
docker run -d --network=back_net --name comment  max89k/comment:1.0
docker run -d --network=back_net --name post  max89k/post:1.1
docker run -d --network=back_net --name mongo_db --network-alias=post_db --network-alias=comment_db mongo:latest
````

* при таком запуске можно подключить контейнер только к одной сети, поэтому UI "не увидит" post и comment. приложение не заработает

* нужно поместить контейнеры post & comment в обе сети, используя команду docker network connect <network> <container>. К контейнеру можно "обращаться" по заданному имени. 
````
docker network connect front_net post
docker network connect front_net comment
````

### проверка сетевого стека на докер-хосте

* используем утилиты bridge-utils. Стаивм их, зайдя по ssh на докер-хост. все действия выполняем с него

````
docker-machine ssh docker-host
sudo apt-get update && sudo apt-get install bridge-utils
````

* просмотр списка сетей в docker
````
docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
63489bc7665e        back_net            bridge              local
979d25f8fa22        bridge              bridge              local
68600fb7a02e        front_net           bridge              local
d1bce0c2954d        host                host                local
5c6cfc98ce97        none                null                local
309aaa3dc4e7        reddit              bridge              local
````
* команда для отображения всех сетевых интерфейсов типа brigde

````
ifonfig | grep br
````
* видим три bridge сети: созданные нами front, back, и дефолтную. 

* просмотр подробной информации о bridge-интерфейсе с помощью утилиты brctl (http://xgu.ru/wiki/man:brctl)
````
brctl show br-63489bc7665e

br-63489bc7665e         8000.02425cdd6147       no              veth77508d6
                                                        veth8794a8f
                                                        vethcb950d1
````
* видим. что данная сеть (это back_net) подключена к трем veth-интерфесам. veth-интерфейсы - это те части виртуальных пар
интерфейсов, которые лежат в сетевом пространстве хоста и также отображаются в ifconfig. Вторые их части лежат внутри контейнеров


* просмотр iptables

````
sudo iptables -nL -t nat

Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  10.0.1.0/24          0.0.0.0/0           
MASQUERADE  all  --  10.0.2.0/24          0.0.0.0/0           
MASQUERADE  all  --  172.18.0.0/16        0.0.0.0/0           
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0           
MASQUERADE  tcp  --  10.0.1.2             10.0.1.2             tcp dpt:9292

Chain DOCKER (2 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:9292 to:10.0.1.2:9292
````

* цепочка postrouting содержит в себе правила для выпуска во внешнюю сеть контейнеров из bridge-сетей.

##### результат публикации портов
* контейнер UI запущен с параметром -p 9292:9292 - публикация порта для доступа к нему снаружи. 
* смотрим вывод iptables, цепочка DOCKER, правила DNAT - они отвечают за перенаправления трафика на адреса конкретных контейнеров

* процесс докер-прокси. Должен быть виден в списке запущенных. Видно информацию о том, какой порт он слушает. 
````
ps ax | grep docker-proxy
11075 ?        Sl     0:00 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 9292 -container-ip 10.0.1.2 -container-port 9292
````

### Docker Compose

0. cheatsheet
````
Commands:
  build              Build or rebuild services
  bundle             Generate a Docker bundle from the Compose file
  config             Validate and view the Compose file
  create             Create services
  down               Stop and remove containers, networks, images, and volumes
  events             Receive real time events from containers
  exec               Execute a command in a running container
  images             List images
  kill               Kill containers
  logs               View output from containers
  pause              Pause services
  port               Print the public port for a port binding
  ps                 List containers
  pull               Pull service images
  push               Push service images
  restart            Restart services
  rm                 Remove stopped containers
  run                Run a one-off command
  scale              Set number of containers for a service
  start              Start services
  stop               Stop services
  top                Display the running processes
  unpause            Unpause services
  up                 Create and start containers
  ````
 
1. Установка утилиты на локальный ПК
```
pip install docker-compose

docker-compose version
docker-compose version 1.25.0, build b42d419
docker-py version: 4.1.0
CPython version: 2.7.17
OpenSSL version: OpenSSL 1.1.1  11 Sep 2018
```
2. Сборка приложения reddit с помощью docker-compose

* создан файл src/docker-compose.yml, описывающий наши контейнеры

* в yml файле образы указаны с ипользованием переменной окружения USERNAME, нужно ее экспортировать перед запуском
````
export USERNAME=max89k

# ключ -d запускает контейнеры в detached-режиме, иначе консоль будет забита выводом всех контейнеров
docker-compose up -d

# выводит информацию о запущенных контейнерах
docker-compose ps

    Name                  Command             State           Ports         
----------------------------------------------------------------------------
src_comment_1   puma                          Up                            
src_post_1      python3 post_app.py           Up                            
src_post_db_1   docker-entrypoint.sh mongod   Up      27017/tcp             
src_ui_1        puma                          Up      0.0.0.0:9292->9292/tcp
````

3. Запуск приложения с помощью docker-compose
````
docker-compose up -d
````

* приложение работает


4. Изменение конфигурации через редактирование docker-compose.yml
 * команда docker-compose down останавливает и стирает контейнеры и всю их инфраструктуру (сети, и пр.)

 * добавим схему с двумя сетями - front_net, back_net, и множеством сетевых алиасов

   * в разделах services, <имя_контейнера>, networks меняем reddit front_net для ui , back_net для mongodb, и обе сети для post , comment.

    * после docker-compose up приложение работает

 * добавим сетевые алиасы, запись будет в виде:
 ````
 networks:
      front_net:
         aliases:
           - post
      back_net:
         aliases:
           - post
 ````
* приложение работает



 * параметризуем порт публикации UI, версии сервисов, через переменные окружения. 
    * переменная UI_PORT разбита на две части (9292 : 9292), чтобы избежать проблем из-за символа двоеточия. 

    * запуск контенеров с ключем --env-file
    ````
    docker-compose --env-file ./data.env up -d
    ````
    * останавливать  контейнеры тоже нужно с ключем --env-file

````
docker-compose --env-file ./data.env  down
````

 * параметризованные параметры записаны в data.env, в коммит пойдет его копия data.env.example


 * все контейнеры, запущенные через compose, имеют в имени префикс src - имя папки, где лежит yml файл. 
 * префикс можно поменять, задав опицю -p (--project-name) при запуске контейнера
 ````
 docker-compose --env-file ./data.env -p HW16 up -d

 Creating network "hw16_front_net" with the default driver

Creating network "hw16_back_net" with the default driver
Creating volume "hw16_post_db" with default driver
Creating hw16_ui_1      ... done
Creating hw16_post_db_1 ... done
Creating hw16_post_1    ... done
Creating hw16_comment_1 ... done
=======
 Creating network "hw16_back_net" with the default driver
 Creating volume "hw16_post_db" with default driver
 Creating hw16_ui_1      ... done
 Creating hw16_post_db_1 ... done
 Creating hw16_post_1    ... done
 Creating hw16_comment_1 ... done

 ````

 * это неудобно, т.к для стирания по команде docker-compose down надо опять указывать и --env-file, и имя проекта. 
 ````
docker-compose --env-file ./data.env -p HW16  down
 ````


 * можно добавить переменную COMPOSE_PROJECT_NAME в .env файл

 * нельзя менять содержимое docker-compose.yml , data.env при запущенных контейнерах, т.к. docker-compose будет заглядывать в него при попытке "загасить" по команде docker-compose down,  и ругаться


 ### Homework 15. Gitlab-CI

ПОдготовка: 
* создана ветка gitlab-ci-1 в основном репозитории

* создали ВМ в гугле (имя gitlab-ci), через докер-машин, docker-ce ставится автоматом

* на ней ставим docker-compose

* созданы необходимые папки, и файл docker-compose.yml
```
mkdir -p /srv/gitlab/config /srv/gitlab/data /srv/gitlab/logs
cd /srv/gitlab/
touch docker-compose.yml
```
Заполняем docker-compose.yml

```
web:
  image: 'gitlab/gitlab-ce:latest'
  restart: always
  hostname: 'gitlab.example.com'
  environment:
    GITLAB_OMNIBUS_CONFIG: |
      external_url 'http://<YOUR-VM-IP>'
  ports:
    - '80:80'
    - '443:443'
    - '2222:22'
  volumes:
    - '/srv/gitlab/config:/etc/gitlab'
    - '/srv/gitlab/logs:/var/log/gitlab'
    - '/srv/gitlab/data:/var/opt/gitlab'
```

* после запуска контейнера из файла (docker-compose up -d) можем проверить доступность GitlabCI  по IP адресу ВМ
 

* указываем пароль для рута

* переход в админ панель http://<IP>/admin

* выключаем регистрацию новых пользователей
*  Settings -> Sign up Restrictions -> Sign-up enabled == OFF

* верхняя панель Groups- explore - add group. Создал homework-15. URL http://<ip>/homework-15

* из меню группы - add project, опции: шаблон blank, имя example. URL http://<ip>/homework-15/example

* копитруем туда свой репозиторий c локальной ВМ
```
# действуем в своей домашней ПК
git checkout gitlab-ci-1
git remote add gitlab http://35.246.233.216/homework-15/example
git push gitlab gitlab-ci-1
```

* видим, что в проекте example появился репозиторий

### CI\CD pipeline

1. На домашней машине создали в репозитории файл .gitlab-ci.yml

2. Закоммитили его, и запушили на гитлаб-машину в облаке
````
git add .gitlab-ci.yml 
git commit -m 'add pipeline definition'
[gitlab-ci-1 919c4b9] add pipeline definition
 1 file changed, 24 insertions(+)
 create mode 100644 .gitlab-ci.yml
git push gitlab gitlab-ci-1
````

3. Теперь если перейти в раздел CI/CD мы увидим, что пайплайн готов к
запуску
Но находится в статусе pending / stuck так как у нас нет runner

3.1. Запустим Runner и зарегистрируем его в интерактивном
режиме

4. Перед тем, как запускать и регистрировать runner
нужно получить токен

Settings -> CI\CD Specific runners - Specific runners manually


На сервере, где работает Gitlab CI выполним команду:
docker run -d --name gitlab-runner --restart always \
-v /srv/gitlab-runner/config:/etc/gitlab-runner \
-v /var/run/docker.sock:/var/run/docker.sock \
gitlab/gitlab-runner:latest

После запуска Runner нужно зарегистрировать, это можно сделать командой:
docker exec -it gitlab-runner gitlab-runner register --run-untagged --locked=false

вводим URL, token, описание, теги (linux, ubuntu, docker). и исполнителя (docker)

новый раннер видим в меню Settings - > CI\CD -> Specific Runners

Runner запускается по коммиту. Можно посмотреть на вывод-промежуточные результаты


5. добавил исходный код реддита в репозиторий
````
git clone https://github.com/express42/reddit.git && rm -rf ./reddit/.git
git add reddit/
git commit -m “Add reddit app”
git push gitlab gitlab-ci-1
````
* изменил описание пайплайна в .юмл
````
image: ruby:2.4.2

variables:
    DATABASE_URL: 'mongodb://mongo/user_posts'

test_unit_job:
  stage: test
  services:
    - mongo:latest
  script:
    - ruby simpletest.rb
````
* нужно создать скрипт simpletest.rb
```
require_relative './app'
require 'test/unit'
require 'rack/test'

set :environment, :test

class MyAppTest < Test::Unit::TestCase
  include Rack::Test::Methods

  def app
    Sinatra::Application
  end

  def test_get_request
    get '/'
    assert last_response.ok?
  end
end
```
*  добавил строку в reddit/gemfile 

```
gem 'rack-test'
```

* запушил все в гитлаб

* конвейер запустился, тест прошел. 


### Описание окружений

* Изменим .gitlab-ci.yml таким образом, чтобы job deploy
стал определением окружения dev, на которое условно
будет выкатываться каждое изменение в коде проекта

* переименован stage-deploy в review ; deploy job -> deploy_dev_job

* добавил пункт environment
```
environment:
name: dev
url: http://dev.example.com
```
* после успешного прохождения конвейера появится пункт environment в GUI (Operations -> Environments)

* Определим два новых этапа: stage и production, первый будет
содержать job имитирующий выкатку на staging окружение, второй
на production окружение.

* Определим эти job таким образом, чтобы они запускались с кнопки
````
staging:
   stage: stage
   when: manual   # этот параметр определяет запуск с кнопки
   script: 
      - echo 'Deploy'
   environment:
      name: stage
      url: https://beta.example.com

production:
   stage: production
   when: manual # этот параметр определяет запуск с кнопки
   script:
       - echo 'Deploy'
   environment: 
       name: production
       url: https://example.com
````


* после завершения работы пайплайна появятся два новых окружения

* на "схеме" пайплайна в GUI видим в пунктах Stage и Production шестереночки - ручной запуск. 

### ввод условия и ограничений
* изменил .yml файл, для добавления в job-ы stage и prod условия - наличия semver тега формата 1.2.30
```
stage:
stage: stage
when: manual
only:
- /^\d+\.\d+\.\d+/ #регулярное выражение
```

* теперь пуше без нужного тега запустится пайплайн без stage и prod.
* для включения этих пунктов нужно добавить тег после коммита, перед пушем
```
git commit -a -m ‘#10 added great feature’
git tag 2.4.69
git push gitlab gitlab-ci-1 --tags
```


### Динамические окружения
* добавим job для определения динамического окружения
```
branch review:
stage: review
script: echo "Deploy to $CI_ENVIRONMENT_SLUG"
environment:
name: branch/$CI_COMMIT_REF_NAME
url: http://$CI_ENVIRONMENT_SLUG.example.com
only:
- branches
except:
- master
```

* теперь, создав сторонние ветки в репозитоприи (bugfix, feature) получим новые окружения с именами review/bugfix, review/feature

* в папке gitlab-ci репозиторий создан docker-compose.yml для развертывания gitlab-сi 

 * HINT если назвать файл с переменнеыми .env, то он подхватится по умолчанию.

 * можно добавить переменную COMPOSE_PROJECT_NAME в .env файл

 * нельзя менять содержимое docker-compose.yml , data.env при запущенных контейнерах, т.к. docker-compose будет заглядывать в него при попытке "загасить" по команде docker-compose down,  и ругаться



## Homework 16 Monitoring -1 

Что сделано: 

создана ветка monitoring-1

### подготовка окружения

* будем использовать ВМ в GCP

* prometheus использует tcp порт 9090 и puma использует 9292;  создадим правила файервола для gcp
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

* Prometheus будет запущен внутри контейнера, для начала воспользуемся готовым образом dockerHub prometheus:v2.1.0. 

```
# открыли порты 9090, запуск в detached режиме
docker run --rm -p 9090:9090 -d --name prometheus prometheus:v2.1.0
```

* узнаем адрес докер-хоста; открыли в браузере этот ip : 9090. работает

```
docker-machine ip docker-host
```

* выбрана метрика prometheus_build_info. output: {branch="HEAD",goversion="go1.9.2",instance="localhost:9090",job="prometheus",revision="85f23d82a045d103ea7f3c89a91fba4a93e6367a",version="2.1.0"} 1

* раздел Targets - представляют собой системы или процессы, за
которыми следит Prometheus. Помним, что Prometheus является
pull системой, поэтому он постоянно делает HTTP запросы на
имеющиеся у него адреса (endpoints). Посмотрим текущий список
целей

* активна цель localhost:9090/metrics. перейдя по адресу host:port/metrics можно посмотреть на информацию, собираемую прометиусом.

* тестовый контейнер prometheus оcтановили

### переупорядочим структуру директорий репозитория

* создана папка docker, в нее перенесены docker-monolith, src, reddit

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
export USER_NAME=max89k
docker build -t $USER_NAME/prometheus .
```

* собираем образы max89k/ui, max89k/post, max89k/comment c помощью скриптов docker_build.sh, лежащими в папках docker/docker-monolith/src/<microservice_name>

* редактируем docker/docker-compose.yml
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

* ui_health - состояние сервиса UI. вводим над кнопкой Execute

* тормозим сервис post ипроверяем, , как изменится статус ui сервиса, который зависим от post.
``` 
docker-compose stop post

Stopping my_project_post_1 ... done
```
* healthcheck "упал", видно на графике

* ищем проблему - смотрим статусы сервисов
```
ui_health_post_availability
ui_healt_comment_availability

```

* видим, что сервис post подвел. Исправим 
```
docker-compose start post
```

### Exporters

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

* про использование node exporter https://github.com/prometheus/node_exporter/blob/master/README.md

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


## Homework №18. Логирование и распределенная трассировка

### План
 - Сбор неструктурированных логов
 - Визуализация логов
 - Сбор структурированных логов
 - Распределенная трасировка

* Обновлен код приложения из репозитория `https://github.com/express42/reddit/tree/logging`
```
$ git clone --branch=logging  https://github.com/express42/reddit.git
```
* Выполнена сборку образов
```
for i in ui post-py comment; do cd src/$i; bash docker_build.sh; cd -; done
```

## Подготовка окружения
* Создадим Docker хост в GCE и настроим локальное окружение
```
$ export GOOGLE_PROJECT=вщслук-258609

$ docker-machine create --driver google \
  --google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
  --google-machine-type n1-standard-1 \
  --google-zone europe-west1-b \
  --google-open-port 5601/tcp \
  --google-open-port 9292/tcp \
  --google-open-port 9411/tcp \
  logging2

$ eval $(docker-machine env logging2)

$ docker-machine ip logging2
34.67.154.64
```

## Логирование Docker контейнеров
## Elastic Stack: ELK - EFK
* Elastic стек включает в себя 3 осовных компонента:
 - ElasticSearch (TSDB и поисковый движок для хранения данных)
 - Logstash (для агрегации и трансформации данных)
 - Kibana (для визуализации)

* Вместо Logstash мы будем использовать Fluentd, получим EFK.
* В каталоге `docker` создадим файл `docker-compose-logging.yml`
```
wget https://raw.githubusercontent.com/express42/otus-snippets/master/hw-25/docker-compose-logging1.yml
```

## Fluentd
* Создадим в вашем проекте *microservices* директорию `logging/fluentd`, внутри создадим `Dockerfile`
```
FROM fluent/fluentd:v0.12
RUN gem install fluent-plugin-elasticsearch --no-rdoc --no-ri --version 1.9.5
RUN gem install fluent-plugin-grok-parser --no-rdoc --no-ri --version 1.0.0
ADD fluent.conf /fluentd/etc
```

* В директории `logging/fluentd` создадан файл конфигурации `fluent.conf`
```
wget https://raw.githubusercontent.com/express42/otus-snippets/master/hw-25/fluent.conf
```

* Соберем docker image для fluentd
```
docker build -t $USER_NAME/fluentd .
```

## Структурированные логи
* Лог-сообщения также должны иметь понятный для выбранной системы логирования формат, чтобы избежать ненужной траты ресурсов на преобразование данных в нужный вид. Структурированные логи мы рассмотрим на примере сервиса post.

* Правим `.env` файл и меняем теги приложения на logging, после чего запустим сервисы приложения `$ docker-compose up -d` и выполним команду для просмотра логов post сервиса `docker-compose logs -f post`
```
$ docker-compose logs -f post
Attaching to docker_post_1
post_1     | {"addr": "172.28.0.3", "event": "request", "level": "info", "method": "GET", "path": "/healthcheck?", "request_id": null, "response_status": 200, "service"
: "post", "timestamp": "2019-12-19 20:38:19"}
post_1     | {"addr": "172.28.0.3", "event": "request", "level": "info", "method": "GET", "path": "/healthcheck?", "request_id": null, "response_status": 200, "service"
: "post", "timestamp": "2019-12-19 20:38:19"}
post_1     | {"addr": "172.28.0.3", "event": "request", "level": "info", "method": "GET", "path": "/healthcheck?", "request_id": null, "response_status": 200, "service"
: "post", "timestamp": "2019-12-19 20:38:24"}
post_1     | {"addr": "172.28.0.3", "event": "request", "level": "info", "method": "GET", "path": "/healthcheck?", "request_id": null, "response_status": 200, "service"
: "post", "timestamp": "2019-12-19 20:38:24"}
post_1     | {"addr": "172.28.0.3", "event": "request", "level": "info", "method": "GET", "path": "/healthcheck?", "request_id": null, "response_status": 200, "service"
: "post", "timestamp": "2019-12-19 20:38:29"}
```
* Создадим несколько новых постов и посмотрим, как это отразится в логах
```
post_1     | {"event": "post_create", "level": "info", "message": "Successfully created a new post", "params": {"link": "http://www2.ru", "title": "test post2"}, "request_id": "48c1177e-bf6d-4a3f-8f21-e4660925e89c", "service": "post", "timestamp": "2019-12-19 20:43:47"}
post_1     | {"addr": "172.28.0.3", "event": "request", "level": "info", "method": "POST", "path": "/add_post?", "request_id": "48c1177e-bf6d-4a3f-8f21-e4660925e89c", "response_status": 200, "service": "post", "timestamp": "2019-12-19 20:43:47"}
post_1     | {"event": "find_all_posts", "level": "info", "message": "Successfully retrieved all posts from the database", "params": {}, "request_id": "433f4c67-11cf-4055-871e-2503b4f926f7", "service": "post", "timestamp": "2019-12-19 20:43:47"}
post_1     | {"addr": "172.28.0.3", "event": "request", "level": "info", "method": "GET", "path": "/posts?", "request_id": "433f4c67-11cf-4055-871e-2503b4f926f7", "response_status": 200, "service": "post", "timestamp": "2019-12-19 20:43:47"}
```


## Отправка логов во Fluentd
* Для отправки логов во Fluentd используем docker драйвер fluentd `https://docs.docker.com/config/containers/logging/fluentd/`

* Определим драйвер для логирования для сервиса post внутри `docker-compose.yml`
```
services:
  post:
    image: ${USER_NAME}/post
    environment:
      - POST_DATABASE_HOST=post_db
      - POST_DATABASE=posts
    depends_on:
      - post_db
    ports:
      - "5000:5000"
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: service.post
```

* Перезапустим сервисы приложения и логгирования
```
$ docker-compose -f docker-compose-logging.yml up -d
$ docker-compose down
$ docker-compose up -d
```

## Kibana issue
* Проверяем запуск Kibana `http://34.67.154.64:5601`
```
Kibana server is not ready yet
```
* Проверим запущенные контейнеры
```
$ docker-compose -f docker-compose-logging.yml ps

         Name                       Command                State                               Ports
--------------------------------------------------------------------------------------------------------------------------------
logging_elasticsearch_1   /usr/local/bin/docker-entr ...   Exit 78
logging_fluentd_1         tini -- /bin/entrypoint.sh ...   Up        0.0.0.0:24224->24224/tcp, 0.0.0.0:24224->24224/udp, 5140/tcp
logging_kibana_1          /usr/local/bin/dumb-init - ...   Up        0.0.0.0:5601->5601/tcp
```

* Посмотрим, что случилось с elasticsearch
```
$ docker-compose -f docker-compose-logging.yml logs elasticsearch

elasticsearch_1  | ERROR: [2] bootstrap checks failed
elasticsearch_1  | [1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
elasticsearch_1  | [2]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
```
* Решение проблемы тут `https://github.com/deviantony/docker-elk/issues/243`
* Лечим проблему, дополним конфиг переменными
```
    image: elasticsearch:7.5.0
    environment:
      - node.name=elasticsearch
      - cluster.name=docker-cluster
      - node.master=true
      - cluster.initial_master_nodes=elasticsearch
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
```
* убираем жалобу на недостаток памяти max_maxp_count
```
docker-machine ssh logging2 sudo sysctl -w vm.max_map_count=262144
```

* Перезапустим логгинг, проверим
```

$ docker-compose -f docker-compose-logging.yml ps
         Name                       Command               State                              Ports
------------------------------------------------------------------------------------------------------------------------------
logging_elasticsearch_1   /usr/local/bin/docker-entr ...   Up      0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp
logging_fluentd_1         tini -- /bin/entrypoint.sh ...   Up      0.0.0.0:24224->24224/tcp, 0.0.0.0:24224->24224/udp, 5140/tcp
logging_kibana_1          /usr/local/bin/dumb-init - ...   Up      0.0.0.0:5601->5601/tcp
```
### Фильтры
* Добавим фильтр для парсинга json логов, приходящих от post сервиса, в конфиг `logging/fluentd/fluent.conf`
```
<filter service.post>
@type parser
format json
key_name log
</filter>
```
* Перезапустим сервис fluentd
```
$ docker-compose -f docker-compose-logging.yml up -d fluentd
$ docker-compose -f docker-compose-logging.yml ps
         Name                       Command               State                              Ports
------------------------------------------------------------------------------------------------------------------------------
logging_elasticsearch_1   /usr/local/bin/docker-entr ...   Up      0.0.0.0:9200->9200/tcp, 9300/tcp
logging_fluentd_1         tini -- /bin/entrypoint.sh ...   Up      0.0.0.0:24224->24224/tcp, 0.0.0.0:24224->24224/udp, 5140/tcp
logging_kibana_1          /usr/local/bin/dumb-init - ...   Up      0.0.0.0:5601->5601/tcp
```

* Для корректного разбора логов необходимо использовать более высокую версию `fluent-plugin-elasticsearch`, скорректирован `Dockerfile` до версии 1.18.1. 

## Неструктурированные логи
* Неструктурированные логи отличаются отсутствием четкой структуры данных. Также часто бывает, что формат лог-сообщений не подстроен под систему централизованного логирования, что существенно увеличивает затраты вычислительных и временных ресурсов на обработку данных и выделение нужной информации. На примере сервиса ui рассмотрим пример логов с неудобным форматом сообщений.

### Логирование UI сервиса
* По аналогии с post сервисом определим для ui сервиса драйвер для логирования fluentd в `docker/docker-compose.yml`
```
 ui:
    image: ${USERNAME}/ui:${UI_VERSION}
    environment:
      - POST_SERVICE_HOST=post
      - POST_SERVICE_PORT=5000
      - COMMENT_SERVICE_HOST=comment
      - COMMENT_SERVICE_PORT=9292
    ports:
      - ${APP_PORT}/tcp
    depends_on:
      - post
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: service.ui
    networks:
      - front_net
```

* Перезапустим ui сервис
```
$ docker-compose stop ui
$ docker-compose rm ui
$ docker-compose up -d
```

### Парсинг логов ui-сервиса
* Добавим в `/docker/fluentd/fluent.conf` регулярное выражение для разбора не структурированных логов
```
<filter service.ui>
  @type parser
  format /\[(?<time>[^\]]*)\]  (?<level>\S+) (?<user>\S+)[\W]*service=(?<service>\S+)[\W]*event=(?<event>\S+)[\W]*(?:path=(?<path>\S+)[\W]*)?request_id=(?<request_id>\S+)[\W]*(?:remote_addr=(?<remote_addr>\S+)[\W]*)?(?:method= (?<method>\S+)[\W]*)?(?:response_status=(?<response_status>\S+)[\W]*)?(?:message='(?<message>[^\']*)[\W]*)?/
  key_name log
</filter>
```

* Пересоберем образ fluentd и перезапустим kibana для применения изменений

```
$ docker-compose -f docker-compose-logging.yml down
$ docker-compose -f docker-compose-logging.yml up -d
```

* Созданные регулярки могут иметь ошибки, их сложно менять и невозможно читать. Для облегчения задачи парсинга вместо стандартных регулярок можно использовать grok-шаблоны. По-сути grok’и - это именованные шаблоны регулярных выражений (очень похоже на функции). Можно использовать готовый regexp, просто сославшись на него как на функцию `fluent.conf`.
```
<filter service.ui>
@type parser
format grok
grok_pattern %{RUBY_LOGGER}
key_name log
</filter>
```


## Распределенный трейсинг. Zipkin

* Добавим в `compose-файл` для сервисов логирования сервис распределенного трейсинга *Zipkin*
```
services:
  zipkin:
    image: openzipkin/zipkin
    ports:
      - "9411:9411"
```
* Правим наш `docker-compose.yml`, добавив для каждого сервиса поддержку ENV переменных и зададим параметризованный параметр ZIPKIN_ENABLED
```
environment:
- ZIPKIN_ENABLED=${ZIPKIN_ENABLED}
```

* В .env файле укажем `ZIPKIN_ENABLED=true` и перевыкатим приложение `docker-compose up -d`

* Zipkin должен быть в одной сети с приложениями, поэтому нужно объявить эти сети в
`docker-compose-logging.yml`

```
services:
  zipkin:
    image: openzipkin/zipkin
    ports:
      - "9411:9411"
    networks:
      - front_net
      - back_net
```

* Пересоздадим наши сервисы, откроем Zipkin WEB UI на порту 9411, пока никаких трейсов поиск не выдает, т.к. никаких запросов нашему приложению еще не поступало.

```
$ docker-compose -f docker-compose-logging.yml -f docker-compose.yml down
$ docker-compose -f docker-compose-logging.yml -f docker-compose.yml up -d
```

* Откроем главную страницу приложения и обновим ее несколько раз. Заглянув затем в UI Zipkin, увидим несколько трейсов. Нажмем на один из трейсов, чтобы посмотреть, как запрос шел через нашу систему микросервисов и каково общее время обработки запроса у нашего приложения при запросе главной страницы.
* Видим, что первым делом наш запрос попал к ui сервису, который смог обработать наш запрос за суммарное время равное 187.566 ms.
* Из этих 187 ms ушло 134.155ms на то чтобы ui мог направить запрос post сервису по пути /posts и получить от него ответ в виде списка постов. Post сервис в свою очередь использовал функцию обращения к БД за списком постов, на что ушло 4.827 ms.
