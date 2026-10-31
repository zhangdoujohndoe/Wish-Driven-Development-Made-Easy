# 许愿式编程（Wish-Driven Development）指南

一个开源、模型无关、厂商中立（vendor-neutral）的自动化软件工程流程，旨在让 LLM 能够管理、编程和测试那些总代码量远超模型最大上下文窗口的软件项目。

该架构通过自主沙箱循环、轻量级代理网关以及原生 Git 生命周期命令实现容错式的错误恢复，从而践行 **"人类掌舵，智能体执行"（Human Steer, Agent Execute）** 的理念。

---

## 🏗️ 架构与组件矩阵

| 层级 | 组件 | 用途 | 大型代码库的关键策略 |
| :--- | :--- | :--- | :--- |
| **1. 智能体执行** | **OpenHands** | 自主编排器，负责规划任务、编辑文件并运行本地测试。 | **工具式探索（Tool-Use Exploration）：** 不读取整个代码库，而是在 Docker 沙箱中使用 `ripgrep` 和 `find` 动态定位并修改文件。 |
| **2. 代理网关** | **LiteLLM** | 抽象层，将各类后端 API 转换为标准化的 OpenAI 格式，并内置消费追踪与按模型的预算控制。 | **双脑路由（Split-Brain Routing）：** 将重度推理路由到云端"思考者"（Thinker）模型，将批量代码生成路由到高速本地"工作者"（Worker）模型，以优化 token 成本与准确率。 |
| **3. 安全与版本控制**| **原生 Git** | 完全由 LLM 通过 bash 工具执行的稳健版本管理策略。 | **检查点-提交-回滚（Checkpoint-Commit-Rollback）：** LLM 在成功的步骤上提交代码，并在代码出错时运行 `git reset --hard` 进行自我修复。 |

---

## 🧠 双脑网关配置（Split-Brain Gateway）

为了在大型代码库上实现效率最大化，LiteLLM 将认知负载拆分为双模型拓扑：

1. **思考者（编排者）：** 重度推理模型（如 DeepSeek-R1），负责导航代码库架构、调试错误堆栈以及规划逻辑。这也是 OpenHands 作为其主 LLM 使用的模型。
2. **工作者（编码者）：** 专用的高吞吐代码模型（如 Qwen-2.5-Coder:72B），负责生成原始文件更新与样板代码。

> **⚠️ 路由的实际工作原理：** OpenHands 运行的是*单一*智能体循环，无法仅通过提示词在对话中途原生切换模型。因此，工作者模型是通过**在沙箱内部显式调用网关 API** 来触发的（例如通过 bash 工具发起 `curl` 请求，或使用小型辅助脚本）。下方的提示词模板中内置了这一约定。这使得路由真实且可观测——每一次工作者调用都是一条可在 LiteLLM 日志中审计的真实 HTTP 请求。

### LiteLLM `config.yaml`

将以下配置保存到本地主机上的 `litellm_config.yaml` 文件中：

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

> **前置条件：** 本地的 Ollama 守护进程必须能被 Docker 网络访问到。使用 `OLLAMA_HOST=0.0.0.0 ollama serve` 启动（在 Linux 上，关于 `host.docker.internal` 的注意事项请参见故障排除章节）。

---

## 🛠️ 本地快速上手指南

在本地机器的终端（macOS/Linux）中运行以下命令，即可启动完整的本地流水线。

### 步骤 0：创建共享 Docker 网络

两个容器必须位于同一网络中才能互相通信：

```bash
docker network create wdd-net
```

### 步骤 1：启动 LiteLLM 网关

挂载配置文件，将双模型网关暴露在 `4000` 端口上。

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

### 步骤 2：启动 OpenHands 并连接到你的项目

将本地大型软件仓库挂载到 OpenHands 的工作区中，并将其智能体循环默认指向你的"思考者"模型。

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

*注意：打开 Web UI 仪表盘 `http://localhost:3000` 与你的工作区进行交互。*

> **🔒 安全提示：** 该配置会赋予 LLM 对你仓库的写权限、任意 shell 执行权限以及对你 Docker socket 的访问权限。请针对项目的**副本**（或专用的 `git worktree`）运行——切勿针对你唯一的代码副本。宿主机对这台容器的信任程度，取决于你对模型的信任程度。

---

## 📜 "许愿"执行协议

为了获得可预测的结果，请向 OpenHands 提供高度结构化的提示词，明确指定需求、约束、测试参数以及版本管理规则。

### 提示词模板

````markdown
### 任务：实现 [功能名称 / 问题描述]

### 1. 多 LLM 协作路由
- 使用你的主模型配置（`wdd-thinker`）来梳理文件结构、评估堆栈追踪并制定执行步骤。
- 在生成大段文件语法或执行重复性代码插入时，改为通过网关调用工作者模型，而不是自己编写，例如：

  ```bash
  curl -s http://litellm-gateway:4000/v1/chat/completions \
    -H "Authorization: Bearer $LLM_API_KEY" \
    -d '{"model": "wdd-worker", "messages": [{"role": "user", "content": "..."}]}'
  ```
- 在将工作者的输出写入磁盘之前，先进行审查。思考者负责正确性；工作者负责吞吐量。

### 2. 超大仓库的上下文卫生规范
- 切勿尝试读取整个仓库。使用 `ripgrep` 和 `find` 进行导航。
- 在仓库根目录维护一个持续更新的 `AGENT_STATE.md` 文件：在其中记录已发现的架构要点、关键文件路径和待解决问题，使知识在上下文压缩和新会话中得以延续。

### 3. 强制性的 Git 版本管理与安全网协议
在分析或编写代码之前，必须执行以下 Git 生命周期：
1. 确保当前工作目录是干净的（`git status`）。
2. 创建并切换到新的隔离功能分支：`git checkout -b ai/wish-[功能名称]`。
3. 每当完成一个子任务，或让某个特定文件通过语法检查时，就进行一次微型提交（`git commit -am "checkpoint: [简要说明]"`）。

### 4. 容错与搞砸后的恢复策略
- 如果你最近一次的代码修改导致了语法错误、无法解决的循环依赖，或你在 2 次尝试内无法修复的破坏性回归，你必须使用以下命令自主回滚到上一个已知的良好检查点：`git reset --hard HEAD~1`。
- 不要尝试在破损代码之上无限叠加补丁。回滚、重新阅读，然后换一种方法再试。

### 5. 验收标准与验证步骤
要判定此任务成功完成，你必须：
1. 通过代码搜索工具定位正确的模块。
2. [在此插入明确的功能需求细节]。
3. 通过终端命令运行项目的本地测试执行器（例如 `pytest tests/` 或 `npm test`）。
4. 确保测试执行器返回退出码 0。

### 6. 测试完整性（测试之测试规则）
- 除非任务描述明确要求，否则不要为了让测试通过而修改现有测试。
- 如果你认为某个现有测试本身有误，请保持其原样不动，并在最终总结中报告该问题，而不是默默地"修复"它。

### 7. 最终交付
- 所有测试通过后，不要合并到 main 分支。
- 将仓库留在干净的 `ai/wish-[功能名称]` 分支上，并更新 `AGENT_STATE.md`，总结修改内容与原因。
````

---

## ✅ 人类审批清单

流水线的终点是**你**。在合并智能体的分支之前：

1. `git diff main...ai/wish-[功能名称]` —— 阅读实际的代码差异；确认没有触碰任何不相关的文件。
2. 在**宿主机上**运行测试套件，而不是轻信沙箱的报告。
3. 对照代码差异检查 `AGENT_STATE.md`，排查无法解释的行为。
4. 使用 `git merge` 接受，或使用 `git checkout main && git branch -D ai/wish-[功能名称]` 丢弃整轮"愿望实现"。

---

## 💡 核心最佳实践

* **状态与运行时持久化：** OpenHands 直接写入你映射的目录，因此代码修改能在容器重启后保留。但是，任何容器级依赖（`tzdata`、系统软件包）都不会持久化。如有必要，请在开场的提示词中包含项目环境的安装命令（如 `npm install` 或 `pip install`）。
* **成本可观测性：** 由于所有流量都经过 LiteLLM，你可以开箱即用地获得按请求的消费日志、token 计量以及按模型的速率限制。请为 `LITELLM_MASTER_KEY` 设置预算，以免智能体失控的循环在凌晨 3 点悄悄耗尽你的 API 额度。
* **模型选择：** 由于该框架严重依赖复杂推理、工具使用规划和代码地图导航，请优先为思考者选用**推理模型**（如 DeepSeek-R1），为工作者选用高性能的开源权重编码模型（如 **Qwen-2.5-Coder:72B**）。
* **固定版本：** `:main-latest` 标签很方便，直到上游某次破坏性变更悄悄摧毁你的流水线。一旦配置跑通，请固定到具体的发布版本标签（例如 `openhands:0.x.y`）。
* **工作区整洁性：** 如果 LLM 跑偏了，只需打开宿主机终端，运行 `git checkout main && git branch -D ai/wish-[功能名称]`，整个沙箱实验就会立即从你的活动本地工作区中消失。

---

## 🔧 故障排除

* **OpenHands 无法连接网关（`Connection refused`）：** 确认两个容器都位于 `wdd-net` 网络中（使用 `docker network inspect wdd-net`），并且 `LLM_BASE_URL` 使用的是容器名（`http://litellm-gateway:4000`），而不是 `localhost`——在容器内部，`localhost` 指的是它自己。
* **工作者模型不可达（macOS/Linux）：** 在 **macOS/Windows** 上，`host.docker.internal` 开箱即用。在 **Linux** 上，需要为容器添加 `--add-host=host.docker.internal:host-gateway` 参数，并确保 Ollama 监听的是 `0.0.0.0`，而不是 `127.0.0.1`。
* **Docker socket 与 bind-mount 的怪异行为：** 由于 OpenHands 同时挂载了宿主机的 Docker socket *和*宿主机的仓库路径，嵌套沙箱中 bind-mount `/workspace` 时会解析为宿主机路径。请确保容器内外的绝对路径完全一致，以避免隐蔽的路径错配。

---

## 📄 许可证

基于 [MIT 许可证](LICENSE) 分发。*（发布前请在此处填写你的实际许可证。）*
