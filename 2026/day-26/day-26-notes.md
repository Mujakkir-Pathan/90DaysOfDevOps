# Day 26 – GitHub CLI (gh): Manage GitHub from Your Terminal

## Overview

Today I explored the GitHub CLI (`gh`), a powerful tool that allows developers and DevOps engineers to interact with GitHub directly from the terminal. It helps reduce context switching, improves productivity, and enables automation of GitHub workflows.

---

# Task 1: Install and Authenticate

![snapshot](screenshots/task1.png)

## Observation

The GitHub CLI supports multiple authentication methods:

* Browser-based OAuth authentication
* Personal Access Token (PAT)
* GitHub Enterprise authentication
* SSH authentication for Git operations

---

# Task 2: Working with Repositories

![snapshot](screenshots/task2.1.png)
![snapshot](screenshots/task2.2.png)
![snapshot](screenshots/task2.png)

## Observation

Repository creation, management, and navigation can be performed entirely from the terminal without opening GitHub in a browser.

---

# Task 3: Issues

![snapshot](screenshots/task3.1.png)
![snapshot](screenshots/task3.png)

## Answer: How could you use `gh issue` in a script or automation?

The `gh issue` command can be used to:

* Automatically create issues when monitoring tools detect failures.
* Open incident tickets during production outages.
* Generate bug reports from CI/CD pipelines.
* Close resolved issues after successful deployments.
* Build automated maintenance workflows.

---

# Task 4: Pull Requests

![snapshot](screenshots/task4.1.png)
![snapshot](screenshots/task4.png)

### Answer: What merge methods does `gh pr merge` support?

GitHub CLI supports:

* Merge Commit
* Squash Merge
* Rebase Merge

### Answer: How would you review someone else's PR using `gh`?

gh pr view --web

gh pr review <pr-num> --approve

---

# Task 5: GitHub Actions & Workflows (Preview)

![snapshot](screenshots/task5.png)

## Answer: How could `gh run` and `gh workflow` be useful in a CI/CD pipeline?

These commands can help:

* Monitor deployment pipelines from the terminal.
* Check workflow status without opening GitHub.
* Re-run failed workflows quickly.
* Automate deployment approvals.
* Integrate workflow monitoring into scripts.
* Trigger CI/CD workflows programmatically.

---

# Key Learnings

1. GitHub CLI allows complete GitHub repository management from the terminal.
2. Issues, Pull Requests, and Actions workflows can be automated using `gh`.
3. JSON output support makes GitHub CLI ideal for scripting and DevOps automation.
4. GitHub CLI reduces context switching and increases productivity.
5. It integrates well with CI/CD pipelines and infrastructure automation workflows.

---

# Conclusion

GitHub CLI is an essential tool for DevOps engineers and developers. It enables repository management, issue tracking, pull request workflows, and GitHub Actions monitoring directly from the terminal while also providing powerful automation capabilities.

