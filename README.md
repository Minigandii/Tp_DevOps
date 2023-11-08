## TP-1: Docker
### Docker
#### DockerFile
```DockerFile
FROM postgres:14.1-alpine
ENV POSTGRES_DB=db 
   POSTGRES_USER=usr
   POSTGRES_PASSWORD=pwd
COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d/
COPY 02-InsertData.sql /docker-entrypoint-initdb.d/
```

Dans cette section, on prépare notre conteneur de base de données. Pour commencer, on dit à Docker quelle image utiliser. Les lignes avec "ENV" sont pour configurer notre base de données, comme son nom, nom d'utilisateur et mot de passe. Avec "COPY", on met nos fichiers SQL dans le conteneur lors de sa création.
#### Commande
Voici les étapes pour lancer nos conteneurs :
```Command
1- docker network create app-network

2- docker build -t minigandii/databasecontainer .

3- docker run -p "5432:5432" --net=app-network --name=database -d minigandii/databasecontainer

4- docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
```
Étape 1 : Crée un réseau local.
Étape 2 : Construit notre image à partir du Dockerfile.
Étape 3 : Lance notre conteneur sur le réseau créé.
Étape 4 : Démarre un outil d'administration de base de données sur le même réseau.
### Back-API
Ici, on réduit la taille de notre image et on organise les couches. Dans notre cas, on construit l'image avec Maven, puis on la prépare en copiant les fichiers nécessaires.

```dockerfile
#Build
#Construction de l'image maeven
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
#Définition d'une variable d'environnement lié au dossier
ENV MYAPP_HOME /opt/myapp
#définit le dossier de travail
WORKDIR $MYAPP_HOME
#copie le fichier pom
COPY pom.xml .
#copie le dossier src
COPY src ./src
#lance la construction du projet Maeven
RUN mvn package -DskipTests

FROM amazoncorretto:17
#Définition d'un variable d'environnement
ENV MYAPP_HOME /opt/myapp
#Définition du dossier de travail
WORKDIR $MYAPP_HOME
#copie les fichier jar de target dans la dossier de travail
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
#lancement du programme une fois l'image lancée
ENTRYPOINT java -jar myapp.jar
```

### Link Application
Le fichier docker-compose.yml permet de simplifier la gestion de nos Dockerfiles et offre une vue plus claire de nos conteneurs et images.

On définit une liste de "services" qui contiennent tous nos conteneurs. Chaque conteneur (backend, httpd, etc.) est configuré avec des mots-clés :

"build" : pour créer notre image.
"network" : pour connecter notre conteneur aux réseaux appropriés.
"depends-on" : pour spécifier les dépendances de notre conteneur.

## TP-2: Github-Action

Testcontainers est une bibliothèque Java qui facilite la gestion de différents conteneurs dans nos tests, notamment pour tester la base de données et les autres services.
```yaml
name: CI devops 2023
on:
  push:
    branches:
      - main
  pull_request:
jobs:
  test-backend: #nom du test
    runs-on: ubuntu-22.04 #test réalisé sur ubunto
    steps: #étapes à réaliser
      - uses: actions/checkout@v2.5.0 #copie le github dans l'environnement test
      - name: Set up JDK 17 #définition du nom de l'étape
        use: actions/setup-java@v2 #précise la version de Java
        with:
          java-version: 17
      - name: Build and test with Maven #définition du nom de l'étape
        run: mvn clear verify #lance la commande pour le test
```

Vous pouvez voir les résultats de SonarCloud ici. Il montre deux vulnérabilités de sécurité (dans l'onglet "Sécurité") et une couverture de code de seulement 53,6% (dans l'onglet "Couverture"). SonarCloud dit que l'analyse n'est pas satisfaisante, car il faut une couverture supérieure à 80%.

## TP-3: Ansible
Dans notre inventaire, on a un fichier setup qui configure Ansible.
##### setup.yml:
```yaml
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: /etc/ansible/id_rsa2
 children:
   prod:
     hosts: clement.gandillon.takima.cloud
```
Ce fichier dit à Ansible l'adresse du serveur (hosts), l'utilisateur (ansible_user) et la clé d'accès (ansible_ssh_private_key_file).

La commande suivante configure tous les serveurs en utilisant ce fichier :
```command
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

Pour que Ansible commence les configurations, on crée un "playbook" qui exécute nos "rôles", lançant ainsi les configurations et les services sur notre serveur.
###### playbook.yml
```yaml
- hosts: all
  gather_facts: false
  become: true
  roles:
    - docker
    - network
    - database
    - app
    - proxy
```
Tous les serveurs sont concernés (hosts: all), et cela s'exécute en tant que super-utilisateur (become: true). Ensuite, on liste les rôles à exécuter avec Ansible (roles: -docker...).
###### docker/tasks/main.yml
```yaml
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest
  
- name: Install lvm2
  yum:
    name: lvm2
    state: latest
  
- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  yum:
    name: docker-ce
    state: present
  
- name: Install python3
  yum:
    name: python3
    state: present
  
- name: Install docker with Python 3
  pip:
    name: docker
    executable: pip3
  vars:
    ansible_python_interpreter: /usr/bin/python3

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```
Le docker_container tasks permet de configurer les containers docker dans notre serveur. On a besoin de plusieurs installation comme docker (*Install Docker*), python (*Install python3*) et lvm2 (*Install lvm2*). On ajoute également un dossier pour notre projet (*add repo docker*) et on termine le rôles en vérifiant que tout est lancé (*Make sure Docker is running*).
