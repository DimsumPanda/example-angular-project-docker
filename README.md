# Example Angular Project - Docker Example
This is the first part introduction for the example Jenkins CI/CD process.

This README is used to introduce teams to Docker, containerization, multi-stage builds, and docker-compose to our appdev teams. 
The multi-stage angular project mostly references the project in this [blog post](https://dev.to/avatsaev/create-efficient-angular-docker-images-with-multi-stage-builds-1f3n); however, additional details are added regarding docker commands/components, setting container environment variables, and leveraging docker-compose.

1. Example Angular Project - Docker Example (You are here)
2. [Example Angular Project - Create Deployment Objects with Terraform and Docker-Compose]()
3. [Example Angular Project - Build with Jenkinsfile]()
4. [Example Angular Project - CNAME DNS switch]()


## Table of Contents
1. [Create Dockerfile](#create-your-angular-dockerfile)
2. [Create a Docker Compose File](#create-a-docker-compose-file)

## <a name="create-your-angular-dockerfile">Create your Angular Dockerfile</a>
The following Angular Docker tutorial steps references the multistage build in this [blog post](https://dev.to/avatsaev/create-efficient-angular-docker-images-with-multi-stage-builds-1f3n).

Let's dockerize an Angular app with Docker's Multi-Stage Builds.

### Prerequisites
* NodeJS +8
* Angular CLI (npm i -g @angular/cli@latest)
* Docker +17.05
* Basic understanding of Docker and Angular CLI commands

### The Plan
To dockerize a basic Angular app built with Angular CLI, we need to do the following:

* npm install the dependencies (dev dependencies included)
* ng build with --prod flag
* move the artifacts from dist folder to a publicly accessible folder (via an an nginx server)
* Setup an nginx config file, and spin up the http server

**We'll do this in 2 stages:**
* Build stage: will depend on a Node alpine Docker image
* Setup stage: will depend on NGINX alpine Docker image and use the artifacts from the build stage, and the nginx config from our project.

### Installing dependencies ###
The original linked tutorial assumes you have npm & ng installed already.
If you do not, run the following commands. These instructions are for macOS with [homebrew](https://brew.sh/) installed.  
`brew install npm`  
`npm link @angular/cli` # fix ng path  

more details can be found [here](https://stackoverflow.com/questions/37227794/ng-command-not-found-while-creating-new-project-using-angular-cli)

### Initialize an empty Angular project
`$ ng new myapp`  
? Would you like to add Angular routing? No  
? Which stylesheet format would you like to use? CSS  


### Add a default nginx config
At the root of your Angular project, create nginx folder and create a file named default.conf with the following contents (./nginx/default.conf):
```
server {

  listen 80;

  sendfile on;

  default_type application/octet-stream;


  gzip on;
  gzip_http_version 1.1;
  gzip_disable      "MSIE [1-6]\.";
  gzip_min_length   1100;
  gzip_vary         on;
  gzip_proxied      expired no-cache no-store private auth;
  gzip_types        text/plain text/css application/json application/javascript application/x-javascript text/xml application/xml application/xml+rss text/javascript;
  gzip_comp_level   9;


  root /usr/share/nginx/html;


  location / {
    try_files $uri $uri/ /index.html =404;
  }

}
```
### Create the Docker file:
Save the following in a file called Dockerfile. There are two stages to this Dockerfile, only the second stage gets baked into the image that we will eventually push to our Container Registry.

First create Dockerfile inside myapp: 
`cd myapp`
`touch Dockerfile`

Save the following into your Dockerfile:
```

### STAGE 1: Build ###

# We label our stage as ‘builder’
FROM node:10-alpine as builder

COPY package.json package-lock.json ./

## Storing node modules on a separate layer will prevent unnecessary npm installs at each build

RUN npm ci && mkdir /ng-app && mv ./node_modules ./ng-app

WORKDIR /ng-app

COPY . .

## Build the angular app in production mode and store the artifacts in dist folder

RUN npm run ng build -- --prod --output-path=dist


### STAGE 2: Setup ###

FROM nginx:1.14.1-alpine

## Copy our default nginx config
COPY nginx/default.conf /etc/nginx/conf.d/

## Remove default nginx website
RUN rm -rf /usr/share/nginx/html/*

## From ‘builder’ stage copy over the artifacts in dist folder to default nginx public folder
COPY --from=builder /ng-app/dist /usr/share/nginx/html

CMD ["nginx", "-g", "daemon off;"]
```
### Build the image
docker build -t {image_tag} {path of Dockerfile}

`$ docker build -t myapp .`

### Run the container
`$ docker run -p 8080:80 myapp`

And done, your dockerized app will be accessible at http://localhost:8080

And the size of the image is only ~15.8MB, which will be even less once pushed to a Docker repository.

## Docker Clean Up
Make sure you keep your docker workspace clean as you develop locally.
At this time, please delete the container and images you made in the previous step.
1. List the Containers
`docker ps -a`
2. Make sure the container running myapp has been stopped. If it is still running, then use the following command:
`docker stop {contaiiner_ID}`
3. Delete the stopped container
```
$ docker rm {container_id}
```
4. List the Docker Images
`docker images`
5. Remove the Docker image with the myapp tag using the image ID
`docker rmi {image_ID}`

## <a name="create-a-docker-compose-file">Create a Docker Compose File</a>
Now that you have a Dockerfile, you are ready to create a docker image using a Docker Compose file!
If you had created an image previiously, make sure to remove the image.

Run `$ docker images`
Make sure the myapp image has been removed from the machine:
`$ docker rmi <your image id>`

### The docker-compose file
The docker-compose file is the equivalent of the docker run command. It makes it easier to pass in parameters and make sure exactly how a container is run and what image it is using.

For this section, I am referencing the following [blog post](https://medium.com/joolsoftware/how-to-set-up-an-angular-cli-project-with-docker-compose-a3ec78f179ab).

"A Docker file is enough to build, but we want to be able to start multiple containers with one file. For example, if we also want to add our Node Express back-end application and our database which all contain their own Dockerfile, we could manage them within a single configuration.
This is where we can use the docker-compose tool.
Docker compose is a tool which reads a docker-compose YAML file to start your application services."

## Create a Docker Compose file
Create a docker compose file outside of the myapp project you created previously.
```
cd ../
touch docker-compose.yml
```
A Docker compose file starts of with the version of the configuration file that we are going to use.
`version: '2.0'`

Below the version we start defining our services. Our service will be called angular-service.
Here we also need to define the container name we generate with the CONTAINER_NAME instruction and the build location of our Dockerfile with the BUILD instruction.
Next we map our volumes from our local directory to the directory that we make in our environment with the VOLUMES instruction.
Then we specify on which port we want to run our application and what port we have exposed in our Dockerfile with the PORTS instruction.

```
version: '2.0' # version syntax
services: # Here we define our service(s)
    angular-service: # The name of the service
      container_name: myapp  # Container name
      build: ./myapp
      ports: 
        - '8080:80' # Port mapping
      environment:
        - "EXAMPLE1=${EXAMPLE1}"
```

Instead of "build: {path}", you can also replace this with "image: {Container_registry}/{image_name}:{version}"
You can also add environment variables to the container by specifying the environment parameter.

The COMMAND instruction is a way to instruct Docker to run npm install and ng serve. This is one way to do it, I will use this example for now but there are other ways to install the node packages and run the application.
The following command opens bash in the running container:
`$ docker exec -i -t {container_id} /bin/bash` 
Try /bin/sh if /bin/bash is not available.
In this container you could also install the node packages and serve the application.

### Run docker compose
First create the environment variable locally. 
For Linux,
`$ export EXAMPLE1="Let's do this thing"`

Run this where the docker-compose file is located in.
`docker-compose up --build -d`

You should be able to view it at http://localhost:8080

List the docker containers:

`docker ps`

Bash into the container:

`$ docker exec -i -t {container_id} /bin/sh`

Inside the container, check that the environment variable appears inside the container.

`echo $EXAMPLE1`

You should see the output "Let's do this thing" in the container.
Exit the container:
`exit`

Check the logs in the container:
`$ docker logs {container_id}`

Stop the container:
`$ docker stop {container_id}`

Remove the container:
`$ docker rm {container_id}`

Remove the image by listing the images and removing it by the image_id:
`$ docker images`

`$ docker rmi {image_id}`

Continue to the next part in the CCS CI/CD Angular App Tutorial: [Example Angular Project - Create Deployment Objects with Terraform and Docker-Compose]()

