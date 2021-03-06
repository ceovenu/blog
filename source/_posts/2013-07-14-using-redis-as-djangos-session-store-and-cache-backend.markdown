---
layout: post
title: "Using Redis as Django's session store and cache backend"
date: 2013-07-14
permalink: "/blog/2013/07/14/using-redis-as-django-session-store-and-cache-backend/"
comments: true
categories: tech
published: true
tags: django redis
---

[Redis](http://redis.io) is an in-memory key-value store, somewhat similar to Memcached. Because Redis keeps its dataset in memory, storage and retrieval is very fast. It's a good idea to use this kind of solution for storing ephemeral application data, such as contents of the cache, or temporary information associated with active user sessions. This unburdens your database system from performing unnecessary read and write operations and can considerably speed up your application. Modules for using Redis together with Django are available and quite easy to set up.

<!-- more -->

## Prerequisites

I'm going to assume you have a Django application running in a virtual environment, under supervisord on a Debian server. Instructions for creating such a setup can be found in a [previous article](/blog/2013/06/09/django-nginx-gunicorn-virtualenv-supervisor/).

## Redis

Setting up Redis on Debian is simple using `apt-get`. On a RedHat-based system you can use the equivalent `yum` command.

### Install Redis

    $ sudo apt-get install redis-server
    $ redis-server --version
    Redis server version 2.4.14 (00000000:0)


### Configure Redis to connect over a socket

You can connect to a local Redis instance over the network layer (TCP to the loopback interface) or through a unix socket file. 

In order to avoid the small overhead of TCP, we can configure Redis to accept direct socket connections. To do this, edit your `/etc/redis/redis.conf` file, comment out the `bind` and `port` directives and uncomment the two `unixsocket` directives.

```cfg
# Accept connections on the specified port, default is 6379.
# If port 0 is specified Redis will not listen on a TCP socket.
# port 6379

# If you want you can bind a single interface, if the bind option is not
# specified all the interfaces will listen for incoming connections.
#
# bind 127.0.0.1

# Specify the path for the unix socket that will be used to listen for
# incoming connections. There is no default, so Redis will not listen
# on a unix socket when not specified.
#
unixsocket /var/run/redis/redis.sock
unixsocketperm 755
```

After making changes to its configuration you will need to restart Redis:

    $ sudo service redis-server restart

You can now check if Redis is up and accepting connections:

    $ redis-cli ping
    PONG


#### Redis socket permissions

Default permissions on the Redis socket are very restrictive on Debian and allow only the user `redis` to connect through it. You can relax these permissions and allow any local user to connect to Redis like this:

    $ sudo chmod 777 /var/run/redis/redis.sock

For security reasons, it may be better to restrict access to the socket to a chosen group of users. You can add the user which your application will run as to the group `redis` and change the permissions on the socket file to allow writes only from the group.

    $ sudo usermod -a -G redis django_username
    $ sudo chmod 775 /var/run/redis/redis.sock

Groups are assigned at login, so if your application is running, you'll need to restart it. If you're running an application called `hello` via `supervisor`, you can restart it like this:

    $ sudo supervisorctl restart hello

On their site, you will find more information about [getting started with Redis](http://redis.io/topics/quickstart).



### Python bindings for Redis

In order to use Redis with a Python application such as Django, you'll need to install the Redis-Python interface bindings module. You can install it in your virtualenv with `pip`:

    $ source bin/activate
    (hello_django) $ pip install redis


## Redis as backend for Django session data

You can now set up Redis as a store for Django's session data. You can use  [django-redis-sessions](https://github.com/martinrusev/django-redis-sessions) module to do so.

Inside your virtual environment install django-redis-sessions:

    (hello_django) $ pip install django-redis-sessions

Now, configure `redis_sessions.session` as the session engine in your `setting.py`. Since we're using a socket to connect, we also need to provide its path:

```python
SESSION_ENGINE = 'redis_sessions.session'
SESSION_REDIS_UNIX_DOMAIN_SOCKET_PATH = '/var/run/redis/redis.sock'
```

That's it. After you restart your application, session data should be stored in Redis instead of the database.

More information: [working with sessions in Django](https://docs.djangoproject.com/en/dev/topics/http/sessions/).


## Redis as backend for Django's cache

We can also set up Redis to store our application's cache data. We can use [django-redis-cache](https://github.com/sebleier/django-redis-cache) for this.

Install django-redis-cache in your virtual environment:

    (hello_django) $ pip install django-redis-cache 

And add the following to your `settings.py` file:

```python
CACHES = {
    'default': {
        'BACKEND': 'redis_cache.RedisCache',
        'LOCATION': '/var/run/redis/redis.sock',
    },
}
```

If you haven't set up Django's cacheing middleware, you should do so by adding lines 2 and 4 from the snippet below to `MIDDLEWARE_CLASSES` in your `settings.py`.

```
MIDDLEWARE_CLASSES = (
    'django.middleware.cache.UpdateCacheMiddleware',    # This middleware must be first on the list
    'django.middleware.common.CommonMiddleware',
    'django.middleware.cache.FetchFromCacheMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    (...)
```

You can now restart your application and start using Django's Redis-powered cache.

More information: [using Django's cache](https://docs.djangoproject.com/en/dev/topics/cache/).


## Redis for multiple applications

Please note that the solution described above can only be used by a single Django application. If you want to use multiple Django applications with Redis, you could try to separate their namespaces, by using key prefixes, or using a different Redis numeric database for each (`1` for one application, `2` for another, etc). These solutions are not advised however.

If you want to run multiple applications each with a Redis cache, the recommendation is to run a separate Redis instance for each application. On Chris Laskey's blog you can find instructions for [setting up multiple Redis instances on the same server](http://chrislaskey.com/blog/342/running-multiple-redis-instances-on-the-same-server/).
