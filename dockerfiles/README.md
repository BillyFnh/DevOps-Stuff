# Collection of `dockerfile` / `docker-compose.yaml`

## Pipeline Worker For Continuous Deployment

Using a combination of Terraform, Azure CLI, and Kubectl to provision Azure resources and deploy workloads to AKS.

Credentials for Azure, or other SSH keys are passed in from host via volume mount.

### Dockerfile

```dockerfile
FROM debian:stable-slim

RUN apt-get update && apt-get install -y sudo curl gnupg gnupg2 software-properties-common apt-transport-https lsb-release ca-certificates apt-utils

# install azure cli
RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash

# install terraform
RUN curl -fsSL https://apt.releases.hashicorp.com/gpg | apt-key add -
RUN apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
RUN apt-get update && apt-get install -y terraform

# install kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
RUN install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

CMD ["/bin/bash"]
```

### Example docker-compose

```yaml
version: "3.7"
services:
  pipeline-worker:
    container_name: pipeline-worker
    image: <image-name>
    volumes:
      - "/root/.azure:/root/.azure"
      - "/root/.docker:/root/.docker"
      - "/root/.ssh:/root/.ssh"
```

## Simple NPM HTTP Server

Using NPM http-server to serve static webpages.

```dockerfile
FROM node:12.2.0

WORKDIR /app
COPY /app/dist/app /app
RUN npm install --global http-server
CMD http-server --proxy http://localhost:4200? --port 4200
```

## Development Environment - Angular

Using Angular's `ng serve` to serve webpage for dev and test purposes.

```dockerfile
FROM node:12.2.0

# install chrome for protractor tests
RUN wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add -
RUN sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list'
RUN apt-get update && apt-get install -yq google-chrome-stable

# set working directory
WORKDIR /app

# add `/app/node_modules/.bin` to $PATH
ENV PATH /app/node_modules/.bin:$PATH

# install and cache app dependencies
COPY /app/package.json /app/package.json
RUN npm install
RUN npm install -g @angular/cli@7.3.9

# add app
COPY /app /app

# start app
CMD npm run start
```

## Remote Script Executor - Splunk Correlation Search Extraction

For executing script remotely.

```dockerfile
FROM ubuntu:16.04
WORKDIR /root
COPY . .

RUN apt-get update
RUN apt-get install -y software-properties-common
RUN add-apt-repository ppa:deadsnakes/ppa
RUN apt-get update
RUN apt-get install -y vim
RUN apt-get install -y python3.6
RUN apt-get install -y curl
RUN curl https://bootstrap.pypa.io/get-pip.py | python3.6 - --user
RUN python3.6 -m pip install requests

RUN apt-get install -y sshpass
RUN mkdir /root/.ssh

# save fingerprint of target machine to prevent interactive terminal prompts
RUN echo "<ip-address> ecdsa-sha2-nistp256 <ssh-key>"  > /root/.ssh/known_hosts

CMD ./some-script.sh
```

## Python Scrapy

For executing Scrapy for web scraping.

```dockerfile
FROM python:3

WORKDIR /root/scraper
RUN pip install Scrapy toolz requests
COPY /scraper /root/scraper
RUN mkdir /data
RUN mkdir /data/scraper-result-archive

CMD scrapy crawl apps && scrapy crawl docs && python3 ./convert.py
```

## Custom MongoDB ReplicaSet

Uses Mongo official image (version 4.2.3) and custom startup scripts to initialize/reconfigure MongoDB ReplicaSet.

```dockerfile
FROM mongo:4.2.3
WORKDIR /root
ENV MONGODB_ID mongo-0

RUN apt update
RUN apt-get install net-tools

COPY /startup-script-mongo /root

CMD ./startup-$MONGODB_ID.sh
```

## Node.js

For running Node.js applications.

```dockerfile
FROM node:10
WORKDIR /usr/src/app
COPY ./package*.json ./
COPY ./comparisonWorker.js ./comparisonWorker.js
COPY ./compareResult.js ./compareResult.js
COPY ./metrics.js ./metrics.js

RUN npm install
CMD ["node", "comparisonWorker.js"]
```

## RabbitMQ Server

Loads custom configuration file onto RabbitMQ Server.

```dockerfile
FROM rabbitmq
COPY ./rabbitmq.conf /etc/rabbitmq/
COPY ./enabled_plugins /etc/rabbitmq/
CMD ["rabbitmq-server"]
```
