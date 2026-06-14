# Day 29 - Docker Basics

## Objective

Today's goal was to understand Docker, learn why containers exist, and run my first containers.

---

# Task 1: What is Docker?

## What is a Container and Why Do We Need It?

A container is a lightweight and isolated package that contains an application along with all its dependencies, libraries, and configuration files. Containers help ensure that an application runs the same way in different environments. They solve the "it works on my machine" problem and use fewer system resources compared to traditional virtual machines.

## Containers vs Virtual Machines

A Virtual Machine contains its own complete operating system and runs on top of a hypervisor. A container does not have its own operating system; instead, it shares the host operating system's kernel. Because of this, containers are much lighter and faster than virtual machines.

| Containers           | Virtual Machines      |
| -------------------- | --------------------- |
| Share host OS kernel | Have their own OS     |
| Lightweight          | Heavyweight           |
| Fast startup         | Slower startup        |
| Lower resource usage | Higher resource usage |
| More portable        | Less portable         |

## Docker Architecture

Docker architecture consists of the following components:

### Docker Client

The Docker Client is the interface through which users interact with Docker by running commands.

### Docker Daemon

The Docker Daemon runs in the background and manages images, containers, networks, and volumes.

### Docker Image

A Docker image is a blueprint that contains the application, dependencies, libraries, and configuration required to run a container.

### Docker Container

A Docker container is a running instance of a Docker image that executes the application in an isolated environment.

### Docker Registry

A Docker Registry stores Docker images. Docker Hub is the most commonly used public registry.

## Docker Architecture Flow

When I run a Docker command, the Docker Client sends the request to the Docker Daemon. The daemon checks whether the required image is available locally. If it is not available, Docker downloads the image from Docker Hub. After the image is downloaded, Docker creates and starts a container from that image.

```text
User
  |
  v
Docker Client
  |
  v
Docker Daemon
  |
  +----> Docker Hub (if image not available locally)
  |
  v
Docker Image
  |
  v
Docker Container
```

---

# Task 2: Install Docker

![here is screenshot](screenshots/task2.png)

---

# Task 3: Run Real Containers

## Run an Nginx container and access it in your browser

![here is screenshot](screenshots/task3.1.png) 

## Run Ubuntu Container in Interactive Mode

![here is screenshot](screenshots/task3.2.png) 


## List Running Containers

![here is screenshot](screenshots/task3.2.png) 

## List All Containers

![here is screenshot](screenshots/task3.3.png) 

## Stop a Container &  Remove a Container

![here is screenshot](screenshots/task3.4.png) 

---

# Task 4: Explore Docker Features

## Detached Mode

![here is screenshot](screenshots/task4.1.png) 

## Custom Container Name

![here is screenshot](screenshots/task4.2.png) 

## Port Mapping

![here is screenshot](screenshots/task4.3.png) 

## View Container Logs

![here is screenshot](screenshots/task4.4.png) 

## Execute Commands Inside a Running Container

![here is screenshot](screenshots/task4.5.png) 

---

# Key Learnings

* Docker containers package applications and dependencies together.
* Containers are lighter and faster than Virtual Machines.
* Docker Images are templates used to create containers.
* Docker Hub stores and distributes Docker images.
* Containers can run in interactive or detached mode.
* Port mapping allows services inside containers to be accessed externally.
* Docker logs help troubleshoot running containers.
* Docker exec allows commands to be executed inside running containers.

---

# Conclusion

Today I learned Docker fundamentals, understood Docker architecture, worked with Docker images and containers, ran real-world containers like Nginx and Ubuntu, explored container management, and gained hands-on experience with container lifecycle operations.

