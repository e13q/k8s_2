# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Работа в кластере
Важно: при работе в кластере указывать namespace либо в манифестах, либо по умолчанию в настройках ПО с графическим интерфейсом (Lens, например).

Пример запуска пода с Nginx описан в ```\deploy\yc-sirius\edu-charming-yonath\``` в файлах ```simple-pod.yaml``` и ```simple-service.yaml```.
Реализация основана на работе балансировщика (Application Load Balancer) по принципу "домен -> nodePort". Достаточно указать nodePort, и при обращении к нужному URL балансировщик направит запрос на соответствующий под.

### Postgresql SSL
Пример пода с загрузкой SSL сертификата для проверки доступности Postgresql реализован в файле ```\deploy\yc-sirius\edu-charming-yonath\simple-psql-testing.yaml```.
Подключиться к поду и подключиться к postgresql через psql.


### Секреты
Для создания секрета на кластере можно использовать .env файл с данными.
Пример содержания .env:
```
SECRET_KEY=DJANGO-SECRET-KEY
DATABASE_URL=postgres://POSTGRES_USER:POSTGRES_PASSWORD_@postgresql.default.svc.cluster.local:5432/POSTGRES
ALLOWED_HOSTS=192.168.1.1,127.0.0.1,localhost
```

Далее выполняем команду в директории с файлом:
```
kubectl create secret generic django-secrets --from-env-file=.env
```

### Сборка и публикация образа

Сборка образа с тегом, включающим хэш коммита


Получите хэш текущего коммита:
```
$commitHash = git rev-parse HEAD
```

Соберите Docker-образ с тегом на основе хэша:
```
docker build -t your-dockerhub-username/django-site:$commitHash .
```

Загрузка образа на Docker Hub

```
docker login -u your-dockerhub-username
```

```
docker push your-dockerhub-username/django-site:$commitHash
```

### Как запустить

Секреты:
```
kubectl create secret generic django-secrets --from-env-file=.env
```
Django:
```
kubectl apply -f deploy.yaml
```
Django - миграция:
```
kubectl apply -f job-migrate.yaml
```
Сервис:
```
kubectl apply -f service.yaml
```
CronJob на чистку сессий:
```
kubectl apply -f cronjob-clearsessions.yaml
```

Готово! Ресурс доступен на выделенном админом домене.
В моём случае это [edu-charming-yonath.sirius-k8s.dvmn.org](https://edu-charming-yonath.sirius-k8s.dvmn.org/)

### Создание суперпользователя / запуск Management скриптов

Подключаемся к поду с Django
```
kubectl exec -it django-7b77cdd848-6jrf6 /bin/bash
```

Запускаем Management-команду
```
python3 manage.py createsuperuser
```

Отвечаем на вопросы и готово.

### Где искать ошибки
Ошибки можно обнаружить в логах пода с Django. Например:
```
kubectl logs django-7b77cdd848-6jrf6
```

### Обновление новой версии

[Собираем новый образ](#сборка-и-публикация-образа)
Указываем новый образ в манифестах deploy.yaml, job-migrate.yaml, cronjob-clearsessions.yaml.
Запускаем по очереди команды:
Обновим Django:
```
kubectl apply -f deploy.yaml
```
Выполним Django - миграцию:
```
kubectl apply -f job-migrate.yaml
```
Обновим CronJob на чистку сессий:
```
kubectl delete -f cronjob-clearsessions.yaml && kubectl apply -f cronjob-clearsessions.yaml
```
Запускаем
```
kubectl get pods
```
Если статусы у Django "Running", а у миграции "Completed" - всё завершено успешно. 
```
NAME                      READY   STATUS      RESTARTS   AGE
django-7b77cdd848-6jrf6   1/1     Running     0          17m
django-migrate-kvznf      0/1     Completed   0          17m
```
В ином случае [смотрим ошибки](#где-искать-ошибки)