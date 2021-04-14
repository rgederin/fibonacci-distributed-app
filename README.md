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

Data which will be displayed in `Indecies I have seen` and in `Calculated falues` will come from Postgres and Redis respectively:

![app-with-be](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/app-with-be.png)


## Application workflow

Overal application workflow is described below:

![workflow](https://github.com/rgederin/fibonacci-distributed-app/blob/master/img/workflow.png)


## Development environment architecture 

Development and production environment will slightly differ. Development environment will consist of next components (running with docker):

* Nginx
* React server
* Express Server
* Worker
* Redis 
* Postgres
