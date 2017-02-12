Lesson 1: Set up a web service
==============================

### 1: Create a file called `docker-compose.yml` in the root of your project.

This file will contain all information about the services, networks and volumes that will make up your development environment stack.  As we progress with this project, we will modify this file to modify services, define additional ones, add volumes and configure each volume and service to work together to build a viable development stack for our project.

### 2.  Open the file in your favorite IDE or editor

### 3.  Add the following line to the top of the file:

`version: '2'`

This key designates which version of docker-compose file format that our `.yml` file.  Backwards compatibility has been maintained in docker-compose currently so that earlier versions will work with the current version of Docker Compose, but each version incorporates new features of Docker that weren't available in previous versions.  While backwards compatibility with the file format has been maintained, not all formats work with all versions of Docker; the various `version` key values are tied to specific Docker versions.

If you see a `docker-compose.yml` file without a version key, it is a `version 1` docker-compose file.  These files are considered deprecated, and support for `version ` docker-compose.yml files will be removed eventually.  `Version 2` is the current recommended format.  It requires that you are running Docker Engine version 1.10.0 or greater.  To designate a `docker-compose.yml` file as compliant with `Version 2`, you must explicitely include this line in your `.yml` file.

### 4.  Next add a `services` key below the `version` key

Add the following line below the `version` key in your `docker-compose.yml` file:

~~~
services:
~~~

A Docker service is an instance of a Docker image that is used in your stack for a specific purpose.  Each service will provide a specific, isolated application in the overall configuration, and will be granted permission through configuration settings included in your `docker-compose.yml` file to interact with other service containers in the stack.  This concept is referred to as `application containerization`, and is a operating system level virtualization method for deploying and running distributed applications without launching an entire virtualization environment such as a virtual machine.  Each container houses all the components such as files, environment variables and libraries necessary to run the its application.

### 5.  Next add a `web` service key below the `services` key, and define your web service as follows:

~~~
  web:
    image: nginx:latest
    ports:
        - 8000:80
~~~

The `image` key designates which docker image to use to launch this particular service (in this case the web server, which is designated as the `web` service).

The `ports` key configures the mapping of `external` (host) ports to `internal` (container) ports for this service.  The `nginx` container's Dockerfile exposes an internal port, port 80, to the docker network layer and services running in othr containers on the Docker network.  On the host operating system, we can control which ports our host operating system uses to access the service container's services by mapping one of the available ports on our host system to the internally exposed port on the service container.  We will look at the structure of a Dockerfile a bit later.

Our configuration is mapping port `8000` on our host system to the internally exposed port `80` on our NginX container.

### 6.  Next, let's launch the services we've defined in our stack with Docker Compose.

Execute the following command:

`docker-compose up -d`

`docker-compose up` builds, creates, starts, and attaches to containers for a service, in this case the service we've defined as our `web` service.  If we have linked to other services (we'll cover this later), this command will also start those services.

By default, `docker-compose up` runs a service interactively, and when the command exits, the services that were launched with that command are terminated.  For a service such as a web service that needs to persist, this is not a desired behavior.  Because of that, there is a parameter, `-d` that can be passed when executing this command that will launch the services defined in your `docker-compose.yml` file as background services, and those services that need to persist (such as our `web` service) after execution of the `docker-compose` command ends will continue to run in the background.

### 7.  Finally, let's see if our web service is running an NginX web server.

Through our port mapping, we can access this service from our host environment on port `8000`.  Launch a web browser, navigate to `http://localhost:8000`, and you should see the default NginX welcome page:

![Welcome to NginX](https://github.com/lhridley/dockerdrop/raw/01-add-nginx/docs/nginx-welcome.png "Welcome to NginX")

At this point, your `docker-compose.yml` should look like the following:

~~~
version: '2'
services:
  web:
    image: nginx:latest
    ports:
        - 8000:80
~~~
