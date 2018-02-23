+++
date = "2018-02-23"
draft = false
description = "Simple Django + Celery + nginx + uWSGI deployment for small and medium-sized Python applications"
title = "Simple Django + Celery + nginx + uWSGI deployment 2018"
+++

## A simple approach that works

Literally hundreds of different approaches and tutorials to hosting a small or medium sized Django application can be found on the web. In this article I'll describe a very simple setup that we have been using successfully for quite a while at [CASHLINK Payments](https://cashlink.io), a payment startup I co-founded.

In contrast to many other Celery deployments, our setup can fit all of Django, nginx, uWSGI, and Celery workers into a **single Docker** image that may (but doesn't need to) be deployed multiple times for redundancy. To get started, all you have to do is run the Docker image. To scale up, you can simply run multiple instances of the Docker image. While we recommend Docker for reproducible deployments, our setup also works great with  other deployment methods like Ansible or supervisord.

We assume that you use Python 3 and Django 2.0, but other versions of Python and/or Django should work identically. Also, we assume that you use Ubuntu 17.10. Some file paths may be different on other operating systems.

## Nginx configuration

Let's start with the first component any HTTP request will reach: the reverse proxy. This surprisingly simple nginx configuration will get you started:

```nginx
server {
  server_name demo.com;
  listen 80;

  location / {
    include uwsgi_params;
    uwsgi_pass unix:/tmp/uwsgi.sock;
  }
}
```

If you don't use our Docker image (see below), you'll have to install the Ubuntu packages `nginx` and `uwsgi-plugin-python3`.

For a more secure configuration, please see [the GitHub repository for this tutorial](https://github.com/jonashaag/django-nginx-uwsgi-celery-deployment).

## uWSGI configuration

nginx will pass all HTTP requests to uWSGI, which is a Python web server.

uWSGI supports dozens of configuration file formats. Since we consider uWSGI our deployment's "main" process, we'll simply use command-line arguments to configure our instance.

```
uwsgi \
  --plugin python3 \
  --socket /tmp/uwsgi.sock \
  --threads 2 \
  --uid www-data \
  --gid www-data \
  --venv /srv/venv \
  --wsgi demoproject.wsgi:application
```

In the [Docker image](https://github.com/jonashaag/django-nginx-uwsgi-celery-deployment) the `uwsgi` command is used as the main `CWD`. If you're using supervisord, you can wire up the `uwsgi` command as the process to watch.

## Celery deployment

While Celery makes it easy to separate your main application and your Celery workers (and indeed this is recommended for large deployments), for simple applications it's actually much easier to always run the both together.

The reasoning here is that small applications will evolve task and main code simultaneously, with interfaces changing heavily in the early phases. Combining the both allows you to release new versions of your code base in a single step. Running multiple services that may fail independently also makes monitoring more cumbersome.

uWSGI has a plethora of features, one of which is the ability to attach other processes/daemons to be monitored by the master process. We'll spawn our Celery workers with uWSGI's `attach-daemon2` directive:

```
uwsgi \
  ...
  --attach-daemon2 'cmd=venv/bin/celery -A demoproject worker -l info -c 1 -B --scheduler django_celery_beat.schedulers:DatabaseScheduler'
```

Note that we use one worker process per uWSGI process here. Feel free to tune this with the `-c` parameter. You can skip the `-B … --scheduler …` parts if you don't use periodic tasks. 

## Celery monitoring

The recommended method of monitoring your Celery instances is [flower](flower.readthedocs.io).

Basic configuration is pretty simple: All you'll have to do is start a flower server. The server can be attached to uWSGI as well:

```
uwsgi \
  ...
  --attach-daemon2 'cmd=venv/bin/celery -A demoproject flower -l info'
```

This starts a web interface on port 5555. Of course you should make sure that your server is only accessible for authorized personell. For a very simple authentication mechanism you could configure nginx to require HTTP Basic Authentication.

Note that Flower won't work with the development-mode Django SQL database based Celery broker that we use in our setup. You can use a [free Cloud-hosted Redis database](https://redislabs.com/) as message broker for testing flower on your local machine.

See the [Celery monitoring documentation](http://docs.celeryproject.org/en/latest/userguide/monitoring.html#flower-real-time-celery-web-monitor) for details on Celery monitoring, and the [nginx documentation](http://nginx.org/en/docs/) for details on Basic Auth.
