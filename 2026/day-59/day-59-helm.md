# **Day 59 – Helm — Kubernetes Package Manager**

## **Task**

Completed the Helm task by installing Helm, deploying and customizing a Bitnami chart, performing an upgrade and rollback, and creating and installing a custom Helm chart.

---
 
## All Screenshots are here 
![Proof of work](screenshots/)
 
---

## **Challenge Tasks**

### **Task 1: Install Helm**

1. Installed Helm using the appropriate installation method for the environment.
2. Verified the installation with `helm version` and `helm env`.

Three core concepts:

- **Chart** — a package of Kubernetes manifest templates
- **Release** — a specific installation of a chart in the cluster
- **Repository** — a collection of charts (like a package repo)

---

### **Task 2: Add a Repository and Search**

1. Added the Bitnami repository:
   `helm repo add bitnami https://charts.bitnami.com/bitnami`
2. Updated the repository with `helm repo update`.
3. Searched the repository using `helm search repo nginx` and `helm search repo bitnami`.

---

### **Task 3: Install a Chart**

1. Deployed nginx using `helm install my-nginx bitnami/nginx`.
2. Checked the created Kubernetes resources with `kubectl get all`.
3. Inspected the release using `helm list`, `helm status my-nginx`, and `helm get manifest my-nginx`.

One Helm command replaced manually writing the Deployment, Service, and ConfigMap resources.

---

### **Task 4: Customize with Values**

1. Viewed the default chart values using `helm show values bitnami/nginx`.
2. Installed a custom release with `--set replicaCount=3 --set service.type=NodePort`.
3. Created `custom-values.yaml` with replica count, Service type, and resource limits.
4. Installed another release using `-f custom-values.yaml`.
5. Checked the applied overrides using `helm get values <release-name>`.

**Verify:** Does the values file release have the correct replicas and service type?

Yes. The values file release had **3 replicas** and the Service type was set to **NodePort**.

---

### **Task 5: Upgrade and Rollback**

1. Upgraded `my-nginx` using `helm upgrade my-nginx bitnami/nginx --set replicaCount=5`.
2. Checked the release history using `helm history my-nginx`.
3. Rolled back the release using `helm rollback my-nginx 1`.
4. Verified that the rollback created a new revision instead of overwriting the previous revision.

The rollback created revision **3**, while revision **2** remained in the release history.

**Verify:** How many revisions after the rollback?

There were **3 revisions** after the rollback.

---

### **Task 6: Create Your Own Chart**

1. Scaffolded a custom chart using `helm create my-app`.
2. Explored `Chart.yaml`, `values.yaml`, and `templates/deployment.yaml`.
3. Reviewed Go template syntax such as `{{ .Values.replicaCount }}` and `{{ .Chart.Name }}`.
4. Updated `values.yaml` to use **3 replicas** and the `nginx:1.25` image.
5. Validated the chart using `helm lint my-app`.
6. Previewed the generated manifests using `helm template my-release ./my-app`.
7. Installed the chart using `helm install my-release ./my-app`.
8. Upgraded the release using `helm upgrade my-release ./my-app --set replicaCount=5`.

**Verify:** After installing, 3 replicas? After upgrading, 5?

Yes. The custom chart initially deployed **3 replicas** and was successfully upgraded to **5 replicas**.

---

### **Task 7: Clean Up**

1. Uninstalled all Helm releases using `helm uninstall <name>`.
2. Removed the custom chart directory and values file.
3. Reviewed the `--keep-history` option for retaining release history for auditing.

**Verify:** Does `helm list` show zero releases?

Yes. `helm list` showed **zero active releases** after cleanup.

---

- **What Helm is and the three core concepts**
  - Helm is a package manager for Kubernetes that simplifies deploying and managing applications.
  - **Chart** — a package containing Kubernetes manifest templates.
  - **Release** — a specific installed instance of a Helm chart.
  - **Repository** — a collection of Helm charts.

- **How to install, customize, upgrade, and rollback**
  - Installed Helm and verified it with `helm version` and `helm env`.
  - Added the Bitnami repository and deployed the nginx chart.
  - Customized releases using `--set` and `custom-values.yaml`.
  - Upgraded releases by changing chart values.
  - Rolled back a release to an earlier revision using `helm rollback`.

- **The structure of a Helm chart and how Go templating works**
  - `Chart.yaml` contains chart metadata.
  - `values.yaml` contains configurable default values.
  - `templates/` contains Kubernetes manifest templates.
  - Go template expressions such as `{{ .Values.replicaCount }}` dynamically insert values from `values.yaml`, while `{{ .Chart.Name }}` references chart metadata.

- **Your `custom-values.yaml` with explanations**
  - `replicaCount: 3` — configured the application to run three replicas.
  - `service.type: NodePort` — exposed the application through a Kubernetes NodePort Service.
  - `resources` — defined CPU and memory requests/limits to control resource allocation for the Pods.
