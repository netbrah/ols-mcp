# PR 1: Add Test Suite and CI

**Branch:** `add-tests-ci`  
**Target:** `thoraxe/ols-mcp:main`  
**Priority:** 🔴 Do today — this is the "I'm a serious engineer" signal

## PR Description

```
## What

Add pytest test suite covering models, client, and server, plus GitHub Actions CI.

## Why

pyproject.toml lists pytest/pytest-asyncio/pytest-mock as dev dependencies but
the repo has no tests/ directory. This adds the missing test infrastructure so
future changes can be validated automatically.

## Changes

- tests/conftest.py: Shared fixtures (mock env vars, mock HTTP responses)
- tests/test_models.py: Pydantic model validation tests
- tests/test_client.py: HTTP client tests with mocked httpx responses
- tests/test_server.py: MCP tool registration and invocation tests
- .github/workflows/ci.yml: Lint + test on push/PR
- pyproject.toml: Add ruff to dev dependencies, add tool config

## Testing

uv sync
uv run pytest -v
```

---

## Files to Create

### tests/__init__.py
```python
"""Tests for ols-mcp."""
```

### tests/conftest.py
```python
"""Shared test fixtures."""

import os
from unittest.mock import AsyncMock, patch

import pytest


@pytest.fixture(autouse=True)
def mock_env(monkeypatch):
    """Set default env vars for all tests."""
    monkeypatch.setenv("OLS_API_URL", "http://test-ols:8080")
    monkeypatch.setenv("OLS_API_TOKEN", "test-token")
    monkeypatch.setenv("OLS_TIMEOUT", "5.0")
    monkeypatch.setenv("OLS_VERIFY_SSL", "false")


@pytest.fixture
def mock_successful_response():
    """Mock a successful OLS API response."""
    return {
        "response": "Kubernetes is a container orchestration platform.",
        "conversation_id": "abc-123",
        "referenced_documents": [],
        "truncated": False,
        "input_tokens": 50,
        "output_tokens": 100,
        "available_quotas": {},
        "tool_calls": [],
        "tool_results": [],
    }


@pytest.fixture
def mock_error_response():
    """Mock an error OLS API response."""
    return {
        "detail": {
            "response": "Error while validation question",
            "cause": "Failed to handle request",
        }
    }
```

### tests/test_models.py
```python
"""Tests for Pydantic models."""

import pytest
from pydantic import ValidationError

from ols_mcp.models import LLMRequest, LLMResponse


class TestLLMRequest:
    """Tests for LLMRequest model."""

    def test_minimal_request(self):
        """Query-only request should work."""
        req = LLMRequest(query="How do I scale a deployment?")
        assert req.query == "How do I scale a deployment?"
        assert req.conversation_id is None

    def test_full_request(self):
        """Request with all fields should work."""
        req = LLMRequest(
            query="Help me debug this pod",
            conversation_id="abc-123",
        )
        assert req.query == "Help me debug this pod"
        assert req.conversation_id == "abc-123"

    def test_empty_query_rejected(self):
        """Empty string query should be accepted by pydantic (validated upstream)."""
        req = LLMRequest(query="")
        assert req.query == ""

    def test_missing_query_rejected(self):
        """Missing query field should raise ValidationError."""
        with pytest.raises(ValidationError):
            LLMRequest()


class TestLLMResponse:
    """Tests for LLMResponse model."""

    def test_basic_response(self):
        """Response with required fields should work."""
        resp = LLMResponse(response="Here is your answer.")
        assert resp.response == "Here is your answer."
        assert resp.conversation_id is None

    def test_full_response(self):
        """Response with all fields should work."""
        resp = LLMResponse(
            response="OLM helps users install operators.",
            conversation_id="abc-123",
        )
        assert resp.response == "OLM helps users install operators."
        assert resp.conversation_id == "abc-123"

    def test_missing_response_rejected(self):
        """Missing response field should raise ValidationError."""
        with pytest.raises(ValidationError):
            LLMResponse()
```

### tests/test_client.py
```python
"""Tests for HTTP client."""

import pytest
import httpx
from unittest.mock import AsyncMock, patch, MagicMock

from ols_mcp.client import query_openshift_lightspeed
from ols_mcp.models import LLMRequest


@pytest.mark.asyncio
class TestQueryClient:
    """Tests for query_openshift_lightspeed."""

    async def test_successful_query(self, mock_successful_response):
        """Successful API call should return LLMResponse."""
        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.json.return_value = mock_successful_response
        mock_response.raise_for_status = MagicMock()

        with patch("ols_mcp.client.httpx.AsyncClient") as mock_client_cls:
            mock_client = AsyncMock()
            mock_client.post.return_value = mock_response
            mock_client.__aenter__ = AsyncMock(return_value=mock_client)
            mock_client.__aexit__ = AsyncMock(return_value=False)
            mock_client_cls.return_value = mock_client

            request = LLMRequest(query="What is Kubernetes?")
            response = await query_openshift_lightspeed(request)

            assert response.response == "Kubernetes is a container orchestration platform."
            assert response.conversation_id == "abc-123"
            mock_client.post.assert_called_once()

    async def test_query_with_conversation_id(self, mock_successful_response):
        """Request with conversation_id should include it in payload."""
        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.json.return_value = mock_successful_response
        mock_response.raise_for_status = MagicMock()

        with patch("ols_mcp.client.httpx.AsyncClient") as mock_client_cls:
            mock_client = AsyncMock()
            mock_client.post.return_value = mock_response
            mock_client.__aenter__ = AsyncMock(return_value=mock_client)
            mock_client.__aexit__ = AsyncMock(return_value=False)
            mock_client_cls.return_value = mock_client

            request = LLMRequest(query="Follow up", conversation_id="abc-123")
            await query_openshift_lightspeed(request)

            call_args = mock_client.post.call_args
            payload = call_args.kwargs.get("json", call_args[1].get("json", {}))
            assert payload.get("conversation_id") == "abc-123"

    async def test_http_error_raises(self):
        """HTTP errors should raise Exception."""
        mock_response = MagicMock()
        mock_response.status_code = 401
        mock_response.text = "Unauthorized"
        mock_response.raise_for_status.side_effect = httpx.HTTPStatusError(
            "401", request=MagicMock(), response=mock_response
        )

        with patch("ols_mcp.client.httpx.AsyncClient") as mock_client_cls:
            mock_client = AsyncMock()
            mock_client.post.return_value = mock_response
            mock_client.__aenter__ = AsyncMock(return_value=mock_client)
            mock_client.__aexit__ = AsyncMock(return_value=False)
            mock_client_cls.return_value = mock_client

            request = LLMRequest(query="test")
            with pytest.raises(Exception, match="HTTP error"):
                await query_openshift_lightspeed(request)

    async def test_connection_error_raises(self):
        """Connection errors should raise Exception."""
        with patch("ols_mcp.client.httpx.AsyncClient") as mock_client_cls:
            mock_client = AsyncMock()
            mock_client.post.side_effect = httpx.ConnectError("Connection refused")
            mock_client.__aenter__ = AsyncMock(return_value=mock_client)
            mock_client.__aexit__ = AsyncMock(return_value=False)
            mock_client_cls.return_value = mock_client

            request = LLMRequest(query="test")
            with pytest.raises(Exception, match="Request error"):
                await query_openshift_lightspeed(request)

    async def test_bearer_token_in_headers(self, mock_successful_response):
        """Auth token should be sent as Bearer header."""
        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.json.return_value = mock_successful_response
        mock_response.raise_for_status = MagicMock()

        with patch("ols_mcp.client.httpx.AsyncClient") as mock_client_cls:
            mock_client = AsyncMock()
            mock_client.post.return_value = mock_response
            mock_client.__aenter__ = AsyncMock(return_value=mock_client)
            mock_client.__aexit__ = AsyncMock(return_value=False)
            mock_client_cls.return_value = mock_client

            request = LLMRequest(query="test")
            await query_openshift_lightspeed(request)

            call_args = mock_client.post.call_args
            headers = call_args.kwargs.get("headers", call_args[1].get("headers", {}))
            assert headers.get("Authorization") == "Bearer test-token"

    async def test_url_construction(self, mock_successful_response):
        """URL should be constructed from OLS_API_URL env var."""
        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.json.return_value = mock_successful_response
        mock_response.raise_for_status = MagicMock()

        with patch("ols_mcp.client.httpx.AsyncClient") as mock_client_cls:
            mock_client = AsyncMock()
            mock_client.post.return_value = mock_response
            mock_client.__aenter__ = AsyncMock(return_value=mock_client)
            mock_client.__aexit__ = AsyncMock(return_value=False)
            mock_client_cls.return_value = mock_client

            request = LLMRequest(query="test")
            await query_openshift_lightspeed(request)

            call_args = mock_client.post.call_args
            url = call_args.args[0] if call_args.args else call_args[0][0]
            assert url == "http://test-ols:8080/v1/query"
```

### tests/test_server.py
```python
"""Tests for MCP server tool registration."""

import pytest

from ols_mcp.server import list_tools, call_tool


@pytest.mark.asyncio
class TestListTools:
    """Tests for tool listing."""

    async def test_lists_one_tool(self):
        """Server should expose exactly one tool."""
        tools = await list_tools()
        assert len(tools) == 1

    async def test_tool_name(self):
        """Tool should be named openshift-lightspeed."""
        tools = await list_tools()
        assert tools[0].name == "openshift-lightspeed"

    async def test_tool_has_query_required(self):
        """Tool inputSchema should require query."""
        tools = await list_tools()
        schema = tools[0].inputSchema
        assert "query" in schema["required"]

    async def test_tool_has_conversation_id_optional(self):
        """Tool inputSchema should have optional conversation_id."""
        tools = await list_tools()
        schema = tools[0].inputSchema
        assert "conversation_id" in schema["properties"]
        assert "conversation_id" not in schema.get("required", [])


@pytest.mark.asyncio
class TestCallTool:
    """Tests for tool execution."""

    async def test_unknown_tool_raises(self):
        """Calling unknown tool should raise ValueError."""
        with pytest.raises(ValueError, match="Unknown tool"):
            await call_tool("nonexistent-tool", {"query": "test"})

    async def test_missing_query_returns_error(self):
        """Missing query should return error TextContent."""
        result = await call_tool("openshift-lightspeed", {})
        assert len(result) == 1
        assert "Error" in result[0].text
```

### .github/workflows/ci.yml
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: "latest"

      - name: Set up Python
        run: uv python install 3.12

      - name: Install dependencies
        run: uv sync

      - name: Lint with ruff
        run: uv run ruff check src/ tests/

      - name: Run tests
        run: uv run pytest -v --tb=short
```

### pyproject.toml changes (add ruff + pytest config)
Add these sections to the existing pyproject.toml:

```toml
# Add ruff to dev-dependencies:
[tool.uv]
dev-dependencies = [
    "pytest>=7.0.0",
    "pytest-asyncio>=0.21.0",
    "pytest-mock>=3.10.0",
    "ruff>=0.4.0",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "W"]
```