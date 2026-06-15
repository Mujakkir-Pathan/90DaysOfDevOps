# Day 30 – Docker Images & Container Lifecycle

## Objective

Understand the relationship between Docker images and containers, learn image layers and caching, and practice the complete container lifecycle.

---

# Task 1: Docker Images

### Screenshot 1: Docker Images List

![screnshot](screenshots/task1.1.png)

---

## Compare Ubuntu vs Alpine

### Why is Ubuntu much larger than Alpine?

Ubuntu is a full Linux distribution that contains more packages, libraries, and utilities. Alpine is a minimal Linux distribution designed specifically for containers and contains only essential components, making it significantly smaller.

### Screenshot 2: Image Inspection Output

![screnshot](screenshots/task1.3.png)

---

## What information can you see from image inspection?

* Image ID
* Creation Date
* Operating System
* Architecture
* Environment Variables
* Default Command
* Filesystem Layers

### Screenshot 3: Image Removal

![screnshot](screenshots/task1.4.png)

---

# Task 2: Image Layers

### Screenshot 4: Nginx Image History

![screnshot](screenshots/task2.png)

---

## What are layers and why does Docker use them?

Docker layers are read-only filesystem changes stacked together to form an image. Each instruction used during image creation generates a layer.

Docker uses layers to:

* Save storage space
* Reuse existing layers
* Speed up image builds
* Improve caching efficiency
* Reduce download times

### Why do some layers show 0B?

Layers such as ENV, CMD, ENTRYPOINT, LABEL, and EXPOSE only modify metadata and do not change the filesystem, so they appear as 0B.

---

# Task 3: Container Lifecycle

The following lifecycle operations were performed:

### 5: Created State
### 6: Running State
### 7: Paused State
### 8: Exited State
### 9: Removed Container

![screnshot](screenshots/task3.png)

---

## Observed Container States

Created → Running → Paused → Running → Exited → Running → Exited → Removed

---

# Task 4: Working with Running Containers

### Screenshot 10: Nginx Running in Detached Mode

![screnshot](screenshots/task4.1.png)

### Screenshot 11: Container Logs

![screnshot](screenshots/task4.2.png)
![screnshot](screenshots/task4.3.png)

### Screenshot 12: Container Filesystem

![screnshot](screenshots/task4.4.png)

### Screenshot 13: Single Command Execution Inside Container

![screnshot](screenshots/task4.5.png)

### Screenshot 14: Container Inspection

![screnshot](screenshots/task4.6.png)
![screnshot](screenshots/task4.7.png)

---

## Container Inspection Findings

### IP Address

172.17.0.2

### Port Mapping

80/tcp exposed inside the container.

No host port mapping was configured.

### Mounts

No mounts were attached to the container.

---

# Task 5: Cleanup

### Screenshot 15: Docker Cleanup Activities

![screnshot](screenshots/task5.1.png)
![screnshot](screenshots/task5.2.png)
![screnshot](screenshots/task5.3.png)

### Screenshot 16: Docker Disk Usage

![screnshot](screenshots/task5.4.png)

---
# Conclusion

Today I learned how Docker images and containers work, explored image layers and caching, practiced the complete container lifecycle, worked with running containers, inspected container details, and performed Docker cleanup operations.

