# docker-basics


Whats the difference between docker image and container



An image is a snapshot instance of the an app and the environment it needs to run in i.e the application code and all the properties of the VM (OS, networking, other required installed applications). While a container is an instance of an image.

You can create multiple instances of the same image and run them at the same time. 



DockerFile
Docker file is a list of execution steps that docker will use to build an image. Images can be build on top of each other. You can have a base image with all the environment configuration and use that to build a set of application images on top.



```

FROM node:8.11.4-alpine

RUN apk update && apk add bash

# Install the application under /app/
COPY node_modules /app/node_modules
COPY dist /app/dist
COPY config /app/config
COPY package.json /app

# Run the app
EXPOSE  10010
WORKDIR /app
CMD ["node", "./dist/server/index.js"]
```


In the above code block

the base image to build this new image is defined in 'FROM'
Run a list of execution steps, in the above block, its just copying a bunch of files to app dir
EXPOSE instruction informs Docker that the container listens on the specified network ports at runtime
The CMD instruction has three forms:

* CMD ["executable", "param1", "param2"] (exec form, this is the preferred form)

* CMD ["param1", "param2"] (as default parameters to ENTRYPOINT)

* CMD command param1 param2 (shell form)

> https://docs.docker.com/engine/reference/builder/



## Build an image
To build an image using the DockerFile

`docker build -f /path/to/a/Dockerfile .`
You can specify a repository and tag at which to save the new image if the build succeeds:

`docker build -t shykes/myapp .`

To tag the image into multiple repositories after the build, add multiple -t parameters when you run the build command:

`docker build -t shykes/myapp:1.0.2 -t shykes/myapp:latest .`


Before the Docker daemon runs the instructions in the Dockerfile, it performs a preliminary validation of the Dockerfile and returns an error if the syntax is incorrect:

```
docker build -t test/myapp .

Sending build context to Docker daemon 2.048 kB
Error response from daemon: Unknown instruction: RUNCMD
```

## Run a container using an image
There are two ways to create/start a container

### Command line:
```
docker run <ImageId>

-d : run the container in detach mode (run the container in as a seperate process and free up the terminal)

-v: verbose

-p: port <exposed-port>/<internal> (80:10010)
```

### Docker-compose
Docker compose is almost like a list of recipes to define the essentials needed to create/start a container. Similar to defining the variables in command line, we can define all the configuration in the docker-compose file.

```
service-service-name:
	image: ${DOCKER_PREFIX}/services/<image_name>:latest
	container_name: service-<service-name>
	environment:
		LOG_LEVEL: info
		LOG_EXPRESSION: "service.*"
		IAM_URL: http://service-iam:10010
	networks:
		- network-name
	volumes:
      - <host_volume_space>:<container_volume_space>
    ports:
      - <exposed_port>:<internal_port>
	logging:
		driver: "json-file"
		options:
			max-size: "2m"
			max-file: "2"
```


The above code is using an image to build a container. But, we can also use DockerFile to specify the image to build the container from


```
service-service-name:
	build:
		context: .
		dockerfile: ./DockerFile
	container_name: service-<service-name>
	environment:
		LOG_LEVEL: info
		LOG_EXPRESSION: "service.*"
		IAM_URL: http://service-iam:10010
	networks:
		- network-name
	volumes:
      - <host_volume_space>:<container_volume_space>
    ports:
      - <exposed_port>:<internal_port>
	logging:
		driver: "json-file"
		options:
			max-size: "2m"
			max-file: "2"
```


To create/run the container, the command will be:

`docker-compose up -d service-service-name`


## Connectivity between containers
If the containers are in the same network, you can use the service names to establish connection over tcp because docker keeps an internal ip table with service name. So, for example in the code block below, service 1 is connecting to service 2 over http using just the service-2 container name and the port its running on. Also, there's a new property in service-1 "depends_on" telling docker that it needs to create service 2 before service 1

```
service-1:
	build:
		context: .
		dockerfile: ./DockerFile
	container_name: service-1
	depends_on: service-2
	environment:
		LOG_LEVEL: info
		LOG_EXPRESSION: "service.*"
		SERVICE_2: http://service-2:10010
	networks:
		- network-name
	volumes:
      - <host_volume_space>:<container_volume_space>
    ports:
      - <exposed_port>:<internal_port>
	logging:
		driver: "json-file"
		options:
			max-size: "2m"
			max-file: "2"
service-2:
	image: ${DOCKER_PREFIX}/services/<image_name>:latest
	container_name: service-2
	networks:
		- network-name
	volumes:
      - <host_volume_space>:<container_volume_space>
	logging:
		driver: "json-file"
		options:
			max-size: "2m"
			max-file: "2"
```


### Get a container to connect to a service running on localhost (hack, dont do this unless you have to, and definitely not for prod)

all containers are isolated, the network setting in place is by default `bridge`. docker has a way of connecting container running on the same network (docker network, not the host network). that’s why we use the container names with internal ports to connect. so, when trying to access a service running on localhost for example `http://service2:10010/...`
would resolve to `127.0.0.1` which is not part of docker’s internal ip list. So, you get a connection refuse error

the fix (At least for mac):
instead of hostname use `http://host.docker.internal`
i.e
instead of
`SERVICE_2: http://service-2:10010`
use:
`SERVICE_2: http://host.docker.internal:10010`



https://docs.docker.com/docker-for-mac/networking/



https://docs.docker.com/docker-for-mac/networking/



## Handy commands

`https://docs.docker.com/engine/reference/commandline/docker/`



### List all the containers
`docker ps -a`


### Stop a running container 
`docker stop <container-id>`


### Stop a container using docker-compose
`docker-compose stop <service-name>`


### Start a container
`docker start <container-id>`


### Remove the container
`docker rm <container-id>`


### Run a command in a running container
```
docker exec <container-id> <command>
-i: interactive terminal
-t: tty

docker exec -it <container-id> "sh -c date"
```


### Run a command in a non running container (hack)
`list the images, and create a new container off the image and run the command`

```
docker images

docker run -it <image-id> sh -c "date"
```


### Get logs from container
```
docker logs <container-id>

-f: tail the logs


Take down all the container listed in docker-compose file
docker-compose down

--remove-orphan: Remove containers for services not defined in the Compose file
-v, --volumes: Remove named volumes declared in the `volumes` section of the Compose file and anonymous volumes attached to containers.
```

### List all the images
`docker images`


### Remove an image
```
docker rmi <image-id>

-f: force
```

### Update the version of image
```
docker pull <image_name>:<tag>


docker-componse pull <container_name>
```

### create a docker table

`docker ps -a --format "table {{.Names}}\t{{.ID}}\t{{.Status}}\t{{.Command}}\t{{.Ports}}"`




