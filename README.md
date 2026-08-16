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

## Развёртывание в Kubernetes

Манифесты, с помощью которых будем запускать проект, лежат в каталоге `kubernetes/` (Deployment + Service + Ingress + CronJob). Секретные настройки (`SECRET_KEY`, `DATABASE_URL`) не лежат в манифестах - они вынесены в отдельный объект Kubernetes `Secret`, который хранится только в кластере и не попадает в git вместе с кодом. Это защищает чувствительные данные от случайной утечки через репозиторий.

Для установки кластера и работы с манифестами понадобятся:
- **Docker** и **Docker Compose** - для локального запуска базы данных PostgreSQL и сборки образа Django
- **kubectl** — клиент для управления кластером
- **minikube** — локальный кластер Kubernetes (в примерах ниже драйвер `kvm2`, для своего окружения подберите свой: `minikube start --driver=<ваш_драйвер>` и т.д.).

### Запустить БД PostgreSQL в Docker

Deployment обращается к базе PostgreSQL. Её необходимо поднять из корня проекта:

```shell
docker compose up -d db
```
После перезагрузки локальной машины или сервера, контейнер может остановиться, чтобы его включить используйте следующую команду:

```shell
docker start k8s-test-django-db-1
```

### Собрать и загрузить образ в кластер

Выполните следующие команды в терминале:

```shell
docker build -t django_app backend_main_django # собираем образ Docker
minikube image load django_app # загружаем образ в кластер
```

### Включить Ingress контроллер

Сам объект Ingress - это только правила. За реальный приём трафика на порту 80
отвечает Ingress контроллер. В minikube он ставится аддоном:

```shell
minikube addons enable ingress
kubectl get pods -n ingress-nginx # контроллер должен быть в статусе Running
```

### Создание Secret в кластере

Создайте `secret.yml` и запишите туда необходимые настройки и добавьте переменные окружения(`SECRET_KEY`, `DATABASE_URL`, `DEBUG`, `ALLOWED_HOSTS`):

```yml
apiVersion: v1
kind: Secret
metadata:
  name: django-secret
type: Opaque
stringData:
  SECRET_KEY: <секретный_ключ>
  DATABASE_URL: postgres://test_k8s:<пароль>@host.minikube.internal:5432/test_k8s
  DEBUG: "False"
  ALLOWED_HOSTS: 127.0.0.1,localhost,<ваш_домен>,...
```

После создания и настройки файла, выполняем команду для запуска объекта Secret в кластере:

```shell
kubectl apply -f secret.yml
kubectl get secret # проверить что Secret создался
```
> Сам файл `secret.yml` коммитить нельзя т.к. он содержит чувствительные данные, его название занесено в `.gitignore`

### Применить манифесты
Для запуска проекта введите следующее:

```shell
kubectl apply -f kubernetes/ # запустить Deployment, Service, Ingress и CronJob
kubectl rollout status deployment/django-deploy # проверить статус запуска
kubectl get svc django # проверяем информацию по запущенному сервису
# не должно быть публичного порта и TYPE=ClusterIP
kubectl get ingress django # проверяем информацию по объекту Ingress
# в ADRESS должен быть ip нужной ноды
```

### Добавить домен в /etc/hosts

Файл /etc/hosts - локальный аналог DNS. Браузер смотрит в него раньше, чем
к DNS-серверу, поэтому домен откроется только у вас на машине. Узнайте ip нужной ноды(если не сделали до этого) и запишите в файл hosts в одну строчку через пробел ip ноды и необходимый домен:

```shell
minikube ip # узнаем ip ноды
sudo nano /etc/hosts # заходим в редактор и добавляем строку с ip и доменом
# эта команда работает только для системы linux
ping -c1 star-burger.test # на наш запрос должен ответить ip указнной ноды
curl -I http://star-burger.test # а данная команда должна возвращать статус 302
```
### Открыть сайт

После выполнения всех вышеуказанных пунктов сайт будет доступен на стандартном порту 80 по адресу http://star-burger.test

### Регулярная очистка устаревших сессий (CronJob)

Информацию о посетителях сайта Django хранит в базе данных - в модели `Session` стандартного приложения `django.contrib.sessions`. Сессия создаётся для каждого посетителя и со временем накапливается в БД. Для их удаления есть стандартная
команда Django `python manage.py clearsessions`, которая стирает только **просроченные** сессии. Чтобы не запускать очистку вручную, в кластере настроен **CronJob** (`kubernetes/django-clearsessions-cronjob.yml`). Kubernetes сам запускает задачу
по расписанию - первого числа каждого месяца в 15:00 (`schedule: "0 15 1 * *"`, часовая зона `Europe/Moscow`). Каждый запуск создаёт разовый под, который выполняет `clearsessions` и завершается со статусом `Completed`.

CronJob применяется вместе с остальными манифестами:

```shell
kubectl apply -f kubernetes/
kubectl get cronjob django-clearsessions # эта команда выводит информацию о задачах с их расписанием
```
Если Вы хотите запустить очистку немедленно, выполните следующие команды:

```shell
kubectl create job django-clearsessions-once --from=cronjob/django-clearsessions
kubectl get jobs django-clearsessions-once   # проверка - COMPLETIONS должно стать 1/1
kubectl get pods -l job-name=django-clearsessions-once # ещё проверка - pod должен быть со статусом Completed
```

## Цель проекта

Проект создан в учебных целях.