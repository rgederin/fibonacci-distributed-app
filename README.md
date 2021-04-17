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

### Dockerize client and server components

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

### Dockerize nginx

First of all, below you could find diagram which is cleary described Nginx role in our development environment. We are using it as proxy for routing requests to the client and server.

![nginx-role](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/nginx-role.png)

In order to configure nginx for our needs we will use `default.conf` file where we will add configuration rules to the Nginx:

![default-conf](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/default-conf.png)

To be more precise, below you could find [default.conf](https://github.com/rgederin/fibonacci-distributed-app/blob/master/nginx/default.conf) content:

```
upstream client {
  server client:3000;
}

upstream api {
  server api:5000;
}

server {
  listen 80;

  location / {
    proxy_pass http://client;
  }

  location /api {
    rewrite /api/(.*) /$1 break;
    proxy_pass http://api;
  }

  location /sockjs-node {
    proxy_pass http://client;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
  }
}
```

After this we need to dockerize nginx using base image and put there our configuretion. As previously we will do this using [Dockerfile.dev](https://github.com/rgederin/fibonacci-distributed-app/blob/master/nginx/Dockerfile.dev):

```
FROM nginx
COPY ./default.conf /etc/nginx/conf.d/default.conf
```

### Using docker compose for running multi container application

First of all we need to create docker-compose.yml and specify there all needful information which are needful for running containers.

```
version: "3"
services:
  postgres:
    image: "postgres:latest"
    environment:
      POSTGRES_PASSWORD: postgres

  redis:
    image: "redis:latest"

  nginx:
    restart: always
    build:
      dockerfile: Dockerfile.dev 
      context: ./nginx
    ports:
      - "3050:80"
    depends_on:
      - api
      - client
      
  api:
    build:
      dockerfile: Dockerfile.dev
      context: ./server
    volumes:
      - /app/node_modules
      - ./server:/app
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379
      PGUSER: postgres
      PGHOST: postgres
      PGDATABASE: postgres
      PGPASSWORD: postgres
      PGPORT: 5432

  worker:
    build:
      dockerfile: Dockerfile.dev
      context: ./worker
    volumes:
      - /app/node_modules
      - ./worker:/app
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379

  client:
    stdin_open: true
    build:
      dockerfile: Dockerfile.dev
      context: ./client
    volumes:
      - /app/node_modules
      - ./client:/app
```

In order to start our application you should execute `docker compose up` and when all containers will be up and running, check http://localhost:3050/ page. You should see something like:

![running-app-local](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/running-app-local.png)

And to verify that all containers are up and running you could execute `docker ps`:

![docker-ps](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/docker-ps.png)

## Continuous Integration with Travis CI

We will use Travis CI for organizing CI process and Docker Hub for storing docker images for our services. Below you could see whole CI/CD flow (deployment to AWS Elastic Beanstalk we will review in next section):

![flow](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/flow.png)

### Production docker files

First of all we will add `production` Docker files for each of our services. In our case there will be little to no differences between dev and prod docker files, but however lets follow this approach.

Production Dockerfiles for `worker` and `server` services are the same and differ from dev ones only with starting command:

```
FROM node:14.14.0-alpine
WORKDIR "/app"
COPY ./package.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "start"]
```

For client service we need to add one more nginx instance in order to serving static React content. 

![flow](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/nginx-prod.png)

Also we need to add corresponding `default.conf` to [client nginx](https://github.com/rgederin/fibonacci-distributed-app/blob/master/client/nginx/default.conf):

```
server {
  listen 3000;
 
  location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
    try_files $uri $uri/ /index.html;
  }
}
```
After this our production [Dockerfile](https://github.com/rgederin/fibonacci-distributed-app/blob/master/client/Dockerfile) will looks like:

```
FROM node:alpine as builder
WORKDIR '/app'
COPY ./package.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx
EXPOSE 3000
COPY ./nginx/default.conf /etc/nginx/conf.d/default.conf
COPY  --from=builder /app/build /usr/share/nginx/html
```

### Setting up Travis CI

First thing which we need to do is to integrate our repository with Travis CI - this is quite straight forward (https://docs.travis-ci.com/user/tutorial/#to-get-started-with-travis-ci-using-github).

Also we need to specify in Travis CI `DOCKER_ID` and `DOCKER_PASSWORD` credentials in order to push images to Docker Hub:

![travis](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/travis.png)

After this lets add `.travis.yml` in the root of our repository:

```
sudo: required
services:
  - docker

before_install:
  - docker build -t rgederin/fib-client-test -f ./client/Dockerfile.dev ./client

script:
  - docker run -e CI=true rgederin/fib-client-test npm test -- --coverage
after_success:
  # Build production docker images
  - docker build -t rgederin/fib-client ./client
  - docker build -t rgederin/fib-server ./server
  - docker build -t rgederin/fib-worker ./worker
  - docker build -t rgederin/fib-nginx ./nginx

  # Log in to the docker CLI
  - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_ID" --password-stdin

  # Push images to Docker hub
  - docker push rgederin/fib-client
  - docker push rgederin/fib-server
  - docker push rgederin/fib-worker
  - docker push rgederin/fib-nginx
```

After this push your changes in github and check build in Travis CI:

![travis-2](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/travis-2.png)

And check Docker Hub (be sure that Travis deploys new images there)

![docker-hub](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/docker-hub.png)

## Deploy on AWS

We will use AWS Elastic Beanstalk for deploying our multi container application. In order to describe for AWS EB our containers configuration we will use [Dockerrun.aws.json](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_docker_v2config.html) file. It has similar purpose as `docker-compose.yml`:


![run](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/run.png)

So below you could find our `Dockerrun.aws.json` file:

```
{
    "AWSEBDockerrunVersion": 2,
    "containerDefinitions": [
        {
            "name": "client",
            "image": "rgederin/fib-client",
            "hostname": "client",
            "essential": false,
            "memory": 128
        },
        {
            "name": "server",
            "image": "rgederin/fib-server",
            "hostname": "api",
            "essential": false,
            "memory": 128
        },
        {
            "name": "worker",
            "image": "rgederin/fib-worker",
            "hostname": "worker",
            "essential": false,
            "memory": 128
        },
        {
            "name": "nginx",
            "image": "rgederin/fib-nginx",
            "essential": true,
            "portMappings": [
                {
                    "hostPort": 80,
                    "containerPort": 80
                }
            ],
            "links": [
                "client",
                "server"
            ],
            "memory": 128
        }
    ]
}
```

As you could see there we did not mention Postrges and Redis because we will use AWS RDS and AWS Elastic Cache (managed services) respectively:

![aws1](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/aws1.png)

![aws2](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/aws2.png)

![aws3](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/aws3.png)

### AWS resources creation

At this point we need to create require infrastructure on AWS, we could do this using Management Console UI or with CloudFormation/Terraform/AWS CDK. Below is some guide how we could do this in Management Console.

**EBS Application Creation**

1. Go to AWS Management Console and use Find Services to search for Elastic Beanstalk
2. Click “Create Application”
3. Set Application Name 
4. Scroll down to Platform and select Docker
5. In Platform Branch, select Multi-Container Docker running on 64bit Amazon Linux
6. Click Create Application
7. You may need to refresh, but eventually, you should see a green checkmark underneath Health.

![eb1](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/eb1.png)


**RDS Database Creation**

1. Go to AWS Management Console and use Find Services to search for RDS
2. Click Create database button
3. Select PostgreSQL
4. In Templates, check the Free tier box.
5. Scroll down to Settings.
6. Set DB Instance identifier
7. Set Master Username
8. Set Master Password
9. Scroll down to Connectivity. Make sure VPC is set to Default VPC
10. Scroll down to Additional Configuration and click to unhide.
11. Set Initial database name
12. Scroll down and click Create Database button

![rds](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/rds.png)


**ElastiCache Redis Creation**

1. Go to AWS Management Console and use Find Services to search for ElastiCache
2. Click Redis in sidebar
3. Click the Create button
4. Make sure Cluster Mode Enabled is NOT ticked
5. In Redis Settings form, set Name
6. Change Node type to 'cache.t2.micro'
7. Change Replicas per Shard to 0
8. Scroll down and click Create button

![ec](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/ec.png)


**Creating a Custom Security Group**

1. Go to AWS Management Console and use Find Services to search for VPC
2. Find the Security section in the left sidebar and click Security Groups
3. Click Create Security Group button
4. Set Security group name to multi-docker
5. Set Description to multi-docker
6. Make sure VPC is set to default VPC
7. Click Create Button
8. Scroll down and click Inbound Rules
9. Click Edit Rules button
10. Click Add Rule
11. Click in the box next to Source and start typing 'sg' into the box. Select the Security Group you just created.
12. Click Create Security Group

![sg](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/sg.png)

When security group is created we need to apply it to RDS, EC and EB instances.

**Setting Environment Variables in Elastic Beanstalk**

1. Go to AWS Management Console and use Find Services to search for Elastic Beanstalk
2. Click Environments in the left sidebar.
3. Select your env
4. Click Configuration
5. In the Software row, click the Edit button
6. Scroll down to Environment properties
7. Set REDIS_HOST key to the primary endpoint, remember to omit :6379
8. Set REDIS_PORT to 6379
9. Set PGUSER to username which you specified in RDS
10. Set PGPASSWORD to password which you specified in RDS
11. Set the PGHOST key to the RDS endpoint.
12. Set PGDATABASE to database name which you specified in RDS
13. Set PGPORT to 5432
14. Click Apply button

![eb2](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/eb2.png)


After all instances restart and go from No Data, to Severe, you should see a green checkmark under Health.

### Travis setup for deploying to AWS

1. Go to your Travis Dashboard and find the project repository for the application we are working on.
2. On the repository page, click "More Options" and then "Settings"
3. Create an AWS_ACCESS_KEY variable and paste your IAM access key
4. Create an AWS_SECRET_KEY variable and paste your IAM secret key

After this we need to add additional config in `.travis.yml`:

```
deploy:
  provider: elasticbeanstalk
  region: "us-east-2"
  app: "fib-application-eb"
  env: "{your EB env name}"
  bucket_name: "{yourr EB S3 bucket name}"
  bucket_path: "fib-app"
  on:
    branch: master
  access_key_id: $AWS_ACCESS_KEY
  secret_access_key: $AWS_SECRET_KEY
```

And push all our changes in GitHub. After this you need to check your build in Travis CI and then check you EB application.

![eb3](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/eb3.png)

![eb4](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/eb4.png)