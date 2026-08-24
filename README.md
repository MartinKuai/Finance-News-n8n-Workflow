# finance-news-n8n-workflow

一个面向财经内容生产的 n8n AI 自动化工作流示例，展示如何把 RSS 新闻采集、正文获取、结构化研究、内容写作、质量审核和发布边界组织成一条可复现的 low-code workflow。

这个项目的重点不是堆叠复杂 Agent，而是把 AI 节点放在清晰、可观测、可控制的业务流程中，体现：

- Low-code Automation：使用 n8n 编排定时任务、RSS、HTTP、条件分支和可视化执行。
- API Integration：通过 HTTP Request 接入 OpenAI-compatible 第三方模型服务与 Jina Reader。
- AI Orchestration：以 Researcher、Writer、Reviewer 三个职责明确的 AI 节点完成研究、写作和审核。
- Deterministic Control：用 JSON 解析、字段校验、KEEP/SKIP、PASS/REJECT 和一次性返工控制流程质量。

## 工作流架构

```text
Schedule Trigger
      ↓
RSS Source Collection
      ↓
Normalize + URL Deduplicate
      ↓
Single Article Input
      ↓
Jina Reader / Article Fetch
      ↓
Researcher → KEEP / SKIP
      ↓ KEEP
Writer → Reviewer → PASS / REJECT
                  ↓ PASS
             Publisher Boundary

REJECT → Revision Writer → Final Reviewer
                         → Final Publish / HOLD
```

V1 先聚焦单篇财经新闻 happy path，使用 `Limit=1` 控制输入规模，便于逐节点验证和展示完整数据流。多 RSS、循环处理和更丰富的发布渠道可以在此基础上继续扩展。

## 核心工程设计

### 1. AI 节点职责隔离

```text
Raw Article
    ↓
Researcher
    ↓
Structured Research Notes
    ↓
Writer
    ↓
Structured Draft
    ↓
Reviewer
```

- 原始正文只进入 Researcher。
- Writer 只读取结构化 Research Notes，不直接读取原始正文。
- Reviewer 读取 Research Notes + Draft，检查事实、数字、时间、来源、因果表达、市场影响和格式。
- 返工 Writer 只接收原始 Research Notes + 精简 `revision_brief`，不接收完整 Reviewer 输出。

### 2. 机器可读的中间结果

Researcher 输出包含：

`decision`、`reason`、`event_type`、`entities`、`market`、`event_time`、`source`、`source_tier`、`facts`、`market_relevance`、`uncertainties`

Writer 输出包含：

`headline`、`summary`、`key_facts`、`why_it_matters`、`possible_market_impact`、`watch_next`、`telegram_copy`

Reviewer 输出包含：

`status`、`scores`、`issues`、`revision_brief`

所有 LLM 返回结果都经过最小必要的 JSON 解析和字段校验，再进入下一节点。

### 3. 有界审核与发布

- Researcher 的 `SKIP` 结果直接结束，不进入写作和发布。
- Reviewer 首次 `REJECT` 只允许一次 Revision Writer。
- Final Reviewer 再次 `REJECT` 时进入 `HOLD`，不发布。
- Publisher 是确定性边界，只接受 Reviewer 的 `PASS` 结果。

## 模型与集成

- 模型：`GLM-5.2`
- 第三方 OpenAI-compatible Endpoint：`https://chatapi.weixin.qq.com/openai/v1/chat/completions`
- 正文获取：Jina Reader
- RSS 示例源：BBC Business RSS
- 凭据管理：使用 n8n Credentials，API Key 和 Token 不写入仓库
- 发布边界：保留 `telegram_copy`，可继续连接 Telegram 或其他消息渠道

这里的 OpenAI-compatible 仅表示请求格式兼容；模型服务来自第三方提供商，不依赖官方 OpenAI 节点。

## 导入与运行

1. 打开本地 n8n，选择导入工作流。
2. 导入 [`workflows/finance-news-n8n-workflow.json`](workflows/finance-news-n8n-workflow.json)。
3. 在 n8n Credentials 中选择已有的 Generic Bearer Auth 凭据。
4. 手动执行工作流，查看 Researcher、Writer、Reviewer 和 Publisher Boundary 的结构化输出。
5. 如需真实消息发布，再为 Publisher Boundary 配置目标渠道凭据。

仓库不会保存任何明文 API Key、Token 或其他敏感凭据。

## 真实验证路径

已使用真实 BBC Business RSS 新闻完成：

`RSS → Normalize → Deduplicate → Jina Reader → Researcher → KEEP → Writer → Reviewer PASS → Publisher Boundary`

Publisher Boundary 会保留标题、来源、文章链接、审核状态和最终 `telegram_copy`，方便后续接入 Telegram、Slack 或其他发布节点。

## 仓库结构

```text
.
├── README.md
├── docs/
│   └── verification.md
├── workflows/
│   └── finance-news-n8n-workflow.json
└── .gitignore
```

更多实现说明见 [`docs/verification.md`](docs/verification.md)。
