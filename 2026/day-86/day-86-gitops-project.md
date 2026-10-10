# Day 86 -- GitOps Project: End-to-End CI/CD Pipeline with AI-BankApp

## Task

Two days of ArgoCD -- setup, self-healing, sync strategies, App of Apps, and RBAC. Today you wire the complete GitOps pipeline end to end. A developer pushes a code change, GitHub Actions builds and pushes the Docker image, updates the Kubernetes manifest in Git, and ArgoCD automatically deploys the new version to EKS. Zero human intervention from `git push` to production.

This is the AI-BankApp's actual production workflow from `.github/workflows/gitops-ci.yml`.

Reference: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch: `feat/gitops`)

---

## project

[AI-BankApp-DevOps](https://github.com/Mujakkir-Pathan/devboard/tree/feat/k8s)

---

## All screenshot's

[screenshots of AI-BankApp-DevOps](screenshots/)

---

## Challenge Tasks

### Task 1: Study the AI-BankApp's GitOps CI Pipeline

Open `.github/workflows/gitops-ci.yml` from the AI-BankApp repo. This is a production-grade GitOps CI pipeline.

**The workflow triggers on:**

```yaml
on:
  push:
    branches: [feat/gitops]
    paths:
      - 'src/**'
      - 'pom.xml'
      - 'Dockerfile'
  workflow_dispatch:
```

It only runs when application code changes (`src/`, `pom.xml`, `Dockerfile`) -- not when Kubernetes manifests change. This prevents infinite loops since the pipeline itself updates manifests.

**The pipeline steps:**

| **Step**                | **What it does**                                                   |
| ----------------------- | ------------------------------------------------------------------ |
| Checkout code           | Clones the repo                                                    |
| Set up JDK 21           | Installs Java 21 with Maven cache                                  |
| Build with Maven        | `./mvnw clean package -DskipTests -B`                              |
| Run tests               | `./mvnw test -B` (non-blocking: `continue-on-error: true`)         |
| Set image tag           | Uses `git rev-parse --short HEAD` as the tag (e.g., `1c7cb0e`)     |
| Login to DockerHub      | Authenticates with secrets                                         |
| Build and push image    | Pushes `trainwithshubham/ai-bankapp-eks:latest` and `:sha`         |
| Update K8s manifest     | Uses `sed` to update the image tag in `k8s/bankapp-deployment.yml` |
| Commit updated manifest | Commits the change with `[skip ci]` to avoid re-triggering         |

**The critical GitOps step** is the last two:

```yaml
- name: Update Kubernetes deployment manifest
  run: |
    sed -i "s|image: ${{ env.DOCKERHUB_REPO }}:.*|image: ${{ env.DOCKERHUB_REPO }}:${{ steps.tag.outputs.sha_short }}|" k8s/bankapp-deployment.yml

- name: Commit updated manifest
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add k8s/bankapp-deployment.yml
    git diff --staged --quiet || git commit -m "ci: update bankapp image to ${{ steps.tag.outputs.sha_short }} [skip ci]"
    git push
```

**Why `[skip ci]`?** Without it, the commit that updates the manifest would trigger the pipeline again, which would update the manifest again -- an infinite loop. `[skip ci]` tells GitHub Actions to ignore this commit.

**The handoff to ArgoCD:**

```text
GitHub Actions commits new image tag to k8s/bankapp-deployment.yml
         |
    ArgoCD detects the new commit (within 3 minutes)
         |
    ArgoCD compares: cluster has old image, Git has new image
         |
    ArgoCD syncs: performs a rolling update
         |
    New pods start with the new image, old pods terminate
         |
    Zero downtime deployment complete
```

**What I learned / completed:**
- The workflow is triggered by application-code changes on `feat/gitops`, while manifest-only commits are excluded to prevent repeated builds.
- The workflow builds the Spring Boot application, runs tests, builds and pushes a Docker image, updates the Kubernetes manifest with the commit SHA, and pushes that manifest change back to Git.
- In this fork, the image repository was configured as `mujakkirpathan/ai-bankapp-eks`.

---

### Task 2: Set Up the Pipeline on Your Fork

To run the full pipeline, you need your own fork with GitHub Secrets.

**1. Fork the repo** (if not done on Day 84):

```text
https://github.com/TrainWithShubham/AI-BankApp-DevOps -> Fork
```

**2. Create a DockerHub access token:**

- Go to [https://hub.docker.com/settings/security](https://hub.docker.com/settings/security)
- Create a new access token with Read/Write permissions
- Note the token

**3. Add GitHub Secrets to your fork:**

- Go to your fork > Settings > Secrets and variables > Actions
- Add these secrets:
  - `DOCKERHUB_USERNAME` -- your DockerHub username
  - `DOCKERHUB_TOKEN` -- the access token from step 2

**4. Update the workflow to push to your DockerHub repo:** Edit `.github/workflows/gitops-ci.yml` in your fork:

```yaml
env:
  DOCKERHUB_REPO: mujakkirpathan/ai-bankapp-eks
```

**5. Update the ArgoCD Application to watch your fork:**

```bash
argocd app set bankapp --repo https://github.com/Mujakkir-Pathan/AI-BankApp-DevOps.git
```

**6. Update the Kubernetes deployment to pull from your DockerHub:** Edit `k8s/bankapp-deployment.yml`:

```yaml
image: mujakkirpathan/ai-bankapp-eks:latest
```

Commit and push all changes to your fork's `feat/gitops` branch.

**What I configured:**
- GitHub Actions secrets `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`.
- `DOCKERHUB_REPO` as `mujakkirpathan/ai-bankapp-eks`.
- ArgoCD source repository as `https://github.com/Mujakkir-Pathan/AI-BankApp-DevOps.git`, branch `feat/gitops`, path `k8s`.

---

### Task 3: Trigger the Full Pipeline

Make a visible code change in the application. Edit a file in `src/`:

For example, edit `src/main/resources/templates/fragments/layout.html` -- change the page title or footer text to include your name:

```html
<!-- Find the title or footer and add your touch -->
<title>AI BankApp - Built by YourName</title>
```

Commit and push:

```bash
git add src/
git commit -m "feat: customize app title"
git push origin feat/gitops
```

**Watch the pipeline:**

1. Go to your fork > Actions tab
2. The "GitOps CI - Build & Push to DockerHub" workflow should be running
3. Watch each step: build -> test -> push -> update manifest -> commit

**After the pipeline completes:**

- Check the last commit on your `feat/gitops` branch -- you should see a commit from `github-actions[bot]` with the message `ci: update bankapp image to <sha> [skip ci]`
- The `k8s/bankapp-deployment.yml` file now has the new image tag

**Watch ArgoCD sync:**

```bash
argocd app get bankapp --refresh
argocd app wait bankapp
```

Or watch in the ArgoCD UI -- you will see a new sync event with the updated revision.

Check the pods:

```bash
kubectl get pods -n bankapp -w
```

You should see a rolling update -- new pods starting with the new image while old pods terminate gracefully.

Verify the change is live:

```bash
kubectl port-forward svc/bankapp-service -n bankapp 8080:8080
```

Open `http://localhost:8080` and confirm your title change is visible.

**You just completed a full GitOps cycle:** code change -> CI builds image -> updates manifest -> ArgoCD deploys to production. Zero manual intervention.

**Actual result:**
- Changed the footer in `src/main/resources/templates/fragments/layout.html` to `© 2026 BankApp. Built with Spring Boot. Built by Mujakkir Pathan.`
- Committed the code change as `67da204` (`changed footer as per the task 86`) and pushed it to `feat/gitops`.
- GitHub Actions completed successfully, built and pushed the Docker image tagged `67da204`, and created manifest-update commit `fe4260d` (`ci: update bankapp image to 67da204 [skip ci]`).
- ArgoCD synchronized revision `fe4260d`; the deployment used `mujakkirpathan/ai-bankapp-eks:67da204`.
- Verified the updated footer in the browser.

---

### Task 4: Test Drift Detection and Recovery

GitOps means the cluster must always match Git. Test what happens when someone makes unauthorized changes.

**Scenario 1 -- Someone scales down the app directly:**

```bash
kubectl scale deployment bankapp -n bankapp --replicas=1
```

Check ArgoCD:

```bash
argocd app get bankapp
```

Status should show `OutOfSync`. With `selfHeal: true`, ArgoCD will correct it within 3 minutes. Monitor:

```bash
kubectl get pods -n bankapp -w
```

The replica count will return to 4 (or whatever the manifest specifies).

**Scenario 2 -- Someone updates the image tag directly:**

```bash
kubectl set image deployment/bankapp bankapp=nginx:latest -n bankapp
```

ArgoCD detects the drift and reverts it to the image tag from Git. The BankApp pods restart with the correct image.

**Scenario 3 -- Someone deletes a critical resource:**

```bash
kubectl delete service bankapp-service -n bankapp
```

ArgoCD recreates it from Git.

**View all drift events:**

```bash
argocd app history bankapp
```

In the ArgoCD UI, click the application and look at the "Events" tab. Every self-heal action is logged with the before/after state.

**Document:** In each scenario, how long did ArgoCD take to detect and fix the drift? What would happen if `selfHeal` was disabled?

**Observed results:**
- **Scenario 1 — Scale deployment:** Scaled `bankapp` to one replica. The later deployment check showed two replicas, but the HPA was configured with a minimum of two replicas. Because HPA could restore the replica count independently, this test did not isolate ArgoCD self-healing.
- **Scenario 2 — Change image:** Set the deployment image to `nginx:latest`. A subsequent image check showed `mujakkirpathan/ai-bankapp-eks:67da204`, indicating the desired image was restored. The transient `OutOfSync` state and exact recovery time were not captured.
- **Scenario 3 — Delete Service:** Deleted `bankapp-service`; a check about 59 seconds later showed the Service present again.
- **Timing:** Exact detection/recovery timings were not recorded for every scenario.
- **If `selfHeal` were disabled:** ArgoCD would still detect differences during refresh, but automated synchronization would not automatically correct live-cluster changes solely because of drift. A manual sync or another configured sync trigger would be needed to restore the Git-defined state.

---

### Task 5: Reflect on the Complete DevOps Pipeline

Step back and look at everything you have built across the entire 90-day challenge that connects to this GitOps pipeline:

```text
[Developer writes code]
    |
[Git push to GitHub]  ........... Day 22-28: Git & GitHub
    |
[GitHub Actions CI]   ........... Day 40-49: GitHub Actions
    |-- Build with Maven
    |-- Run tests
    |-- Build Docker image  ..... Day 29-37: Docker
    |-- Push to DockerHub
    |-- Update K8s manifest
    |-- Commit back to Git
    |
[ArgoCD detects change] ........ Day 84-86: GitOps
    |
[ArgoCD syncs to EKS]  ........ Day 81-83: EKS
    |-- Rolling update
    |-- Health checks pass
    |-- HPA scales as needed ... Day 78-80: Helm (HPA, values)
    |
[Prometheus scrapes metrics] ... Day 73-77: Observability
    |-- Grafana dashboards
    |-- Alerts if something breaks
    |
[App is live with zero downtime]
```

Every block in this challenge connects to the next. This is what a DevOps pipeline looks like in production.

**Complete DevOps pipeline map:**

```text
Developer writes code
        |
        v
Git push to GitHub (Days 22-28)
        |
        v
GitHub Actions CI (Days 40-49)
  |-- Build with Maven and run tests
  |-- Build Docker image (Days 29-37)
  |-- Push image to DockerHub
  |-- Update Kubernetes manifest with image SHA
  |-- Commit manifest update to Git
        |
        v
ArgoCD detects Git change (Days 84-86)
        |
        v
ArgoCD syncs desired state to EKS (Days 81-83)
  |-- Rolling update and health checks
  |-- HPA manages replicas (Days 78-80)
        |
        v
Prometheus / Grafana / alerts (Days 73-77)
        |
        v
Application available to users
```

**Reflection:** The pipeline separates responsibilities: DockerHub stores the built image, Git stores the source code and desired Kubernetes configuration, and ArgoCD reconciles the EKS cluster with that configuration. The image tag in Git connects a specific running version to the source commit that built it.

---

### Task 6: Complete Teardown

**Delete everything. This is the end of the EKS and ArgoCD block.**

Delete ArgoCD applications:

```bash
argocd app delete bankapp --cascade -y
argocd app delete monitoring --cascade -y 2>/dev/null
argocd app delete envoy-gateway --cascade -y 2>/dev/null
argocd app delete root-app --cascade -y 2>/dev/null
```

The `--cascade` flag tells ArgoCD to delete all Kubernetes resources managed by each application.

Wait for cleanup:

```bash
kubectl get all -n bankapp 2>/dev/null
kubectl get all -n monitoring 2>/dev/null
```

**Destroy the EKS cluster with Terraform:**

```bash
cd AI-BankApp-DevOps/terraform
terraform destroy
```

Confirm deletion (type `yes`). This takes 10-15 minutes.

**Verify in the AWS Console:**

- EKS: no clusters
- EC2: no instances, no load balancers, no EBS volumes
- VPC: the `bankapp-eks` VPC is gone
- IAM: clean up roles with `eksctl` or `bankapp-eks` in the name

**Final cost check:** Review AWS Billing Dashboard. All EKS charges should stop within the hour.

**Teardown status:** ArgoCD application deletion was attempted, but the App of Apps/root-app and child application cleanup became stuck while resources were still being reconciled. Terraform destroy was then discussed, but no final Terraform destroy output or AWS-console verification was recorded here. Therefore, complete teardown remains **unverified**.

**Map the 3-day ArgoCD journey:**

| **Day** | **What You Built**                                                 |
| ------- | ------------------------------------------------------------------ |
| 84      | ArgoCD setup, first GitOps deploy, self-healing                    |
| 85      | Sync waves, rollbacks, App of Apps, notifications, RBAC            |
| 86      | Full CI/CD pipeline, code-to-production, drift detection, teardown |

---

## Hints

- `[skip ci]` in the commit message prevents infinite pipeline loops -- the manifest update commit must not trigger a new build
- The `sed` command in the pipeline updates the image tag in-place. In a Helm-based GitOps setup, you would update `values.yaml` instead
- ArgoCD webhook integration (Settings > Repositories > webhook) gives instant sync instead of waiting 3 minutes
- `--cascade` on `argocd app delete` is critical -- without it, ArgoCD deletes the Application resource but leaves all Kubernetes resources running (and billing)
- If Terraform destroy hangs, check for lingering load balancers or EBS volumes created by Kubernetes that Terraform does not know about
- The `workflow_dispatch` trigger lets you manually re-run the pipeline from the GitHub Actions UI -- useful for debugging
- The image tag uses `git rev-parse --short HEAD` (7 char SHA) -- this gives exact traceability from running pod to git commit
- Reference: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch: `feat/gitops`)

---

# Day 86 — GitOps Project: End-to-End CI/CD Pipeline

## 1. GitOps Pipeline

Here is how my AI-BankApp moves from a code change to a running application:

```text
I update the application code
            |
            v
I push the change to GitHub
            |
            v
GitHub Actions builds the app
and runs the tests
            |
            v
GitHub Actions builds a Docker image
and pushes it to DockerHub
            |
            v
The workflow updates the image tag
in the Kubernetes manifest in Git
            |
            v
The workflow commits the manifest change
            |
            v
ArgoCD notices the new Git commit
            |
            v
ArgoCD syncs the change to Amazon EKS
            |
            v
Kubernetes replaces the old pods
with pods running the new image
```

In simple words, GitHub Actions builds and publishes the image. It then updates the image tag in Git. ArgoCD reads that change and updates the application running in EKS.

In my project, I changed the footer in `src/main/resources/templates/fragments/layout.html` to:

`© 2026 BankApp. Built with Spring Boot. Built by Mujakkir Pathan.`

The code change was committed as `67da204`. GitHub Actions built and pushed `mujakkirpathan/ai-bankapp-eks:67da204`. It then created another commit, `fe4260d`, to update the Kubernetes manifest. ArgoCD synced that change, and I checked the updated footer in the browser.

## 2. GitHub Actions Workflow Explained

The workflow file is `.github/workflows/gitops-ci.yml`. It runs when application files change on the `feat/gitops` branch. It can also be started manually from the GitHub Actions page.

Here is what each step does:

1. **Checkout the code:** GitHub Actions gets the latest code from my repository.
2. **Set up Java 21:** The workflow prepares the Java environment needed by the Spring Boot application.
3. **Build the application:** Maven packages the application using `./mvnw clean package -DskipTests -B`.
4. **Run the tests:** The workflow runs `./mvnw test -B`. The test step is configured to allow the pipeline to continue even if the tests fail.
5. **Create an image tag:** The workflow uses the short Git commit SHA, such as `67da204`, as the image tag. This helps me identify which code version an image contains.
6. **Log in to DockerHub:** The workflow uses the `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` GitHub secrets. The credentials are kept out of the workflow file.
7. **Build and push the image:** The Docker image is built and pushed to `mujakkirpathan/ai-bankapp-eks`.
8. **Update the Kubernetes manifest:** The workflow changes the image tag in `k8s/bankapp-deployment.yml` to the new commit SHA.
9. **Commit the manifest change:** GitHub Actions commits and pushes the updated manifest to my `feat/gitops` branch. The generated commit message includes `[skip ci]`.
10. **Deploy through ArgoCD:** ArgoCD detects the new manifest revision and syncs the desired configuration to EKS.

The workflow is configured to run for application-code changes, rather than Kubernetes-manifest-only changes. The `[skip ci]` marker also helps prevent the generated manifest commit from starting another workflow run.

Each tool has a separate job:

- **GitHub** stores my application code and Kubernetes manifests.
- **GitHub Actions** builds the application and updates the manifest.
- **DockerHub** stores the Docker image.
- **ArgoCD** keeps the Kubernetes cluster in line with the configuration in Git.
- **Amazon EKS** runs the application.

## 3. Drift Detection Results

I tested three changes directly in the Kubernetes cluster to see how the system responded. The goal was to check whether the live cluster returned to the state described in Git.

| Test | What I changed | What I observed |
|---|---|---|
| Scale the deployment | I scaled `bankapp` down to one replica. | A later check showed two replicas. However, the HPA was configured with a minimum of two replicas, so it may have restored the count without ArgoCD. This test did not prove ArgoCD self-healing on its own. |
| Change the image | I changed the image to `nginx:latest`. | A later check showed the original image, `mujakkirpathan/ai-bankapp-eks:67da204`, again. This indicated that the intended image had been restored. |
| Delete the Service | I deleted `bankapp-service`. | About 59 seconds later, I checked again and found the Service had been recreated. |

I did not record exact detection and recovery times for every test, so I cannot give an accurate timing for all three scenarios.

### What if self-healing were disabled?

ArgoCD could still detect differences between Git and the cluster during a refresh. However, it would not automatically sync just to correct live changes. I would need to sync the application manually or rely on another configured sync trigger.

The replica-count test also showed why it is important to understand other Kubernetes components. The HPA can change replica counts independently of ArgoCD.

## 4. Full DevOps Pipeline Map

The skills from earlier days fit together to make this deployment pipeline work:

```text
I write or update the application
            |
            v
Git and GitHub — Days 22–28
I push the code to my repository
            |
            v
GitHub Actions — Days 40–49
I build the app and run the tests
            |
            v
Docker — Days 29–37
I build the image and push it to DockerHub
            |
            v
The workflow updates the Kubernetes manifest
with the new image tag and pushes it to Git
            |
            v
ArgoCD / GitOps — Days 84–86
ArgoCD detects the Git change and syncs it
            |
            v
Amazon EKS — Days 81–83
Kubernetes runs the updated application
and replaces the old pods during the rollout
            |
            v
Helm and HPA — Days 78–80
Deployment configuration and replica scaling
help manage the application
            |
            v
Observability — Days 73–77
Prometheus collects metrics, Grafana displays
dashboards, and alerts help identify problems
            |
            v
The updated application is available to users
```

This helped me understand how each part connects. The image is stored in DockerHub, the desired image version is recorded in Git, and ArgoCD applies that desired state to EKS.

## 5. Key Takeaways from the 3-Day GitOps Block

### Day 84 — Getting Started with ArgoCD

I connected ArgoCD to my Git repository and deployed AI-BankApp through GitOps. I also learned how ArgoCD compares the configuration in Git with what is running in Kubernetes.

### Day 85 — Going Deeper with ArgoCD

I explored sync strategies, sync waves, rollbacks, the App of Apps pattern, notifications, and RBAC. These features help manage deployments and access as an environment grows.

### Day 86 — Connecting CI/CD with GitOps

I connected the code change, image build, manifest update, and ArgoCD deployment into one workflow. I changed the BankApp footer, watched GitHub Actions build and push the image, and verified that ArgoCD deployed the updated version.

I also tested three types of cluster drift. The image was restored, and the deleted Service reappeared. The replica test was less conclusive because the HPA could restore the replica count itself.

### What I learned overall

- Git keeps the desired Kubernetes configuration, so I can track which version should be deployed.
- GitHub Actions builds and publishes the image, while ArgoCD handles deployment to the cluster.
- Using a Git commit SHA as an image tag makes it easier to trace a running version back to its code.
- Self-healing can correct unwanted changes, but I need to consider other Kubernetes controllers when testing it.
- After finishing a cloud lab, I must verify that all resources have been removed. I attempted the ArgoCD cleanup, but I did not capture final Terraform-destroy output or verify the AWS Console, so I cannot confirm that every AWS resource was deleted.

