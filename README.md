Line RABEHANTA  
21960168

# TP DEVOPS
## TP 1 
1. L'utilisation de l'option `-e` permet d'éviter de garder en clair dans un 
fichier texte nos identifiants. A la fin, j'ai le Dockerfile suivant :

```Dockerfile 
FROM postgres:14.1-alpine

COPY ./01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY ./02-InsertData.sql /docker-entrypoint-initdb.d
```

Et je construis et lance le conteneur avec la commande

```
docker run -p 5433:5432 -d -h 127.0.0.1 --name postgres-server -e 
POSTGRES_PASSWORD -e POSTGRES_USR -e POSTGRES_DB postgres-server
```

