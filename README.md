# docker-shibboleth-renater-sp

[![Docker Pulls](https://img.shields.io/docker/pulls/abesesr/docker-shibboleth-renater-sp.svg)](https://hub.docker.com/r/abesesr/docker-shibboleth-renater-sp/) [![build-test-pubtodockerhub](https://github.com/abes-esr/docker-shibboleth-renater-sp/actions/workflows/build-test-pubtodockerhub.yml/badge.svg)](https://github.com/abes-esr/docker-shibboleth-renater-sp/actions/workflows/build-test-pubtodockerhub.yml)

Ce dépôt propose une image docker générique permettant de mettre en place un fournisseur de service pré-connecté sur la [Fédération d'identités Education-Recherche (FER)](https://services.renater.fr/federation/index).

Fonctionnement : ce fournisseur de service s'intégre sur une application tiers en se positionnant en amont des flux HTTP et en jouant le rôle de reverse proxy authentifiant. C'est à dire qu'une requête HTTP permettant d'accéder à l'application va tout d'abord transiter par cette brique pour ensuite être transmise à l'application cible. L'avantage de cette approche c'est de pouvoir intégrer une authentification SAML sur une application sans avoir à la modifier en profondeur, c'est ce reverse proxy qui se charge de transmettre une preuve de l'authentification à l'application. Cette preuve contient l'identifiant de l'utilisateur ainsi que les attributs fournits par le fournisseur d'identités (ex: mail, nom, prénom, supannEtablissement, etc ...). A noter qu'il existe des avantages mais aussi des inconvéniants à utiliser ce type d'architecture, vous pouvez consulter [la documentation qui explique tout cela plus en détail](https://services.renater.fr/federation/documentation/generale/shib-et-reverseproxy).

Technologies : cette image docker utilise un serveur apache (basée sur son image docker officielle) et mod_shib (module Shibboleth s'intégrant dans apache). Ce sont les briques techniques préconisées par RENATER pour implémenter un fournisseur de service.

<img src="https://docs.google.com/drawings/d/e/2PACX-1vRMfhC8StjZi8KUXUnAoQA7MJG4BymqQue0dIxnsR9-VchR9dJTOh3tRU8j_3ngpvPaU9rReELFVrG8/pub?w=945&amp;h=665">

(le [lien](https://docs.google.com/drawings/d/1lGluW3Kpq7p2j2Hx8SmAqkDAKFmw6Zu7P8YuhTPV0ew/edit) pour modifier le schéma)



## Comment utiliser cette image ?

### Configuration

Les variables suivantes sont utilisées pour personnaliser votre conteneur :
- ``RENATER_SP_TEST_OR_PROD`` : pour basculer facilement le fournisseur de service sur la fédération RENATER de TEST ou de PROD (valeur par défaut: ``TEST``)
- ``RENATER_SP_ENTITY_ID`` : l'identifiant technique de votre fournisseur de service (valeur par défaut: non-renseigné ; exemple de valeur : ``https://apollo-dev.theses.fr/sp``)
- ``RENATER_SP_ADMIN_MAIL`` : l'adresse mail de contact qui sera utilisé dans les page d'erreur d'Apache et de Shibboleth (valeur par défaut : non-renseigné ; exemple de valeur: ``admin@theses.fr``)
- ``RENATER_SP_HTTPD_SERVER_NAME`` : le nom public du serveur web de votre fournisseur de service (valeur par défaut : non-renseigné ; exemple de valeur : ``https://apollo-dev.theses.fr``)
- ``RENATER_SP_HTTPD_LOG_LEVEL`` : le niveau de log du serveur apache (valeur par défaut ``info ssl:warn``)
- ``RENATER_SP_HTTPD_PROTECTED_PATH`` : le chemin à protéger par une authentification dans la fédération d'identités (valeur par défaut: ``/my-protected-url/``)
- ``RENATER_SP_HTTPD_PUBLIC_PROXY_TO`` : l'adresse du serveur (interne) de votre application qui sera "reverse proxifiée" en mode non authentifié, elle correspond au chemin ``/`` de l'URL publique
- ``RENATER_SP_HTTPD_PROTECTED_PROXY_TO`` : l'adresse du serveur (interne) de votre application qui sera "reverse proxifiée" en mode authentifié, elle correspond au chemin ``/my-protected-url/`` de l'URL publique (à adapter avec la valeur passée dans ``RENATER_SP_HTTPD_PROTECTED_PATH``). Dans la pluspart des cas, la valeur de ``RENATER_SP_HTTPD_PROTECTED_PROXY_TO`` vaut la même chose que ``RENATER_SP_HTTPD_PUBLIC_PROXY_TO`` car l'application à authentifier est la même que celle qui possède des pages publique.

Pour les passer en argument à votre conteneur docker, vous pouvez les positionner dans un fichier ``.env`` à coté de votre ``docker-compose.yml`` en prenant exemple sur [``.env-dist``](https://github.com/abes-esr/docker-shibboleth-renater-sp/blob/main/.env-dist) qui propose des exemples de valeurs. Votre ``docker-compose.yml`` doit alors transmettre ces variables au conteneur en les précisant dans la section [``environment`` comme dans cet exemple](https://github.com/abes-esr/docker-shibboleth-renater-sp/blob/0fdb9619c4e4b8bb2f50dfda1f93c4a1d65df4bb/docker-compose.yml#L13-L23).


### Configuration en TEST

1) Vous avez uniquement besoin de personnaliser les valeurs du ``.env`` (cf section ci-dessus) en précisant ``RENATER_SP_TEST_OR_PROD="TEST"``, puis vous pouvez lancer votre conteneur puis consulter l'URL https://votre-ip/Shibboleth.sso/Metadata pour récupérer les métadonnées attendue dans l'étape 2 par le guichet RENATER.

2) Vous devez ensuite enregistrer votre fournisseur de service dans la [fédération d'identités Education-Recherche de test](https://federation.renater.fr/registry?action=get_all). Remarque : pour le certificat (couple clé privé/publique), vous pouvez utiliser [celui qui est embarqué dans l'image docker](https://github.com/abes-esr/docker-shibboleth-renater-sp/tree/main/image/shibboleth/ssl).

3) Vous pouvez alors tester votre fournisseur de service en naviguant sur l'URL suivante : https://votre-ip/my-protected-url/ (adapter en fonction de la valeur de ``RENATER_SP_HTTPD_PROTECTED_PATH``)

### Configuration en PROD

1) Vous avez besoin de personnaliser les valeurs du ``.env`` (cf section ci-dessus) en précisant ``RENATER_SP_TEST_OR_PROD="PROD"``

2) Vous avez besoin de générer un couple de clé SSH privée/publique dédié pour le démon shibboleth qui tournera dans le conteneur. Ces clés sont ici dans l'image : ``/etc/shibboleth/ssl/server.key`` et ``/etc/shibboleth/ssl/server.crt``. Remarque : la clé ``server.key`` (privée) is critique et ne doit jamais être partagée mais pour des raison de démonstration, un couple de clé ``server.key`` et ``server.crt`` est inclu dans cette image ce qui permet de tester facilement cette image sans avoir à en générer une soit même. En revanche, cette clé ne doit **jamais être utilisée en production** car elle est [accessible publiquement dans le code source ici](https://github.com/abes-esr/docker-shibboleth-renater-sp/blob/main/image/shibboleth/ssl/server.key). Pour votre passage en production, nous aurez besoin de générer un couple de clé vous même. Voici les lignes de commandes permettant de générer un couple de clé auto-signées avec un long délai d'expiration (7300 jours = 20 ans!) comme recommandé par [RENATER](https://services.renater.fr/federation/documentation/generale/certificats-saml#recommandations_techniques_pour_les_certificats) :
   ```
   cd docker-shibboleth-sp/volume/shibboleth/ssl/
   openssl genrsa -out server.key 2048
   openssl req -new -key server.key -out server.csr
   openssl x509 -req -days 7300 -in server.csr -signkey server.key -out server.crt
   ```

3) Vous devez ensuite lancer votre conteneur puis consulter l'URL https://votre-ip/Shibboleth.sso/Metadata pour récupérer les métadonnées attendue dans l'étape suivante par le guichet RENATER.

4) Vous devez ensuite enregistrer votre fournisseur de service dans la [fédération d'identités Education-Recherche de production](https://federation.renater.fr/registry?action=get_all)

5) Vous pouvez tester votre fournisseur de service en naviguant sur l'URL suivante : https://votre-ip/my-protected-url/


### Démarrer l'application

Pour démarrer l'application depuis le [docker-compose exemple](https://github.com/abes-esr/docker-shibboleth-renater-sp/blob/main/docker-compose.yml) :
```bash
cd docker-shibboleth-renater-sp/
cp .env-dist .env
# personnalisez les valeurs de .env

docker-compose up
```

Vous pouvez alors vous baser sur le [docker-compose.yml exemple](https://github.com/abes-esr/docker-shibboleth-renater-sp/blob/main/docker-compose.yml) pour l'intégrer dans votre application. A noter que la dernière version disponible de l'image est la suivante :
```bash
docker pull abesesr/docker-shibboleth-renater-sp:1.0.2
```

## Développements

### Comment construire cette image docker ?

Cette image est construite et publiée automatiquement sur dockerhub à l'aide de github action mais il est également possible de la construire localement avec cette commande (utile pour le développement de cette image) :
```bash
cd docker-shibboleth-sp/
docker-compose build
```

### Comment publier une nouvelle version de cette image ?

Il est nécessaire d'utiliser [l'action github create-release.yml](https://github.com/abes-esr/docker-shibboleth-renater-sp/actions/workflows/create-release.yml), merci de se référer à cette procédure : https://github.com/abes-esr/abes-politique-developpement/blob/main/01-Gestion%20du%20code%20source.md#publier-une-nouvelle-release-dune-application

## Voir aussi

- Image dérivée de https://github.com/BibCnrs/docker-shibboleth-sp avec un exemple de configuration ici https://github.com/BibCnrs/BibRP/
- Cette image s'inspire aussi de :
  - https://github.com/mconf/apache-shib-docker/
  - https://gitlab.oit.duke.edu/devil-ops/duke-shibboleth-container


