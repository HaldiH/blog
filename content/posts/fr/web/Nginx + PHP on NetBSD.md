---
title: Nginx + PHP sur NetBSD
date: 2020-10-09T02:47:02+02:00
description: Installation de Nginx et PHP sur NetBSD
draft: false
authors: [Hugo Haldi]
weight: 01
tags: ["SysAdmin", "web"]
---

## Introduction

Vous avez installé NetBSD sur votre grille-pain, mais vous ne savez pas quoi en faire ? Ne cherchez plus! Grâce à cet article, vous pourrez héberger votre site web sur votre merveilleux appareil à fabriquer des toasts.

Cet article va traiter de l'installation du gestionnaire de paquets [pkgin](https://pkgin.net), de [nginx](https://www.nginx.com) et de [php](https://www.php.net), ainsi de la configuration de ces derniers.

## Installation de pkgin

Si vous avez déjà pkgin installé sur votre NetBSD, vous pouvez passer à la [section suivante](#php-fpm).

pkgin est un gestionnaire de paquets pour NetBSD, à la manière de apt-get si vous avez déjà touché à Debian ou Ubuntu, pacman pour Archlinux, yum pour Fedora et CentOS, etc.

Pour l'installer, on va utiliser sa version précompilée fournie par NetBSD. La commande `pkg_add`, de l'outil [pkgsrc](https://www.netbsd.org/docs/pkgsrc/) permet d'installer un paquet à partir d'une URL. On peut définir la variable d'environnement `PKG_PATH` afin de dire à `pkg_add` où trouver les paquets

```shell
export PKG_PATH="http://ftp.netbsd.org/pub/pkgsrc/packages/NetBSD/$(uname -p)/$(uname -r)/All"
```

Et maintenant, on va installer pkgin

```shell
pkg_add pkgin
```

On peut éventuellement éditer la configuration de pkgin

```shell
vi /usr/pkg/etc/pkgin/repositories.conf
```

Je change la variable `$arch` vers `aarch64` dans mon cas, car mon grille-pain se trouve être un Raspberry Pi 3, sinon renseigner votre architecture (`uname -p`)

```ini
https://cdn.netbsd.org/pub/pkgsrc/packages/NetBSD/aarch64/9.0/All
```

Maintenant on met à jour la liste des paquets

```shell
pkgin update
```

Ça y est, vous avez un gestionnaire de paquets tout beau, tout neuf ! Passons maintenant au vif du sujet.

## PHP-FPM

PHP-FPM (FastCGI Process Manager) est une API permettant de connecter un serveur web (Nginx dans notre cas) et PHP, via l'utilisation d'un socket.

### Installation de PHP-FPM

Pour l'installer, rien de plus simple

```shell
pkgin in php74-fpm
```

Vous remarquerez le *74* dans *php74-fpm*; il s'agit en fait de la version 7.4 de PHP. Vous pouvez faire un `pkgin search php` pour chercher une version plus récente de *php-fpm*, PHP 7.4 étant la dernière en date au moment de la rédaction.

### Configuration de PHP-FPM

On remarquera que le gentil monsieur nous dit quoi faire, en effet dans le terminal on peut lire

```console
The following files should be created for php74-fpm-7.4.10nb4:

        /etc/rc.d/php_fpm (m=0755)
            [/usr/pkg/share/examples/rc.d/php_fpm]
```

Alors on peut simplement copier le fichier `php_fpm` dans `/etc/rc.d/`, étant le fichier d'initialisation du serveur (celui qui permet le démarrage, l'arrêt, le redémarage, ... de php-fpm)

```shell
cp /usr/pkg/share/examples/rc.d/php_fpm /etc/rc.d/php_fpm
```

Le [système rc.d](https://www.netbsd.org/docs/guide/en/chap-rc.html) permet d'exécuter des programmes automatiquement au démarrage du système. On peut donc ajouter la règle d'initialisation de php-fpm dans `/etc/rc.conf`. À la fin du fichier, ajouter

```ini
php_fpm=YES
```

Maintenant il ne nous reste plus qu'à démarer le service

```shell
service php_fpm start
```

Si vous obtenez le message *Starting php_fpm.*, c'est parfait !

C'est tout pour PHP. Maintenant passons à Nginx.

## NGINX

Nginx (à prononcer engine-x) est un serveur HTTP assez léger écrit en C++, qui conviendra parfaitement à nos besoins de légerté concernant l'hébergement web chez le pain de mie (le grille-pain, vous vous souvenez ?)

### Installation de Nginx

Pour installer Nginx et toutes ses dépendances

```shell
pkgin in nginx
```

### Configuration de Nginx

Comme avant, l'installeur nous donne quelques tâches à faire pour finaliser l'installation

```console
The following files should be created for nginx-1.19.2:

        /etc/rc.d/nginx (m=0755)
            [/usr/pkg/share/examples/rc.d/nginx]
```

On va donc copier le fichier d'initialisation de nginx

```shell
cp /usr/pkg/share/examples/rc.d/nginx /etc/rc.d/nginx
```

Puis éditer `/etc/rc.conf` pour y ajouter à la fin

```ini
nginx=YES
```

On peut à présent démarer Nginx

```shell
service nginx start
```

Si tout se passe bien, vous obenez le message *Starting nginx.*

### Hello, World

Ca y est ! Vous pouvez dès à présent accéder à votre grille-pain du futur depuis votre smartphone via le navigateur. Vous allez obtenir la page par défaut de Nginx

![Nginx welcome](/img/Nginx_welcome.png)

C'est déjà bien, mais avec ça vous ne pouvez toujours pas exécuter de PHP côté serveur. Il va falloir approfondir les configurations.

### Configuration avancée de Nginx

On va éditer la configuration de Nginx qui se trouve dans `/usr/pkg/etc/nginx/nginx.conf`. Modifier le champs *server* pour obtenir un résultat similaire

```nginx
server {
    listen       80;
    server_name  localhost;
    root         /var/www; # Racine des documents; vous pouvez la modifier si besoin
    #charset koi8-r;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        index index.html index.htm index.php;
        try_files $uri $uri/ /index.html;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   share/examples/nginx/html;
    }

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include        /usr/pkg/etc/nginx/fastcgi_params;
    }
}
```

Il est aussi nécessaire de modifier l'utilisateur et le groupe dans la configuration de php-fpm `/usr/pkg/etc/php-fpm.d/www.conf`

```ini
user = nginx
group = nginx
```

Maintenant on peut redémarer les services

```shell
service php_fpm restart
service nginx restart
```

On peut maintenant créer un fichier à la racine de notre serveur web `/var/www/index.php` (ou votre racine si vous en avez choisi une autre)

```php
<?php
phpinfo();
?>
```

Et voilà ! Vous devriez tomber sur la page d'information PHP en essayant d'accéder à votre machine via un navigateur.

## Conclusion

Maintenant que vous avez un seveur web Nginx + PHP, vous pouvez héberger toutes sortes de contenus. Cependant beaucoup d'applications PHP requiert des bibliothèques supplémentaires pour fonctionner.

Par exemple, la bibliothèque [php-intl](https://www.php.net/manual/en/book.intl.php) qui permet d'étendre la prise en charge des caractères [Unicode](https://en.wikipedia.org/wiki/Unicode), peut s'installer sur PHP 7.4 de cette manière

```shell
pkgin in php74-intl
```

Et il reste à redémarer le serveur PHP-FPM pour prendre en compte les changements

```shell
service php_fpm restart
```

De prochains articles sur l'installations d'applications web comme [Nextcloud](https://nextcloud.com) ou [Passbolt](https://www.passbolt.com) permettront d'approfondir les détails de configuration de ces services.
