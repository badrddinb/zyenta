# Story 012: Product Scout Agent

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-012 |
| **Sprint** | Phase 1 - Sprint 2 |
| **Title** | Product Scout Agent with Supplier Integration |
| **Priority** | High |
| **Story Points** | 13 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-009, STORY-010 |

## User Story

**As a** store creator
**I want** an AI agent that finds and ranks winning products from suppliers
**So that** my store is stocked with products likely to sell well

## Acceptance Criteria

### AC1: Supplier Base Class
- [ ] `SupplierConfig` model with api_key, api_secret, account_id
- [ ] `BaseSupplier` abstract class with name property
- [ ] Abstract `search_products()` method returning `list[ScoutedProduct]`
- [ ] Abstract `get_product_details()` method
- [ ] `validate_connection()` method for testing API access

### AC2: AliExpress Supplier (Mock)
- [ ] `AliExpressSupplier` implementing `BaseSupplier`
- [ ] HTTP client with timeout configuration
- [ ] `search_products()` with retry logic
- [ ] Mock data generation for development
- [ ] Proper logging of search operations

### AC3: Product Ranking System
- [ ] `ScoringWeights` dataclass with configurable weights
- [ ] `ProductRanker` class with scoring methods
- [ ] Scoring factors: rating, reviews, orders, margin, shipping, images
- [ ] `calculate_score()` returning 0-100 score
- [ ] `rank_products()` sorting and returning top N

### AC4: Scout Input/Output Schemas
- [ ] `SupplierAccountInfo` model for account details
- [ ] `ScoutInput` with niche, supplier_accounts, max_products, min_rating, min_orders
- [ ] `ScoutOutput` with products, keywords_used, total_found, suppliers_searched

### AC5: Scout Agent Implementation
- [ ] Extends `BaseAgent[ScoutInput, ScoutOutput]`
- [ ] LLM-based keyword generation from niche
- [ ] Multi-supplier product search
- [ ] Filtering by minimum requirements
- [ ] Ranking and selection of top products
- [ ] Deduplication by title similarity
- [ ] Progress updates throughout process

### AC6: Keyword Generation
- [ ] System prompt for product research expert
- [ ] Generate 15-20 relevant search keywords
- [ ] Include niche terms, categories, specific products, trends

## Technical Details

### Files to Create

1. **`app/agents/scout/suppliers/base.py`**
   - SupplierConfig
   - BaseSupplier abstract class

2. **`app/agents/scout/suppliers/aliexpress.py`**
   - AliExpressSupplier implementation

3. **`app/agents/scout/ranking.py`**
   - ScoringWeights
   - ProductRanker

4. **`app/agents/scout/schemas.py`**
   - SupplierAccountInfo
   - ScoutInput
   - ScoutOutput

5. **`app/agents/scout/agent.py`**
   - ScoutAgent

### Scoring Weights (Default)

| Factor | Weight | Description |
|--------|--------|-------------|
| Rating | 0.20 | Product rating (4.5+ stars preferred) |
| Reviews | 0.15 | Review count (social proof) |
| Orders | 0.20 | Order count (proven demand) |
| Margin | 0.25 | Profit margin potential |
| Shipping | 0.10 | Shipping time (faster better) |
| Images | 0.10 | Image count and quality |

### Scoring Logic

```python
def _score_rating(self, rating: float) -> float:
    if rating >= 4.8: return 100
    elif rating >= 4.5: return 90
    elif rating >= 4.0: return 70
    elif rating >= 3.5: return 50
    else: return 30

def _score_margin(self, cost: float, suggested: float) -> float:
    margin = (suggested - cost) / cost * 100
    if margin >= 150: return 100  # 150%+ markup
    elif margin >= 100: return 80
    elif margin >= 50: return 60
    # ...
```

### Generation Flow

```
Input: { niche: "Cyberpunk home decor", max_products: 20 }
                    ↓
        [1. Generate Keywords]
        - LLM generates 15-20 search keywords
                    ↓
        [2. Search Suppliers]
        - Query each connected supplier API
        - Fetch 100+ candidate products
                    ↓
        [3. Filter Products]
        - Remove products below min_rating
        - Remove products below min_orders
                    ↓
        [4. Score & Rank]
        - Calculate winning score for each
        - Sort by score descending
                    ↓
        [5. Deduplicate]
        - Remove duplicate products
        - Return top N products
                    ↓
Output: ScoutOutput { products: [...], keywords_used: [...], ... }
```

### Mock Product Generation

For development without real supplier APIs:

```python
def _generate_mock_products(self, keywords: list[str], count: int):
    # Generate realistic mock products with:
    # - Varied prices ($5-$50)
    # - Random ratings (3.5-5.0)
    # - Varied order counts
    # - Multiple images
    # - Shipping times (7-30 days)
```

## Definition of Done

- [ ] BaseSupplier abstraction allows future supplier additions
- [ ] AliExpress mock generates realistic products
- [ ] ProductRanker correctly scores and ranks products
- [ ] Keyword generation produces relevant search terms
- [ ] Deduplication removes similar products
- [ ] Progress updates at each major stage
- [ ] Test coverage for ranking algorithms
- [ ] Code review completed

## Testing

### Unit Tests

```python
def test_product_ranker_scoring():
    ranker = ProductRanker()
    product = ScoutedProduct(
        rating=4.7,
        review_count=500,
        order_count=1000,
        cost_price=Decimal("10.00"),
        suggested_price=Decimal("30.00"),
        shipping_time_days=10,
        images=[...],
    )
    score = ranker.calculate_score(product)
    assert 70 < score < 100  # High-quality product
```

### Integration Tests

```python
@pytest.mark.asyncio
async def test_scout_agent_finds_products(mock_llm_service, sample_context):
    agent = ScoutAgent(llm_service=mock_llm_service)
    result = await agent.run(
        ScoutInput(niche="Home decor", max_products=10),
        sample_context
    )
    assert len(result.data.products) <= 10
    assert all(p.winning_score > 0 for p in result.data.products)
```

## Notes

- AliExpress API requires OAuth - mock for now
- Real supplier integration in future sprints
- Consider adding CJ Dropshipping and Spocket suppliers
- Product scoring weights may need tuning based on real data

## Related Stories

- Depends on: STORY-009 (Base Agent), STORY-010 (LLM Service)
- Used by: STORY-013 (Copywriter uses products), STORY-014 (Orchestrator)
