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
## Как развернуть проект в Minikube
#### ✅ Структура каталога `kubernetes/`
- django-configmap.yaml
- django-secret.yaml      # создается по инструкции
- django-deployment.yaml
- django-service.yaml
- django-ingress.yaml
- django-migrate-job.yaml
- django-clearsessions-cronjob.yaml


Для развёртывания проекта понадобится:

*   `Docker`: Для локальной разработки и запуска контейнеров. Установите его с [официального сайта](https://docs.docker.com/get-docker/).
*   ``Minikube``: Для создания локального Kubernetes кластера. Инструкции по установке: [Minikube Installation](https://minikube.sigs.k8s.io/docs/start/).
*   `kubectl`: Инструмент командной строки для управления Kubernetes кластерами. Инструкции по установке: [kubectl Installation](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
*   `Helm`: Менеджер пакетов для Kubernetes. Инструкции по установке: [Helm Installation](https://helm.sh/docs/intro/install/).
*   Драйвер виртуализации: Minikube требует драйвер для запуска кластера (например, Docker Desktop).


### Запуск Minikube кластера

1. Запустите кластер с выделением ресурсов:
```bash
minikube start
```

2. Проверьте статус:
```bash
minikube status
```

3. Опционально откройте дашборд Kubernetes:
```bash
minikube dashboard
```

### Настройка Ingress
1. Включите ingress-nginx контроллер:
```bash
minikube addons enable ingress
```
2. Запустите туннель для корректной маршрутизации:
```bash
minikube tunnel
```

3. Добавьте в hosts-файл вашей ОС строку, где IP — вывод minikube ip:
```bash
ваш_IP  star-burger.test
```



### Развёртывание проекта 

1. Создайте файл django-secret.yaml в kubernetes/ (укажите там свои SECRET_KEY и [DATABASE_URL](https://github.com/jazzband/dj-database-url)):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secret
type: Opaque
stringData:
  SECRET_KEY: "ваш_секретный_ключ_django"
  DATABASE_URL: postgres://test_k8s:OwOtBep9Frut@my-postgres-postgresql:5432/test_k8s
```

2. Развёртывание PostgreSQL в Minikube через [Helm](https://helm.sh/). Для установки PostgreSQL используем официальный [Helm chart от Bitnami](https://artifacthub.io/packages/helm/bitnami/postgresql).

- Войти в Docker Hub (если требуется)
```bash
  helm registry login docker.io
```
- Установка PostgreSQL с паролем для пользователя postgres
```bash
  helm install my-postgres oci://registry-1.docker.io/bitnamicharts/postgresql --set auth.postgresPassword=yourpassword
```
- Проверьте, что поды и PVC создались успешно:
```bash
  kubectl get pods
  kubectl get pvc
```
- Подключение к PostgreSQL из кластера для проверки
```bash
  kubectl run pg-client --rm -ti --image=postgres --env="PGPASSWORD=yourpassword" --command -- psql -h my-postgres-postgresql -U postgres
```
- После изменения конфигурации перезапустите поды с приложением:
```bash
  kubectl rollout restart deployment django-deployment
```

3. Примените все манифесты одной командой:
- CronJob настроен на запуск в 6:00 утра 15-го числа каждого месяца.
```bash
kubectl apply -f kubernetes/
```

4. Откройте в браузере:

http://star-burger.test/










