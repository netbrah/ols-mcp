# PR 4: Add Feedback, Health, and Authorization Tools

**Branch:** `feedback-health-tools`  
**Target:** `thoraxe/ols-mcp:main`  
**Priority:** 🟡 Wave 2 (after vacation)

## PR Description

```
## What

Add three new MCP tools: ols-feedback (submit user feedback),
ols-health (check service readiness/liveness), and ols-authorize
(verify user authorization).

## Why

The OLS API exposes feedback, health, and auth endpoints that are
useful for agentic workflows. An AI agent can rate its own interactions,
verify the service is up before expensive queries, and pre-check
user permissions in multi-user setups.

Reference: openapi.json /v1/feedback, /readiness, /liveness, /authorized

## Changes

- src/ols_mcp/models.py: Add FeedbackRequest, FeedbackResponse,
  HealthResponse models
- src/ols_mcp/client.py: Add submit_feedback(), check_health(),
  check_authorization() functions
- src/ols_mcp/server.py: Register 3 new tools with proper inputSchemas
- tests/: Full test coverage for new tools

## Testing

uv run pytest -v
```

---

## Implementation Notes

### models.py additions

```python
class FeedbackRequest(BaseModel):
    """Request model for submitting feedback on an OLS response."""

    conversation_id: str = Field(..., description="Conversation ID (UUID)")
    user_question: str = Field(..., description="The original user question")
    llm_response: str = Field(..., description="The LLM response to rate")
    sentiment: Optional[int] = Field(
        None, description="Sentiment score (1 = positive, -1 = negative)"
    )
    user_feedback: Optional[str] = Field(
        None, description="Free-text feedback"
    )


class FeedbackResponse(BaseModel):
    """Response from the feedback endpoint."""

    response: str = Field(..., description="Feedback storage confirmation")


class HealthResponse(BaseModel):
    """Combined health check response."""

    ready: bool = Field(..., description="Whether the service is ready")
    alive: bool = Field(..., description="Whether the service is alive")
```

### client.py additions

```python
async def submit_feedback(request: FeedbackRequest) -> FeedbackResponse:
    """Submit user feedback to OLS."""
    cfg = _get_config()
    url = f"{cfg['api_url'].rstrip('/')}/v1/feedback"
    # POST with FeedbackRequest payload, return FeedbackResponse

async def check_health() -> HealthResponse:
    """Check OLS readiness and liveness."""
    cfg = _get_config()
    ready_url = f"{cfg['api_url'].rstrip('/')}/readiness"
    alive_url = f"{cfg['api_url'].rstrip('/')}/liveness"
    # GET both endpoints, combine into HealthResponse

async def check_authorization() -> dict:
    """Check if current token is authorized."""
    cfg = _get_config()
    url = f"{cfg['api_url'].rstrip('/')}/authorized"
    # POST, return AuthorizationResponse data
```

### server.py — 3 new tools

```python
Tool(name="ols-feedback", description="Submit feedback on an OLS response", ...)
Tool(name="ols-health", description="Check OLS service health (readiness + liveness)", ...)
Tool(name="ols-authorize", description="Verify user authorization with OLS", ...)
```