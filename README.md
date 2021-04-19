# Test_Jenkins_1
Créer un pipeline Jenkins sur un conteneur Docker (image jenkins/jenkins) qui execute un script pyhton script.py dispo sur ce repo github. Afin de pouvoir l'executer, on utilisera un conteneur docker:dind.

1. Créer un network docker 

```sh
docker network create jenkins 
```
2. Dans un premier conteneur on execute l'image docker:dind

```sh
docker run --name jenkins-docker --detach \
  --privileged --network jenkins --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 docker:dind --storage-driver overlay2
```
--privileged : L'exécution de Docker dans Docker nécessite actuellement un accès privilégié pour fonctionner correctement.  
--network jenkins : Cela correspond au réseau créé à l'étape précédente.  
--network-alias docker : Rend le Docker in Docker disponible en tant que nom d'hôte docker au sein du réseau jenkins.  
--env DOCKER_TLS_CERTDIR=/certs : 	Active l'utilisation de TLS dans le serveur Docker. En raison de l'utilisation d'un conteneur privilégié, c'est recommandé, bien que cela nécessite l'utilisation du volume partagé décrit ci-dessous. Cette variable d'environnement contrôle le répertoire racine dans lequel les certificats Docker TLS sont gérés.     
--volume jenkins-docker-certs:/certs/client : volume de sauvegarde   
--volume jenkins-data:/var/jenkins-home  : volume de sauvegarde    
--storage-driver overlay2 : 	Le pilote de stockage pour le volume Docker.     

3. Créer un Dockerfile pour l'image docker de jenkins/jenkins : 

```sh
FROM jenkins/jenkins:2.277.2-lts-jdk11
USER root
RUN apt-get update && apt-get install -y apt-transport-https \
       ca-certificates curl gnupg2 \
       software-properties-common
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
RUN apt-key fingerprint 0EBFCD88
RUN add-apt-repository \
       "deb [arch=amd64] https://download.docker.com/linux/debian \
       $(lsb_release -cs) stable"
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.24.6 docker-workflow:1.26"
```
4. Puis contruire une nouvelle image docker à partir du Dockerfile : 

```sh
docker build -t myjenkins-blueocean:1.1
```
5. Puis executer l'image dans un conteneur : 

```sh 
  docker run --name jenkins-blueocean --detach \
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  --volume "$HOME":/home \
  myjenkins-blueocean:1.1
```
6. Récupérer le code d'initialisation de Jenkins dans 
```sh docker logs jenkins-blueocean```
7. Créer un pipeline puis executer le Jenkinsfile suivant : 

```sh
 pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'f0877267-74f7-40ef-a621-d2d66c776fc1', url: 'https://github.com/Valou0401/Test_Jenkins_1.git']]])
            }
        }
        stage('Build'){
            agent {
                docker {
                    image 'python:latest' 
                }
            }
            steps {
                git branch: 'main', credentialsId: 'f0877267-74f7-40ef-a621-d2d66c776fc1', url: 'https://github.com/Valou0401/Test_Jenkins_1.git'
                echo'git ok'
                sh 'python script.py'
            }
        }
        stage('Test'){
            steps {
                echo 'the job has been tested'
            }
        }
    }
    
}
```
