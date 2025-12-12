# Sprint 2: Genesis Engine Core - Actionable Implementation Plan

## Overview

This document provides step-by-step implementation instructions for Sprint 2 of Zyenta's Phase 1 MVP. Sprint 2 builds the **Genesis Engine** - the AI-powered core that creates stores autonomously.

**Objective:** Build a fully functional Genesis Engine with Brand, Scout, and Copywriter agents that can take a niche prompt and generate a complete store configuration.

**Prerequisites:** Sprint 1 (Foundation) must be completed with:
- Monorepo configured and working
- Database schema deployed
- Docker infrastructure running
- Shared packages available

---

## Task 1: FastAPI Project Setup

### 1.1 Directory Structure

Create the following structure:

```bash
mkdir -p services/genesis-engine/app/{agents/{brand,scout,copywriter},api/v1,core,models,services,workers}
mkdir -p services/genesis-engine/app/agents/scout/suppliers
mkdir -p services/genesis-engine/tests/{test_agents,test_api}
touch services/genesis-engine/app/__init__.py
touch services/genesis-engine/app/agents/__init__.py
touch services/genesis-engine/app/agents/brand/__init__.py
touch services/genesis-engine/app/agents/scout/__init__.py
touch services/genesis-engine/app/agents/scout/suppliers/__init__.py
touch services/genesis-engine/app/agents/copywriter/__init__.py
touch services/genesis-engine/app/api/__init__.py
touch services/genesis-engine/app/api/v1/__init__.py
touch services/genesis-engine/app/core/__init__.py
touch services/genesis-engine/app/models/__init__.py
touch services/genesis-engine/app/services/__init__.py
touch services/genesis-engine/app/workers/__init__.py
touch services/genesis-engine/tests/__init__.py
touch services/genesis-engine/tests/test_agents/__init__.py
touch services/genesis-engine/tests/test_api/__init__.py
```

### 1.2 Python Project Configuration

**File: `services/genesis-engine/pyproject.toml`**
```toml
[project]
name = "genesis-engine"
version = "0.1.0"
description = "Zyenta Genesis Engine - AI Store Generation Service"
readme = "README.md"
requires-python = ">=3.11"
license = { text = "MIT" }
authors = [{ name = "Zyenta Team" }]

dependencies = [
    "fastapi>=0.109.0",
    "uvicorn[standard]>=0.27.0",
    "pydantic>=2.6.0",
    "pydantic-settings>=2.1.0",
    "httpx>=0.26.0",
    "openai>=1.12.0",
    "anthropic>=0.18.0",
    "celery[redis]>=5.3.0",
    "redis>=5.0.0",
    "sqlalchemy>=2.0.0",
    "asyncpg>=0.29.0",
    "psycopg2-binary>=2.9.0",
    "boto3>=1.34.0",
    "python-multipart>=0.0.9",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "tenacity>=8.2.0",
    "structlog>=24.1.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=4.1.0",
    "httpx>=0.26.0",
    "black>=24.1.0",
    "ruff>=0.2.0",
    "mypy>=1.8.0",
    "pre-commit>=3.6.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.black]
line-length = 88
target-version = ["py311"]

[tool.ruff]
line-length = 88
select = ["E", "F", "I", "N", "W", "UP"]
ignore = ["E501"]

[tool.mypy]
python_version = "3.11"
strict = true
ignore_missing_imports = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

**File: `services/genesis-engine/requirements.txt`**
```txt
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

# Security
python-jose[cryptography]>=3.3.0
passlib[bcrypt]>=1.7.4
python-multipart>=0.0.9

# Utilities
tenacity>=8.2.0
structlog>=24.1.0
```

**File: `services/genesis-engine/requirements-dev.txt`**
```txt
-r requirements.txt

# Testing
pytest>=8.0.0
pytest-asyncio>=0.23.0
pytest-cov>=4.1.0

# Linting & Formatting
black>=24.1.0
ruff>=0.2.0
mypy>=1.8.0
pre-commit>=3.6.0
```

### 1.3 Configuration Management

**File: `services/genesis-engine/app/core/config.py`**
```python
"""Application configuration using Pydantic Settings."""

from functools import lru_cache
from typing import Literal

from pydantic import Field, PostgresDsn, RedisDsn
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """Application settings loaded from environment variables."""

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    # Application
    app_name: str = "Genesis Engine"
    app_version: str = "0.1.0"
    environment: Literal["development", "staging", "production"] = "development"
    debug: bool = False
    log_level: str = "INFO"

    # API
    api_v1_prefix: str = "/api/v1"
    cors_origins: list[str] = ["http://localhost:3000"]

    # Database
    database_url: PostgresDsn = Field(
        default="postgresql://zyenta:zyenta_dev@localhost:5432/zyenta"
    )

    # Redis
    redis_url: RedisDsn = Field(default="redis://localhost:6379")

    # S3/MinIO
    s3_endpoint: str = "http://localhost:9000"
    s3_access_key: str = "minioadmin"
    s3_secret_key: str = "minioadmin"
    s3_bucket: str = "zyenta-media"
    s3_region: str = "us-east-1"

    # LLM APIs
    openai_api_key: str | None = None
    anthropic_api_key: str | None = None
    default_llm_provider: Literal["openai", "anthropic"] = "openai"
    default_llm_model: str = "gpt-4-turbo-preview"

    # Supplier APIs
    aliexpress_app_key: str | None = None
    aliexpress_app_secret: str | None = None

    # Celery
    celery_broker_url: str = "redis://localhost:6379/0"
    celery_result_backend: str = "redis://localhost:6379/1"

    # Security
    secret_key: str = "change-this-in-production"
    access_token_expire_minutes: int = 30

    @property
    def async_database_url(self) -> str:
        """Convert sync URL to async URL."""
        url = str(self.database_url)
        return url.replace("postgresql://", "postgresql+asyncpg://")


@lru_cache
def get_settings() -> Settings:
    """Get cached settings instance."""
    return Settings()


settings = get_settings()
```

**File: `services/genesis-engine/app/core/logging.py`**
```python
"""Structured logging configuration."""

import logging
import sys

import structlog


def setup_logging(log_level: str = "INFO") -> None:
    """Configure structured logging."""

    # Configure structlog
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.StackInfoRenderer(),
            structlog.dev.set_exc_info,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.dev.ConsoleRenderer()
            if log_level == "DEBUG"
            else structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(
            getattr(logging, log_level.upper())
        ),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )

    # Configure standard library logging
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=getattr(logging, log_level.upper()),
    )


def get_logger(name: str) -> structlog.BoundLogger:
    """Get a logger instance."""
    return structlog.get_logger(name)
```

### 1.4 FastAPI Application Entry Point

**File: `services/genesis-engine/app/main.py`**
```python
"""FastAPI application entry point."""

from contextlib import asynccontextmanager
from typing import AsyncGenerator

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.api.health import router as health_router
from app.api.v1.router import api_router
from app.core.config import settings
from app.core.logging import get_logger, setup_logging

logger = get_logger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    """Application lifespan handler."""
    # Startup
    setup_logging(settings.log_level)
    logger.info(
        "Starting Genesis Engine",
        version=settings.app_version,
        environment=settings.environment,
    )
    yield
    # Shutdown
    logger.info("Shutting down Genesis Engine")


def create_app() -> FastAPI:
    """Create and configure the FastAPI application."""
    app = FastAPI(
        title=settings.app_name,
        version=settings.app_version,
        description="Zyenta Genesis Engine - AI-powered store generation service",
        docs_url="/docs" if settings.debug else None,
        redoc_url="/redoc" if settings.debug else None,
        lifespan=lifespan,
    )

    # CORS middleware
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # Include routers
    app.include_router(health_router, tags=["Health"])
    app.include_router(api_router, prefix=settings.api_v1_prefix)

    return app


app = create_app()
```

### 1.5 Health Check Endpoint

**File: `services/genesis-engine/app/api/health.py`**
```python
"""Health check endpoints."""

from fastapi import APIRouter, status
from pydantic import BaseModel

from app.core.config import settings

router = APIRouter()


class HealthResponse(BaseModel):
    """Health check response model."""

    status: str
    version: str
    environment: str


class ReadinessResponse(BaseModel):
    """Readiness check response model."""

    status: str
    database: str
    redis: str
    llm: str


@router.get(
    "/health",
    response_model=HealthResponse,
    status_code=status.HTTP_200_OK,
)
async def health_check() -> HealthResponse:
    """Basic health check endpoint."""
    return HealthResponse(
        status="healthy",
        version=settings.app_version,
        environment=settings.environment,
    )


@router.get(
    "/ready",
    response_model=ReadinessResponse,
    status_code=status.HTTP_200_OK,
)
async def readiness_check() -> ReadinessResponse:
    """Readiness check with dependency status."""
    # TODO: Implement actual health checks for each service
    return ReadinessResponse(
        status="ready",
        database="connected",
        redis="connected",
        llm="configured",
    )
```

### 1.6 API Router Structure

**File: `services/genesis-engine/app/api/v1/router.py`**
```python
"""API v1 router configuration."""

from fastapi import APIRouter

from app.api.v1 import jobs, stores

api_router = APIRouter()

api_router.include_router(stores.router, prefix="/stores", tags=["Stores"])
api_router.include_router(jobs.router, prefix="/jobs", tags=["Jobs"])
```

**File: `services/genesis-engine/app/api/v1/stores.py`**
```python
"""Store management endpoints."""

from fastapi import APIRouter, BackgroundTasks, HTTPException, status
from pydantic import BaseModel, Field

from app.core.logging import get_logger
from app.models.store import CreateStoreRequest, StoreResponse
from app.services.orchestrator import GenesisOrchestrator

logger = get_logger(__name__)
router = APIRouter()


class CreateStoreResponse(BaseModel):
    """Response for store creation."""

    store_id: str
    job_id: str
    message: str


@router.post(
    "",
    response_model=CreateStoreResponse,
    status_code=status.HTTP_202_ACCEPTED,
)
async def create_store(
    request: CreateStoreRequest,
    background_tasks: BackgroundTasks,
) -> CreateStoreResponse:
    """
    Create a new store from a niche prompt.

    This triggers the Genesis Engine to:
    1. Generate brand identity
    2. Scout products from suppliers
    3. Generate product copy and pages
    """
    logger.info("Creating new store", niche=request.niche, user_id=request.user_id)

    try:
        orchestrator = GenesisOrchestrator()
        result = await orchestrator.start_genesis(
            user_id=request.user_id,
            niche=request.niche,
            preferences=request.preferences,
        )

        return CreateStoreResponse(
            store_id=result.store_id,
            job_id=result.job_id,
            message="Store creation started. Monitor progress via /jobs/{job_id}",
        )
    except Exception as e:
        logger.error("Failed to start store creation", error=str(e))
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Failed to start store creation",
        )


@router.get("/{store_id}", response_model=StoreResponse)
async def get_store(store_id: str) -> StoreResponse:
    """Get store details by ID."""
    # TODO: Implement database fetch
    raise HTTPException(
        status_code=status.HTTP_501_NOT_IMPLEMENTED,
        detail="Not implemented yet",
    )
```

**File: `services/genesis-engine/app/api/v1/jobs.py`**
```python
"""Job status endpoints."""

from fastapi import APIRouter, HTTPException, status

from app.core.logging import get_logger
from app.models.job import JobResponse, JobStatus

logger = get_logger(__name__)
router = APIRouter()


@router.get("/{job_id}", response_model=JobResponse)
async def get_job_status(job_id: str) -> JobResponse:
    """Get the status of a Genesis job."""
    # TODO: Implement job status retrieval from database/Redis
    logger.info("Fetching job status", job_id=job_id)

    # Placeholder response
    return JobResponse(
        id=job_id,
        status=JobStatus.PROCESSING,
        progress=50,
        current_step="Generating brand identity",
        started_at=None,
        completed_at=None,
    )


@router.post("/{job_id}/cancel", status_code=status.HTTP_200_OK)
async def cancel_job(job_id: str) -> dict:
    """Cancel a running Genesis job."""
    logger.info("Cancelling job", job_id=job_id)
    # TODO: Implement job cancellation
    return {"message": f"Job {job_id} cancellation requested"}
```

### 1.7 Dockerfile

**File: `services/genesis-engine/Dockerfile`**
```dockerfile
# Build stage
FROM python:3.11-slim as builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Production stage
FROM python:3.11-slim

WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

# Copy application code
COPY app/ ./app/

# Create non-root user
RUN useradd --create-home --shell /bin/bash appuser
USER appuser

# Expose port
EXPOSE 8001

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import httpx; httpx.get('http://localhost:8001/health')" || exit 1

# Run application
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8001"]
```

---

## Task 2: Base Agent Class Implementation

### 2.1 Base Agent Interface

**File: `services/genesis-engine/app/agents/base.py`**
```python
"""Base agent class with common functionality."""

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Any, Generic, TypeVar

from pydantic import BaseModel

from app.core.logging import get_logger
from app.services.llm import LLMService

logger = get_logger(__name__)


class AgentStatus(str, Enum):
    """Agent execution status."""

    IDLE = "idle"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"


InputT = TypeVar("InputT", bound=BaseModel)
OutputT = TypeVar("OutputT", bound=BaseModel)


@dataclass
class AgentContext:
    """Context passed to agents during execution."""

    store_id: str
    user_id: str
    job_id: str
    niche: str
    preferences: dict[str, Any] = field(default_factory=dict)
    metadata: dict[str, Any] = field(default_factory=dict)


@dataclass
class AgentResult(Generic[OutputT]):
    """Result returned by an agent."""

    success: bool
    data: OutputT | None = None
    error: str | None = None
    duration_seconds: float = 0.0
    metadata: dict[str, Any] = field(default_factory=dict)


class BaseAgent(ABC, Generic[InputT, OutputT]):
    """
    Abstract base class for all Genesis agents.

    Each agent follows a consistent pattern:
    1. Validate input
    2. Execute main logic (with retries)
    3. Return structured output
    """

    def __init__(self, llm_service: LLMService | None = None):
        """Initialize the agent."""
        self.llm = llm_service or LLMService()
        self.status = AgentStatus.IDLE
        self.logger = get_logger(self.__class__.__name__)

    @property
    @abstractmethod
    def name(self) -> str:
        """Agent name for logging and identification."""
        pass

    @property
    @abstractmethod
    def description(self) -> str:
        """Agent description."""
        pass

    @abstractmethod
    async def _execute(
        self,
        input_data: InputT,
        context: AgentContext,
    ) -> OutputT:
        """
        Execute the agent's main logic.

        Args:
            input_data: Validated input data
            context: Execution context

        Returns:
            Agent output data
        """
        pass

    async def run(
        self,
        input_data: InputT,
        context: AgentContext,
    ) -> AgentResult[OutputT]:
        """
        Run the agent with error handling and metrics.

        Args:
            input_data: Input data for the agent
            context: Execution context

        Returns:
            AgentResult with success/failure and data
        """
        start_time = datetime.utcnow()
        self.status = AgentStatus.RUNNING
        self.logger.info(
            f"Starting {self.name}",
            store_id=context.store_id,
            job_id=context.job_id,
        )

        try:
            result = await self._execute(input_data, context)
            duration = (datetime.utcnow() - start_time).total_seconds()
            self.status = AgentStatus.COMPLETED

            self.logger.info(
                f"Completed {self.name}",
                store_id=context.store_id,
                duration_seconds=duration,
            )

            return AgentResult(
                success=True,
                data=result,
                duration_seconds=duration,
            )

        except Exception as e:
            duration = (datetime.utcnow() - start_time).total_seconds()
            self.status = AgentStatus.FAILED
            error_msg = str(e)

            self.logger.error(
                f"Failed {self.name}",
                store_id=context.store_id,
                error=error_msg,
                duration_seconds=duration,
            )

            return AgentResult(
                success=False,
                error=error_msg,
                duration_seconds=duration,
            )

    def _update_progress(
        self,
        context: AgentContext,
        progress: int,
        step: str,
    ) -> None:
        """Update job progress (to be implemented with Redis/DB)."""
        self.logger.debug(
            "Progress update",
            job_id=context.job_id,
            progress=progress,
            step=step,
        )
        # TODO: Update progress in database/Redis
```

### 2.2 Pydantic Models

**File: `services/genesis-engine/app/models/store.py`**
```python
"""Store-related Pydantic models."""

from datetime import datetime
from enum import Enum
from typing import Any

from pydantic import BaseModel, Field


class StoreStatus(str, Enum):
    """Store status enumeration."""

    CREATING = "CREATING"
    PENDING_REVIEW = "PENDING_REVIEW"
    ACTIVE = "ACTIVE"
    PAUSED = "PAUSED"
    ARCHIVED = "ARCHIVED"


class StorePreferences(BaseModel):
    """User preferences for store generation."""

    style: str | None = Field(None, description="Visual style preference")
    tone: str | None = Field(None, description="Brand voice tone")
    target_audience: str | None = Field(None, description="Target audience")
    price_range: str | None = Field(None, description="Product price range")


class CreateStoreRequest(BaseModel):
    """Request model for creating a new store."""

    user_id: str = Field(..., description="User ID")
    niche: str = Field(..., min_length=3, max_length=200, description="Store niche")
    preferences: StorePreferences | None = Field(None, description="Generation preferences")


class BrandIdentity(BaseModel):
    """Generated brand identity."""

    name: str
    tagline: str | None = None
    logo_prompt: str | None = None
    colors: dict[str, str] = Field(
        default_factory=lambda: {
            "primary": "#000000",
            "secondary": "#ffffff",
            "accent": "#0066ff",
        }
    )
    fonts: dict[str, str] = Field(
        default_factory=lambda: {
            "heading": "Inter",
            "body": "Inter",
        }
    )
    tone: str = "professional"
    story: str | None = None


class StoreResponse(BaseModel):
    """Store response model."""

    id: str
    user_id: str
    name: str
    slug: str
    niche: str
    status: StoreStatus
    brand_identity: BrandIdentity | None = None
    subdomain: str
    custom_domain: str | None = None
    created_at: datetime
    updated_at: datetime
```

**File: `services/genesis-engine/app/models/product.py`**
```python
"""Product-related Pydantic models."""

from decimal import Decimal
from enum import Enum

from pydantic import BaseModel, Field


class SupplierProvider(str, Enum):
    """Supported supplier providers."""

    ALIEXPRESS = "ALIEXPRESS"
    CJ_DROPSHIPPING = "CJ_DROPSHIPPING"
    SPOCKET = "SPOCKET"


class ProductStatus(str, Enum):
    """Product status enumeration."""

    DRAFT = "DRAFT"
    ACTIVE = "ACTIVE"
    OUT_OF_STOCK = "OUT_OF_STOCK"
    ARCHIVED = "ARCHIVED"


class ProductImage(BaseModel):
    """Product image data."""

    url: str
    alt_text: str | None = None
    position: int = 0


class ProductVariant(BaseModel):
    """Product variant data."""

    sku: str | None = None
    title: str
    price: Decimal
    cost_price: Decimal
    stock: int = 0
    options: dict[str, str] = Field(default_factory=dict)


class ScoutedProduct(BaseModel):
    """Product data from scouting."""

    source_provider: SupplierProvider
    source_id: str
    source_url: str | None = None
    original_title: str
    description: str
    cost_price: Decimal
    suggested_price: Decimal
    images: list[ProductImage] = Field(default_factory=list)
    variants: list[ProductVariant] = Field(default_factory=list)
    review_count: int = 0
    rating: float = 0.0
    order_count: int = 0
    shipping_time_days: int | None = None
    winning_score: float = 0.0  # Calculated score


class EnhancedProduct(BaseModel):
    """Product with AI-generated content."""

    source_product: ScoutedProduct
    title: str  # AI-rewritten title
    description: str  # AI-generated description
    meta_title: str
    meta_description: str
    slug: str
```

**File: `services/genesis-engine/app/models/job.py`**
```python
"""Job-related Pydantic models."""

from datetime import datetime
from enum import Enum
from typing import Any

from pydantic import BaseModel, Field


class JobStatus(str, Enum):
    """Job status enumeration."""

    PENDING = "PENDING"
    PROCESSING = "PROCESSING"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"
    CANCELLED = "CANCELLED"


class AgentType(str, Enum):
    """Agent type enumeration."""

    BRAND_GENERATOR = "BRAND_GENERATOR"
    PRODUCT_SCOUT = "PRODUCT_SCOUT"
    COPYWRITER = "COPYWRITER"
    IMAGE_PROCESSOR = "IMAGE_PROCESSOR"


class JobResponse(BaseModel):
    """Job status response."""

    id: str
    status: JobStatus
    progress: int = Field(ge=0, le=100)
    current_step: str | None = None
    error: str | None = None
    started_at: datetime | None = None
    completed_at: datetime | None = None
    output: dict[str, Any] | None = None


class JobProgress(BaseModel):
    """Job progress update."""

    job_id: str
    progress: int = Field(ge=0, le=100)
    current_step: str
    agent_type: AgentType | None = None
```

---

## Task 3: LLM Service Integration

### 3.1 LLM Service Abstraction

**File: `services/genesis-engine/app/services/llm.py`**
```python
"""LLM service abstraction for multiple providers."""

from abc import ABC, abstractmethod
from enum import Enum
from typing import Any

from openai import AsyncOpenAI
from anthropic import AsyncAnthropic
from pydantic import BaseModel
from tenacity import retry, stop_after_attempt, wait_exponential

from app.core.config import settings
from app.core.logging import get_logger

logger = get_logger(__name__)


class LLMProvider(str, Enum):
    """Supported LLM providers."""

    OPENAI = "openai"
    ANTHROPIC = "anthropic"


class LLMMessage(BaseModel):
    """LLM message format."""

    role: str  # "system", "user", "assistant"
    content: str


class LLMResponse(BaseModel):
    """LLM response format."""

    content: str
    model: str
    usage: dict[str, int] | None = None


class BaseLLMClient(ABC):
    """Abstract base class for LLM clients."""

    @abstractmethod
    async def chat(
        self,
        messages: list[LLMMessage],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
        response_format: dict | None = None,
    ) -> LLMResponse:
        """Send a chat completion request."""
        pass


class OpenAIClient(BaseLLMClient):
    """OpenAI API client."""

    def __init__(self):
        self.client = AsyncOpenAI(api_key=settings.openai_api_key)
        self.default_model = "gpt-4-turbo-preview"

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=2, max=10),
    )
    async def chat(
        self,
        messages: list[LLMMessage],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
        response_format: dict | None = None,
    ) -> LLMResponse:
        """Send chat completion request to OpenAI."""
        model = model or self.default_model

        kwargs: dict[str, Any] = {
            "model": model,
            "messages": [{"role": m.role, "content": m.content} for m in messages],
            "temperature": temperature,
            "max_tokens": max_tokens,
        }

        if response_format:
            kwargs["response_format"] = response_format

        response = await self.client.chat.completions.create(**kwargs)

        return LLMResponse(
            content=response.choices[0].message.content or "",
            model=response.model,
            usage={
                "prompt_tokens": response.usage.prompt_tokens if response.usage else 0,
                "completion_tokens": response.usage.completion_tokens if response.usage else 0,
                "total_tokens": response.usage.total_tokens if response.usage else 0,
            },
        )


class AnthropicClient(BaseLLMClient):
    """Anthropic API client."""

    def __init__(self):
        self.client = AsyncAnthropic(api_key=settings.anthropic_api_key)
        self.default_model = "claude-3-sonnet-20240229"

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=2, max=10),
    )
    async def chat(
        self,
        messages: list[LLMMessage],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
        response_format: dict | None = None,
    ) -> LLMResponse:
        """Send chat completion request to Anthropic."""
        model = model or self.default_model

        # Extract system message if present
        system_message = None
        chat_messages = []
        for msg in messages:
            if msg.role == "system":
                system_message = msg.content
            else:
                chat_messages.append({"role": msg.role, "content": msg.content})

        kwargs: dict[str, Any] = {
            "model": model,
            "messages": chat_messages,
            "max_tokens": max_tokens,
            "temperature": temperature,
        }

        if system_message:
            kwargs["system"] = system_message

        response = await self.client.messages.create(**kwargs)

        return LLMResponse(
            content=response.content[0].text if response.content else "",
            model=response.model,
            usage={
                "prompt_tokens": response.usage.input_tokens,
                "completion_tokens": response.usage.output_tokens,
                "total_tokens": response.usage.input_tokens + response.usage.output_tokens,
            },
        )


class LLMService:
    """
    Unified LLM service that abstracts provider differences.

    Usage:
        llm = LLMService()
        response = await llm.chat([
            LLMMessage(role="system", content="You are a helpful assistant."),
            LLMMessage(role="user", content="Generate a brand name for..."),
        ])
    """

    def __init__(
        self,
        provider: LLMProvider | None = None,
    ):
        """Initialize LLM service with specified or default provider."""
        self.provider = provider or LLMProvider(settings.default_llm_provider)

        if self.provider == LLMProvider.OPENAI:
            if not settings.openai_api_key:
                raise ValueError("OpenAI API key not configured")
            self.client = OpenAIClient()
        elif self.provider == LLMProvider.ANTHROPIC:
            if not settings.anthropic_api_key:
                raise ValueError("Anthropic API key not configured")
            self.client = AnthropicClient()
        else:
            raise ValueError(f"Unsupported provider: {self.provider}")

        logger.info(f"Initialized LLM service with provider: {self.provider}")

    async def chat(
        self,
        messages: list[LLMMessage],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
        response_format: dict | None = None,
    ) -> LLMResponse:
        """Send a chat completion request."""
        logger.debug(
            "LLM chat request",
            provider=self.provider,
            model=model,
            message_count=len(messages),
        )

        response = await self.client.chat(
            messages=messages,
            model=model,
            temperature=temperature,
            max_tokens=max_tokens,
            response_format=response_format,
        )

        logger.debug(
            "LLM chat response",
            provider=self.provider,
            model=response.model,
            usage=response.usage,
        )

        return response

    async def generate_json(
        self,
        messages: list[LLMMessage],
        model: str | None = None,
        temperature: float = 0.7,
    ) -> str:
        """Generate JSON response from LLM."""
        # For OpenAI, use JSON mode
        response_format = None
        if self.provider == LLMProvider.OPENAI:
            response_format = {"type": "json_object"}

        response = await self.chat(
            messages=messages,
            model=model,
            temperature=temperature,
            response_format=response_format,
        )

        return response.content
```

---

## Task 4: Brand Agent Implementation

### 4.1 Brand Agent

**File: `services/genesis-engine/app/agents/brand/schemas.py`**
```python
"""Brand agent input/output schemas."""

from pydantic import BaseModel, Field


class BrandInput(BaseModel):
    """Input for brand generation."""

    niche: str = Field(..., description="Store niche/category")
    style: str | None = Field(None, description="Visual style preference")
    tone: str | None = Field(None, description="Brand voice tone")
    target_audience: str | None = Field(None, description="Target audience description")


class BrandColors(BaseModel):
    """Brand color palette."""

    primary: str = Field(..., description="Primary brand color (hex)")
    secondary: str = Field(..., description="Secondary color (hex)")
    accent: str = Field(..., description="Accent color (hex)")
    background: str = Field(default="#ffffff", description="Background color")
    text: str = Field(default="#1a1a1a", description="Text color")
    muted: str = Field(default="#6b7280", description="Muted text color")


class BrandFonts(BaseModel):
    """Brand typography."""

    heading: str = Field(..., description="Heading font family")
    body: str = Field(..., description="Body font family")


class BrandVoice(BaseModel):
    """Brand voice characteristics."""

    tone: str = Field(..., description="Brand tone (e.g., professional, playful)")
    personality: list[str] = Field(default_factory=list, description="Brand personality traits")
    writing_style: str = Field(..., description="Writing style description")


class BrandOutput(BaseModel):
    """Output from brand generation."""

    name: str = Field(..., description="Brand name")
    tagline: str = Field(..., description="Brand tagline")
    logo_prompt: str = Field(..., description="Prompt for logo generation")
    colors: BrandColors
    fonts: BrandFonts
    voice: BrandVoice
    story: str = Field(..., description="Brand story (2-3 paragraphs)")
    domain_suggestions: list[str] = Field(
        default_factory=list,
        description="Suggested domain names",
    )
```

**File: `services/genesis-engine/app/agents/brand/prompts.py`**
```python
"""Brand generation prompts."""

BRAND_GENERATION_SYSTEM_PROMPT = """You are an expert brand strategist and creative director with deep expertise in e-commerce branding. Your task is to create a complete, cohesive brand identity for a new online store.

You must respond with valid JSON in the exact format specified. Be creative but practical - the brand should appeal to real customers and work well for an e-commerce context.

Guidelines:
- Brand names should be unique, memorable, and easy to spell
- Colors should be modern and appropriate for the niche
- The tone should match the target audience
- All text should be professional and engaging"""


BRAND_GENERATION_USER_PROMPT = """Create a complete brand identity for an e-commerce store in this niche:

**Niche:** {niche}
{style_section}
{tone_section}
{audience_section}

Generate a JSON response with this exact structure:
{{
    "name": "Brand Name (unique, memorable, 1-2 words)",
    "tagline": "Catchy tagline (5-10 words)",
    "logo_prompt": "Detailed prompt for generating a logo (describe style, elements, colors)",
    "colors": {{
        "primary": "#hexcode (main brand color)",
        "secondary": "#hexcode (complementary color)",
        "accent": "#hexcode (call-to-action color)",
        "background": "#hexcode (background color, usually light)",
        "text": "#hexcode (main text color)",
        "muted": "#hexcode (secondary text color)"
    }},
    "fonts": {{
        "heading": "Font name for headings (choose from: Inter, Poppins, Montserrat, Playfair Display, Roboto, Open Sans, Lato, Raleway, Oswald, Merriweather)",
        "body": "Font name for body text (choose from same list)"
    }},
    "voice": {{
        "tone": "Brand tone (e.g., professional, playful, luxurious, friendly, edgy)",
        "personality": ["trait1", "trait2", "trait3"],
        "writing_style": "Description of how the brand communicates"
    }},
    "story": "Brand story (2-3 paragraphs about the brand's mission, values, and what makes it special)",
    "domain_suggestions": ["domain1.com", "domain2.com", "domain3.com"]
}}"""


def build_brand_prompt(
    niche: str,
    style: str | None = None,
    tone: str | None = None,
    target_audience: str | None = None,
) -> str:
    """Build the brand generation prompt."""
    style_section = f"**Style Preference:** {style}" if style else ""
    tone_section = f"**Tone:** {tone}" if tone else ""
    audience_section = f"**Target Audience:** {target_audience}" if target_audience else ""

    return BRAND_GENERATION_USER_PROMPT.format(
        niche=niche,
        style_section=style_section,
        tone_section=tone_section,
        audience_section=audience_section,
    )
```

**File: `services/genesis-engine/app/agents/brand/agent.py`**
```python
"""Brand generation agent."""

import json

from app.agents.base import AgentContext, BaseAgent
from app.agents.brand.prompts import BRAND_GENERATION_SYSTEM_PROMPT, build_brand_prompt
from app.agents.brand.schemas import BrandInput, BrandOutput
from app.services.llm import LLMMessage, LLMService


class BrandAgent(BaseAgent[BrandInput, BrandOutput]):
    """
    Agent responsible for generating brand identity.

    Takes a niche and preferences, generates:
    - Brand name and tagline
    - Color palette
    - Typography
    - Brand voice
    - Brand story
    """

    @property
    def name(self) -> str:
        return "Brand Generator"

    @property
    def description(self) -> str:
        return "Generates complete brand identity from niche"

    async def _execute(
        self,
        input_data: BrandInput,
        context: AgentContext,
    ) -> BrandOutput:
        """Execute brand generation."""
        self._update_progress(context, 10, "Preparing brand generation")

        # Build the prompt
        user_prompt = build_brand_prompt(
            niche=input_data.niche,
            style=input_data.style,
            tone=input_data.tone,
            target_audience=input_data.target_audience,
        )

        self._update_progress(context, 30, "Generating brand identity via LLM")

        # Call LLM
        messages = [
            LLMMessage(role="system", content=BRAND_GENERATION_SYSTEM_PROMPT),
            LLMMessage(role="user", content=user_prompt),
        ]

        response = await self.llm.generate_json(
            messages=messages,
            temperature=0.8,  # Higher creativity for branding
        )

        self._update_progress(context, 70, "Parsing brand data")

        # Parse response
        try:
            data = json.loads(response)
            brand_output = BrandOutput(**data)
        except (json.JSONDecodeError, ValueError) as e:
            self.logger.error("Failed to parse brand response", error=str(e))
            raise ValueError(f"Invalid brand generation response: {e}")

        self._update_progress(context, 90, "Validating brand identity")

        # Validate colors are valid hex codes
        self._validate_colors(brand_output)

        self._update_progress(context, 100, "Brand generation complete")

        return brand_output

    def _validate_colors(self, brand: BrandOutput) -> None:
        """Validate color hex codes."""
        import re
        hex_pattern = re.compile(r"^#[0-9A-Fa-f]{6}$")

        colors = brand.colors.model_dump()
        for name, color in colors.items():
            if not hex_pattern.match(color):
                self.logger.warning(f"Invalid color for {name}: {color}, using default")
                # Could set to default here if needed
```

---

## Task 5: Product Scout Agent Implementation

### 5.1 Supplier Base Class

**File: `services/genesis-engine/app/agents/scout/suppliers/base.py`**
```python
"""Base class for supplier integrations."""

from abc import ABC, abstractmethod
from typing import Any

from pydantic import BaseModel

from app.core.logging import get_logger
from app.models.product import ScoutedProduct


class SupplierConfig(BaseModel):
    """Supplier API configuration."""

    api_key: str
    api_secret: str | None = None
    account_id: str | None = None


class BaseSupplier(ABC):
    """Abstract base class for supplier integrations."""

    def __init__(self, config: SupplierConfig):
        self.config = config
        self.logger = get_logger(self.__class__.__name__)

    @property
    @abstractmethod
    def name(self) -> str:
        """Supplier name."""
        pass

    @abstractmethod
    async def search_products(
        self,
        keywords: list[str],
        max_results: int = 100,
    ) -> list[ScoutedProduct]:
        """
        Search for products by keywords.

        Args:
            keywords: Search keywords
            max_results: Maximum number of results

        Returns:
            List of scouted products
        """
        pass

    @abstractmethod
    async def get_product_details(
        self,
        product_id: str,
    ) -> ScoutedProduct | None:
        """
        Get detailed product information.

        Args:
            product_id: Product ID

        Returns:
            Product details or None if not found
        """
        pass

    async def validate_connection(self) -> bool:
        """Validate API connection."""
        try:
            # Simple test query
            await self.search_products(["test"], max_results=1)
            return True
        except Exception as e:
            self.logger.error(f"Connection validation failed: {e}")
            return False
```

### 5.2 AliExpress Supplier (Mock Implementation)

**File: `services/genesis-engine/app/agents/scout/suppliers/aliexpress.py`**
```python
"""AliExpress supplier integration."""

from decimal import Decimal
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential

from app.agents.scout.suppliers.base import BaseSupplier, SupplierConfig
from app.models.product import ProductImage, ProductVariant, ScoutedProduct, SupplierProvider


class AliExpressSupplier(BaseSupplier):
    """
    AliExpress Affiliate API integration.

    Note: This is a simplified implementation. The actual AliExpress API
    requires OAuth and has specific endpoint structures.
    """

    BASE_URL = "https://api.aliexpress.com"  # Placeholder

    def __init__(self, config: SupplierConfig):
        super().__init__(config)
        self.client = httpx.AsyncClient(timeout=30.0)

    @property
    def name(self) -> str:
        return "AliExpress"

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=2, max=10),
    )
    async def search_products(
        self,
        keywords: list[str],
        max_results: int = 100,
    ) -> list[ScoutedProduct]:
        """
        Search AliExpress products.

        In production, this would call the actual AliExpress API.
        For now, returns mock data for development.
        """
        self.logger.info(
            "Searching AliExpress products",
            keywords=keywords,
            max_results=max_results,
        )

        # TODO: Implement actual AliExpress API integration
        # For development, return mock products
        return self._generate_mock_products(keywords, max_results)

    async def get_product_details(
        self,
        product_id: str,
    ) -> ScoutedProduct | None:
        """Get product details from AliExpress."""
        self.logger.info("Fetching product details", product_id=product_id)

        # TODO: Implement actual API call
        return None

    def _generate_mock_products(
        self,
        keywords: list[str],
        count: int,
    ) -> list[ScoutedProduct]:
        """Generate mock products for development."""
        products = []
        keyword = keywords[0] if keywords else "product"

        for i in range(min(count, 20)):  # Limit mock data
            products.append(
                ScoutedProduct(
                    source_provider=SupplierProvider.ALIEXPRESS,
                    source_id=f"ali_{i}_{hash(keyword) % 10000}",
                    source_url=f"https://aliexpress.com/item/{i}.html",
                    original_title=f"{keyword.title()} Product {i + 1} - High Quality",
                    description=f"High quality {keyword} product with excellent features. "
                    f"Perfect for home decoration and daily use. Made with premium materials.",
                    cost_price=Decimal(str(round(5 + i * 2.5, 2))),
                    suggested_price=Decimal(str(round((5 + i * 2.5) * 2.5, 2))),
                    images=[
                        ProductImage(
                            url=f"https://placeholder.com/product_{i}_1.jpg",
                            alt_text=f"{keyword} product image 1",
                            position=0,
                        ),
                        ProductImage(
                            url=f"https://placeholder.com/product_{i}_2.jpg",
                            alt_text=f"{keyword} product image 2",
                            position=1,
                        ),
                    ],
                    variants=[
                        ProductVariant(
                            sku=f"SKU-{i}-A",
                            title="Default",
                            price=Decimal(str(round((5 + i * 2.5) * 2.5, 2))),
                            cost_price=Decimal(str(round(5 + i * 2.5, 2))),
                            stock=100,
                            options={"size": "Standard"},
                        ),
                    ],
                    review_count=50 + i * 10,
                    rating=4.0 + (i % 10) * 0.1,
                    order_count=100 + i * 50,
                    shipping_time_days=14 + (i % 7),
                    winning_score=0.0,  # Will be calculated later
                )
            )

        return products

    async def close(self) -> None:
        """Close the HTTP client."""
        await self.client.aclose()
```

### 5.3 Product Ranking Logic

**File: `services/genesis-engine/app/agents/scout/ranking.py`**
```python
"""Product ranking and scoring logic."""

from dataclasses import dataclass

from app.models.product import ScoutedProduct


@dataclass
class ScoringWeights:
    """Weights for product scoring factors."""

    rating: float = 0.20
    reviews: float = 0.15
    orders: float = 0.20
    margin: float = 0.25
    shipping: float = 0.10
    image_quality: float = 0.10


class ProductRanker:
    """
    Ranks products based on multiple factors to identify "winners".

    Scoring factors:
    - Rating (4.5+ stars preferred)
    - Review count (social proof)
    - Order count (proven demand)
    - Profit margin (cost vs suggested price)
    - Shipping time (faster is better)
    - Image quality (number and resolution)
    """

    def __init__(self, weights: ScoringWeights | None = None):
        self.weights = weights or ScoringWeights()

    def calculate_score(self, product: ScoutedProduct) -> float:
        """
        Calculate winning score for a product.

        Returns:
            Score between 0 and 100
        """
        scores = {
            "rating": self._score_rating(product.rating),
            "reviews": self._score_reviews(product.review_count),
            "orders": self._score_orders(product.order_count),
            "margin": self._score_margin(product.cost_price, product.suggested_price),
            "shipping": self._score_shipping(product.shipping_time_days),
            "image_quality": self._score_images(product.images),
        }

        # Weighted sum
        total_score = sum(
            scores[key] * getattr(self.weights, key)
            for key in scores
        )

        return round(total_score, 2)

    def rank_products(
        self,
        products: list[ScoutedProduct],
        top_n: int = 20,
    ) -> list[ScoutedProduct]:
        """
        Rank products and return top N.

        Args:
            products: List of products to rank
            top_n: Number of top products to return

        Returns:
            Top N products sorted by winning score
        """
        # Calculate scores
        for product in products:
            product.winning_score = self.calculate_score(product)

        # Sort by score descending
        ranked = sorted(products, key=lambda p: p.winning_score, reverse=True)

        return ranked[:top_n]

    def _score_rating(self, rating: float) -> float:
        """Score rating (0-5 scale to 0-100)."""
        if rating >= 4.8:
            return 100
        elif rating >= 4.5:
            return 90
        elif rating >= 4.0:
            return 70
        elif rating >= 3.5:
            return 50
        else:
            return 30

    def _score_reviews(self, count: int) -> float:
        """Score review count."""
        if count >= 1000:
            return 100
        elif count >= 500:
            return 90
        elif count >= 100:
            return 70
        elif count >= 50:
            return 50
        elif count >= 10:
            return 30
        else:
            return 10

    def _score_orders(self, count: int) -> float:
        """Score order count."""
        if count >= 5000:
            return 100
        elif count >= 1000:
            return 90
        elif count >= 500:
            return 70
        elif count >= 100:
            return 50
        else:
            return 30

    def _score_margin(self, cost: float, suggested: float) -> float:
        """Score profit margin."""
        if cost <= 0:
            return 0

        margin = (float(suggested) - float(cost)) / float(cost) * 100

        if margin >= 150:  # 150%+ markup
            return 100
        elif margin >= 100:
            return 80
        elif margin >= 50:
            return 60
        elif margin >= 30:
            return 40
        else:
            return 20

    def _score_shipping(self, days: int | None) -> float:
        """Score shipping time (faster is better)."""
        if days is None:
            return 50  # Unknown

        if days <= 7:
            return 100
        elif days <= 14:
            return 80
        elif days <= 21:
            return 60
        elif days <= 30:
            return 40
        else:
            return 20

    def _score_images(self, images: list) -> float:
        """Score image quality based on count."""
        count = len(images)

        if count >= 5:
            return 100
        elif count >= 3:
            return 80
        elif count >= 2:
            return 60
        elif count >= 1:
            return 40
        else:
            return 0
```

### 5.4 Scout Agent

**File: `services/genesis-engine/app/agents/scout/schemas.py`**
```python
"""Scout agent input/output schemas."""

from pydantic import BaseModel, Field

from app.models.product import ScoutedProduct, SupplierProvider


class SupplierAccountInfo(BaseModel):
    """Supplier account information."""

    provider: SupplierProvider
    api_key: str
    api_secret: str | None = None
    account_id: str | None = None


class ScoutInput(BaseModel):
    """Input for product scouting."""

    niche: str = Field(..., description="Product niche to scout")
    supplier_accounts: list[SupplierAccountInfo] = Field(
        default_factory=list,
        description="Connected supplier accounts",
    )
    max_products: int = Field(default=20, ge=5, le=50, description="Max products to return")
    min_rating: float = Field(default=4.0, ge=0, le=5, description="Minimum product rating")
    min_orders: int = Field(default=10, ge=0, description="Minimum order count")


class ScoutOutput(BaseModel):
    """Output from product scouting."""

    products: list[ScoutedProduct] = Field(default_factory=list)
    keywords_used: list[str] = Field(default_factory=list)
    total_found: int = Field(default=0)
    suppliers_searched: list[str] = Field(default_factory=list)
```

**File: `services/genesis-engine/app/agents/scout/agent.py`**
```python
"""Product scout agent."""

from app.agents.base import AgentContext, BaseAgent
from app.agents.scout.ranking import ProductRanker
from app.agents.scout.schemas import ScoutInput, ScoutOutput
from app.agents.scout.suppliers.aliexpress import AliExpressSupplier
from app.agents.scout.suppliers.base import BaseSupplier, SupplierConfig
from app.models.product import ScoutedProduct, SupplierProvider
from app.services.llm import LLMMessage


class ScoutAgent(BaseAgent[ScoutInput, ScoutOutput]):
    """
    Agent responsible for finding winning products.

    Process:
    1. Generate search keywords from niche
    2. Query supplier APIs
    3. Analyze and rank products
    4. Return top products
    """

    @property
    def name(self) -> str:
        return "Product Scout"

    @property
    def description(self) -> str:
        return "Finds and ranks winning products from suppliers"

    async def _execute(
        self,
        input_data: ScoutInput,
        context: AgentContext,
    ) -> ScoutOutput:
        """Execute product scouting."""
        self._update_progress(context, 5, "Generating search keywords")

        # Generate keywords from niche
        keywords = await self._generate_keywords(input_data.niche)
        self.logger.info("Generated keywords", keywords=keywords)

        self._update_progress(context, 20, "Connecting to suppliers")

        # Initialize suppliers
        suppliers = self._init_suppliers(input_data)

        if not suppliers:
            self.logger.warning("No suppliers configured, using mock data")
            # Use AliExpress with mock config for development
            suppliers = [
                AliExpressSupplier(
                    SupplierConfig(api_key="mock", api_secret="mock")
                )
            ]

        self._update_progress(context, 30, "Searching products")

        # Search products from all suppliers
        all_products: list[ScoutedProduct] = []
        suppliers_searched: list[str] = []

        for supplier in suppliers:
            try:
                products = await supplier.search_products(
                    keywords=keywords,
                    max_results=100,  # Get more, then filter
                )
                all_products.extend(products)
                suppliers_searched.append(supplier.name)
                self.logger.info(
                    f"Found products from {supplier.name}",
                    count=len(products),
                )
            except Exception as e:
                self.logger.error(f"Supplier search failed: {supplier.name}", error=str(e))

        self._update_progress(context, 60, f"Found {len(all_products)} products, ranking")

        # Filter by minimum requirements
        filtered = [
            p for p in all_products
            if p.rating >= input_data.min_rating and p.order_count >= input_data.min_orders
        ]

        self._update_progress(context, 70, "Analyzing and scoring products")

        # Rank products
        ranker = ProductRanker()
        top_products = ranker.rank_products(filtered, top_n=input_data.max_products)

        self._update_progress(context, 90, "Deduplicating results")

        # Deduplicate (by title similarity - simplified)
        deduplicated = self._deduplicate_products(top_products)

        self._update_progress(context, 100, "Scouting complete")

        return ScoutOutput(
            products=deduplicated,
            keywords_used=keywords,
            total_found=len(all_products),
            suppliers_searched=suppliers_searched,
        )

    async def _generate_keywords(self, niche: str) -> list[str]:
        """Generate search keywords from niche using LLM."""
        messages = [
            LLMMessage(
                role="system",
                content="You are a product research expert. Generate search keywords for finding products.",
            ),
            LLMMessage(
                role="user",
                content=f"""Generate 15-20 search keywords for finding products in this niche: "{niche}"

Include:
- Main niche terms
- Related product categories
- Specific product types
- Trending variations

Return only the keywords, one per line, no numbering or explanations.""",
            ),
        ]

        response = await self.llm.chat(messages=messages, temperature=0.7)
        keywords = [
            line.strip()
            for line in response.content.split("\n")
            if line.strip() and len(line.strip()) > 2
        ]

        return keywords[:20]  # Limit to 20 keywords

    def _init_suppliers(self, input_data: ScoutInput) -> list[BaseSupplier]:
        """Initialize supplier clients from input."""
        suppliers: list[BaseSupplier] = []

        for account in input_data.supplier_accounts:
            config = SupplierConfig(
                api_key=account.api_key,
                api_secret=account.api_secret,
                account_id=account.account_id,
            )

            if account.provider == SupplierProvider.ALIEXPRESS:
                suppliers.append(AliExpressSupplier(config))
            # TODO: Add other suppliers

        return suppliers

    def _deduplicate_products(
        self,
        products: list[ScoutedProduct],
    ) -> list[ScoutedProduct]:
        """Remove duplicate products based on title similarity."""
        seen_titles: set[str] = set()
        unique_products: list[ScoutedProduct] = []

        for product in products:
            # Simple dedup by normalized title
            normalized = product.original_title.lower().strip()[:50]
            if normalized not in seen_titles:
                seen_titles.add(normalized)
                unique_products.append(product)

        return unique_products
```

---

## Task 6: Copywriter Agent Implementation

### 6.1 Copywriter Schemas

**File: `services/genesis-engine/app/agents/copywriter/schemas.py`**
```python
"""Copywriter agent input/output schemas."""

from pydantic import BaseModel, Field

from app.models.product import EnhancedProduct, ScoutedProduct
from app.models.store import BrandIdentity


class CopywriterInput(BaseModel):
    """Input for copywriting."""

    store_name: str
    niche: str
    brand_identity: BrandIdentity
    products: list[ScoutedProduct]


class GeneratedPage(BaseModel):
    """Generated page content."""

    type: str = Field(..., description="Page type (privacy, terms, about, etc.)")
    title: str
    slug: str
    content: str = Field(..., description="HTML or Markdown content")
    meta_title: str
    meta_description: str


class CopywriterOutput(BaseModel):
    """Output from copywriting."""

    products: list[EnhancedProduct] = Field(default_factory=list)
    pages: list[GeneratedPage] = Field(default_factory=list)
```

### 6.2 Copywriter Prompts

**File: `services/genesis-engine/app/agents/copywriter/prompts.py`**
```python
"""Copywriting prompts."""

PRODUCT_COPY_SYSTEM = """You are an expert e-commerce copywriter. Write compelling, SEO-optimized product descriptions that convert browsers into buyers.

Guidelines:
- Use the brand voice consistently
- Highlight benefits over features
- Include relevant keywords naturally
- Create urgency without being pushy
- Keep descriptions scannable with short paragraphs"""


PRODUCT_COPY_USER = """Write product copy for this item:

**Brand:** {brand_name}
**Brand Tone:** {brand_tone}
**Niche:** {niche}

**Original Product Title:** {original_title}
**Original Description:** {original_description}
**Price:** ${price}

Generate a JSON response:
{{
    "title": "SEO-optimized title (60 chars max)",
    "description": "Compelling description (150-300 words, use HTML formatting)",
    "meta_title": "Meta title for SEO (60 chars max)",
    "meta_description": "Meta description (155 chars max)",
    "slug": "url-friendly-slug"
}}"""


PAGE_GENERATION_SYSTEM = """You are a professional copywriter creating legal and informational pages for an e-commerce store. Write clear, professional content that protects the business while being customer-friendly."""


ABOUT_PAGE_PROMPT = """Write an About Us page for:

**Brand:** {brand_name}
**Tagline:** {tagline}
**Niche:** {niche}
**Brand Story:** {brand_story}
**Brand Tone:** {brand_tone}

Create engaging content (300-500 words) that:
- Tells the brand story
- Establishes credibility
- Connects emotionally with customers
- Includes a call to action

Return JSON:
{{
    "title": "About Us",
    "content": "HTML content",
    "meta_title": "About {brand_name} | Our Story",
    "meta_description": "Learn about {brand_name}..."
}}"""


PRIVACY_POLICY_PROMPT = """Generate a Privacy Policy for:

**Brand:** {brand_name}
**Website:** {website}

Include standard sections:
- Information collection
- How we use information
- Cookie policy
- Third-party services
- Data security
- Contact information

Return JSON:
{{
    "title": "Privacy Policy",
    "content": "HTML content with proper headings",
    "meta_title": "Privacy Policy | {brand_name}",
    "meta_description": "Read our privacy policy..."
}}"""


TERMS_OF_SERVICE_PROMPT = """Generate Terms of Service for:

**Brand:** {brand_name}
**Website:** {website}

Include standard sections:
- Acceptance of terms
- Use of the service
- Products and pricing
- Orders and payment
- Shipping and delivery
- Returns and refunds
- Limitation of liability
- Governing law

Return JSON:
{{
    "title": "Terms of Service",
    "content": "HTML content with proper headings",
    "meta_title": "Terms of Service | {brand_name}",
    "meta_description": "Read our terms of service..."
}}"""


REFUND_POLICY_PROMPT = """Generate a Refund Policy for:

**Brand:** {brand_name}
**Niche:** {niche}

Include:
- Return window (30 days recommended)
- Condition requirements
- Refund process
- Non-refundable items
- Exchange policy
- Contact information

Return JSON:
{{
    "title": "Refund Policy",
    "content": "HTML content",
    "meta_title": "Refund Policy | {brand_name}",
    "meta_description": "Learn about our refund policy..."
}}"""
```

### 6.3 Copywriter Agent

**File: `services/genesis-engine/app/agents/copywriter/agent.py`**
```python
"""Copywriter agent for generating product and page content."""

import json
import re

from app.agents.base import AgentContext, BaseAgent
from app.agents.copywriter.prompts import (
    ABOUT_PAGE_PROMPT,
    PAGE_GENERATION_SYSTEM,
    PRIVACY_POLICY_PROMPT,
    PRODUCT_COPY_SYSTEM,
    PRODUCT_COPY_USER,
    REFUND_POLICY_PROMPT,
    TERMS_OF_SERVICE_PROMPT,
)
from app.agents.copywriter.schemas import (
    CopywriterInput,
    CopywriterOutput,
    GeneratedPage,
)
from app.models.product import EnhancedProduct
from app.services.llm import LLMMessage


class CopywriterAgent(BaseAgent[CopywriterInput, CopywriterOutput]):
    """
    Agent responsible for generating all store content.

    Generates:
    - Product titles and descriptions
    - SEO meta tags
    - Legal pages (Privacy, Terms, Refund)
    - About page
    """

    @property
    def name(self) -> str:
        return "Copywriter"

    @property
    def description(self) -> str:
        return "Generates product copy and store pages"

    async def _execute(
        self,
        input_data: CopywriterInput,
        context: AgentContext,
    ) -> CopywriterOutput:
        """Execute copywriting."""
        total_products = len(input_data.products)
        enhanced_products: list[EnhancedProduct] = []
        pages: list[GeneratedPage] = []

        # Generate product copy
        self._update_progress(context, 5, "Starting product copywriting")

        for i, product in enumerate(input_data.products):
            progress = 5 + int((i / total_products) * 50)
            self._update_progress(
                context,
                progress,
                f"Writing copy for product {i + 1}/{total_products}",
            )

            enhanced = await self._generate_product_copy(
                product=product,
                brand_name=input_data.store_name,
                brand_tone=input_data.brand_identity.voice.tone,
                niche=input_data.niche,
            )
            enhanced_products.append(enhanced)

        # Generate pages
        self._update_progress(context, 60, "Generating About page")
        about_page = await self._generate_about_page(input_data)
        pages.append(about_page)

        self._update_progress(context, 70, "Generating Privacy Policy")
        privacy_page = await self._generate_privacy_policy(input_data)
        pages.append(privacy_page)

        self._update_progress(context, 80, "Generating Terms of Service")
        terms_page = await self._generate_terms_of_service(input_data)
        pages.append(terms_page)

        self._update_progress(context, 90, "Generating Refund Policy")
        refund_page = await self._generate_refund_policy(input_data)
        pages.append(refund_page)

        self._update_progress(context, 100, "Copywriting complete")

        return CopywriterOutput(
            products=enhanced_products,
            pages=pages,
        )

    async def _generate_product_copy(
        self,
        product,
        brand_name: str,
        brand_tone: str,
        niche: str,
    ) -> EnhancedProduct:
        """Generate enhanced copy for a single product."""
        prompt = PRODUCT_COPY_USER.format(
            brand_name=brand_name,
            brand_tone=brand_tone,
            niche=niche,
            original_title=product.original_title,
            original_description=product.description[:500],
            price=product.suggested_price,
        )

        messages = [
            LLMMessage(role="system", content=PRODUCT_COPY_SYSTEM),
            LLMMessage(role="user", content=prompt),
        ]

        response = await self.llm.generate_json(messages=messages, temperature=0.7)

        try:
            data = json.loads(response)
            return EnhancedProduct(
                source_product=product,
                title=data.get("title", product.original_title),
                description=data.get("description", product.description),
                meta_title=data.get("meta_title", product.original_title[:60]),
                meta_description=data.get("meta_description", product.description[:155]),
                slug=data.get("slug", self._generate_slug(product.original_title)),
            )
        except (json.JSONDecodeError, KeyError) as e:
            self.logger.warning(f"Failed to parse product copy: {e}")
            return EnhancedProduct(
                source_product=product,
                title=product.original_title,
                description=product.description,
                meta_title=product.original_title[:60],
                meta_description=product.description[:155],
                slug=self._generate_slug(product.original_title),
            )

    async def _generate_about_page(self, input_data: CopywriterInput) -> GeneratedPage:
        """Generate About Us page."""
        prompt = ABOUT_PAGE_PROMPT.format(
            brand_name=input_data.store_name,
            tagline=input_data.brand_identity.tagline or "",
            niche=input_data.niche,
            brand_story=input_data.brand_identity.story or "",
            brand_tone=input_data.brand_identity.voice.tone,
        )

        return await self._generate_page("about", prompt, input_data.store_name)

    async def _generate_privacy_policy(self, input_data: CopywriterInput) -> GeneratedPage:
        """Generate Privacy Policy page."""
        prompt = PRIVACY_POLICY_PROMPT.format(
            brand_name=input_data.store_name,
            website=f"{input_data.store_name.lower().replace(' ', '')}.com",
        )

        return await self._generate_page("privacy-policy", prompt, input_data.store_name)

    async def _generate_terms_of_service(self, input_data: CopywriterInput) -> GeneratedPage:
        """Generate Terms of Service page."""
        prompt = TERMS_OF_SERVICE_PROMPT.format(
            brand_name=input_data.store_name,
            website=f"{input_data.store_name.lower().replace(' ', '')}.com",
        )

        return await self._generate_page("terms-of-service", prompt, input_data.store_name)

    async def _generate_refund_policy(self, input_data: CopywriterInput) -> GeneratedPage:
        """Generate Refund Policy page."""
        prompt = REFUND_POLICY_PROMPT.format(
            brand_name=input_data.store_name,
            niche=input_data.niche,
        )

        return await self._generate_page("refund-policy", prompt, input_data.store_name)

    async def _generate_page(
        self,
        page_type: str,
        prompt: str,
        brand_name: str,
    ) -> GeneratedPage:
        """Generate a single page."""
        messages = [
            LLMMessage(role="system", content=PAGE_GENERATION_SYSTEM),
            LLMMessage(role="user", content=prompt),
        ]

        response = await self.llm.generate_json(messages=messages, temperature=0.5)

        try:
            data = json.loads(response)
            return GeneratedPage(
                type=page_type,
                title=data.get("title", page_type.replace("-", " ").title()),
                slug=page_type,
                content=data.get("content", ""),
                meta_title=data.get("meta_title", f"{page_type.title()} | {brand_name}"),
                meta_description=data.get("meta_description", ""),
            )
        except (json.JSONDecodeError, KeyError) as e:
            self.logger.warning(f"Failed to parse {page_type} page: {e}")
            return GeneratedPage(
                type=page_type,
                title=page_type.replace("-", " ").title(),
                slug=page_type,
                content=f"<p>{page_type.title()} content coming soon.</p>",
                meta_title=f"{page_type.title()} | {brand_name}",
                meta_description="",
            )

    def _generate_slug(self, title: str) -> str:
        """Generate URL-friendly slug from title."""
        slug = title.lower()
        slug = re.sub(r"[^a-z0-9\s-]", "", slug)
        slug = re.sub(r"[\s_]+", "-", slug)
        slug = re.sub(r"-+", "-", slug)
        return slug[:50].strip("-")
```

---

## Task 7: Genesis Orchestrator

### 7.1 Orchestrator Service

**File: `services/genesis-engine/app/services/orchestrator.py`**
```python
"""Genesis orchestrator - coordinates all agents."""

from dataclasses import dataclass
from datetime import datetime
import uuid

from app.agents.brand.agent import BrandAgent
from app.agents.brand.schemas import BrandInput
from app.agents.copywriter.agent import CopywriterAgent
from app.agents.copywriter.schemas import CopywriterInput
from app.agents.scout.agent import ScoutAgent
from app.agents.scout.schemas import ScoutInput
from app.agents.base import AgentContext
from app.core.logging import get_logger
from app.models.store import BrandIdentity, StorePreferences

logger = get_logger(__name__)


@dataclass
class GenesisResult:
    """Result of genesis process."""

    store_id: str
    job_id: str
    success: bool = False
    error: str | None = None


class GenesisOrchestrator:
    """
    Orchestrates the complete store generation process.

    Flow:
    1. Create store record (CREATING status)
    2. Run Brand Agent -> Generate brand identity
    3. Run Scout Agent -> Find products
    4. Run Copywriter Agent -> Generate content
    5. Update store (PENDING_REVIEW status)
    """

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
        """
        Start the genesis process for a new store.

        This is the main entry point that coordinates all agents.
        """
        # Generate IDs
        store_id = f"store_{uuid.uuid4().hex[:12]}"
        job_id = f"job_{uuid.uuid4().hex[:12]}"

        logger.info(
            "Starting Genesis",
            store_id=store_id,
            job_id=job_id,
            niche=niche,
        )

        # Create context
        context = AgentContext(
            store_id=store_id,
            user_id=user_id,
            job_id=job_id,
            niche=niche,
            preferences=preferences.model_dump() if preferences else {},
        )

        try:
            # TODO: Create store record in database with CREATING status

            # Step 1: Generate Brand
            logger.info("Running Brand Agent", store_id=store_id)
            brand_input = BrandInput(
                niche=niche,
                style=preferences.style if preferences else None,
                tone=preferences.tone if preferences else None,
                target_audience=preferences.target_audience if preferences else None,
            )

            brand_result = await self.brand_agent.run(brand_input, context)
            if not brand_result.success or not brand_result.data:
                raise Exception(f"Brand generation failed: {brand_result.error}")

            brand_identity = BrandIdentity(
                name=brand_result.data.name,
                tagline=brand_result.data.tagline,
                logo_prompt=brand_result.data.logo_prompt,
                colors={
                    "primary": brand_result.data.colors.primary,
                    "secondary": brand_result.data.colors.secondary,
                    "accent": brand_result.data.colors.accent,
                },
                fonts={
                    "heading": brand_result.data.fonts.heading,
                    "body": brand_result.data.fonts.body,
                },
                tone=brand_result.data.voice.tone,
                story=brand_result.data.story,
            )

            # Step 2: Scout Products
            logger.info("Running Scout Agent", store_id=store_id)
            scout_input = ScoutInput(
                niche=niche,
                supplier_accounts=[],  # TODO: Get from user's connected accounts
                max_products=20,
            )

            scout_result = await self.scout_agent.run(scout_input, context)
            if not scout_result.success or not scout_result.data:
                raise Exception(f"Product scouting failed: {scout_result.error}")

            # Step 3: Generate Copy
            logger.info("Running Copywriter Agent", store_id=store_id)
            copywriter_input = CopywriterInput(
                store_name=brand_identity.name,
                niche=niche,
                brand_identity=brand_identity,
                products=scout_result.data.products,
            )

            copy_result = await self.copywriter_agent.run(copywriter_input, context)
            if not copy_result.success or not copy_result.data:
                raise Exception(f"Copywriting failed: {copy_result.error}")

            # TODO: Save all generated data to database
            # TODO: Update store status to PENDING_REVIEW

            logger.info(
                "Genesis completed successfully",
                store_id=store_id,
                products_count=len(copy_result.data.products),
                pages_count=len(copy_result.data.pages),
            )

            return GenesisResult(
                store_id=store_id,
                job_id=job_id,
                success=True,
            )

        except Exception as e:
            logger.error(
                "Genesis failed",
                store_id=store_id,
                error=str(e),
            )
            # TODO: Update job status to FAILED

            return GenesisResult(
                store_id=store_id,
                job_id=job_id,
                success=False,
                error=str(e),
            )
```

---

## Task 8: Testing

### 8.1 Test Configuration

**File: `services/genesis-engine/tests/conftest.py`**
```python
"""Pytest configuration and fixtures."""

import pytest
from unittest.mock import AsyncMock, MagicMock

from app.services.llm import LLMService, LLMResponse


@pytest.fixture
def mock_llm_service():
    """Mock LLM service for testing."""
    mock = AsyncMock(spec=LLMService)

    # Default response
    mock.chat.return_value = LLMResponse(
        content='{"name": "TestBrand"}',
        model="test-model",
        usage={"total_tokens": 100},
    )

    mock.generate_json.return_value = '{"name": "TestBrand", "tagline": "Test tagline"}'

    return mock


@pytest.fixture
def sample_context():
    """Sample agent context for testing."""
    from app.agents.base import AgentContext

    return AgentContext(
        store_id="test_store_123",
        user_id="test_user_456",
        job_id="test_job_789",
        niche="Test niche",
    )
```

### 8.2 Agent Tests

**File: `services/genesis-engine/tests/test_agents/test_brand_agent.py`**
```python
"""Tests for Brand Agent."""

import json
import pytest
from unittest.mock import AsyncMock

from app.agents.brand.agent import BrandAgent
from app.agents.brand.schemas import BrandInput


@pytest.mark.asyncio
async def test_brand_agent_success(mock_llm_service, sample_context):
    """Test successful brand generation."""
    # Setup mock response
    mock_response = {
        "name": "NeonNest",
        "tagline": "Illuminate Your Space",
        "logo_prompt": "Minimalist neon light logo",
        "colors": {
            "primary": "#FF00FF",
            "secondary": "#00FFFF",
            "accent": "#FFFF00",
            "background": "#1a1a1a",
            "text": "#ffffff",
            "muted": "#888888",
        },
        "fonts": {
            "heading": "Poppins",
            "body": "Inter",
        },
        "voice": {
            "tone": "futuristic",
            "personality": ["bold", "innovative", "edgy"],
            "writing_style": "Concise and impactful",
        },
        "story": "NeonNest was born from a passion for cyberpunk aesthetics...",
        "domain_suggestions": ["neonnest.com", "neon-nest.co"],
    }

    mock_llm_service.generate_json.return_value = json.dumps(mock_response)

    # Create agent with mock
    agent = BrandAgent(llm_service=mock_llm_service)

    # Run agent
    input_data = BrandInput(niche="Cyberpunk home decor")
    result = await agent.run(input_data, sample_context)

    # Assertions
    assert result.success is True
    assert result.data is not None
    assert result.data.name == "NeonNest"
    assert result.data.colors.primary == "#FF00FF"


@pytest.mark.asyncio
async def test_brand_agent_invalid_response(mock_llm_service, sample_context):
    """Test handling of invalid LLM response."""
    mock_llm_service.generate_json.return_value = "invalid json"

    agent = BrandAgent(llm_service=mock_llm_service)
    input_data = BrandInput(niche="Test niche")

    result = await agent.run(input_data, sample_context)

    assert result.success is False
    assert result.error is not None
```

**File: `services/genesis-engine/tests/test_agents/test_scout_agent.py`**
```python
"""Tests for Scout Agent."""

import pytest
from decimal import Decimal

from app.agents.scout.agent import ScoutAgent
from app.agents.scout.schemas import ScoutInput
from app.agents.scout.ranking import ProductRanker
from app.models.product import ScoutedProduct, SupplierProvider, ProductImage


@pytest.mark.asyncio
async def test_scout_agent_generates_keywords(mock_llm_service, sample_context):
    """Test keyword generation."""
    mock_llm_service.chat.return_value.content = """
    cyberpunk decor
    neon lights
    LED wall art
    futuristic furniture
    sci-fi decorations
    """

    agent = ScoutAgent(llm_service=mock_llm_service)
    keywords = await agent._generate_keywords("Cyberpunk home decor")

    assert len(keywords) > 0
    assert "cyberpunk decor" in keywords


def test_product_ranker_scoring():
    """Test product scoring logic."""
    ranker = ProductRanker()

    product = ScoutedProduct(
        source_provider=SupplierProvider.ALIEXPRESS,
        source_id="test_123",
        original_title="Test Product",
        description="Test description",
        cost_price=Decimal("10.00"),
        suggested_price=Decimal("30.00"),  # 200% margin
        images=[ProductImage(url="http://test.com/1.jpg")],
        review_count=500,
        rating=4.7,
        order_count=1000,
        shipping_time_days=10,
    )

    score = ranker.calculate_score(product)

    assert score > 0
    assert score <= 100


def test_product_ranker_ranking():
    """Test product ranking."""
    ranker = ProductRanker()

    products = [
        ScoutedProduct(
            source_provider=SupplierProvider.ALIEXPRESS,
            source_id="low_score",
            original_title="Low Score Product",
            description="Desc",
            cost_price=Decimal("10.00"),
            suggested_price=Decimal("12.00"),
            review_count=5,
            rating=3.0,
            order_count=10,
        ),
        ScoutedProduct(
            source_provider=SupplierProvider.ALIEXPRESS,
            source_id="high_score",
            original_title="High Score Product",
            description="Desc",
            cost_price=Decimal("10.00"),
            suggested_price=Decimal("40.00"),
            images=[ProductImage(url="http://test.com/1.jpg") for _ in range(5)],
            review_count=1000,
            rating=4.9,
            order_count=5000,
            shipping_time_days=7,
        ),
    ]

    ranked = ranker.rank_products(products, top_n=2)

    assert ranked[0].source_id == "high_score"
    assert ranked[0].winning_score > ranked[1].winning_score
```

---

## Verification Checklist

After completing all tasks, verify:

### Project Setup
- [ ] FastAPI app starts without errors: `uvicorn app.main:app --reload`
- [ ] Health endpoint responds: `curl http://localhost:8001/health`
- [ ] API docs accessible: `http://localhost:8001/docs`

### Agents
- [ ] Brand Agent generates valid brand identity
- [ ] Scout Agent returns ranked products
- [ ] Copywriter Agent generates product descriptions and pages

### Integration
- [ ] LLM service connects to OpenAI/Anthropic
- [ ] All agents work together via orchestrator
- [ ] Tests pass: `pytest tests/ -v`

### Docker
- [ ] Container builds successfully: `docker build -t genesis-engine .`
- [ ] Container runs: `docker run -p 8001:8001 genesis-engine`

---

## Next Steps

After completing Sprint 2, proceed to:

**Sprint 3: Media Studio**
- Image processing pipeline
- Background removal
- Watermark detection/removal
- S3 storage integration

Refer to `PHASE1_EXECUTION_PLAN.md` for detailed specifications.
