FROM registry.access.redhat.com/ubi8/ubi:latest as node-base
RUN yum install -y npm && yum clean all && rm -rf /var/cache/yum

FROM node:10
WORKDIR /app
COPY package.json /app
RUN npm install
COPY . /app
CMD node app.js
EXPOSE 8080