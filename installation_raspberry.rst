Installation d'un raspberry Pi 3
================================

Installation d'une distribution de base sans écran ni clavier
-------------------------------------------------------------

Tout est expliqué
`ici <https://raspbian-france.fr/raspberry-pi-sans-ecran-sans-clavier/>`_
mais je rappelle quelques points.

- après avoir copié la distribution sur une carte SD, créer un fichier
  ``ssh`` dans la partition ``boot`` de la carte SD. Au démarrage, le
  Raspberry vérifie si un tel fichier existe pour démarrer le service ssh.

- Effectuer quelques configurations avec :

.. code-block:: bash

    pi@raspberry:~ $ sudo raspi-config

pour impérativement :
  - changer le mot de passe du compte pi,

et par exemple :
  - changer le nom de la machine,
  - configurer les locales en français,
  - activer I2C (pour le capteur de luminosité TSL 2561),
  - étendre les partitions à l'ensemble de la carte SD.

- mettre à jour la distribution :

.. code-block:: bash

    pi@raspberry:~ $ sudo apt-get update
    pi@raspberry:~ $ sudo apt-get upgrade


- copier ma clé publique ssh sur le raspberry pour un accès plus sécurisé
  et plus facile :

.. code-block:: bash

  [chris@ordi ~]$ ssh-copy-id -i ~/.ssh/id_rsa pi@raspberry


- installer ssmtp pour que les programmes puissent envoyer des mails :

.. code-block:: bash

    pi@raspberry:~ $ sudo apt-get install ssmtp
    pi@raspberry:~ $ sudo cat /etc/ssmtp/ssmtp.conf
    # l'adresse mail correspondant à root
    # c-à-d que si l'on écrit à root, cela arrive à votre.adresse@provider.com
    root=votre.adresse@provider.com
    # le serveur de mail sortant est gmail
    mailhub=smtp.gmail.com:587
    FromLineOverride=YES
    UseSTARTTLS=YES
    TLS_CA_File=/etc/pki/tls/certs/ca-bundle.crt
    # identifiant et mot de passe google
    # générer un mot de passe spécifique avec "mot de passe des applications"
    # sur myaccount.google.com
    AuthUser=mon.identifiant
    AuthPass=monmotdepasse
    # pour ré-écrire un peu le mail
    rewriteDomain=mon-domaine
    hostname=nomdemamachine

Installation de log2ram
-------------------------
`log2ram <https://github.com/azlux/log2ram>`_ permet d'écrire les logs en RAM
et donc d'économiser la carte SD.

Attention : vos logs peuvent être saturés si votre machine ouvre des ports
sur Internet. Le fichier btmp, qui stocke tous les échecs d'authentification,
peut saturer votre ramdisk et empêcher le fonctionnement correct de la
machine.
