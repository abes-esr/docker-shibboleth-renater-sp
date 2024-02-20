# docker-shibboleth-renater-sp

[![Docker Pulls](https://img.shields.io/docker/pulls/abesesr/docker-shibboleth-renater-sp.svg)](https://hub.docker.com/r/abesesr/docker-shibboleth-renater-sp/) [![build-test-pubtodockerhub](https://github.com/abes-esr/docker-shibboleth-renater-sp/actions/workflows/build-test-pubtodockerhub.yml/badge.svg)](https://github.com/abes-esr/docker-shibboleth-renater-sp/actions/workflows/build-test-pubtodockerhub.yml)

Ce dépôt propose une image docker 🐳 générique permettant de mettre en place un fournisseur de service pré-connecté sur la [Fédération d'identités Education-Recherche (FER)](https://services.renater.fr/federation/index).

<img src="https://docs.google.com/drawings/d/e/2PACX-1vSTwzl0nQDoAVwzCUnAC9I5icDhcA2_YlHn6Glx1GFdiZSRFGdA9EPbeR50kMP6njUfsDeRut_L9aXJ/pub?w=301&amp;h=216">

Fonctionnement : ce fournisseur de service s'intégre sur une application tiers en se positionnant en amont des flux HTTP et en jouant le rôle de reverse proxy authentifiant. C'est à dire qu'une requête HTTP permettant d'accéder à l'application va tout d'abord transiter par cette brique pour ensuite être transmise à l'application cible. L'avantage de cette approche c'est de pouvoir intégrer une authentification SAML sur une application sans avoir à la modifier en profondeur, c'est ce reverse proxy qui se charge de transmettre une preuve de l'authentification à l'application. Cette preuve contient l'identifiant de l'utilisateur ainsi que les attributs fournits par le fournisseur d'identités (ex: mail, nom, prénom, supannEtablissement, etc ...). A noter qu'il existe des avantages mais aussi des inconvéniants à utiliser ce type d'architecture, vous pouvez consulter [la documentation qui explique tout cela plus en détail](https://services.renater.fr/federation/documentation/generale/shib-et-reverseproxy).

Technologies : cette image docker utilise un serveur apache (basée sur son image docker officielle) et mod_shib (module Shibboleth s'intégrant dans apache). Ce sont les briques techniques préconisées par RENATER pour implémenter un fournisseur de service.

<img src="https://docs.google.com/drawings/d/e/2PACX-1vRMfhC8StjZi8KUXUnAoQA7MJG4BymqQue0dIxnsR9-VchR9dJTOh3tRU8j_3ngpvPaU9rReELFVrG8/pub?w=945&amp;h=665">

(le [lien](https://docs.google.com/drawings/d/1lGluW3Kpq7p2j2Hx8SmAqkDAKFmw6Zu7P8YuhTPV0ew/edit) pour modifier le schéma)



## Comment utiliser cette image ?

### Configuration

Pour configurer le conteneur, vous devez lui passer des variables d'environnement, pour cela vous pouvez créer un fichier ``.env`` à coté de votre ``docker-compose.yml`` en prenant exemple sur les variables d'environnement du [``docker-compose.yml``](./docker-compose.yml) qui propose des exemples de valeurs en expliquant leur signification.

Si vous souhaitez injecter des configurations apache spécifiques dans la configuration du serveur apache, vous pouvez ajouter des fichiers de configuration via des volumes aux endroits suivants dans le conteneur :
- ``/usr/local/apache2/conf/extra/httpd-vhosts-begin.inc.conf`` et ``/usr/local/apache2/conf/extra/httpd-vhosts-end.inc.conf`` : pour injecter de la configuration au niveau global du virtualhost ([ici](./image/httpd-vhosts.conf#L53-L53)) et ([ici](./image/httpd-vhosts.conf#L109-L110)) 
- ``/usr/local/apache2/conf/extra/httpd-vhosts.public_proxy.inc.conf`` : pour injecter de la configuration au niveau du ProxyPass des URL publiques ([ici exactement](./image/httpd-vhosts.conf#L59))
- ``/usr/local/apache2/conf/extra/httpd-vhosts.protected_proxy.inc.conf`` : pour injecter de la configuration au niveau du ProxyPass des URL protégées ([ici exactement](./image/httpd-vhosts.conf#L91))


### Configuration en TEST

1) Vous avez uniquement besoin de personnaliser les valeurs du ``.env`` (cf section ci-dessus) en précisant ``RENATER_SP_TEST_OR_PROD="TEST"``, puis vous pouvez lancer votre conteneur puis consulter l'URL https://votre-ip/Shibboleth.sso/Metadata pour récupérer les métadonnées attendue dans l'étape 2 par le guichet RENATER.

2) Vous devez ensuite enregistrer votre fournisseur de service dans la [fédération d'identités Education-Recherche de test](https://federation.renater.fr/registry?action=get_all). Remarque : pour le certificat (couple clé privé/publique), vous pouvez utiliser [celui qui est embarqué dans l'image docker](https://github.com/abes-esr/docker-shibboleth-renater-sp/tree/main/image/shibboleth/ssl).

3) Vous pouvez alors tester votre fournisseur de service en naviguant sur l'URL suivante : https://votre-ip/my-protected-url/ (adapter en fonction de la valeur de ``RENATER_SP_HTTPD_PROTECTED_PATH``)

### Configuration en PROD

1) Vous avez besoin de personnaliser les valeurs du ``.env`` (cf section ci-dessus) en précisant ``RENATER_SP_TEST_OR_PROD="PROD"``

2) Vous avez besoin de générer un couple de clé SSH privée/publique dédié pour le démon shibboleth qui tournera dans le conteneur. Ces clés sont ici dans l'image : ``/etc/shibboleth/ssl/server-demo.key`` et ``/etc/shibboleth/ssl/server-demo.crt``. Remarque : la clé ``xxxxxx.key`` (privée) est critique et ne doit jamais être partagée mais pour des raison de démonstration, un couple de clé ``server-demo.key`` et ``server-demo.crt`` est inclu dans cette image ce qui permet de tester facilement cette image sans avoir à en générer une soit même. En revanche, cette clé ne doit **jamais être utilisée en production** car elle est [accessible publiquement dans le code source ici](https://github.com/abes-esr/docker-shibboleth-renater-sp/blob/main/image/shibboleth/ssl/server-demo.key). Pour votre passage en production, nous aurez besoin de générer un couple de clé vous même. Voici les lignes de commandes permettant de générer un couple de clé auto-signées avec un long délai d'expiration (7300 jours = 20 ans!) comme recommandé par [RENATER](https://services.renater.fr/federation/documentation/generale/certificats-saml#recommandations_techniques_pour_les_certificats) :
   ```
   cd docker-shibboleth-sp/volume/shibboleth/ssl/
   openssl genrsa -out server-prod.key 2048
   openssl req -new -key server-prod.key -out server-prod.csr
   openssl x509 -req -days 7300 -in server-prod.csr -signkey server-prod.key -out server-prod.crt
   ```
3) Vous devez ensuite positionner les variables suivantes dans le fichier ``.env`` :
   ```
   RENATER_SP_CERTIFICATE_CRT=ssl/server-prod.crt
   RENATER_SP_CERTIFICATE_KEY=ssl/server-prod.key
   ```

4) Vous devez ensuite lancer votre conteneur puis consulter l'URL https://votre-ip/Shibboleth.sso/Metadata pour récupérer les métadonnées attendue dans l'étape suivante par le guichet RENATER.

5) Vous devez ensuite enregistrer votre fournisseur de service dans la [fédération d'identités Education-Recherche de production](https://federation.renater.fr/registry?action=get_all)

6) Vous pouvez tester votre fournisseur de service en naviguant sur l'URL suivante : https://votre-ip/my-protected-url/


### Démarrer l'application

Vous pouvez vous baser sur le [docker-compose.yml exemple](https://github.com/abes-esr/docker-shibboleth-renater-sp/blob/main/docker-compose.yml) pour l'intégrer dans votre application comme un reverse proxy applicatif. A noter que la dernière version disponible de l'image est la suivante :
```bash
docker pull abesesr/docker-shibboleth-renater-sp:1.7.0
```

## Développements

### Comment construire cette image docker ?

Cette image est construite et publiée automatiquement sur dockerhub à l'aide de github action mais il est également possible de la construire localement avec cette commande (utile pour le développement de cette image) :
```bash
cd docker-shibboleth-sp/
docker-compose build
```

### Démarrer l'application pour le développement

Pour démarrer l'application depuis le [docker-compose exemple](https://github.com/abes-esr/docker-shibboleth-renater-sp/blob/main/docker-compose.yml) :
```bash
cd docker-shibboleth-renater-sp/
cp .env-dist .env
# personnalisez les valeurs de .env

docker-compose up
```

Puis se rendre sur (en ignorant les certificat invalides) :
- https://127.0.0.1/ pour une URL non protégée
- https://127.0.0.1/my-protected-url/ pour une URL non protégée par fédération d'identiés


### Comment publier une nouvelle version de cette image ?

Il est nécessaire d'utiliser [l'action github create-release.yml](https://github.com/abes-esr/docker-shibboleth-renater-sp/actions/workflows/create-release.yml), merci de se référer à cette procédure : https://github.com/abes-esr/abes-politique-developpement/blob/main/01-Gestion%20du%20code%20source.md#publier-une-nouvelle-release-dune-application

## Voir aussi

- Image dérivée de https://github.com/BibCnrs/docker-shibboleth-sp avec un exemple de configuration ici https://github.com/BibCnrs/BibRP/
- Exemple d'utilisation ici : https://github.com/abes-esr/theses-docker
- Cette image s'inspire aussi de :
  - https://github.com/mconf/apache-shib-docker/
  - https://gitlab.oit.duke.edu/devil-ops/duke-shibboleth-container


