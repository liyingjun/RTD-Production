Please bear in mind this is for a Ubuntu 14.04 environment

============
Installation
============

    sudo mkdir /usr/share/readthedocs
    sudo chown your_username readthedocs
    sudo apt-get install build-essential python-dev libxml2-dev libxslt1-dev zlib1g-dev virtualenv git 
    cd /usr/share/ 
    virtualenv readthedocs 
    cd readthedocs 
    source bin/activate 
    pip install --upgrade pip 
    mkdir checkouts
    cd checkouts
    git clone https://github.com/rtfd/readthedocs.org.git 
    cd readthedocs.org
    pip install -r requirements.txt
    ./manage.py collectstatic

    sudo apt-get install libpq-dev redis-server
    cd /usr/share/readthedocs/
    source bin/activate
    pip install psycopg2 redis django-redis-cache django-redis-sessions
    pip install --upgrade hiredis

--------------
verified redis
--------------

    redis-cli ping

which returns `PONG`

------------------
Configure settings
------------------

Add a new file `postgres.py` to `/usr/share/readthedocs/checkouts/readthedocs.org/readthedocs/settings/`

    import os
    
    from .base import *
    
    DEBUG = False
    PRODUCTION_DOMAIN = '192.168.80.129'
    PUBLIC_API_URL = 'https://{0}'.format(PRODUCTION_DOMAIN)
   
    SLUMBER_USERNAME = 'test'
    SLUMBER_PASSWORD = 'test'
    SLUMBER_API_HOST = 'http://192.168.80.129'
    GROK_API_HOST = 'http://192.168.80.129'
    
    DATABASES = {
        'default': {
          'ENGINE': 'django.db.backends.mysql',
          'NAME': 'rtd',
          'USER': 'rtd',
          'PASSWORD': 'rtd123123',
          'HOST': 'localhost',
        }
    }
    
    EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
    DEFAULT_FROM_EMAIL = 'test@163.com'
    EMAIL_USE_TLS = True
    EMAIL_HOST = 'smtp.163.com'
    EMAIL_HOST_USER = 'test@163.com'
    EMAIL_HOST_PASSWORD = 'test'
    EMAIL_PORT = 25
    
    HAYSTACK_CONNECTIONS = {
        'default': {
            'ENGINE': 'haystack.backends.elasticsearch_backend.ElasticsearchSearchEngine',
            'URL': 'http://127.0.0.1:9200/',
            'INDEX_NAME': 'readthedocs',
        },
    } 
    
    FILE_SYNCER = 'readthedocs.privacy.backends.syncers.DoubleRemotePuller'
    
    REDIS = {
        'host': 'localhost',
        'port': 6379,
        'db': 0,
    }
    BROKER_URL = 'redis://localhost:6379/0'
    CELERYD_HIJACK_ROOT_LOGGER = True
    CELERY_EAGER_PROPAGATES_EXCEPTIONS = True
    #CELERY_REDIRECT_STDOUTS_LEVEL = 'DEBUG'
    CELERY_ALWAYS_EAGER = True
    CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
    
    CACHES = {
        'default': {
            'BACKEND': 'redis_cache.RedisCache',
            'LOCATION': 'localhost:6379',
            'PREFIX': 'docs',
            'OPTIONS': {
                'DB': 1,
                'PARSER_CLASS': 'redis.connection.HiredisParser'
            },
        },
    }
    
    if not os.environ.get('DJANGO_SETTINGS_SKIP_LOCAL', False):
        try:
            from local_settings import *  # noqa
        except ImportError:
            pass

------------------
Configure Database
------------------

    mysql> create database rtd;
    mysql> GRANT ALL PRIVILEGES ON rtd.* TO 'rtd'@'%' IDENTIFIED BY 'rtd123123';
    mysql> GRANT ALL PRIVILEGES ON rtd.* TO 'rtd'@'localhost' IDENTIFIED BY 'rtd123123';

    ./manage.py migrate
    ./manage.py createsuperuser
    ./manage.py loaddata test_data

------------
Apache setup
------------

    sudo apt-get install libapache2-mod-wsgi

this should be enabled automatically, it its not enable it:

    sudo a2enmod wsgi

setup a vhost (`/etc/apache2/sites-available/docs.my-domain.com.conf`)

    <VirtualHost *:80>
        ServerName docs.my-domain.com
        
        LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\"" common
		LogFormat "%{X-Forwarded-For}i %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\""  combined
		CustomLog  /usr/share/readthedocs/checkouts/readthedocs.org/logs/access_log combined
		ErrorLog   /usr/share/readthedocs/checkouts/readthedocs.org/logs/error_log
        
        Alias /static/admin /usr/share/readthedocs/lib/python2.7/site-packages/django/contrib/admin/static/admin
        Alias /static       /usr/share/readthedocs/checkouts/readthedocs.org/media/static
        Alias /media        /usr/share/readthedocs/checkouts/readthedocs.org/media

        WSGIPassAuthorization On
        WSGIScriptAlias / /usr/share/readthedocs/checkouts/readthedocs.org/readthedocs/wsgi.py
        WSGIProcessGroup www-data
        WSGIDaemonProcess www-data \
            python-path=/usr/share/readthedocs/checkouts/readthedocs.org/readthedocs/:/usr/share/readthedocs/lib/python2.7/site-packages/:/usr/share/readthedocs/:/usr/share/readthedocs/checkouts/readthedocs.org/:/usr/share/readthedocs/lib/python2.7/site-packages/django/db/backends/postgresql_psycopg2/:/usr/share/readthedocs/checkouts/readthedocs.org/readthedocs/settings/

        <Directory /usr/share/readthedocs/checkouts/readthedocs.org/readthedocs/>
            <Files wsgi.py>
                Require all granted
            </Files>
        </Directory>
        <Directory /usr/share/readthedocs/lib/python2.7/site-packages/django/contrib/admin/static/admin/>
                Require all granted
        </Directory>
        <Directory /usr/share/readthedocs/checkouts/readthedocs.org/readthedocs/static/>
                Require all granted
        </Directory>
        <Directory /usr/share/readthedocs/checkouts/readthedocs.org/media/>
                Require all granted
        </Directory>
    </VirtualHost>

fix the permissions for apache:

    sudo chown -R www-data: /usr/share/readthedocs

and create cache dir in apache home directory:

    sudo mkdir /var/www/.cache/
    sudo chown www-data: /var/www/.cache

enable the vhost and restart:

    sudo a2ensite docs.my-domain.com.conf
    sudo service apache2 graceful

ElasticSearch

    sudo apt-get install icedtea-7-jre-jamvm
    wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
    echo "deb http://packages.elastic.co/elasticsearch/1.7/debian stable main" | sudo tee -a /etc/apt/sources.list.d/elasticsearch-1.7.list
    sudo apt-get update && sudo apt-get install elasticsearch

edit `/etc/elasticsearch/elasticsearch.yml` and add the lines:

    network.bind_host: localhost
    network.host: localhost

and restart ES:

    sudo service elasticsearch restart

add icu as specified:

    sudo /usr/share/elasticsearch/bin/plugin -install elasticsearch/elasticsearch-analysis-icu/2.3.0

and restart apache
