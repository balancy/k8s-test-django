# Django site in minikube

Dockerized Django site for experiments with Kubernetes.

Django app using Nginx Unit image is launched inside container. Nging Unit server fulfills two functions: it serves static and media files as web-server, and launches Python with Django as app server. [More about Nginx Unit](https://unit.nginx.org/).

## Local development setup

Run django app and the database:

```shell-session
docker-compose up
```

Run 2 following commands in the new tab:

```shell-session
docker-compose run web ./manage.py migrate
docker-compose run web ./manage.py createsuperuser
```

Use environment variables for more precise settings. All environment variables are acessible inside `docker-compose.yml` file.

- `DEBUG` - setting for turning on/off debug mode. Could be "false" or "true". [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

- `SECRET_KEY` - setting to be kept in secret. Used for hashes generation. Could be sequence of any symbols. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

- `ALLOWED HOSTS` - settings for listing allowed hosts. Could contain several values, separated by commas, for example `127.0.0.1,192.168.0.1,site.test`. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

- `DATABASE_URL` - link to the PostgreSQL database in specific [format](https://github.com/jacobian/dj-database-url#url-schema).

## Local production setup

1. Install [Virtualbox](https://www.virtualbox.org/wiki/Downloads)

2. Install [Minikube](https://minikube.sigs.k8s.io/docs/start/)

3. Install [Kubectl](https://kubernetes.io/docs/tasks/tools/)

4. Start Minikube cluster in VirtualBox environment

```
minikube start --vm-driver=virtualbox
```

5. Install [Helm](https://helm.sh/)

6. Install Helm Chart for postgreql

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install psql-db bitnami/postgresql
```

7. Create database, user and password following the instructions given on previous step

8. Build django app image

```
minikube image -t django-app:latest build backend_main_django
```

9. Rename `configmap.example.yml` to `configmap.yml` and define environment variables

```
mv kubernetes/configmap.example.yml kubernetes/configmap.yml
```

Environment variables:

- `DEBUG` - setting for turning on/off debug mode. Could be "false" or "true". [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

- `SECRET_KEY` - setting to be kept in secret. Used for hashes generation. Could be sequence of any symbols. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

- `ALLOWED HOSTS` - settings for listing allowed hosts. Could contain several values, separated by commas, for example `127.0.0.1,192.168.0.1,site.test`. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

- `DATABASE_URL` - link to created by your PostgreSQL database in specific [format](https://github.com/jacobian/dj-database-url#url-schema).

10. Create configmap

```
kubectl apply -f kubernetes/configmap.yml
```

11. Create django application deployment and service

```
kubectl apply -f kubernetes/django-service.yml
```

12. Create ingress

```
k apply -f kubernetes/ingress.yml
```

13. Add `star-burger.test` to local hosts

```
echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```

14. Migrate the database

```
kubectl apply -f kubernetes/django-migrate.yml
```

15. Create cronjob which clears django sessions every month

```
kubectl apply -f kubernetes/django-clearsessions.yml
```