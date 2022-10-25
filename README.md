# Nginx + SSL + PHP + docker-compose

Ce repo est un exemple de configuration d'un site en PHP servi par Nginx via HTTPS dans un environnement géré par docker-compose.

L'objectif de cette démo est de faire servir à Nginx en HTTPS le résultat d'un script PHP affichant un `phpinfo()`.

:warning: _**Si vous suivez ce pas-à-pas, pensez à remplacer le domaine `example.com` avec le nom d'hôte de votre VM partout où vous le rencontrerez**_

:information_source: Les ressources présentées ici sont adaptées de cet article : https://pentacent.medium.com/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71

## Étape 1 : configurer Nginx et PHP pour servir le contenu

La configuration du serveur indiquera que :
- la racine des documents est `/var/www/html/public` (dans le conteneur "nginx")
- si le chemin de la requête HTTP ne correspond à aucun fichier sur dans la racine des documents, la requête sera traitée par le fichier `index.php`
- les fichiers portant l'extension `.php` seront interprétés par le service "php"

La configuration du serveur se trouve dans le fichier [`.docker/nginx/site.conf`](./.docker/nginx/site.conf). Le service "nginx" sera dépendant du service "php" car sans interpréteur PHP, Nginx ne pourra pas traiter les requêtes impliquant du code PHP.

Pour cette démonstration, nous utiliserons directement une image de PHP-FPM car nous n'avons pas besoin de plus ; il reste possible de customiser la configuration du service "php" en se basant sur un `Dockerfile`.

```yaml
services:
    nginx:
        image: nginx:alpine
        volumes:
            - ./.docker/nginx/site.conf:/etc/nginx/conf.d/default.conf
            - ./app:/var/www/html
        ports:
            - "80:80"
        depends_on:
            - php
    php:
        image: php:fpm-alpine
        volumes:
            - ./app:/var/www/html
```

:warning: Contenu de `.docker/nginx/site.conf` pour avoir un site fonctionnel à ce stade :
```
server {
    listen 80;

    server_name example.com;

    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    
    index index.php index.html;
    root /var/www/html/public;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

## Étape 2 : ajouter le service Certbot

Certbot est un programme qui facilite l'obtention de certificats SSL via Let's Encrypt.

Lorsqu'il fait une demande de délivrance de certificat SSL, Certbot enregistre le certificat et d'autres informations dans le répertoire `/etc/letsencrypt/`.

Pendant le processus d'obtention d'un certificat, les serveurs de Let's Encrypt ont besoin de vérifier que le serveur pour lequel Certbot fait la demande est bien géré par la même autorité qui utilise Certbot ; Certbot a donc besoin d'un répertoire accessible depuis le web pour y déposer les ressources qui serviront à la vérification (ici, on opte pour `/var/www/certbot`).

```yml
services:
    # ...
    certbot:
        image: certbot/certbot:latest
        volumes:
            - ./.docker/certbot/conf:/etc/letsencrypt
            - ./.docker/certbot/www:/var/www/certbot
```

Pour rendre `/var/www/certbot` accessible depuis le web, on configurera Nginx pour servir son contenu dans deux étapes :ok_hand:

## Étape 3 : configurer Nginx pour HTTPS

Pour utiliser un certificat SSL on a besoin de customiser la configuration du site côté Nginx, notamment :
- faire écouter Nginx sur le port 443 (port standard pour HTTPS)
- activer l'utilisation d'un certificat SSL
- indiquer à Nginx l'emplacement des fichiers du certificat SSL
- faire rediriger le traffic HTTP vers HTTPS (par obligatoire mais plus secure)

La nouvelle configuration de notre site devrait ressembler à ceci :

```
server {
    listen 80;

    server_name example.com;
    
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;

    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    
    index index.php index.html;
    root /var/www/html/public;

    # Permet d'utiliser index.php comme front controller
    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    # Permet de faire interpréter le PHP par le service "php"
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

La nouvelle configuration du serveur indique que l'écoute des requêtes en HTTPS se fera sur le port 443 du conteneur, il faut donc l'ajouter aux ports exposés au système hôte et relayés par celui-ci. De plus, comme Nginx a besoin d'avoir accès aux certificats SSL, il faut partager au service le répertoire qui les contiendra en ajoutant un volume :

```yml
services:
    nginx:
        # ...
        ports:
            # ...
            - "443:443"
        # ...
        volumes:
            # ...
            - ./.docker/certbot/conf:/etc/letsencrypt
```

**À ce stade le service "nginx" ne démarrera plus car le certificat SSL n'existe pas (il n'a pas été délivré) !!!**

## Étape 4 : configurer Nginx pour valider la délivrance du certificat SSL

Comme expliqué plus haut, Let's Encrypt a besoin de vérifier que le serveur pour lequel on lui demande de délivrer un certificat SSL est bien administré par la même personne qui fait la demande via Certbot. Pour cela, Certbot met à disposition de Let's Encrypt un moyen de vérifier son autorité à l'URL suivante : `http://<hostname>/.well-known/acme-challenge/` (qui correspond à ce qui se trouve dans `/var/www/certbot` au niveau du conteneur).

Il faut donc que Nginx puisse servir le contenu de vérification généré par Certbot pour que Let's Encrypt puisse valider l'autorité du serveur. La solution ici est de partager le volume entre les services "certbot" et "nginx" (Certbot écrira dans le répertoire, Nginx le rendra accessible sur le web) :

```yml
services:
    nginx:
        # ...
        volumes:
            # ...
            - ./.docker/certbot/www:/var/www/certbot
```

Il faut ensuite indiquer à Nginx de ne pas rediriger les requêtes HTTP reçues pour `/.well-known/acme-challenge/` mais de les servir depuis le répertoire `/var/www/certbot` en modifiant la configuration du serveur :

```
server {
    listen 80;

    server_name example.com;
    
    # Redirige le traffice HTTP vers HTTPS
    location / {
        return 301 https://$host$request_uri;
    }

    # Sert à répondre au serveur de Let's Encrypt lors de la vérification pour la délivrance du certificat SSL
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}
# ...
```

**Non, notre configuration n'est pas encore fonctionnelle&hellip; mais presque ! Promis !**

## Étape 5 : obtenir le certificat SSL (enfin !)

Petit résumé du problème qu'on a actuellement :
- Nginx est configuré pour gérer les requêtes en HTTPS mais il nous manque le certificat SSL
- Nginx est configuré pour faciliter l'obtention du certificat SSL
- mais Nginx ne peut toujours pas démarrer parce qu'il n'a pas le certificat SSL qu'il doit contribuer à obtenir :exploding_head:

:egg: :chicken: ?

La solution est presque magique et réside dans ce petit script : [`init-letsencrypt.sh`](./init-letsencrypt.sh)

Ce script va, dans l'ordre :
- générer un faux certificat SSL dans `.docker/certbot/conf` pour que Nginx puisse démarrer
- démarrer le service Nginx pour rendre accessible les ressources que Let's Encrypt utilisera pour vérifier l'autorité du serveur
- supprimer le faux certificat (n'affectera pas Nginx tant que le serveur ne rechargera pas sa configuration)
- exécuter le service "certbot" pour obtenir un certificat SSL auprès de Let's Encrypt
- recharger la configuration du serveur Nginx (avec le bon certificat cette fois)

:warning: **Avant de lancer le script, il faut impérativement remplacer le domaine `example.com` par le nom d'hôte de votre VM et l'adresse e-mail `email@example.com` par une adresse e-mail valide dans le script, et rendre votre VM publique pour que Let's Encrypt puisse faire ses vérifications !!!**

:bulb: Let's Encrypt limite les demandes de certificats à 5 par heure et par domaine ; si vous atteignez la limite de 5 demandes refusées, vous ne pourrez pas renouveler votre demander pendant une heure. Afin de s'assurer que votre configuration fonctionne, vous pouvez indiquer au script que vous ne souhaitez pas obtenir de certificat mais juste tester votre setup (pas de limitation) en passant la variable `staging` à `1` (`0` pour un vrai certificat). Ceci vous permettra d'anticiper les erreurs d'otention mais ne vous permettra pas d'utiliser HTTPS puisqu'aucun certificat ne sera effectivement délivré.

_Are you ready ?_
```
sudo ./init-letsencrypt.sh
```

_(Si le script n'est pas exécutable, vous pouvez y remédier avec : `chmod +x init-letsencrypt.sh`)_

**À ce stade, votre configuration devrait être valide, tout comme votre certificat SSL ! Bravo !** :clap:

Si vous accédez à votre site en HTTP, vous devriez être automatiquement redirigé.e en HTTPS sans aucun avertissement de sécurité ! Alors, il est pas magnifique, ce `phpinfo()` ?

### Étape bonus : gérer le renouvellement automatique du certificat

Hé non, un certificat SSL n'est pas délivré pour une durée indéterminée car toutes les bonnes choses ont malheureusement une fin :wink:

Mais on a un service "certbot" qui, pour l'instant, est à usage unique&hellip; Est-ce qu'on ne pourrait pas le mettre à profit et lui faire renouveler le certificat quand celui-ci arrive à expiration ?

Si !

```yml
services:
    # ...
    certbot:
        # ...
        entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```

Et, quitte à automatiser des choses, on peut même automatiser le rechargement du certificat au niveau de Nginx, des fois que celui-ci viendrait à être renouveler alors que le serveur est en route&hellip;

```yml
services:
    nginx:
        # ...
        command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"
```
