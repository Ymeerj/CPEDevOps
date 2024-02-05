# DevOps - TP 01 - Docker

## Création du docker de base de données

## Docker de bd:

On commence par créer le Dockerfile utilisé pour la création de l’image

```docker
# Image utilisée
FROM postgres:14.1-alpine

# Définition de la base de données de pase et logins
ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

# On copie les scripts SQL pour qu'ils soient lancés à l'initialisation de la BD
COPY ./sql_scripts /docker-entrypoint-initdb.d
```

Les commandes executées sont les suivantes :

```bash
# On créer l'image jeremy/bdops depuis le fichier Dockerfile du répertoire courrant
docker build -t jeremy/bdops .

# On créé le réseau qui va être utilisé lors de ce TP
docker network create ops-network

# On met l'image dans un conteneur avec les tags suivants:
# --network pour définir le réseau partagé utilisé
# -p pour exposer le port du conteneur a notre localhost
# --name pour définir le nom du conteneur
# -v pour définir le volume utilisé pour la persistence des données
docker run --network "ops-network" -p "5432:5432" \
--name bdops -v $PWD/data:/var/lib/postgresql/data jeremy/bdops
```

Pour faciliter l’exploration de la BD j’ai préféré utilisé DBeaver plutôt que Adminer. L’utilisation de celui-ci aurait été interessante pour la compréhension des réseaux Docker mais nous y reviendrons plus tard.

*Why should we run the container with a flag `-e` to give the environment variables?*

Nous devrions exécuter le conteneur avec l'option `-e` pour fournir des variables d'environnement afin de ne pas enregistrer dans le Dockerfile les logins utilisés. Cela permet de garantir une meilleure sécurité.

*Why do we need a volume to be attached to our postgres container?*

Comme expliquer dans les commentaires des commandes, utilisé un volume permet de garantir la persistence des données dans le conteneur.

## Backend API

1-2 Why do we need a multistage build? And explain each step of this dockerfile.

Un build multistage permet de bénéficier de l’utilisation de plusieurs images sans allourdir l’image finale.

```bash
# Build
## On commence par récupérer l'image de base qui contient le JDK
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
## ON définit la variable d'environnement MYAPP_HOME avec la valeur /opt/myapp
ENV MYAPP_HOME /opt/myapp
## On se "déplace" dans le répertoire définit la ligne du dessus
WORKDIR $MYAPP_HOME
## On copie le pom.xml qui contient les dépendances de l'application
COPY pom.xml .
## On copie le répertoire src de notre machine au conteneur. 
## Celui-ci contient les sources qui doivent être compilées
COPY src ./src
## On utilise maven pour compiler le code passé
RUN mvn package -DskipTests

# Run
## On récupère l'image qui contient le jre
FROM amazoncorretto:17
## ON définit la variable d'environnement MYAPP_HOME avec la valeur /opt/myapp
ENV MYAPP_HOME /opt/myapp
## On se "déplace" dans le répertoire définit la ligne du dessus
WORKDIR $MYAPP_HOME
## On copie a l'aide de "from" le code compilé du premier conteneur au deuxième
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
## On execute la commande java -jar myapp.jar pour lancer notre backend
ENTRYPOINT java -jar myapp.jar
```

Le code source fournit est fonctionnel cependant nous devons modifier le fichier `simple-api/src/main/resources/application.yml` pour configurer l’accès a la base de données utilisé par l’API.

```yaml
datasource:
    url: jdbc:postgresql://database:5432/db
    username: usr
    password: pwd
    driver-class-name: org.postgresql.Driver
```

Ici l’url n’est pas [localhost](http://localhost) car la base de données et l’API sont dans le même réseau Docker mais ne sont pas “visible” sur l’host. On utilise donc le nom du conteneur comme URL.

## HTTP Server

La première approche avec le conteneur httpd est d’ajouter un fichier index.html au bon endroit pour avoir une première page statique visible.

Dockerfile utilisé:

```docker
FROM httpd

# On copie notre src contenant le index.html au répertoire d'Apache
COPY ./src /usr/local/apache2/htdocs
```

Le chemin `/usr/local/apache2/htdocs` correspond au DocumentRoot. C’est le répertoire utilisé par défaut lorqu’une requete est faite sur le port 80 au serveur Apache.

### Utilisation d’un reverse proxy

On utilise ici Apache pour faire une redirection de nos appels sur le port 80 vers l’api. Pour ajouter les lignes nécessaires au fichier de configuration on passe par une commande run.

```docker
FROM httpd

COPY ./src /usr/local/apache2/htdocs

RUN echo '<VirtualHost *:80> \n\
  ProxyPreserveHost On \n\
  ProxyPass / http://backend:8080/ \n\
  ProxyPassReverse / http://backend:8080/ \n\
  </VirtualHost> \n\
  LoadModule proxy_module modules/mod_proxy.so \n\
  LoadModule proxy_http_module modules/mod_proxy_http.so \n\
  ' >> /usr/local/apache2/conf/httpd.conf
```

## Docker-compose

### 1-3 Document docker-compose most important commands

Dans le cadre de ce TP j’ai pu utiliser plusieurs fois les commandes suivantes:

| Commande             | Fonction                                                                                           |
|----------------------|----------------------------------------------------------------------------------------------------|
| docker-compose build | Construit les images nécessaires définit dans le docker-compose.yml si aucun fichier n’est précisé |
| docker-compose up    | Démarre les services, souvent utilisé avec -d pour lancer les services en détaché                  |
| docker-compose down  | Éteint les services                                                                                |
| docker-compose ls    | Liste les services actifs                                                                          |
| docker-compose logs  | Permet de visualiser les logs d’une composition de conteneur                                       |

### 1-4 Document your docker-compose file.

```yaml
version: '3.7'

services:
	#Service de base de données
  database:
			# On définit le contexte du build.
			# ici le chemin vers le répertoire contenant le dockerfile
      build: 
        context: ./bd 
			# Réseaux docker utilisés, on en a un ici : ops-network
      networks:
        - ops-network
	#Service backend
  backend:
    build:
			# On définit le contexte du build.
			# ici le chemin vers le répertoire contenant le dockerfile
      context: ./java/simple-api-student-main
		# Réseaux docker utilisés, on en a un ici : ops-network
    networks:
      - ops-network
		# Dépendance du service, permet d'attendre que les services décris soient 
		# finis avant de build 
		# On attend que le backend soit prêt
    depends_on:
      - database
	#Service apache
  httpd:
		# On définit le contexte du build.
		# ici le chemin vers le répertoire contenant le dockerfile
    build:
      context: ./http
		# Port exposé, ici on expose uniquement le 80
    ports:
      - "80:80"
		# Réseaux docker utilisés, on en a un ici : ops-network
    networks:
      - ops-network
		# Dépendance du service, permet d'attendre que les services décris soient 
		# finis avant de build 
		# On attend que le backend soit prêt
    depends_on:
      - backend

# Déclaration des réseaux utilisés
networks:
	# On déclare ops-network 
  ops-network:
    external: true
```

## 1-5 Document your publication commands and published images in dockerhub.

Je me suis d’abord login avec `docker login`.

Une fois chose faites j’ai appliquer un tag sur l’image de ma base de données avec `docker tag jerem/bdops jeremydp/mybd:1.0`

Je l’ai ensuite publié avec `docker push jeremydp/mybd:1.0`

L’image est maintenant publique et peut être utilisé par n’importe qui