# Day 48 – GitHub Actions Project: End-to-End CI/CD Pipeline
---

###  [ Devboad Project URL](https://github.com/Mujakkir-Pathan/github-actions-capstone)

###  [All YML files created](https://github.com/Mujakkir-Pathan/github-actions-capstone/tree/main/.github/workflows)

### [Docker Hub Link to backend image](https://hub.docker.com/repository/docker/mujakkirpathan/devboard-backend/general)

### [Docker Hub Link to frontend image](https://hub.docker.com/repository/docker/mujakkirpathan/devboard-frontend/general)


---
## Task

Built a complete production-style GitHub Actions CI/CD pipeline by combining workflows, reusable workflows, Docker, secrets, triggers, deployments, and scheduled jobs into a single project.

## Expected Output

- Created a GitHub repository with a working Dockerized application and a complete CI/CD pipeline.
- Created multiple workflow files that work together using reusable workflows.
- Created `day-48-actions-project.md`.
- Verified the complete pipeline with a successful GitHub Actions run.

---

## Challenge Tasks

### Task 1: Set Up the Project Repo

- Created the `github-actions-capstone` repository.
- Used the Dockerized DevBoard application.
- Added Dockerfiles for the frontend and backend.
- Added application tests.
- Added a `README.md` with the project description.

---

### Task 2: Reusable Workflow — Build & Test

Created `.github/workflows/reusable-build-test.yml`.

Completed the following:

- Configured the `workflow_call` trigger.
- Added inputs for the Go version and test execution.
- Checked out the repository.
- Set up the Go runtime.
- Installed dependencies.
- Ran tests conditionally.
- Set the `test_result` output as `passed` or `failed`.

---

### Task 3: Reusable Workflow — Docker Build & Push

Created `.github/workflows/reusable-docker.yml`.

Completed the following:

- Configured the `workflow_call` trigger.
- Accepted Docker image information as inputs.
- Used Docker Hub credentials through GitHub Secrets.
- Logged in to Docker Hub.
- Built and pushed Docker images.
- Exported the Docker image URL as an output.

---

### Task 4: PR Pipeline

Created `.github/workflows/pr-pipeline.yml`.

Completed the following:

- Triggered the workflow on pull requests to `main`.
- Called the reusable Build & Test workflow.
- Added a `pr-comment` job that runs after successful tests.
- Printed the PR validation summary.
- Prevented Docker image build and push during pull requests.

**Verify:** Open a PR — does it run tests only (no Docker push)?

**Answer:**

Yes. Opening a pull request ran only the Build & Test workflow, and no Docker images were built or pushed.

---

### Task 5: Main Branch Pipeline

Created `.github/workflows/main-pipeline.yml`.

Completed the following:

- Triggered the workflow on pushes to `main`.
- Executed the reusable Build & Test workflow.
- Built and pushed frontend and backend Docker images with `latest` and commit SHA tags.
- Deployed after successful image builds.
- Used the `production` environment for deployment.

**Verify:** Merge a PR to main — does it run tests → build Docker → deploy in sequence?

**Answer:**

Yes. The pipeline executed Build & Test → Docker Build & Push → Deploy successfully in sequence.

---

### Task 6: Scheduled Health Check

Created `.github/workflows/health-check.yml`.

Completed the following:

- Triggered the workflow every 12 hours using Cron.
- Added manual execution using `workflow_dispatch`.
- Pulled the latest Docker image.
- Started the required containers.
- Waited for services to become ready.
- Verified the application health endpoint.
- Stopped and removed the containers.
- Generated a GitHub Actions Job Summary using `$GITHUB_STEP_SUMMARY`.

---

### Task 7: Add Badges & Documentation

Completed the following:

- Added GitHub Actions workflow badges to the repository `README.md`.
- Added a CI/CD pipeline architecture diagram to the project documentation.

Pipeline flow:

- PR opened → Build & Test → PR checks passed
- Merge to main → Build & Test → Docker Build & Push → Deploy
- Every 12 hours → Health Check

**What would you add next?**

**Answer:**

- Slack notifications
- Multi-environment deployments (Development, Staging, Production)
- Automatic rollback on deployment failures
- Image vulnerability scanning
- Deployment notifications

---

## Brownie Points: Add Security to Your Pipeline

Implemented the following DevSecOps improvements:

- Added Trivy image vulnerability scanning after the Docker build.
- Configured the pipeline to fail when CRITICAL vulnerabilities are detected.
- Uploaded the vulnerability scan report as a workflow artifact.
