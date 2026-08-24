# finance-news-n8n-workflow

An n8n V1 workflow for turning a real finance-news RSS item into a structured, review-gated brief. The project demonstrates low-code automation, API integration, deterministic branching, and AI orchestration without adding RAG, a vector database, market-data APIs, agents, or long-term memory.

## What is included

- `workflows/finance-news-n8n-workflow.json` — n8n workflow export.
- `docs/verification.md` — sources, model, credential boundary, verified paths, and known limitations.

## Flow

```text
Schedule
  → RSS source
  → single-item limit
  → normalize + URL deduplicate
  → Jina Reader article fetch
  → Researcher JSON
  → KEEP/SKIP
  → Writer JSON
  → Reviewer JSON
  → PASS/REJECT
  → one bounded revision
  → Final Reviewer
  → Publisher boundary or HOLD
```

The raw article is provided only to Researcher. Writer receives structured Research Notes, Reviewer receives Research Notes plus Draft, and the revision writer receives Research Notes plus only the concise `revision_brief`.

## Import and credentials

1. In n8n, import `workflows/finance-news-n8n-workflow.json`.
2. Configure an existing Generic Bearer Auth credential in n8n Credentials for the HTTP nodes.
3. Do not put API keys or tokens in this repository. The provider endpoint is OpenAI-compatible, but it is a third-party service rather than the official OpenAI API.
4. Telegram is intentionally a publisher boundary in V1. Without a Telegram credential, the workflow retains `telegram_copy` in visible execution output and does not send it.

## Current V1 boundary

The first phase is intentionally limited to one article per run (`Limit=1`) and one stable BBC Business RSS source. A real article was fetched through Jina Reader and manually verified through Researcher → KEEP → Writer → Reviewer PASS → Publisher boundary. The bounded Revision/HOLD and SKIP branches are wired; full automatic replay still depends on the third-party provider remaining available.

See [`docs/verification.md`](docs/verification.md) for the detailed evidence and limitations.
