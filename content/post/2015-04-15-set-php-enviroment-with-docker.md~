---
layout: post
title: Set up PHP enviroment with Docker
category: [tech]
tags: [php, docker]
---
Set up PHP develop eviroment with docker.

## docker image

I created my image based on offical 5.5-apache image, with following:

{% highlight bash %}
FROM php:5.5-apache

# install wget and unzip in order to install php extensions from github
RUN apt-get update && apt-get install -y wget unzip


# enable mbstirng as we can't controll the compilation flag of php
RUN docker-php-ext-install mbstring

# install mcrypt and pdo
RUN apt-get update && apt-get install -y libmcrypt-dev \
    && docker-php-ext-install mcrypt \
    && docker-php-ext-install mysql

# install redis extension
WORKDIR /root
RUN wget https://github.com/nicolasff/phpredis/archive/2.2.5.zip \
    && unzip 2.2.5.zip \ 
    && mv phpredis-2.2.5 /usr/src/php/ext/phpredis \
    && docker-php-ext-install phpredis
#RUN cp /usr/local/etc/php/conf.d/ext-phpredis-2.2.5.ini /etc/php5/mods-available/redis.ini

# install timecop
RUN wget https://github.com/hnw/php-timecop/archive/master.zip \
    && unzip master.zip \
    && mv php-timecop-master /usr/src/php/ext/php-timecop \
    && docker-php-ext-install php-timecop

RUN mkdir /var/log/httpd/
RUN mkdir -p /var/log/ci && chmod 777 /var/log/ci
COPY load.conf /etc/apache2/mods-enabled/
COPY klab.conf /etc/apache2/sites-enabled/
COPY klab_mainte.conf /etc/apache2/sites-enabled/
COPY php.ini /usr/local/etc/php/
{% endhighlight %}

mbstring, mcrypt, mysql, redis, timecop is the php extension needed by my project.
Note that they should be installed through offically provided docker-php-ext-install command.
load.conf contains configure that enables apache module needed by my project, like mod_rewrite.

klab.conf is the directory configure for my site.

klab_mainte.conf contains the rewrite rules for my site.

php.ini is for php configuration, here I only edited the error level, enabled short tags, set timezone.

## docker composer

As I need redis container, so I used docker composer with:

{% highlight yaml %}
web:
  build: .
  ports:
   - "8080:80"
  volumes:
   - klab-php:/klab
  links:
   - redis
redis:
  image: redis
{% endhighlight %}

I used **volumes** to share my source code with container. The strange thing is that, when I used docker1.4, volumes won't work
with docker-composer, it just won't mount. But start container directly with -v will work. It is solved as I upgrade my docker to
1.5.

## Start

```docker-composer up -d```, here we go!
