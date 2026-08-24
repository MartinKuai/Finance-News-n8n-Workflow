# finance-news-n8n-workflow V1

## 当前状态

- n8n 工作流：`finance-news-n8n-workflow`
- 工作流 ID：`Kmw1fuXCXnS3xtEK`
- 已保存节点数：25
- 当前保持未激活，适合先手动执行验证，避免第三方接口不稳定时被定时任务反复触发。
- 已导出文件：`finance-news-n8n-workflow.json`

## 已落地的数据流

`Schedule Trigger → RSS Read → Limit(1) → Normalize → Deduplicate → Fetch Article(Jina Reader) → Article Payload → Researcher HTTP → Researcher Parse/Validate → KEEP? → Writer HTTP → Writer Parse/Validate → Reviewer HTTP → Reviewer Parse/Validate → PASS? → Publisher Boundary`

第一轮拒绝分支已接通：

`Reviewer REJECT → Revision Writer → Final Reviewer → Final PASS? → Final Publisher Boundary / HOLD`

另外保留了 `SKIP` 分支，SKIP 不进入 Writer、Reviewer 或发布。

## 模型与凭据依赖

- 第三方 OpenAI-compatible 接口：`https://chatapi.weixin.qq.com/openai/v1/chat/completions`
- 模型：`GLM-5.2`
- HTTP 节点使用 n8n 中已存在的 Generic Bearer Auth 凭据条目。
- 没有读取、复制或记录任何 API Key、Token 或 Authorization 值。
- 原先的官方 OpenAI `Message a model` 节点已停用并保留为未使用节点；当前路径不再把第三方服务当作官方 OpenAI 服务。
- Telegram 凭据尚未配置，因此 Publisher 只保留 `telegram_copy` 作为可见的 Publisher boundary，不会实际发送消息。

## 已验证路径

1. 从 BBC Business RSS 拉取真实财经文章：`https://feeds.bbci.co.uk/news/business/rss.xml`。
2. Limit 节点限制为单篇文章，完成 Normalize、URL 去重和 Jina Reader 正文获取。
3. Researcher 真实调用成功，并解析为机器可读 JSON，包含 `KEEP/SKIP`、事件类型、实体、市场、事实证据和不确定性等字段。
4. `KEEP` 真分支已执行。
5. Writer 真实调用成功，只接收结构化 `research_notes`，没有接收原始正文。
6. Reviewer 真实调用成功并返回机器可读 `PASS`、scores、issues、revision_brief。
7. `PASS` 真分支已执行，Publisher boundary 输出了最终 `telegram_copy`，但 `published=false`。
8. 一次完整重放曾进入首轮拒绝后的 Revision Writer 分支，但第三方接口连接在该请求处被远端关闭；另一次重放在 Researcher 请求处遇到同类连接重置。

## 结构约束落实情况

- 原始正文只进入 Researcher HTTP 节点。
- Writer 只接收 Research Notes 和文章元数据。
- Reviewer 只接收 Research Notes + Draft。
- Revision Writer 只接收原始 Research Notes + 精简 `revision_brief`，不接收完整 Reviewer 输出。
- 只允许一次返工；最终 Reviewer 仍为 REJECT 时进入 HOLD，不发布。
- 未加入 RAG、向量库、数据库、行情 API、Agent、长期记忆或复杂多模型路由。

## V1 边界与后续项

- 当前是第一阶段单篇新闻 happy path，使用 Limit=1；多 RSS 源集中维护、Loop Over Items/Split in Batches、失败重试与完整 SKIP/HOLD 实测留到下一阶段。
- 正文优先使用 Jina Reader；本次 BBC 文章正文获取成功。若后续遇到访问不稳定，可暂时退回 RSS description，但不引入复杂抓虫。
- 由于第三方服务出现连接重置，完整自动重放尚未形成稳定的端到端 PASS 证据；当前可展示证据是逐节点真实执行到 Publisher boundary 的 PASS 路径。
