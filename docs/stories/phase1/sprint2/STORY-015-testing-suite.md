# Story 015: Testing Suite

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-015 |
| **Sprint** | Phase 1 - Sprint 2 |
| **Title** | Testing Suite for Genesis Engine |
| **Priority** | High |
| **Story Points** | 5 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-008 through STORY-014 |

## User Story

**As a** developer
**I want** a comprehensive testing suite for the Genesis Engine
**So that** I can verify all agents and services work correctly

## Acceptance Criteria

### AC1: Test Configuration
- [ ] `conftest.py` with shared fixtures
- [ ] Mock LLM service fixture
- [ ] Sample agent context fixture
- [ ] pytest-asyncio configured for async tests

### AC2: Brand Agent Tests
- [ ] Test successful brand generation
- [ ] Test handling of invalid LLM response
- [ ] Test color validation
- [ ] Test with various niche inputs

### AC3: Scout Agent Tests
- [ ] Test keyword generation
- [ ] Test product scoring logic
- [ ] Test product ranking
- [ ] Test deduplication logic
- [ ] Test with empty supplier list (mock fallback)

### AC4: Copywriter Agent Tests
- [ ] Test product copy generation
- [ ] Test page generation (about, privacy, terms, refund)
- [ ] Test slug generation
- [ ] Test fallback handling

### AC5: API Tests
- [ ] Test health endpoint
- [ ] Test readiness endpoint
- [ ] Test create store endpoint
- [ ] Test job status endpoint

### AC6: Test Coverage
- [ ] Minimum 70% code coverage
- [ ] Coverage report generated
- [ ] Critical paths covered

## Technical Details

### Files to Create

1. **`tests/conftest.py`**
   - Fixtures for mocking and setup

2. **`tests/test_agents/test_brand_agent.py`**
   - Brand agent tests

3. **`tests/test_agents/test_scout_agent.py`**
   - Scout agent and ranking tests

4. **`tests/test_agents/test_copywriter_agent.py`**
   - Copywriter agent tests

5. **`tests/test_api/test_health.py`**
   - Health endpoint tests

6. **`tests/test_api/test_stores.py`**
   - Store creation tests

### Test Fixtures

```python
# conftest.py

@pytest.fixture
def mock_llm_service():
    """Mock LLM service for testing."""
    mock = AsyncMock(spec=LLMService)
    mock.chat.return_value = LLMResponse(
        content='{"name": "TestBrand"}',
        model="test-model",
        usage={"total_tokens": 100},
    )
    mock.generate_json.return_value = '{"name": "TestBrand", "tagline": "Test"}'
    return mock

@pytest.fixture
def sample_context():
    """Sample agent context for testing."""
    return AgentContext(
        store_id="test_store_123",
        user_id="test_user_456",
        job_id="test_job_789",
        niche="Test niche",
    )

@pytest.fixture
def sample_brand_output():
    """Sample brand output for testing."""
    return {
        "name": "TestBrand",
        "tagline": "Test Tagline",
        "colors": {
            "primary": "#FF0000",
            "secondary": "#00FF00",
            "accent": "#0000FF",
            "background": "#FFFFFF",
            "text": "#000000",
            "muted": "#888888",
        },
        "fonts": {"heading": "Inter", "body": "Inter"},
        "voice": {
            "tone": "professional",
            "personality": ["friendly"],
            "writing_style": "Clear and concise",
        },
        "story": "Test story...",
        "domain_suggestions": ["test.com"],
        "logo_prompt": "A simple logo",
    }
```

### Test Examples

```python
# test_brand_agent.py

@pytest.mark.asyncio
async def test_brand_agent_success(mock_llm_service, sample_context, sample_brand_output):
    mock_llm_service.generate_json.return_value = json.dumps(sample_brand_output)

    agent = BrandAgent(llm_service=mock_llm_service)
    result = await agent.run(BrandInput(niche="Test niche"), sample_context)

    assert result.success is True
    assert result.data.name == "TestBrand"
    assert result.data.colors.primary == "#FF0000"


@pytest.mark.asyncio
async def test_brand_agent_invalid_json(mock_llm_service, sample_context):
    mock_llm_service.generate_json.return_value = "not valid json"

    agent = BrandAgent(llm_service=mock_llm_service)
    result = await agent.run(BrandInput(niche="Test"), sample_context)

    assert result.success is False
    assert "Invalid" in result.error
```

```python
# test_scout_agent.py

def test_product_ranker_high_score():
    """High-quality product should score high."""
    ranker = ProductRanker()
    product = ScoutedProduct(
        source_provider=SupplierProvider.ALIEXPRESS,
        source_id="test",
        original_title="Test",
        description="Test",
        cost_price=Decimal("10.00"),
        suggested_price=Decimal("40.00"),  # 300% margin
        images=[ProductImage(url="http://test.com/1.jpg") for _ in range(5)],
        review_count=1000,
        rating=4.9,
        order_count=5000,
        shipping_time_days=7,
    )

    score = ranker.calculate_score(product)
    assert score > 80


def test_product_ranker_ranking_order():
    """Products should be ranked by score."""
    ranker = ProductRanker()
    products = [low_score_product, high_score_product]

    ranked = ranker.rank_products(products, top_n=2)

    assert ranked[0].winning_score > ranked[1].winning_score
```

```python
# test_health.py

@pytest.mark.asyncio
async def test_health_endpoint():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/health")

    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "healthy"
    assert "version" in data
```

### Running Tests

```bash
# Run all tests
pytest tests/ -v

# Run with coverage
pytest tests/ --cov=app --cov-report=html

# Run specific test file
pytest tests/test_agents/test_brand_agent.py -v

# Run tests matching pattern
pytest tests/ -k "brand" -v
```

## Definition of Done

- [ ] All test files created
- [ ] Mock LLM service works correctly
- [ ] All agent tests pass
- [ ] All API tests pass
- [ ] Coverage report shows >70% coverage
- [ ] Tests run in CI pipeline
- [ ] Code review completed

## Testing Matrix

| Component | Unit Tests | Integration Tests |
|-----------|------------|-------------------|
| Brand Agent | ✓ | ✓ (with LLM) |
| Scout Agent | ✓ | ✓ (with mock supplier) |
| Copywriter Agent | ✓ | ✓ (with LLM) |
| Product Ranker | ✓ | N/A |
| Health API | ✓ | ✓ |
| Stores API | ✓ | ✓ |
| Orchestrator | ✓ | ✓ |

## Notes

- Use pytest-asyncio for async test support
- Mock external services (LLM, suppliers) in unit tests
- Integration tests can use real services in CI
- Keep tests fast - mock expensive operations
- Use fixtures for common test data

## Related Stories

- Tests all components from STORY-008 through STORY-014
- Required for CI/CD pipeline validation
