# Rapport DevOps

## TP part 01 - Docker  
**1-1 Document your database container essentials: commands and Dockerfile.**  
```
# on utilise l'image postgres version 14.1
FROM postgres:14.1-alpine

# on initialise les variables de la base de données
ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

# on copie les deux fichiers SQL de création de tables et de remplissage de la table dans le container
COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d/
COPY 02-InsertData.sql /docker-entrypoint-initdb.d/
```
**1-2 Why do we need a multistage build? And explain each step of this Dockerfile.**  
A multistage build uses multiple FROM statements in the same Dockerfile. Each statement starts
a new stage and has its own image and instruction. It can be used to seperate the build and 
the runtime environment
```
# Build stage
# On utilise l'image Corretto 17 avec maven pour la phase de build et on nomme cette phase myapp-build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
#on définit une variable d'environnement MYAPP_HOME qui vaut /opt/myapp
ENV MYAPP_HOME /opt/myapp
# On définit le répertoire de travail
WORKDIR $MYAPP_HOME
# On copie le fichier pom.xml et le dossier src
COPY pom.xml .
COPY src ./src
# On empaquete l'application en sautant les tests
RUN mvn package -DskipTests

# Run stage
# On utilise l'image Amazon Corretto 17
FROM amazoncorretto:17
# On déclare une autre variable d'environnement
ENV MYAPP_HOME /opt/myapp
# On définit le répertoire de travail
WORKDIR $MYAPP_HOME
#On copie les fichiers .jar dans le container
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
# On définit le fichier à lancer par défaut
ENTRYPOINT java -jar myapp.jar
```
**1-3 Document docker-compose most important commands.**

**version:** Définit la version de Docker Compose utilisée  
**services:** Définit la liste des services déployés sous forme de container  
**build:** Indique le dossier à build  
**container_name:** Définit le nom du container
**networks:** Spécifie à quel réseau le service est connecté. Permet la communication entre les containers
**depends_on:** Spécifie la dépendance entre les services, si le service spécifié n'est pas en cours d'éxecution, 
le service actuel ne sera démarré  
**ports:** Définit les ports qui seront exposés par le container

**1-4 Document your docker-compose file.**  
```
version: '3.7' # Version de Docker Compose

# Liste des services utilisés
services:
    backend: # service backend
        build: ./simple-api # on build le dossier simple-api
        container_name: backend # on appelle le container backend
        ports: # on définit les ports
        - "8081:8081"
        networks: # on définit le network
        - app-network
        depends_on: # on définit le service dont il dépend
        - database
    
    database: # service de la bdd
        build: ./database # on build le dossier database
        container_name: database # on appelle le container database
        networks: # on définit le réseau utilisé
            - app-network
    
    httpd: # service httpd (proxy)
        build: ./http-server # on build le dossier http-server
        container_name: httpd # on donne le nom httpd au container 
        networks: # on définit son réseau
            - app-network
        depends_on: # on défiint le container dont il dépend
            - backend

networks: # on déclare les réseaux
app-network: # le réseau utilisé par les services du projet
```
**1-5 Document your publication commands and published images in dockerhub.**  

`docker tag tp-devops-SERVICE MYUSERNAME/tp-devops-SERVICE:1.0`  
Cette commande crée un nouveau tag pour l'image Docker tp-devops-SERVICE. 
Le tag permet de distinguer différentes versions de l'image.

`docker push USERNAME/tp-devops-SERVICE:1.0`
Cette commande pousse l'image créer avec son tag sur Docker Hub, associé à mon compte  
J'ai ainsi ces images sur mon Docker Hub : 
- ttchpln/tp-devops-httpd : le proxy
- ttchpln/tp-devops-database : ma base de données
- tchpln/tp-devops-backend : mon api

## TP part 02 - GitHub Actions  
**2-1 What are testcontainers?**  

Testcontainers is a Java library used for integration testing that provides lightweight
instances of Docker containers for the tests.

**2-2 Document your GitHub Actions configurations.**  

```
name: CI devops 2023 # nom du workflow
on: # workflow déclenché sur les branches main et develop
  push:
    branches:
      - main
      - develop 
  pull_request:

jobs: # liste des jobs
  test-backend: # nouveau job nommé test-backend
    runs-on: ubuntu-22.04 # job qui tourne sur une VM Ubuntu 22.04
    steps: # étapes à exécuter dans le job
     # vérifie le code avec actions/checkout@v3
      - uses: actions/checkout@v3
      - name: Set up JDK 17 # configuration de l'environnement Java
        uses: actions/setup-java@v3
        with:
          java-version: 17
          distribution: 'corretto'

     # Build l'application avec Maven
      - name: Build and test with Maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=ttnchpln_TP-DevOps -Dsonar.organization=ttnchpln -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml
        working-directory: simple-api


  build-and-push-docker-image: # nouveau job
    needs: test-backend # ce job s'exécute uniquement quand test-backend sera exécuté
    runs-on: ubuntu-22.04 # la VM utilisé
    steps: # les étapes du job  
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Login to Docker Hub # connection à Docker Hub
        run: docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_TOKEN_PASSWORD }}
      
      - name: Build image and push backend # Build et Push des images du backend
        uses: docker/build-push-action@v3
        with:
          context: ./simple-api
          tags: ${{secrets.DOCKER_USERNAME}}/tp-devops-simple-api:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database # Build et Push des images de la base de données
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags: ${{secrets.DOCKER_USERNAME}}/tp-devops-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd # Build et Push des images du proxy httpd
        uses: docker/build-push-action@v3
        with:
          context: ./http-server
          tags: ${{secrets.DOCKER_USERNAME}}/tp-devops-http-server:latest 
          push: ${{ github.ref == 'refs/heads/main' }}  
```

**2-3 Document your quality gate configuration.**  
Je ne sais pas où trouver la quality gate, mais le principe c'est de définir des critères 
de validations en se basant sur les critères de qualités de code.

## TP part 03 – Ansible  
**3-1 Document your inventory and base commands.**  
```
all: #groupe all qui inclue tout les hotes
  vars: # définit les variables utilisées
    ansible_user: centos # nom de l'utilisateur pour la connection SSH
    ansible_ssh_private_key_file: /etc/ansible/id_rsa # clé RSA pour la connection SSH
  children: # liste des autres inventaires
    prod: # groupe prod
      hosts: titouan.chapalain.takima.cloud # nom de l'hote
```
**3-2 Document your playbook.**  

```
- hosts: all # on exécute sur le groupe all
  gather_facts: false # on ne récupère pas les faits
  become: true # on devient super user
  roles: # on exécute les roles suivants :
    - docker
    - network
    - database
    - app
    - proxy
    - front
```

**3-3 Document your docker_container tasks configuration.**  

```
---

# Install Docker

- name: Install device-mapper-persistent-data # installation de "device-mapper-persistent-data" 
  yum:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2 # installation de "lvm2"
  yum:
    name: lvm2
    state: latest

- name: add repo docker # ajout du repo Docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker # installation de Docker
  yum:
    name: docker-ce
    state: present

- name: Install python3 # Installation de Python3
  yum:
    name: python3
    state: present

- name: Install docker with Python 3 # Installation de Docker avec pip (pour Python)
  pip:
    name: docker
    executable: pip3
  vars:
    ansible_python_interpreter: /usr/bin/python3

- name: Make sure Docker is running # on s'assure que Docker est lancé
  service: name=docker state=started
  tags: docker

```
