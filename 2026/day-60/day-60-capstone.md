# Day 60 – Capstone: Deploy WordPress + MySQL on Kubernetes

## Task

Completed the Kubernetes capstone by deploying a real WordPress + MySQL application and bringing together the major concepts learned throughout the previous ten days: Pods, Deployments, Services, ConfigMaps, Secrets, storage, StatefulSets, resource management, autoscaling, and Helm.

---

## Challenge Tasks

### Task 1: Create the Namespace (Day 52)

1. Created the `capstone` namespace.
2. Set `capstone` as the default namespace using:

```bash
kubectl config set-context --current --namespace=capstone
```

---

### Task 2: Deploy MySQL (Days 54-56)

1. Created the MySQL Secret with `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, and `MYSQL_PASSWORD` using `stringData`.
2. Created a Headless Service for MySQL with `clusterIP: None` on port `3306`.
3. Created a MySQL StatefulSet with:
   - Image: `mysql:8.0`
   - `envFrom` referencing the Secret
   - Resource requests: `cpu: 250m`, `memory: 512Mi`
   - Resource limits: `cpu: 500m`, `memory: 1Gi`
   - A `volumeClaimTemplates` section requesting `1Gi`
   - The volume mounted at `/var/lib/mysql`
4. Verified MySQL using:

```bash
kubectl exec -it mysql-0 -- mysql -u wordpress -p... -e "SHOW DATABASES;"
```

**Verify:** Can you see the `wordpress` database?

**Yes.** The `wordpress` database was successfully listed in MySQL.

---

### Task 3: Deploy WordPress (Days 52, 54, 57)

1. Created a ConfigMap with `WORDPRESS_DB_HOST` set to `mysql-0.mysql.capstone.svc.cluster.local:3306` and `WORDPRESS_DB_NAME`.
2. Created a Deployment with 2 replicas using `wordpress:latest` that:
   - Used `envFrom` for the ConfigMap.
   - Used `secretKeyRef` for `WORDPRESS_DB_USER` and `WORDPRESS_DB_PASSWORD` from the MySQL Secret.
   - Configured resource requests and limits.
   - Configured liveness and readiness probes on `/wp-login.php` port `80`.
3. Verified both WordPress Pods were `1/1 Running`.

**Verify:** Are both WordPress pods running and ready?

**Yes.** Both WordPress Pods were `1/1 Running` and ready.

---

### Task 4: Expose WordPress (Day 53)

1. Created a NodePort Service targeting the WordPress Pods on port `30080`.
2. Accessed WordPress using the Kind port-forward:

```bash
kubectl port-forward svc/wordpress 8080:80 -n capstone
```

3. Completed the WordPress setup wizard and created a blog post.

**Verify:** Can you see the WordPress setup page?

**Yes.** The WordPress setup page was accessed successfully, the setup was completed, and a blog post was created.

---

### Task 5: Test Self-Healing and Persistence

1. Deleted one WordPress Pod and watched the Deployment recreate it. The replacement Pod became `1/1 Running`, and the Deployment returned to `2/2`.
2. Deleted the MySQL Pod:

The StatefulSet recreated `mysql-0`, which returned to `1/1 Running`.
3. Refreshed WordPress after MySQL recovered and verified that the existing blog post was still present.

**Verify:** After deleting both pods, is your blog post still there?

**Yes.** The blog post remained available after both the WordPress Pod and MySQL Pod were deleted and recreated, confirming persistent database storage through the MySQL PVC.

---

### Task 6: Set Up HPA (Day 58)

1. Created an HPA targeting the WordPress Deployment with:
   - CPU target: `50%`
   - Minimum replicas: `2`
   - Maximum replicas: `10`
2. Verified the HPA after metrics became available:
3. Ran:

```bash
kubectl get all -n capstone
```

**Verify:** Does the HPA show correct min/max and target?

**Yes.** The HPA showed a `50%` CPU target, minimum `2` replicas, maximum `10` replicas, and `2` current replicas.

---

### Task 7: (Bonus) Compare with Helm (Day 59)

1. Installed WordPress using the Bitnami Helm chart in a separate `helm-test` namespace:

```bash
helm install wp-helm bitnami/wordpress -n helm-test
```

2. Compared the Helm deployment with the manually created Kubernetes deployment.

The manual approach provided more granular control because each resource and configuration was explicitly defined. Helm provided a faster and more reusable deployment through a packaged chart.

3. Cleaned up the Helm deployment:

Verified that no application resources remained in `helm-test`.

---

### Task 8: Clean Up and Reflect

1. Took the final look with:

```bash
kubectl get all -n capstone
```

2. Used these twelve concepts in one deployment:

| Concept Used |  
|---|
| Namespace |
| Secret |
| ConfigMap |
| PVC |
| StatefulSet |
| Headless Service | 
| Deployment | 
| NodePort Service | 
| Resource Limits | 
| Probes |
| HPA | 
| Helm |
3. Deleted the `capstone` namespace:

```bash
kubectl delete namespace capstone
```

4. Reset the Kubernetes context to the `default` namespace:

```bash
kubectl config set-context --current --namespace=default
```
**Verify:** Did deleting the namespace remove everything?

**Yes.** After deleting the `capstone` namespace, `kubectl get all -n capstone` returned:

No resources found in capstone namespace.


The Kubernetes context was also successfully reset to `default`.

---

## Architecture of the Deployment

                    ┌──────────────────────────┐
                    │   capstone Namespace     │
                    │                          │
                    │  ┌────────────────────┐  │
                    │  │ WordPress Service   │  │
                    │  │ NodePort :30080     │  │
                    │  └─────────┬──────────┘  │
                    │            │              │
                    │     ┌──────┴──────┐       │
                    │     │             │       │
                    │  WordPress     WordPress  │
                    │    Pod           Pod      │
                    │     │             │       │
                    │     └──────┬──────┘       │
                    │            │              │
                    │      Deployment           │
                    │        (2 replicas)       │
                    │            │              │
                    │            ▼              │
                    │   MySQL Headless Service  │
                    │        :3306              │
                    │            │              │
                    │            ▼              │
                    │        mysql-0             │
                    │     MySQL StatefulSet      │
                    │            │              │
                    │            ▼              │
                    │       MySQL PVC            │
                    │         1Gi                │
                    │                          │
                    │  ConfigMap ──► WordPress │
                    │  Secret ─────► MySQL + WP│
                    │  HPA ────────► Deployment│
                    └──────────────────────────┘

## Self-Healing and Persistence Results

| Test | Result |
| --- | --- |
| Deleted a WordPress Pod | Deployment recreated the Pod automatically |
| WordPress replicas after recovery | `2/2 Ready` |
| Deleted `mysql-0` | StatefulSet recreated `mysql-0` |
| MySQL after recovery | `1/1 Running` |
| WordPress blog post after MySQL recovery | Still present |
| Persistence result | MySQL data survived Pod deletion through the PVC |

## Concepts Mapped to Learning Days

| Concept | Learned On |
| --- | --- |
| Namespace | Day 52 |
| Deployment | Day 52 |
| Services | Day 53 |
| ConfigMap | Day 54 |
| Secrets | Day 54 |
| Persistent Volumes / PVC | Day 55 |
| StatefulSet | Day 56 |
| Resource Requests & Limits | Day 57 |
| Liveness & Readiness Probes | Day 57 |
| HPA | Day 58 |
| Helm | Day 59 |
| WordPress + MySQL Capstone | Day 60 |

## Reflection

### What Was Hardest?

The hardest part was connecting all the Kubernetes concepts together, especially understanding how WordPress communicated with MySQL through the StatefulSet's DNS name, Headless Service, Secret, and ConfigMap.

### What Clicked?

The biggest thing that clicked was how Kubernetes resources work together rather than independently. The Deployment handled WordPress replicas and self-healing, the StatefulSet handled MySQL identity and storage, the Service provided networking, the PVC preserved data, and the HPA controlled WordPress scaling.

### What Would I Add for Production?

For a production deployment, I would add:

- TLS and Ingress
- Stronger Secret management, such as an external secrets manager
- Proper MySQL backup and restore procedures
- Monitoring and alerting
- PodDisruptionBudgets
- Security contexts
- Network policies
- Non-root containers
- Resource tuning
- A production-grade managed database instead of running MySQL directly in the cluster

---
