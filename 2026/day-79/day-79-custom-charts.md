# Day 79 -- Creating a Custom Helm Chart for AI-BankApp

## Task

I created a custom Helm chart for the AI-BankApp project and converted the existing Kubernetes manifests into reusable Helm templates.

The goal was to move from manually managing individual Kubernetes YAML files to a single configurable Helm chart that could deploy the BankApp, MySQL, Ollama, storage, services, and autoscaling configuration.

---

## All project

[today's Project](helm-chart/)

---

## All screenshot's

[screenshots of task](screenshots/)

---

## Task 1: Review the Existing Kubernetes Manifests

The AI-BankApp project was used from the `feat/gitops` branch.

The existing `k8s/` directory contained 12 Kubernetes manifests:

1. `namespace.yml`
2. `configmap.yml`
3. `secrets.yml`
4. `pv.yml`
5. `pvc.yml`
6. `bankapp-deployment.yml`
7. `mysql-deployment.yml`
8. `ollama-deployment.yml`
9. `service.yml`
10. `hpa.yml`
11. `gateway.yml`
12. `cert-manager.yml`

These manifests were reviewed to understand which values needed to become configurable Helm values.

---

## Task 2: Create the Helm Chart

I created the Helm chart directory and scaffolded a new chart using:

```text
helm create bankapp
```

The default generated templates were removed because the chart needed custom templates for the existing AI-BankApp resources.

The chart structure was then organized around:

* `Chart.yaml`
* `values.yaml`
* `_helpers.tpl`
* `NOTES.txt`
* ConfigMap
* Secret
* Storage
* BankApp Deployment
* MySQL Deployment
* Ollama Deployment
* Services
* HPA

---

## Task 3: Configure Chart.yaml and values.yaml

### Chart.yaml

The chart was configured with:

* Chart name: `bankapp`
* Chart version: `0.1.0`
* Application version: `1.0.0`
* Application description
* Maintainer: `TrainWithShubham`
* Relevant keywords for BankApp, Spring Boot, MySQL, Ollama, and AI

### values.yaml

The configurable values were organized into sections for:

* BankApp
* MySQL
* Ollama
* Shared configuration
* Secrets
* Storage
* Gateway

The BankApp configuration included:

* 4 replicas
* `trainwithshubham/ai-bankapp-eks:latest`
* CPU and memory requests/limits
* ClusterIP service on port `8080`
* HPA enabled
* Minimum replicas: 2
* Maximum replicas: 4
* Target CPU utilization: 70%

The MySQL configuration included:

* MySQL `8.0`
* 5Gi persistent storage
* CPU and memory requests/limits

The Ollama configuration included:

* `ollama/ollama:latest`
* `tinyllama` model
* 10Gi persistent storage
* CPU and memory requests/limits

The chart also supported optional Gateway configuration.

---

## Task 4: Convert Kubernetes Manifests into Helm Templates

The existing Kubernetes resources were converted into reusable Helm templates.

### ConfigMap

The ConfigMap was templated so that database configuration and the Ollama service URL could be generated from Helm values and the release name.

### Secret

Database credentials were moved into Helm values and encoded using Helm's `b64enc` function.

### Storage

Storage resources were made configurable.

The chart could create a `gp3` StorageClass for environments such as EKS, while the Kind installation could use the existing `standard` StorageClass.

### BankApp Deployment

The BankApp Deployment was templated with configurable:

* Image repository
* Image tag
* Replica configuration
* Resource requests and limits
* Environment variables
* Ollama dependency

When HPA was enabled, the Deployment did not define a fixed replica count so that the HPA could control scaling.

### MySQL Deployment

The MySQL Deployment was made configurable through Helm values and connected to its persistent volume.

### Ollama Deployment

The Ollama Deployment was configured with:

* Ollama image
* Persistent storage
* Configurable model
* Resource limits
* Readiness probe
* Liveness probe

The configured `tinyllama` model was pulled when the Ollama container started.

### Services

Services were created for:

* BankApp on port `8080`
* MySQL on port `3306`
* Ollama on port `11434`

### HPA

The HPA was converted into a Helm template with configurable:

* Minimum replicas
* Maximum replicas
* CPU utilization target

---

## Task 5: Validate the Helm Chart

### 1. Helm Lint

The chart was validated using:

```text
helm lint bankapp/
```

The chart passed Helm lint successfully. The icon message was only an informational recommendation.

### 2. Helm Template

The chart was rendered locally using:

```text
helm template my-bankapp bankapp/
```

The templates rendered successfully.

### 3. Test Helm Overrides

Helm values were overridden during rendering to verify that the chart was actually configurable.

The BankApp image tag was changed, the replica count was changed, and Ollama was disabled.

The rendered configuration reflected these overrides correctly, including removing the Ollama resources and its dependency from the BankApp Deployment.

### 4. Dry Run

A Helm installation dry run was performed to validate the complete release without creating Kubernetes resources.

The dry run completed successfully.

### 5. Install on Kind

The chart was installed into the `bankapp` namespace on the Kind cluster.

Because the Kind cluster already had the `standard` StorageClass, StorageClass creation was disabled and the persistent resources were configured to use `standard`.

The Helm release was successfully deployed.

The deployed resources included:

* BankApp Deployment
* MySQL Deployment
* Ollama Deployment
* BankApp Service
* MySQL Service
* Ollama Service
* HPA
* ConfigMap
* Secret
* MySQL PVC
* Ollama PVC

The BankApp, MySQL, and Ollama workloads reached the Running/Ready state.

The persistent volume claims for MySQL and Ollama were successfully bound.

---

## Task 6: Verify Application Access

The BankApp service was exposed locally through Kubernetes port forwarding.

The correct Helm-generated service name was:

```text
my-bankapp-service
```

The application was accessed through port `8081` on the EC2 host.

The BankApp application opened successfully in the browser.

This confirmed that the Helm chart successfully deployed the application and its required dependencies.

---

## Documentation

### Raw Kubernetes YAML vs Helm Chart

| Raw Kubernetes YAML                             | Helm Chart                              |
| ----------------------------------------------- | --------------------------------------- |
| Multiple separate YAML files                    | One reusable chart                      |
| Values are mostly hardcoded                     | Values are configurable                 |
| Manual changes are required                     | Values can be overridden                |
| Reuse is limited                                | Same chart can be reused                |
| Resource names are manually defined             | Names can be generated from the release |
| Configuration and deployment are separate files | Templates and values work together      |

### Important Helm Template Concepts Used

| Helm concept                         | Purpose                                       |
| ------------------------------------ | --------------------------------------------- |
| `{{ .Values.x }}`                    | Reads a value from `values.yaml`              |
| `{{ .Release.Name }}`                | Gets the Helm release name                    |
| `{{ include "bankapp.fullname" . }}` | Generates a consistent resource name          |
| `{{ if ... }}`                       | Conditionally creates resources/configuration |
| `{{ default ... }}`                  | Provides a default value                      |
| `{{ b64enc }}`                       | Base64-encodes secret values                  |
| `{{ quote }}`                        | Adds quotes around a rendered value           |
| `{{-` / `-}}`                        | Controls template whitespace                  |

### Helm Chart Behavior

The chart was tested with different values to confirm that it behaved dynamically.

For example, disabling Ollama removed the Ollama Deployment, Service, PVC, and the BankApp dependency on Ollama from the rendered configuration.

Changing the BankApp image tag also changed the image used by the generated Deployment.

This demonstrated the main benefit of Helm: the same Kubernetes application can be deployed with different configurations without manually editing multiple YAML files.

### Cleanup

After verification, the Helm release was removed from the Kind cluster using Helm.

The Day 79 custom Helm chart work was completed with the application successfully deployed, validated, accessed, and cleaned up.

