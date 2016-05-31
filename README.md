Please bear in mind this is for a Ubuntu 14.04 environment

============
Installation
============

    sudo mkdir /usr/share/readthedocs
    sudo chown your_username readthedocs
    sudo apt-get install build-essential python-dev libxml2-dev libxslt1-dev zlib1g-dev git texlive-latex-base texlive-latex-recommended texlive-lang-cjk chktex texlive-fonts-extra texlive-fonts-recommended -y
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
    # make sure the system virtualenv version
    
    deactive
    pip install virtualenv==15.0.1
    

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
    
    DEBUG = True
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
    # You will need to start a celery server by running `./manage.py celeryd`
    # if CELERY_ALWAYS_EAGER is set to False
    CELERY_ALWAYS_EAGER = False
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

Fix the permissions for apache:

    sudo chown -R www-data: /usr/share/readthedocs

Create cache dir in apache home directory:

    sudo mkdir /var/www/.cache/
    sudo chown www-data: /var/www/.cache

Enable the vhost and restart:

    sudo a2ensite docs.my-domain.com.conf
    sudo service apache2 restart

-------------
ElasticSearch
-------------

    sudo apt-get install openjdk-7-jre-headless
    wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
    echo "deb http://packages.elastic.co/elasticsearch/1.3/debian stable main" | sudo tee -a /etc/apt/sources.list.d/elasticsearch-1.3.list
    sudo apt-get update && sudo apt-get install elasticsearch

Edit `/etc/elasticsearch/elasticsearch.yml` and add the lines:

    network.bind_host: localhost
    network.host: localhost

Configure Elasticsearch to automatically start during bootup:

    sudo update-rc.d elasticsearch defaults 95 10
    
Add icu as specified:

    sudo /usr/share/elasticsearch/bin/plugin -install elasticsearch/elasticsearch-analysis-icu/2.3.0

Restart ES:

    sudo service elasticsearch restart

----------------
Setup ES mapping
----------------

Add a new script `/usr/share/readthedocs/checkouts/readthedocs.org/put_mapping.py`

    import os

    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "readthedocs.settings.postgres")

    from readthedocs.search.indexes import Index, PageIndex, ProjectIndex, SectionIndex

    # Create the index.
    index = Index()
    index_name = index.timestamped_index()
    index.create_index(index_name)
    index.update_aliases(index_name)
    # Update mapping
    proj = ProjectIndex()
    proj.put_mapping()
    page = PageIndex()
    page.put_mapping()
    sec = SectionIndex()
    sec.put_mapping()
    
Running the script:

    source /usr/share/readthedocs/bin/active
    python put_mapping.py

--------------
Running celery
--------------

    ./manage.py celeryd_detach
or
    [Daemonizing](http://docs.celeryproject.org/en/latest/tutorials/daemonizing.html)
