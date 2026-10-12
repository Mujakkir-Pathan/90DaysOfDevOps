# Day 36 - Dockerize a Real Application

## Objective

Dockerize a real-world application by creating Dockerfiles, Docker Compose configuration, and deploying Docker images to Docker Hub.

## Project

**DevBoard** - A task management application with:

* React (Frontend)
* Go + Gin (Backend)
* PostgreSQL (Database)

## Tasks Completed

###  Task 1: Picked an Application

* Used the DevBoard project.
* Removed the existing Docker configuration and created my own.

[https://github.com/Mujakkir-Pathan/devboard](https://github.com/Mujakkir-Pathan/devboard)

###  Task 2: Dockerfile

* Created a Dockerfile for the frontend.
* Created a Dockerfile for the backend.
* Used lightweight base images.
* Added `.dockerignore` files.
* Built and tested the Docker images.

[https://github.com/Mujakkir-Pathan/devboard/blob/main/frontend/Dockerfile](https://github.com/Mujakkir-Pathan/devboard/blob/main/frontend/Dockerfile)

[https://github.com/Mujakkir-Pathan/devboard/blob/main/backend/Dockerfile](https://github.com/Mujakkir-Pathan/devboard/blob/main/backend/Dockerfile)

###  Task 3: Docker Compose

Created a `docker-compose.yml` with:

* Frontend service
* Backend service
* PostgreSQL service
* Persistent volume for PostgreSQL
* Custom Docker network
* Environment variables using `.env`
* Database healthcheck

[https://github.com/Mujakkir-Pathan/devboard/blob/main/docker-compose.yml](https://github.com/Mujakkir-Pathan/devboard/blob/main/docker-compose.yml)

###  Task 4: Ship It

* Tagged Docker images.
* Pushed images to Docker Hub.
* Added project documentation in `README.md`.

[https://hub.docker.com/repository/docker/mujakkirpathan/devboard-frontend/general](https://hub.docker.com/repository/docker/mujakkirpathan/devboard-frontend/general)

[https://hub.docker.com/repository/docker/mujakkirpathan/devboard-backend/general](https://hub.docker.com/repository/docker/mujakkirpathan/devboard-backend/general)

###  Task 5: Test

* Removed local containers and images.
* Pulled images from Docker Hub.
* Verified the application works using Docker Compose.

### Done

## Files Created

* `frontend/Dockerfile`
* `backend/Dockerfile`
* `docker-compose.yml`
* `.env`
* `.dockerignore`
* `README.md`
* `day-36-docker-project.md`

[here is the recordind](recording/rec.mp4)

## Outcome

Successfully Dockerized a complete multi-container application and deployed the images to Docker Hub.

