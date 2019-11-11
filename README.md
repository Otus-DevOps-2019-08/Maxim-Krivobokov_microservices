# Maxim-Krivobokov_microservices
Maxim-Krivobokov microservices repository

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
