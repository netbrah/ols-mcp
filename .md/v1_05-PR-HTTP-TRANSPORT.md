# PR 5: Add HTTP Transport and Deployment Manifests

**Branch:** `http-transport`  
**Target:** `thoraxe/ols-mcp:main`  
**Priority:** 🟢 Wave 3 (week 3+)

## PR Description

```
## What

Add HTTP/SSE transport option alongside stdio, plus OpenShift
deployment manifests for running ols-mcp as a cluster service.

## Why

stdio transport only works for local Claude Code integration. To use
ols-mcp from any MCP client (web UIs, other agents, the OLS console
itself via mcp_apps.py proxy), it needs an HTTP transport. This also
enables deploying ols-mcp alongside OLS in the same namespace.

## Changes

- src/ols_mcp/server.py: Add TRANSPORT env var switching between
  stdio (default) and http mode using MCP SDK's streamablehttp server
- Dockerfile: Multi-stage build for container image
- deploy/: OpenShift Deployment, Service, Route, ConfigMap
- .env.example: Document TRANSPORT option

## Testing

# stdio (default, existing behavior)
uv run python -m ols_mcp.server

# http mode
TRANSPORT=http OLS_MCP_PORT=8000 uv run python -m ols_mcp.server
# Then: curl http://localhost:8000/mcp/v1 (MCP endpoint)
```

---

## Implementation Notes

### server.py changes

```python
import os

TRANSPORT = os.getenv("TRANSPORT", "stdio")

async def main():
    """Main entry point — supports stdio and http transports."""
    if TRANSPORT == "http":
        from mcp.server.streamable_http import StreamableHTTPServer
        port = int(os.getenv("OLS_MCP_PORT", "8000"))
        http_server = StreamableHTTPServer(server, port=port)
        await http_server.run()
    else:
        async with stdio_server() as (read_stream, write_stream):
            await server.run(
                read_stream,
                write_stream,
                server.create_initialization_options(),
            )
```

### Dockerfile

```dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

FROM base AS deps
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

FROM base AS runtime
COPY --from=deps /app/.venv /app/.venv
COPY src/ src/
ENV PATH="/app/.venv/bin:$PATH"
ENV TRANSPORT=http
ENV OLS_MCP_PORT=8000
EXPOSE 8000
CMD ["python", "-m", "ols_mcp.server"]
```

### deploy/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ols-mcp
  namespace: openshift-lightspeed
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ols-mcp
  template:
    metadata:
      labels:
        app: ols-mcp
    spec:
      containers:
        - name: ols-mcp
          image: quay.io/openshift-lightspeed/ols-mcp:latest
          ports:
            - containerPort: 8000
          env:
            - name: TRANSPORT
              value: "http"
            - name: OLS_API_URL
              value: "http://lightspeed-service:8080"
            - name: OLS_VERIFY_SSL
              value: "false"
          envFrom:
            - secretRef:
                name: ols-mcp-auth
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ols-mcp
  namespace: openshift-lightspeed
spec:
  selector:
    app: ols-mcp
  ports:
    - port: 8000
      targetPort: 8000
```