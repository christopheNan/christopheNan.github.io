Installation du CRM Creme
================================

Installation sur un Raspberry Pi 3
----------------------------------

L'installation se fait classiquement et est expliquée sur
`le forum de Creme`_.
J'apporte quelques précisions supplémentaires.

Prérequis
*********

installer mercurial, python3, pyhon3-dev, openjdk, libopenjp2 :

.. code-block:: bash

   pi@raspberry:~ $ sudo apt-get install mercurial python3-venv python3-dev openjdk-8-jdk libopenjp2-7

Création d'un utilisateur particulier
*************************************

Pour le service web, avec les répertoires utiles pour les différentes
applications :

.. code-block:: bash

    pi@raspberry:~ $ sudo adduser creme
    pi@raspberry:~ $ sudo mkdir /home/creme/{log,nginx,ssl,uwsgi}
    pi@raspberry:~ $ sudo chown creme:creme /home/creme/{log,nginx,ssl,uwsgi}
    pi@raspberry:~ $ # on prépare les services qui seront installés
    pi@raspberry:~ $ sudo ln -s /home/creme/nginx/nginx.service /etc/systemd/system/nginx.service
    pi@raspberry:~ $ sudo ln -s /home/creme/uwsgi/uwsgi.service /etc/systemd/system/uwsgi.service
    pi@raspberry:~ $ sudo su creme

Environnement virtuel et téléchargement du CRM
***********************************************

Installer creme dans un environnement virtuel de cet utilisateur :

.. code-block:: bash

    creme@raspberry:~ $ mkdir -p ~/.Envs/creme21 && python3 -m venv ~/.Envs/creme21
    creme@raspberry:~ $ ln -s ~/.Envs/creme21 ~/.Envs/creme  # pour être indépendant de la version
    creme@raspberry:~ $ source ~/.Envs/creme/bin/activate
    (creme)creme@raspberry:~ $ hg clone https://bitbucket.org/hybird/creme_crm-2.1
    ...

Pour Creme aussi, créer un lien symbolique indépendant de la version pour
faciliter les mises à jour et migrations futures :

.. code-block:: bash

    creme@raspberry:~ $ ln -s ~/creme_crm-2.1 ~/creme_crm

Fichiers media
**************

Comme nous sommes en production avec un frontal web, il faut générer les
fichiers media optimisés :

.. code-block:: bash

    (creme)creme@raspberry:~/creme_crm $ python manage.py generatemedia


Nginx
*****

Installer nginx en frontal web et pour gérer le SSL.

Attention, je n'ai pu faire fonctionner nginx qu'en configurant
``accept_mutex`` sur ``off`` sur mon raspberry. :

.. code-block:: bash

    pi@raspberry:~ $ sudo apt-get install nginx
    pi@raspberry:~ $ # copier le fichier de configuration ci-dessous
    pi@raspberry:~ $ # dans le répertoire nginx de l'utilisateur creme
    pi@raspberry:~ $ cat /home/creme/nginx/nginx.conf
    user              creme;
    worker_processes  4;
    error_log         /home/creme/log/nginx_error.log;
    pid               /run/creme/nginx.pid;


    events {
        worker_connections  8;
        accept_mutex off;
    }


    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /home/creme/log/access.log  main;

        sendfile        on;

        keepalive_timeout  65;

        upstream django {
            server unix:///home/creme/creme.sock; # file socket
        }

        # configuration of the server
        server {
            listen      443;
            # the domain name it will serve for
            server_name 192.168.1.54; # votre IP machine ou FQDN
            charset     utf-8;

            ssl                    on;
            ssl_certificate        /home/creme/ssl/cert.pem;
            ssl_certificate_key    /home/creme/ssl/key.pem;
            ssl_client_certificate /home/creme/ssl/ac.pem;
            ssl_verify_client      on;
            ssl_session_timeout    5m;
            ssl_protocols          TLSv1.2;
            ssl_ciphers            HIGH:!aNULL:!MD5;
            ssl_prefer_server_ciphers   on;

            # max upload size
            client_max_body_size 75M;

            # Django media
            location /media  {
                alias /home/creme/creme_crm/creme/media;  # Creme media files
            }

            location /static_media {
                alias /home/creme/creme_crm/creme/media/static ; # Creme static files
            }

            # Tout ce qui n'est pas media vers le serveur django.
            location / {
                uwsgi_pass   django;
                uwsgi_param  QUERY_STRING       $query_string;
                uwsgi_param  REQUEST_METHOD     $request_method;
                uwsgi_param  CONTENT_TYPE       $content_type;
                uwsgi_param  CONTENT_LENGTH     $content_length;

                uwsgi_param  REQUEST_URI        $request_uri;
                uwsgi_param  PATH_INFO          $document_uri;
                uwsgi_param  DOCUMENT_ROOT      $document_root;
                uwsgi_param  SERVER_PROTOCOL    $server_protocol;
                uwsgi_param  REQUEST_SCHEME     $scheme;
                uwsgi_param  HTTPS              $https if_not_empty;

                uwsgi_param  REMOTE_ADDR        $remote_addr;
                uwsgi_param  REMOTE_USER        $ssl_client_s_dn;
                uwsgi_param  REMOTE_PORT        $remote_port;
                uwsgi_param  SERVER_PORT        $server_port;
                uwsgi_param  SERVER_NAME        $server_name;
            }
        }
    }

Ensuite, configurer nginx en tant que service systemd :

.. code-block:: bash

    pi@raspberry:~ $ cat /home/creme/nginx/nginx.service
    [Unit]
    Description=reverse proxy server
    After=uwsgi.service

    [Service]
    Type=forking
    PIDFile=/run/creme/nginx.pid
    ExecStartPre=/usr/sbin/nginx -t -c /home/creme/nginx/nginx.conf
    ExecStart=/usr/sbin/nginx -c /home/creme/nginx/nginx.conf
    ExecReload=/usr/sbin/nginx -c /home/creme/nginx/nginx.conf -s reload
    ExecStop=/usr/sbin/nginx -s quit

    [Install]
    WantedBy=multi-user.target


Uwsgi
*****

Installer uwsgi pour servir les fichiers django :

.. code-block:: bash

    (creme)creme@raspberry:~ $ pip install uwsgi
    (creme)creme@raspberry:~ $ cat /home/creme/uwsgi/uwsgi.service
    [Unit]
    Description=serveur creme
    After=nginx.service

    [Service]
    Type=forking
    User=creme
    Group=creme
    RuntimeDirectory=creme
    PIDFile=/run/creme/uwsgi.pid
    ExecStart=/home/creme/.Envs/creme/bin/uwsgi --ini /home/creme/uwsgi/creme_uwsgi.ini --daemonize /home/creme/log/uwsgi.log
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target

    (creme)creme@raspberry:~ $ cat /home/creme/uwsgi/creme_uwsgi.ini
    # creme_uwsgi.ini file
    [uwsgi]
    ~
    chdir           = /home/creme/creme_crm
    module          = creme.wsgi
    home            = /home/creme/.Envs/creme
    ~
    master          = true
    processes       = 5
    socket          = /home/creme/creme.sock
    chmod-socket    = 666
    vacuum          = true
    safe-pidfile    = /run/creme/uwsgi.pid

S'assurer de disposer d'un fichier `/home/creme/creme_crm/creme/wsgi.py`
(c'est le fichier *module* du fichier de configuration ci-dessus). S'il n'est
pas présent, voici le contenu du mien :

.. code-block:: python

    import os
    from os.path import dirname, abspath
    import sys


    CREME_ROOT = dirname(abspath(__file__))
    sys.path.append(CREME_ROOT)

    os.environ['DJANGO_SETTINGS_MODULE'] = 'creme.settings'

    from django.core.wsgi import get_wsgi_application
    application = get_wsgi_application()


Configuration de l'authentification par certificat pour les clients
===================================================================

configuration de Creme / Django
-------------------------------

Je suppose dans ce guide que les comptes django des utilisateurs ont été
préalablement créés et que l'authentification concerne uniquement donc
uniquement des utilisateurs déjà existants.

Ajout de l'application mobile
*****************************

L'application `mobile` est désactivée par défaut dans creme. Il convient de
l'activer avant de faire la migration de la base de données et la génération
des medias. Pour l'activer, dans le fichier `creme/settings.py`,
supprimer le caractère croisillon devant `creme.mobile`.

Middlewares
***********

Modifions les middlewares pour authentifier automatiquement à partir de la
variable ``REMOTE_USER`` dans les entêtes de la requête web. Dans le
fichier ``creme/local_settings.py``, rajouter la variable suivante :

.. code-block:: python

    LOCAL_MIDDLEWARE = [
      'django.contrib.auth.middleware.AuthenticationMiddleware',
      'django.contrib.auth.middleware.RemoteUserMiddleware',
    ]

Dans le fichier ``creme/settings.py``, ajouter les lignes suivantes
  en fin de fichier pour prendre en compte la variable de
  ``creme/local_settings.py`` :

.. code-block:: python

    MIDDLEWARE = MIDDLEWARE + LOCAL_MIDDLEWARE

Module d'authentification personnalisé
**************************************

Ajouter un module d'authentification personnalisé. Vous pouvez placer ce
fichier dans le répertoire où vous avez installé creme et l'appeler par
exemple ``monauth.py``. Dans le fichier ``creme/local_settings.py``, rajouter
alors la variable suivante :

.. code-block:: python

    AUTHENTICATION_BACKENDS = ('monauth.PropagationBackend',)

Ce module hérite de RemoteUserBackend pour lire les informations dans
``REMOTE_USER``. La fonction ``clean_username`` est modifiée pour extraire le
nom d'utilisateur à partir de ``REMOTE_USER`` (qui contient le DN du
certificat, voir la section `Nginx`_).
La variable de classe ``create_unknown_user`` est placée à ``False`` pour ne
pas créer d'utilisateur dans la base automatiquement. Je n'ai pas testé de la
placer à ``True``, ce qui pourrait fonctionner car `Nginx`_ est configuré
pour n'accepter que des clients qui ont des certificats.
La fonction ``has_perm`` est tirée de la fonction standard d'authentification
de Creme (dans le fichier
``/home/creme/creme_crm/creme/creme_core/auth/backend.py``) et dépend donc de
votre version installée de Creme (ici 2.1) :

.. code-block:: python

    from creme.creme_core.auth.entity_credentials import EntityCredentials
    from creme.creme_core.auth import SUPERUSER_PERM

    from django.contrib.auth.backends import RemoteUserBackend

    _ADD_PREFIX = 'add_'
    _EXPORT_PREFIX = 'export_'


    class PropagationBackend(RemoteUserBackend):
        supports_object_permissions = True
        create_unknown_user = False

        def clean_username(self, remote_user):
            return remote_user.split('/')[-1].split('=')[-1]

        def has_perm(self, user_obj, perm, obj=None):
            if obj is not None:
                return EntityCredentials(user_obj, obj).has_perm(perm)

            if user_obj.role is not None:
                app_name, dot, action_name = perm.partition('.')

                if not action_name:
                    return user_obj.is_superuser if app_name == SUPERUSER_PERM else \
                           user_obj.has_perm_to_access(app_name)

                if action_name == 'can_admin':
                    return user_obj.has_perm_to_admin(app_name)

                if action_name.startswith(_ADD_PREFIX):
                    return user_obj.role.can_create(app_name, action_name[len(_ADD_PREFIX):])

                if action_name.startswith(_EXPORT_PREFIX):
                    return user_obj.role.can_export(app_name, action_name[len(_EXPORT_PREFIX):])

            return False

Voilà, il ne reste plus qu'à lancer django :

.. code-block:: bash

    pi@raspberry:~ $ sudo systemctl start uwsgi.service
    pi@raspberry:~ $ sudo systemctl start nginx.service

.. _le forum de Creme: https://www.cremecrm.com/forum/showthread.php?tid=126


Installation d'une version pour pré-production
==============================================

Afin de tester une nouvelle version de creme avant sa mise en production,
j'ai mis en place la configuration suivante.

liens symboliques pré-production
--------------------------------
Nous avons déjà mis en place des liens symboliques pour la production. Mettons
ceux pour la pré-production (supposons pour une version 2.1 de creme) pour
l'environnement virtuel Python et creme.

.. code-block:: bash

    creme@raspberry:~ $ ln -s ~/.Envs/creme21 ~/.Envs/creme_preprod
    creme@raspberry:~ $ source ~/.Envs/creme_preprod/bin/activate
    (creme_preprod)creme@raspberry:~ $ ln -s creme_crm-2.1 creme_preprod


Modification des fichiers de configuration
------------------------------------------

Je garde un seul frontal nginx, mais qui dessert deux uwsgi. Le site de
pré-production a la même adresse mais est préfixé par `preprod/`. Il faut donc
rajouter des sections `location` au fichier de configuration de nginx :

.. code-block:: bash

    creme@raspberry:~ $ cat /home/creme/nginx/nginx.conf
    ...
    # section à insérer dans la zone http
        upstream django_preprod {
            server unix:///home/creme/creme_preprod.sock; # for a file socket
        }
    # ...

    # sections à insérer dans la zone server

       # Pre-production site
        location /preprod/media  {
            alias /home/creme/creme_preprod/creme/media;
        }

        location /preprod/static_media {
            alias /home/creme/creme_preprod/creme/media/static;
        }

        # Finally, send all non-media requests to the Django server.
        location /preprod/ {
            uwsgi_pass  django_preprod;
            include     /home/creme/nginx/uwsgi_params.conf;
        }

        # Production site
        location /media  {
            alias /home/creme/creme_preprod/creme/media;
        }

Il faut dupliquer le fichier `uwsgi/creme_uwsgi.ini` vers
`uwsgi/creme_uwsgi_preprod.conf` et modifier le contenu de ce nouveau fichier
comme ceci :

.. code-block:: bash

    creme@raspberry:~ $ cat /home/creme/uwsgi/creme_uwsgi_preprod.ini
    # creme_uwsgi_preprod.ini file
    [uwsgi]
    ~
    chdir           = /home/creme/creme_preprod
    module          = creme.wsgi
    home            = /home/creme/.Envs/creme_preprod
    ~
    master          = true
    processes       = 5
    socket          = /home/creme/creme_preprod.sock
    chmod-socket    = 666
    vacuum          = true
    safe-pidfile    = /run/creme/uwsgi_preprod.pid

Modification de creme/django
----------------------------

Modifions creme/django pour utiliser un préfixe `preprod/` dans les urls. Dans
le fichier `creme/local_settings.py`, on ajoute les lignes suivantes :

.. code-block:: bash

    creme@raspberry:~ $ tail -n 8 /home/creme/creme_preprod/creme/local_settings.py
    # For pre-production usage only
    URL_PREFIX = 'preprod/'
    MEDIA_URL = 'http://127.0.0.1:8000/{prefix}media/'.format(prefix=URL_PREFIX)
    PRODUCTION_MEDIA_URL = '/{prefix}static_media/'.format(prefix=URL_PREFIX)

    if URL_PREFIX:
        DEBUG = True

Ensuite, modifions le fichier `creme/urls.py` pour rajouter le préfixe à toutes
les urls :

.. code-block:: bash

    creme@raspberry:~ $ tail -n 5 /home/creme/creme_preprod/creme/local_settings.py

    # Ne pas oublier d'ajouter l'import de la fonction path du module django.urls
    if settings.URL_PREFIX:
        urlpatterns = [path(r'{prefix}/'.format(prefix=settings.URL_PREFIX), include(urlpatterns))]


Regénération des fichiers média
-------------------------------

Prenons en compte les modifications de la configuration :

.. code-block:: bash

    (creme_preprod)creme@raspberry:~/creme_preprod $ python manage.py generatemedia


La version pré-production de creme doit être fonctionnelle. Il n'y a plus
qu'à tester.

Passage en production
---------------------
Le passage en production se fait en basculant les liens symboliques vers les
versions des environnements virtuels et de creme :

.. code-block:: bash

    creme@raspberry:~ $ rm ~/creme_crm ~/.Envs/creme
    creme@raspberry:~ $ ln -s ~/.Envs/creme21 ~/.Envs/creme
    creme@raspberry:~ $ ln -s ~/creme_crm-2.1 ~/creme_crm

En enlevant la configuration de pré-production des fichiers de configuration,
notamment la variable `URL_PREFIX` de `local_settings.py`.

Prenons en compte les modifications de la configuration dans les fichiers
media :

.. code-block:: bash

    creme@raspberry:~/creme_crm $ source ../../.Envs/creme/bin/activate
    (creme)creme@raspberry:~/creme_crm $ python manage.py generatemedia


La version « production » de creme doit être maintenant fonctionnelle.
il suffit donc de relancer les services avec `systemctl`.

Attention toutefois à la base de données : la base de production et celle de
pré-production sont différentes. Si la production et la pré-production ne
sont pas dans la même version de creme, il faudra faire une migration. Si les
versions sont les mêmes, une copie de la base de production doit suffire.
