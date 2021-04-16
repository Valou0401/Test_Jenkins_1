# Test_Jenkins_1
Create a simple pipeline to build script.py in a jenkins project on docker container.

1. install and launch the jenkins/jenkins image from dockerHub in a docker container :

docker run -d \
-v jenkins_home:/var/jenkins_home \
-p 8080:8080 \
-p 50000:50000 \
--name jenkins \
jenkins/jenkins:latest

2. create an admin account with the initialization password in docker logs jenkins.


