# Day 78 -- Introduction to Helm and Chart Basics

## Task
I worked with Helm to understand how Kubernetes applications can be packaged, configured, versioned, and managed as reusable Helm charts instead of managing every Kubernetes manifest separately.

The AI-BankApp project was used from the `feat/gitops` branch. Its `k8s/` directory contained 12 raw Kubernetes YAML files that included Deployments, Services, ConfigMaps, Secrets, PVCs, HPA, and other resources.

---

## All screenshot's

[screenshots of task](screenshots/)

---

## Challenge Tasks

### Task 1: Understand Helm Concepts

#### 1. What is Helm?

Helm is a package manager for Kubernetes, similar to how `apt` is used for Ubuntu packages.

Instead of writing every Kubernetes manifest manually, Helm packages Kubernetes resources into reusable units called **charts**.

A Helm chart can contain templates for resources such as:

- Deployment or StatefulSet
- Service
- ConfigMap
- Secret
- PersistentVolumeClaim
- ServiceMonitor
- Other Kubernetes resources

Helm also supports templating, which means the same chart can be used in different environments by changing values instead of creating completely different YAML files.

#### 2. Core Concepts

| Concept | Explanation |
|---|---|
| **Chart** | A reusable package containing templates and configuration for Kubernetes resources. |
| **Release** | A running installation of a chart inside a Kubernetes cluster. The same chart can be installed multiple times using different release names. |
| **Repository** | A location where Helm charts are stored and shared. It is similar in concept to a package repository. |
| **Values** | Configuration used to customize a chart, such as image tags, replicas, resources, passwords, persistence, and metrics. |

The relationship can be understood as:

```text
Helm Repository
      |
      v
    Chart
      |
      +---- values.yaml / custom values
      |
      v
 Helm renders templates
      |
      v
 Kubernetes resources
      |
      v
    Release
```

#### 3. Why Helm over Raw Manifests?

The AI-BankApp had 12 separate Kubernetes YAML files in its `k8s/` directory.

With raw manifests, configuration changes have to be made directly in YAML files. For example, changing an image tag requires editing the relevant manifest.

Helm provides:

- **Templating:** one chart can be used for dev, staging, and production with different values.
- **Versioning:** charts have versions and Helm keeps release revisions.
- **Rollback:** a release can be rolled back using `helm rollback`.
- **Dependencies:** charts can use other charts as dependencies.
- **Community charts:** common applications such as MySQL, Redis, Prometheus, and others are available as charts.

---

### Task 2: Install Helm and Explore the AI-BankApp

#### Set up the Kind cluster

The AI-BankApp repository was cloned from the `feat/gitops` branch:

```bash
git clone -b feat/gitops https://github.com/TrainWithShubham/AI-BankApp-DevOps.git
cd AI-BankApp-DevOps
```

The Kind cluster was created using the project's configuration:

```bash
kind create cluster --config setup-k8s/kind-config.yml
```

The resulting cluster used:

- Kind cluster: `tws-cluster`
- Kubernetes node image: `kindest/node:v1.35.0`
- 1 control-plane node
- 2 worker nodes
- Kubernetes context: `kind-tws-cluster`

#### Install Helm

Helm was installed on the Linux control host using:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

The installed Helm version was:

```text
v3.22.0
```

#### Verify Kubernetes connectivity

The Kubernetes cluster was verified with:

```bash
kubectl cluster-info
```

The control plane and CoreDNS were reachable.

The initial Helm release list was empty:

```bash
helm list
```

#### Explore the raw manifests

The `k8s/` directory contained exactly 12 files:

```text
bankapp-deployment.yml
cert-manager.yml
configmap.yml
gateway.yml
hpa.yml
mysql-deployment.yml
namespace.yml
ollama-deployment.yml
pv.yml
pvc.yml
secrets.yml
service.yml
```

These raw manifests represented the Kubernetes resources that would eventually benefit from Helm templating and values.

---

### Task 3: Deploy MySQL Using a Helm Chart

The Bitnami Helm repository was added and updated:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

The MySQL chart was searched using:

```bash
helm search repo bitnami/mysql
```

The MySQL release was deployed using the Bitnami chart.

During the actual deployment, the Bitnami chart required the legacy image configuration, so the working installation used:

```bash
helm install bankapp-mysql bitnami/mysql \
  --set global.security.allowInsecureImages=true \
  --set image.repository=bitnamilegacy/mysql \
  --set auth.rootPassword='Test@123' \
  --set auth.database=bankappdb \
  --set primary.resources.requests.memory=256Mi \
  --set primary.resources.requests.cpu=250m \
  --set primary.resources.limits.memory=512Mi \
  --set primary.resources.limits.cpu=500m \
  --set primary.persistence.size=5Gi
```

The release was successfully deployed:

```text
NAME: bankapp-mysql
STATUS: deployed
REVISION: 1
CHART VERSION: 14.0.3
APP VERSION: 9.4.0
```

#### Verify the deployed resources

Using:

```bash
kubectl get all -l app.kubernetes.io/instance=bankapp-mysql
```

the following resources were found:

```text
pod/bankapp-mysql-0                         1/1 Running
service/bankapp-mysql                      ClusterIP
service/bankapp-mysql-headless             ClusterIP None
statefulset.apps/bankapp-mysql             1/1
```

The label `app.kubernetes.io/instance=bankapp-mysql` was used because the resources were managed by the Helm release.

The PVC was also created:

```text
data-bankapp-mysql-0    Bound    5Gi    RWO    standard
```

The Helm-managed Secret was present:

```text
bankapp-mysql    Opaque
```

#### Verify MySQL

MySQL was checked using:

```bash
kubectl exec -it bankapp-mysql-0 -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"
```

The database list included:

```text
bankappdb
information_schema
mysql
performance_schema
sys
```

Therefore, the `bankappdb` database expected by the AI-BankApp configuration was successfully created.

#### Raw YAML vs Helm

With raw Kubernetes YAML, the MySQL deployment required multiple files and manually connected resources such as:

```text
mysql-deployment.yml
secrets.yml
pvc.yml
pv.yml
service.yml
```

With Helm, the Bitnami chart generated and managed the required Kubernetes resources from the chart templates and supplied values.

---

### Task 4: Customize a Deployment with Values Files

A values file named `mysql-values.yaml` was created.

```yaml
auth:
  rootPassword: Test@123
  database: bankappdb

primary:
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi
  persistence:
    size: 5Gi
    storageClass: ""

metrics:
  enabled: true
  serviceMonitor:
    enabled: false
```

The values file was used to create a second Helm release:

```bash
helm install bankapp-mysql-v2 bitnami/mysql -f mysql-values.yaml
```

The release was successfully deployed as:

```text
NAME: bankapp-mysql-v2
STATUS: deployed
REVISION: 1
CHART VERSION: 14.0.3
APP VERSION: 9.4.0
```

The chart's configurable values were inspected using:

```bash
helm show values bitnami/mysql | head -80
```

This showed that the chart exposes many configurable options through `values.yaml`, including image settings, storage, resources, metrics, replication, and other configuration.

#### Explanation of `mysql-values.yaml`

| Field | Purpose |
|---|---|
| `auth.rootPassword` | Sets the MySQL root password. |
| `auth.database` | Creates the initial `bankappdb` database. |
| `primary.resources.limits.cpu` | Sets the maximum CPU available to the primary MySQL container. |
| `primary.resources.limits.memory` | Sets the maximum memory available to the primary MySQL container. |
| `primary.resources.requests.cpu` | Sets the CPU requested by the MySQL container. |
| `primary.resources.requests.memory` | Sets the memory requested by the MySQL container. |
| `primary.persistence.size` | Sets the persistent storage size to 5Gi. |
| `primary.persistence.storageClass` | Leaves the storage class unset so the cluster's default storage class can be used. |
| `metrics.enabled` | Enables MySQL metrics support. |
| `metrics.serviceMonitor.enabled` | Keeps ServiceMonitor creation disabled. |

The second release was cleaned up as required:

```bash
helm uninstall bankapp-mysql-v2
```

---

### Task 5: Manage Releases -- Upgrade, Rollback, Uninstall

Helm tracks changes to a release using revisions.

The MySQL release was upgraded to enable metrics:

```bash
helm upgrade bankapp-mysql bitnami/mysql \
  --set auth.rootPassword=Test@123 \
  --set auth.database=bankappdb \
  --set metrics.enabled=true \
  --reuse-values
```

The upgrade completed successfully as revision 4.

The actual release history was then checked:

```bash
helm history bankapp-mysql
```

The history contained:

```text
REVISION  STATUS
1         superseded
2         failed
3         failed
4         deployed
```

Revisions 2 and 3 were failed upgrade attempts caused by StatefulSet specification changes that Kubernetes did not allow.

The release was then rolled back:

```bash
helm rollback bankapp-mysql 1
```

The rollback succeeded.

The new history became:

```text
REVISION  STATUS
1         superseded
2         failed
3         failed
4         superseded
5         deployed
```

The important point is that rolling back to revision 1 did not delete the later history. Helm created a new revision, revision 5, representing the rollback operation.

#### Helm rollback vs raw manifests

With raw manifests, `kubectl apply` does not provide the same built-in Helm release revision and rollback workflow.

With Helm:

```text
Install
  ↓
Revision 1
  ↓
Upgrade
  ↓
New revision
  ↓
Rollback
  ↓
New revision containing the rollback
```

The rendered Helm manifest was also exported using:

```bash
helm get manifest bankapp-mysql > bankapp-mysql-helm-manifest.yaml
```

The raw AI-BankApp MySQL manifest was compared against the Helm-generated resources.

The raw manifest used:

```yaml
kind: Deployment
metadata:
  name: mysql
```

The Bitnami chart used a `StatefulSet` for the MySQL primary.

The raw AI-BankApp manifest directly specified the MySQL image, resources, environment variables, volume mount, PVC reference, and probes. The Helm chart generated Kubernetes resources from reusable templates and values.

---

### Task 6: Explore a Chart's Structure

The Bitnami MySQL chart was pulled locally:

```bash
helm pull bitnami/mysql --untar
ls mysql/
```

The extracted chart contained:

```text
Chart.lock
Chart.yaml
README.md
charts
templates
values.schema.json
values.yaml
```

#### Chart structure

The important parts were:

```text
mysql/
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── primary/
    │   ├── statefulset.yaml
    │   ├── svc.yaml
    │   ├── svc-headless.yaml
    │   ├── configmap.yaml
    │   ├── initialization-configmap.yaml
    │   ├── startdb-configmap.yaml
    │   └── pdb.yaml
    ├── secondary/
    │   ├── statefulset.yaml
    │   ├── svc.yaml
    │   ├── svc-headless.yaml
    │   ├── configmap.yaml
    │   └── pdb.yaml
    ├── _helpers.tpl
    ├── NOTES.txt
    ├── secrets.yaml
    ├── metrics-svc.yaml
    ├── servicemonitor.yaml
    └── other supporting templates
```

The actual current Bitnami chart contained more templates than the simplified structure in the task because the chart supports additional features.

#### Important files

| File/Directory | Purpose |
|---|---|
| `Chart.yaml` | Contains Helm chart metadata such as name, chart version, application version, description, dependencies, and maintainers. |
| `values.yaml` | Contains default configuration values that control how the chart is rendered. |
| `charts/` | Contains chart dependencies. |
| `templates/` | Contains Kubernetes resource templates used by Helm. |
| `templates/primary/statefulset.yaml` | Template used to create the primary MySQL StatefulSet. |
| `templates/primary/svc.yaml` | Template used to create the primary MySQL Service. |
| `templates/_helpers.tpl` | Contains reusable Helm template helpers. |
| `templates/secrets.yaml` | Template for Secret-related resources. |
| `templates/NOTES.txt` | Displays useful post-install information to the user. |
| `templates/metrics-svc.yaml` | Creates the metrics service when metrics are enabled. |
| `templates/servicemonitor.yaml` | Creates a Prometheus ServiceMonitor when enabled. |
| `templates/secondary/` | Contains templates for secondary MySQL instances. |

#### Go template syntax

The `templates/primary/statefulset.yaml` file was inspected with:

```bash
grep -n '{{' mysql/templates/primary/statefulset.yaml | head -20
```

The output showed Helm template expressions such as:

```text
apiVersion: {{ include "common.capabilities.statefulset.apiVersion" . }}
name: {{ include "mysql.primary.fullname" . }}
namespace: {{ include "common.names.namespace" . | quote }}
podManagementPolicy: {{ .Values.primary.podManagementPolicy | quote }}
```

The important pattern is:

```text
{{ .Values.primary.podManagementPolicy }}
```

Here:

- `.Values` refers to Helm values.
- `primary` refers to the primary MySQL configuration section.
- `podManagementPolicy` is the specific value being read.
- `{{ ... }}` tells Helm to evaluate the expression while rendering the template.

The basic flow is:

```text
values.yaml
     ↓
.Values.some.setting
     ↓
Helm template
     ↓
Rendered Kubernetes YAML
     ↓
Kubernetes resource
```

For example, if a chart contains:

```yaml
replicas: {{ .Values.primary.replicaCount }}
```

then:

```bash
--set primary.replicaCount=3
```

can override that value and Helm can render the template with `3`.

#### `Chart.yaml`

The actual `Chart.yaml` contained:

```yaml
apiVersion: v2
name: mysql
description: MySQL is a fast, reliable, scalable, and easy to use open source relational database system.
version: 14.0.3
appVersion: 9.4.0
```

It also contained chart metadata, dependencies, maintainers, source information, and image annotations.

#### Difference between `version` and `appVersion`

The difference is:

- **`version`** is the version of the **Helm chart itself**.
- **`appVersion`** is the version of the **application packaged/deployed by the chart**.

For this chart:

```yaml
version: 14.0.3
appVersion: 9.4.0
```

This means the Helm chart version was `14.0.3`, while the MySQL application version represented by the chart was `9.4.0`.

These versions are independent. A chart can change while continuing to package the same application version.

### Helm Chart vs AI-BankApp Raw MySQL Manifest

| Aspect | AI-BankApp `k8s/mysql-deployment.yml` | Bitnami MySQL Helm Chart |
|---|---|---|
| **Secrets** | Uses manually defined Secret references | Generated and managed through Helm templates |
| **Storage** | Manual PVC/storage configuration | Configured through Helm values such as `primary.persistence.size` |
| **Replicas** | Defined directly in Kubernetes YAML | Configurable through chart values |
| **Metrics** | Not included in the raw MySQL Deployment | Can be enabled with `metrics.enabled: true` |
| **Rollback** | Manual process such as reverting/reapplying manifests | Built-in `helm rollback` |
| **Workload** | `Deployment` | `StatefulSet` for the MySQL primary |

### Why the AI-BankApp's 12 Raw YAML Files Would Benefit from Being a Helm Chart

The AI-BankApp's 12 raw YAML files contain many Kubernetes resources that have to be maintained individually.

Converting them into a Helm chart would provide:

1. **Reusable templates**  
   The same Kubernetes resource definitions could be reused without copying complete YAML files.

2. **Environment-specific configuration**  
   Dev, staging, and production could use different values files instead of manually editing manifests.

3. **Centralized configuration**  
   Image tags, replicas, resource limits, storage sizes, and other settings could be controlled through Helm values.

4. **Release management**  
   Helm would keep release revisions and provide `helm history`, `helm upgrade`, `helm rollback`, and `helm uninstall`.

5. **Cleaner application packaging**  
   The application's Kubernetes resources could be packaged as one reusable chart instead of managing 12 independent YAML files.

6. **Dependencies**  
   External components such as MySQL could be managed through Helm dependencies or community charts.

The main idea is:

```text
Raw Kubernetes approach

12 YAML files
     ↓
Manual configuration
     ↓
kubectl apply
     ↓
Kubernetes


Helm approach

One reusable chart
     +
Values for each environment
     ↓
Helm rendering
     ↓
Kubernetes resources
     ↓
Managed Helm release
```

---

## Cleanup

The task required the MySQL Helm release and extracted chart to be removed.

The release was uninstalled successfully:

```bash
helm uninstall bankapp-mysql
```

Output:

```text
release "bankapp-mysql" uninstalled
```

The extracted chart directory was then removed:

```bash
rm -rf mysql/
```

The second release had already been removed during Task 4:

```bash
helm uninstall bankapp-mysql-v2
```

---

## Final Result

Day 78 covered the complete Helm basics workflow:

```text
Helm installation
      ↓
Kind Kubernetes cluster
      ↓
Helm repository
      ↓
MySQL chart
      ↓
Helm release
      ↓
Values file
      ↓
Upgrade
      ↓
Release history
      ↓
Rollback
      ↓
Chart structure
      ↓
Cleanup
```

The main takeaway was that Helm separates **Kubernetes resource templates** from **deployment configuration**. Templates describe how resources are created, while values control how a particular deployment is configured.
