# Day 89 -- Production AI Agents: KubeHealer and AIOps

## Task

You have built agents that diagnose problems. Today you build one that fixes them -- autonomously. KubeHealer is a production-grade AI agent that scans your Kubernetes cluster for broken pods, diagnoses the root cause, proposes fixes, and applies approved changes. It uses Temporal for durable execution, so a workflow can resume after a worker crashes.

This is AIOps -- AI-powered operations. Not a chatbot. A system that watches, reasons, and acts.

Reference: https://github.com/TrainWithShubham/agentic-ai-for-devops -- modules 4, 5  
KubeHealer repo: https://github.com/TrainWithShubham/kubehealer

---

## project

[AI stack](https://github.com/Mujakkir-Pathan/kubehealer)

---

## All screenshot's

[screenshots of AI-BankApp-DevOps](screenshots/)


---

## Challenge Tasks

### Task 1: Understand AIOps and Production Guardrails (Module 4)

Before building production agents, understand the rules:

1. **What is AIOps?**
   - Using AI to help automate IT operations: monitoring, diagnosis, and remediation.
   - Augmenting humans with intelligent automation rather than replacing human judgment.
   - Handling routine issues, such as image typos and resource limits, while escalating issues that need a human decision.

2. **Production guardrails every AI agent needs:**

| **Guardrail** | **Why** | **Example** |
| --- | --- | --- |
| **Human approval** | Agents should not make risky changes without permission. | Present proposed fixes and wait for approval. |
| **Scope limits** | Agents should operate only in allowed namespaces and clusters. | Prevent access to `kube-system` or production databases. |
| **Audit trail** | Every action and decision should be recorded. | Review the Temporal workflow history. |
| **Rollback capability** | Changes should be reversible where possible. | Use targeted patches and retain a rollback plan. |
| **Timeout and retry limits** | Agents must not loop forever. | Limit retries and set activity timeouts. |
| **Escalation path** | Unresolved issues need human attention. | Report that a missing ConfigMap requires a human decision. |

3. **Why durable execution (Temporal) matters:**
   - Without durability, a worker crash can interrupt work and make it difficult to know which steps have already completed.
   - Temporal records workflow history. When a worker restarts, it can replay the workflow from that history and continue pending work.
   - This matters for infrastructure automation because repeated or partially completed changes can cause problems.

4. **When to use AI agents vs traditional automation:**

| **Use AI Agents When** | **Use Traditional Automation When** |
| --- | --- |
| A problem needs reasoning, such as diagnosing an unknown error. | The problem has a known, fixed solution. |
| There are multiple possible causes and fixes. | One condition maps to one predictable action. |
| A human benefits from a natural-language explanation. | A deterministic script can safely perform the task. |
| Examples: troubleshooting and root-cause analysis. | Examples: scheduled scaling, known restarts, and standard deployments. |

**What I learned:** AIOps combines investigation and operational actions with guardrails. AI should not be given unrestricted access to infrastructure.

---

### Task 2: Set Up KubeHealer

KubeHealer lives in a separate repository. It was cloned and set up locally:

```bash
git clone https://github.com/TrainWithShubham/kubehealer.git
cd kubehealer
```

**Prerequisites used:**

- Docker
- Kind
- Python 3.14.4
- Temporal CLI 1.9.1
- Ollama with Gemma2
- A working Kubernetes cluster

The original task uses Claude Sonnet 4 through an Anthropic API key. The API account used for this lab had no available credits, so Ollama Gemma2 was used instead. Therefore, this run was not powered by Claude.

**Start the infrastructure:**

1. A Kind cluster was created:

```bash
kind create cluster --name kubehealer-demo
```

2. Temporal was started in a separate terminal:

```bash
temporal server start-dev
```

The Temporal UI was available at `http://localhost:8233`.

3. The Python environment was created and dependencies installed:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

4. The worker confirmed the local model and Kubernetes connection:

```text
[OK] Ollama Gemma2
[OK] Kubernetes cluster

KubeHealer worker started (Ollama Gemma2). Waiting for tasks...
```

**Architecture used:**

- **Ollama Gemma2:** local language model used for diagnosis instead of Claude.
- **KubeHealer worker:** runs the workflow activities and Kubernetes tools.
- **kubectl:** reads Kubernetes state and applies supported changes.
- **Temporal:** records workflow progress and coordinates durable execution.
- **KubeHealer starter:** starts a healing workflow.

---

### Task 3: Deploy Broken Applications

KubeHealer needs something to diagnose and fix. The repository's `chaos/` manifests were applied:

```bash
kubectl apply -f chaos/
```

Output:

```text
deployment.apps/web-app configured
deployment.apps/config-app unchanged
deployment.apps/memory-hog unchanged
```

The repository's manifests used a Deployment named `memory-hog` rather than the original task's standalone Pod named `memory-app`.

**The three broken applications:**

| Application | Problem | Expected diagnosis |
| --- | --- | --- |
| `web-app` | Invalid Nginx image name, causing `ImagePullBackOff`. | Correct the image name to `nginx:latest`. |
| `memory-hog` | Memory limit too low, causing an out-of-memory termination. | Increase the memory limit, subject to approval. |
| `config-app` | References a ConfigMap named `app-config` that does not exist. | Escalate for human attention rather than inventing a ConfigMap. |

Initial pod status showed the problems:

```text
config-app-69cbb89568-hc7gm   0/1   CreateContainerConfigError
memory-hog-6b7985c6b7-lp94s   0/1   CrashLoopBackOff
web-app-5dc9679784-g4z7k      0/1   ImagePullBackOff
```

The memory workload later showed `OOMKilled`, which matched the memory-limit issue.

---

### Task 4: Run KubeHealer

The Temporal worker was started with:

```bash
python3 worker.py
```

A healing workflow was started in another terminal. Manual approval was used:

```bash
python3 starter.py --manual
```

The workflow used Ollama Gemma2 because the Anthropic API account had no available credits.

**Diagnosis and decisions:**

| Application | Agent diagnosis / observed issue | Decision and result |
| --- | --- | --- |
| `web-app` | Image pull failure due to an invalid image name. | Approved. The image was patched to `nginx:latest`, and the replacement pod reached `1/1 Running`. |
| `memory-hog` | Memory-related crash / `OOMKilled`. | Rejected. The workload remained unhealthy. |
| `config-app` | Missing ConfigMap reference. | Rejected. The workload remained in `CreateContainerConfigError`. |

The workflow waited for a decision for each diagnosed pod. Decisions were sent using Temporal signals. The workflow then completed.

Verification command:

```bash
kubectl get pods
```

Observed result after the workflow:

```text
config-app-69cbb89568-hc7gm   0/1   CreateContainerConfigError
memory-hog-6b7985c6b7-lp94s   0/1   OOMKilled
web-app-7b4658b994-7cvx9      1/1   Running
```

**Result:** The approved `web-app` fix worked. The other two workloads stayed broken because their proposed fixes were rejected. This demonstrated that the agent did not apply every possible change automatically.

---

### Task 5: Test Crash Recovery (Temporal Durability)

The broken applications were redeployed:

```bash
kubectl apply -f chaos/
```

The worker was started in the background and a new manual workflow was triggered:

```bash
python3 worker.py &
python3 starter.py --manual
```

The workflow ID was:

```text
kubehealer-1792330594
```

The worker process was terminated during the workflow, before sending approval signals:

```bash
kill 45849
```

The worker was then restarted:

```bash
python3 worker.py &
```

The existing workflow was inspected in the Temporal UI:

`http://localhost:8233`

The workflow was checked after the worker restarted, and the diagnosis/recovery stage was confirmed in the UI.

**What this demonstrated:**

1. The worker process can stop while a workflow is in progress.
2. Temporal retains the workflow's event history.
3. A restarted worker can pick up workflow tasks for the existing execution.
4. Completed activities can be replayed from history rather than rerun as new work.
5. The Temporal UI provides an audit trail of workflow events and activity results.

**Important:** Temporal preserves workflow progress and coordinates recovery. It does not itself diagnose or fix Kubernetes problems; that logic lives in KubeHealer's workflow and activities.

---

### Task 6: Reflect on the Agentic AI Journey

Map the 3-day progression:

| **Day** | **Module** | **What You Built** | **Pattern** |
| --- | --- | --- | --- |
| 87 | 0-2 | Docker Error Explainer + Docker Agent | Basic LLM -> ReAct Agent |
| 88 | 3, 6 | Multi-tool Agent + MCP Server + CI/CD Analyzer | Multi-domain tools, MCP protocol |
| 89 | 4-5 | KubeHealer -- production self-healing agent | Temporal durability, human approval, guardrails |

**The evolution:**

```text
Day 87: LLM explains errors (passive)
   |
Day 88: Agent diagnoses across Docker/K8s/CI (autonomous investigation)
   |
Day 89: Agent diagnoses AND fixes with approval (autonomous action)
```

**Key principles for production AI agents:**

1. **Tools are just CLI wrappers** -- commands such as `docker` and `kubectl` can be exposed through programmatic tools.
2. **The ReAct pattern is universal** -- agents can reason, act through tools, observe results, and decide what to do next.
3. **MCP standardizes tool access** -- it provides a common protocol for connecting AI applications with tools.
4. **Guardrails are not optional** -- use approval, scope limits, and audit trails.
5. **Durability matters** -- Temporal helps workflows recover after worker interruptions.
6. **Know when NOT to use AI** -- simple if/then automation is better for known, predictable problems.

**Where this connects to the rest of the challenge:**

| **Day** | **Connection to Agentic AI** |
| --- | --- |
| 29-37 (Docker) | Docker tools in Module 2 wrap the same commands learned earlier. |
| 40-49 (GitHub Actions) | The CI/CD Analyzer in Module 6 diagnoses pipeline failures. |
| 50-67 (Kubernetes) | Kubernetes tools in Module 3 and KubeHealer use `kubectl`. |
| 73-77 (Observability) | Agents could query Prometheus and Loki for metric-based diagnosis. |
| 84-86 (ArgoCD) | An agent could trigger ArgoCD syncs or rollbacks with appropriate permissions and approval. |

**What I learned:** The three days progressed from explaining errors, to investigating them with tools, to taking guarded action. Reliable production agents need more than a capable model: they also need limited permissions, human oversight, recorded decisions, and recovery from interruptions.

**Clean up:**

The Kind cluster was deleted:

```bash
kind delete cluster --name kubehealer-demo
```

Output:

```text
Deleting cluster "kubehealer-demo" ...
Deleted nodes: ["kubehealer-demo-control-plane"]
```

The Temporal development server was stopped with `Ctrl+C` in its terminal. The Python virtual environment can be exited with:

```bash
deactivate
```

---

## AIOps, KubeHealer, and the 90-Day Challenge

### 1. AIOps Principles and the Six Production Guardrails

**AIOps** applies AI to IT operations to help detect problems, investigate causes, recommend actions, and, where permitted, remediate incidents. A production AIOps system should support engineers rather than blindly make changes.

**Core principles**
- **Observe before acting:** collect pod status, events, logs, and other evidence before proposing a fix.
- **Diagnose with evidence:** connect the proposed cause to the actual Kubernetes error instead of guessing.
- **Use bounded actions:** allow only specific operations within a defined namespace and resource scope.
- **Keep humans in control:** require approval for risky or production-impacting changes.
- **Record decisions:** preserve workflow history, tool results, approvals, and actions for review.
- **Recover safely:** design workflows to continue after worker interruptions without losing recorded execution history.

**Six production guardrails**
1. **Human approval:** pause before applying changes that require a person to approve them.
2. **Namespace and resource scope:** restrict the agent to the intended cluster, namespace, and resources.
3. **Tool and command allowlist:** expose only required tools and permitted operations; avoid arbitrary shell commands.
4. **Least-privilege access:** use Kubernetes RBAC so the agent has only the permissions it needs.
5. **Timeouts, retries, and limits:** prevent unbounded retries, repeated restarts, or actions that continue indefinitely.
6. **Audit trail and verification:** record diagnoses, approvals, and changes, then verify the resulting workload state.

These guardrails reduce risk but cannot guarantee that every diagnosis or remediation is correct. Production use still needs testing, monitoring, and a rollback plan.

### 2. KubeHealer Architecture: Temporal + Claude + kubectl

```text
Kubernetes cluster
       |
       | pod status, events, logs
       v
KubeHealer workflow
       |
       +--> Temporal
       |      Records workflow history and coordinates execution
       |
       +--> AI diagnosis
       |      Claude in the intended setup
       |      Interprets evidence and suggests a cause and remediation
       |
       +--> Human approval
       |      Approves or rejects the proposed action
       |
       +--> kubectl tools
       |      Read cluster state and apply permitted changes
       |
       v
Verify workload status and record the outcome
```

- **Temporal:** manages durable workflow execution and records workflow history. If a worker stops, the workflow can continue when a worker returns.
- **Claude:** the intended model for analyzing Kubernetes evidence and proposing a diagnosis or fix.
- **`kubectl`:** reads Kubernetes state and carries out allowed cluster operations.
- **Human approval:** provides a checkpoint before a proposed change is applied.
- **Verification:** checks whether the workload recovered after the action.

**Actual lab implementation:** Claude was not used because the Anthropic API account had no available credits. The lab used **Ollama Gemma2** instead. Therefore, the tested architecture was Temporal + Ollama Gemma2 + `kubectl`, with approval for proposed changes. Claude is documented as the intended model, not the model used in this run.

### 3. The Three Broken Applications and the Agent's Diagnosis

The lab applied manifests from the repository's `chaos/` directory.

| Workload | Observed symptom | Diagnosis indicated by the evidence | Lab outcome |
| --- | --- | --- | --- |
| `config-app` | `CreateContainerConfigError` | The container could not start because a required configuration reference was unavailable or incorrect. Pod events should be checked to confirm the exact missing configuration. | Proposed fix was rejected, so the workload remained unhealthy. |
| `memory-hog` | `CrashLoopBackOff`, later `OOMKilled` | The container exceeded its memory limit and was terminated by the system. | Proposed fix was rejected, so the workload remained unhealthy. |
| `web-app` | `ImagePullBackOff` | Kubernetes could not pull the configured container image; the image reference or availability needed correction. | The proposed image patch was approved, and the replacement pod reached `1/1 Running`. |

Not all recommendations were applied. The `web-app` change was approved and verified as successful. The `config-app` and `memory-hog` changes were deliberately rejected. A diagnosis or proposed fix is not the same as a verified recovery.

### 4. How Crash Recovery Works: Stop Worker, Restart, Resume

Temporal separates durable workflow execution from the worker process that runs workflow and activity code.

1. Start the KubeHealer workflow and allow it to record progress in Temporal.
2. Stop the worker process while the workflow is in progress.
3. Temporal retains the workflow's event history; stopping the worker does not erase that history.
4. Restart the worker, for example with `python3 worker.py &`.
5. The worker polls Temporal for tasks and can continue processing the existing workflow execution.
6. Inspect the workflow in the Temporal UI at `http://localhost:8233` and check the recorded workflow stage and activity history.

This demonstrates workflow durability: Temporal can coordinate continuation after a worker interruption. It does **not** automatically repair Kubernetes, and it does not make every external action exactly-once. Activities that interact with external systems should tolerate retries and avoid unsafe duplicate changes.

### 5. When to Use AI Agents vs. Traditional Automation

| Situation | Better fit | Why |
| --- | --- | --- |
| A known error has one deterministic fix | Traditional automation | A tested script or rule is simpler and more predictable. |
| A fixed sequence of deployment steps must run every time | Traditional automation / CI/CD | The workflow is known in advance and should be repeatable. |
| The cause is unclear and evidence must be compared across logs, events, and tools | AI agent with read-only investigation | The agent can interpret different evidence sources and propose likely causes. |
| Several possible fixes have different risks | AI agent with human approval | The agent can explain options while a person controls consequential changes. |
| A change is high-risk or affects production availability | Human-led process with guarded automation | Approval, testing, rollback, and clear ownership are essential. |

**Rule of thumb:** use traditional automation for predictable, repeatable tasks. Consider an AI agent when investigation requires interpreting ambiguous evidence across multiple tools. Keep permissions limited and require approval for risky changes.

### 6. How Agentic AI Connects to the Other Topics in the 90-Day Challenge

| Challenge topic | Connection to agentic AI |
| --- | --- |
| Linux and shell | Agents can use carefully scoped commands and inspect system output. |
| Git and GitHub | Agent changes can be tracked, reviewed, and version-controlled. |
| Docker | Docker CLI tools help an agent inspect containers, images, and logs. |
| CI/CD and GitHub Actions | A CI/CD analyzer can inspect failed jobs and logs, identify likely causes, and suggest a fix. |
| Kubernetes | `kubectl` gives an agent controlled access to workload status, events, logs, and approved operations. |
| Terraform and infrastructure as code | An agent may explain plan output or suggest configuration changes; infrastructure changes should be reviewed and approved. |
| Ansible and configuration management | Predictable configuration tasks are usually best handled by playbooks, with AI helping investigate unexpected failures. |
| Monitoring and observability | Metrics, logs, and alerts provide evidence for diagnosis and help verify remediation. |
| Helm | Agents can help interpret release status and chart configuration; changes should be validated before deployment. |
| ArgoCD and GitOps | Agents can explain drift or sync failures and recommend changes through Git; Git remains the source of truth. |
| Security and DevSecOps | Least privilege, secret protection, policy checks, and audit logs constrain agent actions. |
| Agentic AI, MCP, and workflows | Tools connect the model to operations, MCP standardizes tool access, and Temporal supports durable multi-step workflows. |

Across the challenge, the progression is from learning infrastructure tools, to automating repeatable work, to using an agent to investigate problems and propose guarded actions. Agents should build on existing DevOps practices, not replace version control, testing, observability, security controls, or human accountability.

---

## Final Checklist

- [x] AIOps principles and the six production guardrails.
- [x] KubeHealer architecture: Temporal + Ollama Gemma2 + `kubectl`.
- [x] Three broken applications and the issues diagnosed for each.
- [x] Crash recovery tested by stopping and restarting the worker and checking the existing workflow in Temporal.
- [x] When to use AI agents versus traditional automation.
- [x] How agentic AI connects to other topics in the 90-day challenge.
- [x] Kind cluster deleted.

**Note:** This lab used Ollama Gemma2 instead of Claude because the Anthropic API account had no available credits. The web application fix was verified as successful; the memory and ConfigMap fixes were deliberately rejected and those workloads remained unhealthy.
