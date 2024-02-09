# TP de DevOps

Ce projet est le résultat de suivi du TP suivant : [text](http://school.pages.takima.io/devops-resources/)

Les points du TP sont les suivants:

- Utilisation de docker pour la conteneurisation de parties d'une application
- Mise en place de github actions pour l'automatisation de la publication des images docker ainsi que du déploiement de l'application
- Utilisation d'Ansible pour le déploiement de l'application


Le projet est divisé en cinq parties:

- database postgres
- API Java Spring
- Reverse Proxy avec load balancing HTTP Apache 
- Front en VueJS
- Ansible pour le déploiement


# Partie 1 - Docker

## Docker de base de données:

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

## Partie 2 - GitHub actions

### *What are testcontainers?*

Testcontainer est une librairie de test qui permet l’utilisation de containers docker pour la mise en place de tests.

### 2-2 Document your Github Actions configurations.

Après la troisième partie de TP le projet est constitué de trois actions:

- test-backend.yml - Test le backend
- build-and-push-docker-image.yml - Créé et publie les différentes images docker
- deploy-app.yml - Deploie l'application à l'aide d'Ansible 

```yml
#test-backend.yml
name: CI devops 2023 - test
on: #Cet action est la première a s'exécuter, elle s'execute lors de push sur les branches main et develop
  #to begin you want to launch this job in main and develop
  push:
    branches:
      - main
      - develop

jobs:
  test-backend:
    runs-on: ubuntu-22.04 #Environnement sur lequel l'action est exécuté
    steps:
      #checkout your github code using actions/checkout@v2.5.0. L'action permet de checkout notre répo pour que l'action puisse l'utiliser
      - uses: actions/checkout@v2.5.0 # Action utilisé

      # Etape d'initialisation de JDK 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3 #Action utilisé
        with: # Arguments passés à l'action
          distribution: "zulu" # Distribution Java utilisé
          java-version: "17" # Argument passé à l'action définissant la version de java

      # Etape de test de l'application avec sonar
      - name: Test build with sonar
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=Ymeerj_CPEDevOps -Dsonar.organization=ymeerj -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./java/simple-api-student-main/pom.xml
      # - name: Build and test with Maven
      #   run: mvn clean verify --file java/simple-api-student-main
```

```yml
#build-and-push-docker-image
name: CI devops 2023 - deploy
on:
  #Le workflow s'éxécute en cas de succès du workflow "CI devops 2023 - test"
  workflow_run:
    types:
      - completed
    workflows:
      - "CI devops 2023 - test"
    branches:
      - main

jobs:
  build-and-push-docker-image:
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04
    if: ${{ github.event.workflow_run.conclusion == 'success' }} #Ne s'exécute qu'en cas de succès du workflow précédent

    # steps to perform in job
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./java/simple-api-student-main
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/opsapi:latest
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./bd
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/mybd:latest
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./http
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/opshttp:latest
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push front
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./front
          # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/opsfront:latest
          # build on feature branches, push only on main branch
          push: ${{ github.ref == 'refs/heads/main' }}

```

```yml
#deploy-app.yml
name: CI devops 2023 - deploy website
on:
  # Ce workflow s'exécute lors d'une execution complète du workflow "CI devops 2023 - deploy"
  workflow_run:
    types:
      - completed
    workflows:
      - "CI devops 2023 - deploy"
    branches:
      - main

jobs:
  deploy-app:
    runs-on: ubuntu-22.04 #Environnement du workflow
    if: ${{ github.event.workflow_run.conclusion == 'success' }} # S'éxecute uniquement si le workflow "CI devops 2023 - deploy" s'est soldé par un succès

    steps:
      #Checkout du code pour le workflow
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      #Execution du playbook présent dans ./ansible pour le déploiement de l'application
      - name: run playbook
        uses: dawidd6/action-ansible-playbook@v2 # Action utilisé, permet l'éxecution de playbook yml
        with:
          playbook: ansible/playbook.yml # Chemin vers le playbook
          key: ${{secrets.ANSIBLE_TOKEN}} # Clé SSH utilisé
          vault_password: ${{secrets.ANSIBLE_VAULT_PASSWORD}} #Mot de passe du vault utilisé
          #options : --inventory Chemin vers l'inventaire Ansible utilisé
          #          --extra-vars Chemin vers le vault contenant les variables nécessaires au déploiement de l'application
          options: |
            --inventory ansible/inventories/setup.yml
            --extra-vars "@ansible/vault.yml"

```


# Partie 3 - Ansible

Après avoir téléchargé la clé privé pour l’accès SSH, modifié ses droits d’accès puis la testé (avec `chmod` et `ssh`), j’ai rajouté mon host (`jeremy.deplace.takima.cloud`) a mon fichier d’hosts (`/etc/ansible/hosts`).

Note: le répertoire `/etc/ansible` n’existait pas et devait être créée pour y rajouter le fichier `hosts`.

A l’aide de `ansible all -m ping` on vérifie que la configuration fonctionne bien pour ansible.

Le TD nous donne les directives suivantes qui permettent d’installer un apache sur notre machine:

```bash
ansible all -m yum -a "name=httpd state=present" 
--private-key=<path_to_your_ssh_key> -u centos --become  
```

## 3-1 Document your inventory and base commands

Le fichier d’inventaire suivant à été rédigé et sauvegardé sous `projet/ansible/inventories/setup.yaml` :

```yaml
all:
  vars:
	  #utilisateur faisant partie du groupe pouvant 
	  #posséder les super privilèges
    ansible_user: centos
		#Chemin vers la clé privé utilisé pour l'accès ssh
    ansible_ssh_private_key_file: /home/jerem/CPE/DevOps/id_rsa
	children:
    prod:
			#Url de mon serveur
      hosts: jeremy.deplace.takima.cloud
```

Le fichier d’inventaire permet de définir pour un “projet” ansible la liste des machines visées, les identifiants utilisées, les variables utilisées, etc.

Pour les prochaines commandes ansible, plutôt que d’utiliser les hosts par défaut définis dans le fichier `hosts` on passe par notre inventory qui contient la liste des hosts. 

L’option `--inventory chemin_vers_l_inventaire` permet de définir l’inventaire pour la commande ansible utilisée. exemple : 

`ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become`

Cette commande permet de desinstaller l’Apache précédemment installé.

Pour l’instant seul la commande `ansible all` a été utilisé. Cependant plusieurs options ont été utilisées :

| option      | Utilité                                                                 | Exemple / note                                                                         |
|-------------|-------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| -u          | Utilisateur pour lequel ansible va se connecter                         |                                                                                        |
| --become    | Permet d’obtenir les droits super utilisateurs pour les modules appelés | Nécessite que l’utilisateur utilisé fasse partie du groupe pouvant les obtenirs        |
| —inventory  | Chemin vers l’inventaire utilisé                                        | ./inventories/setup.yml                                                                |
| -m          | Nom du module utilisé sur le(s) machines liées à notre appel            | yum (gestionnaire de paquet)                                                           |
| -a          | Argument(s) du module utilisé, on a utilisé cette commande              | avec yum : name=httpd state=absent                                                     |
| —extra-vars | Variables supplémentaires passés lors de l’appel.                       | L’argument passé peut être une variable directement ou bien un fichier (voir un vault) |

Après avoir rajouter un playbook ainsi que des roles on obtient l’architecture de fichiers suivants pour la partie ansible du projet :

```bash
ansible
 ┣ inventories
 ┃ ┗ setup.yml
 ┣ roles
 ┃ ┣ api/ #Répertoire contenant les fichiers nécessaire au role api
 ┃ ┣ bd/ #Répertoire contenant les fichiers nécessaire au role bd
 ┃ ┣ docker/ #Répertoire contenant les fichiers nécessaire au role docker
 ┃ ┣ http/ #Répertoire contenant les fichiers nécessaire au role http
 ┃ ┣ front/ #Répertoire contenant les fichiers nécessaire au role front
 ┃ ┗ network/ #Répertoire contenant les fichiers nécessaire au role network 
 ┗ playbook.yml
```

## 3-2 Document your playbook

```bash
#playbook.yml
- hosts: all
  gather_facts: false
  become: true

  roles:
    - docker #responsable de l’installation de docker
    - network #responsable de la création du réseau utilisé par les conteneurs du projet
    - bd #responsable de la création d’un conteneur contenant la base de données du projet
    - api #responsable de la création d’un conteneur contenant l’api du projet
    - http # responsable de la création d’un conteneur servant de reverse proxy pour l'application
```

**Exemple de fonctionnement d’un role**

Chaque rôle possède une task `main.yml` qui définit les tâches liés à un rôles

```bash
#./roles/bd/tasks/main.yml
- name: Run my bd container
  import_tasks: ./run_bd.yml
```

Ici on importe les tâches décrites dans le fichier exemple: `run_bd.yml`