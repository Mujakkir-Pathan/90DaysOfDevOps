# Day 80 -- Helm Project: Multi-Environment Deployment and CI/CD

## Task

Two days of Helm -- chart basics and a custom chart for the AI-BankApp. Today I brought it all together by creating environment-specific values for dev, staging, and production, adding Helm hooks, packaging the chart, and understanding how Helm integrates into the AI-BankApp's CI/CD pipeline.

Reference: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch: `feat/gitops`)

---

## project on github

[AI-BankApp-DevOps](https://github.com/Mujakkir-Pathan/AI-BankApp-DevOps/tree/main/helm-chart/bankapp/)

---

## All screenshot's

[screenshots of task](screenshots/)


---

## Challenge Tasks

### Task 1: Create Environment-Specific Values

Created three environment-specific values files for the same `bankapp` Helm chart.

### `bankapp/values-dev.yaml`

```yaml
bankapp:
  replicaCount: 1
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "latest"
    pullPolicy: Always
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "250m"
  autoscaling:
    enabled: false

mysql:
  enabled: true
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "250m"
  persistence:
    size: 2Gi
    storageClass: standard

ollama:
  enabled: true
  model: tinyllama
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "1.5Gi"
      cpu: "1000m"
  persistence:
    size: 5Gi
    storageClass: standard

storageClass:
  create: false
```

### `bankapp/values-staging.yaml`

```yaml
bankapp:
  replicaCount: 2
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "v1.2.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 3
    targetCPUUtilization: 75

mysql:
  enabled: true
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  persistence:
    size: 5Gi
    storageClass: gp3

ollama:
  enabled: true
  model: tinyllama
  persistence:
    size: 10Gi
    storageClass: gp3

secrets:
  mysqlRootPassword: StagingPass@456
  mysqlUser: root
  mysqlPassword: StagingPass@456

storageClass:
  create: true
```

### `bankapp/values-prod.yaml`

```yaml
bankapp:
  replicaCount: 4
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "v1.2.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 4
    targetCPUUtilization: 70

mysql:
  enabled: true
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  persistence:
    size: 20Gi
    storageClass: gp3

ollama:
  enabled: true
  model: tinyllama
  resources:
    requests:
      memory: "2Gi"
      cpu: "900m"
    limits:
      memory: "2.5Gi"
      cpu: "1500m"
  persistence:
    size: 10Gi
    storageClass: gp3

secrets:
  mysqlRootPassword: ProdSecure@789
  mysqlUser: root
  mysqlPassword: ProdSecure@789

storageClass:
  create: true

gateway:
  enabled: true
```

**Compare the environments:**

| Setting          | Dev        | Staging    | Prod       |
| ---------------- | ---------- | ---------- | ---------- |
| BankApp replicas | 1 (fixed)  | 2-3 (HPA)  | 2-4 (HPA)  |
| Image tag        | latest     | v1.2.0     | v1.2.0     |
| MySQL storage    | 2Gi        | 5Gi        | 20Gi       |
| MySQL resources  | 128Mi/100m | 256Mi/250m | 512Mi/500m |
| Ollama memory    | 1Gi        | 2Gi        | 2.5Gi      |
| Gateway          | disabled   | disabled   | enabled    |

**Deploy to different environments:**

The dev environment was installed successfully:

```bash
helm install bankapp-dev bankapp/ -f bankapp/values-dev.yaml -n dev --create-namespace
```

Result:

```text
NAME: bankapp-dev
NAMESPACE: dev
STATUS: deployed
REVISION: 1
CHART: bankapp-0.1.0
APP VERSION: 1.0.0
```

The staging configuration was rendered to verify the replica configuration:

```bash
helm template bankapp-staging bankapp/ -f bankapp/values-staging.yaml --set bankapp.autoscaling.enabled=false | grep "replicas:"
```

Output:

```text
replicas: 2
```

The production configuration was rendered to verify the replica configuration:

```bash
helm template bankapp-prod bankapp/ -f bankapp/values-prod.yaml --set bankapp.autoscaling.enabled=false | grep "replicas:"
```

Output:

```text
replicas: 4
```

The `--set` option was used only for the render check so that the fixed `replicas` value could be verified without changing the values file.

The same Helm chart was therefore configured differently for dev, staging, and production using separate values files.

---

### Task 2: Add Helm Hooks

Created `bankapp/templates/pre-install-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "bankapp.fullname" . }}-db-ready
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      containers:
        - name: db-check
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
            - |
              echo "Waiting for MySQL to be ready..."
              until nc -z {{ include "bankapp.fullname" . }}-mysql 3306; do
                echo "MySQL not ready, retrying in 3s..."
                sleep 3
              done
              echo "MySQL is ready!"
          resources:
            requests: { memory: "32Mi", cpu: "50m" }
            limits: { memory: "64Mi", cpu: "100m" }
      restartPolicy: Never
  backoffLimit: 10
```

#### Hook annotations explained

* `helm.sh/hook: pre-install,pre-upgrade` makes the Job run before a Helm install or upgrade.
* The Job checks whether MySQL is accepting connections on port `3306`.
* `helm.sh/hook-weight: "0"` controls the execution order when multiple hooks are present.
* `helm.sh/hook-delete-policy: before-hook-creation` removes the previous hook Job before creating a new one during another install or upgrade.
* The existing BankApp Deployment also uses init containers to wait for MySQL, so the hook provides an additional readiness check.

Created the Helm test at `bankapp/templates/tests/test-connection.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "bankapp.fullname" . }}-test
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ['sh', '-c', 'wget -qO- http://{{ include "bankapp.fullname" . }}-service:8080/actuator/health']
  restartPolicy: Never
```

Ran the Helm test:

```bash
helm test bankapp-dev -n dev
```

Result:

```text
NAME: bankapp-dev
LAST DEPLOYED: Mon Sep 28 05:13:07 2026
NAMESPACE: dev
STATUS: deployed
REVISION: 1
TEST SUITE:     bankapp-dev-test
Last Started:   Mon Sep 28 05:35:18 2026
Last Completed: Mon Sep 28 05:35:20 2026
Phase:          Succeeded
```

The test successfully reached the Spring Boot health endpoint, confirming that the deployed BankApp service was responding.

---

### Task 3: Package and Version the Chart

Ran Helm lint before packaging:

```bash
helm lint bankapp/
```

Output:

```text
==> Linting bankapp/
[INFO] Chart.yaml: icon is recommended
1 chart(s) linted, 0 chart(s) failed
```

The chart passed linting successfully. The icon message was only an informational recommendation.

Packaged the initial chart:

```bash
helm package bankapp/
```

Created:

```text
bankapp-0.1.0.tgz
```

Updated `bankapp/Chart.yaml`:

```yaml
version: 0.2.0
appVersion: "1.1.0"
```

Re-packaged the chart:

```bash
helm package bankapp/
```

Created:

```text
bankapp-0.2.0.tgz
```

The chart directory therefore produced both versions:

```text
bankapp-0.1.0.tgz
bankapp-0.2.0.tgz
```

Installed the packaged chart:

```bash
helm install my-bankapp bankapp-0.2.0.tgz -f bankapp/values-dev.yaml -n bankapp --create-namespace
```

Result:

```text
NAME: my-bankapp
LAST DEPLOYED: Mon Sep 28 06:45:32 2026
NAMESPACE: bankapp
STATUS: deployed
REVISION: 1
```

The installed package was `bankapp-0.2.0` with app version `1.1.0`.

Created a local chart repository directory:

```bash
mkdir chart-repo
cp bankapp-*.tgz chart-repo/
```

Generated the repository index:

```bash
helm repo index chart-repo/ --url https://Mujakkir-Pathan.github.io/helm-charts
```

The generated `chart-repo/index.yaml` contained entries for both:

```text
bankapp-0.2.0.tgz
bankapp-0.1.0.tgz
```

The chart repository index was generated locally. It was not published to GitHub Pages as part of this task.

---

### Task 4: Understand Helm in the AI-BankApp GitOps Pipeline

The existing raw-manifest GitOps pipeline works as follows:

```text
Developer pushes code
  -> GitHub Actions builds Docker image
  -> Tags with git commit SHA
  -> Updates image tag in k8s/bankapp-deployment.yml via sed
  -> Commits the change back to the repo
  -> ArgoCD detects the change and syncs to EKS
```

With Helm, the pipeline can work as:

```text
Developer pushes code
  -> GitHub Actions builds Docker image
  -> Tags with git commit SHA
  -> Updates image.tag in helm-chart/values-prod.yaml
  -> Commits the change back to the repo
  -> ArgoCD detects the change
  -> ArgoCD renders the Helm chart
  -> Kubernetes resources are applied to EKS
```

The CI step can update the Helm values file using `yq`:

```yaml
- name: Update Helm values with new image tag
  run: |
    TAG=${{ steps.tag.outputs.sha_short }}
    yq -i '.bankapp.image.tag = "'$TAG'"' helm-chart/bankapp/values-prod.yaml

- name: Commit updated Helm values
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add helm-chart/bankapp/values-prod.yaml
    git diff --staged --quiet || git commit -m "ci: update bankapp image to $TAG [skip ci]"
    git push
```

The important change is that the CI pipeline updates the Helm values file instead of directly editing a Kubernetes Deployment manifest.

#### ArgoCD with raw manifests

The current raw-manifest source is:

```yaml
source:
  path: k8s
```

ArgoCD reads the Kubernetes YAML files directly from the `k8s` directory.

#### ArgoCD with Helm

The Helm source can be:

```yaml
source:
  path: helm-chart/bankapp
  helm:
    valueFiles:
      - values-prod.yaml
```

ArgoCD supports Helm natively. It renders the Helm chart using the selected values file and then applies the resulting Kubernetes resources.

#### Advantages of Helm with ArgoCD

* **Environment management:** The same chart can use different values files for dev, staging, and production.
* **Less duplication:** Templates can represent common Kubernetes resources instead of maintaining separate YAML files for every environment.
* **Centralized configuration:** Environment-specific settings are kept in values files.
* **Versioning:** Helm charts can be packaged and versioned.
* **Templating:** Image tags, replicas, resources, storage, and other settings can be changed through values.
* **GitOps compatibility:** ArgoCD can monitor the Helm chart and values stored in Git.
* **Drift detection:** ArgoCD compares the desired state from Git with the state running in Kubernetes.
* **Rollback support:** Helm releases provide revision history that can be used for release management.

The important flow is:

```text
Helm Chart + Values
        |
        v
   Helm renders
        |
        v
 Kubernetes YAML
        |
        v
   Kubernetes / EKS
```

Helm does not replace Kubernetes. Helm generates and manages the Kubernetes resources that Kubernetes runs.

---

### Task 5: Helm Best Practices for Production

#### 1. Always use `helm upgrade --install`

Reviewed the production deployment pattern:

```bash
helm upgrade --install bankapp bankapp/ \
  -f bankapp/values-prod.yaml \
  --set bankapp.image.tag=$GIT_SHA \
  -n bankapp --create-namespace \
  --wait --timeout 300s \
  --atomic
```

The command was reviewed for understanding of production deployment behavior rather than executed as the production deployment.

* `--install` creates the release if it does not exist.
* `--set bankapp.image.tag=$GIT_SHA` allows CI/CD to deploy the exact Git commit image.
* `--wait` waits for the Kubernetes resources to become ready.
* `--timeout 300s` gives resources up to five minutes to become ready.
* `--atomic` automatically rolls back the release if the upgrade fails.

#### 2. Use `helm diff` before upgrading

Installed the Helm Diff plugin:

```bash
helm plugin install https://github.com/databus23/helm-diff
```

The plugin was installed successfully as version `v3.15.15`.

The release that existed in the `bankapp` namespace was `my-bankapp`, so the diff was run against that release:

```bash
helm diff upgrade my-bankapp bankapp/ \
  -f bankapp/values-prod.yaml \
  -n bankapp
```

The command successfully generated the Helm diff. The output was large, so it was not reproduced in full.

`helm diff` only previews the changes. It does not perform the upgrade.

#### 3. Resource quotas per namespace

Created `bankapp/templates/resourcequota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: {{ include "bankapp.fullname" . }}-quota
  namespace: {{ .Release.Namespace }}
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
```

Verified the rendered ResourceQuota:

```bash
helm template bankapp bankapp/ -f bankapp/values-prod.yaml | grep -A 10 "kind: ResourceQuota"
```

Output:

```text
kind: ResourceQuota
metadata:
  name: bankapp-quota
  namespace: default
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
```

The namespace appeared as `default` because `helm template` was run without `-n`. During an actual namespaced deployment, `.Release.Namespace` uses the release namespace.

#### 4. Production secrets management

The staging and production values files used the task-provided example credentials for the lab.

Real production passwords should not be committed directly into Git or stored as plaintext in `values.yaml`.

For production secret management, the following approaches were reviewed:

* External Secrets Operator with AWS Secrets Manager
* Sealed Secrets
* HashiCorp Vault

The important production principle is:

```text
Application configuration
        |
        v
     Helm values
        |
        +---- non-sensitive configuration
        |
        v
External secret system
        |
        v
 Kubernetes Secret
```

Real production credentials should therefore be stored in a dedicated secret-management system rather than directly in the Helm chart repository.

---

### Task 6: Clean Up and Review

Checked all Helm releases:

```bash
helm list -A
```

Output:

```text
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
bankapp-dev     dev             1               2026-09-28 05:13:07.212491989 +0000 UTC deployed        bankapp-0.1.0   1.0.0
my-bankapp      bankapp         1               2026-09-28 06:45:32.750739981 +0000 UTC deployed        bankapp-0.2.0   1.1.0
```

#### Reflect and document the 3-day Helm journey

| Day | Concept                                        | AI-BankApp Connection                                                                                                              |
| --- | ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| 78  | Helm install, repos, values, upgrade, rollback | Deployed MySQL for the BankApp via Bitnami chart                                                                                   |
| 79  | Custom chart from scratch, Go templates        | Converted 12 raw `k8s/` manifests into a Helm chart                                                                                |
| 80  | Multi-env values, hooks, packaging, CI/CD      | Created a multi-environment Helm chart with dev/staging/prod configurations, hooks, packaging, and CI/CD integration understanding |

#### When would you use Helm vs raw manifests vs Kustomize?

| Approach      | Best For                                      | AI-BankApp Example                                               |
| ------------- | --------------------------------------------- | ---------------------------------------------------------------- |
| Raw manifests | Simple, single-env deployments                | The current `k8s/` directory                                     |
| Helm          | Multi-env, complex apps with dependencies     | The custom `bankapp` chart with services, HPA, hooks, and values |
| Kustomize     | Overlays on existing manifests, no templating | Useful for patching `k8s/` without rewriting the manifests       |

#### Clean up

Removed the dev Helm release:

```bash
helm uninstall bankapp-dev -n dev
```

Output:

```text
release "bankapp-dev" uninstalled
```

Deleted the dev namespace:

```bash
kubectl delete namespace dev
```

Output:

```text
namespace "dev" deleted
```

Deleted the Kind cluster:

```bash
kind delete cluster --name tws-cluster
```

Output:

```text
Deleting cluster "tws-cluster" ...
Deleted nodes: ["tws-cluster-worker2" "tws-cluster-worker" "tws-cluster-control-plane"]
```

The `my-bankapp` release in the `bankapp` namespace was not removed because it was not included in the specified cleanup commands.

---

## Documentation

### Environment-specific values

Created:

```text
bankapp/values-dev.yaml
bankapp/values-staging.yaml
bankapp/values-prod.yaml
```

The three files use the same Helm chart but provide different replicas, resources, storage, autoscaling, image tags, and Gateway configuration.

### Helm hooks

Created:

```text
bankapp/templates/pre-install-job.yaml
bankapp/templates/tests/test-connection.yaml
```

The `pre-install,pre-upgrade` hook checks MySQL readiness before the Helm install or upgrade proceeds.

The `test` hook verifies the BankApp Spring Boot health endpoint when `helm test` is executed.

### Helm package

Created:

```text
bankapp-0.1.0.tgz
bankapp-0.2.0.tgz
```

The chart version was updated from `0.1.0` to `0.2.0`, and the application version was updated from `1.0.0` to `1.1.0`.

### GitOps CI/CD integration

The Helm-based GitOps flow is:

```text
Developer
    |
    v
Git Push
    |
    v
GitHub Actions
    |
    +--> Build Docker Image
    |
    +--> Tag Image with Git SHA
    |
    +--> Update Helm values
    |
    v
Git Repository
    |
    v
ArgoCD
    |
    +--> Render Helm Chart
    |
    +--> Detect Drift
    |
    v
Kubernetes / EKS
```

### Helm vs Raw Manifests vs Kustomize

| Approach      | Best For                                   | AI-BankApp Example              |
| ------------- | ------------------------------------------ | ------------------------------- |
| Raw manifests | Simple, single-environment deployments     | Existing `k8s/` directory       |
| Helm          | Multi-environment and complex applications | Custom `bankapp` Helm chart     |
| Kustomize     | Patching and overlaying existing manifests | Modifying `k8s/` using overlays |

### Production secrets management

For production, real credentials should not be stored directly in Helm values or Git.

The production setup should use a dedicated secret-management solution such as:

* External Secrets Operator + AWS Secrets Manager
* Sealed Secrets
* HashiCorp Vault

The Helm chart should contain configuration references rather than real production credentials.


