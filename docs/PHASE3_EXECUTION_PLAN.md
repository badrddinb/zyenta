# Phase 3: "Growth" Hub - Production Execution Plan

## Overview

This document outlines the complete execution plan for building Zyenta's Phase 3: **The Growth Hub**. This phase establishes the autonomous marketing layer that drives traffic and sales through social media and paid advertising.

### Phase 3 Objectives
- [ ] Build "Pluggable" OAuth system for external platforms
- [ ] Launch Social Agent (Organic TikTok/Instagram/Pinterest/X)
- [ ] Launch Ads Agent (Meta/Google/TikTok with Budget Management)
- [ ] Launch "Autopilot Marketing" tier

### Prerequisites
- Phase 1 (Genesis Engine) fully operational
- Phase 2 (Ops Foundation) deployed
- Media Studio generating images and videos
- Active stores with products and content

---

## 1. Growth Engine Service Architecture

### 1.1 Service Structure

```
services/growth-engine/
├── app/
│   ├── __init__.py
│   ├── main.py                    # FastAPI app entry
│   ├── config.py                  # Settings management
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── router.py
│   │   │   ├── oauth.py           # OAuth endpoints
│   │   │   ├── social.py          # Social posting endpoints
│   │   │   ├── ads.py             # Ads management endpoints
│   │   │   └── analytics.py       # Performance analytics
│   │   └── webhooks/
│   │       ├── meta.py
│   │       ├── tiktok.py
│   │       └── google.py
│   │
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── base.py                # Base marketing agent
│   │   │
│   │   ├── social/
│   │   │   ├── __init__.py
│   │   │   ├── agent.py           # Social Agent orchestrator
│   │   │   ├── content_planner.py # Content calendar generation
│   │   │   ├── caption_writer.py  # AI caption generation
│   │   │   ├── hashtag_optimizer.py
│   │   │   ├── scheduler.py       # Posting scheduler
│   │   │   └── platforms/
│   │   │       ├── __init__.py
│   │   │       ├── base.py
│   │   │       ├── tiktok.py
│   │   │       ├── instagram.py
│   │   │       ├── pinterest.py
│   │   │       └── twitter.py
│   │   │
│   │   └── ads/
│   │       ├── __init__.py
│   │       ├── agent.py           # Ads Agent orchestrator
│   │       ├── campaign_manager.py
│   │       ├── creative_generator.py
│   │       ├── budget_optimizer.py
│   │       ├── ab_tester.py
│   │       └── platforms/
│   │           ├── __init__.py
│   │           ├── base.py
│   │           ├── meta.py
│   │           ├── google.py
│   │           └── tiktok.py
│   │
│   ├── oauth/
│   │   ├── __init__.py
│   │   ├── manager.py             # OAuth flow manager
│   │   ├── token_store.py         # Secure token storage
│   │   └── providers/
│   │       ├── __init__.py
│   │       ├── base.py
│   │       ├── meta.py            # FB/IG OAuth
│   │       ├── tiktok.py
│   │       ├── google.py
│   │       ├── pinterest.py
│   │       └── twitter.py
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── connection.py          # Platform connections
│   │   ├── content.py             # Content models
│   │   ├── campaign.py            # Ad campaign models
│   │   └── analytics.py           # Analytics models
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── llm.py                 # LLM for content generation
│   │   ├── database.py
│   │   ├── queue.py
│   │   ├── media.py               # Media asset handling
│   │   └── analytics.py           # Analytics aggregation
│   │
│   └── workers/
│       ├── __init__.py
│       ├── celery_app.py
│       ├── posting_tasks.py       # Social posting workers
│       ├── ads_tasks.py           # Ads management workers
│       └── analytics_tasks.py     # Analytics sync workers
│
├── tests/
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

### 1.2 Package Configuration

```
# services/growth-engine/requirements.txt
fastapi==0.109.0
uvicorn[standard]==0.27.0
pydantic==2.5.3
pydantic-settings==2.1.0

# Database
prisma==0.12.0
asyncpg==0.29.0

# Queue & Cache
celery==5.3.4
redis==5.0.1

# OAuth & API Clients
httpx==0.26.0
authlib==1.3.0
python-jose[cryptography]==3.3.0

# Platform SDKs
facebook-business==19.0.0
google-ads==23.0.0
tiktok-business-api==0.1.0
tweepy==4.14.0
pinterest-api==0.2.0

# AI & ML
openai==1.6.1
anthropic==0.8.1

# Utilities
python-multipart==0.0.6
python-dotenv==1.0.0
cryptography==41.0.7
```

---

## 2. OAuth Integration System

### 2.1 OAuth Manager

```python
# services/growth-engine/app/oauth/manager.py
from typing import Optional, Dict, Any
from dataclasses import dataclass
from enum import Enum
import httpx
from authlib.integrations.httpx_client import AsyncOAuth2Client
from app.oauth.token_store import TokenStore
from app.config import settings


class Platform(str, Enum):
    META = "meta"           # Facebook + Instagram
    TIKTOK = "tiktok"
    GOOGLE = "google"       # Google Ads + YouTube
    PINTEREST = "pinterest"
    TWITTER = "twitter"


@dataclass
class OAuthConfig:
    client_id: str
    client_secret: str
    authorize_url: str
    token_url: str
    scopes: list[str]
    redirect_uri: str


PLATFORM_CONFIGS: Dict[Platform, OAuthConfig] = {
    Platform.META: OAuthConfig(
        client_id=settings.META_APP_ID,
        client_secret=settings.META_APP_SECRET,
        authorize_url="https://www.facebook.com/v18.0/dialog/oauth",
        token_url="https://graph.facebook.com/v18.0/oauth/access_token",
        scopes=[
            "pages_manage_posts",
            "pages_read_engagement",
            "instagram_basic",
            "instagram_content_publish",
            "instagram_manage_insights",
            "ads_management",
            "ads_read",
            "business_management",
        ],
        redirect_uri=f"{settings.API_URL}/api/v1/oauth/callback/meta",
    ),
    Platform.TIKTOK: OAuthConfig(
        client_id=settings.TIKTOK_CLIENT_KEY,
        client_secret=settings.TIKTOK_CLIENT_SECRET,
        authorize_url="https://www.tiktok.com/v2/auth/authorize/",
        token_url="https://open.tiktokapis.com/v2/oauth/token/",
        scopes=[
            "user.info.basic",
            "video.publish",
            "video.upload",
        ],
        redirect_uri=f"{settings.API_URL}/api/v1/oauth/callback/tiktok",
    ),
    Platform.GOOGLE: OAuthConfig(
        client_id=settings.GOOGLE_CLIENT_ID,
        client_secret=settings.GOOGLE_CLIENT_SECRET,
        authorize_url="https://accounts.google.com/o/oauth2/v2/auth",
        token_url="https://oauth2.googleapis.com/token",
        scopes=[
            "https://www.googleapis.com/auth/adwords",
            "https://www.googleapis.com/auth/youtube.upload",
            "https://www.googleapis.com/auth/youtube",
        ],
        redirect_uri=f"{settings.API_URL}/api/v1/oauth/callback/google",
    ),
    Platform.PINTEREST: OAuthConfig(
        client_id=settings.PINTEREST_APP_ID,
        client_secret=settings.PINTEREST_APP_SECRET,
        authorize_url="https://www.pinterest.com/oauth/",
        token_url="https://api.pinterest.com/v5/oauth/token",
        scopes=[
            "boards:read",
            "boards:write",
            "pins:read",
            "pins:write",
            "user_accounts:read",
        ],
        redirect_uri=f"{settings.API_URL}/api/v1/oauth/callback/pinterest",
    ),
    Platform.TWITTER: OAuthConfig(
        client_id=settings.TWITTER_CLIENT_ID,
        client_secret=settings.TWITTER_CLIENT_SECRET,
        authorize_url="https://twitter.com/i/oauth2/authorize",
        token_url="https://api.twitter.com/2/oauth2/token",
        scopes=[
            "tweet.read",
            "tweet.write",
            "users.read",
            "offline.access",
        ],
        redirect_uri=f"{settings.API_URL}/api/v1/oauth/callback/twitter",
    ),
}


class OAuthManager:
    """Manages OAuth flows for all supported platforms"""

    def __init__(self):
        self.token_store = TokenStore()

    async def get_authorization_url(
        self,
        platform: Platform,
        user_id: str,
        store_id: str,
    ) -> str:
        """Generate authorization URL for platform OAuth"""

        config = PLATFORM_CONFIGS[platform]

        # Generate state token for security
        state = self.token_store.generate_state_token(
            user_id=user_id,
            store_id=store_id,
            platform=platform,
        )

        client = AsyncOAuth2Client(
            client_id=config.client_id,
            client_secret=config.client_secret,
            redirect_uri=config.redirect_uri,
        )

        # Platform-specific URL generation
        if platform == Platform.TIKTOK:
            # TikTok uses different parameter names
            url = (
                f"{config.authorize_url}?"
                f"client_key={config.client_id}&"
                f"scope={','.join(config.scopes)}&"
                f"response_type=code&"
                f"redirect_uri={config.redirect_uri}&"
                f"state={state}"
            )
        else:
            url, _ = client.create_authorization_url(
                config.authorize_url,
                scope=" ".join(config.scopes),
                state=state,
            )

        return url

    async def handle_callback(
        self,
        platform: Platform,
        code: str,
        state: str,
    ) -> Dict[str, Any]:
        """Handle OAuth callback and exchange code for tokens"""

        # Verify state token
        state_data = self.token_store.verify_state_token(state)
        if not state_data:
            raise ValueError("Invalid or expired state token")

        config = PLATFORM_CONFIGS[platform]

        # Exchange code for tokens
        async with httpx.AsyncClient() as client:
            if platform == Platform.TIKTOK:
                response = await client.post(
                    config.token_url,
                    data={
                        "client_key": config.client_id,
                        "client_secret": config.client_secret,
                        "code": code,
                        "grant_type": "authorization_code",
                        "redirect_uri": config.redirect_uri,
                    },
                )
            else:
                response = await client.post(
                    config.token_url,
                    data={
                        "client_id": config.client_id,
                        "client_secret": config.client_secret,
                        "code": code,
                        "grant_type": "authorization_code",
                        "redirect_uri": config.redirect_uri,
                    },
                )

            response.raise_for_status()
            tokens = response.json()

        # Get account info
        account_info = await self._get_account_info(platform, tokens)

        # Store tokens securely
        await self.token_store.store_tokens(
            user_id=state_data["user_id"],
            store_id=state_data["store_id"],
            platform=platform,
            tokens=tokens,
            account_info=account_info,
        )

        return {
            "platform": platform,
            "account_id": account_info.get("id"),
            "account_name": account_info.get("name"),
            "connected": True,
        }

    async def _get_account_info(
        self,
        platform: Platform,
        tokens: Dict[str, Any],
    ) -> Dict[str, Any]:
        """Fetch account information after OAuth"""

        access_token = tokens.get("access_token")

        async with httpx.AsyncClient() as client:
            if platform == Platform.META:
                # Get Facebook user and pages
                response = await client.get(
                    "https://graph.facebook.com/v18.0/me",
                    params={
                        "access_token": access_token,
                        "fields": "id,name,accounts{id,name,access_token,instagram_business_account}",
                    },
                )
                data = response.json()
                return {
                    "id": data["id"],
                    "name": data["name"],
                    "pages": data.get("accounts", {}).get("data", []),
                }

            elif platform == Platform.TIKTOK:
                response = await client.get(
                    "https://open.tiktokapis.com/v2/user/info/",
                    headers={"Authorization": f"Bearer {access_token}"},
                    params={"fields": "open_id,display_name,avatar_url"},
                )
                data = response.json()
                return {
                    "id": data["data"]["user"]["open_id"],
                    "name": data["data"]["user"]["display_name"],
                }

            elif platform == Platform.GOOGLE:
                response = await client.get(
                    "https://www.googleapis.com/oauth2/v2/userinfo",
                    headers={"Authorization": f"Bearer {access_token}"},
                )
                data = response.json()
                return {
                    "id": data["id"],
                    "name": data.get("name", data.get("email")),
                }

            elif platform == Platform.PINTEREST:
                response = await client.get(
                    "https://api.pinterest.com/v5/user_account",
                    headers={"Authorization": f"Bearer {access_token}"},
                )
                data = response.json()
                return {
                    "id": data["username"],
                    "name": data.get("business_name", data["username"]),
                }

            elif platform == Platform.TWITTER:
                response = await client.get(
                    "https://api.twitter.com/2/users/me",
                    headers={"Authorization": f"Bearer {access_token}"},
                )
                data = response.json()
                return {
                    "id": data["data"]["id"],
                    "name": data["data"]["name"],
                    "username": data["data"]["username"],
                }

        return {}

    async def refresh_tokens(
        self,
        user_id: str,
        store_id: str,
        platform: Platform,
    ) -> Dict[str, Any]:
        """Refresh expired access tokens"""

        stored = await self.token_store.get_tokens(user_id, store_id, platform)
        if not stored or "refresh_token" not in stored["tokens"]:
            raise ValueError("No refresh token available")

        config = PLATFORM_CONFIGS[platform]

        async with httpx.AsyncClient() as client:
            response = await client.post(
                config.token_url,
                data={
                    "client_id": config.client_id,
                    "client_secret": config.client_secret,
                    "refresh_token": stored["tokens"]["refresh_token"],
                    "grant_type": "refresh_token",
                },
            )
            response.raise_for_status()
            new_tokens = response.json()

        # Preserve refresh token if not returned
        if "refresh_token" not in new_tokens:
            new_tokens["refresh_token"] = stored["tokens"]["refresh_token"]

        await self.token_store.store_tokens(
            user_id=user_id,
            store_id=store_id,
            platform=platform,
            tokens=new_tokens,
            account_info=stored["account_info"],
        )

        return new_tokens

    async def revoke_connection(
        self,
        user_id: str,
        store_id: str,
        platform: Platform,
    ) -> bool:
        """Revoke OAuth connection and delete stored tokens"""

        stored = await self.token_store.get_tokens(user_id, store_id, platform)
        if not stored:
            return True

        # Attempt to revoke token at platform
        # (Best effort - may not be supported by all platforms)
        try:
            await self._revoke_at_platform(platform, stored["tokens"])
        except Exception:
            pass  # Proceed with local deletion even if revocation fails

        await self.token_store.delete_tokens(user_id, store_id, platform)
        return True

    async def _revoke_at_platform(
        self,
        platform: Platform,
        tokens: Dict[str, Any],
    ):
        """Revoke token at the platform's OAuth server"""
        # Implementation varies by platform
        pass
```

### 2.2 Token Store

```python
# services/growth-engine/app/oauth/token_store.py
from typing import Optional, Dict, Any
import json
import secrets
from datetime import datetime, timedelta
from cryptography.fernet import Fernet
import redis.asyncio as redis
from app.config import settings


class TokenStore:
    """Secure storage for OAuth tokens"""

    def __init__(self):
        self.redis = redis.from_url(settings.REDIS_URL)
        self.fernet = Fernet(settings.TOKEN_ENCRYPTION_KEY.encode())

    def generate_state_token(
        self,
        user_id: str,
        store_id: str,
        platform: str,
    ) -> str:
        """Generate secure state token for OAuth flow"""

        state = secrets.token_urlsafe(32)
        data = {
            "user_id": user_id,
            "store_id": store_id,
            "platform": platform,
            "created_at": datetime.utcnow().isoformat(),
        }

        # Store with 10 minute expiry
        key = f"oauth:state:{state}"
        self.redis.setex(key, 600, json.dumps(data))

        return state

    def verify_state_token(self, state: str) -> Optional[Dict[str, Any]]:
        """Verify and consume state token"""

        key = f"oauth:state:{state}"
        data = self.redis.get(key)

        if not data:
            return None

        # Delete token (single use)
        self.redis.delete(key)

        return json.loads(data)

    async def store_tokens(
        self,
        user_id: str,
        store_id: str,
        platform: str,
        tokens: Dict[str, Any],
        account_info: Dict[str, Any],
    ):
        """Store OAuth tokens securely"""

        # Encrypt sensitive token data
        encrypted_tokens = self.fernet.encrypt(
            json.dumps(tokens).encode()
        ).decode()

        data = {
            "tokens": encrypted_tokens,
            "account_info": account_info,
            "updated_at": datetime.utcnow().isoformat(),
        }

        key = f"oauth:tokens:{user_id}:{store_id}:{platform}"
        await self.redis.set(key, json.dumps(data))

        # Also store in database for persistence
        await self._persist_to_database(
            user_id, store_id, platform, data, account_info
        )

    async def get_tokens(
        self,
        user_id: str,
        store_id: str,
        platform: str,
    ) -> Optional[Dict[str, Any]]:
        """Retrieve and decrypt stored tokens"""

        key = f"oauth:tokens:{user_id}:{store_id}:{platform}"
        data = await self.redis.get(key)

        if not data:
            # Try to load from database
            data = await self._load_from_database(user_id, store_id, platform)
            if not data:
                return None

        parsed = json.loads(data)

        # Decrypt tokens
        decrypted_tokens = json.loads(
            self.fernet.decrypt(parsed["tokens"].encode())
        )

        return {
            "tokens": decrypted_tokens,
            "account_info": parsed["account_info"],
        }

    async def delete_tokens(
        self,
        user_id: str,
        store_id: str,
        platform: str,
    ):
        """Delete stored tokens"""

        key = f"oauth:tokens:{user_id}:{store_id}:{platform}"
        await self.redis.delete(key)
        await self._delete_from_database(user_id, store_id, platform)

    async def _persist_to_database(
        self,
        user_id: str,
        store_id: str,
        platform: str,
        data: Dict[str, Any],
        account_info: Dict[str, Any],
    ):
        """Persist token data to database"""
        from app.services.database import prisma

        await prisma.platformconnection.upsert(
            where={
                "storeId_platform": {
                    "storeId": store_id,
                    "platform": platform,
                }
            },
            create={
                "storeId": store_id,
                "userId": user_id,
                "platform": platform,
                "encryptedTokens": data["tokens"],
                "accountId": account_info.get("id"),
                "accountName": account_info.get("name"),
                "status": "CONNECTED",
            },
            update={
                "encryptedTokens": data["tokens"],
                "accountId": account_info.get("id"),
                "accountName": account_info.get("name"),
                "status": "CONNECTED",
                "updatedAt": datetime.utcnow(),
            },
        )

    async def _load_from_database(
        self,
        user_id: str,
        store_id: str,
        platform: str,
    ) -> Optional[str]:
        """Load token data from database"""
        from app.services.database import prisma

        connection = await prisma.platformconnection.find_unique(
            where={
                "storeId_platform": {
                    "storeId": store_id,
                    "platform": platform,
                }
            }
        )

        if not connection:
            return None

        return json.dumps({
            "tokens": connection.encryptedTokens,
            "account_info": {
                "id": connection.accountId,
                "name": connection.accountName,
            },
        })

    async def _delete_from_database(
        self,
        user_id: str,
        store_id: str,
        platform: str,
    ):
        """Delete token data from database"""
        from app.services.database import prisma

        await prisma.platformconnection.delete(
            where={
                "storeId_platform": {
                    "storeId": store_id,
                    "platform": platform,
                }
            }
        )
```

---

## 3. Social Agent (Organic Marketing)

### 3.1 Social Agent Orchestrator

```python
# services/growth-engine/app/agents/social/agent.py
from typing import List, Optional
from dataclasses import dataclass
from datetime import datetime, timedelta
from app.agents.base import BaseMarketingAgent
from app.agents.social.content_planner import ContentPlanner
from app.agents.social.caption_writer import CaptionWriter
from app.agents.social.hashtag_optimizer import HashtagOptimizer
from app.agents.social.scheduler import PostingScheduler
from app.agents.social.platforms import get_platform_client
from app.oauth.token_store import TokenStore


@dataclass
class SocialPost:
    id: str
    platform: str
    content_type: str  # image, video, carousel
    caption: str
    hashtags: List[str]
    media_urls: List[str]
    scheduled_time: datetime
    status: str  # draft, scheduled, published, failed


@dataclass
class ContentCalendar:
    store_id: str
    start_date: datetime
    end_date: datetime
    posts: List[SocialPost]
    platforms: List[str]


class SocialAgent(BaseMarketingAgent):
    """
    Autonomous Social Media Marketing Agent.
    Handles content planning, creation, and posting across platforms.
    """

    def __init__(self):
        super().__init__()
        self.content_planner = ContentPlanner()
        self.caption_writer = CaptionWriter()
        self.hashtag_optimizer = HashtagOptimizer()
        self.scheduler = PostingScheduler()
        self.token_store = TokenStore()

    async def generate_content_calendar(
        self,
        store_id: str,
        days: int = 30,
        posts_per_day: int = 2,
    ) -> ContentCalendar:
        """Generate a content calendar for the store"""

        # Get store data
        store = await self.get_store_data(store_id)
        products = await self.get_store_products(store_id)
        media_assets = await self.get_store_media(store_id)
        connected_platforms = await self.get_connected_platforms(store_id)

        if not connected_platforms:
            raise ValueError("No social platforms connected")

        # Generate content plan using AI
        content_plan = await self.content_planner.create_plan(
            niche=store.niche,
            brand_voice=store.brandIdentity.get("tone", "professional"),
            products=products,
            days=days,
            posts_per_day=posts_per_day,
        )

        # Create posts for each planned content
        posts = []
        for planned_content in content_plan:
            post = await self._create_post(
                store=store,
                planned_content=planned_content,
                media_assets=media_assets,
                platforms=connected_platforms,
            )
            posts.append(post)

        # Schedule posts with optimal timing
        scheduled_posts = await self.scheduler.schedule_posts(
            posts=posts,
            platforms=connected_platforms,
            timezone=store.settings.get("timezone", "UTC"),
        )

        calendar = ContentCalendar(
            store_id=store_id,
            start_date=datetime.utcnow(),
            end_date=datetime.utcnow() + timedelta(days=days),
            posts=scheduled_posts,
            platforms=connected_platforms,
        )

        # Save calendar to database
        await self._save_calendar(calendar)

        return calendar

    async def _create_post(
        self,
        store: any,
        planned_content: dict,
        media_assets: List[dict],
        platforms: List[str],
    ) -> SocialPost:
        """Create a social post from planned content"""

        # Select appropriate media
        media_urls = self._select_media(
            planned_content["content_type"],
            planned_content.get("product_id"),
            media_assets,
        )

        # Generate platform-specific captions
        captions = {}
        for platform in platforms:
            caption = await self.caption_writer.write(
                content_type=planned_content["content_type"],
                product=planned_content.get("product"),
                brand_voice=store.brandIdentity.get("tone"),
                platform=platform,
                call_to_action=planned_content.get("cta"),
            )

            # Optimize hashtags for platform
            hashtags = await self.hashtag_optimizer.optimize(
                niche=store.niche,
                caption=caption,
                platform=platform,
            )

            captions[platform] = {
                "text": caption,
                "hashtags": hashtags,
            }

        return SocialPost(
            id=self.generate_id(),
            platform=platforms[0],  # Primary platform
            content_type=planned_content["content_type"],
            caption=captions[platforms[0]]["text"],
            hashtags=captions[platforms[0]]["hashtags"],
            media_urls=media_urls,
            scheduled_time=None,  # Set by scheduler
            status="draft",
        )

    async def publish_post(self, post_id: str) -> dict:
        """Publish a scheduled post to its platform"""

        post = await self._get_post(post_id)
        if not post:
            raise ValueError(f"Post not found: {post_id}")

        # Get platform client with valid tokens
        client = await self._get_platform_client(post.store_id, post.platform)

        try:
            # Upload media first
            media_ids = []
            for media_url in post.media_urls:
                media_id = await client.upload_media(media_url)
                media_ids.append(media_id)

            # Create post
            result = await client.create_post(
                caption=post.caption,
                hashtags=post.hashtags,
                media_ids=media_ids,
            )

            # Update post status
            await self._update_post_status(
                post_id=post_id,
                status="published",
                platform_post_id=result["id"],
                published_at=datetime.utcnow(),
            )

            return {
                "success": True,
                "platform_post_id": result["id"],
                "url": result.get("url"),
            }

        except Exception as e:
            await self._update_post_status(
                post_id=post_id,
                status="failed",
                error=str(e),
            )
            raise

    async def get_analytics(
        self,
        store_id: str,
        platform: str,
        days: int = 30,
    ) -> dict:
        """Get analytics for social media performance"""

        client = await self._get_platform_client(store_id, platform)

        insights = await client.get_insights(
            start_date=datetime.utcnow() - timedelta(days=days),
            end_date=datetime.utcnow(),
        )

        posts = await self._get_published_posts(store_id, platform, days)

        return {
            "overview": {
                "followers": insights.get("followers"),
                "follower_growth": insights.get("follower_growth"),
                "total_reach": insights.get("reach"),
                "total_impressions": insights.get("impressions"),
                "total_engagement": insights.get("engagement"),
                "engagement_rate": insights.get("engagement_rate"),
            },
            "posts": [
                {
                    "id": post.id,
                    "published_at": post.published_at,
                    "likes": post.analytics.get("likes"),
                    "comments": post.analytics.get("comments"),
                    "shares": post.analytics.get("shares"),
                    "reach": post.analytics.get("reach"),
                    "engagement_rate": post.analytics.get("engagement_rate"),
                }
                for post in posts
            ],
            "top_performing": self._get_top_posts(posts, 5),
        }

    async def _get_platform_client(self, store_id: str, platform: str):
        """Get authenticated platform client"""

        # Get connection
        from app.services.database import prisma

        connection = await prisma.platformconnection.find_unique(
            where={
                "storeId_platform": {
                    "storeId": store_id,
                    "platform": platform,
                }
            }
        )

        if not connection:
            raise ValueError(f"Platform not connected: {platform}")

        # Get tokens
        tokens = await self.token_store.get_tokens(
            connection.userId,
            store_id,
            platform,
        )

        # Check if refresh needed
        if self._tokens_expired(tokens["tokens"]):
            from app.oauth.manager import OAuthManager
            oauth = OAuthManager()
            tokens = await oauth.refresh_tokens(
                connection.userId,
                store_id,
                platform,
            )

        return get_platform_client(platform, tokens["tokens"])

    def _select_media(
        self,
        content_type: str,
        product_id: Optional[str],
        media_assets: List[dict],
    ) -> List[str]:
        """Select appropriate media for the post"""

        if product_id:
            # Get product-specific media
            product_media = [
                m for m in media_assets
                if m.get("productId") == product_id
            ]
            if product_media:
                if content_type == "video":
                    videos = [m for m in product_media if m["type"] == "VIDEO"]
                    return [videos[0]["url"]] if videos else []
                else:
                    images = [m for m in product_media if m["type"] == "IMAGE"]
                    return [images[0]["url"]] if images else []

        # Fallback to general store media
        if content_type == "video":
            videos = [m for m in media_assets if m["type"] == "VIDEO"]
            return [videos[0]["url"]] if videos else []
        else:
            images = [m for m in media_assets if m["type"] == "IMAGE"]
            return [images[0]["url"]] if images else []
```

### 3.2 Content Planner

```python
# services/growth-engine/app/agents/social/content_planner.py
from typing import List
from datetime import datetime, timedelta
from app.services.llm import LLMService


class ContentPlanner:
    """AI-powered content planning for social media"""

    def __init__(self):
        self.llm = LLMService()

    async def create_plan(
        self,
        niche: str,
        brand_voice: str,
        products: List[dict],
        days: int = 30,
        posts_per_day: int = 2,
    ) -> List[dict]:
        """Create a content plan using AI"""

        total_posts = days * posts_per_day

        # Generate content themes
        themes = await self._generate_themes(niche, brand_voice, products)

        # Create content calendar
        content_plan = []
        product_index = 0
        theme_index = 0

        for day in range(days):
            for post_num in range(posts_per_day):
                content_type = self._get_content_type(day, post_num)
                theme = themes[theme_index % len(themes)]

                # Rotate between product-focused and brand-focused content
                if post_num == 0 and products:
                    # Product showcase post
                    product = products[product_index % len(products)]
                    content = await self._plan_product_post(
                        product=product,
                        theme=theme,
                        brand_voice=brand_voice,
                    )
                    product_index += 1
                else:
                    # Brand/lifestyle post
                    content = await self._plan_brand_post(
                        niche=niche,
                        theme=theme,
                        brand_voice=brand_voice,
                    )

                content["day"] = day
                content["post_number"] = post_num
                content["content_type"] = content_type
                content_plan.append(content)

                theme_index += 1

        return content_plan

    async def _generate_themes(
        self,
        niche: str,
        brand_voice: str,
        products: List[dict],
    ) -> List[str]:
        """Generate content themes using AI"""

        prompt = f"""
        Generate 10 engaging content themes for a {niche} brand with a {brand_voice} voice.

        The brand sells products like:
        {[p['title'] for p in products[:5]]}

        Create themes that:
        1. Showcase products naturally
        2. Provide value to the audience
        3. Build brand awareness
        4. Encourage engagement
        5. Mix promotional and educational content

        Return a JSON array of theme objects with:
        - name: Theme name
        - description: Brief description
        - content_angle: How to approach the content
        - best_for: Type of content (image, video, carousel)
        """

        response = await self.llm.generate(
            prompt=prompt,
            response_format={"type": "json_object"},
        )

        return response["themes"]

    async def _plan_product_post(
        self,
        product: dict,
        theme: dict,
        brand_voice: str,
    ) -> dict:
        """Plan a product-focused post"""

        prompt = f"""
        Create a social media post plan for this product:

        Product: {product['title']}
        Description: {product['description'][:200]}
        Price: ${product['sellingPrice']}

        Theme: {theme['name']}
        Brand Voice: {brand_voice}

        Return a JSON object with:
        - headline: Attention-grabbing headline
        - key_points: 3 key points to highlight
        - cta: Call to action
        - hook: Opening hook for the caption
        - emotion: Primary emotion to evoke
        """

        response = await self.llm.generate(
            prompt=prompt,
            response_format={"type": "json_object"},
        )

        return {
            "type": "product",
            "product_id": product["id"],
            "product": product,
            "theme": theme,
            **response,
        }

    async def _plan_brand_post(
        self,
        niche: str,
        theme: dict,
        brand_voice: str,
    ) -> dict:
        """Plan a brand/lifestyle post"""

        prompt = f"""
        Create a social media post plan for brand awareness:

        Niche: {niche}
        Theme: {theme['name']}
        Brand Voice: {brand_voice}

        This should NOT be a direct product post.
        Focus on: lifestyle, education, entertainment, or community.

        Return a JSON object with:
        - headline: Attention-grabbing headline
        - content_idea: What the post is about
        - cta: Call to action (engagement focused)
        - hook: Opening hook for the caption
        - emotion: Primary emotion to evoke
        """

        response = await self.llm.generate(
            prompt=prompt,
            response_format={"type": "json_object"},
        )

        return {
            "type": "brand",
            "theme": theme,
            **response,
        }

    def _get_content_type(self, day: int, post_num: int) -> str:
        """Determine content type based on schedule"""
        # Mix of content types throughout the week
        if day % 7 in [0, 3]:  # Sunday, Wednesday
            return "video"
        elif day % 7 in [2, 5]:  # Tuesday, Friday
            return "carousel"
        else:
            return "image"
```

### 3.3 Platform Client (TikTok Example)

```python
# services/growth-engine/app/agents/social/platforms/tiktok.py
from typing import List, Optional, Dict, Any
from datetime import datetime
import httpx
from .base import BasePlatformClient


class TikTokClient(BasePlatformClient):
    """TikTok API client for content publishing"""

    BASE_URL = "https://open.tiktokapis.com/v2"

    def __init__(self, tokens: Dict[str, Any]):
        self.access_token = tokens["access_token"]
        self.open_id = tokens.get("open_id")

    async def _request(
        self,
        method: str,
        endpoint: str,
        **kwargs,
    ) -> Dict[str, Any]:
        """Make authenticated API request"""

        headers = {
            "Authorization": f"Bearer {self.access_token}",
            "Content-Type": "application/json",
        }

        async with httpx.AsyncClient() as client:
            response = await client.request(
                method,
                f"{self.BASE_URL}{endpoint}",
                headers=headers,
                **kwargs,
            )
            response.raise_for_status()
            return response.json()

    async def upload_media(self, media_url: str) -> str:
        """
        Upload video to TikTok.
        TikTok requires videos to be uploaded before creating a post.
        """

        # Initialize upload
        init_response = await self._request(
            "POST",
            "/post/publish/video/init/",
            json={
                "post_info": {
                    "title": "Video upload",
                    "privacy_level": "SELF_ONLY",  # Draft mode
                },
                "source_info": {
                    "source": "PULL_FROM_URL",
                    "video_url": media_url,
                },
            },
        )

        publish_id = init_response["data"]["publish_id"]

        # Poll for upload completion
        max_attempts = 30
        for _ in range(max_attempts):
            status = await self._request(
                "POST",
                "/post/publish/status/fetch/",
                json={"publish_id": publish_id},
            )

            if status["data"]["status"] == "PUBLISH_COMPLETE":
                return publish_id

            if status["data"]["status"] == "FAILED":
                raise Exception(f"Upload failed: {status['data'].get('fail_reason')}")

            await asyncio.sleep(2)

        raise Exception("Upload timeout")

    async def create_post(
        self,
        caption: str,
        hashtags: List[str],
        media_ids: List[str],
    ) -> Dict[str, Any]:
        """
        Publish a video post to TikTok.

        Note: TikTok's API publishes videos that were uploaded as drafts.
        """

        # Combine caption and hashtags
        full_caption = f"{caption}\n\n{' '.join(['#' + h for h in hashtags])}"

        # TikTok requires the user to complete publishing through their app
        # We can only upload as a draft with SELF_ONLY privacy

        response = await self._request(
            "POST",
            "/post/publish/video/init/",
            json={
                "post_info": {
                    "title": full_caption[:150],  # TikTok title limit
                    "privacy_level": "PUBLIC_TO_EVERYONE",
                    "disable_duet": False,
                    "disable_comment": False,
                    "disable_stitch": False,
                },
                "source_info": {
                    "source": "FILE_UPLOAD",
                    "video_size": 0,  # Will be set during upload
                },
            },
        )

        return {
            "id": response["data"]["publish_id"],
            "status": "published",
        }

    async def get_insights(
        self,
        start_date: datetime,
        end_date: datetime,
    ) -> Dict[str, Any]:
        """Get account insights"""

        # TikTok Creator API for insights
        response = await self._request(
            "POST",
            "/user/info/",
            params={
                "fields": "follower_count,following_count,likes_count,video_count",
            },
        )

        user_data = response["data"]["user"]

        return {
            "followers": user_data.get("follower_count", 0),
            "following": user_data.get("following_count", 0),
            "likes": user_data.get("likes_count", 0),
            "videos": user_data.get("video_count", 0),
        }

    async def get_post_analytics(self, post_id: str) -> Dict[str, Any]:
        """Get analytics for a specific post"""

        response = await self._request(
            "POST",
            "/video/query/",
            json={
                "filters": {
                    "video_ids": [post_id],
                },
                "fields": [
                    "id",
                    "title",
                    "view_count",
                    "like_count",
                    "comment_count",
                    "share_count",
                ],
            },
        )

        if not response["data"]["videos"]:
            return {}

        video = response["data"]["videos"][0]
        return {
            "views": video.get("view_count", 0),
            "likes": video.get("like_count", 0),
            "comments": video.get("comment_count", 0),
            "shares": video.get("share_count", 0),
        }
```

---

## 4. Ads Agent (Paid Marketing)

### 4.1 Ads Agent Orchestrator

```python
# services/growth-engine/app/agents/ads/agent.py
from typing import List, Optional, Dict, Any
from dataclasses import dataclass
from datetime import datetime, timedelta
from decimal import Decimal
from enum import Enum
from app.agents.base import BaseMarketingAgent
from app.agents.ads.campaign_manager import CampaignManager
from app.agents.ads.creative_generator import CreativeGenerator
from app.agents.ads.budget_optimizer import BudgetOptimizer
from app.agents.ads.ab_tester import ABTester


class CampaignObjective(str, Enum):
    AWARENESS = "awareness"
    TRAFFIC = "traffic"
    ENGAGEMENT = "engagement"
    CONVERSIONS = "conversions"
    CATALOG_SALES = "catalog_sales"


class CampaignStatus(str, Enum):
    DRAFT = "draft"
    PENDING_REVIEW = "pending_review"
    ACTIVE = "active"
    PAUSED = "paused"
    COMPLETED = "completed"
    FAILED = "failed"


@dataclass
class Campaign:
    id: str
    store_id: str
    platform: str
    name: str
    objective: CampaignObjective
    status: CampaignStatus
    daily_budget: Decimal
    total_budget: Optional[Decimal]
    start_date: datetime
    end_date: Optional[datetime]
    targeting: Dict[str, Any]
    creatives: List[Dict[str, Any]]
    performance: Dict[str, Any]


class AdsAgent(BaseMarketingAgent):
    """
    Autonomous Paid Advertising Agent.
    Manages ad campaigns across Meta, Google, and TikTok Ads.
    """

    def __init__(self):
        super().__init__()
        self.campaign_manager = CampaignManager()
        self.creative_generator = CreativeGenerator()
        self.budget_optimizer = BudgetOptimizer()
        self.ab_tester = ABTester()

    async def create_campaign(
        self,
        store_id: str,
        platform: str,
        objective: CampaignObjective,
        daily_budget: Decimal,
        duration_days: int = 7,
        auto_optimize: bool = True,
    ) -> Campaign:
        """Create and launch a new ad campaign"""

        # Get store and product data
        store = await self.get_store_data(store_id)
        products = await self.get_store_products(store_id, limit=10)
        media_assets = await self.get_store_media(store_id)

        # Generate targeting based on niche
        targeting = await self._generate_targeting(
            niche=store.niche,
            platform=platform,
            objective=objective,
        )

        # Generate ad creatives
        creatives = await self.creative_generator.generate(
            store=store,
            products=products,
            media_assets=media_assets,
            platform=platform,
            objective=objective,
            variations=3,  # A/B test variations
        )

        # Create campaign structure
        campaign = Campaign(
            id=self.generate_id(),
            store_id=store_id,
            platform=platform,
            name=f"{store.name} - {objective.value} - {datetime.now().strftime('%Y%m%d')}",
            objective=objective,
            status=CampaignStatus.DRAFT,
            daily_budget=daily_budget,
            total_budget=daily_budget * duration_days if duration_days else None,
            start_date=datetime.utcnow(),
            end_date=datetime.utcnow() + timedelta(days=duration_days) if duration_days else None,
            targeting=targeting,
            creatives=creatives,
            performance={},
        )

        # Launch campaign on platform
        result = await self.campaign_manager.launch(
            campaign=campaign,
            store_id=store_id,
        )

        campaign.status = CampaignStatus.PENDING_REVIEW

        # Save campaign
        await self._save_campaign(campaign)

        # Schedule optimization tasks if enabled
        if auto_optimize:
            await self._schedule_optimization(campaign.id)

        return campaign

    async def _generate_targeting(
        self,
        niche: str,
        platform: str,
        objective: CampaignObjective,
    ) -> Dict[str, Any]:
        """Generate targeting parameters using AI"""

        prompt = f"""
        Generate targeting parameters for a {platform} ad campaign.

        Niche: {niche}
        Objective: {objective.value}

        Return targeting as JSON with platform-appropriate fields:
        - demographics (age, gender, locations)
        - interests (relevant interests for the niche)
        - behaviors (shopping behaviors, device usage)
        - custom_audiences (lookalike, retargeting suggestions)
        """

        response = await self.llm.generate(
            prompt=prompt,
            response_format={"type": "json_object"},
        )

        # Translate to platform-specific format
        if platform == "meta":
            return self._format_meta_targeting(response)
        elif platform == "google":
            return self._format_google_targeting(response)
        elif platform == "tiktok":
            return self._format_tiktok_targeting(response)

        return response

    async def optimize_campaign(self, campaign_id: str):
        """Run optimization routine on a campaign"""

        campaign = await self._get_campaign(campaign_id)
        if not campaign or campaign.status != CampaignStatus.ACTIVE:
            return

        # Get current performance
        performance = await self.campaign_manager.get_performance(
            campaign_id=campaign_id,
            platform=campaign.platform,
        )

        # Run A/B test analysis
        ab_results = await self.ab_tester.analyze(
            campaign_id=campaign_id,
            creatives=campaign.creatives,
            performance=performance,
        )

        # Pause underperforming creatives
        for creative_id in ab_results["pause"]:
            await self.campaign_manager.pause_creative(
                campaign_id=campaign_id,
                creative_id=creative_id,
            )

        # Scale winning creatives
        for creative_id in ab_results["scale"]:
            await self.campaign_manager.increase_budget(
                campaign_id=campaign_id,
                creative_id=creative_id,
                increase_percent=20,
            )

        # Optimize budget allocation
        budget_recommendations = await self.budget_optimizer.optimize(
            campaign=campaign,
            performance=performance,
        )

        if budget_recommendations["action"] == "increase":
            await self.campaign_manager.adjust_budget(
                campaign_id=campaign_id,
                new_budget=budget_recommendations["new_budget"],
            )
        elif budget_recommendations["action"] == "pause":
            await self.campaign_manager.pause_campaign(campaign_id)

        # Log optimization
        await self._log_optimization(
            campaign_id=campaign_id,
            ab_results=ab_results,
            budget_recommendations=budget_recommendations,
        )

    async def get_campaign_analytics(
        self,
        store_id: str,
        days: int = 30,
    ) -> Dict[str, Any]:
        """Get comprehensive ads analytics for a store"""

        campaigns = await self._get_store_campaigns(store_id, days)

        total_spend = Decimal("0")
        total_revenue = Decimal("0")
        total_impressions = 0
        total_clicks = 0
        total_conversions = 0

        campaign_metrics = []

        for campaign in campaigns:
            performance = campaign.performance or {}

            spend = Decimal(str(performance.get("spend", 0)))
            revenue = Decimal(str(performance.get("revenue", 0)))

            total_spend += spend
            total_revenue += revenue
            total_impressions += performance.get("impressions", 0)
            total_clicks += performance.get("clicks", 0)
            total_conversions += performance.get("conversions", 0)

            roas = revenue / spend if spend > 0 else Decimal("0")

            campaign_metrics.append({
                "id": campaign.id,
                "name": campaign.name,
                "platform": campaign.platform,
                "status": campaign.status,
                "spend": float(spend),
                "revenue": float(revenue),
                "roas": float(roas),
                "impressions": performance.get("impressions", 0),
                "clicks": performance.get("clicks", 0),
                "conversions": performance.get("conversions", 0),
                "cpc": performance.get("cpc", 0),
                "cpm": performance.get("cpm", 0),
                "ctr": performance.get("ctr", 0),
            })

        overall_roas = total_revenue / total_spend if total_spend > 0 else Decimal("0")
        overall_ctr = (total_clicks / total_impressions * 100) if total_impressions > 0 else 0

        return {
            "overview": {
                "total_spend": float(total_spend),
                "total_revenue": float(total_revenue),
                "roas": float(overall_roas),
                "impressions": total_impressions,
                "clicks": total_clicks,
                "conversions": total_conversions,
                "ctr": overall_ctr,
            },
            "campaigns": campaign_metrics,
            "by_platform": self._aggregate_by_platform(campaign_metrics),
            "recommendations": await self._generate_recommendations(
                campaign_metrics, overall_roas
            ),
        }

    async def _generate_recommendations(
        self,
        campaign_metrics: List[Dict],
        overall_roas: Decimal,
    ) -> List[str]:
        """Generate AI-powered recommendations"""

        prompt = f"""
        Analyze these ad campaign metrics and provide actionable recommendations:

        Overall ROAS: {overall_roas}
        Campaigns: {campaign_metrics}

        Provide 3-5 specific recommendations to improve performance.
        Focus on: budget allocation, targeting improvements, creative suggestions.
        """

        response = await self.llm.generate(prompt=prompt)
        return response.get("recommendations", [])

    def _format_meta_targeting(self, targeting: Dict) -> Dict[str, Any]:
        """Format targeting for Meta Ads API"""
        return {
            "geo_locations": {
                "countries": targeting.get("locations", ["US"]),
            },
            "age_min": targeting.get("demographics", {}).get("age_min", 18),
            "age_max": targeting.get("demographics", {}).get("age_max", 65),
            "genders": targeting.get("demographics", {}).get("genders", [1, 2]),
            "interests": [
                {"id": interest["id"], "name": interest["name"]}
                for interest in targeting.get("interests", [])
            ],
            "behaviors": targeting.get("behaviors", []),
        }

    def _format_google_targeting(self, targeting: Dict) -> Dict[str, Any]:
        """Format targeting for Google Ads API"""
        return {
            "location_targets": targeting.get("locations", ["US"]),
            "age_range": targeting.get("demographics", {}).get("age_range"),
            "gender": targeting.get("demographics", {}).get("gender"),
            "affinity_audiences": targeting.get("interests", []),
            "in_market_audiences": targeting.get("behaviors", []),
        }

    def _format_tiktok_targeting(self, targeting: Dict) -> Dict[str, Any]:
        """Format targeting for TikTok Ads API"""
        return {
            "location_ids": targeting.get("locations", []),
            "age_groups": targeting.get("demographics", {}).get("age_groups", []),
            "gender": targeting.get("demographics", {}).get("gender", "GENDER_UNLIMITED"),
            "interest_category_ids": targeting.get("interests", []),
            "operating_systems": ["ANDROID", "IOS"],
        }
```

### 4.2 Budget Optimizer

```python
# services/growth-engine/app/agents/ads/budget_optimizer.py
from typing import Dict, Any, Optional
from decimal import Decimal
from dataclasses import dataclass
from enum import Enum


class BudgetAction(str, Enum):
    INCREASE = "increase"
    DECREASE = "decrease"
    MAINTAIN = "maintain"
    PAUSE = "pause"


@dataclass
class OptimizationResult:
    action: BudgetAction
    new_budget: Optional[Decimal]
    reason: str
    confidence: float


class BudgetOptimizer:
    """
    AI-powered budget optimization for ad campaigns.
    Uses ROAS, CPA, and trend analysis to make decisions.
    """

    # Thresholds for optimization decisions
    ROAS_EXCELLENT = Decimal("3.0")
    ROAS_GOOD = Decimal("2.0")
    ROAS_ACCEPTABLE = Decimal("1.0")
    ROAS_POOR = Decimal("0.5")

    MAX_DAILY_BUDGET_INCREASE = Decimal("0.20")  # 20%
    MIN_LEARNING_SPEND = Decimal("50.0")  # Minimum spend before decisions

    def __init__(self):
        pass

    async def optimize(
        self,
        campaign: Any,
        performance: Dict[str, Any],
    ) -> OptimizationResult:
        """Determine budget optimization action"""

        spend = Decimal(str(performance.get("spend", 0)))
        revenue = Decimal(str(performance.get("revenue", 0)))
        conversions = performance.get("conversions", 0)
        days_running = performance.get("days_running", 0)

        # Calculate metrics
        roas = revenue / spend if spend > 0 else Decimal("0")
        cpa = spend / conversions if conversions > 0 else Decimal("999")

        # Check if still in learning phase
        if spend < self.MIN_LEARNING_SPEND or days_running < 3:
            return OptimizationResult(
                action=BudgetAction.MAINTAIN,
                new_budget=None,
                reason="Campaign still in learning phase",
                confidence=0.5,
            )

        # Excellent performance - scale up
        if roas >= self.ROAS_EXCELLENT:
            increase = min(
                campaign.daily_budget * self.MAX_DAILY_BUDGET_INCREASE,
                Decimal("100.0"),  # Max $100 increase
            )
            return OptimizationResult(
                action=BudgetAction.INCREASE,
                new_budget=campaign.daily_budget + increase,
                reason=f"Excellent ROAS of {roas:.2f}x - scaling budget",
                confidence=0.9,
            )

        # Good performance - moderate increase
        if roas >= self.ROAS_GOOD:
            increase = campaign.daily_budget * Decimal("0.10")
            return OptimizationResult(
                action=BudgetAction.INCREASE,
                new_budget=campaign.daily_budget + increase,
                reason=f"Good ROAS of {roas:.2f}x - gradual scaling",
                confidence=0.75,
            )

        # Acceptable performance - maintain
        if roas >= self.ROAS_ACCEPTABLE:
            return OptimizationResult(
                action=BudgetAction.MAINTAIN,
                new_budget=None,
                reason=f"Acceptable ROAS of {roas:.2f}x - maintaining budget",
                confidence=0.7,
            )

        # Poor performance - reduce or pause
        if roas < self.ROAS_POOR:
            return OptimizationResult(
                action=BudgetAction.PAUSE,
                new_budget=Decimal("0"),
                reason=f"Poor ROAS of {roas:.2f}x - pausing campaign",
                confidence=0.85,
            )

        # Suboptimal - reduce budget
        decrease = campaign.daily_budget * Decimal("0.20")
        return OptimizationResult(
            action=BudgetAction.DECREASE,
            new_budget=campaign.daily_budget - decrease,
            reason=f"Suboptimal ROAS of {roas:.2f}x - reducing budget",
            confidence=0.7,
        )

    async def allocate_budget_across_campaigns(
        self,
        total_budget: Decimal,
        campaigns: list,
    ) -> Dict[str, Decimal]:
        """Allocate budget across multiple campaigns based on performance"""

        allocations = {}
        total_weight = Decimal("0")

        # Calculate performance weights
        for campaign in campaigns:
            if campaign.status != "active":
                allocations[campaign.id] = Decimal("0")
                continue

            perf = campaign.performance or {}
            roas = Decimal(str(perf.get("roas", 1.0)))

            # Weight by ROAS (higher ROAS = more budget)
            weight = max(roas, Decimal("0.1"))
            campaign._temp_weight = weight
            total_weight += weight

        # Allocate based on weights
        for campaign in campaigns:
            if campaign.status != "active":
                continue

            weight = getattr(campaign, "_temp_weight", Decimal("1"))
            allocation = (weight / total_weight) * total_budget

            # Apply min/max constraints
            allocation = max(allocation, Decimal("10.0"))  # Min $10/day
            allocations[campaign.id] = allocation

        return allocations

    async def forecast_performance(
        self,
        campaign: Any,
        new_budget: Decimal,
        days: int = 7,
    ) -> Dict[str, Any]:
        """Forecast performance at new budget level"""

        current_budget = campaign.daily_budget
        perf = campaign.performance or {}

        # Simple linear scaling (could be improved with ML)
        scale_factor = float(new_budget / current_budget) if current_budget > 0 else 1

        # Diminishing returns after 2x budget
        if scale_factor > 2:
            scale_factor = 2 + (scale_factor - 2) * 0.5

        return {
            "forecasted_spend": float(new_budget * days),
            "forecasted_conversions": int(perf.get("conversions", 0) * scale_factor * (days / 7)),
            "forecasted_revenue": float(Decimal(str(perf.get("revenue", 0))) * Decimal(str(scale_factor)) * (Decimal(str(days)) / Decimal("7"))),
            "confidence": 0.7 if scale_factor < 1.5 else 0.5,
        }
```

### 4.3 Platform Client (Meta Ads Example)

```python
# services/growth-engine/app/agents/ads/platforms/meta.py
from typing import Dict, Any, List, Optional
from datetime import datetime
import httpx
from facebook_business.api import FacebookAdsApi
from facebook_business.adobjects.adaccount import AdAccount
from facebook_business.adobjects.campaign import Campaign
from facebook_business.adobjects.adset import AdSet
from facebook_business.adobjects.ad import Ad
from facebook_business.adobjects.adcreative import AdCreative


class MetaAdsClient:
    """Meta (Facebook/Instagram) Ads API client"""

    def __init__(self, tokens: Dict[str, Any], ad_account_id: str):
        self.access_token = tokens["access_token"]
        self.ad_account_id = ad_account_id

        # Initialize Facebook Business SDK
        FacebookAdsApi.init(access_token=self.access_token)
        self.account = AdAccount(f"act_{ad_account_id}")

    async def create_campaign(
        self,
        name: str,
        objective: str,
        daily_budget: int,  # in cents
        status: str = "PAUSED",
    ) -> Dict[str, Any]:
        """Create a new campaign"""

        campaign = Campaign(parent_id=self.account.get_id())
        campaign.update({
            Campaign.Field.name: name,
            Campaign.Field.objective: self._map_objective(objective),
            Campaign.Field.status: status,
            Campaign.Field.special_ad_categories: [],
        })

        campaign.remote_create()

        return {
            "id": campaign.get_id(),
            "name": name,
            "status": status,
        }

    async def create_ad_set(
        self,
        campaign_id: str,
        name: str,
        daily_budget: int,
        targeting: Dict[str, Any],
        optimization_goal: str,
        billing_event: str = "IMPRESSIONS",
        start_time: Optional[datetime] = None,
        end_time: Optional[datetime] = None,
    ) -> Dict[str, Any]:
        """Create an ad set within a campaign"""

        ad_set = AdSet(parent_id=self.account.get_id())
        ad_set.update({
            AdSet.Field.name: name,
            AdSet.Field.campaign_id: campaign_id,
            AdSet.Field.daily_budget: daily_budget,
            AdSet.Field.billing_event: billing_event,
            AdSet.Field.optimization_goal: optimization_goal,
            AdSet.Field.targeting: targeting,
            AdSet.Field.status: "PAUSED",
        })

        if start_time:
            ad_set.update({AdSet.Field.start_time: start_time.isoformat()})
        if end_time:
            ad_set.update({AdSet.Field.end_time: end_time.isoformat()})

        ad_set.remote_create()

        return {
            "id": ad_set.get_id(),
            "name": name,
        }

    async def create_ad_creative(
        self,
        name: str,
        page_id: str,
        image_url: Optional[str] = None,
        video_id: Optional[str] = None,
        headline: str = "",
        description: str = "",
        call_to_action: str = "SHOP_NOW",
        link_url: str = "",
    ) -> Dict[str, Any]:
        """Create an ad creative"""

        creative_data = {
            AdCreative.Field.name: name,
            AdCreative.Field.object_story_spec: {
                "page_id": page_id,
                "link_data": {
                    "link": link_url,
                    "message": description,
                    "name": headline,
                    "call_to_action": {
                        "type": call_to_action,
                        "value": {"link": link_url},
                    },
                },
            },
        }

        if image_url:
            # Upload image first
            image_hash = await self._upload_image(image_url)
            creative_data[AdCreative.Field.object_story_spec]["link_data"]["image_hash"] = image_hash
        elif video_id:
            creative_data[AdCreative.Field.object_story_spec]["video_data"] = {
                "video_id": video_id,
                "call_to_action": {
                    "type": call_to_action,
                    "value": {"link": link_url},
                },
            }

        creative = AdCreative(parent_id=self.account.get_id())
        creative.update(creative_data)
        creative.remote_create()

        return {
            "id": creative.get_id(),
            "name": name,
        }

    async def create_ad(
        self,
        name: str,
        ad_set_id: str,
        creative_id: str,
        status: str = "PAUSED",
    ) -> Dict[str, Any]:
        """Create an ad"""

        ad = Ad(parent_id=self.account.get_id())
        ad.update({
            Ad.Field.name: name,
            Ad.Field.adset_id: ad_set_id,
            Ad.Field.creative: {"creative_id": creative_id},
            Ad.Field.status: status,
        })

        ad.remote_create()

        return {
            "id": ad.get_id(),
            "name": name,
        }

    async def get_campaign_insights(
        self,
        campaign_id: str,
        start_date: datetime,
        end_date: datetime,
    ) -> Dict[str, Any]:
        """Get campaign performance insights"""

        campaign = Campaign(campaign_id)
        insights = campaign.get_insights(
            fields=[
                "spend",
                "impressions",
                "clicks",
                "reach",
                "actions",
                "action_values",
                "cpc",
                "cpm",
                "ctr",
            ],
            params={
                "time_range": {
                    "since": start_date.strftime("%Y-%m-%d"),
                    "until": end_date.strftime("%Y-%m-%d"),
                },
            },
        )

        if not insights:
            return {}

        data = insights[0]

        # Parse conversions and revenue from actions
        conversions = 0
        revenue = 0
        for action in data.get("actions", []):
            if action["action_type"] == "purchase":
                conversions = int(action["value"])

        for value in data.get("action_values", []):
            if value["action_type"] == "purchase":
                revenue = float(value["value"])

        return {
            "spend": float(data.get("spend", 0)),
            "impressions": int(data.get("impressions", 0)),
            "clicks": int(data.get("clicks", 0)),
            "reach": int(data.get("reach", 0)),
            "conversions": conversions,
            "revenue": revenue,
            "cpc": float(data.get("cpc", 0)),
            "cpm": float(data.get("cpm", 0)),
            "ctr": float(data.get("ctr", 0)),
            "roas": revenue / float(data.get("spend", 1)) if data.get("spend") else 0,
        }

    async def update_campaign_status(
        self,
        campaign_id: str,
        status: str,
    ):
        """Update campaign status"""

        campaign = Campaign(campaign_id)
        campaign.update({Campaign.Field.status: status})
        campaign.remote_update()

    async def update_ad_set_budget(
        self,
        ad_set_id: str,
        daily_budget: int,
    ):
        """Update ad set daily budget"""

        ad_set = AdSet(ad_set_id)
        ad_set.update({AdSet.Field.daily_budget: daily_budget})
        ad_set.remote_update()

    async def _upload_image(self, image_url: str) -> str:
        """Upload image to Facebook"""

        from facebook_business.adobjects.adimage import AdImage

        # Download image first
        async with httpx.AsyncClient() as client:
            response = await client.get(image_url)
            image_data = response.content

        # Save temporarily and upload
        import tempfile
        with tempfile.NamedTemporaryFile(suffix=".jpg", delete=False) as f:
            f.write(image_data)
            temp_path = f.name

        image = AdImage(parent_id=self.account.get_id())
        image[AdImage.Field.filename] = temp_path
        image.remote_create()

        import os
        os.unlink(temp_path)

        return image[AdImage.Field.hash]

    def _map_objective(self, objective: str) -> str:
        """Map generic objective to Meta-specific objective"""

        mapping = {
            "awareness": "OUTCOME_AWARENESS",
            "traffic": "OUTCOME_TRAFFIC",
            "engagement": "OUTCOME_ENGAGEMENT",
            "conversions": "OUTCOME_SALES",
            "catalog_sales": "OUTCOME_SALES",
        }
        return mapping.get(objective, "OUTCOME_TRAFFIC")
```

---

## 5. Database Schema Extensions

### 5.1 New Prisma Models

```prisma
// packages/database/prisma/schema.prisma (additions)

// ==================== PLATFORM CONNECTIONS ====================

model PlatformConnection {
  id              String    @id @default(cuid())
  storeId         String
  store           Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)
  userId          String

  platform        String    // meta, tiktok, google, pinterest, twitter
  status          ConnectionStatus @default(PENDING)

  // Account info
  accountId       String?
  accountName     String?

  // Encrypted tokens
  encryptedTokens String    @db.Text

  // Platform-specific metadata
  metadata        Json?     // pages, ad accounts, etc.

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@unique([storeId, platform])
  @@index([storeId])
}

enum ConnectionStatus {
  PENDING
  CONNECTED
  EXPIRED
  REVOKED
  ERROR
}

// ==================== SOCIAL CONTENT ====================

model ContentCalendar {
  id          String    @id @default(cuid())
  storeId     String
  store       Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)

  startDate   DateTime
  endDate     DateTime
  status      CalendarStatus @default(DRAFT)

  posts       SocialPost[]

  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@index([storeId])
}

enum CalendarStatus {
  DRAFT
  ACTIVE
  PAUSED
  COMPLETED
}

model SocialPost {
  id              String    @id @default(cuid())
  calendarId      String?
  calendar        ContentCalendar? @relation(fields: [calendarId], references: [id], onDelete: Cascade)
  storeId         String
  store           Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)

  platform        String
  contentType     String    // image, video, carousel
  caption         String    @db.Text
  hashtags        String[]
  mediaUrls       String[]

  scheduledTime   DateTime?
  publishedTime   DateTime?
  platformPostId  String?
  platformUrl     String?

  status          PostStatus @default(DRAFT)
  error           String?

  // Analytics
  analytics       Json?     // likes, comments, shares, etc.

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([storeId])
  @@index([status])
  @@index([scheduledTime])
}

enum PostStatus {
  DRAFT
  SCHEDULED
  PUBLISHING
  PUBLISHED
  FAILED
}

// ==================== AD CAMPAIGNS ====================

model AdCampaign {
  id              String    @id @default(cuid())
  storeId         String
  store           Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)

  platform        String
  platformCampaignId String?

  name            String
  objective       String
  status          CampaignStatus @default(DRAFT)

  dailyBudget     Decimal   @db.Decimal(10, 2)
  totalBudget     Decimal?  @db.Decimal(10, 2)
  spentAmount     Decimal   @db.Decimal(10, 2) @default(0)

  startDate       DateTime
  endDate         DateTime?

  targeting       Json
  creatives       AdCreative[]

  // Performance metrics
  impressions     Int       @default(0)
  clicks          Int       @default(0)
  conversions     Int       @default(0)
  revenue         Decimal   @db.Decimal(10, 2) @default(0)

  autoOptimize    Boolean   @default(true)

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([storeId])
  @@index([status])
}

enum CampaignStatus {
  DRAFT
  PENDING_REVIEW
  ACTIVE
  PAUSED
  COMPLETED
  REJECTED
  ERROR
}

model AdCreative {
  id              String    @id @default(cuid())
  campaignId      String
  campaign        AdCampaign @relation(fields: [campaignId], references: [id], onDelete: Cascade)

  platformCreativeId String?

  name            String
  type            String    // image, video, carousel
  headline        String
  description     String    @db.Text
  callToAction    String
  mediaUrls       String[]
  linkUrl         String

  status          String    @default("active")

  // A/B test tracking
  impressions     Int       @default(0)
  clicks          Int       @default(0)
  conversions     Int       @default(0)
  spend           Decimal   @db.Decimal(10, 2) @default(0)

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([campaignId])
}

model CampaignOptimizationLog {
  id              String    @id @default(cuid())
  campaignId      String

  action          String    // budget_increase, budget_decrease, creative_paused, etc.
  previousValue   Json?
  newValue        Json?
  reason          String
  confidence      Float

  createdAt       DateTime  @default(now())

  @@index([campaignId])
}
```

---

## 6. Implementation Order

### Sprint 1: OAuth Foundation (Weeks 1-2)
1. Setup Growth Engine FastAPI project
2. Implement OAuth manager
3. Build secure token storage
4. Create Meta OAuth flow
5. Create TikTok OAuth flow
6. Build connection management UI

### Sprint 2: Social Agent Core (Weeks 3-4)
1. Implement content planner with AI
2. Build caption writer
3. Create hashtag optimizer
4. Implement posting scheduler
5. Build TikTok platform client
6. Build Instagram platform client

### Sprint 3: Social Agent Advanced (Weeks 5-6)
1. Add Pinterest and Twitter clients
2. Implement content calendar UI
3. Build analytics aggregation
4. Create social dashboard
5. Add post preview and editing
6. Implement bulk scheduling

### Sprint 4: Ads Agent Core (Weeks 7-8)
1. Implement campaign manager
2. Build creative generator
3. Create Meta Ads client
4. Implement budget optimizer
5. Build campaign creation UI
6. Add performance tracking

### Sprint 5: Ads Agent Advanced (Weeks 9-10)
1. Add Google Ads client
2. Add TikTok Ads client
3. Implement A/B testing
4. Build budget allocation
5. Create ads dashboard
6. Add automated optimization

### Sprint 6: Integration & Polish (Weeks 11-12)
1. End-to-end testing
2. Performance optimization
3. Cross-platform analytics
4. Documentation
5. Security audit
6. Beta launch preparation

---

## 7. Success Metrics

### Technical Metrics
- [ ] OAuth flow completes in < 30 seconds
- [ ] Post scheduling accuracy within 5 minutes
- [ ] Campaign creation time < 2 minutes
- [ ] Ad optimization runs every 6 hours
- [ ] Analytics sync every 4 hours

### Business Metrics
- [ ] Social engagement rate > 3%
- [ ] Ad ROAS > 2x within 30 days
- [ ] Customer acquisition cost reduced by 20%
- [ ] 30-day content calendar fills in < 5 minutes
- [ ] 50% of users connect at least 1 platform

---

## 8. Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| Platform API changes | High | Abstract SDK layers, version pinning |
| Token expiration handling | Medium | Proactive refresh, graceful degradation |
| Ad account bans | High | Compliance checks, content review |
| Budget overspend | High | Hard limits, real-time monitoring |
| Creative fatigue | Medium | Automatic variation generation |
| Platform rate limits | Medium | Request queuing, backoff strategies |

---

*Phase 3 establishes the autonomous marketing hub that drives traffic and sales through organic social media and paid advertising, completing the "zero-effort" e-commerce vision.*
