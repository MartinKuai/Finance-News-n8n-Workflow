# V1 实现与验证说明

## 项目概览

- 工作流：`finance-news-n8n-workflow`
- n8n 工作流 ID：`Kmw1fuXCXnS3xtEK`
- 当前导出节点数：25
- V1 输入策略：单次处理一篇新闻，使用 `Limit=1`
- 主要目标：展示 low-code automation、API integration 和 AI orchestration 的完整闭环

## 数据流

```text
Schedule Trigger
→ RSS Read
→ Limit(1)
→ Normalize
→ Deduplicate
→ Fetch Article / Jina Reader
→ Article Payload
→ Researcher HTTP
→ Researcher Parse / Validate
→ KEEP?
→ Writer HTTP
→ Writer Parse / Validate
→ Reviewer HTTP
→ Reviewer Parse / Validate
→ PASS?
→ Publisher Boundary
```

审核分支：

```text
Reviewer REJECT
→ Revision Writer
→ Final Reviewer
→ Final PASS?
→ Final Publish Boundary / HOLD
```

Researcher 的 `SKIP` 分支直接结束，不进入 Writer、Reviewer 或发布。

## 模型与外部服务

- 模型：`GLM-5.2`
- Endpoint：`https://chatapi.weixin.qq.com/openai/v1/chat/completions`
- 接入方式：n8n HTTP Request + Generic Bearer Auth Credential
- RSS：BBC Business RSS
- 正文：Jina Reader
- 发布：Publisher Boundary 保留结构化 `telegram_copy`

这里的 OpenAI-compatible 表示请求格式兼容，模型服务来自第三方提供商。凭据由 n8n Credentials 管理，仓库不保存任何 API Key、Token 或 Authorization 值。

## 结构化输出

### Researcher

```json
{
  "decision": "KEEP|SKIP",
  "reason": "...",
  "event_type": "MACRO|COMPANY|MARKET|POLICY|GEOPOLITICS|OTHER",
  "entities": [],
  "market": [],
  "event_time": null,
  "source": "...",
  "source_tier": "...",
  "facts": [{"claim": "...", "evidence": "...", "confidence": "HIGH|MEDIUM|LOW"}],
  "market_relevance": "...",
  "uncertainties": []
}
```

### Writer

```json
{
  "headline": "...",
  "summary": "...",
  "key_facts": [],
  "why_it_matters": "...",
  "possible_market_impact": "...",
  "watch_next": [],
  "telegram_copy": "..."
}
```

### Reviewer

```json
{
  "status": "PASS|REJECT",
  "scores": {},
  "issues": [],
  "revision_brief": []
}
```

## 已验证路径

使用真实 BBC Business 新闻完成了以下路径验证：

`RSS → Normalize → Deduplicate → Jina Reader → Researcher → KEEP → Writer → Reviewer PASS → Publisher Boundary`

验证重点包括：

- RSS 项目字段被统一为文章元数据和去重键。
- Jina Reader 获取正文后，原始正文只进入 Researcher。
- Researcher 生成结构化 Research Notes。
- Writer 只接收 Research Notes，不接收原始正文。
- Reviewer 接收 Research Notes + Draft，并返回机器可读 PASS 结果。
- Publisher Boundary 输出文章元数据、审核状态和最终 `telegram_copy`。

## 工程约束

- 只保留 Researcher、Writer、Reviewer 三个 AI 职责节点。
- Writer、Reviewer 和 Revision Writer 使用明确的 allow-listed 输入。
- Revision Writer 最多执行一次。
- Final Reviewer 仍为 REJECT 时进入 HOLD，不进入发布。
- 未加入 RAG、向量数据库、行情 API、Agent、长期记忆或复杂多模型路由。

## 下一阶段扩展

- 将单一 RSS 扩展为集中维护的 3–5 个稳定财经源。
- 接入 Loop Over Items / Split in Batches 进行多篇处理。
- 增加更完整的 SKIP、HOLD 和发布渠道演示。
- 在保持当前数据边界的基础上增加 Telegram、Slack 等 Publisher 适配器。
