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


---
# 🚀 Пошаговое развёртывание в кластере Kubernetes на Яндекс Облаке
Кратко о среде
- `Домен`: edu-roman-grachev.sirius-k8s.dvmn.org
- `Namespace` Kubernetes: edu-roman-grachev
- `Кластер`: Managed Kubernetes в Яндекс Облаке (yc-sirius или аналогичный, выделенный вам)
- `База данных`: PostgreSQL (Managed Service Yandex) с защищённым SSL-подключением
- `Docker Registry`: Docker Hub (используется для хранения и получения образов приложения)
- `Object Storage`: Yandex S3 Bucket для статики и медиа с доступом через секреты Kubernetes


#### Структура каталога `edu-roman-grachev/`
- django-configmap.yaml
- django-secret.yaml      # создается по инструкции
- django-deployment.yaml
- django-ingress.yaml
- django-migrate-job.yaml
- django-clearsessions-cronjob.yaml
- django-service.yaml


#### Подготовка и загрузка в Docker Hub

1. Получить текущий git-хэш: `git rev-parse --short HEAD`
2. Соберите Docker-образ: `docker build -t grroma:<git-хэш> -f ./backend_main_django`
3. Добавьте тег: `docker tag grroma:<git-хэш>  grroma/django_app:<git-хэш>`
4. Загрузите образ: `docker push grroma/django_app:<git-хэш>`


#### Настройка Kubernetes ресурсов.

- Создайте файл django-secret.yaml в edu-roman-grachev/ (укажите там свои SECRET_KEY и [DATABASE_URL](https://github.com/jazzband/dj-database-url)):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secret
type: Opaque
stringData:
  SECRET_KEY: "ваш_секретный_ключ_django"
  DATABASE_URL: "postgres://<user>:<password>@<host>:<port>/<dbname>"
```
#### Получение SSL-сертификата для подключения к базе данных PostgreSQL
PostgreSQL-хосты с публичным доступом поддерживают только шифрованные соединения. Чтобы использовать их, получите SSL-сертификат

- Linux(Bash)/macOS(Zsh)
```bash
  mkdir -p ~/.postgresql && \
  wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
      --output-document ~/.postgresql/root.crt && \
  chmod 0600 ~/.postgresql/root.crt
```

- Windows(PowerShell)
```bash
  mkdir $HOME\.postgresql; curl.exe -o $HOME\.postgresql\root.crt https://storage.yandexcloud.net/cloud-certs/CA.pem
```

Сертификат будет сохранен в файле $HOME\.postgresql\root.crt

#### Запуск и проверка
- Примените все ресурсы:

```bash
  kubectl apply -f edu-roman-grachev/
```

- Примените по очередно:
```bash
  kubectl apply -f django-service.yaml -n edu-roman-grachev
  kubectl apply -f django-secret.yaml -n edu-roman-grachev
  kubectl apply -f django-configmap.yaml -n edu-roman-grachev
  kubectl apply -f django-deployment.yaml -n edu-roman-grachev
  kubectl apply -f django-ingress.yaml -n edu-roman-grachev
  kubectl apply -f django-migrate-job.yaml -n edu-roman-grachev
  kubectl apply -f django-clearsessions-cronjob.yaml -n edu-roman-grachev
```

#### Проверьте состояние подов, сервисов и Ingress:

```bash
kubectl get pods -n edu-roman-grachev
kubectl get svc -n edu-roman-grachev
kubectl get ingress -n edu-roman-grachev
```

#### Сайт доступен по адресу: https://edu-roman-grachev.sirius-k8s.dvmn.org

#### Для доступа к сайту и админке через локальный проброс портов:

```bash
kubectl port-forward service/django 8000:80 -n edu-roman-grachev
```
- Сайт будет доступен по адресу: http://localhost:8000
- Админка: http://localhost:8000/admin/

## Цели проекта
Код написан в учебных целях — это урок в курсе по Python и веб-разработке на сайте [Devman](https://dvmn.org/).