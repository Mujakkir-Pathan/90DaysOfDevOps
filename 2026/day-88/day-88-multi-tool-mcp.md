# Day 88 -- Multi-Tool Agents, MCP, and CI/CD Analyzer

Reference: https://github.com/TrainWithShubham/agentic-ai-for-devops (modules 3, 6)

Goal: extend the Docker-only agent to Docker + Kubernetes, learn the Model Context Protocol (MCP), and build a CI/CD Failure Analyzer for GitHub Actions.

---

## AI models with upgarded version and matched version of fastmcp

[Agentic ](https://github.com/Mujakkir-Pathan/agentic-ai-for-devops)

---

## All screenshot's

[screenshots of AI-BankApp-DevOps](screenshots/)



---

## Task 1: Multi-Tool DevOps Agent (Module 3)

The Docker agent had 3 tools. Today 3 Kubernetes tools are added to the same agent, giving it 6 tools across 2 domains.

### Setup: Kind cluster with a broken pod

```bash
kind create cluster --name devops-demo
kubectl apply -f module-3/broken_pod.yaml
```

`broken_pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod
  namespace: default
spec:
  containers:
  - name: app
    image: nginx:alpine
    command: ["sh", "-c", "echo 'app starting...' && sleep 2 && exit 1"]
```

### Setup: broken Docker container

```bash
docker run -d --name broken-container nginx:alpine sh -c "echo 'container starting...' && sleep 2 && exit 1"
```

### The 6 tools in `module-3/agent.py`

Docker tools (from Day 87):

- `list_containers()` -- `docker ps -a`
- `get_logs(container_name)` -- `docker logs`
- `inspect_container(container_name)` -- `docker inspect`

Kubernetes tools (new):

```python
@tool
def list_pods(namespace: str = "default") -> str:
    """List all pods in a Kubernetes namespace with their status."""
    result = subprocess.run(
        ["kubectl", "get", "pods", "-n", namespace],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr

@tool
def describe_pod(pod_name: str, namespace: str = "default") -> str:
    """Get detailed info about a Kubernetes pod including events and conditions."""
    result = subprocess.run(
        ["kubectl", "describe", "pod", pod_name, "-n", namespace],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr

@tool
def get_events(namespace: str = "default") -> str:
    """Get recent Kubernetes events in a namespace (useful for troubleshooting)."""
    result = subprocess.run(
        ["kubectl", "get", "events", "-n", namespace, "--sort-by=.lastTimestamp"],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr
```

### Run it

```bash
python3 module-3/agent.py
```

Questions that span both domains:

```
> What's broken across Docker and Kubernetes?
> Why is broken-pod crashing?
> Are there any unhealthy containers on Docker?
> Describe the events in the default namespace
```

The agent picks tools based on the question: Docker question -> Docker tools, pod question -> Kubernetes tools, both -> all of them. This is the ReAct pattern: one agent, many tools, one brain that decides what to use.

---

## Task 2: Understanding the Model Context Protocol (MCP)

MCP is an open standard (created by Anthropic) for connecting AI models to external tools and data sources. Instead of writing tools inside the agent code, you expose them via MCP and any compatible client can use them.

### Why MCP matters for DevOps

| Without MCP                                    | With MCP                                |
| ---------------------------------------------- | --------------------------------------- |
| Tools are locked to one framework (LangChain)  | Tools work with any MCP client          |
| Every AI client re-implements Docker/K8s tools | Write once, use everywhere              |
| Tool access tied to the agent code             | Tools exposed as a discoverable service |

### MCP-compatible clients

- Claude Desktop
- VS Code (GitHub Copilot)
- Cursor
- Claude Code
- Any LangChain agent via `langchain-mcp-adapters`

### Architecture

```
[MCP Server]                    [MCP Clients]
  |                                  |
  |-- list_pods()                    |-- Claude Desktop
  |-- describe_pod()      <--->      |-- VS Code Copilot
  |-- get_events()                   |-- Your Python agent
  |                                  |-- Any MCP client
  |
  (exposes tools via stdio/HTTP)
```

---

## Task 3: Build and Use the MCP Server (Module 3)

### `module-3/mcp_server.py`

```python
import subprocess
from fastmcp import FastMCP

mcp = FastMCP("Kubernetes Tools")

@mcp.tool
def list_pods(namespace: str = "default") -> str:
    """List all pods in a Kubernetes namespace with their status."""
    result = subprocess.run(
        ["kubectl", "get", "pods", "-n", namespace],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr

@mcp.tool
def describe_pod(pod_name: str, namespace: str = "default") -> str:
    """Get detailed info about a Kubernetes pod including events and conditions."""
    # ...

@mcp.tool
def get_events(namespace: str = "default") -> str:
    """Get recent Kubernetes events in a namespace."""
    # ...

if __name__ == "__main__":
    mcp.run()
```

Key differences from LangChain tools:

- `@mcp.tool` instead of `@tool` -- registered with the MCP server
- `FastMCP("Kubernetes Tools")` -- creates a named MCP server
- `mcp.run()` -- starts the server (stdio transport by default)
- Any MCP client can discover and call these tools

### `module-3/agent_with_mcp.py` (the MCP client)

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

async def main():
    client = MultiServerMCPClient({
        "docker-mcp": {
            "transport": "stdio",
            "command": "python",
            "args": ["mcp_server.py"]
        }
    })

    tools = await client.get_tools()    # Dynamically discovers tools from MCP
    llm = ChatOllama(model="gemma4", temperature=0.8)
    agent = create_agent(llm, tools)    # Same ReAct agent, but tools come from MCP
```

The agent does not define tools locally. It connects to the MCP server and discovers them at runtime.

### Run the MCP agent

```bash
cd module-3
python3 agent_with_mcp.py
```

```
> List the pods in my cluster
> Why is broken-pod crashing?
> What events happened recently?
```

### Configure Claude Desktop with the MCP server

Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS):

```json
{
  "mcpServers": {
    "kubernetes-tools": {
      "command": "python3",
      "args": ["/full/path/to/agentic-ai-for-devops/module-3/mcp_server.py"]
    }
  }
}
```

Restart Claude Desktop, then ask: "List the pods in my cluster". It will call the MCP server's `list_pods()` tool.

---

## Task 4: CI/CD Failure Analyzer (Module 6)

Same agent pattern, applied to CI/CD. It uses the `gh` CLI to diagnose GitHub Actions failures.

### Prerequisite

```bash
gh auth login
```

### The 3 tools in `module-6/ci_analyzer.py`

```python
@tool
def list_workflow_runs(status: str = "failure") -> str:
    """List recent GitHub Actions workflow runs. Use status='failure' for failed runs."""
    result = subprocess.run(
        ["gh", "run", "list", "--status", status, "--limit", "5"],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr

@tool
def get_failed_logs(run_id: str) -> str:
    """Get the failed step logs from a GitHub Actions run. Pass the run ID."""
    result = subprocess.run(
        ["gh", "run", "view", run_id, "--log-failed"],
        capture_output=True, text=True,
    )
    output = result.stdout + result.stderr
    if len(output) > 5000:
        output = output[:5000] + "\n\n[...truncated, showing first 5000 chars]"
    return output

@tool
def get_workflow_file(workflow_name: str) -> str:
    """Read a GitHub Actions workflow YAML file. Pass the filename like 'ci.yml'."""
    import pathlib
    path = pathlib.Path(f".github/workflows/{workflow_name}")
    if path.exists():
        return path.read_text()
    return f"File not found: {path}"
```

Note the log truncation in `get_failed_logs`: LLMs have token limits, so 100KB of CI logs cannot be sent. Truncating to 5000 characters keeps it within bounds while preserving the most important part (the failed step output).

### Run it inside the AI-BankApp repo

```bash
cd AI-BankApp-DevOps
python3 ../agentic-ai-for-devops/module-6/ci_analyzer.py
```

```
> What failed in my last CI run?
> Show me the recent workflow runs
> Read the gitops-ci.yml workflow file and explain what it does
```

### Test with a deliberately broken workflow

Create `.github/workflows/broken-ci.yml` in a test repo:

```yaml
name: Broken CI
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test    # Will fail -- no package.json!
```

Push it, let it fail, then ask the agent: "Why did broken-ci fail?"

---

## Task 5: Build Your Own Tool

Any CLI command can be a tool. Options offered: Terraform Plan Analyzer, AWS Resource Checker, Log Searcher.

### Tool built: Log Searcher (Option C)

```python
@tool
def search_logs(keyword: str, namespace: str = "default") -> str:
    """Search for a keyword in the logs of all pods in a namespace."""
    pods = subprocess.run(
        ["kubectl", "get", "pods", "-n", namespace, "-o", "name"],
        capture_output=True, text=True,
    )
    results = []
    for pod in pods.stdout.strip().split("\n"):
        if not pod:
            continue
        logs = subprocess.run(
            ["kubectl", "logs", pod, "-n", namespace, "--tail=100"],
            capture_output=True, text=True,
        )
        if keyword.lower() in logs.stdout.lower():
            results.append(f"{pod}: found '{keyword}'")
    return "\n".join(results) if results else f"No pods contain '{keyword}' in their logs"
```

Add it to the tools list of `module-3/agent.py`, run the agent, and ask a question that triggers it, for example: "Which pods have 'error' in their logs?"

### How the agent decides when to use it

- The LLM is given each tool's name, argument schema, and docstring.
- The docstring ("Search for a keyword in the logs of all pods in a namespace") is the only thing that tells the model what the tool is for, so a clear docstring is what makes the tool get picked.
- When the question is about finding a word or message across many pods, the model matches it to `search_logs` and extracts `keyword` (and `namespace`) from the question as arguments.
- For a question like "why is broken-pod crashing?", `describe_pod` or `get_events` match better, so the agent uses those instead.
- After the tool returns, the agent reads the output and either answers or calls another tool.

---

## Task 6: Clean Up

```bash
# Delete Kind cluster
kind delete cluster --name devops-demo

# Remove broken container
docker rm -f broken-container 2>/dev/null

# Deactivate Python venv (if needed later)
deactivate
```

### What was built today

| Module                | What             | Tools                  | Pattern                 |
| --------------------- | ---------------- | ---------------------- | ----------------------- |
| 3 (agent.py)          | Multi-tool agent | 3 Docker + 3 K8s       | LangChain ReAct         |
| 3 (mcp_server.py)     | MCP server       | 3 K8s tools via MCP    | FastMCP                 |
| 3 (agent_with_mcp.py) | MCP client agent | Tools from MCP server  | LangChain + MCP adapter |
| 6                     | CI/CD analyzer   | 3 GitHub Actions tools | LangChain ReAct         |

---

# Summary Sections

## 1. The Multi-Tool Agent Architecture (6 tools across 2 domains)

```
                        User question
                              |
                              v
                  +-----------------------+
                  |  LLM (ReAct agent)    |
                  |  Reason -> Act ->     |
                  |  Observe -> Answer    |
                  +-----------+-----------+
                              |
          +-------------------+-------------------+
          |                                       |
   Docker domain                          Kubernetes domain
   - list_containers()                    - list_pods()
   - get_logs(container_name)             - describe_pod(pod_name, namespace)
   - inspect_container(container_name)    - get_events(namespace)
          |                                       |
     docker CLI                              kubectl CLI
```

- One agent, one LLM, one tool list. The tools are the only thing that differs per domain.
- Each tool wraps a CLI command with `subprocess.run` and returns stdout (or stderr on failure).
- The ReAct loop: the model reasons about the question, picks a tool, reads the result, and repeats until it can answer.
- Adding a new domain means adding more tools, not changing the agent logic.

## 2. MCP Explained: Server, Client, Protocol, Why It Matters

- **Protocol:** MCP (Model Context Protocol) is an open standard from Anthropic that defines how AI applications discover and call external tools and data. Transport is stdio (local process) or HTTP.
- **Server:** a program that exposes tools. Here, `mcp_server.py` built with FastMCP exposes `list_pods`, `describe_pod`, and `get_events`.
- **Client:** an AI application that connects to servers, asks what tools exist, and calls them. Examples: Claude Desktop, VS Code Copilot, Cursor, Claude Code, or a LangChain agent using `langchain-mcp-adapters`.
- **Discovery:** the client calls the server at runtime to get the tool list (`client.get_tools()`), so tools are not hardcoded in the client.
- **Why it matters for DevOps:**
  - Write the Docker/K8s tools once, use them from any MCP client.
  - Tools are not tied to one framework like LangChain.
  - Tools become a discoverable service that teams can share.

## 3. How the MCP Server Differs from Hardcoded LangChain Tools

| Aspect          | Hardcoded LangChain tools (`agent.py`)  | MCP server (`mcp_server.py`)                  |
| --------------- | --------------------------------------- | --------------------------------------------- |
| Decorator       | `@tool`                                 | `@mcp.tool`                                   |
| Where tools live | Inside the agent code                  | In a separate server process                  |
| Registration    | Passed in the agent's tool list         | Registered with `FastMCP("Kubernetes Tools")` |
| Startup         | Runs with the agent                     | `mcp.run()` starts a server (stdio default)   |
| Discovery       | Static, known at write time             | Dynamic, `await client.get_tools()`           |
| Reuse           | Only that framework/agent               | Any MCP client                                |
| Agent code      | Defines tools and the agent             | Only connects and creates the agent           |

The tool function bodies are nearly identical. What changes is where they live and how they are exposed.

## 4. The Custom Tool Built and How the Agent Used It

- **Tool:** `search_logs(keyword, namespace)` (Log Searcher). It lists pods in a namespace, pulls the last 100 log lines of each, and reports which pods contain the keyword.
- **Added to:** the tools list of the multi-tool agent.
- **How the agent decides to use it:** the model reads the tool's docstring and parameters. A question like "which pods mention 'error' in their logs?" matches the docstring, so the model calls `search_logs` with `keyword="error"`. Questions about a single pod's crash match `describe_pod` / `get_events` better.
- **Design notes:** `--tail=100` and the keyword filter keep output small for the model's token limit; the empty-result message gives the model a clear answer when nothing matches.

## 5. Tool Pattern Template (works for any CLI)

```python
import subprocess
from langchain_core.tools import tool

@tool
def my_tool(arg1: str, arg2: str = "default") -> str:
    """One clear sentence on what this does and when to use it.
    Mention what each argument means."""
    result = subprocess.run(
        ["cli-name", "subcommand", arg1, "--flag", arg2],
        capture_output=True, text=True,
    )
    output = result.stdout + result.stderr
    if len(output) > 5000:                      # respect LLM token limits
        output = output[:5000] + "\n[...truncated]"
    return output or "No output"
```

The same function as an MCP tool: swap `@tool` for `@mcp.tool` and register it on a `FastMCP(...)` server.

The pattern is always the same:

1. Define tools that wrap CLI commands
2. Create an LLM instance
3. Create a ReAct agent
4. The agent reasons about the question, calls tools, reads output, answers

Tips:

- The docstring is the tool's instruction manual for the LLM, so make it specific.
- Pass arguments as a list to `subprocess.run` (never build a shell string from model-provided input).
- Always return stderr when stdout is empty so the agent can see errors.
- Truncate long output.
- Prefer read-only commands (`get`, `describe`, `logs`, `plan`) for diagnostic agents.
