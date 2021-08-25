---
title: 'Installation d\''un raspberry Pi 3'
---

Installation d\'une distribution de base sans écran ni clavier
==============================================================

Tout est expliqué sur [le site de
raspbian](https://raspbian-france.fr/raspberry-pi-sans-ecran-sans-clavier/)
mais je rappelle quelques points.

-   après avoir copié la distribution sur une carte SD, créer un fichier
    `ssh` dans la partition `boot` de la carte SD. Au démarrage, le
    Raspberry vérifie si un tel fichier existe pour démarrer le service
    ssh.
-   Effectuer quelques configurations avec :

``` {.bash}
pi@raspberry:~ $ sudo raspi-config
```

pour impérativement :

:   -   changer le mot de passe du compte pi,

et par exemple :

:   -   changer le nom de la machine,
    -   configurer les locales en français,
    -   activer I2C (pour le capteur de luminosité TSL 2561),
    -   étendre les partitions à l\'ensemble de la carte SD.

-   mettre à jour la distribution :

``` {.bash}
pi@raspberry:~ $ sudo apt-get update
pi@raspberry:~ $ sudo apt-get upgrade
```

-   copier ma clé publique ssh sur le raspberry pour un accès plus
    sécurisé et plus facile :

``` {.bash}
[chris@ordi ~]$ ssh-copy-id -i ~/.ssh/id_rsa pi@raspberry
```

Envoi de mails
--------------

### avec ssmtp

Très simple mais ne comporte pas de tampon, donc le message est perdu s'il
n'y a pas de connectivité Internet par exemple.


``` {.bash}
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
```

### avec postfix

Moins simple, mais les mails sont conservés le temps que la machine arrive
à les envoyer.
On utilise une configuration en relais pur.

-   installer postfix et les bibliothèques SASL

``` {.bash}
pi@raspberry:~ $ sudo apt-get install postfix libsasl2-modules
```

-   configurer postfix

``` {.bash}
pi@raspberry:~ $ sudo cat /etc/postfix/main.cf
relayhost = [smtp.gmail.com]:587
smtp_use_tls = yes
# activer l'authentication SASL
smtp_sasl_auth_enable = yes
# emplacement du fichier des mots de passe
smtp_sasl_password_maps = hash:/etc/postfix/sasl/sasl_passwd
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
# Enable STARTTLS encryption
smtp_tls_security_level = encrypt

pi@raspberry:~ $ sudo cat /etc/postfix/sasl/sasl_passwd
[smtp.gmail.com]:587    username@gmail.com:password

pi@raspberry:~ $
```
-    compiler les fichiers de mots de passe

``` {.bash}
pi@raspberry:~ $ sudo postmap /etc/postfix/sasl/sasl_passwd
```


Installation de log2ram
=======================

[log2ram](https://github.com/azlux/log2ram) permet d\'écrire les logs en
RAM et donc d\'économiser la carte SD.

Attention : vos logs peuvent être saturés si votre machine ouvre des
ports sur Internet. Le fichier btmp, qui stocke tous les échecs
d\'authentification, peut saturer votre ramdisk et empêcher le
fonctionnement correct de la machine.
