## report

* создали ВМ в гугле, черех докер-машин
* на ней ставим docker-compose

* созданы необходимые папки, и файл docker-compose.yml

mkdir -p /srv/gitlab/config /srv/gitlab/data /srv/gitlab/logs
# cd /srv/gitlab/
# touch docker-compose.yml

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
```

```

* переход в админ панель http://<IP>/admin

* выключаем регистрацию новых пользователей
*  Settings -> Sign up Restrictions -> Sign-up enabled == OFF

* верхняя панель Groups- explore - add group. Создал homework-15. URL http://<ip>/homework-15

* из меню группы - add project, blank, имя example. URL http://<ip>/homework-15/example

* копитруем туда свой репозиторий c локальной ВМ
```
# действуем в своей домашней ПК
git checkout gitlab-ci-1
git remote add gitlab http://35.246.233.216/homework-15/example
git push gitlab gitlab-ci-1
```

* видим в проекте example появился репозиторий

### CI\CD pipeline

На домашней машине создали в репозитории файл .gitlab-ci.yml

Закоммитили его, и запушили на гитлаб-машину в облаке
````
git add .gitlab-ci.yml 
git commit -m 'add pipeline definition'
[gitlab-ci-1 919c4b9] add pipeline definition
 1 file changed, 24 insertions(+)
 create mode 100644 .gitlab-ci.yml
git push gitlab gitlab-ci-1
````

Теперь если перейти в раздел CI/CD мы увидим, что пайплайн готов к
запуску
Но находится в статусе pending / stuck так как у нас нет runner

Запустим Runner и зарегистрируем его в интерактивном
режиме

Перед тем, как запускать и регистрировать runner
нужно получить токен

Settings -> CI\CD Specific runners - Specific runners manually

http://35.246.233.216/
T2JPso8hqQmYv_Nsy8dW

На сервере, где работает Gitlab CI выполните команду:
docker run -d --name gitlab-runner --restart always \
-v /srv/gitlab-runner/config:/etc/gitlab-runner \
-v /var/run/docker.sock:/var/run/docker.sock \
gitlab/gitlab-runner:latest

После запуска Runner нужно зарегистрировать, это можно сделать командой:
root@gitlab-ci:~# docker exec -it gitlab-runner gitlab-runner register --run-untagged --locked=false

вводим URL, token, описание, теги (linux, ubuntu, docker). и исполнителя (docker)

Settings - > CI\CD -> Specific Runners