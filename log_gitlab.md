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

Runner запускается по коммиту. Можно посмотреть на вывод-промежуточные результаты


* добавил исходный код реддит в репозиторий
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

* Изменим пайплайн таким образом, чтобы job deploy
стал определением окружения dev, на которое условно
будет выкатываться каждое изменение в коде проекта

* переименован stage-deploy в review ; deploy job -> deploy_dev_job

* добавил environment


* после успешного прохождения конвейера появится пункт environment в GUI (Operations - > Environments)

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
   when: manual
   script:
       - echo 'Deploy'
   environment: 
       name: production
       url: https://example.com
````


* осталось дописать про диначиское окружение



* в папке gitlab-ci репозиторий создан docker-compose.yml для развертывания gitlab-сi 