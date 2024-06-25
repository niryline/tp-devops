Line RABEHANTA  
21960168

# TP DEVOPS
Structure du dossier de travail : 
TP/
 |_ postgres/
 |_ api/
 |_ http-server/
 

## TP 1 

### BD
1-1.  L'utilisation de l'option `-e` permet d'éviter de garder en clair dans un 
fichier texte nos identifiants. A la fin, j'ai le Dockerfile suivant dans 
`./postgres` :

```Dockerfile 
FROM postgres:14.1-alpine

COPY ./01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY ./02-InsertData.sql /docker-entrypoint-initdb.d
```

Et je construis et lance le conteneur postgres avec la commande suivante :

```
docker run -p 5433:5432 -d -h 127.0.0.1 --name postgres-server -e 
POSTGRES_PASSWORD -e POSTGRES_USR -e POSTGRES_DB --network app-network 
-v ./postgres/data:/var/lib/postgresql/data postgres-server
```

où les trois variables `POSTGRES_*` ont été exportées préalablement dans le 
shell.

Pour me connecter à la base de données, je suis directement passée par la 
ligne de commandes avec `psql -h localhost -p 5433 -U usr -d db`

### Backend
1-2. Le Dockerfile dans `./api` :

```Dockerfile
# 1er stage : basé sur une image de la version 3.8.6 de maven avec Java 17
# nom du stage = myapp-build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build 
# définit la variable d'environnement MYAPP_HOME et sa valeur
ENV MYAPP_HOME /opt/myapp
# se place dans le répertoire /opt/myapp
WORKDIR $MYAPP_HOME
# y copie le fichier des dépendances pom.xml de l'appli et son code source
COPY simpleapi/pom.xml .
COPY simpleapi/src ./src
# compile sans exécuter les tests 
RUN mvn package -DskipTests

# 2e stage : basé sur une image qui a java 17
FROM amazoncorretto:17
# définit la variable d'environnement MYAPP_HOME et sa valeur
ENV MYAPP_HOME /opt/myapp
# se place dans le répertoire /opt/myapp
WORKDIR $MYAPP_HOME
# récupère l'exécutable .jar obtenu dans le stage d'avant
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

# exécute le .jar ai démarrage du conteneur
ENTRYPOINT java -jar myapp.jar
```

Un build "multistage" (où chaque `FROM` marque le début d'un "stage") permet 
d'isoler chaque étape (build, exécution) du lancement d'une application. Grâce à
ça, on peut notamment optimiser la taille de nos images pour n'y garder que le 
strict minimum (par exemple ne pas prendre de quoi compiler dans la phase 
d'exécution).

Le conteneur est lancé avec la commande :
`docker run --name bck -p 8888:8080 -h localhost --network app-network backend`

Pour permettre la connexion à la base de données, en plus de l'utilisation du 
réseau `app-network`, je complète le fichier `application.yaml` avec :

```yaml
datasource:
    url: jdbc:postgresql://postgres-server:5432/db
    username: usr
    password: pwd
```

### Serveur http
Le Dockerfile dans `./http-server` :

```Dockerfile
FROM httpd:2.4
COPY index.html /usr/local/apache2/htdocs/
COPY httpd.conf /usr/local/apache2/conf/
```

Le fichier de configuration d'httpd est copié depuis la machine host avec :
`docker cp httpd.conf apache:/usr/local/apache2/conf/`

La configuration du reverse proxy qui permettra de rediriger les requêtes reçues 
par le serveur apache au backend :
```
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://bck:8080/
ProxyPassReverse / http://bck:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

Le conteneur est construit et lancé avec la commande :
`docker run -dit --name apache -p 9999:80 --network app-network apache`

### Compose 
1-3. et 1-4. Dans le répertoire `TP/`, en reprenant toutes les options utilisées
jusqu'à présent (identifiants postgres dans un fichier `.env` situé au même 
niveau que le fichier compose, volumes) :

```yaml
# version du format du fichier
version: '3.7'

# définition des 3 services de notre application, appartenant au même réseau
# app network
# chaque service est construit avec le dockerfile situé dans le chemin après 
# "build:"
services:
  bck: 
    build: ./api 
    networks:
      - app-network
    depends_on:
      - postgres-server

  postgres-server: 
    build: ./postgres
    networks:
      - app-network
		# utilisation d'un volume pour la persistance des données
    volumes:
      - ./postgres/data:/var/lib/postgresql/data
		# usage de variables d'environnement pour les données sensibles
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USR}
      POSTGRES_PASSWORD: ${POSTGRES_PWD}

  apache: 
    build: ./http-server
		# seul le proxy est exposé (et disponible depuis ma machine sur le port 9999)
    ports:
      - "9999:80"
    networks:
      - app-network
    depends_on:
      - bck

networks:
  app-network:
    name: app-network
		# permet de dire que le réseau a été créé en dehors du contexte du 
		# fichier compose (avant qu'on a lancé docker compose up)
    external: true
```

La construction et le lancement des conteneurs est lancée avec 
`docker compose up`.

`docker compose down` permet d'arrêter et supprimer les conteneurs et images qui
ont été créés précédemment.

`docker compose logs` permet d'obtenir les logs des services.

`docker compose` est important car il permet la mise en place et configuration 
de tous les conteneurs nécessaires à notre application en un seul fichier. On 
n'a plus à lancer toutes les commandes une à une et assure la reproductibilité 
de tous nos builds.  

Enfin, la publication des images est utile pour le versionnement de nos images, 
toujours dans une volonté de reproductibilité et d'historique de nos builds.

1.5 - 
`docker tag postgres niryantso/database:1.0` : renomme mon image locale 
`postgres` avec mon nom d'utilisateur DockerHub et versionne l'image 
(obligatoire pour la publication).

`docker push niryantso/database:1.0`: publie mon image sur DockerHub et la rend 
téléchargable.


## TP2
Le dossier de travail est le même.

2.1 - `testcontainers` est une bibliothèque Java qui permet de faire des tests 
unitaires ou d'intégration en utilisant des instances conteneurisées de bases de données, ou encore de navigateurs web, etc. En gros, elle permet de 
conteneuriser les tests (tout comme on a conteneurisé les phases de build et 
d'exécution dans le TP1). Une de leurs particularités c'est qu'ils sont 
automatiquement "nettoyés" après les tests.

2.2 - Mes configurations séparées (build+test/déploiement) de Github Actions :
`build.yml` (contient la configuration Sonar)

```yaml
# nom du workflow
name: CI devops 2024
on:
  # la pipeline est déclenchée au moment où on push sur main et develop
	# et à la création de PR
  push:
    branches: 
      - main
      - develop
  pull_request:

jobs:
  test-backend: 
	  # tourne sur une vm ubuntu 22.04
    runs-on: ubuntu-22.04
    steps:
		  # checkout le code du repository
      - uses: actions/checkout@v2.5.0

      - name: Set up JDK 17
			  # met en place java 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
					# active le système de cache des dépendances maven
          cache: maven 

      - name: Build and test with Maven
        env:
				  # credentials pour se connecter 
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
			  # lance le build et les tests maven + l'analyse Sonar
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=niryline_tp-devops --file ./api/simpleapi/pom.xml
```

`main.yml` (lancé seulement si on push sur main et seulement si la pipeline de 
build a terminé)
```yaml
name: Deploy
on:
  # le workflow s'exécute seulement quand le workflow CI devops 2024 a fini sur 
	# la branche main
  workflow_run:
    workflows: ["CI devops 2024"]
    branches: [main]
    types:
      - completed

jobs:
  build-and-push-docker-image:
    runs-on: ubuntu-22.04

    steps:
      - name: Login to DockerHub
			  # se connecte à dockerhub en utilisant les secrets qu'on a mis sur 
				# github
        run: docker login -u ${{ secrets.DOCKERHUB_USR }} -p ${{ secrets.DOCKERHUB_PWD }}

      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Build image and push backend
			  # action qui permet de build et publier les images docker, même principe
				# que le fichier compose en ce qui concerne la construction depuis les 
				# dockerfile
        uses: docker/build-push-action@v3
        with:
          # chemin relatif par rapport à la racine du projet git
          context: ./api/
					# tag comme dans la commande docker tag
          tags:  ${{secrets.DOCKERHUB_USR}}/tp1-bck:latest
          # l'image est publiée seulement si on est sur la branche main
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./postgres
          tags: ${{secrets.DOCKERHUB_USR}}/tp1-postgres-server:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./http-server
          tags: ${{secrets.DOCKERHUB_USR}}/tp1-apache:latest
          push: ${{ github.ref == 'refs/heads/main' }}
```

## TP3
Le dossier de travail est le même, avec le répertoire `ansible/` en plus à la 
racine.

3.1 - Inventory `ansible/inventories/setup.yml` :

```yaml
# groupe 
all:
 # variables globales 
 vars:
   # utilisateur ssh
   ansible_user: centos
	# clé privée ssh
   ansible_ssh_private_key_file: ~/.ssh/id_rsa_devops
 # sous-groupe
 children:
   # sous-sous-groupe
   prod:
	   # liste des hôtes du groupe
     hosts: line.rabehanta.takima.cloud
```

Pour ping les hôtes : `ansible -i ansible/inventories/setup.yml all -m ping`

3.2 - `ansible/playbook.yml`

```yaml
# pour tous les hôtes du groupe 'all' de l'inventory
- hosts: all
  # pas de collecte des informations sur les hôtes
  gather_facts: false
	# permet aux tâches d'être exécutée par le superutilisateur
  become: true
	# liste des ensembles de tâches qui seront exécutées sur l'hôte
  # (1 rôle = 1 ensemble de tâches liées)
	roles:
    - docker
    - network
    - database
    - app
    - proxy
    - front

  # petit test de connexion à l'hôte 
  tasks:
    - name: Test connection
      ping:
```

Pour lancer le playbook en utilisant le fichier d'hôtes précédent : 
`ansible-playbook -i inventories/setup.yml playbook.yml`

Les fichiers de tâche `main.yml` ressemblent à 
```yaml
---
# tasks file for roles/front
- name: Run Front
  # nom du module ansible
  docker_container:
	  # nom du conteneur
    name: tp1-front
    image: niryantso/tp1-front:5.0
		# s'assure que le conteneur est lancé
    state: started
		# réseau docker
    networks:
      - name: "app-network"
```

Pour le proxy, j'ai rajouté de quoi mapper le port 80 pour y avoir accès : 
```yaml
ports:
      - "80:80"
```

Pour la configuration du reverse proxy afin de connecter le back et le front : 
1. J'ai utilisé nginx déjà partiellement configuré par le code du front pour 
faire un premier proxy qui redirige les requêtes du front vers le proxy apache 

fichier de config de nginx (`default.conf`) : 
```conf
upstream proxy {
	server tp1-apache:80;
}

server {
  listen       80;
  server_name  line.rabehanta.takima.cloud;

  server_tokens off;

  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
    try_files $uri /$uri /index.html;
  }

	# renvoyer les requetes vers notre proxy
	location /api/ {
		proxy_pass http://proxy/api/;
	}
}
```

L'url utilisée par le front pour faire ses appels d'API : 
`VUE_APP_API_URL=line.rabehanta.takima.cloud/api`

2. J'ai configuré le reverse proxy apache pour recevoir les requêtes du front 
puis du back 

```conf
<VirtualHost *:80>
ProxyPreserveHost On

# rediriger les requêtes de / vers notre front
ProxyPass / http://tp1-front:80/
ProxyPassReverse / http://tp1-front:80/

# autoriser les requêtes CORS
Header add Access-Control-Allow-Origin "*"

# activation du module mod_rewrite
RewriteEngine On

# réécrire les requêtes de /api en supprimant "api/" et les rediriger 
# vers notre back
RewriteRule ^/api/(.*)$ http://tp1-bck:8080/$1 [P,L]
ProxyPassReverse / http://tp1-bck:8080/

</VirtualHost>
```

Remarque : j'ai pris pas mal de temps pour réussir à faire les bonnes 
redirections, et je me demande si en temps normal on utiliserait 2 reverse proxy
comme ici (d'après le schéma du TP, non, mais en même temps l'énoncé disait 
quand même de changer la configuration du proxy apache, et je ne voyais pas 
comment faire pour le front à part en y mettant aussi un proxy).