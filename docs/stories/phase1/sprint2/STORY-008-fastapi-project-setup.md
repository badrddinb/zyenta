# Story 008: FastAPI Project Setup

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-008 |
| **Sprint** | Phase 1 - Sprint 2 |
| **Title** | FastAPI Project Setup for Genesis Engine |
| **Priority** | Critical |
| **Story Points** | 8 |
| **Status** | Ready for Development |
| **Dependencies** | Sprint 1 Complete |

## User Story

**As a** developer
**I want** a properly configured FastAPI project for the Genesis Engine service
**So that** I can build AI agents with a robust Python API framework

## Acceptance Criteria

### AC1: Directory Structure
- [ ] `services/genesis-engine/app/` with subdirectories for agents, api, core, models, services, workers
- [ ] `services/genesis-engine/app/agents/` with subdirectories for brand, scout, copywriter
- [ ] `services/genesis-engine/app/agents/scout/suppliers/` for supplier integrations
- [ ] `services/genesis-engine/tests/` with test_agents and test_api subdirectories
- [ ] All directories contain `__init__.py` files

### AC2: Python Project Configuration
- [ ] `pyproject.toml` with project metadata and dependencies
- [ ] `requirements.txt` with production dependencies
- [ ] `requirements-dev.txt` with development dependencies
- [ ] Tool configurations for black, ruff, mypy, pytest

### AC3: Configuration Management
- [ ] `app/core/config.py` with Pydantic Settings class
- [ ] Environment variables for database, Redis, S3, LLM APIs, Celery
- [ ] `@lru_cache` for settings singleton pattern
- [ ] Async database URL property

### AC4: Logging Setup
- [ ] `app/core/logging.py` with structlog configuration
- [ ] JSON rendering for production, console for development
- [ ] Proper log levels based on environment
- [ ] `get_logger()` helper function

### AC5: FastAPI Application
- [ ] `app/main.py` with lifespan context manager
- [ ] CORS middleware configured
- [ ] Health and API routers included
- [ ] `create_app()` factory function

### AC6: Health Endpoints
- [ ] `GET /health` returns status, version, environment
- [ ] `GET /ready` returns dependency health (database, redis, llm)
- [ ] Proper response models with Pydantic

### AC7: API Router Structure
- [ ] `app/api/v1/router.py` with API router
- [ ] `app/api/v1/stores.py` with store endpoints (placeholder)
- [ ] `app/api/v1/jobs.py` with job status endpoints (placeholder)
- [ ] Proper prefix and tags configuration

### AC8: Dockerfile
- [ ] Multi-stage build (builder + production)
- [ ] Python 3.11 slim base image
- [ ] Non-root user for security
- [ ] Health check configured
- [ ] Exposed on port 8001

## Technical Details

### Directory Structure

```
services/genesis-engine/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── brand/
│   │   │   └── __init__.py
│   │   ├── scout/
│   │   │   ├── __init__.py
│   │   │   └── suppliers/
│   │   │       └── __init__.py
│   │   └── copywriter/
│   │       └── __init__.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── health.py
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── router.py
│   │       ├── stores.py
│   │       └── jobs.py
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py
│   │   └── logging.py
│   ├── models/
│   │   └── __init__.py
│   ├── services/
│   │   └── __init__.py
│   └── workers/
│       └── __init__.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_agents/
│   │   └── __init__.py
│   └── test_api/
│       └── __init__.py
├── pyproject.toml
├── requirements.txt
├── requirements-dev.txt
└── Dockerfile
```

### Key Dependencies

```
# Core
fastapi>=0.109.0
uvicorn[standard]>=0.27.0
pydantic>=2.6.0
pydantic-settings>=2.1.0

# HTTP Client
httpx>=0.26.0

# LLM
openai>=1.12.0
anthropic>=0.18.0

# Task Queue
celery[redis]>=5.3.0
redis>=5.0.0

# Database
sqlalchemy>=2.0.0
asyncpg>=0.29.0
psycopg2-binary>=2.9.0

# Storage
boto3>=1.34.0

# Utilities
tenacity>=8.2.0
structlog>=24.1.0
```

### Settings Configuration

```python
class Settings(BaseSettings):
    app_name: str = "Genesis Engine"
    environment: Literal["development", "staging", "production"]
    database_url: PostgresDsn
    redis_url: RedisDsn
    openai_api_key: str | None
    anthropic_api_key: str | None
    s3_endpoint: str
    s3_bucket: str
    # ... etc
```

## Definition of Done

- [ ] All directories and files created as specified
- [ ] `pip install -r requirements.txt` succeeds
- [ ] `uvicorn app.main:app --reload` starts without errors
- [ ] `GET /health` returns 200 with correct JSON
- [ ] `GET /ready` returns 200
- [ ] `GET /docs` shows OpenAPI documentation (in debug mode)
- [ ] Docker image builds: `docker build -t genesis-engine .`
- [ ] Docker container runs: `docker run -p 8001:8001 genesis-engine`
- [ ] Code passes `ruff check` and `black --check`
- [ ] Code review completed

## Testing

1. Install dependencies: `pip install -r requirements-dev.txt`
2. Run the server: `uvicorn app.main:app --reload --port 8001`
3. Test health: `curl http://localhost:8001/health`
4. Test readiness: `curl http://localhost:8001/ready`
5. Build Docker: `docker build -t genesis-engine .`
6. Run Docker: `docker run -p 8001:8001 genesis-engine`

## Notes

- This is the foundation for all Genesis Engine work
- Keep the API structure extensible for future endpoints
- Use async/await consistently throughout
- Log important events with structured logging

## Related Stories

- Depends on: Sprint 1 (foundation)
- Blocks: STORY-009 through STORY-015
