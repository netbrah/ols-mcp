# PR 3: Add Streaming Query Support

**Branch:** `streaming-query`  
**Target:** `thoraxe/ols-mcp:main`  
**Priority:** 🟡 Wave 2 (after vacation)

## PR Description

```
## What

Add a second MCP tool `openshift-lightspeed-stream` that calls
POST /v1/streaming_query and returns the full streamed response.

## Why

OLS has a dedicated streaming endpoint that returns SSE chunks including
tool calls and results from the agent loop. For long-running queries
(troubleshooting, log analysis), streaming provides faster time-to-first-token
and visibility into the agent's reasoning process.

Reference: openapi.json /v1/streaming_query endpoint

## Changes

- src/ols_mcp/client.py: Add stream_openshift_lightspeed() function
  using httpx SSE streaming
- src/ols_mcp/server.py: Register second tool, collect stream chunks
  and return full response
- tests/test_client.py: Streaming client tests with mocked SSE
- tests/test_server.py: Tool registration tests for new tool

## Testing

uv run pytest -v
```

---

## Implementation Notes

### client.py addition

```python
async def stream_openshift_lightspeed(request: LLMRequest) -> str:
    """Stream a query to the OLS streaming endpoint.

    Collects all SSE chunks and returns the complete response text.
    The /v1/streaming_query endpoint returns server-sent events where
    each data line contains a JSON fragment or plain text token.
    """
    cfg = _get_config()
    url = f"{cfg['api_url'].rstrip('/')}/v1/streaming_query"

    headers = {
        "Content-Type": "application/json",
        "Accept": "text/event-stream",
    }
    if cfg["api_token"]:
        headers["Authorization"] = f"Bearer {cfg['api_token']}"

    data = _build_payload(request)
    # Set media_type for streaming
    data["media_type"] = request.media_type or "text/plain"

    collected = []

    async with httpx.AsyncClient(
        timeout=cfg["timeout"], verify=cfg["verify_ssl"]
    ) as client:
        try:
            async with client.stream("POST", url, json=data, headers=headers) as resp:
                resp.raise_for_status()
                async for line in resp.aiter_lines():
                    if line.startswith("data: "):
                        chunk = line[6:]
                        if chunk.strip() == "[DONE]":
                            break
                        collected.append(chunk)

        except httpx.HTTPStatusError as e:
            raise Exception(f"HTTP error {e.response.status_code}")
        except httpx.RequestError as e:
            raise Exception(f"Request error: {str(e)}")

    return "".join(collected)
```

### server.py — Add second tool

Add to `list_tools()`:
```python
Tool(
    name="openshift-lightspeed-stream",
    description=(
        "Stream a query to OpenShift LightSpeed for real-time responses. "
        "Best for long-running queries like troubleshooting and log analysis."
    ),
    inputSchema={
        # Same schema as openshift-lightspeed
    },
)
```

Add to `call_tool()`:
```python
elif name == "openshift-lightspeed-stream":
    # Build request same as above
    response_text = await stream_openshift_lightspeed(request)
    return [TextContent(type="text", text=response_text)]
```