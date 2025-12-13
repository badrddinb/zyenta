# Phase 1 - Sprint 2: Genesis Engine Core

## Sprint Overview

| Field | Value |
|-------|-------|
| **Sprint** | 2 |
| **Phase** | 1 - "Creator" MVP |
| **Duration** | 2 weeks |
| **Total Story Points** | 60 |
| **Status** | Ready for Development |
| **Prerequisites** | Sprint 1 Complete |

## Sprint Goal

Build the Genesis Engine - the AI-powered core that creates stores autonomously. This includes the FastAPI service, three AI agents (Brand, Scout, Copywriter), and an orchestrator to coordinate them.

## Stories

| ID | Title | Priority | Points | Dependencies | Status |
|----|-------|----------|--------|--------------|--------|
| [STORY-008](./STORY-008-fastapi-project-setup.md) | FastAPI Project Setup | Critical | 8 | Sprint 1 | Ready |
| [STORY-009](./STORY-009-base-agent-models.md) | Base Agent Class & Models | Critical | 5 | STORY-008 | Ready |
| [STORY-010](./STORY-010-llm-service.md) | LLM Service Integration | Critical | 5 | STORY-008 | Ready |
| [STORY-011](./STORY-011-brand-agent.md) | Brand Agent | High | 8 | STORY-009, STORY-010 | Ready |
| [STORY-012](./STORY-012-scout-agent.md) | Product Scout Agent | High | 13 | STORY-009, STORY-010 | Ready |
| [STORY-013](./STORY-013-copywriter-agent.md) | Copywriter Agent | High | 8 | STORY-009, STORY-010, STORY-011 | Ready |
| [STORY-014](./STORY-014-orchestrator.md) | Genesis Orchestrator | High | 8 | STORY-011, STORY-012, STORY-013 | Ready |
| [STORY-015](./STORY-015-testing-suite.md) | Testing Suite | High | 5 | All above | Ready |

## Dependency Graph

```
STORY-008 (FastAPI Setup)
    │
    ├──► STORY-009 (Base Agent & Models)
    │         │
    │         ├──► STORY-011 (Brand Agent) ─────┐
    │         │                                  │
    │         ├──► STORY-012 (Scout Agent) ─────┼──► STORY-014 (Orchestrator)
    │         │                                  │          │
    │         └──► STORY-013 (Copywriter) ──────┘          │
    │                    ▲                                  │
    │                    │                                  ▼
    └──► STORY-010 (LLM Service) ─────────────────► STORY-015 (Testing)
```

## Suggested Implementation Order

1. **STORY-008** - FastAPI Project Setup (foundation)
2. **STORY-009** - Base Agent Class & Models (framework)
3. **STORY-010** - LLM Service Integration (can parallel with 009)
4. **STORY-011** - Brand Agent (first agent)
5. **STORY-012** - Product Scout Agent (can parallel with 011)
6. **STORY-013** - Copywriter Agent (needs brand output)
7. **STORY-014** - Genesis Orchestrator (coordinates all)
8. **STORY-015** - Testing Suite (validates everything)

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Genesis Engine Service                     │
│                     (FastAPI + Python)                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                  Genesis Orchestrator                    │ │
│  │  Coordinates: Brand → Scout → Copywriter                │ │
│  └─────────────────────────────────────────────────────────┘ │
│                             │                                 │
│        ┌────────────────────┼────────────────────┐           │
│        ▼                    ▼                    ▼           │
│  ┌───────────┐       ┌───────────┐       ┌───────────┐      │
│  │   Brand   │       │   Scout   │       │Copywriter │      │
│  │   Agent   │       │   Agent   │       │   Agent   │      │
│  └─────┬─────┘       └─────┬─────┘       └─────┬─────┘      │
│        │                   │                   │             │
│        └───────────────────┼───────────────────┘             │
│                            ▼                                 │
│                   ┌─────────────────┐                        │
│                   │   LLM Service   │                        │
│                   │ (OpenAI/Claude) │                        │
│                   └─────────────────┘                        │
│                                                               │
├─────────────────────────────────────────────────────────────┤
│  API Layer: /health, /ready, /api/v1/stores, /api/v1/jobs   │
└─────────────────────────────────────────────────────────────┘
```

## Key Technical Components

### Agent Pattern

```python
class BaseAgent(ABC, Generic[InputT, OutputT]):
    async def run(self, input_data: InputT, context: AgentContext) -> AgentResult[OutputT]:
        # Error handling, timing, logging
        result = await self._execute(input_data, context)
        return AgentResult(success=True, data=result)
```

### Genesis Flow

```
POST /api/v1/stores { niche: "Cyberpunk home decor" }
    │
    ▼
Brand Agent → { name: "NeonNest", colors: {...}, fonts: {...} }
    │
    ▼
Scout Agent → { products: [20 winning products with scores] }
    │
    ▼
Copywriter Agent → { enhanced_products: [...], pages: [about, privacy, terms, refund] }
    │
    ▼
Response: { store_id: "store_xxx", job_id: "job_xxx" }
```

## Definition of Done for Sprint

- [ ] All 8 stories completed and code reviewed
- [ ] FastAPI app starts and serves endpoints
- [ ] All three agents generate valid output
- [ ] Orchestrator coordinates full genesis flow
- [ ] Tests pass with >70% coverage
- [ ] Docker image builds and runs
- [ ] API documentation accessible at /docs

## Technical Stack

| Component | Technology |
|-----------|------------|
| Framework | FastAPI 0.109+ |
| Python | 3.11+ |
| LLM Providers | OpenAI, Anthropic |
| Task Queue | Celery (prepared, not fully integrated) |
| HTTP Client | httpx |
| Validation | Pydantic 2.6+ |
| Logging | structlog |
| Testing | pytest, pytest-asyncio |

## Verification Checklist

After completing all stories:

### Service Health
- [ ] `uvicorn app.main:app --reload` starts without errors
- [ ] `GET /health` returns 200 with version info
- [ ] `GET /ready` returns 200

### Agents
- [ ] Brand Agent generates valid brand identity
- [ ] Scout Agent returns scored products
- [ ] Copywriter Agent generates descriptions and pages

### Integration
- [ ] `POST /api/v1/stores` triggers full genesis flow
- [ ] All agents work together via orchestrator
- [ ] LLM service connects to API providers

### Testing & Docker
- [ ] `pytest tests/ -v` passes all tests
- [ ] `docker build -t genesis-engine .` succeeds
- [ ] `docker run -p 8001:8001 genesis-engine` runs

## Next Sprint

After completing Sprint 2, proceed to **Sprint 3: Media Studio**:
- Image processing pipeline
- Background removal (rembg)
- Watermark detection/removal
- Lifestyle background generation
- S3 storage integration

See `SPRINT3_MEDIA_STUDIO_ACTION_PLAN.md` for details.

## Notes

- Keep agents modular for easy testing and maintenance
- Use mock suppliers for development (real APIs in later sprints)
- Log all LLM calls for debugging and cost tracking
- Consider caching for repeated queries
