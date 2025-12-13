# Story 014: Genesis Orchestrator

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-014 |
| **Sprint** | Phase 1 - Sprint 2 |
| **Title** | Genesis Orchestrator Service |
| **Priority** | High |
| **Story Points** | 8 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-011, STORY-012, STORY-013 |

## User Story

**As a** system
**I want** an orchestrator that coordinates all Genesis agents
**So that** a complete store can be generated from a single niche input

## Acceptance Criteria

### AC1: Orchestrator Structure
- [ ] `GenesisResult` dataclass with store_id, job_id, success, error
- [ ] `GenesisOrchestrator` class coordinating all agents
- [ ] Agent initialization in constructor
- [ ] `start_genesis()` async method as main entry point

### AC2: Genesis Flow
- [ ] Generate unique store_id and job_id
- [ ] Create AgentContext with all necessary data
- [ ] Step 1: Run Brand Agent
- [ ] Step 2: Run Scout Agent
- [ ] Step 3: Run Copywriter Agent with brand and products
- [ ] Return comprehensive result

### AC3: Error Handling
- [ ] Catch and log errors at each step
- [ ] Return failure result with error message
- [ ] TODO markers for database status updates
- [ ] Graceful handling of agent failures

### AC4: Integration with API
- [ ] `POST /api/v1/stores` endpoint calls orchestrator
- [ ] Returns 202 Accepted with store_id and job_id
- [ ] Provides message for monitoring via /jobs/{job_id}

### AC5: Logging
- [ ] Log start of each agent execution
- [ ] Log completion with product/page counts
- [ ] Log errors with full context
- [ ] Use structured logging throughout

## Technical Details

### Files to Create/Update

1. **`app/services/orchestrator.py`**
   - GenesisResult dataclass
   - GenesisOrchestrator class

2. **`app/api/v1/stores.py`** (update)
   - Integrate orchestrator into create_store endpoint

### Orchestration Flow

```
start_genesis(user_id, niche, preferences)
                    ↓
        [1. Generate IDs]
        - store_id: "store_{uuid}"
        - job_id: "job_{uuid}"
                    ↓
        [2. Create Context]
        - AgentContext with all params
                    ↓
        [3. Brand Agent]
        - Input: BrandInput(niche, style, tone, audience)
        - Output: BrandOutput → BrandIdentity
                    ↓
        [4. Scout Agent]
        - Input: ScoutInput(niche, accounts, max_products)
        - Output: ScoutOutput → products[]
                    ↓
        [5. Copywriter Agent]
        - Input: CopywriterInput(store_name, niche, brand, products)
        - Output: CopywriterOutput → enhanced products + pages
                    ↓
        [Return GenesisResult]
        - success: true/false
        - store_id, job_id
        - error message if failed
```

### Agent Coordination

```python
class GenesisOrchestrator:
    def __init__(self):
        self.brand_agent = BrandAgent()
        self.scout_agent = ScoutAgent()
        self.copywriter_agent = CopywriterAgent()

    async def start_genesis(
        self,
        user_id: str,
        niche: str,
        preferences: StorePreferences | None = None,
    ) -> GenesisResult:
        # Generate IDs
        store_id = f"store_{uuid.uuid4().hex[:12]}"
        job_id = f"job_{uuid.uuid4().hex[:12]}"

        # Create context
        context = AgentContext(
            store_id=store_id,
            user_id=user_id,
            job_id=job_id,
            niche=niche,
        )

        try:
            # Step 1: Brand
            brand_result = await self.brand_agent.run(brand_input, context)
            if not brand_result.success:
                raise Exception(f"Brand generation failed: {brand_result.error}")

            # Step 2: Scout
            scout_result = await self.scout_agent.run(scout_input, context)
            # ...

            # Step 3: Copywriter
            copy_result = await self.copywriter_agent.run(copy_input, context)
            # ...

            return GenesisResult(store_id=store_id, job_id=job_id, success=True)

        except Exception as e:
            return GenesisResult(store_id=store_id, job_id=job_id, success=False, error=str(e))
```

### API Response

```python
class CreateStoreResponse(BaseModel):
    store_id: str
    job_id: str
    message: str

# Response:
{
    "store_id": "store_a1b2c3d4e5f6",
    "job_id": "job_x1y2z3w4v5u6",
    "message": "Store creation started. Monitor progress via /jobs/{job_id}"
}
```

### TODO Placeholders

```python
# TODO: Create store record in database with CREATING status
# TODO: Get supplier accounts from user's connected accounts
# TODO: Save all generated data to database
# TODO: Update store status to PENDING_REVIEW
# TODO: Update job status to FAILED
```

## Definition of Done

- [ ] GenesisOrchestrator coordinates all three agents
- [ ] Unique IDs generated for each genesis run
- [ ] Errors in any agent stop the pipeline
- [ ] API endpoint returns 202 with IDs
- [ ] Comprehensive logging at each step
- [ ] TODO markers for database integration
- [ ] Test coverage for orchestrator flow
- [ ] Code review completed

## Testing

### Unit Tests

```python
@pytest.mark.asyncio
async def test_orchestrator_success(mock_agents):
    orchestrator = GenesisOrchestrator()
    result = await orchestrator.start_genesis(
        user_id="user_123",
        niche="Cyberpunk home decor",
    )

    assert result.success is True
    assert result.store_id.startswith("store_")
    assert result.job_id.startswith("job_")
```

### Integration Tests

```python
@pytest.mark.asyncio
async def test_api_create_store():
    response = await client.post("/api/v1/stores", json={
        "user_id": "user_123",
        "niche": "Pet accessories",
    })

    assert response.status_code == 202
    data = response.json()
    assert "store_id" in data
    assert "job_id" in data
```

## Notes

- This is the main entry point for store generation
- Future: Add Celery task for background processing
- Future: Implement database persistence
- Future: Add WebSocket for real-time progress updates
- Consider retry logic for individual agents

## Related Stories

- Depends on: STORY-011 (Brand), STORY-012 (Scout), STORY-013 (Copywriter)
- Enables: End-to-end store generation flow
