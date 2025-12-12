# Sprint 3: Media Studio - Actionable Implementation Plan

## Overview

This document provides step-by-step implementation instructions for Sprint 3 of Zyenta's Phase 1 MVP. Sprint 3 builds the **Media Studio** - the image processing service that transforms supplier images into professional product photos.

**Objective:** Build a fully functional image processing pipeline that can remove backgrounds, detect/remove watermarks, enhance images, and generate lifestyle backgrounds.

**Prerequisites:** Sprint 1 (Foundation) and Sprint 2 (Genesis Engine) must be completed with:
- Monorepo configured and working
- Database schema deployed
- Docker infrastructure running
- Genesis Engine service operational

---

## Task 1: FastAPI Project Setup

### 1.1 Directory Structure

```bash
# Create directory structure
mkdir -p services/media-studio/app/{api/v1,processors,services,workers,models}
mkdir -p services/media-studio/tests/{test_processors,test_api}
mkdir -p services/media-studio/samples  # Sample images for testing

# Create __init__.py files
touch services/media-studio/app/__init__.py
touch services/media-studio/app/api/__init__.py
touch services/media-studio/app/api/v1/__init__.py
touch services/media-studio/app/processors/__init__.py
touch services/media-studio/app/services/__init__.py
touch services/media-studio/app/workers/__init__.py
touch services/media-studio/app/models/__init__.py
touch services/media-studio/tests/__init__.py
touch services/media-studio/tests/test_processors/__init__.py
touch services/media-studio/tests/test_api/__init__.py
```

### 1.2 Python Project Configuration

**File: `services/media-studio/pyproject.toml`**
```toml
[project]
name = "media-studio"
version = "0.1.0"
description = "Zyenta Media Studio - AI-powered Image Processing Service"
readme = "README.md"
requires-python = ">=3.11"
license = { text = "MIT" }
authors = [{ name = "Zyenta Team" }]

dependencies = [
    # Web Framework
    "fastapi>=0.109.0",
    "uvicorn[standard]>=0.27.0",
    "pydantic>=2.6.0",
    "pydantic-settings>=2.1.0",

    # HTTP Client
    "httpx>=0.26.0",
    "aiofiles>=23.2.0",

    # Image Processing
    "Pillow>=10.2.0",
    "rembg>=2.0.50",
    "opencv-python-headless>=4.9.0",
    "numpy>=1.26.0",

    # AI/ML
    "torch>=2.2.0",
    "torchvision>=0.17.0",
    "openai>=1.12.0",

    # Task Queue
    "celery[redis]>=5.3.0",
    "redis>=5.0.0",

    # Storage
    "boto3>=1.34.0",

    # Utilities
    "tenacity>=8.2.0",
    "structlog>=24.1.0",
    "python-multipart>=0.0.9",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=4.1.0",
    "black>=24.1.0",
    "ruff>=0.2.0",
    "mypy>=1.8.0",
]

# Optional: GPU acceleration
gpu = [
    "onnxruntime-gpu>=1.17.0",
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

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

**File: `services/media-studio/requirements.txt`**
```txt
# Web Framework
fastapi>=0.109.0
uvicorn[standard]>=0.27.0
pydantic>=2.6.0
pydantic-settings>=2.1.0

# HTTP Client
httpx>=0.26.0
aiofiles>=23.2.0

# Image Processing
Pillow>=10.2.0
rembg>=2.0.50
opencv-python-headless>=4.9.0
numpy>=1.26.0

# AI/ML (CPU versions for broader compatibility)
torch>=2.2.0
torchvision>=0.17.0
onnxruntime>=1.17.0

# OpenAI for DALL-E
openai>=1.12.0

# Task Queue
celery[redis]>=5.3.0
redis>=5.0.0

# Storage
boto3>=1.34.0

# Utilities
tenacity>=8.2.0
structlog>=24.1.0
python-multipart>=0.0.9
```

**File: `services/media-studio/requirements-dev.txt`**
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
```

### 1.3 Configuration Management

**File: `services/media-studio/app/core/config.py`**
```python
"""Application configuration using Pydantic Settings."""

from functools import lru_cache
from typing import Literal

from pydantic import Field
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
    app_name: str = "Media Studio"
    app_version: str = "0.1.0"
    environment: Literal["development", "staging", "production"] = "development"
    debug: bool = False
    log_level: str = "INFO"

    # API
    api_v1_prefix: str = "/api/v1"
    cors_origins: list[str] = ["http://localhost:3000"]
    max_upload_size_mb: int = 50

    # Redis
    redis_url: str = "redis://localhost:6379"

    # S3/MinIO Storage
    s3_endpoint: str = "http://localhost:9000"
    s3_access_key: str = "minioadmin"
    s3_secret_key: str = "minioadmin"
    s3_bucket: str = "zyenta-media"
    s3_region: str = "us-east-1"
    cdn_url: str | None = None  # CloudFront URL in production

    # OpenAI (for DALL-E backgrounds)
    openai_api_key: str | None = None

    # Processing Settings
    image_max_dimension: int = 4096
    image_quality: int = 90
    thumbnail_size: tuple[int, int] = (200, 200)
    medium_size: tuple[int, int] = (800, 800)
    large_size: tuple[int, int] = (1600, 1600)

    # Celery
    celery_broker_url: str = "redis://localhost:6379/0"
    celery_result_backend: str = "redis://localhost:6379/1"

    # Processing Timeouts (seconds)
    download_timeout: int = 30
    processing_timeout: int = 120

    @property
    def storage_base_url(self) -> str:
        """Get the base URL for stored assets."""
        if self.cdn_url:
            return self.cdn_url
        return f"{self.s3_endpoint}/{self.s3_bucket}"


@lru_cache
def get_settings() -> Settings:
    """Get cached settings instance."""
    return Settings()


settings = get_settings()
```

**File: `services/media-studio/app/core/logging.py`**
```python
"""Structured logging configuration."""

import logging
import sys

import structlog


def setup_logging(log_level: str = "INFO") -> None:
    """Configure structured logging."""
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

    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=getattr(logging, log_level.upper()),
    )


def get_logger(name: str) -> structlog.BoundLogger:
    """Get a logger instance."""
    return structlog.get_logger(name)
```

### 1.4 FastAPI Application

**File: `services/media-studio/app/main.py`**
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
    setup_logging(settings.log_level)
    logger.info(
        "Starting Media Studio",
        version=settings.app_version,
        environment=settings.environment,
    )
    yield
    logger.info("Shutting down Media Studio")


def create_app() -> FastAPI:
    """Create and configure the FastAPI application."""
    app = FastAPI(
        title=settings.app_name,
        version=settings.app_version,
        description="Zyenta Media Studio - AI-powered image processing service",
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

**File: `services/media-studio/app/api/health.py`**
```python
"""Health check endpoints."""

from fastapi import APIRouter, status
from pydantic import BaseModel

from app.core.config import settings

router = APIRouter()


class HealthResponse(BaseModel):
    """Health check response."""

    status: str
    version: str
    environment: str


@router.get("/health", response_model=HealthResponse)
async def health_check() -> HealthResponse:
    """Basic health check."""
    return HealthResponse(
        status="healthy",
        version=settings.app_version,
        environment=settings.environment,
    )


@router.get("/ready")
async def readiness_check() -> dict:
    """Readiness check with dependencies."""
    # TODO: Check S3, Redis connections
    return {
        "status": "ready",
        "storage": "connected",
        "redis": "connected",
    }
```

**File: `services/media-studio/app/api/v1/router.py`**
```python
"""API v1 router configuration."""

from fastapi import APIRouter

from app.api.v1 import images, jobs

api_router = APIRouter()

api_router.include_router(images.router, prefix="/images", tags=["Images"])
api_router.include_router(jobs.router, prefix="/jobs", tags=["Jobs"])
```

---

## Task 2: Image Models and Schemas

### 2.1 Pydantic Models

**File: `services/media-studio/app/models/image.py`**
```python
"""Image processing models."""

from enum import Enum
from typing import Any

from pydantic import BaseModel, Field, HttpUrl


class OperationType(str, Enum):
    """Available image operations."""

    REMOVE_BACKGROUND = "remove_background"
    REMOVE_WATERMARK = "remove_watermark"
    ENHANCE = "enhance"
    RESIZE = "resize"
    GENERATE_BACKGROUND = "generate_background"
    CROP = "crop"


class ImageFormat(str, Enum):
    """Supported image formats."""

    JPEG = "jpeg"
    PNG = "png"
    WEBP = "webp"


class ResizeMode(str, Enum):
    """Resize modes."""

    FIT = "fit"  # Fit within dimensions, maintain aspect ratio
    FILL = "fill"  # Fill dimensions, crop if needed
    STRETCH = "stretch"  # Stretch to exact dimensions


class BackgroundStyle(str, Enum):
    """Predefined background styles."""

    WHITE = "white"
    TRANSPARENT = "transparent"
    MARBLE_DESK = "marble_desk"
    WOODEN_TABLE = "wooden_table"
    LIVING_ROOM = "living_room"
    OUTDOOR = "outdoor"
    STUDIO = "studio"
    CUSTOM = "custom"


# Operation Parameters
class RemoveBackgroundParams(BaseModel):
    """Parameters for background removal."""

    output_format: ImageFormat = ImageFormat.PNG
    alpha_matting: bool = False
    alpha_matting_foreground_threshold: int = 240
    alpha_matting_background_threshold: int = 10


class RemoveWatermarkParams(BaseModel):
    """Parameters for watermark removal."""

    detection_sensitivity: float = Field(default=0.5, ge=0.0, le=1.0)
    inpaint_radius: int = Field(default=3, ge=1, le=10)


class EnhanceParams(BaseModel):
    """Parameters for image enhancement."""

    upscale: bool = True
    upscale_factor: int = Field(default=2, ge=1, le=4)
    sharpen: bool = True
    sharpen_amount: float = Field(default=1.0, ge=0.0, le=3.0)
    denoise: bool = False
    auto_contrast: bool = True
    auto_color: bool = True


class ResizeParams(BaseModel):
    """Parameters for resizing."""

    width: int | None = Field(None, ge=1, le=4096)
    height: int | None = Field(None, ge=1, le=4096)
    mode: ResizeMode = ResizeMode.FIT
    background_color: str = "#ffffff"


class GenerateBackgroundParams(BaseModel):
    """Parameters for AI background generation."""

    style: BackgroundStyle = BackgroundStyle.STUDIO
    custom_prompt: str | None = None  # For CUSTOM style
    lighting: str = "soft"  # soft, dramatic, natural
    perspective: str = "front"  # front, angle, top-down


class CropParams(BaseModel):
    """Parameters for cropping."""

    x: int = Field(ge=0)
    y: int = Field(ge=0)
    width: int = Field(ge=1)
    height: int = Field(ge=1)


# Operation Definition
class ImageOperation(BaseModel):
    """Single image operation."""

    type: OperationType
    params: dict[str, Any] = Field(default_factory=dict)


# Request/Response Models
class ProcessImageRequest(BaseModel):
    """Request to process a single image."""

    image_url: HttpUrl
    operations: list[ImageOperation]
    output_format: ImageFormat = ImageFormat.JPEG
    output_quality: int = Field(default=90, ge=1, le=100)
    generate_sizes: bool = True  # Generate thumbnail, medium, large


class BatchProcessRequest(BaseModel):
    """Request to process multiple images."""

    images: list[ProcessImageRequest]
    webhook_url: HttpUrl | None = None  # Callback when complete


class ImageSize(BaseModel):
    """Image size variant."""

    name: str
    url: str
    width: int
    height: int
    size_bytes: int


class ProcessedImage(BaseModel):
    """Processed image result."""

    original_url: str
    sizes: list[ImageSize]
    format: ImageFormat
    metadata: dict[str, Any] = Field(default_factory=dict)


class ProcessImageResponse(BaseModel):
    """Response from image processing."""

    job_id: str
    status: str
    result: ProcessedImage | None = None


class JobStatus(str, Enum):
    """Processing job status."""

    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"


class ProcessingJob(BaseModel):
    """Image processing job."""

    id: str
    status: JobStatus
    progress: int = Field(ge=0, le=100)
    current_operation: str | None = None
    result: ProcessedImage | None = None
    error: str | None = None
    created_at: str
    completed_at: str | None = None
```

---

## Task 3: S3 Storage Service

### 3.1 Storage Service Implementation

**File: `services/media-studio/app/services/storage.py`**
```python
"""S3-compatible storage service."""

import io
import uuid
from datetime import datetime
from pathlib import Path
from typing import BinaryIO

import boto3
from botocore.config import Config
from botocore.exceptions import ClientError
from PIL import Image

from app.core.config import settings
from app.core.logging import get_logger

logger = get_logger(__name__)


class StorageService:
    """
    S3-compatible storage service for media assets.

    Handles uploading, downloading, and URL generation for images.
    Works with both AWS S3 and MinIO.
    """

    def __init__(self):
        """Initialize S3 client."""
        self.client = boto3.client(
            "s3",
            endpoint_url=settings.s3_endpoint,
            aws_access_key_id=settings.s3_access_key,
            aws_secret_access_key=settings.s3_secret_key,
            region_name=settings.s3_region,
            config=Config(signature_version="s3v4"),
        )
        self.bucket = settings.s3_bucket
        self._ensure_bucket_exists()

    def _ensure_bucket_exists(self) -> None:
        """Create bucket if it doesn't exist."""
        try:
            self.client.head_bucket(Bucket=self.bucket)
        except ClientError:
            logger.info(f"Creating bucket: {self.bucket}")
            try:
                self.client.create_bucket(Bucket=self.bucket)
            except ClientError as e:
                logger.error(f"Failed to create bucket: {e}")

    def generate_key(
        self,
        store_id: str,
        product_id: str | None = None,
        filename: str | None = None,
        suffix: str = "",
    ) -> str:
        """
        Generate a unique storage key.

        Format: stores/{store_id}/products/{product_id}/{uuid}_{suffix}.{ext}
        """
        unique_id = uuid.uuid4().hex[:12]
        timestamp = datetime.utcnow().strftime("%Y%m%d")

        if filename:
            ext = Path(filename).suffix.lower() or ".jpg"
        else:
            ext = ".jpg"

        name = f"{timestamp}_{unique_id}"
        if suffix:
            name = f"{name}_{suffix}"
        name = f"{name}{ext}"

        if product_id:
            return f"stores/{store_id}/products/{product_id}/{name}"
        return f"stores/{store_id}/media/{name}"

    async def upload_image(
        self,
        image: Image.Image,
        key: str,
        format: str = "JPEG",
        quality: int = 90,
    ) -> str:
        """
        Upload a PIL Image to S3.

        Args:
            image: PIL Image object
            key: Storage key
            format: Image format (JPEG, PNG, WEBP)
            quality: JPEG/WEBP quality (1-100)

        Returns:
            Public URL of uploaded image
        """
        # Convert image to bytes
        buffer = io.BytesIO()

        save_kwargs = {"format": format}
        if format in ("JPEG", "WEBP"):
            save_kwargs["quality"] = quality
        if format == "JPEG":
            save_kwargs["progressive"] = True
            # Convert RGBA to RGB for JPEG
            if image.mode == "RGBA":
                background = Image.new("RGB", image.size, (255, 255, 255))
                background.paste(image, mask=image.split()[3])
                image = background

        image.save(buffer, **save_kwargs)
        buffer.seek(0)

        # Determine content type
        content_types = {
            "JPEG": "image/jpeg",
            "PNG": "image/png",
            "WEBP": "image/webp",
        }
        content_type = content_types.get(format, "image/jpeg")

        # Upload to S3
        try:
            self.client.upload_fileobj(
                buffer,
                self.bucket,
                key,
                ExtraArgs={
                    "ContentType": content_type,
                    "CacheControl": "max-age=31536000",  # 1 year cache
                },
            )

            url = self.get_public_url(key)
            logger.info("Uploaded image", key=key, format=format, size=buffer.tell())
            return url

        except ClientError as e:
            logger.error("Failed to upload image", key=key, error=str(e))
            raise

    async def upload_bytes(
        self,
        data: bytes,
        key: str,
        content_type: str = "image/jpeg",
    ) -> str:
        """Upload raw bytes to S3."""
        try:
            self.client.put_object(
                Bucket=self.bucket,
                Key=key,
                Body=data,
                ContentType=content_type,
                CacheControl="max-age=31536000",
            )
            return self.get_public_url(key)
        except ClientError as e:
            logger.error("Failed to upload bytes", key=key, error=str(e))
            raise

    async def download_image(self, key: str) -> Image.Image:
        """Download an image from S3 as PIL Image."""
        try:
            response = self.client.get_object(Bucket=self.bucket, Key=key)
            image_data = response["Body"].read()
            return Image.open(io.BytesIO(image_data))
        except ClientError as e:
            logger.error("Failed to download image", key=key, error=str(e))
            raise

    def get_public_url(self, key: str) -> str:
        """Get the public URL for a stored object."""
        if settings.cdn_url:
            return f"{settings.cdn_url}/{key}"
        return f"{settings.s3_endpoint}/{self.bucket}/{key}"

    async def delete(self, key: str) -> bool:
        """Delete an object from S3."""
        try:
            self.client.delete_object(Bucket=self.bucket, Key=key)
            logger.info("Deleted object", key=key)
            return True
        except ClientError as e:
            logger.error("Failed to delete object", key=key, error=str(e))
            return False

    async def exists(self, key: str) -> bool:
        """Check if an object exists."""
        try:
            self.client.head_object(Bucket=self.bucket, Key=key)
            return True
        except ClientError:
            return False


# Singleton instance
_storage: StorageService | None = None


def get_storage() -> StorageService:
    """Get storage service singleton."""
    global _storage
    if _storage is None:
        _storage = StorageService()
    return _storage
```

---

## Task 4: Base Processor and Image Utilities

### 4.1 Base Processor Class

**File: `services/media-studio/app/processors/base.py`**
```python
"""Base processor class for image operations."""

from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Any

from PIL import Image

from app.core.logging import get_logger


@dataclass
class ProcessingResult:
    """Result of an image processing operation."""

    success: bool
    image: Image.Image | None = None
    error: str | None = None
    metadata: dict[str, Any] | None = None


class BaseProcessor(ABC):
    """Abstract base class for image processors."""

    def __init__(self):
        self.logger = get_logger(self.__class__.__name__)

    @property
    @abstractmethod
    def name(self) -> str:
        """Processor name."""
        pass

    @abstractmethod
    async def process(
        self,
        image: Image.Image,
        params: dict[str, Any],
    ) -> ProcessingResult:
        """
        Process an image.

        Args:
            image: Input PIL Image
            params: Processing parameters

        Returns:
            ProcessingResult with processed image
        """
        pass

    def validate_image(self, image: Image.Image) -> bool:
        """Validate input image."""
        if image is None:
            return False
        if image.size[0] < 10 or image.size[1] < 10:
            return False
        return True
```

### 4.2 Image Utilities

**File: `services/media-studio/app/services/image_utils.py`**
```python
"""Image utility functions."""

import io
from typing import Tuple

import httpx
from PIL import Image, ImageEnhance, ImageFilter

from app.core.config import settings
from app.core.logging import get_logger

logger = get_logger(__name__)


async def download_image(url: str) -> Image.Image:
    """
    Download an image from a URL.

    Args:
        url: Image URL

    Returns:
        PIL Image object
    """
    async with httpx.AsyncClient(timeout=settings.download_timeout) as client:
        response = await client.get(url)
        response.raise_for_status()

        image_data = response.content
        image = Image.open(io.BytesIO(image_data))

        # Convert to RGB/RGBA
        if image.mode not in ("RGB", "RGBA"):
            if image.mode == "P" and "transparency" in image.info:
                image = image.convert("RGBA")
            else:
                image = image.convert("RGB")

        logger.info(
            "Downloaded image",
            url=url[:100],
            size=image.size,
            mode=image.mode,
        )

        return image


def resize_image(
    image: Image.Image,
    max_size: Tuple[int, int],
    mode: str = "fit",
    background_color: str = "#ffffff",
) -> Image.Image:
    """
    Resize an image.

    Args:
        image: Input image
        max_size: Maximum (width, height)
        mode: fit, fill, or stretch
        background_color: Background for padding

    Returns:
        Resized image
    """
    if mode == "fit":
        # Maintain aspect ratio, fit within bounds
        image.thumbnail(max_size, Image.Resampling.LANCZOS)
        return image

    elif mode == "fill":
        # Fill dimensions, crop if needed
        target_ratio = max_size[0] / max_size[1]
        image_ratio = image.width / image.height

        if image_ratio > target_ratio:
            # Image is wider, crop width
            new_width = int(image.height * target_ratio)
            left = (image.width - new_width) // 2
            image = image.crop((left, 0, left + new_width, image.height))
        else:
            # Image is taller, crop height
            new_height = int(image.width / target_ratio)
            top = (image.height - new_height) // 2
            image = image.crop((0, top, image.width, top + new_height))

        return image.resize(max_size, Image.Resampling.LANCZOS)

    elif mode == "stretch":
        return image.resize(max_size, Image.Resampling.LANCZOS)

    return image


def generate_sizes(
    image: Image.Image,
) -> dict[str, Image.Image]:
    """
    Generate multiple size variants of an image.

    Returns:
        Dictionary with thumbnail, medium, large variants
    """
    sizes = {}

    # Thumbnail
    thumb = image.copy()
    thumb.thumbnail(settings.thumbnail_size, Image.Resampling.LANCZOS)
    sizes["thumbnail"] = thumb

    # Medium
    medium = image.copy()
    medium.thumbnail(settings.medium_size, Image.Resampling.LANCZOS)
    sizes["medium"] = medium

    # Large
    large = image.copy()
    large.thumbnail(settings.large_size, Image.Resampling.LANCZOS)
    sizes["large"] = large

    return sizes


def auto_enhance(image: Image.Image) -> Image.Image:
    """Apply automatic enhancements to an image."""
    # Auto contrast
    enhancer = ImageEnhance.Contrast(image)
    image = enhancer.enhance(1.1)

    # Auto color
    enhancer = ImageEnhance.Color(image)
    image = enhancer.enhance(1.05)

    # Slight sharpening
    image = image.filter(ImageFilter.UnsharpMask(radius=1, percent=50, threshold=3))

    return image


def get_image_metadata(image: Image.Image) -> dict:
    """Extract metadata from an image."""
    return {
        "width": image.width,
        "height": image.height,
        "mode": image.mode,
        "format": image.format,
        "has_transparency": image.mode == "RGBA",
    }
```

---

## Task 5: Background Removal Processor

### 5.1 Background Removal Implementation

**File: `services/media-studio/app/processors/background.py`**
```python
"""Background removal processor using rembg."""

from typing import Any

from PIL import Image
from rembg import remove, new_session

from app.processors.base import BaseProcessor, ProcessingResult
from app.core.logging import get_logger

logger = get_logger(__name__)

# Initialize rembg session (loads model once)
_session = None


def get_rembg_session():
    """Get or create rembg session."""
    global _session
    if _session is None:
        logger.info("Loading rembg model...")
        _session = new_session("u2net")
        logger.info("rembg model loaded")
    return _session


class BackgroundRemovalProcessor(BaseProcessor):
    """
    Removes background from product images using rembg.

    Uses the U2-Net model for accurate segmentation.
    Outputs PNG with transparency.
    """

    @property
    def name(self) -> str:
        return "Background Removal"

    async def process(
        self,
        image: Image.Image,
        params: dict[str, Any],
    ) -> ProcessingResult:
        """
        Remove background from image.

        Params:
            alpha_matting: bool - Enable alpha matting for better edges
            alpha_matting_foreground_threshold: int - Foreground threshold
            alpha_matting_background_threshold: int - Background threshold
        """
        try:
            self.logger.info(
                "Removing background",
                size=image.size,
                mode=image.mode,
            )

            # Get parameters
            alpha_matting = params.get("alpha_matting", False)
            fg_threshold = params.get("alpha_matting_foreground_threshold", 240)
            bg_threshold = params.get("alpha_matting_background_threshold", 10)

            # Get session
            session = get_rembg_session()

            # Remove background
            result = remove(
                image,
                session=session,
                alpha_matting=alpha_matting,
                alpha_matting_foreground_threshold=fg_threshold,
                alpha_matting_background_threshold=bg_threshold,
            )

            # Ensure RGBA mode
            if result.mode != "RGBA":
                result = result.convert("RGBA")

            self.logger.info(
                "Background removed successfully",
                output_size=result.size,
            )

            return ProcessingResult(
                success=True,
                image=result,
                metadata={
                    "alpha_matting": alpha_matting,
                    "has_transparency": True,
                },
            )

        except Exception as e:
            self.logger.error("Background removal failed", error=str(e))
            return ProcessingResult(
                success=False,
                error=str(e),
            )


class BackgroundReplacementProcessor(BaseProcessor):
    """
    Replaces image background with a solid color or gradient.

    For AI-generated backgrounds, see LifestyleBackgroundProcessor.
    """

    @property
    def name(self) -> str:
        return "Background Replacement"

    async def process(
        self,
        image: Image.Image,
        params: dict[str, Any],
    ) -> ProcessingResult:
        """
        Replace background with specified color.

        Params:
            color: str - Hex color code (e.g., "#ffffff")
            transparent: bool - Keep transparent background
        """
        try:
            color = params.get("color", "#ffffff")
            keep_transparent = params.get("transparent", False)

            if keep_transparent:
                # Just ensure RGBA mode
                if image.mode != "RGBA":
                    image = image.convert("RGBA")
                return ProcessingResult(success=True, image=image)

            # Create background
            background = Image.new("RGB", image.size, color)

            # Composite if image has transparency
            if image.mode == "RGBA":
                background.paste(image, mask=image.split()[3])
            else:
                background = image.convert("RGB")

            return ProcessingResult(
                success=True,
                image=background,
                metadata={"background_color": color},
            )

        except Exception as e:
            self.logger.error("Background replacement failed", error=str(e))
            return ProcessingResult(success=False, error=str(e))
```

---

## Task 6: Watermark Removal Processor

### 6.1 Watermark Detection and Removal

**File: `services/media-studio/app/processors/watermark.py`**
```python
"""Watermark detection and removal processor."""

from typing import Any, Tuple

import cv2
import numpy as np
from PIL import Image

from app.processors.base import BaseProcessor, ProcessingResult
from app.core.logging import get_logger

logger = get_logger(__name__)


class WatermarkRemovalProcessor(BaseProcessor):
    """
    Detects and removes watermarks from images.

    Uses a combination of techniques:
    1. Edge detection to find watermark regions
    2. Inpainting to fill removed areas

    Note: This is a basic implementation. For production,
    consider using more advanced ML-based approaches.
    """

    @property
    def name(self) -> str:
        return "Watermark Removal"

    async def process(
        self,
        image: Image.Image,
        params: dict[str, Any],
    ) -> ProcessingResult:
        """
        Detect and remove watermarks.

        Params:
            detection_sensitivity: float - Sensitivity for detection (0-1)
            inpaint_radius: int - Radius for inpainting (1-10)
        """
        try:
            sensitivity = params.get("detection_sensitivity", 0.5)
            inpaint_radius = params.get("inpaint_radius", 3)

            self.logger.info(
                "Attempting watermark removal",
                sensitivity=sensitivity,
                inpaint_radius=inpaint_radius,
            )

            # Convert PIL to OpenCV format
            cv_image = self._pil_to_cv2(image)

            # Detect watermark regions
            mask = self._detect_watermark(cv_image, sensitivity)

            if mask is None or np.sum(mask) == 0:
                self.logger.info("No watermark detected")
                return ProcessingResult(
                    success=True,
                    image=image,
                    metadata={"watermark_detected": False},
                )

            # Inpaint to remove watermark
            result = cv2.inpaint(
                cv_image,
                mask,
                inpaint_radius,
                cv2.INPAINT_TELEA,
            )

            # Convert back to PIL
            result_pil = self._cv2_to_pil(result, image.mode)

            self.logger.info("Watermark removed successfully")
            return ProcessingResult(
                success=True,
                image=result_pil,
                metadata={
                    "watermark_detected": True,
                    "watermark_area_percent": float(np.sum(mask) / mask.size * 100),
                },
            )

        except Exception as e:
            self.logger.error("Watermark removal failed", error=str(e))
            return ProcessingResult(success=False, error=str(e))

    def _detect_watermark(
        self,
        image: np.ndarray,
        sensitivity: float,
    ) -> np.ndarray | None:
        """
        Detect watermark regions in an image.

        This uses a combination of techniques:
        1. Convert to grayscale
        2. Apply edge detection
        3. Look for semi-transparent overlays
        4. Find text-like patterns

        Returns a binary mask of detected watermark regions.
        """
        # Convert to grayscale
        if len(image.shape) == 3:
            gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
        else:
            gray = image

        # Detect edges
        edges = cv2.Canny(gray, 50, 150)

        # Look for semi-transparent regions (common in watermarks)
        # This is simplified - real implementation would be more sophisticated

        # Threshold to find light regions (potential watermarks)
        threshold = int(255 * (1 - sensitivity * 0.3))
        _, light_regions = cv2.threshold(gray, threshold, 255, cv2.THRESH_BINARY)

        # Find regions that are both edges and light
        potential_watermark = cv2.bitwise_and(edges, light_regions)

        # Dilate to connect nearby regions
        kernel = np.ones((5, 5), np.uint8)
        dilated = cv2.dilate(potential_watermark, kernel, iterations=2)

        # Find contours and filter by size
        contours, _ = cv2.findContours(
            dilated, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE
        )

        # Create mask from significant contours
        mask = np.zeros(gray.shape, dtype=np.uint8)
        image_area = gray.shape[0] * gray.shape[1]

        for contour in contours:
            area = cv2.contourArea(contour)
            # Watermarks are typically 1-20% of image area
            if 0.01 * image_area < area < 0.2 * image_area:
                cv2.drawContours(mask, [contour], -1, 255, -1)

        # Dilate mask slightly for better inpainting
        mask = cv2.dilate(mask, kernel, iterations=1)

        return mask

    def _pil_to_cv2(self, image: Image.Image) -> np.ndarray:
        """Convert PIL Image to OpenCV format."""
        if image.mode == "RGBA":
            # Convert RGBA to BGR
            return cv2.cvtColor(np.array(image), cv2.COLOR_RGBA2BGR)
        elif image.mode == "RGB":
            return cv2.cvtColor(np.array(image), cv2.COLOR_RGB2BGR)
        else:
            return np.array(image.convert("RGB"))

    def _cv2_to_pil(self, image: np.ndarray, mode: str = "RGB") -> Image.Image:
        """Convert OpenCV image to PIL format."""
        if len(image.shape) == 3:
            image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
        pil_image = Image.fromarray(image)
        if mode == "RGBA" and pil_image.mode != "RGBA":
            pil_image = pil_image.convert("RGBA")
        return pil_image
```

---

## Task 7: Image Enhancement Processor

### 7.1 Enhancement Implementation

**File: `services/media-studio/app/processors/enhancement.py`**
```python
"""Image enhancement processor."""

from typing import Any

import numpy as np
from PIL import Image, ImageEnhance, ImageFilter

from app.processors.base import BaseProcessor, ProcessingResult
from app.core.logging import get_logger

logger = get_logger(__name__)


class EnhancementProcessor(BaseProcessor):
    """
    Enhances image quality through various techniques.

    Capabilities:
    - Upscaling (basic bicubic, or Real-ESRGAN if available)
    - Sharpening
    - Denoising
    - Auto contrast/color adjustment
    """

    @property
    def name(self) -> str:
        return "Image Enhancement"

    async def process(
        self,
        image: Image.Image,
        params: dict[str, Any],
    ) -> ProcessingResult:
        """
        Enhance image quality.

        Params:
            upscale: bool - Whether to upscale
            upscale_factor: int - Upscale factor (2-4)
            sharpen: bool - Apply sharpening
            sharpen_amount: float - Sharpening intensity (0-3)
            denoise: bool - Apply denoising
            auto_contrast: bool - Auto adjust contrast
            auto_color: bool - Auto adjust color saturation
        """
        try:
            result = image.copy()

            # Upscaling
            if params.get("upscale", True):
                factor = params.get("upscale_factor", 2)
                result = self._upscale(result, factor)
                self.logger.info(f"Upscaled by {factor}x")

            # Denoising (before sharpening)
            if params.get("denoise", False):
                result = self._denoise(result)
                self.logger.info("Applied denoising")

            # Auto contrast
            if params.get("auto_contrast", True):
                result = self._auto_contrast(result)
                self.logger.info("Applied auto contrast")

            # Auto color
            if params.get("auto_color", True):
                result = self._auto_color(result)
                self.logger.info("Applied auto color")

            # Sharpening (last step)
            if params.get("sharpen", True):
                amount = params.get("sharpen_amount", 1.0)
                result = self._sharpen(result, amount)
                self.logger.info(f"Applied sharpening: {amount}")

            return ProcessingResult(
                success=True,
                image=result,
                metadata={
                    "enhanced": True,
                    "final_size": result.size,
                },
            )

        except Exception as e:
            self.logger.error("Enhancement failed", error=str(e))
            return ProcessingResult(success=False, error=str(e))

    def _upscale(self, image: Image.Image, factor: int) -> Image.Image:
        """
        Upscale image.

        Uses high-quality bicubic resampling.
        For production, consider integrating Real-ESRGAN.
        """
        new_size = (image.width * factor, image.height * factor)
        return image.resize(new_size, Image.Resampling.LANCZOS)

    def _sharpen(self, image: Image.Image, amount: float) -> Image.Image:
        """Apply unsharp mask sharpening."""
        if amount <= 0:
            return image

        # Unsharp mask parameters based on amount
        radius = 1.0 + (amount * 0.5)
        percent = int(50 + (amount * 50))
        threshold = 3

        return image.filter(
            ImageFilter.UnsharpMask(
                radius=radius,
                percent=percent,
                threshold=threshold,
            )
        )

    def _denoise(self, image: Image.Image) -> Image.Image:
        """Apply median filter for noise reduction."""
        return image.filter(ImageFilter.MedianFilter(size=3))

    def _auto_contrast(self, image: Image.Image) -> Image.Image:
        """Automatically adjust contrast."""
        enhancer = ImageEnhance.Contrast(image)

        # Calculate current contrast and adjust
        # Simple approach: slight boost
        return enhancer.enhance(1.1)

    def _auto_color(self, image: Image.Image) -> Image.Image:
        """Automatically adjust color saturation."""
        if image.mode not in ("RGB", "RGBA"):
            return image

        enhancer = ImageEnhance.Color(image)
        return enhancer.enhance(1.05)


class ResizeProcessor(BaseProcessor):
    """Resizes images to specified dimensions."""

    @property
    def name(self) -> str:
        return "Resize"

    async def process(
        self,
        image: Image.Image,
        params: dict[str, Any],
    ) -> ProcessingResult:
        """
        Resize image.

        Params:
            width: int - Target width
            height: int - Target height
            mode: str - fit, fill, or stretch
            background_color: str - Background color for padding
        """
        try:
            width = params.get("width")
            height = params.get("height")
            mode = params.get("mode", "fit")
            bg_color = params.get("background_color", "#ffffff")

            if not width and not height:
                return ProcessingResult(
                    success=False,
                    error="Width or height must be specified",
                )

            # Calculate missing dimension
            if not width:
                width = int(image.width * (height / image.height))
            if not height:
                height = int(image.height * (width / image.width))

            target_size = (width, height)

            if mode == "fit":
                result = image.copy()
                result.thumbnail(target_size, Image.Resampling.LANCZOS)

            elif mode == "fill":
                result = self._fill_resize(image, target_size)

            elif mode == "stretch":
                result = image.resize(target_size, Image.Resampling.LANCZOS)

            else:
                return ProcessingResult(
                    success=False,
                    error=f"Unknown resize mode: {mode}",
                )

            return ProcessingResult(
                success=True,
                image=result,
                metadata={
                    "original_size": image.size,
                    "new_size": result.size,
                    "mode": mode,
                },
            )

        except Exception as e:
            self.logger.error("Resize failed", error=str(e))
            return ProcessingResult(success=False, error=str(e))

    def _fill_resize(
        self,
        image: Image.Image,
        target_size: tuple[int, int],
    ) -> Image.Image:
        """Resize to fill, cropping if necessary."""
        target_ratio = target_size[0] / target_size[1]
        image_ratio = image.width / image.height

        if image_ratio > target_ratio:
            # Crop width
            new_width = int(image.height * target_ratio)
            left = (image.width - new_width) // 2
            image = image.crop((left, 0, left + new_width, image.height))
        else:
            # Crop height
            new_height = int(image.width / target_ratio)
            top = (image.height - new_height) // 2
            image = image.crop((0, top, image.width, top + new_height))

        return image.resize(target_size, Image.Resampling.LANCZOS)
```

---

## Task 8: Lifestyle Background Generator

### 8.1 AI Background Generation

**File: `services/media-studio/app/processors/lifestyle.py`**
```python
"""Lifestyle background generation using AI."""

from typing import Any

from PIL import Image
import openai

from app.processors.base import BaseProcessor, ProcessingResult
from app.core.config import settings
from app.core.logging import get_logger

logger = get_logger(__name__)


# Predefined background prompts
BACKGROUND_PROMPTS = {
    "marble_desk": "Product photography on elegant white marble desk surface, soft natural lighting from window, minimalist style, clean aesthetic, professional product photo background",
    "wooden_table": "Product on rustic wooden table surface, warm natural lighting, cozy atmosphere, lifestyle product photography, shallow depth of field",
    "living_room": "Product in modern minimalist living room setting, neutral colors, soft ambient lighting, lifestyle photography, blurred background",
    "outdoor": "Product in bright outdoor setting, natural daylight, garden or patio background, fresh and vibrant atmosphere, lifestyle photography",
    "studio": "Product on clean white studio background, professional lighting setup, soft shadows, commercial product photography",
    "white": "Pure white background, professional product photography, soft even lighting",
}


class LifestyleBackgroundProcessor(BaseProcessor):
    """
    Generates lifestyle backgrounds for product images.

    Uses DALL-E to generate contextual backgrounds,
    then composites the product onto them.
    """

    def __init__(self):
        super().__init__()
        if settings.openai_api_key:
            self.client = openai.AsyncOpenAI(api_key=settings.openai_api_key)
        else:
            self.client = None

    @property
    def name(self) -> str:
        return "Lifestyle Background Generator"

    async def process(
        self,
        image: Image.Image,
        params: dict[str, Any],
    ) -> ProcessingResult:
        """
        Generate and apply lifestyle background.

        Params:
            style: str - Background style (marble_desk, wooden_table, etc.)
            custom_prompt: str - Custom prompt for CUSTOM style
            lighting: str - Lighting preference (soft, dramatic, natural)
            perspective: str - Camera perspective (front, angle, top-down)
        """
        try:
            style = params.get("style", "studio")
            custom_prompt = params.get("custom_prompt")
            lighting = params.get("lighting", "soft")

            # Input must have transparency (background already removed)
            if image.mode != "RGBA":
                return ProcessingResult(
                    success=False,
                    error="Image must have transparent background (RGBA mode)",
                )

            # For simple styles, use solid colors
            if style == "white":
                result = self._apply_solid_background(image, "#ffffff")
                return ProcessingResult(success=True, image=result)

            if style == "transparent":
                return ProcessingResult(success=True, image=image)

            # Generate AI background
            if not self.client:
                self.logger.warning("OpenAI not configured, using solid background")
                result = self._apply_solid_background(image, "#f5f5f5")
                return ProcessingResult(
                    success=True,
                    image=result,
                    metadata={"fallback": True},
                )

            # Build prompt
            if style == "custom" and custom_prompt:
                prompt = f"Product photography background: {custom_prompt}, {lighting} lighting, professional quality"
            else:
                base_prompt = BACKGROUND_PROMPTS.get(style, BACKGROUND_PROMPTS["studio"])
                prompt = f"{base_prompt}, {lighting} lighting"

            # Generate background
            background = await self._generate_background(prompt, image.size)

            if background is None:
                self.logger.warning("Background generation failed, using fallback")
                result = self._apply_solid_background(image, "#f5f5f5")
                return ProcessingResult(
                    success=True,
                    image=result,
                    metadata={"fallback": True},
                )

            # Composite product onto background
            result = self._composite(image, background)

            return ProcessingResult(
                success=True,
                image=result,
                metadata={
                    "style": style,
                    "ai_generated": True,
                },
            )

        except Exception as e:
            self.logger.error("Lifestyle background generation failed", error=str(e))
            return ProcessingResult(success=False, error=str(e))

    async def _generate_background(
        self,
        prompt: str,
        size: tuple[int, int],
    ) -> Image.Image | None:
        """Generate background using DALL-E."""
        try:
            # DALL-E 3 supports specific sizes
            # Map to closest supported size
            dalle_size = self._get_dalle_size(size)

            self.logger.info(
                "Generating background with DALL-E",
                prompt=prompt[:100],
                size=dalle_size,
            )

            response = await self.client.images.generate(
                model="dall-e-3",
                prompt=prompt,
                size=dalle_size,
                quality="standard",
                n=1,
            )

            # Download the generated image
            image_url = response.data[0].url

            import httpx
            async with httpx.AsyncClient() as client:
                resp = await client.get(image_url)
                resp.raise_for_status()

                from io import BytesIO
                background = Image.open(BytesIO(resp.content))

                # Resize to match product image size
                background = background.resize(size, Image.Resampling.LANCZOS)

                return background

        except Exception as e:
            self.logger.error("DALL-E generation failed", error=str(e))
            return None

    def _get_dalle_size(self, size: tuple[int, int]) -> str:
        """Map image size to supported DALL-E size."""
        # DALL-E 3 supports: 1024x1024, 1024x1792, 1792x1024
        width, height = size
        ratio = width / height

        if ratio > 1.5:
            return "1792x1024"
        elif ratio < 0.67:
            return "1024x1792"
        else:
            return "1024x1024"

    def _apply_solid_background(
        self,
        image: Image.Image,
        color: str,
    ) -> Image.Image:
        """Apply solid color background."""
        background = Image.new("RGB", image.size, color)
        if image.mode == "RGBA":
            background.paste(image, mask=image.split()[3])
        else:
            background.paste(image)
        return background

    def _composite(
        self,
        product: Image.Image,
        background: Image.Image,
    ) -> Image.Image:
        """Composite product onto background."""
        # Ensure same size
        if background.size != product.size:
            background = background.resize(product.size, Image.Resampling.LANCZOS)

        # Convert background to RGB if needed
        if background.mode != "RGB":
            background = background.convert("RGB")

        # Composite
        result = background.copy()
        if product.mode == "RGBA":
            result.paste(product, mask=product.split()[3])
        else:
            result.paste(product)

        return result
```

---

## Task 9: Processing Pipeline and API

### 9.1 Image Processing Pipeline

**File: `services/media-studio/app/services/pipeline.py`**
```python
"""Image processing pipeline orchestrator."""

import uuid
from datetime import datetime
from typing import Any

from PIL import Image

from app.core.logging import get_logger
from app.models.image import (
    ImageOperation,
    ImageSize,
    OperationType,
    ProcessedImage,
    ProcessImageRequest,
)
from app.processors.background import (
    BackgroundRemovalProcessor,
    BackgroundReplacementProcessor,
)
from app.processors.enhancement import EnhancementProcessor, ResizeProcessor
from app.processors.lifestyle import LifestyleBackgroundProcessor
from app.processors.watermark import WatermarkRemovalProcessor
from app.services.image_utils import download_image, generate_sizes, get_image_metadata
from app.services.storage import get_storage

logger = get_logger(__name__)


class ImageProcessingPipeline:
    """
    Orchestrates image processing operations.

    Processes images through a sequence of operations
    and uploads results to storage.
    """

    def __init__(self):
        self.storage = get_storage()

        # Initialize processors
        self.processors = {
            OperationType.REMOVE_BACKGROUND: BackgroundRemovalProcessor(),
            OperationType.REMOVE_WATERMARK: WatermarkRemovalProcessor(),
            OperationType.ENHANCE: EnhancementProcessor(),
            OperationType.RESIZE: ResizeProcessor(),
            OperationType.GENERATE_BACKGROUND: LifestyleBackgroundProcessor(),
        }

    async def process(
        self,
        request: ProcessImageRequest,
        store_id: str,
        product_id: str | None = None,
    ) -> ProcessedImage:
        """
        Process an image through the pipeline.

        Args:
            request: Processing request with operations
            store_id: Store ID for storage organization
            product_id: Optional product ID

        Returns:
            ProcessedImage with URLs to processed images
        """
        job_id = uuid.uuid4().hex[:12]

        logger.info(
            "Starting image processing",
            job_id=job_id,
            url=str(request.image_url)[:100],
            operations=[op.type for op in request.operations],
        )

        try:
            # Download image
            image = await download_image(str(request.image_url))
            original_metadata = get_image_metadata(image)

            # Apply operations in sequence
            for operation in request.operations:
                image = await self._apply_operation(image, operation)

            # Generate size variants
            sizes: list[ImageSize] = []

            if request.generate_sizes:
                size_variants = generate_sizes(image)

                for size_name, size_image in size_variants.items():
                    key = self.storage.generate_key(
                        store_id=store_id,
                        product_id=product_id,
                        suffix=size_name,
                    )

                    url = await self.storage.upload_image(
                        image=size_image,
                        key=key,
                        format=request.output_format.value.upper(),
                        quality=request.output_quality,
                    )

                    sizes.append(ImageSize(
                        name=size_name,
                        url=url,
                        width=size_image.width,
                        height=size_image.height,
                        size_bytes=0,  # TODO: Calculate actual size
                    ))
            else:
                # Just upload the processed image
                key = self.storage.generate_key(
                    store_id=store_id,
                    product_id=product_id,
                )

                url = await self.storage.upload_image(
                    image=image,
                    key=key,
                    format=request.output_format.value.upper(),
                    quality=request.output_quality,
                )

                sizes.append(ImageSize(
                    name="original",
                    url=url,
                    width=image.width,
                    height=image.height,
                    size_bytes=0,
                ))

            logger.info(
                "Image processing completed",
                job_id=job_id,
                sizes=[s.name for s in sizes],
            )

            return ProcessedImage(
                original_url=str(request.image_url),
                sizes=sizes,
                format=request.output_format,
                metadata={
                    "original": original_metadata,
                    "operations_applied": [op.type.value for op in request.operations],
                    "processed_at": datetime.utcnow().isoformat(),
                },
            )

        except Exception as e:
            logger.error(
                "Image processing failed",
                job_id=job_id,
                error=str(e),
            )
            raise

    async def _apply_operation(
        self,
        image: Image.Image,
        operation: ImageOperation,
    ) -> Image.Image:
        """Apply a single operation to an image."""
        processor = self.processors.get(operation.type)

        if processor is None:
            logger.warning(f"Unknown operation type: {operation.type}")
            return image

        result = await processor.process(image, operation.params)

        if not result.success:
            logger.error(
                f"Operation failed: {operation.type}",
                error=result.error,
            )
            raise Exception(f"Operation {operation.type} failed: {result.error}")

        return result.image
```

### 9.2 API Endpoints

**File: `services/media-studio/app/api/v1/images.py`**
```python
"""Image processing API endpoints."""

import uuid
from typing import Any

from fastapi import APIRouter, BackgroundTasks, HTTPException, status
from pydantic import BaseModel

from app.core.logging import get_logger
from app.models.image import (
    BatchProcessRequest,
    ProcessedImage,
    ProcessImageRequest,
    ProcessImageResponse,
)
from app.services.pipeline import ImageProcessingPipeline

logger = get_logger(__name__)
router = APIRouter()

# In-memory job storage (use Redis in production)
_jobs: dict[str, dict[str, Any]] = {}


class ProcessRequest(BaseModel):
    """Request wrapper with store context."""

    store_id: str
    product_id: str | None = None
    request: ProcessImageRequest


@router.post(
    "/process",
    response_model=ProcessImageResponse,
    status_code=status.HTTP_202_ACCEPTED,
)
async def process_image(
    data: ProcessRequest,
    background_tasks: BackgroundTasks,
) -> ProcessImageResponse:
    """
    Process a single image.

    Operations are applied in the order specified.
    Results are uploaded to S3 and URLs are returned.
    """
    job_id = f"img_{uuid.uuid4().hex[:12]}"

    logger.info(
        "Image processing requested",
        job_id=job_id,
        store_id=data.store_id,
        operations=[op.type for op in data.request.operations],
    )

    # Initialize job status
    _jobs[job_id] = {
        "status": "processing",
        "progress": 0,
        "result": None,
        "error": None,
    }

    # Process in background
    background_tasks.add_task(
        _process_image_task,
        job_id,
        data.store_id,
        data.product_id,
        data.request,
    )

    return ProcessImageResponse(
        job_id=job_id,
        status="processing",
        result=None,
    )


@router.post(
    "/process/sync",
    response_model=ProcessedImage,
)
async def process_image_sync(data: ProcessRequest) -> ProcessedImage:
    """
    Process a single image synchronously.

    Use this for simple operations or when immediate results are needed.
    For batch processing or complex operations, use /process.
    """
    pipeline = ImageProcessingPipeline()

    result = await pipeline.process(
        request=data.request,
        store_id=data.store_id,
        product_id=data.product_id,
    )

    return result


@router.post("/batch", status_code=status.HTTP_202_ACCEPTED)
async def batch_process(
    data: BatchProcessRequest,
    background_tasks: BackgroundTasks,
) -> dict:
    """
    Process multiple images in batch.

    Returns a batch job ID to track progress.
    """
    batch_id = f"batch_{uuid.uuid4().hex[:12]}"

    # TODO: Implement batch processing with Celery

    return {
        "batch_id": batch_id,
        "status": "queued",
        "total_images": len(data.images),
    }


async def _process_image_task(
    job_id: str,
    store_id: str,
    product_id: str | None,
    request: ProcessImageRequest,
) -> None:
    """Background task for image processing."""
    try:
        pipeline = ImageProcessingPipeline()
        result = await pipeline.process(
            request=request,
            store_id=store_id,
            product_id=product_id,
        )

        _jobs[job_id] = {
            "status": "completed",
            "progress": 100,
            "result": result.model_dump(),
            "error": None,
        }

    except Exception as e:
        logger.error(f"Background processing failed: {e}")
        _jobs[job_id] = {
            "status": "failed",
            "progress": 0,
            "result": None,
            "error": str(e),
        }
```

**File: `services/media-studio/app/api/v1/jobs.py`**
```python
"""Job status endpoints."""

from fastapi import APIRouter, HTTPException, status

from app.core.logging import get_logger
from app.models.image import ProcessingJob, JobStatus

logger = get_logger(__name__)
router = APIRouter()

# Reference to jobs storage from images module
from app.api.v1.images import _jobs


@router.get("/{job_id}", response_model=ProcessingJob)
async def get_job_status(job_id: str) -> ProcessingJob:
    """Get the status of a processing job."""
    job = _jobs.get(job_id)

    if not job:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Job {job_id} not found",
        )

    return ProcessingJob(
        id=job_id,
        status=JobStatus(job["status"]),
        progress=job["progress"],
        current_operation=None,
        result=job["result"],
        error=job["error"],
        created_at="",  # TODO: Track timestamps
        completed_at=None,
    )
```

---

## Task 10: Celery Workers

### 10.1 Celery Configuration

**File: `services/media-studio/app/workers/celery_app.py`**
```python
"""Celery application configuration."""

from celery import Celery

from app.core.config import settings

celery_app = Celery(
    "media_studio",
    broker=settings.celery_broker_url,
    backend=settings.celery_result_backend,
    include=["app.workers.tasks"],
)

# Celery configuration
celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",
    enable_utc=True,
    task_track_started=True,
    task_time_limit=300,  # 5 minutes
    task_soft_time_limit=240,  # 4 minutes
    worker_prefetch_multiplier=1,
    worker_concurrency=4,
)

# Task routes
celery_app.conf.task_routes = {
    "app.workers.tasks.process_image": {"queue": "media"},
    "app.workers.tasks.process_batch": {"queue": "media"},
}
```

**File: `services/media-studio/app/workers/tasks.py`**
```python
"""Celery tasks for image processing."""

import asyncio
from typing import Any

from app.workers.celery_app import celery_app
from app.core.logging import get_logger
from app.models.image import ProcessImageRequest
from app.services.pipeline import ImageProcessingPipeline

logger = get_logger(__name__)


@celery_app.task(bind=True, name="app.workers.tasks.process_image")
def process_image(
    self,
    store_id: str,
    product_id: str | None,
    request_data: dict[str, Any],
) -> dict[str, Any]:
    """
    Celery task for processing a single image.

    Args:
        store_id: Store ID
        product_id: Optional product ID
        request_data: Serialized ProcessImageRequest

    Returns:
        Serialized ProcessedImage result
    """
    logger.info(
        "Processing image via Celery",
        task_id=self.request.id,
        store_id=store_id,
    )

    try:
        request = ProcessImageRequest(**request_data)
        pipeline = ImageProcessingPipeline()

        # Run async pipeline in sync context
        loop = asyncio.new_event_loop()
        asyncio.set_event_loop(loop)

        try:
            result = loop.run_until_complete(
                pipeline.process(
                    request=request,
                    store_id=store_id,
                    product_id=product_id,
                )
            )
        finally:
            loop.close()

        return result.model_dump()

    except Exception as e:
        logger.error(
            "Celery task failed",
            task_id=self.request.id,
            error=str(e),
        )
        raise


@celery_app.task(bind=True, name="app.workers.tasks.process_batch")
def process_batch(
    self,
    store_id: str,
    images: list[dict[str, Any]],
    webhook_url: str | None = None,
) -> dict[str, Any]:
    """
    Celery task for batch image processing.

    Args:
        store_id: Store ID
        images: List of serialized ProcessImageRequest
        webhook_url: Optional webhook for completion notification

    Returns:
        Batch processing results
    """
    logger.info(
        "Processing batch via Celery",
        task_id=self.request.id,
        store_id=store_id,
        image_count=len(images),
    )

    results = []
    errors = []

    for i, image_data in enumerate(images):
        try:
            # Update progress
            self.update_state(
                state="PROGRESS",
                meta={"current": i + 1, "total": len(images)},
            )

            request = ProcessImageRequest(**image_data)
            pipeline = ImageProcessingPipeline()

            loop = asyncio.new_event_loop()
            asyncio.set_event_loop(loop)

            try:
                result = loop.run_until_complete(
                    pipeline.process(
                        request=request,
                        store_id=store_id,
                        product_id=None,
                    )
                )
                results.append(result.model_dump())
            finally:
                loop.close()

        except Exception as e:
            logger.error(f"Batch item {i} failed: {e}")
            errors.append({"index": i, "error": str(e)})

    # TODO: Send webhook notification if URL provided

    return {
        "total": len(images),
        "successful": len(results),
        "failed": len(errors),
        "results": results,
        "errors": errors,
    }
```

---

## Task 11: Dockerfile

**File: `services/media-studio/Dockerfile`**
```dockerfile
# Build stage
FROM python:3.11-slim as builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libgl1-mesa-glx \
    libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Production stage
FROM python:3.11-slim

WORKDIR /app

# Install runtime dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgl1-mesa-glx \
    libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*

# Copy installed packages
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

# Copy application
COPY app/ ./app/

# Create non-root user
RUN useradd --create-home --shell /bin/bash appuser
USER appuser

EXPOSE 8002

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import httpx; httpx.get('http://localhost:8002/health')" || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8002"]
```

---

## Verification Checklist

### Service Setup
- [ ] FastAPI app starts: `uvicorn app.main:app --port 8002 --reload`
- [ ] Health endpoint: `curl http://localhost:8002/health`
- [ ] API docs: `http://localhost:8002/docs`

### Processors
- [ ] Background removal works with sample image
- [ ] Watermark detection runs without errors
- [ ] Enhancement processor upscales and sharpens
- [ ] Resize processor handles all modes

### Storage
- [ ] S3/MinIO connection works
- [ ] Images upload successfully
- [ ] URLs are accessible

### Integration
- [ ] Full pipeline processes image through multiple operations
- [ ] Size variants are generated correctly
- [ ] Celery workers process tasks

---

## Next Steps

After completing Sprint 3, proceed to:

**Sprint 4: Dashboard MVP**
- Next.js 14 project setup
- Authentication with NextAuth.js
- Dashboard layout and navigation
- Store creation wizard
- Job progress tracking

Refer to `PHASE1_EXECUTION_PLAN.md` for detailed specifications.
