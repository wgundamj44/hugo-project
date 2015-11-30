---
title: "Use Nginx to manage django sites in docker"
categories: ["tech"]
date: 2015-11-30
tags: ["Django", "nginx", "uwsgi", "docker"]
---
In [previous post]({{< ref "2015-08-13-project-note2.md" >}}), I setuped my django develop enviroment with docker. It works fine but with one problem, it use Django's `runserver` command to manage all the static files, the page load time is very long, sometimes it even stucks in page loading forever. So I decide to to nginx to replace command of Django.

### Set up a uwsgi
To use python with nginx, uwsgi is our first choice.

I created mysite.yaml like the following:
```yaml
uwsgi:
	socket: 0.0.0.0:8000
	master: true
	no-orphans: true
	processes: 1
	uid: root
	gid: root
	chdir: /code # where my django app resides in the docker container
	env: DJANGO_SETTINGS_MODULE=xxxx # my django setting file
	module: xxx # my wsgi module
	buffer-size: 40960
	enable-threads: true
```
Several notes here:
- root as uid, gid is a bad choice. I set it like this just for convenience.
- socket is bind to 0.0.0.0:8000, so that nginx can get access to this socket from another docker container
- when started, uwsgi will start a the wsgi module set in the module entry, and serve it at port 8000.
Save this mysite.yaml to ./uwsgi folder, our web container becomes:
```yaml
web:
	build: ./python
	volumes:
	  - ./mycode:/code
	  - ./uwsgi:/uwsgi
	command: uswgi --yaml /uwsgi/mysite.yaml
	ports:
	  - "8000:8000"
	links:
	  - db
	  - rabbitmq
```

### Set up nginx
With uwsgi set, the nginx site configure file mysite.conf likes like:
```nginx
server {
	listen          8080;
	server_name     my.test-site.local;
	client_max_body_size 50M;
	ssl off;
	location ~* /favicon.ico {
		empty_gif;
	}

	location / {
		include uwsgi_params;
		uwsgi_pass  uwsgicluster;
		uwsgi_param UWSGI_SCHEME $scheme;
	}

	location ^~ /static/ {
		alias xxxx # path to my static files;
	}
}
upstream uwsgicluster {
	server web:8000;
}
```
About nginx's `nginx.conf`, I used a default one with one modification: Add `daemon off` at head, otherwise when nginx get started, its main process will quit making container exit too, which is not what we want.
The final nginx.conf is like:
```nginx
daemon off;
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;

pid        /var/run/nginx.pid;

events {
	worker_connections  1024;
}
	
http {
	include       /etc/nginx/mime.types;
	default_type  application/octet-stream;

	access_log  /var/log/nginx/access.log  main;
	sendfile        on;
	
	keepalive_timeout  65;
	
	gzip  on;

	server {
		listen 80;
		return 301 https://$host$request_uri;
	}

	include /etc/nginx/conf.d/*.conf;
}
```

And we add a new entry into docker-compose.yaml for nginx container:
```yaml
nginx:
image: nginx
  volumes:
    - ./nginx/conf/nginx.conf:/etc/nginx/nginx.conf
    - ./nginx/conf/conf.d/mysite.conf:/etc/nginx/conf.d/mysite.conf
    - ./mycode:/code
  command: /etc/init.d/nginx start
  ports:
    - "8080:8080"
  links:
    - web
```
The important part here is the `uwsgi_pass` part, which forward all the request other than static files to web:8000. `web:8000` is the service on web container.

Also, with `location` part, the static files now will be handled by nginx.

Now the access the my web app is much faster.

### Add reload capacity to uwsgi
The downside of the above configuration is we lose the ability to automatically restart server when code is changed. To solve this, add these lines to the uswgi entry wsgi file:
```python
import uwsgi
from uwsgidecorators import timer
from django.utils import autoreload

@timer(3)
def change_code_gracefull_reload(sig):
	if autoreload.code_changed():
		uwsgi.reload()
```
This snippets make use of `autoreload` of Django, and restart uwsgi if code is changed.

### Manage multiple sites
To serve multiple sites in the same ip, we should modify both nginx and uwsgi.

#### server block of nginx
For nginx part, we make use of its `server block` mechanism: load different configs based on host name. I add a new conf file for my new site, say "my2.test-site.local".
```nginx
server {
	listen          8080;
	server_name     my2.test-site.local;
	client_max_body_size 50M;
	ssl off;
	location ~* /favicon.ico {
		empty_gif;
	}

	location / {
		include uwsgi_params;
		uwsgi_pass  uwsgicluster;
		uwsgi_param UWSGI_SCHEME $scheme;
	}

	location ^~ /static/ {
		alias xxxx # path to my static files for my2.test-site.local;
	}
}
upstream uwsgicluster {
	server web:3000;
}
```
Note here, we listen to the same port as `my.test-site.local`, but forward the request to different upstream uwsgi, and serve static files in different folder.

Next we should set our new upstream uwsgi.

#### emperor of uwsgi
Of course we can set up two uwsgi manually to serve two sites, uwsgi offers a better way: emperor mode. This emperor mode basically monitors some config files, and when we add or remove uwsgi config files, it add or remove uwsgi process accordingly.

Its usage is simple, just the command `uwsgi --emperor /uwsgi/conf/`, where `/uwsgi/conf/` is the folder where all the sites' uwsgi files reside. We just put the conf for new site into this folder, and everything works automatically.

The conf for `my2.test-site.local` is like:
```yaml
uwsgi:
	socket: 0.0.0.0:3000
	master: true
	no-orphans: true
	processes: 1
	uid: root
	gid: root
	chdir: /code # where my django app resides in the docker container
	env: DJANGO_SETTINGS_MODULE=xxxx # my django setting file
	module: xxx # my wsgi module
	buffer-size: 40960
	enable-threads: true
```
Note the `socket` here, it's bound to port 3000 which consistent with nginx configuration.

Finally, set `my.test-site.local` and `my2.test-site.local` both to `localhost` in hosts settings. Then we have two sites for one ip, and runs perfectly fast.

