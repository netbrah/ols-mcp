# Current State Analysis — OLS MCP Ecosystem

## ols-mcp PoC Status

<a href="https://github.com/thoraxe/ols-mcp">thoraxe/ols-mcp</a> has been dormant since **November 7, 2025** (only 2 commits total, both on that day). It was a PoC by <a href="https://github.com/thoraxe">thoraxe</a> (Erik Jacobs), who is actually an active contributor to <a href="https://github.com/openshift/lightspeed-service">openshift/lightspeed-service</a> with 94 commits there.

## Important Discovery: MCP Is Already Built Into `openshift/lightspeed-service`

The main repo already has **extensive MCP support built directly into it** — not as a separate standalone server, but as a native feature:

- `ols/utils/mcp_utils.py` — MCP client utilities
- `ols/app/endpoints/mcp_apps.py` — MCP proxy endpoints
- `ols/app/endpoints/mcp_client_headers.py` — MCP header management
- `ols/app/models/config.py` — `MCPServerConfig` / `MCPServers` models
- MCP constants, validation, tool filtering, and approval flows

This means the standalone `thoraxe/ols-mcp` PoC was likely **superseded** by the integrated MCP support in the main service.

## OpenAPI Spec Gaps

The current ols-mcp PoC (`thoraxe/ols-mcp`) only sends `query` and `conversation_id` to the OLS API, but the actual `openshift/lightspeed-service` OpenAPI spec shows `LLMRequest` supports significantly more fields:

| Field | Type | Description |
|-------|------|-------------|
| `query` | string | The query string (required) |
| `conversation_id` | string \| null | Optional conversation ID (UUID) |
| `provider` | string \| null | Optional LLM provider override |
| `model` | string \| null | Optional model override |
| `system_prompt` | string \| null | Optional system prompt |
| `attachments` | array \| null | Optional attachments (logs, configs, YAML) |
| `media_type` | string \| null | Optional media type (default: `text/plain`) |

Similarly, the `LLMResponse` is richer than what the PoC handles:

| Field | Type | Description |
|-------|------|-------------|
| `conversation_id` | string | Conversation ID |
| `response` | string | The LLM response text |
| `referenced_documents` | array | URLs/titles of docs used to generate response |
| `truncated` | boolean | Whether conversation history was truncated |
| `input_tokens` | integer | Tokens sent to LLM |
| `output_tokens` | integer | Tokens received from LLM |
| `available_quotas` | object | Remaining quotas |
| `tool_calls` | array | Tool call requests |
| `tool_results` | array | Tool call results |

## OLS API Endpoints (from OpenAPI spec)

The full OLS service exposes these endpoints:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/query` | POST | Main query endpoint |
| `/v1/streaming_query` | POST | Streaming query endpoint |
| `/v1/feedback/status` | GET | Feedback status |
| `/v1/feedback` | POST | Store user feedback |
| `/readiness` | GET | Readiness probe |
| `/liveness` | GET | Liveness probe |
| `/metrics` | GET | Prometheus metrics |
| `/authorized` | POST | Authorization check |
