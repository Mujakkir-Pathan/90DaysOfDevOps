# Day 84 -- Introduction to GitOps and ArgoCD

## Task

You had deployed the AI-BankApp on EKS using `kubectl apply`. That worked, but who ran the command? From which machine? Was the YAML they applied the same as what is in Git? If someone manually edits a Deployment in the cluster, how do you know?

GitOps solved all of this. Git became the single source of truth. A tool watches your Git repository and continuously ensures the cluster matches what is committed. That tool is ArgoCD.

The AI-BankApp project already had ArgoCD installed via Terraform and an Application manifest ready to go. Today you understood GitOps principles, explored ArgoCD, and deployed the AI-BankApp through ArgoCD for the first time.

---

## project

[AI-BankApp-DevOps](https://github.com/Mujakkir-Pathan/AI-BankApp-DevOps/tree/feat/gitops)

---

## All screenshot's

[screenshots of AI-BankApp-DevOps](screenshots/)

---

## Challenge Tasks

### Task 1: Understand GitOps

Research and write notes on:

1. **What is GitOps?**
   - GitOps is a deployment methodology where Git is the single source of truth for infrastructure and application state.
   - An operator such as ArgoCD watches Git and ensures the live cluster matches the desired state in the repository.
   - If someone changes something in the cluster manually, the operator reverts it through self-healing.
   - All changes go through Git using pull requests, code review, and an audit trail.

2. **GitOps vs traditional CI/CD:**

| **Aspect**         | **Traditional CI/CD**              | **GitOps**                                   |
| ------------------ | ---------------------------------- | -------------------------------------------- |
| Deployment trigger | CI pipeline runs `kubectl apply`   | Git commit triggers sync                     |
| Source of truth    | Pipeline scripts                   | Git repository                               |
| Drift detection    | None                               | Continuous reconciliation                    |
| Rollback           | Re-run pipeline or manual          | `git revert`                                 |
| Audit trail        | Pipeline logs                      | Git history                                  |
| Access control     | Pipeline needs cluster credentials | Only ArgoCD has cluster access               |
| Security           | CI server has broad cluster access | Developers push to Git, never to the cluster |

3. **The AI-BankApp's GitOps flow:**

```text
Developer pushes code to feat/gitops
         |
    [GitHub Actions CI]
    - Build Maven project
    - Run tests
    - Build Docker image
    - Push to DockerHub (tagged with git SHA)
    - Update image tag in k8s/bankapp-deployment.yml
    - Commit the change back to Git
         |
    [ArgoCD watches the repo]
    - Detects the new commit
    - Compares k8s/ manifests with live cluster
    - Syncs the change (rolling update)
    - BankApp pods restart with the new image
         |
    [Zero human intervention after git push]

```

4. **Four GitOps principles** (from OpenGitOps):
   - **Declarative** -- the desired state is expressed declaratively through Kubernetes YAML.
   - **Versioned and immutable** -- the desired state is stored in Git, making it versioned and auditable.
   - **Pulled automatically** -- agents such as ArgoCD pull the desired state instead of CI pushing it to the cluster.
   - **Continuously reconciled** -- agents continuously compare the desired state with the actual state and correct drift.

---

### Task 2: Access ArgoCD on Your EKS Cluster

ArgoCD was installed by Terraform on Day 81 (via `terraform/argocd.tf`). It was verified as running:

```text
kubectl get pods -n argocd
```

The ArgoCD admin password was retrieved:

```text
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

**Access the ArgoCD UI:**

The ArgoCD UI was accessed through the LoadBalancer:

```text
export ARGOCD_URL=$(kubectl get svc argocd-server -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ArgoCD URL: http://$ARGOCD_URL"
```

The UI was accessed using the `admin` account.

**Install the ArgoCD CLI:**

The ArgoCD CLI was installed and verified:

```text
# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

# Verify
argocd version --client
```

The CLI was logged in through the LoadBalancer:

```text
argocd login $ARGOCD_URL --username admin --password <your-password> --insecure
```

**Explore the ArgoCD UI:**

- **Applications** -- shows managed applications.
- **Settings > Repositories** -- shows Git repositories ArgoCD can access.
- **Settings > Clusters** -- shows Kubernetes clusters ArgoCD manages, including the EKS `in-cluster`.

---

### Task 3: Study the AI-BankApp's ArgoCD Application Manifest

The `argocd/application.yml` manifest was studied:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/TrainWithShubham/AI-BankApp-DevOps.git
    targetRevision: feat/gitops
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: bankapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

**Break down every field:**

| **Field**               | **Value**                  | **Purpose**                                         |
| ----------------------- | -------------------------- | --------------------------------------------------- |
| `source.repoURL`        | The AI-BankApp GitHub repo | Where ArgoCD fetches manifests from                 |
| `source.targetRevision` | `feat/gitops`              | Which Git branch to watch                           |
| `source.path`           | `k8s`                      | The directory containing Kubernetes manifests       |
| `destination.server`    | `kubernetes.default.svc`   | Deploy to the local cluster (in-cluster)            |
| `destination.namespace` | `bankapp`                  | Target namespace for resources                      |
| `syncPolicy.automated`  | enabled                    | ArgoCD syncs automatically on Git changes           |
| `prune: true`           | enabled                    | Deletes resources removed from Git                  |
| `selfHeal: true`        | enabled                    | Reverts manual changes made directly to the cluster |
| `CreateNamespace=true`  | enabled                    | Creates the `bankapp` namespace if it does not exist |
| `ServerSideApply=true`  | enabled                    | Uses server-side apply for better conflict handling |

---

### Task 4: Deploy the AI-BankApp via ArgoCD

The BankApp namespace was deleted to start from a clean slate:

```text
kubectl delete namespace bankapp 2>/dev/null
```

The AI-BankApp repository was forked to:

```text
https://github.com/Mujakkir-Pathan/AI-BankApp-DevOps.git
```

The ArgoCD Application was created using the fork and the `feat/gitops` branch:

```text
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Mujakkir-Pathan/AI-BankApp-DevOps.git
    targetRevision: feat/gitops
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: bankapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
EOF
```

ArgoCD was used to deploy and manage the AI-BankApp. The application reached a healthy state and the required resources were synchronized.

---

### Task 5: Explore ArgoCD's Live View

The `bankapp` application was explored in the ArgoCD UI.

**The resource tree:**

```text
bankapp (Application)
  |-- Namespace: bankapp
  |-- StorageClass: gp3
  |-- PVC: mysql-pvc (Bound)
  |-- PVC: ollama-pvc (Bound)
  |-- ConfigMap: bankapp-config
  |-- Secret: bankapp-secret
  |-- Deployment: mysql -> ReplicaSet -> Pod
  |-- Deployment: ollama -> ReplicaSet -> Pod
  |-- Deployment: bankapp -> ReplicaSet -> Pod
  |-- Service: mysql-service
  |-- Service: ollama-service
  |-- Service: bankapp-service
  |-- HPA: bankapp-hpa
```

The application resources were explored to view their details, including:

- Pod logs
- Events
- YAML manifest
- Diff

**App Details tab was explored for:**

- Source repository and path
- Last sync time and revision
- Sync status and health status
- Sync history

**Sync history was checked with:**

```text
argocd app history bankapp
```

The sync history showed the revisions synchronized from the `feat/gitops` branch.

---

### Task 6: Test Self-Healing

ArgoCD's `selfHeal: true` was tested by making manual changes directly to the cluster.

**Test 1 -- Manually scale the BankApp:**

```text
kubectl scale deployment bankapp -n bankapp --replicas=1
```

The BankApp deployment was manually scaled down. ArgoCD detected the drift and restored the deployment to the desired state defined in Git.

**Test 2 -- Manually delete a ConfigMap:**

```text
kubectl delete configmap bankapp-config -n bankapp
```

The `bankapp-config` ConfigMap was manually deleted. ArgoCD detected the drift and recreated the ConfigMap from Git.

**Test 3 -- Manually change an environment variable:**

The `MYSQL_PORT` value in the `bankapp-config` ConfigMap was manually changed from `3306` to `9999`.

ArgoCD detected the drift and restored the value to the Git-defined value of `3306`.

**Self-healing behavior:**

The three tests demonstrated that ArgoCD continuously reconciles the cluster with the desired state in Git. Manual changes were not persistent and were automatically corrected by ArgoCD.

---

# GitOps and ArgoCD Notes

## GitOps Principles in Your Own Words

1. **Declarative** -- The desired state is defined declaratively, such as through Kubernetes YAML.

2. **Versioned and immutable** -- The desired state is stored in Git, so changes are versioned, auditable, and traceable.

3. **Pulled automatically** -- ArgoCD pulls the desired state from Git instead of CI pushing changes directly to the cluster.

4. **Continuously reconciled** -- ArgoCD continuously compares the desired state in Git with the actual cluster state and corrects drift.

---

## GitOps vs Traditional CI/CD

| **Aspect**         | **Traditional CI/CD**              | **GitOps**                                   |
| ------------------ | ---------------------------------- | -------------------------------------------- |
| Deployment trigger | CI pipeline runs `kubectl apply`   | Git commit triggers sync                     |
| Source of truth    | Pipeline scripts                   | Git repository                               |
| Drift detection    | None                               | Continuous reconciliation                    |
| Rollback           | Re-run pipeline or manual          | `git revert`                                 |
| Audit trail        | Pipeline logs                      | Git history                                  |
| Access control     | Pipeline needs cluster credentials | Only ArgoCD has cluster access               |
| Security           | CI server has broad cluster access | Developers push to Git, never to the cluster |

---

## The AI-BankApp's GitOps Flow

```text
Developer pushes code to feat/gitops
         |
    [GitHub Actions CI]
    - Build Maven project
    - Run tests
    - Build Docker image
    - Push to DockerHub (tagged with git SHA)
    - Update image tag in k8s/bankapp-deployment.yml
    - Commit the change back to Git
         |
    [ArgoCD watches the repo]
    - Detects the new commit
    - Compares k8s/ manifests with live cluster
    - Syncs the change (rolling update)
    - BankApp pods restart with the new image
         |
    [Zero human intervention after git push]
```

---

## ArgoCD Application Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/TrainWithShubham/AI-BankApp-DevOps.git
    targetRevision: feat/gitops
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: bankapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### Field-by-Field Explanation

| **Field** | **Value** | **Purpose** |
|---|---|---|
| `apiVersion` | `argoproj.io/v1alpha1` | Defines the ArgoCD API version used by the Application resource. |
| `kind` | `Application` | Defines this Kubernetes resource as an ArgoCD Application. |
| `metadata.name` | `bankapp` | Name of the ArgoCD Application. |
| `metadata.namespace` | `argocd` | Namespace where the ArgoCD Application object exists. |
| `spec.project` | `default` | ArgoCD project used by the Application. |
| `source.repoURL` | AI-BankApp GitHub repository | Git repository from which ArgoCD fetches the manifests. |
| `source.targetRevision` | `feat/gitops` | Git branch that ArgoCD watches. |
| `source.path` | `k8s` | Directory containing the Kubernetes manifests. |
| `destination.server` | `https://kubernetes.default.svc` | Deploys to the local Kubernetes cluster (in-cluster). |
| `destination.namespace` | `bankapp` | Namespace where the application resources are deployed. |
| `syncPolicy.automated` | enabled | Allows ArgoCD to synchronize automatically when changes are detected. |
| `prune` | `true` | Deletes resources from the cluster when they are removed from Git. |
| `selfHeal` | `true` | Reverts manual changes made directly to the cluster. |
| `CreateNamespace=true` | enabled | Creates the `bankapp` namespace if it does not exist. |
| `ServerSideApply=true` | enabled | Uses Kubernetes server-side apply for better conflict handling. |

---

## `prune`, `selfHeal`, and `ServerSideApply`

### `prune: true`

`prune` makes ArgoCD remove resources from the cluster when those resources are no longer defined in Git.

**Example:**

```text
Git:     Deployment + Service
Cluster: Deployment + Service

Service removed from Git
        ↓
ArgoCD detects the difference
        ↓
ArgoCD removes the Service from the cluster
```

### `selfHeal: true`

`selfHeal` makes ArgoCD automatically correct manual changes made directly to the cluster.

**Example:**

```text
Git:     MYSQL_PORT = 3306
Cluster: MYSQL_PORT = 9999
              ↓
       ArgoCD detects drift
              ↓
Cluster: MYSQL_PORT = 3306
```

During the self-healing tests, manually scaling the BankApp, deleting the ConfigMap, and changing the ConfigMap value were all corrected by ArgoCD.

### `ServerSideApply=true`

`ServerSideApply=true` tells ArgoCD to use Kubernetes server-side apply when applying resources.

It provides better handling of field ownership and conflicts when Kubernetes resources are managed by different controllers or tools.

