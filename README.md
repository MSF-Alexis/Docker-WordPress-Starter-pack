# Docker WordPress - Starter pack

Un environnement local pour développer avec **WordPress**, MariaDB et phpMyAdmin à l’aide de Docker Compose.

Ce starter pack sépare la configuration des identifiants sensibles, conserve les données MariaDB dans un volume et fournit une base simple pour le développement de sites, de thèmes et d’extensions WordPress.

> Ce projet est destiné au développement local. Il ne constitue pas une configuration prête pour la production.

## Sommaire

- [Services](#services)
- [Prérequis](#prérequis)
- [Structure du projet](#structure-du-projet)
- [Installation](#installation)
- [Gestion des secrets](#gestion-des-secrets)
- [Configuration WordPress](#configuration-wordpress)
- [Commandes utiles](#commandes-utiles)
- [Accès à MariaDB depuis un client SQL](#accès-à-mariadb-depuis-un-client-sql)
- [Données et sauvegardes](#données-et-sauvegardes)
- [Dépannage](#dépannage)
- [Sécurité et limites](#sécurité-et-limites)
- [Contribution](#contribution)

## Services

| Service Compose | Image | Rôle | Accès local |
| --- | --- | --- | --- |
| `wordpress-dev` | `wordpress:php8.5-apache` | WordPress avec PHP et Apache | http://localhost:8080 |
| `mariadb-dev` | `mariadb:latest` | Base de données | Réseau Docker uniquement |
| `phpmyadmin-dev` | `phpmyadmin:latest` | Administration de la base | http://localhost:8888 |

Les services communiquent sur le réseau Docker `backend`.

WordPress et phpMyAdmin attendent que MariaDB soit déclaré sain avant de démarrer.

Les ports web sont liés à `127.0.0.1` afin de limiter leur accès à la machine locale.

## Prérequis

- Docker Engine avec le plugin Docker Compose, ou Docker Desktop.
- Git pour récupérer le dépôt.
- Les ports locaux `8080` et `8888` disponibles.
- Un terminal ouvert à la racine du projet.

Les commandes de ce document utilisent `docker compose`, et non l’ancien exécutable `docker-compose`.

Vérifier l’installation :

```sh
docker --version
docker compose version
```

## Structure du projet

La configuration sensible est placée à côté du dépôt, et non dans son arborescence.

```text
workspace/
├── msf-local-config/
│   ├── .env
│   └── secrets/
│       ├── mariadb_root_password.txt
│       └── wordpress_db_password.txt
│
└── docker-wordpress-starter-pack/
    ├── compose.yaml
    ├── .env.example
    ├── .gitignore
    ├── README.md
    ├── docker/
    │   └── php/
    │       └── custom.ini
    └── ... fichiers WordPress
```

> Le Compose monte la racine du projet dans `/var/www/html`. Ne placez donc pas les secrets ou le fichier `.env` dans cette racine : ils se retrouveraient dans l’arborescence web du conteneur.

Les chemins des secrets dans `compose.yaml` doivent correspondre à cette organisation :

```yaml
secrets:
  mariadb_root_password:
    file: ../msf-local-config/secrets/mariadb_root_password.txt

  wordpress_db_password:
    file: ../msf-local-config/secrets/wordpress_db_password.txt
```

## Installation

### 1. Récupérer le projet

Clonez le dépôt depuis GitHub, puis ouvrez un terminal dans le dossier contenant `compose.yaml`.

Toutes les commandes suivantes sont à exécuter depuis ce dossier.

### 2. Préparer la configuration locale

Créez le dossier `../msf-local-config/`, puis son sous-dossier `secrets/`.

Créez le fichier `../msf-local-config/.env` avec le contenu suivant :

```dotenv
DB_NAME=masolutionformation
DB_USER=wp_user
WP_TABLE_PREFIX=wp_

WORDPRESS_PORT=8080
PHPMYADMIN_PORT=8888
```

| Variable | Description | Valeur proposée |
| --- | --- | --- |
| `DB_NAME` | Nom de la base WordPress | `masolutionformation` |
| `DB_USER` | Compte MariaDB utilisé par WordPress | `wp_user` |
| `WP_TABLE_PREFIX` | Préfixe des tables WordPress | `wp_` |
| `WORDPRESS_PORT` | Port HTTP local de WordPress | `8080` |
| `PHPMYADMIN_PORT` | Port HTTP local de phpMyAdmin | `8888` |

Le fichier `.env.example` du dépôt peut reprendre ces valeurs non sensibles pour documenter la configuration attendue.

### 3. Créer les secrets

Créez les fichiers suivants :

```text
../msf-local-config/secrets/mariadb_root_password.txt
../msf-local-config/secrets/wordpress_db_password.txt
```

Chaque fichier doit contenir uniquement son mot de passe :

- `mariadb_root_password.txt` : mot de passe du compte administrateur MariaDB.
- `wordpress_db_password.txt` : mot de passe du compte défini par `DB_USER`.

Utilisez deux mots de passe différents.

N’ajoutez ni nom de variable, ni guillemets autour des mots de passe. Enregistrez les fichiers en UTF-8 sans BOM.

> Pour une base déjà initialisée, utilisez d’abord ses identifiants actuels. Modifier les fichiers secrets ne modifie pas automatiquement les comptes MariaDB existants.

### 4. Vérifier la configuration PHP

Le fichier suivant doit exister avant le démarrage :

```text
docker/php/custom.ini
```

Il est monté en lecture seule dans le conteneur WordPress :

```yaml
volumes:
  - ./docker/php/custom.ini:/usr/local/etc/php/conf.d/custom.ini:ro
```

### 5. Valider la configuration Compose

```sh
docker compose --env-file ../msf-local-config/.env config --quiet
```

Si la commande ne retourne aucune erreur, démarrez les services :

```sh
docker compose --env-file ../msf-local-config/.env up -d
```

Vérifiez leur état :

```sh
docker compose --env-file ../msf-local-config/.env ps
```

### 6. Ouvrir les applications

- WordPress : http://localhost:8080
- phpMyAdmin : http://localhost:8888

Si WordPress n’est pas encore installé, terminez son installation depuis l’interface web.

Le compte administrateur WordPress créé à cette étape est distinct du compte MariaDB.

Pour vous connecter à phpMyAdmin :

```text
Utilisateur : valeur de DB_USER
Mot de passe : contenu de wordpress_db_password.txt
```

Réservez le compte MariaDB `root` aux opérations d’administration qui le nécessitent.

## Gestion des secrets

Les mots de passe ne sont pas définis directement dans `compose.yaml`.

Compose les monte dans les conteneurs autorisés sous forme de fichiers :

| Secret | Chemin dans le conteneur | Services autorisés |
| --- | --- | --- |
| `mariadb_root_password` | `/run/secrets/mariadb_root_password` | MariaDB |
| `wordpress_db_password` | `/run/secrets/wordpress_db_password` | MariaDB, WordPress |

Les images utilisent les variables suivantes pour lire ces fichiers :

```yaml
# MariaDB
MARIADB_ROOT_PASSWORD_FILE: /run/secrets/mariadb_root_password
MARIADB_PASSWORD_FILE: /run/secrets/wordpress_db_password

# WordPress
WORDPRESS_DB_PASSWORD_FILE: /run/secrets/wordpress_db_password
```

phpMyAdmin ne reçoit pas de secret : les identifiants sont saisis dans son formulaire de connexion.

### Exclusions Git

Conservez au minimum les règles suivantes dans `.gitignore` :

```gitignore
.env
.env.*
!.env.example

secrets/
backups/
```

> Les secrets Compose locaux ne sont pas un coffre-fort chiffré. Les fichiers sources restent présents sur la machine hôte et doivent être protégés par les permissions appropriées.

## Configuration WordPress

### Connexion à la base

WordPress utilise les paramètres suivants :

```yaml
WORDPRESS_DB_HOST: mariadb-dev:3306
WORDPRESS_DB_USER: ${DB_USER}
WORDPRESS_DB_PASSWORD_FILE: /run/secrets/wordpress_db_password
WORDPRESS_DB_NAME: ${DB_NAME}
WORDPRESS_TABLE_PREFIX: ${WP_TABLE_PREFIX:-wp_}
```

`mariadb-dev` est le nom du service sur le réseau Docker. N’utilisez pas `localhost` comme hôte de base de données dans le conteneur WordPress.

### Projet WordPress existant

Si le projet contient déjà un fichier `wp-config.php`, vérifiez qu’il prend en charge les variables d’environnement et la lecture du secret.

Une valeur codée en dur comme celle-ci ignore la configuration du secret :

```php
define('DB_PASSWORD', 'ancien-mot-de-passe');
```

Ne remplacez pas un `wp-config.php` existant sans sauvegarde : il peut contenir des paramètres spécifiques au projet.

### Montage des fichiers

Le montage suivant permet de travailler directement sur les fichiers depuis l’hôte :

```yaml
volumes:
  - .:/var/www/html
```

Les modifications apportées aux fichiers du projet sont utilisées par le conteneur WordPress.

Ce montage donne également au conteneur un accès en écriture au dossier du projet. Gardez les sauvegardes et fichiers confidentiels hors de cette arborescence.

## Commandes utiles

Utilisez le même fichier d’environnement pour toutes les commandes Compose afin de conserver une configuration cohérente.

### Démarrer les services

```sh
docker compose --env-file ../msf-local-config/.env up -d
```

### Arrêter les services sans les supprimer

```sh
docker compose --env-file ../msf-local-config/.env stop
```

### Arrêter et supprimer les conteneurs

```sh
docker compose --env-file ../msf-local-config/.env down
```

Le volume MariaDB est conservé tant que l’option `--volumes` n’est pas utilisée.

### Afficher l’état des services

```sh
docker compose --env-file ../msf-local-config/.env ps
```

### Suivre les logs

```sh
docker compose --env-file ../msf-local-config/.env logs -f
```

### Suivre uniquement les logs WordPress

```sh
docker compose --env-file ../msf-local-config/.env logs -f wordpress-dev
```

### Ouvrir un terminal dans WordPress

```sh
docker compose --env-file ../msf-local-config/.env exec wordpress-dev bash
```

### Vérifier la version de PHP

```sh
docker compose --env-file ../msf-local-config/.env exec wordpress-dev php -v
```

### Appliquer une modification du Compose ou de l’environnement

```sh
docker compose --env-file ../msf-local-config/.env up -d
```

Un simple `restart` ne suffit pas à appliquer toutes les modifications de configuration des conteneurs.

### Redémarrer WordPress après une modification de PHP

```sh
docker compose --env-file ../msf-local-config/.env restart wordpress-dev
```

### Télécharger les images puis recréer les services

```sh
docker compose --env-file ../msf-local-config/.env pull
docker compose --env-file ../msf-local-config/.env up -d
```

Sauvegardez la base avant une mise à jour de MariaDB et vérifiez la compatibilité des versions.

## Accès à MariaDB depuis un client SQL

Par défaut, MariaDB n’expose aucun port sur la machine hôte.

Pour utiliser un client SQL local, ajoutez ce bloc au service `mariadb-dev` :

```yaml
ports:
  - "127.0.0.1:3306:3306"
```

Appliquez la modification :

```sh
docker compose --env-file ../msf-local-config/.env up -d
```

Configurez ensuite votre client :

```text
Hôte : 127.0.0.1
Port : 3306
Base : valeur de DB_NAME
Utilisateur : valeur de DB_USER
Mot de passe : contenu de wordpress_db_password.txt
```

Si le port `3306` est déjà occupé, publiez par exemple `3307:3306`, puis utilisez le port `3307` dans votre client.

## Données et sauvegardes

### Persistance

MariaDB stocke ses données dans le volume nommé `mariadb-volume`, monté sur :

```text
/var/lib/mysql
```

Les données restent présentes lorsque les conteneurs sont arrêtés ou recréés.

### Sauvegarde

Avant une opération importante :

1. Exportez la base depuis phpMyAdmin.
2. Sauvegardez les fichiers WordPress, notamment `wp-content`.
3. Conservez une copie protégée de la configuration locale et des secrets.
4. Stockez ces sauvegardes hors du dossier servi par Apache et hors de Git.
5. Vérifiez qu’une restauration est possible.

### Réinitialisation complète

> Attention : la commande suivante supprime les volumes déclarés par Compose, dont celui contenant la base MariaDB. Cette opération détruit les données locales.

À exécuter uniquement après sauvegarde, pour repartir sur une base vide :

```sh
docker compose --env-file ../msf-local-config/.env down --volumes
docker compose --env-file ../msf-local-config/.env up -d
```

Les fichiers du projet montés depuis l’hôte ne sont pas supprimés par cette commande.

## Dépannage

### Un secret est introuvable

Vérifiez :

- L’existence du fichier sur l’hôte.
- Le chemin déclaré dans `compose.yaml`.
- Les permissions de lecture du fichier.
- L’organisation des dossiers décrite dans ce document.

Relancez ensuite la validation :

```sh
docker compose --env-file ../msf-local-config/.env config --quiet
```

### WordPress ne se connecte pas à MariaDB

Vérifiez :

- Que MariaDB est démarré et sain.
- Que `WORDPRESS_DB_HOST` vaut `mariadb-dev:3306`.
- Que le nom de base et le compte correspondent à la base existante.
- Que le mot de passe du secret correspond au compte MariaDB.
- Que `wp-config.php` utilise effectivement la configuration Docker.

Consultez les logs :

```sh
docker compose --env-file ../msf-local-config/.env logs mariadb-dev wordpress-dev
```

### Un nouveau mot de passe n’est pas pris en compte

Les variables d’initialisation MariaDB s’appliquent lors de la création initiale de la base.

Si le volume existe déjà, modifier `MARIADB_PASSWORD_FILE` ou son fichier ne change pas le mot de passe du compte enregistré.

Pour effectuer une rotation :

1. Sauvegardez la base.
2. Modifiez le mot de passe du compte dans MariaDB.
3. Mettez à jour le fichier secret correspondant.
4. Recréez le conteneur WordPress.
5. Vérifiez la connexion.

Ne supprimez pas le volume pour résoudre ce problème si vous souhaitez conserver vos données.

### MariaDB reste dans l’état `unhealthy`

Consultez d’abord ses logs :

```sh
docker compose --env-file ../msf-local-config/.env logs mariadb-dev
```

Le contrôle de santé utilise :

```text
healthcheck.sh --connect --innodb_initialized
```

Une première initialisation peut prendre du temps. Pour un volume provenant d’une ancienne installation, vérifiez également la compatibilité de l’image et de la configuration de contrôle de santé.

### Un port est déjà utilisé

Modifiez les ports dans le fichier `.env` :

```dotenv
WORDPRESS_PORT=8081
PHPMYADMIN_PORT=8990
```

Puis appliquez la configuration :

```sh
docker compose --env-file ../msf-local-config/.env up -d
```

Si WordPress était déjà installé, ses URL enregistrées peuvent aussi nécessiter une mise à jour.

## Sécurité et limites

- N’utilisez pas le compte MariaDB `root` pour WordPress.
- Ne versionnez jamais de mots de passe, d’exports SQL ou de secrets.
- Conservez les secrets hors de `/var/www/html`.
- Limitez les services publiés à l’interface locale.
- N’exposez pas phpMyAdmin publiquement.
- Protégez les sauvegardes : elles peuvent contenir des données personnelles.
- Pour un environnement reproductible, remplacez les tags évolutifs comme `latest` par des versions explicitement validées.
- Testez les mises à jour sur une copie de la base avant de les appliquer à des données importantes.

Un déploiement en production nécessite une configuration dédiée : HTTPS, gestion des accès, sauvegardes automatisées, supervision, politique de mise à jour et gestion adaptée des secrets.

## Contribution

Avant de proposer une modification :

1. Vérifiez que la configuration Compose est valide.
2. Testez le démarrage et la connexion de WordPress à MariaDB.
3. Vérifiez que les services restent accessibles uniquement comme prévu.
4. Mettez à jour ce README si la configuration change.
5. Assurez-vous qu’aucun secret ni aucune donnée locale n’est inclus dans le commit.