---
date: 2015-08-13
title: Some project notes(2)
categories: [tech]
tags: [docker Django]
---
## Develop enviroment with Docker
I used docker-compose to build my develop enviroment. The project consists of MySQL, django and celery(RabbitMQ). So my docker-compose.yml will be something like:
<!--more-->

```yaml
db:
  build: ./mysql
  environment:
    - MYSQL_ROOT_PASSWORD=testtest
  volumes:
    - /develop/db:/var/lib/mysql

rabbitmq:
  image: tutum/rabbitmq
environment:
  - RABBITMQ_PASS=guest
ports:
  - "5672:5672"  # we forward this port because it's useful for debugging
  - "15672:15672"  # here, we can access rabbitmq management plugin

worker:
  build: ./python
  environment:
    - C_FORCE_ROOT=1 # Root is not allowed to use celery by default
  command: celery worker -A projects.celery -n default@%h --loglevel=debug
  volumes:
    - ./mycode:/code # worker and django app shares same code
  links:
    - db
    - rabbitmq

web:
  build: ./python
  environment:
    - DJANGO_SETTINGS_MODULE=my_config #specify setting file to use
  volumes:
    - ./mycode:/code # mount my code to /code in container
    - /var/log/django:/var/log/django
  command: python manage.py runserver 0.0.0.0:8000
  ports:
    - "8000:8000"
  links:
    - db
    - rabbitmq
```

```yaml
FROM python:2.7
ENV PYTHONUNBUFFERED 1
RUN apt-get update;apt-get install -y mysql-client gettext vim
RUN mkdir /code
RUN mkdir /var/log/django
WORKDIR /code
ADD ./myocde/requirements.txt /code/
RUN pip install -r requirements.txt
RUN pip install ipython
```

### start two sites
Sometimes I need to test my site with different config file. So I made a copy of the docker-compose.yml, and name it site2.yml. Then in site2.yml,
I change the port mapping of web to other ports, eg. 1234:1234.

Then with ```docker-compose -f site2.yml run --serive-ports python manage.py 0.0.0.0:1234```, I can start another site with port 1234.
Note that the --service-ports option  makes the port mapping of web available in started container.
