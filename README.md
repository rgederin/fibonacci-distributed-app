# Fibonacci distributed application

Distributed application for calculating Fibonacci numbers and illustrating multi-containers deployment on AWS Elastic Beanstalk using Travis CI.

## Tools and technologies

* NodeJS
* ReactJS
* Nginx
* Docker
* Docker compose
* Travis CI
* AWS EB/RDS/EC


## Application overview

Frontend + backend application which provides user interface and cumputation logic for Fibonacci numbers.

![app](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/app.png)

Data which will be displayed in `Indices I have seen` and in `Calculated falues` will come from Postgres and Redis respectively:

![app-with-be](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/app-with-be.png)


## Application workflow

Overal application workflow is described below:

![workflow](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/workflow.png)


## Development environment architecture

Development and production environment will slightly differ. Development environment will consist of next components (each of them will run in own Docker container):

![dev-env](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/dev-env.png)

* [Nginx](https://github.com/rgederin/fibonacci-distributed-app/blob/master/nginx) - web server for routing requests. If incoming request want to access frontend - nginx will route it to React server. If incoming request is wants to call API - such request will be routed to Express server. 
* [React server](https://github.com/rgederin/fibonacci-distributed-app/blob/master/client) - simple UI implemented using React JS
* [Express server](https://github.com/rgederin/fibonacci-distributed-app/blob/master/server) - NodeJS Express application for exposing API for the client. 
* Redis - key-value storage which stores all indices and calculated values as key-value pairs.
* Postgres - relational database which stores a permanent list of indices that have bee receives
* [Worker](https://github.com/rgederin/fibonacci-distributed-app/blob/master/worker) - watches Redis for new indices. Pulls each new index, calculates new Fibonacci value, puts it back into Redis. 

## Running multi-container application locally

Each of mentioned components will run in separate Docker container. We will use [docker compose](https://docs.docker.com/compose/) as an orchestration tool for running multiple Docker containers locally. 

Lets firstly dockerize our React and Node JS applications for running locally. We will use `Dockerfile.dev` for local configuration. For React app we will use `react-scripts start` as command for launching react server with our UI and `nodemon` for launching Node apps. Both ways provide for us live changes so we will be able to see our changes which we will make while coding immediately.

[Dockerfile.dev](https://github.com/rgederin/fibonacci-distributed-app/blob/master/client/Dockerfile.dev) for client application:

```
FROM node:alpine
WORKDIR "/app"
COPY ./package.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "start"]
```

[Dockerfile.dev](https://github.com/rgederin/fibonacci-distributed-app/blob/master/server/Dockerfile.dev) for server and worker applications:

```
FROM node:14.14.0-alpine
WORKDIR "/app"
COPY ./package.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]
```
