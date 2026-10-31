# Wish-Driven Development (WDD) Pipeline

An open-source, model-agnostic, and vendor-neutral autonomous engineering pipeline designed to let LLMs manage, program, and test software projects whose total code volume far exceeds the model's maximum context window.

This setup enforces the philosophy of **"Human Steer, Agent Execute"** by leveraging an autonomous sandbox loop, a lightweight proxy gateway, and native Git lifecycle commands for fail-safe error recovery.

---

## 🏗️ Architecture & Component Matrix

| Layer | Component | Purpose | Key Strategy for Large Codebases |
| :--- | :--- | :--- | :--- |
| **1. Agent Execution** | **OpenHands** | The autonomous orchestrator that plans tasks, edits files, and runs local tests. | **Tool-Use Exploration:** Instead of reading the whole codebase, it uses `ripgrep` and `find` in a Docker sandbox to dynamically locate and patch files. |
| **2. Proxy Gateway** | **LiteLLM** | An abstraction layer converting diverse backend APIs into a standardized OpenAI format, with built-in spend tracking and per-model budgets. | **Split-Brain Routing:** Routes heavy reasoning to a cloud "Thinker" model and bulk code generation to a fast local "Worker" model to optimize token cost and accuracy. |
| **3. Safety & Versioning**| **Native Git** | A robust versioning strategy managed entirely by the LLM via bash tool execution. | **Checkpoint-Commit-Rollback:** The LLM commits on successful steps and runs `git reset --hard` to self-heal when code breaks. |

---

## 🧠 Split-Brain Gateway Configuration

To maximize efficiency on large codebases, LiteLLM separates the cognitive load into a dual-model topology:

1. **The Thinker (Orchestrator):** A heavy reasoning model (e.g., DeepSeek-R1) tasked with navigating repo architecture, debugging error traces, and planning logic. This is the model OpenHands uses as its primary LLM.
2. **The Worker (Coder):** A specialized, high-throughput code model (e.g., Qwen-2.5-Coder:72B) tasked with writing raw file updates and boilerplate.

> **⚠️ How routing actually works:** OpenHands runs a *single* agent loop and cannot natively switch models mid-conversation via prompt alone. The Worker is therefore invoked **explicitly over the gateway API from inside the sandbox** (e.g., a `curl` call via the bash tool, or a small helper script). The prompt template below encodes this convention. This keeps the routing honest and observable — every Worker call is a real HTTP request you can audit in LiteLLM's logs.

### LiteLLM `config.yaml`

Save the following configuration as `litellm_config.yaml` on your local host:

```yaml
model_list:
  - model_name: wdd-thinker
    litellm_params:
      model: deepseek/deepseek-reasoner
      api_base: https://api.deepseek.com/v1
      api_key: os.environ/DEEPSEEK_API_KEY
  - model_name: wdd-worker
    litellm_params:
      model: ollama/qwen2.5-coder:72b
      api_base: http://host.docker.internal:11434

litellm_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

> **Prerequisite:** Your local Ollama daemon must be reachable from inside the Docker network. Start it with `OLLAMA_HOST=0.0.0.0 ollama serve` (and on Linux, see the Troubleshooting section regarding `host.docker.internal`).

---

## 🛠️ Local Quickstart Guide

Run these commands in your local machine's terminal (macOS/Linux) to spin up your entire local pipeline.

### Step 0: Create a shared Docker network

Both containers must sit on the same network to communicate:

```bash
docker network create wdd-net
```

### Step 1: Start the LiteLLM Gateway

Mount the configuration file to expose your dual-model gateway on port `4000`.

```bash
docker run -d \
  --name litellm-gateway \
  --network wdd-net \
  -p 4000:4000 \
  -v $(pwd)/litellm_config.yaml:/app/config.yaml \
  -e DEEPSEEK_API_KEY="your_actual_api_key_here" \
  -e LITELLM_MASTER_KEY="sk-wdd-local-key" \
  ghcr.io/berriai/litellm:main-latest \
  --config /app/config.yaml
```

### Step 2: Launch OpenHands Connected to your Project

Mount your local massive software repository into the OpenHands workspace, defaulting its agent loop to your Thinker profile.

```bash
docker run -d \
  --name openhands-workspace \
  --network wdd-net \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /absolute/path/to/your/local/massive-project:/workspace \
  -e LLM_MODEL="openai/wdd-thinker" \
  -e LLM_BASE_URL="http://litellm-gateway:4000" \
  -e LLM_API_KEY="sk-wdd-local-key" \
  -p 3000:3000 \
  --add-host=host.docker.internal:host-gateway \
  ghcr.io/all-hands-ai/openhands:main
```

*Note: Open the Web UI dashboard at `http://localhost:3000` to interact with your workspace.*

> **🔒 Security Notice:** This setup gives an LLM write access to your repository, arbitrary shell execution, and access to your Docker socket. Run it against a **copy of the project** (or a dedicated `git worktree`) — never your only copy. Your host machine trusts this container as much as you trust the model.

---

## 📜 The "Wish" Execution Protocol

To achieve predictable results, feed OpenHands highly structured prompts specifying your requirements, constraints, testing parameters, and versioning rules.

### Prompt Template

````markdown
### TASK: Implement [Feature Name / Issue Description]

### 1. Multi-LLM Collaboration Routing
- Use your primary model profile (`wdd-thinker`) to map files, evaluate stack traces, and formulate execution steps.
- When generating large blocks of file syntax or executing repetitive code insertions, invoke the Worker model over the gateway instead of writing it yourself, e.g.:

  ```bash
  curl -s http://litellm-gateway:4000/v1/chat/completions \
    -H "Authorization: Bearer $LLM_API_KEY" \
    -d '{"model": "wdd-worker", "messages": [{"role": "user", "content": "..."}]}'
  ```
- Review the Worker's output before applying it to disk. The Thinker owns correctness; the Worker owns throughput.

### 2. Context Hygiene for Oversized Repositories
- Never attempt to read the entire repository. Use `ripgrep` and `find` to navigate.
- Maintain a running `AGENT_STATE.md` at the repo root: record discovered architecture notes, key file paths, and open questions there so knowledge survives context compaction and new sessions.

### 3. Mandatory Git Versioning & Safety Net Protocol
Before analyzing or writing code, you must execute the following Git lifecycle:
1. Ensure your current working directory is clean (`git status`).
2. Create and switch to a new isolated feature branch: `git checkout -b ai/wish-[feature-name]`.
3. Make micro-commits (`git commit -am "checkpoint: [brief summary]"`) every time you complete a sub-task or get a specific file to pass syntax checks.

### 4. Fail-Safe & Mess-Up Recovery Policy
- If your latest code modification causes syntax errors, unresolvable circular dependencies, or breaking regressions that you cannot fix within 2 attempts, you MUST autonomously roll back to your last known good checkpoint using: `git reset --hard HEAD~1`.
- Do not attempt to infinitely stack patches on top of broken code. Revert, re-read, and try a different approach.

### 5. Acceptance Criteria & Verification Steps
To consider this task successfully complete, you must:
1. Locate the correct modules via code search tools.
2. [Insert explicit functional requirement details here].
3. Run the project's local test runner via terminal command (e.g., `pytest tests/` or `npm test`).
4. Ensure the test runner returns exit code 0.

### 6. Test Integrity (Test-of-Tests Rule)
- Do NOT modify existing tests to make them pass unless the task description explicitly requires it.
- If you believe an existing test is incorrect, leave it untouched and report it in your final summary instead of "fixing" it silently.

### 7. Final Delivery
- Once all tests pass, do not merge into main.
- Leave the repository on your clean `ai/wish-[feature-name]` branch and update `AGENT_STATE.md` with a summary of what changed and why.
````

---

## ✅ Human Approval Checklist

The pipeline ends with **you**. Before merging an agent's branch:

1. `git diff main...ai/wish-[feature-name]` — read the actual diff; verify no unrelated files were touched.
2. Run the test suite **on your host**, not trusting the sandbox's report.
3. Check `AGENT_STATE.md` against the diff for unexplained behavior.
4. Accept with `git merge`, or discard the entire wish with `git checkout main && git branch -D ai/wish-[feature-name]`.

---

## 💡 Core Best Practices

* **State vs Runtime Persistence:** OpenHands writes directly to your mapped directory, meaning code changes survive container restarts. However, any container-level dependencies (`tzdata`, system packages) do not persist. Include project environment installation commands (like `npm install` or `pip install`) in your opening workspace prompt if necessary.
* **Cost Observability:** Because all traffic flows through LiteLLM, you get per-request spend logs, token accounting, and per-model rate limits out of the box. Set a `LITELLM_MASTER_KEY` budget so a runaway agent loop can't silently drain your API credits at 3 AM.
* **Model Selection:** Since the framework depends heavily on complex reasoning, tool usage planning, and code map navigation, prioritize using **Reasoning Models** for the Thinker (e.g., DeepSeek-R1) and highly capable open-weight coding models for the Worker (e.g., **Qwen-2.5-Coder:72B**).
* **Pin Your Versions:** `:main-latest` tags are convenient until a breaking upstream change silently destroys your pipeline. Pin to specific release tags (e.g., `openhands:0.x.y`) once your setup works.
* **Workspace Cleanliness:** If the LLM veers off track, simply open your host terminal, run `git checkout main && git branch -D ai/wish-[feature-name]`, and the entire sandbox experiment will instantly disappear from your active local workspace.

---

## 🔧 Troubleshooting

* **OpenHands can't reach the gateway (`Connection refused`):** Confirm both containers are on `wdd-net` (`docker network inspect wdd-net`) and that `LLM_BASE_URL` uses the container name (`http://litellm-gateway:4000`), not `localhost` — inside the container, `localhost` is itself.
* **Worker model unreachable (macOS/Linux):** On **macOS/Windows**, `host.docker.internal` works out of the box. On **Linux**, add `--add-host=host.docker.internal:host-gateway` to the LiteLLM container (done in Step 1's network setup) and ensure Ollama is listening on `0.0.0.0`, not `127.0.0.1`.
* **Docker socket + bind-mount weirdness:** Because OpenHands mounts the host Docker socket *and* the host workspace path, a nested sandbox that bind-mounts `/workspace` will resolve to the host path. Keep absolute paths identical inside and outside the container to avoid silent mismatches.

---

## 📄 License

Distributed under the [MIT License](LICENSE). *(Specify your actual license here before publishing.)*
