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
1. L'utilisation de l'option `-e` permet d'éviter de garder en clair dans un 
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

2. Pour me connecter à la base de données, je suis directement passée par la 
ligne de commandes avec `psql -h localhost -p 5433 -U usr -d db`

### Backend
Le Dockerfile dans `./api` :

```Dockerfile
FROM eclipse-temurin:17-jdk-alpine
FROM eclipse-temurin:17-jre-alpine
# Copy resource from previous stage
COPY --from=0 Main.class .

# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY simpleapi/pom.xml .
COPY simpleapi/src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```

Le conteneur est construit et lancé avec la commande :
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
Dans le répertoire `TP/`, en reprenant toutes les options utilisées jusqu'à 
présent (identifiants postgres dans un fichier `.env`, volumes) :
```yaml
version: '3.7'

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
    volumes:
      - ./postgres/data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USR}
      POSTGRES_PASSWORD: ${POSTGRES_PWD}

  apache: 
    build: ./http-server
    ports:
      - "9999:80"
    networks:
      - app-network
    depends_on:
      - bck

networks:
  app-network:
    name: app-network
    external: true
```

La construction des conteneurs est lancée avec `docker compose up`.

`docker compose` est important car il permet la mise en place et configuration 
de tous les conteneurs nécessaires à notre application en un seul fichier. On 
n'a plus à lancer toutes les commandes une à une et assure la reproductibilité 
de tous nos builds.  

Enfin, la publication des images est utile pour le versionnement de nos images, 
toujours dans une volonté de reproductibilité et d'historique de nos builds.


## TP2
Le dossier de travail est le même.

`testcontainers` est une bibliothèque Java qui permet de faire des tests 
unitaires en utilisant des instances conteneurisées de bases de données, ou 
encore de navigateurs web.

Mes configurations séparées (build+test/déploiement) de Github Actions :
`build.yml` (contient la configuration Sonar)
```yaml
name: CI devops 2024
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: 
      - main
      - develop
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven 

     #finally build your app with the latest command
      - name: Build and test with Maven
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Needed to get PR information, if any
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=niryline_tp-devops --file ./api/simpleapi/pom.xml
```

`main.yml` (lancé seulement si on push sur main et seulement si la pipeline de 
build a terminé)
```yaml
name: Deploy
on:
  workflow_run:
    workflows: ["CI devops 2024"]
    branches: [main]
    types:
      - completed

jobs:
  # define job to build and publish docker image
  build-and-push-docker-image:
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04

    # steps to perform in job
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USR }} -p ${{ secrets.DOCKERHUB_PWD }}

      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./api/
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKERHUB_USR}}/tp1-bck:latest
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./postgres
          tags: ${{secrets.DOCKERHUB_USR}}/tp1-postgres-server:latest
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./http-server
          tags: ${{secrets.DOCKERHUB_USR}}/tp1-apache:latest
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}
```