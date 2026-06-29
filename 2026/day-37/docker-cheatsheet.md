#  Docker Cheat Sheet

##  Container Commands

| Command | Description |
|---------|-------------|
| `docker run -it ubuntu bash` | Run container in interactive mode |
| `docker run -d nginx` | Run container in detached mode |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers |
| `docker stop <container>` | Stop a container |
| `docker start <container>` | Start a stopped container |
| `docker restart <container>` | Restart a container |
| `docker rm <container>` | Remove a container |
| `docker exec -it <container> bash` | Open shell inside container |
| `docker logs <container>` | View container logs |
| `docker inspect <container>` | View detailed container information |

---

##  Image Commands

| Command | Description |
|---------|-------------|
| `docker images` | List images |
| `docker pull nginx` | Download image from Docker Hub |
| `docker build -t myapp .` | Build image from Dockerfile |
| `docker tag myapp username/myapp:v1` | Tag image |
| `docker push username/myapp:v1` | Push image to Docker Hub |
| `docker rmi <image>` | Remove image |
| `docker image prune` | Remove unused images |

---

##  Volume Commands

| Command | Description |
|---------|-------------|
| `docker volume create myvolume` | Create named volume |
| `docker volume ls` | List volumes |
| `docker volume inspect myvolume` | Inspect volume |
| `docker volume rm myvolume` | Remove volume |

---

##  Network Commands

| Command | Description |
|---------|-------------|
| `docker network create mynetwork` | Create custom network |
| `docker network ls` | List networks |
| `docker network inspect mynetwork` | Inspect network |
| `docker network connect mynetwork container` | Connect container to network |
| `docker network rm mynetwork` | Remove network |

---

##  Docker Compose Commands

| Command | Description |
|---------|-------------|
| `docker compose up` | Start services |
| `docker compose up -d` | Start in background |
| `docker compose down` | Stop and remove services |
| `docker compose ps` | List services |
| `docker compose logs` | View logs |
| `docker compose build` | Build images |
| `docker compose restart` | Restart services |

---

##  Cleanup Commands

| Command | Description |
|---------|-------------|
| `docker system df` | Show Docker disk usage |
| `docker system prune` | Remove unused resources |
| `docker system prune -a` | Remove all unused images & containers |
| `docker volume prune` | Remove unused volumes |
| `docker network prune` | Remove unused networks |

---

#  Dockerfile Instructions

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image |
| `RUN` | Execute commands during build |
| `COPY` | Copy local files into image |
| `ADD` | Copy files (also supports URLs & archives) |
| `WORKDIR` | Set working directory |
| `EXPOSE` | Document application port |
| `ENV` | Set environment variables |
| `CMD` | Default command |
| `ENTRYPOINT` | Main executable |
| `LABEL` | Add metadata |
| `ARG` | Build-time variables |

---

#  Useful Docker Run Options

| Option | Meaning |
|---------|---------|
| `-d` | Detached mode |
| `-it` | Interactive terminal |
| `-p 8080:80` | Port mapping |
| `-v` | Mount volume |
| `--name` | Assign container name |
| `--network` | Connect to network |
| `--env` | Set environment variable |
| `--rm` | Remove container after exit |
