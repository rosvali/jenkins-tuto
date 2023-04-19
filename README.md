# Jenkins tutorial

https://www.youtube.com/watch?v=6YZvp2GwT0A

## Run Jenkins in a container environment

https://www.jenkins.io/doc/book/installing/docker/

Create a bridge network with docker
```
docker network create jenkins
```

### Build a customize Jenkins image to add Blue Ocean plugin

Dockerfile
```
FROM jenkins/jenkins:2.387.2
USER root
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow"
```

Build the image
```
docker build -t myjenkins-blueocean:2.387.2-1 .
```

Run the container
```
docker run \
  --name jenkins-blueocean \
  --restart=on-failure \
  --detach \
  --network jenkins \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 \
  --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  myjenkins-blueocean:2.387.2-1 
```

### Unlock Jenkins

```
docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword
```

Then install Jenkins with suggested plugins and sign up

## Set up an agent

### Install Docker cloud provider

Install Docker via the Plugin Manager interface and wait to jenkins to reboot

### Configure a cloud

Manage Jenkins -> Configure clouds -> add a new cloud Docker


In docker host uri you need to enter the tcp adress of a median socat container `tcp://172.19.0.3:2375`

and check the enabled

https://hub.docker.com/r/alpine/socat

```
docker run -d --restart=always -p 127.0.0.1:2376:2375 --network jenkins -v /var/run/docker.sock:/var/run/docker.sock alpine/socat tcp-listen:2375,fork,reuseaddr unix-connect:/var/run/docker.sock
```

### Add a docker agent template

choose a label and a name, check the enabled

set the image of the agent `devopsjourney1/myjenkinsagents:python`

Instance Capacity: 2

Remote File System Root: `/home/jenkins`

## Configure a pipeline

Dashboard -> new item -> create a pipeline

Go to configure a pipeline -> and add a pipeline script SCM + Repository url + jenksins path script

Jenksinsfile
```
pipeline {
    agent { 
        node {
            label 'docker-agent-python'
            }
    }
    stages {
        stage('Build') {
            steps {
                echo "Building.."
                sh '''
                cd app
                pip install -r requirements.txt
                '''
            }
        }
        stage('Test') {
            steps {
                echo "Testing.."
                sh '''
                python3 app/hello.py
                python3 app/hello.py --name=Brad
                '''
            }
        }
    }
}
```