FROM node:11.6

RUN apt-get update
RUN npm install pm2 -g

RUN mkdir -p /opt/app
WORKDIR /opt/app
ADD package.json package-lock.json ./
RUN npm ci --production
ADD . /opt/app