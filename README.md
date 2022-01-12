# Django site in minikube

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки используйте переменные окружения. Список доступных переменных можно найти внутри файла `docker-compose.yml`.

#### Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Production

1. Install [Virtualbox](https://www.virtualbox.org/wiki/Downloads)

2. Install [minikube](https://minikube.sigs.k8s.io/docs/start/)

3. Install [kubectl](https://kubernetes.io/docs/tasks/tools/)

4. Start minikube in VirtualBox environment

```
minikube start --vm-driver=virtualbox
```

5. Build django app image

```
minikube image -t django-app:latest build backend_main_django
```

6. Rename configmap.example.yml to configmap.yml and define your environment variables

```
mv kubernetes/configmap.example.yml kubernetes/configmap.yml
```

- `DEBUG` - setting for turning on/off debug mode. Could be "false" or "true".

- `SECRET_KEY` - setting to be kept in secret. Used for hashes generation. Could be sequence of any symbols. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

- `ALLOWED HOSTS` - settings for listing allowed hosts. Could contain several values, separated by commas, for example `127.0.0.1,192.168.0.1,site.test`. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

- `DATABASE_URL` - link to the PostgreSQL database in specific [format](https://github.com/jacobian/dj-database-url#url-schema).

7. Create configmap

```
kubectl apply -f kubernetes/configmap.yml
```

8. Create database deployment and service

```
kubectl apply -f kubernetes/db-service.yml
```

9. Create django application deployment and service

```
kubectl apply -f kubernetes/django-service.yml
```

10. Create ingress

```
k apply -f kubernetes/ingress.yml
```

11. Add `star-burger.test` to local hosts

```
echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```