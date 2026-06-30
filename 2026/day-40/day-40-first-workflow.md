# Day 40 – First GitHub Actions Workflow

## Objective

Today I created my first GitHub Actions workflow and learned how a CI pipeline runs automatically in the cloud.

---

## task 1: Repository Setup

- Created a new public GitHub repository named **github-actions-practice**.
- Cloned the repository to my local machine.
- Created the required workflow directory:

[https://github.com/Mujakkir-Pathan/github-actions-practice](https://github.com/Mujakkir-Pathan/github-actions-practice)

---

## task 2: Created My First Workflow

Created a workflow file:

[https://github.com/Mujakkir-Pathan/github-actions-practice/blob/main/.github/workflows/hello.yml](https://github.com/Mujakkir-Pathan/github-actions-practice/blob/main/.github/workflows/hello.yml)

Configured the workflow to:

- Run on every push to the `main` branch.
- Create one job named `greet`.
- Run on `ubuntu-latest`.
- Check out the repository using `actions/checkout@v4`.
- Print `Hello from GitHub Actions!`.

Committed the changes and pushed them to GitHub.

After pushing the workflow:

- Opened the **Actions** tab.
- Watched my workflow start automatically.
- Verified that the workflow completed successfully.
- Opened the logs of each step to understand what happened during execution.

This was my first successful GitHub Actions pipeline.

[snapshot](screenshots/task2.png)

---

## Task 3: Understand the Anatomy

After creating the workflow, I reviewed each key used in the YAML file to understand its purpose.

### `name`

Defines the name of the workflow or a step. The workflow name is displayed in the GitHub Actions tab, while a step name makes the logs easier to read.

### `on`

Specifies the event that triggers the workflow.

### `jobs`

Defines one or more jobs in the workflow.

### `runs-on`

Specifies the runner (operating system) where the job will execute.

### `steps`

Defines the sequence of actions that the job performs.

Each step runs one after another.

### `uses`

Runs a reusable GitHub Action.

### `run`

Executes shell commands directly on the GitHub runner.

---

## task4 : Added More Workflow Steps

Updated the workflow by adding new steps to:

- Print the current date and time.
- Print the branch name that triggered the workflow.
- List all files in the repository.
- Print the operating system of the GitHub-hosted runner.

Committed the changes and pushed them again.

Verified that all the new steps executed successfully.

[snapshot](screenshots/task4.png)

---

## task 5: Broke the Pipeline on Purpose

To understand pipeline failures, I intentionally added a step that failed using:

Pushed the changes and observed the workflow fail.

Opened the failed step logs and saw:

Learned how GitHub identifies the failed step and displays the error message.

Removed the intentionally failing step.

Committed and pushed the changes again.

Verified that the workflow completed successfully and returned to a green status.

[snapshot](screenshots/task5.1.png)


[snapshot](screenshots/task5.3.png)


---

# What I Learned

- How to create a GitHub Actions workflow.
- How workflows are triggered automatically.
- How jobs and steps execute.
- How to use GitHub-hosted runners.
- How to read workflow logs.
- How to identify and troubleshoot pipeline failures.
- The difference between a YAML syntax error and a command execution failure.

---

## Outcome

Successfully built, executed, broke, debugged, and fixed my first GitHub Actions workflow.

This was my first hands-on experience with a real CI pipeline running in GitHub Actions.
