#Docker
What it is 
docker-machine vs docker  
engine vs cli
images vs container
idempotenty
repos vs tags (latest vs specific tags)  
docker volumes vs 'real' volumes

#Docker commands
- run = start container and optinally run a command
- exec = run a command on a started container
- stop = stop a container
- ps = list containers
- images = show images
- image
    - image ls = show images, same as -images
    - image pull $REPO:$TAG = pull image
    - image push $REPO:$TAG = push image
    - image tag = reate a tag TARGET_IMAGE that refers to SOURCE_IMAGE
- rm = remove a stopped container
- rmi = remove an image if it doesn't have any linked containers
- volume
    * $SUBCOMMANDS for volume management

#Running Docker
$CONTAINER can refer to the "container ID" or the "Name" of the container

##Start a container
- `docker run --options $IMAGE:$TAG $COMMAND`
    - --options
        - -d = for daemonize it--return shell controll
        - -p#:# share ports as pCONTAINER:HOST
        - -t terminal
        - -i interactive
        - -e map an environment variable
    - `docker run alpine:20190925`
    - `docker run alpine:20190925 --name LearnIt`
        - won't work. it will try to run the command '--name LearnIt'  
    - `docker run --name LearnIt alpine:20190925`
    - `docker run -d nginxdemos/hello`
    - `docker run -d -p80:80 nginxdemos/hello`
    - `docker run -v $SOURCE:$TARGET alpine:20190925`


##list images
docker images


##list containers
To show running containers
- `docker ps`

To show all containers, including stopped
- `docker ps -a`

##stop a container
- `docker stop $CONTAINER`


##remove a container
- `docker rm $CONTAINER`


##remove images
- `docker rmi $IMAGE`


##docker hub
hub.docker.com  
or google "$PROGRAM container"  
how to find tags


##list all containers
-`docker ps -a`


#Making images
Don't bake config/connection/auth/cred in image (either when building the code put into the container, or during the creation of the image itself). Feed configuration in through env vars, volume maps (least preferred method) or [config map/secrets] (only if using Kube)

##dockerfiles
See https://docs.docker.com/engine/reference/builder/

Each new line is a new command and therefore probably a new layer. Becasue of this put infrequent changes near the top of the dockerfile and frequent changes at the bottom of the dockerfile.

- FROM $IMAGE:$TAG
    - must be first thing in dockerfile. Except for comments.
    - specifies what base image to use
- COPY $SOURCE $DESTINATION
    - copies files (recursively if pointed at a directory)
- EXPOSE $PORT 
    - Documentation on what should be mapped as a port via docker run -p$PORT:$PORT
- ENV $KEY $VALUE
    - Set $KEY environment value to $VALUE. Useful for passing build parameters.
- ENTRYPOINT ["$COMMAND","-$OPTION1","-$OPTION2", "etc."]
    - What command to run if `docker run` or `docker exec` doesn't specify a command
- CMD
    - depreciated. use ENTRYPOINT.
###example dockerfiles
#### simple
```
FROM busybox
RUN echo "hello world"
```
#### complex
as taken from https://github.com/robertdebock/docker-cntlm/blob/master/Dockerfile
```
FROM alpine:3.8

LABEL version="1.3"

RUN apk add --no-cache --virtual .build-deps curl gcc make musl-dev && \
    curl -o /cntlm-0.92.3.tar.gz http://kent.dl.sourceforge.net/project/cntlm/cntlm/cntlm%200.92.3/cntlm-0.92.3.tar.gz && \
    tar -xvzf /cntlm-0.92.3.tar.gz && \
    cd /cntlm-0.92.3 && ./configure && make && make install && \
    rm -Rf cntlm-0.92.3.tar.gz cntlm-0.92.3 && \
    apk del --no-cache .build-deps

ENV USERNAME   example
ENV PASSWORD   UNSET
ENV DOMAIN     example.com
ENV PROXY      example.com:3128
ENV LISTEN     0.0.0.0:3128
ENV PASSLM     UNSET
ENV PASSNT     UNSET
ENV PASSNTLMV2 UNSET
ENV NOPROXY    UNSET

EXPOSE 3128

ADD start.sh /start.sh
RUN chmod +x /start.sh

CMD /start.sh
```


##multi stage builds
Useful for keeping build seperate from run

```
FROM ruby:2.5.1-alpine3.7  AS build-env
RUN apk update && apk add --no-cache nodejs build-base
RUN apk add yarn --no-cache --repository http://dl-3.alpinelinux.org/alpine/v3.8/community/ --allow-untrusted
RUN mkdir -p /app
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle install -j 4
COPY . ./
COPY package.json yarn.lock ./
RUN yarn install
RUN make _site

VOLUME /app

FROM nginx:1.13.0-alpine
WORKDIR /usr/share/nginx/html
COPY --from=build-env /app/_site ./

EXPOSE 80

##Build dockerfile
`docker build -t repo:tag $Directory`
```

# Docker compose
up
build
override

##sample
```
version: '3.5'
      
services:
  Jenkins:
    image: jenkins/jenkins:latest
    container_name: Jenkins
    networks: 
        - A..._CNTLM_network
    ports: 
        - "8080:8080"
        - "50000:50000"

  Git:
    image: gitlab/gitlab-ce:11.2.1-ce.0
    container_name: Git
    hostname: Git
    networks: 
        - A..._CNTLM_network
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://127.0.0.1:8050'
        gitlab_rails['gitlab_shell_ssh_port'] = 522
    ports:
     - "8050:8050"
     - "522:22"

  CNTLM:
    image: "jaschac/cntlm"
    container_name: CNTLM
    networks: 
        - A..._CNTLM_network
    ports: 
        - "3128:3128"  

networks:
    A..._CNTLM_network:
        driver: bridge

        
```

###override
```
version: '3.5'

services:

  Jenkins:
    volumes:
      - "/d/Temp/Jenkins/MyJenkins:/var/jenkins_home"
    ports:
      - "8080:8080"
      - "50000:50000"
      
  Git:
    volumes:
     - /d/Temp/Jenkins/gitlab/config:/etc/gitlab
    # - /d/Temp/Jenkins/gitlab/logs:/var/log/gitlab
    # - /d/Temp/Jenkins/gitlab/data:/var/opt/gitlab
    privileged: true
    ports:
      - "8050:8050"
      - "522:22"
      
  CNTLM:
    environment:
      - CNTLM_DOMAIN=u....com
      - CNTLM_PROXY_URL=a....com
      - CNTLM_PROXY_PORT=8080
      - CNTLM_USERNAME=P...d
      - CNTLM_PASSLM=9...6
      - CNTLM_PASSNT=F...7
      - CNTLM_PASSNTLMv2=9...B
    ports:
      - "3128:3128"

```

#kubernetes

## How to get your hands on it

###Online
kubernetes.io interactive tutorials (non declaritive though, no writing .yaml files.)

###on your computer
- Kubernetes IN Docker (KIND)
- minikube




#to add
how to use kube
what is a microservice and how to use them. How to architect them. The bits that support that architecture, how that plays back into kube as a scalable/distributed system
