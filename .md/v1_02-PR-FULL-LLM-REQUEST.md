# PR 2: Expose Full LLMRequest Parameters

**Branch:** `full-llm-request`  
**Target:** `thoraxe/ols-mcp:main`  
**Priority:** 🔴 Do today — shows you read the openapi.json

## PR Description

```
## What

Expose all LLMRequest fields (provider, model, system_prompt, attachments,
media_type) and parse the full LLMResponse (referenced_documents,
token counts, truncation status).

## Why

The OLS API accepts 7 fields on /v1/query but the MCP tool only passes
query and conversation_id. This means MCP clients can't send pod logs
as attachments, choose LLM providers, or override system prompts. The
response also drops referenced_documents and token usage info.

Reference: openapi.json LLMRequest schema (lines 663-766)

## Changes

- src/ols_mcp/models.py: Add Attachment, ReferencedDocument models;
  expand LLMRequest/LLMResponse to match openapi.json
- src/ols_mcp/client.py: Pass all fields through to HTTP request,
  parse full response
- src/ols_mcp/server.py: Expand tool inputSchema with all properties,
  include referenced docs and token info in response text
- tests/test_models.py: Tests for new models
- tests/test_client.py: Tests for full request/response handling

## Testing

uv run pytest -v
```

---

## Files to Modify

### src/ols_mcp/models.py (replace entire file)
```python
"""Pydantic models for OpenShift LightSpeed MCP server."""

from typing import Optional
from pydantic import BaseModel, Field


class Attachment(BaseModel):
    """An attachment sent alongside a query (logs, YAML configs, etc.)."""

    attachment_type: str = Field(
        ..., description='Type of attachment: "log", "configuration", etc.'
    )
    content_type: str = Field(
        ..., description="MIME content type (text/plain, application/yaml, etc.)"
    )
    content: str = Field(..., description="The attachment content")


class ReferencedDocument(BaseModel):
    """A document referenced in the LLM response."""

    doc_url: str = Field(..., description="URL of the referenced document")
    doc_title: str = Field(..., description="Title of the referenced document")


class LLMRequest(BaseModel):
    """Request model for OpenShift LightSpeed API.

    Matches the full LLMRequest schema from openapi.json.
    """

    query: str = Field(..., description="The query to send to OpenShift LightSpeed")
    conversation_id: Optional[str] = Field(
        None, description="Optional conversation ID (UUID) for context"
    )
    provider: Optional[str] = Field(
        None, description="LLM provider to use (e.g. openai, azure_openai)"
    )
    model: Optional[str] = Field(None, description="Specific model to use")
    system_prompt: Optional[str] = Field(
        None, description="Custom system prompt override"
    )
    attachments: Optional[list[Attachment]] = Field(
        None, description="Attachments (logs, YAML configs) to include with query"
    )
    media_type: Optional[str] = Field(
        "text/plain", description="Response media type for streaming"
    )


class LLMResponse(BaseModel):
    """Response model from OpenShift LightSpeed API.

    Matches the full LLMResponse schema from openapi.json.
    """

    response: str = Field(..., description="The response from OpenShift LightSpeed")
    conversation_id: Optional[str] = Field(None, description="Conversation ID")
    referenced_documents: list[ReferencedDocument] = Field(
        default_factory=list, description="Documents referenced in the response"
    )
    truncated: bool = Field(
        False, description="Whether conversation history was truncated"
    )
    input_tokens: int = Field(0, description="Tokens sent to LLM")
    output_tokens: int = Field(0, description="Tokens received from LLM")
```

### src/ols_mcp/client.py (replace entire file)
```python
"""HTTP client for OLS API communication."""

import os
import httpx
from dotenv import load_dotenv

from .models import LLMRequest, LLMResponse, ReferencedDocument

load_dotenv()


def _get_config() -> dict:
    """Read OLS connection config from environment."""
    return {
        "api_url": os.getenv("OLS_API_URL", "http://localhost:8080"),
        "api_token": os.getenv("OLS_API_TOKEN"),
        "timeout": float(os.getenv("OLS_TIMEOUT", "30.0")),
        "verify_ssl": os.getenv("OLS_VERIFY_SSL", "true").lower() == "true",
    }


def _build_payload(request: LLMRequest) -> dict:
    """Build the JSON payload from an LLMRequest, omitting None fields."""
    data: dict = {"query": request.query}

    if request.conversation_id:
        data["conversation_id"] = request.conversation_id
    if request.provider:
        data["provider"] = request.provider
    if request.model:
        data["model"] = request.model
    if request.system_prompt:
        data["system_prompt"] = request.system_prompt
    if request.attachments:
        data["attachments"] = [a.model_dump() for a in request.attachments]
    if request.media_type and request.media_type != "text/plain":
        data["media_type"] = request.media_type

    return data


async def query_openshift_lightspeed(request: LLMRequest) -> LLMResponse:
    """Query the OpenShift LightSpeed API."""
    cfg = _get_config()
    url = f"{cfg['api_url'].rstrip('/')}/v1/query"

    headers = {"Content-Type": "application/json", "Accept": "application/json"}
    if cfg["api_token"]:
        headers["Authorization"] = f"Bearer {cfg['api_token']}"

    data = _build_payload(request)

    async with httpx.AsyncClient(
        timeout=cfg["timeout"], verify=cfg["verify_ssl"]
    ) as client:
        try:
            response = await client.post(url, json=data, headers=headers)
            response.raise_for_status()
            body = response.json()

            ref_docs = [
                ReferencedDocument(**doc)
                for doc in body.get("referenced_documents", [])
            ]

            return LLMResponse(
                response=body.get("response", "No response received"),
                conversation_id=body.get(
                    "conversation_id", request.conversation_id
                ),
                referenced_documents=ref_docs,
                truncated=body.get("truncated", False),
                input_tokens=body.get("input_tokens", 0),
                output_tokens=body.get("output_tokens", 0),
            )

        except httpx.HTTPStatusError as e:
            raise Exception(
                f"HTTP error {e.response.status_code}: {e.response.text}"
            )
        except httpx.RequestError as e:
            raise Exception(f"Request error: {str(e)}")
        except Exception as e:
            raise Exception(f"Unexpected error: {str(e)}")
```

### src/ols_mcp/server.py — Updated inputSchema and response formatting

The key change is expanding the `inputSchema` in `list_tools()` and the argument
handling in `call_tool()`. The tool response should now include referenced docs
and token info.

```python
"""Main MCP server implementation."""

import asyncio
import json
import logging
from typing import Any, Sequence

from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

from .models import LLMRequest, Attachment
from .client import query_openshift_lightspeed

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

server = Server("openshift-lightspeed-mcp")


@server.list_tools()
async def list_tools() -> list[Tool]:
    """List available tools."""
    return [
        Tool(
            name="openshift-lightspeed",
            description=(
                "Query OpenShift LightSpeed for assistance with OpenShift, "
                "Kubernetes, and related technologies. Supports attachments "
                "(pod logs, YAML configs) and provider/model selection."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "The question or query to send to OpenShift LightSpeed",
                    },
                    "conversation_id": {
                        "type": "string",
                        "description": "Optional conversation ID for maintaining context",
                    },
                    "provider": {
                        "type": "string",
                        "description": "LLM provider (e.g. openai, azure_openai)",
                    },
                    "model": {
                        "type": "string",
                        "description": "Specific model to use",
                    },
                    "system_prompt": {
                        "type": "string",
                        "description": "Custom system prompt override",
                    },
                    "attachments": {
                        "type": "array",
                        "description": "Attachments (logs, YAML) to send with the query",
                        "items": {
                            "type": "object",
                            "properties": {
                                "attachment_type": {
                                    "type": "string",
                                    "description": 'Type: "log", "configuration", etc.',
                                },
                                "content_type": {
                                    "type": "string",
                                    "description": "MIME type (text/plain, application/yaml)",
                                },
                                "content": {
                                    "type": "string",
                                    "description": "The attachment content",
                                },
                            },
                            "required": [
                                "attachment_type",
                                "content_type",
                                "content",
                            ],
                        },
                    },
                },
                "required": ["query"],
            },
        )
    ]


def _format_response(response) -> str:
    """Format LLMResponse into readable text with metadata."""
    parts = [response.response]

    if response.referenced_documents:
        parts.append("\n\n---\n**Referenced Documents:**")
        for doc in response.referenced_documents:
            parts.append(f"- [{doc.doc_title}]({doc.doc_url})")

    if response.truncated:
        parts.append(
            "\n⚠️ Conversation history was truncated to fit context window."
        )

    if response.input_tokens or response.output_tokens:
        parts.append(
            f"\n📊 Tokens: {response.input_tokens} in / {response.output_tokens} out"
        )

    return "\n".join(parts)


@server.call_tool()
async def call_tool(name: str, arguments: dict[str, Any]) -> Sequence[TextContent]:
    """Execute a tool call."""
    if name != "openshift-lightspeed":
        raise ValueError(f"Unknown tool: {name}")

    try:
        query = arguments.get("query")
        if not query:
            raise ValueError("Query is required")

        attachments = None
        raw_attachments = arguments.get("attachments")
        if raw_attachments:
            attachments = [Attachment(**a) for a in raw_attachments]

        request = LLMRequest(
            query=query,
            conversation_id=arguments.get("conversation_id"),
            provider=arguments.get("provider"),
            model=arguments.get("model"),
            system_prompt=arguments.get("system_prompt"),
            attachments=attachments,
        )

        response = await query_openshift_lightspeed(request)

        return [TextContent(type="text", text=_format_response(response))]

    except Exception as e:
        logger.error(f"Error calling OpenShift LightSpeed: {e}")
        return [TextContent(type="text", text=f"Error: {str(e)}")]


async def main():
    """Main entry point for the MCP server."""
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options(),
        )


if __name__ == "__main__":
    asyncio.run(main())
```