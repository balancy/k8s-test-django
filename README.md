# Django site in k8s cluster

Dockerized Django site for experiments with Kubernetes.

Django app using Nginx Unit image is launched inside container. Nging Unit server fulfills two functions: it serves static and media files as web-server, and launches Python with Django as app server. [More about Nginx Unit](https://unit.nginx.org/).

## Dockerized development setup

Clone the repository

```bash
git clone https://github.com/balancy/k8s-test-django.git
```

Go inside cloned repository

```bash
cd k8s-test-django
```

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

## Minikube development setup

You'll need to install few tools for app to work.

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
```
```
helm install psql-db bitnami/postgresql
```

7. Create database providing its name, user and password following the instructions given on previous step

8. Clone the repository and go inside cloned repo

```bash
git clone https://github.com/balancy/k8s-test-django.git
```

```bash
cd k8s-test-django
```

9. Build versioned django app image (git hash is used for versioning) and push it to [Docker Hub](https://hub.docker.com/)

```
docker build -t <your_dockerhub>/django-app:$(git log -1 --pretty=%h) backend_main_django
```
```
docker push <your_dockerhub>/django-app:$(git log -1 --pretty=%h)
```

- where `<your_dockerhub>` is your personal Docker Hub image repository

10. Copy `configmap.example.yml` to `configmap.yml` and define environment variables in `configmap.yml`

```
cp kubernetes/configmap.example.yml kubernetes/configmap.yml
```

Environment variables:

- `DEBUG` - setting for turning on/off debug mode. Could be "false" or "true". [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

- `SECRET_KEY` - setting to be kept in secret. Used for hashes generation. Could be sequence of any symbols. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

- `ALLOWED HOSTS` - settings for listing allowed hosts. Could contain several values, separated by commas, for example `127.0.0.1,192.168.0.1,site.test`. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

- `DATABASE_URL` - link to PostgreSQL database in specific [format](https://github.com/jacobian/dj-database-url#url-schema). Use database name, username and password specified during the database creation.

11. Create configmap

```
kubectl apply -f kubernetes/configmap.yml
```

12. Create django application deployment and service

```
kubectl apply -f kubernetes/django-service.yml
```

* optional:

In case you need to deploy version of django app image different from the latest, use the following command:

```
kubectl set image deploy/django django=<your_dockerhub>/django-app:<version>
```

- where `version` is the django app image version corresponding to the sequence of git commit 7 first characters

13. Migrate the database

```
kubectl apply -f kubernetes/django-migrate.yml
```

14. Create ingress

```
k apply -f kubernetes/ingress.yml
```

15. Add `star-burger.test` to local hosts

```
echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```

16. Create cronjob which clears django sessions every month

```
kubectl apply -f kubernetes/django-clearsessions.yml
```

## Production setup

#### Launch the app

You need a k8s cluster and postgresql db for app to work.

1. Clone the repository and go inside cloned repo

```bash
git clone https://github.com/balancy/k8s-test-django.git
```

```bash
cd k8s-test-django
```

2. Copy `configmap.example.yml` to `configmap.yml` and define environment variables in `configmap.yml`

```
cp kubernetes/configmap.example.yml kubernetes/configmap.yml
```

Environment variables:

- `DEBUG` - setting for turning on/off debug mode. For production use the "false" value.

- `SECRET_KEY` - setting to be kept in secret. Used for hashes generation. Could be sequence of any symbols. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

- `ALLOWED HOSTS` - settings for listing allowed hosts. Could contain several values, separated by commas, for example `127.0.0.1,192.168.0.1,site.test`. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

- `DATABASE_URL` - link to PostgreSQL database in specific [format](https://github.com/jacobian/dj-database-url#url-schema). Use database name, username and password of your postgresql db.

3. Create configmap

```
kubectl apply -f kubernetes/configmap.yml
```

4. Create django application deployment and service

```
kubectl apply -f kubernetes/django-service.yml
```

5. Migrate the database

```
kubectl apply -f kubernetes/django-migrate.yml
```

6. Create cronjob which clears django sessions every month

```
kubectl apply -f kubernetes/django-clearsessions.yml
```

#### Maintain the app

In case you need to update the app to specific version, use the following command:

```
kubectl set image deploy/django django=<app_dockerhub>/django-app:<version>
```

* you can use `latest` as version to update to the latest available version

In order to check if update if finished, use the following commands:

```
k describe pods | grep Status:
k describe pods | grep Image:
```

If status of pods is `Running` and Image is the image you wanted to update app to, it's finished.
