# Story 009: Base Agent Class & Pydantic Models

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-009 |
| **Sprint** | Phase 1 - Sprint 2 |
| **Title** | Base Agent Class & Pydantic Models |
| **Priority** | Critical |
| **Story Points** | 5 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-008 |

## User Story

**As a** developer
**I want** a base agent class with common functionality and reusable Pydantic models
**So that** all AI agents follow a consistent pattern and share type definitions

## Acceptance Criteria

### AC1: Base Agent Interface
- [ ] `AgentStatus` enum with IDLE, RUNNING, COMPLETED, FAILED states
- [ ] `AgentContext` dataclass with store_id, user_id, job_id, niche, preferences, metadata
- [ ] `AgentResult` generic dataclass with success, data, error, duration_seconds, metadata
- [ ] `BaseAgent` abstract class with generic type parameters for input/output

### AC2: Base Agent Methods
- [ ] `name` abstract property for agent identification
- [ ] `description` abstract property for agent description
- [ ] `_execute()` abstract method for main logic (to be overridden)
- [ ] `run()` method with error handling, timing, and logging
- [ ] `_update_progress()` method for job progress updates

### AC3: Store Models
- [ ] `StoreStatus` enum (CREATING, PENDING_REVIEW, ACTIVE, PAUSED, ARCHIVED)
- [ ] `StorePreferences` model with style, tone, target_audience, price_range
- [ ] `CreateStoreRequest` model with user_id, niche, preferences
- [ ] `BrandIdentity` model with name, tagline, logo_prompt, colors, fonts, tone, story
- [ ] `StoreResponse` model with all store fields

### AC4: Product Models
- [ ] `SupplierProvider` enum (ALIEXPRESS, CJ_DROPSHIPPING, SPOCKET)
- [ ] `ProductStatus` enum (DRAFT, ACTIVE, OUT_OF_STOCK, ARCHIVED)
- [ ] `ProductImage` model with url, alt_text, position
- [ ] `ProductVariant` model with sku, title, price, cost_price, stock, options
- [ ] `ScoutedProduct` model with full product data and winning_score
- [ ] `EnhancedProduct` model with AI-generated content

### AC5: Job Models
- [ ] `JobStatus` enum (PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED)
- [ ] `AgentType` enum (BRAND_GENERATOR, PRODUCT_SCOUT, COPYWRITER, IMAGE_PROCESSOR)
- [ ] `JobResponse` model with id, status, progress, current_step, error, timestamps
- [ ] `JobProgress` model for progress updates

## Technical Details

### Files to Create

1. **`app/agents/base.py`**
   - AgentStatus enum
   - AgentContext dataclass
   - AgentResult generic dataclass
   - BaseAgent abstract class

2. **`app/models/store.py`**
   - StoreStatus enum
   - StorePreferences model
   - CreateStoreRequest model
   - BrandIdentity model
   - StoreResponse model

3. **`app/models/product.py`**
   - SupplierProvider enum
   - ProductStatus enum
   - ProductImage model
   - ProductVariant model
   - ScoutedProduct model
   - EnhancedProduct model

4. **`app/models/job.py`**
   - JobStatus enum
   - AgentType enum
   - JobResponse model
   - JobProgress model

### BaseAgent Pattern

```python
class BaseAgent(ABC, Generic[InputT, OutputT]):
    def __init__(self, llm_service: LLMService | None = None):
        self.llm = llm_service or LLMService()
        self.status = AgentStatus.IDLE
        self.logger = get_logger(self.__class__.__name__)

    @property
    @abstractmethod
    def name(self) -> str: ...

    @abstractmethod
    async def _execute(self, input_data: InputT, context: AgentContext) -> OutputT: ...

    async def run(self, input_data: InputT, context: AgentContext) -> AgentResult[OutputT]:
        # Error handling, timing, logging wrapper
        ...
```

### Key Model Relationships

```
CreateStoreRequest
    └── StorePreferences

ScoutedProduct
    ├── ProductImage[]
    └── ProductVariant[]

EnhancedProduct
    └── ScoutedProduct (source_product)

AgentResult<T>
    └── T (data: BrandOutput | ScoutOutput | CopywriterOutput)
```

## Definition of Done

- [ ] All model files created with proper types
- [ ] BaseAgent class is abstract and cannot be instantiated directly
- [ ] Generic types work correctly (InputT, OutputT)
- [ ] All enums match database schema values
- [ ] Decimal used for monetary fields
- [ ] All models have proper Field descriptions
- [ ] Models can be serialized/deserialized correctly
- [ ] Code review completed

## Testing

```python
# Test model creation
from app.models.product import ScoutedProduct, SupplierProvider
from decimal import Decimal

product = ScoutedProduct(
    source_provider=SupplierProvider.ALIEXPRESS,
    source_id="test123",
    original_title="Test Product",
    description="Test description",
    cost_price=Decimal("10.00"),
    suggested_price=Decimal("25.00"),
)

assert product.winning_score == 0.0  # Default value
```

## Notes

- Use Decimal for all monetary values to avoid floating point issues
- Keep models aligned with database schema
- Use proper type hints for IDE support
- Consider validation rules for fields (min/max lengths, ranges)

## Related Stories

- Depends on: STORY-008 (FastAPI Project Setup)
- Required by: STORY-011, STORY-012, STORY-013 (Agent implementations)
