# Story 010: LLM Service Integration

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-010 |
| **Sprint** | Phase 1 - Sprint 2 |
| **Title** | LLM Service Integration (OpenAI & Anthropic) |
| **Priority** | Critical |
| **Story Points** | 5 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-008 |

## User Story

**As a** developer
**I want** a unified LLM service that abstracts multiple providers (OpenAI, Anthropic)
**So that** agents can use LLM capabilities without provider-specific code

## Acceptance Criteria

### AC1: LLM Provider Abstraction
- [ ] `LLMProvider` enum with OPENAI, ANTHROPIC values
- [ ] `LLMMessage` model with role (system, user, assistant) and content
- [ ] `LLMResponse` model with content, model, usage statistics
- [ ] `BaseLLMClient` abstract class defining the interface

### AC2: OpenAI Client
- [ ] `OpenAIClient` class implementing `BaseLLMClient`
- [ ] Async chat method using `AsyncOpenAI`
- [ ] Support for temperature, max_tokens, response_format
- [ ] JSON mode support (`response_format: {"type": "json_object"}`)
- [ ] Retry logic with exponential backoff (3 attempts)
- [ ] Usage tracking in response

### AC3: Anthropic Client
- [ ] `AnthropicClient` class implementing `BaseLLMClient`
- [ ] Async chat method using `AsyncAnthropic`
- [ ] System message extraction (Anthropic uses separate system param)
- [ ] Support for temperature, max_tokens
- [ ] Retry logic with exponential backoff (3 attempts)
- [ ] Usage tracking in response

### AC4: Unified LLM Service
- [ ] `LLMService` class that wraps provider clients
- [ ] Provider selection from config or parameter
- [ ] `chat()` method for standard completions
- [ ] `generate_json()` method for JSON-formatted responses
- [ ] Proper error handling and logging
- [ ] API key validation on initialization

### AC5: Configuration Integration
- [ ] Read `openai_api_key` from settings
- [ ] Read `anthropic_api_key` from settings
- [ ] Read `default_llm_provider` from settings
- [ ] Read `default_llm_model` from settings

## Technical Details

### Files to Create

1. **`app/services/llm.py`**
   - LLMProvider enum
   - LLMMessage model
   - LLMResponse model
   - BaseLLMClient abstract class
   - OpenAIClient implementation
   - AnthropicClient implementation
   - LLMService unified interface

### Interface Design

```python
class LLMService:
    async def chat(
        self,
        messages: list[LLMMessage],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
        response_format: dict | None = None,
    ) -> LLMResponse: ...

    async def generate_json(
        self,
        messages: list[LLMMessage],
        model: str | None = None,
        temperature: float = 0.7,
    ) -> str: ...
```

### Usage Example

```python
from app.services.llm import LLMService, LLMMessage

llm = LLMService()  # Uses default provider from config

response = await llm.chat([
    LLMMessage(role="system", content="You are a helpful assistant."),
    LLMMessage(role="user", content="Generate a brand name for a cyberpunk store."),
])

print(response.content)  # "NeonNest"
print(response.usage)    # {"total_tokens": 150}
```

### Retry Configuration

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
)
async def chat(...):
    ...
```

### Model Defaults

| Provider | Default Model |
|----------|---------------|
| OpenAI | gpt-4-turbo-preview |
| Anthropic | claude-3-sonnet-20240229 |

## Definition of Done

- [ ] All classes and methods implemented
- [ ] OpenAI integration works with real API (test with API key)
- [ ] Anthropic integration works with real API (test with API key)
- [ ] Provider switching works correctly
- [ ] JSON mode works for OpenAI
- [ ] Retry logic prevents transient failures
- [ ] Logging captures request/response metadata
- [ ] Error messages are clear when API keys missing
- [ ] Code review completed

## Testing

### Unit Tests

```python
@pytest.mark.asyncio
async def test_llm_service_chat(mock_openai_client):
    llm = LLMService()
    response = await llm.chat([
        LLMMessage(role="user", content="Hello")
    ])
    assert response.content is not None
```

### Integration Tests (Manual)

1. Set `OPENAI_API_KEY` in environment
2. Test OpenAI: `await llm.chat([LLMMessage(role="user", content="Say hi")])`
3. Set `ANTHROPIC_API_KEY` in environment
4. Test Anthropic: `LLMService(provider=LLMProvider.ANTHROPIC)`

## Notes

- Never log API keys or sensitive content
- Monitor token usage for cost management
- Consider rate limiting in production
- JSON mode only works with specific OpenAI models
- Anthropic doesn't have native JSON mode - prompt for JSON output

## Related Stories

- Depends on: STORY-008 (FastAPI Project Setup)
- Required by: STORY-011, STORY-012, STORY-013 (All agents use LLM service)
