# thoraxe/ols-mcp — Contribution Roadmap

## Strategy: Slow Burn Fork Approach

Fork `thoraxe/ols-mcp`. Open PRs upstream to thoraxe's repo. Each PR is a
self-contained, mergeable unit. No mega-PRs. No "I built your whole project"
energy. Just steady, high-quality contributions that make his PoC production-real.

---

## Current State of thoraxe/ols-mcp (as of 2026-03-23)

| Metric | Value |
|--------|-------|
| Commits | 2 (both Nov 7 2025, Claude Code generated) |
| Source files | 5 Python files (~260 lines total) |
| Tests | **Zero** (pyproject.toml lists pytest as dev dep, no tests/ dir) |
| CI | None |
| MCP Tools | 1 (`openshift-lightspeed` — wraps `POST /v1/query`) |
| OLS API coverage | ~15% (only `query` + `conversation_id` of 7 fields) |
| Transport | stdio only |

## OLS API Surface (from included openapi.json)

| Endpoint | Exposed? | Priority |
|----------|----------|----------|
| `POST /v1/query` (full params) | Partial (2 of 7 fields) | **PR 2** |
| `POST /v1/streaming_query` | ❌ | **PR 3** |
| `POST /v1/feedback` | ❌ | **PR 4** |
| `GET /v1/feedback/status` | ❌ | **PR 4** |
| `GET /readiness` | ❌ | **PR 4** |
| `GET /liveness` | ❌ | **PR 4** |
| `POST /authorized` | ❌ | **PR 4** |
| `GET /metrics` | ❌ | Future |

---

## The 5 PRs

### Wave 1 — Drop Before Vacation (Today/Tomorrow)

| PR | Title | Branch | Details |
|----|-------|--------|---------|
| **PR 1** | Add test suite and CI | `add-tests-ci` | [01-PR-TESTS-CI.md](./01-PR-TESTS-CI.md) |
| **PR 2** | Expose full LLMRequest parameters | `full-llm-request` | [02-PR-FULL-LLM-REQUEST.md](./02-PR-FULL-LLM-REQUEST.md) |

### Wave 2 — Return From Vacation (Week 2)

| PR | Title | Branch | Details |
|----|-------|--------|---------|
| **PR 3** | Add streaming query support | `streaming-query` | [03-PR-STREAMING.md](./03-PR-STREAMING.md) |
| **PR 4** | Add feedback, health, and auth tools | `feedback-health-tools` | [04-PR-FEEDBACK-HEALTH.md](./04-PR-FEEDBACK-HEALTH.md) |

### Wave 3 — Keep the Pressure (Week 3+)

| PR | Title | Branch | Details |
|----|-------|--------|---------|
| **PR 5** | Add HTTP transport and deployment manifests | `http-transport` | [05-PR-HTTP-TRANSPORT.md](./05-PR-HTTP-TRANSPORT.md) |

---

## Execution Timeline

```
Week 0 (now):   Fork → PR 1 (tests) → PR 2 (full params) → Vacation
                 ↓
Week 1:          thoraxe reviews at his pace. You're on the beach.
                 ↓
Week 2 (back):   Respond to any comments → PR 3 (streaming) → PR 4 (feedback/health)
                 ↓
Week 3:          PR 5 (HTTP transport) → Continue lightspeed-service PRs
                 ↓
Week 4:          thoraxe has seen your name 5+ times → Submit resume
```

## PR Description Template

Keep them clean. No "Hey I'm looking for a job." Just engineering:

```
## What

[One sentence describing the change]

## Why

[One sentence on why this matters — reference the openapi.json or a gap]

## Changes

- file1.py: [what changed]
- file2.py: [what changed]

## Testing

[How to verify — commands to run]
```

## Key Technical Context

### Current Architecture
```
Claude Code ←stdio→ ols-mcp server ←HTTP→ OLS /v1/query
```

### Target Architecture (after all 5 PRs)
```
Claude Code ←stdio→ ols-mcp ←HTTP→ OLS /v1/query
                                   OLS /v1/streaming_query
                                   OLS /v1/feedback
                                   OLS /readiness
                                   OLS /liveness
                                   OLS /authorized

Any MCP client ←HTTP/SSE→ ols-mcp (running as service in cluster)
```

### The openapi.json Schemas You'll Use

**LLMRequest** (7 fields, only 2 currently used):
- `query` (string, required) ✅ used
- `conversation_id` (string|null) ✅ used
- `provider` (string|null) ❌ 
- `model` (string|null) ❌
- `system_prompt` (string|null) ❌
- `attachments` (Attachment[]|null) ❌
- `media_type` (string|null, default "text/plain") ❌

**Attachment** (3 required fields):
- `attachment_type` (string) — "log", "configuration", etc.
- `content_type` (string) — MIME type
- `content` (string) — the actual content

**LLMResponse** (9 fields, only 2 currently parsed):
- `conversation_id` (string) ✅ parsed
- `response` (string) ✅ parsed
- `referenced_documents` (ReferencedDocument[]) ❌
- `truncated` (boolean) ❌
- `input_tokens` (integer) ❌
- `output_tokens` (integer) ❌
- `available_quotas` (object) ❌
- `tool_calls` (array) ❌
- `tool_results` (array) ❌

**FeedbackRequest**:
- `conversation_id` (string, required)
- `user_question` (string, required)
- `llm_response` (string, required)
- `sentiment` (integer|null)
- `user_feedback` (string|null)