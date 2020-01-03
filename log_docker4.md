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

* сравним с выводом команды docker-machine ssh docker-host3 ifconfig. Вывод такй же. 


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
 ````

 * это неудобно, т.к для стирания по команде docker-compose down надо опять указывать и --env-file, и имя проекта. 
 ````
docker-compose --env-file ./data.env -p HW16  down
 ````

 * можно добавить переменную COMPOSE_PROJECT_NAME в .env файл

 * нельзя менять содержимое docker-compose.yml , data.env при запущенных контейнерах, т.к. docker-compose будет заглядывать в него при попытке "загасить" по команде docker-compose down,  и ругаться