#  Day 37 – Docker Revision

##  Self Assessment

| Topic | Status |
|--------|--------|
| Run container (interactive + detached) | ✅ Can Do |
| List, stop & remove containers/images | ✅ Can Do |
| Image layers & caching | ✅ Can Do |
| Write Dockerfile from scratch | ✅ Can Do |
| CMD vs ENTRYPOINT | ✅ Can Do |
| Build & tag image | ✅ Can Do |
| Named volumes | ✅ Can Do |
| Bind mounts | ✅ Can Do |
| Custom networks | ✅ Can Do |
| Docker Compose | ✅ Can Do |
| Environment variables & .env | ✅ Can Do |
| Multi-stage Dockerfile | ✅ Can Do |
| Push image to Docker Hub | ✅ Can Do |
| Healthchecks & depends_on | ✅ Can Do |

---

# Quick-Fire Questions

## 1. Difference between an Image and a Container?

**Image:** Read-only blueprint containing application code and dependencies.

**Container:** Running instance of an image.

---

## 2. What happens to data inside a container when you remove it?

Container data is deleted unless stored in a **named volume** or **bind mount**.

---

## 3. How do two containers on the same custom network communicate?

Using their **container names** as hostnames.

Example:

```bash
ping database
```

---

## 4. What does `docker compose down -v` do?

Stops containers, removes containers, networks **and named volumes**.

`docker compose down` keeps the volumes.

---

## 5. Why are multi-stage builds useful?

- Smaller images
- Better security
- Faster deployments
- Remove unnecessary build tools

---

## 6. Difference between COPY and ADD?

**COPY**
- Copies local files only.

**ADD**
- Can extract tar archives.
- Can download files from URLs.

COPY is preferred unless ADD features are needed.

---

## 7. What does `-p 8080:80` mean?

Maps:

Host Port → Container Port

```
localhost:8080
      ↓
Container:80
```

---

## 8. How do you check Docker disk usage?

```bash
docker system df
```

---

# Weak Spots Practice

## Topic 1

**Docker Compose**

Recreated a multi-container application using:

- PostgreSQL
- Node.js
- Docker Compose
- .env file
- Named volumes
- Custom network

---

## Topic 2

**Multi-stage Docker Build**

Built a production-ready Node.js image using:

- Builder stage
- Runtime stage
- Smaller final image

---

# Revision Summary

Today I revised the complete Docker workflow including:

- Containers
- Images
- Dockerfile
- Volumes
- Bind Mounts
- Networks
- Docker Compose
- Environment Variables
- Multi-stage Builds
- Docker Hub
- Healthchecks
- Cleanup Commands

Docker fundamentals are now well understood and ready for Kubernetes deployment.
