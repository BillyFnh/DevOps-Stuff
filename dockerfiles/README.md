# Collection of dockerfile

## HTTP Server

Using NPM http-server to serve webpages via HTTP.

```dockerfile
FROM node:12.2.0

WORKDIR /app
COPY /app/dist/app /app
RUN npm install --global http-server
CMD sed -i "s/172.31.38.103:8000/$BACKEND_API_ENDPOINT/g" /app/main.js && http-server --proxy http://localhost:4200? --port 4200
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
RUN echo "172.31.50.83 ecdsa-sha2-nistp256 <ssh-key>"  > /root/.ssh/known_hosts

CMD ./start-remote-splunk-conf-extraction.sh
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
