# Phase 3 - Sprint 1: OAuth Foundation - Action Plan

## Sprint Overview

**Duration:** 2 weeks
**Team Size:** 2-3 developers
**Sprint Goal:** Establish the OAuth integration system that enables secure connections to external marketing platforms (Meta, TikTok, Google, Pinterest, Twitter).

### Objectives
- [ ] Setup Growth Engine FastAPI project structure
- [ ] Implement secure OAuth flow manager
- [ ] Build encrypted token storage system
- [ ] Create Meta (Facebook/Instagram) OAuth integration
- [ ] Create TikTok OAuth integration
- [ ] Build connection management UI in dashboard
- [ ] Ensure token refresh and expiration handling

### Prerequisites
- Phase 1 (Genesis Engine) operational
- Phase 2 (Ops Foundation) deployed
- Dashboard framework ready for new UI
- Redis cache available
- PostgreSQL database with Prisma

---

## Day-by-Day Breakdown

### Days 1-2: Growth Engine Setup

#### Tasks
1. **Initialize FastAPI Project**
   ```bash
   cd services/
   mkdir -p growth-engine
   cd growth-engine
   python -m venv venv
   source venv/bin/activate
   ```

2. **Create Project Structure**
   ```
   services/growth-engine/
   ├── app/
   │   ├── __init__.py
   │   ├── main.py
   │   ├── config.py
   │   ├── api/
   │   │   ├── __init__.py
   │   │   └── v1/
   │   │       ├── __init__.py
   │   │       ├── router.py
   │   │       └── oauth.py
   │   ├── oauth/
   │   │   ├── __init__.py
   │   │   ├── manager.py
   │   │   ├── token_store.py
   │   │   └── providers/
   │   │       ├── __init__.py
   │   │       ├── base.py
   │   │       ├── meta.py
   │   │       ├── tiktok.py
   │   │       ├── google.py
   │   │       ├── pinterest.py
   │   │       └── twitter.py
   │   ├── models/
   │   │   ├── __init__.py
   │   │   └── connection.py
   │   └── services/
   │       ├── __init__.py
   │       └── database.py
   ├── tests/
   │   ├── __init__.py
   │   └── test_oauth.py
   ├── requirements.txt
   ├── Dockerfile
   └── docker-compose.yml
   ```

3. **Install Dependencies**
   ```txt
   # requirements.txt
   fastapi==0.109.0
   uvicorn[standard]==0.27.0
   pydantic==2.5.3
   pydantic-settings==2.1.0

   # Database
   prisma==0.12.0
   asyncpg==0.29.0

   # Cache
   redis==5.0.1

   # OAuth & HTTP
   httpx==0.26.0
   authlib==1.3.0
   python-jose[cryptography]==3.3.0

   # Security
   cryptography==41.0.7

   # Utilities
   python-multipart==0.0.6
   python-dotenv==1.0.0
   ```

4. **Create FastAPI Application**
   ```python
   # app/main.py
   from fastapi import FastAPI
   from fastapi.middleware.cors import CORSMiddleware
   from app.api.v1.router import api_router
   from app.config import settings

   app = FastAPI(
       title="Zyenta Growth Engine",
       description="Autonomous Marketing Hub for Social & Ads",
       version="1.0.0",
   )

   app.add_middleware(
       CORSMiddleware,
       allow_origins=settings.ALLOWED_ORIGINS,
       allow_credentials=True,
       allow_methods=["*"],
       allow_headers=["*"],
   )

   app.include_router(api_router, prefix="/api/v1")

   @app.get("/health")
   async def health_check():
       return {"status": "healthy", "service": "growth-engine"}
   ```

5. **Setup Configuration**
   ```python
   # app/config.py
   from pydantic_settings import BaseSettings

   class Settings(BaseSettings):
       # API
       API_URL: str
       ALLOWED_ORIGINS: list[str] = ["http://localhost:3000"]

       # Database
       DATABASE_URL: str
       REDIS_URL: str

       # Security
       TOKEN_ENCRYPTION_KEY: str

       # Meta (Facebook/Instagram)
       META_APP_ID: str
       META_APP_SECRET: str

       # TikTok
       TIKTOK_CLIENT_KEY: str
       TIKTOK_CLIENT_SECRET: str

       # Google
       GOOGLE_CLIENT_ID: str
       GOOGLE_CLIENT_SECRET: str

       # Pinterest
       PINTEREST_APP_ID: str
       PINTEREST_APP_SECRET: str

       # Twitter
       TWITTER_CLIENT_ID: str
       TWITTER_CLIENT_SECRET: str

       class Config:
           env_file = ".env"

   settings = Settings()
   ```

#### Deliverables
- [ ] Working FastAPI app running on `http://localhost:8003`
- [ ] Health check endpoint responding
- [ ] Environment configuration ready
- [ ] Basic project structure complete

---

### Days 3-4: Database Schema & Token Storage

#### Tasks
1. **Add Prisma Schema Models**
   ```prisma
   // packages/database/prisma/schema.prisma

   model PlatformConnection {
     id              String    @id @default(cuid())
     storeId         String
     store           Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)
     userId          String

     platform        String    // meta, tiktok, google, pinterest, twitter
     status          ConnectionStatus @default(PENDING)

     accountId       String?
     accountName     String?

     encryptedTokens String    @db.Text
     metadata        Json?

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
   ```

2. **Run Prisma Migration**
   ```bash
   cd packages/database
   npx prisma migrate dev --name add_platform_connections
   npx prisma generate
   ```

3. **Implement Token Store**
   ```python
   # app/oauth/token_store.py
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

           key = f"oauth:state:{state}"
           self.redis.setex(key, 600, json.dumps(data))  # 10 min expiry

           return state

       async def verify_state_token(self, state: str) -> Optional[Dict[str, Any]]:
           """Verify and consume state token"""
           key = f"oauth:state:{state}"
           data = await self.redis.get(key)

           if not data:
               return None

           await self.redis.delete(key)
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
           encrypted_tokens = self.fernet.encrypt(
               json.dumps(tokens).encode()
           ).decode()

           data = {
               "tokens": encrypted_tokens,
               "account_info": account_info,
               "updated_at": datetime.utcnow().isoformat(),
           }

           # Store in Redis
           key = f"oauth:tokens:{user_id}:{store_id}:{platform}"
           await self.redis.set(key, json.dumps(data))

           # Persist to database
           await self._persist_to_database(
               user_id, store_id, platform, encrypted_tokens, account_info
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
               data = await self._load_from_database(user_id, store_id, platform)
               if not data:
                   return None

           parsed = json.loads(data)
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
           encrypted_tokens: str,
           account_info: Dict[str, Any],
       ):
           """Persist to PostgreSQL via Prisma"""
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
                   "encryptedTokens": encrypted_tokens,
                   "accountId": account_info.get("id"),
                   "accountName": account_info.get("name"),
                   "status": "CONNECTED",
               },
               update={
                   "encryptedTokens": encrypted_tokens,
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
           """Load from PostgreSQL via Prisma"""
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
           """Delete from PostgreSQL via Prisma"""
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

4. **Create Database Service**
   ```python
   # app/services/database.py
   from prisma import Prisma

   prisma = Prisma()

   async def connect_db():
       await prisma.connect()

   async def disconnect_db():
       await prisma.disconnect()
   ```

5. **Update FastAPI Lifecycle**
   ```python
   # app/main.py (add startup/shutdown)
   from app.services.database import connect_db, disconnect_db

   @app.on_event("startup")
   async def startup():
       await connect_db()

   @app.on_event("shutdown")
   async def shutdown():
       await disconnect_db()
   ```

#### Deliverables
- [ ] Database schema updated with PlatformConnection model
- [ ] Secure token encryption/decryption working
- [ ] Redis integration functional
- [ ] Database persistence layer complete

---

### Days 5-7: OAuth Manager & Meta Integration

#### Tasks
1. **Create Base OAuth Provider**
   ```python
   # app/oauth/providers/base.py
   from abc import ABC, abstractmethod
   from typing import Dict, Any
   from dataclasses import dataclass

   @dataclass
   class OAuthConfig:
       client_id: str
       client_secret: str
       authorize_url: str
       token_url: str
       scopes: list[str]
       redirect_uri: str

   class BaseOAuthProvider(ABC):
       """Base class for OAuth providers"""

       @abstractmethod
       async def get_authorization_url(self, state: str) -> str:
           """Generate authorization URL"""
           pass

       @abstractmethod
       async def exchange_code_for_tokens(self, code: str) -> Dict[str, Any]:
           """Exchange authorization code for tokens"""
           pass

       @abstractmethod
       async def refresh_access_token(self, refresh_token: str) -> Dict[str, Any]:
           """Refresh expired access token"""
           pass

       @abstractmethod
       async def get_account_info(self, access_token: str) -> Dict[str, Any]:
           """Fetch account information"""
           pass

       @abstractmethod
       async def revoke_token(self, token: str) -> bool:
           """Revoke access token"""
           pass
   ```

2. **Implement Meta OAuth Provider**
   ```python
   # app/oauth/providers/meta.py
   from typing import Dict, Any
   import httpx
   from authlib.integrations.httpx_client import AsyncOAuth2Client
   from .base import BaseOAuthProvider, OAuthConfig
   from app.config import settings

   class MetaOAuthProvider(BaseOAuthProvider):
       """OAuth provider for Meta (Facebook/Instagram)"""

       def __init__(self):
           self.config = OAuthConfig(
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
           )

       async def get_authorization_url(self, state: str) -> str:
           """Generate Facebook OAuth URL"""
           client = AsyncOAuth2Client(
               client_id=self.config.client_id,
               client_secret=self.config.client_secret,
               redirect_uri=self.config.redirect_uri,
           )

           url, _ = client.create_authorization_url(
               self.config.authorize_url,
               scope=" ".join(self.config.scopes),
               state=state,
           )

           return url

       async def exchange_code_for_tokens(self, code: str) -> Dict[str, Any]:
           """Exchange code for access token"""
           async with httpx.AsyncClient() as client:
               response = await client.post(
                   self.config.token_url,
                   data={
                       "client_id": self.config.client_id,
                       "client_secret": self.config.client_secret,
                       "code": code,
                       "redirect_uri": self.config.redirect_uri,
                   },
               )
               response.raise_for_status()
               return response.json()

       async def refresh_access_token(self, refresh_token: str) -> Dict[str, Any]:
           """Refresh access token (Meta uses long-lived tokens)"""
           async with httpx.AsyncClient() as client:
               response = await client.get(
                   "https://graph.facebook.com/v18.0/oauth/access_token",
                   params={
                       "grant_type": "fb_exchange_token",
                       "client_id": self.config.client_id,
                       "client_secret": self.config.client_secret,
                       "fb_exchange_token": refresh_token,
                   },
               )
               response.raise_for_status()
               return response.json()

       async def get_account_info(self, access_token: str) -> Dict[str, Any]:
           """Get Facebook user and pages info"""
           async with httpx.AsyncClient() as client:
               response = await client.get(
                   "https://graph.facebook.com/v18.0/me",
                   params={
                       "access_token": access_token,
                       "fields": "id,name,accounts{id,name,access_token,instagram_business_account}",
                   },
               )
               response.raise_for_status()
               data = response.json()

               return {
                   "id": data["id"],
                   "name": data["name"],
                   "pages": data.get("accounts", {}).get("data", []),
               }

       async def revoke_token(self, token: str) -> bool:
           """Revoke Facebook access token"""
           async with httpx.AsyncClient() as client:
               response = await client.delete(
                   f"https://graph.facebook.com/v18.0/me/permissions",
                   params={"access_token": token},
               )
               return response.status_code == 200
   ```

3. **Implement OAuth Manager**
   ```python
   # app/oauth/manager.py
   from typing import Dict, Any
   from enum import Enum
   from app.oauth.token_store import TokenStore
   from app.oauth.providers.meta import MetaOAuthProvider
   from app.oauth.providers.tiktok import TikTokOAuthProvider

   class Platform(str, Enum):
       META = "meta"
       TIKTOK = "tiktok"
       GOOGLE = "google"
       PINTEREST = "pinterest"
       TWITTER = "twitter"

   class OAuthManager:
       """Manages OAuth flows for all supported platforms"""

       def __init__(self):
           self.token_store = TokenStore()
           self.providers = {
               Platform.META: MetaOAuthProvider(),
               Platform.TIKTOK: TikTokOAuthProvider(),
               # Others will be added
           }

       async def get_authorization_url(
           self,
           platform: Platform,
           user_id: str,
           store_id: str,
       ) -> str:
           """Generate authorization URL for platform"""
           state = self.token_store.generate_state_token(
               user_id=user_id,
               store_id=store_id,
               platform=platform.value,
           )

           provider = self.providers[platform]
           return await provider.get_authorization_url(state)

       async def handle_callback(
           self,
           platform: Platform,
           code: str,
           state: str,
       ) -> Dict[str, Any]:
           """Handle OAuth callback"""
           # Verify state token
           state_data = await self.token_store.verify_state_token(state)
           if not state_data:
               raise ValueError("Invalid or expired state token")

           provider = self.providers[platform]

           # Exchange code for tokens
           tokens = await provider.exchange_code_for_tokens(code)

           # Get account info
           account_info = await provider.get_account_info(tokens["access_token"])

           # Store tokens
           await self.token_store.store_tokens(
               user_id=state_data["user_id"],
               store_id=state_data["store_id"],
               platform=platform.value,
               tokens=tokens,
               account_info=account_info,
           )

           return {
               "platform": platform.value,
               "account_id": account_info.get("id"),
               "account_name": account_info.get("name"),
               "connected": True,
           }

       async def refresh_tokens(
           self,
           user_id: str,
           store_id: str,
           platform: Platform,
       ) -> Dict[str, Any]:
           """Refresh expired tokens"""
           stored = await self.token_store.get_tokens(user_id, store_id, platform.value)
           if not stored or "refresh_token" not in stored["tokens"]:
               raise ValueError("No refresh token available")

           provider = self.providers[platform]
           new_tokens = await provider.refresh_access_token(
               stored["tokens"]["refresh_token"]
           )

           # Preserve refresh token if not returned
           if "refresh_token" not in new_tokens:
               new_tokens["refresh_token"] = stored["tokens"]["refresh_token"]

           await self.token_store.store_tokens(
               user_id=user_id,
               store_id=store_id,
               platform=platform.value,
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
           """Revoke OAuth connection"""
           stored = await self.token_store.get_tokens(user_id, store_id, platform.value)
           if not stored:
               return True

           provider = self.providers[platform]
           try:
               await provider.revoke_token(stored["tokens"]["access_token"])
           except Exception:
               pass

           await self.token_store.delete_tokens(user_id, store_id, platform.value)
           return True
   ```

4. **Create OAuth API Endpoints**
   ```python
   # app/api/v1/oauth.py
   from fastapi import APIRouter, HTTPException, Query
   from fastapi.responses import RedirectResponse
   from app.oauth.manager import OAuthManager, Platform

   router = APIRouter(prefix="/oauth", tags=["oauth"])
   oauth_manager = OAuthManager()

   @router.get("/connect/{platform}")
   async def connect_platform(
       platform: Platform,
       user_id: str = Query(...),
       store_id: str = Query(...),
   ):
       """Initiate OAuth flow for a platform"""
       try:
           url = await oauth_manager.get_authorization_url(
               platform=platform,
               user_id=user_id,
               store_id=store_id,
           )
           return {"authorization_url": url}
       except Exception as e:
           raise HTTPException(status_code=400, detail=str(e))

   @router.get("/callback/{platform}")
   async def oauth_callback(
       platform: Platform,
       code: str = Query(...),
       state: str = Query(...),
   ):
       """Handle OAuth callback"""
       try:
           result = await oauth_manager.handle_callback(
               platform=platform,
               code=code,
               state=state,
           )

           # Redirect to dashboard with success
           dashboard_url = f"http://localhost:3000/dashboard/marketing?connection=success&platform={platform.value}"
           return RedirectResponse(url=dashboard_url)

       except Exception as e:
           dashboard_url = f"http://localhost:3000/dashboard/marketing?connection=error&message={str(e)}"
           return RedirectResponse(url=dashboard_url)

   @router.delete("/disconnect/{platform}")
   async def disconnect_platform(
       platform: Platform,
       user_id: str = Query(...),
       store_id: str = Query(...),
   ):
       """Revoke platform connection"""
       try:
           await oauth_manager.revoke_connection(
               user_id=user_id,
               store_id=store_id,
               platform=platform,
           )
           return {"success": True, "message": "Connection revoked"}
       except Exception as e:
           raise HTTPException(status_code=400, detail=str(e))

   @router.get("/connections/{store_id}")
   async def get_connections(store_id: str):
       """Get all platform connections for a store"""
       from app.services.database import prisma

       connections = await prisma.platformconnection.find_many(
           where={"storeId": store_id}
       )

       return {
           "connections": [
               {
                   "platform": c.platform,
                   "status": c.status,
                   "account_name": c.accountName,
                   "connected_at": c.createdAt,
               }
               for c in connections
           ]
       }
   ```

5. **Include OAuth Router**
   ```python
   # app/api/v1/router.py
   from fastapi import APIRouter
   from app.api.v1.oauth import router as oauth_router

   api_router = APIRouter()
   api_router.include_router(oauth_router)
   ```

#### Deliverables
- [ ] OAuth Manager implemented
- [ ] Meta OAuth flow working end-to-end
- [ ] Token refresh mechanism functional
- [ ] API endpoints for OAuth operations

---

### Days 8-9: TikTok OAuth Integration

#### Tasks
1. **Implement TikTok OAuth Provider**
   ```python
   # app/oauth/providers/tiktok.py
   from typing import Dict, Any
   import httpx
   from .base import BaseOAuthProvider, OAuthConfig
   from app.config import settings

   class TikTokOAuthProvider(BaseOAuthProvider):
       """OAuth provider for TikTok"""

       def __init__(self):
           self.config = OAuthConfig(
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
           )

       async def get_authorization_url(self, state: str) -> str:
           """Generate TikTok OAuth URL"""
           # TikTok uses different parameter names
           url = (
               f"{self.config.authorize_url}?"
               f"client_key={self.config.client_id}&"
               f"scope={','.join(self.config.scopes)}&"
               f"response_type=code&"
               f"redirect_uri={self.config.redirect_uri}&"
               f"state={state}"
           )
           return url

       async def exchange_code_for_tokens(self, code: str) -> Dict[str, Any]:
           """Exchange code for TikTok tokens"""
           async with httpx.AsyncClient() as client:
               response = await client.post(
                   self.config.token_url,
                   headers={"Content-Type": "application/x-www-form-urlencoded"},
                   data={
                       "client_key": self.config.client_id,
                       "client_secret": self.config.client_secret,
                       "code": code,
                       "grant_type": "authorization_code",
                       "redirect_uri": self.config.redirect_uri,
                   },
               )
               response.raise_for_status()
               data = response.json()

               # TikTok response format
               if data.get("data"):
                   return {
                       "access_token": data["data"]["access_token"],
                       "refresh_token": data["data"].get("refresh_token"),
                       "expires_in": data["data"].get("expires_in"),
                       "open_id": data["data"].get("open_id"),
                   }

               raise ValueError("Invalid TikTok token response")

       async def refresh_access_token(self, refresh_token: str) -> Dict[str, Any]:
           """Refresh TikTok access token"""
           async with httpx.AsyncClient() as client:
               response = await client.post(
                   self.config.token_url,
                   headers={"Content-Type": "application/x-www-form-urlencoded"},
                   data={
                       "client_key": self.config.client_id,
                       "client_secret": self.config.client_secret,
                       "refresh_token": refresh_token,
                       "grant_type": "refresh_token",
                   },
               )
               response.raise_for_status()
               data = response.json()

               if data.get("data"):
                   return {
                       "access_token": data["data"]["access_token"],
                       "refresh_token": data["data"].get("refresh_token"),
                       "expires_in": data["data"].get("expires_in"),
                   }

               raise ValueError("Invalid TikTok refresh response")

       async def get_account_info(self, access_token: str) -> Dict[str, Any]:
           """Get TikTok user info"""
           async with httpx.AsyncClient() as client:
               response = await client.get(
                   "https://open.tiktokapis.com/v2/user/info/",
                   headers={"Authorization": f"Bearer {access_token}"},
                   params={"fields": "open_id,display_name,avatar_url"},
               )
               response.raise_for_status()
               data = response.json()

               if data.get("data", {}).get("user"):
                   user = data["data"]["user"]
                   return {
                       "id": user.get("open_id"),
                       "name": user.get("display_name"),
                       "avatar": user.get("avatar_url"),
                   }

               raise ValueError("Invalid TikTok user info response")

       async def revoke_token(self, token: str) -> bool:
           """Revoke TikTok access token"""
           async with httpx.AsyncClient() as client:
               response = await client.post(
                   "https://open.tiktokapis.com/v2/oauth/revoke/",
                   headers={"Content-Type": "application/x-www-form-urlencoded"},
                   data={
                       "client_key": self.config.client_id,
                       "client_secret": self.config.client_secret,
                       "token": token,
                   },
               )
               return response.status_code == 200
   ```

2. **Register TikTok Provider**
   ```python
   # app/oauth/manager.py (update)
   from app.oauth.providers.tiktok import TikTokOAuthProvider

   # In OAuthManager.__init__:
   self.providers = {
       Platform.META: MetaOAuthProvider(),
       Platform.TIKTOK: TikTokOAuthProvider(),
   }
   ```

3. **Test TikTok OAuth Flow**
   - Create test endpoint
   - Verify authorization URL generation
   - Test callback handling
   - Verify token storage

#### Deliverables
- [ ] TikTok OAuth provider implemented
- [ ] TikTok OAuth flow tested
- [ ] Token refresh working for TikTok
- [ ] Both Meta and TikTok OAuth functional

---

### Days 10-12: Dashboard UI for Connection Management

#### Tasks
1. **Create Marketing Connections Page**
   ```typescript
   // apps/dashboard/app/dashboard/marketing/page.tsx
   'use client';

   import { useState, useEffect } from 'react';
   import { Button } from '@/components/ui/button';
   import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
   import { Badge } from '@/components/ui/badge';

   interface Connection {
     platform: string;
     status: string;
     account_name: string;
     connected_at: string;
   }

   const PLATFORMS = [
     { id: 'meta', name: 'Facebook & Instagram', icon: '📱' },
     { id: 'tiktok', name: 'TikTok', icon: '🎵' },
     { id: 'google', name: 'Google Ads', icon: '🔍', disabled: true },
     { id: 'pinterest', name: 'Pinterest', icon: '📌', disabled: true },
     { id: 'twitter', name: 'Twitter/X', icon: '🐦', disabled: true },
   ];

   export default function MarketingPage() {
     const [connections, setConnections] = useState<Connection[]>([]);
     const [loading, setLoading] = useState(true);
     const [storeId, setStoreId] = useState(''); // Get from session
     const [userId, setUserId] = useState(''); // Get from session

     useEffect(() => {
       loadConnections();
     }, []);

     async function loadConnections() {
       try {
         const response = await fetch(
           `http://localhost:8003/api/v1/oauth/connections/${storeId}`
         );
         const data = await response.json();
         setConnections(data.connections);
       } catch (error) {
         console.error('Failed to load connections:', error);
       } finally {
         setLoading(false);
       }
     }

     async function connectPlatform(platform: string) {
       try {
         const response = await fetch(
           `http://localhost:8003/api/v1/oauth/connect/${platform}?user_id=${userId}&store_id=${storeId}`
         );
         const data = await response.json();

         // Open OAuth popup
         window.location.href = data.authorization_url;
       } catch (error) {
         console.error('Failed to connect platform:', error);
       }
     }

     async function disconnectPlatform(platform: string) {
       if (!confirm('Are you sure you want to disconnect this platform?')) {
         return;
       }

       try {
         await fetch(
           `http://localhost:8003/api/v1/oauth/disconnect/${platform}?user_id=${userId}&store_id=${storeId}`,
           { method: 'DELETE' }
         );
         loadConnections();
       } catch (error) {
         console.error('Failed to disconnect platform:', error);
       }
     }

     function isConnected(platformId: string): boolean {
       return connections.some(
         (c) => c.platform === platformId && c.status === 'CONNECTED'
       );
     }

     function getConnection(platformId: string): Connection | undefined {
       return connections.find((c) => c.platform === platformId);
     }

     return (
       <div className="container mx-auto py-8">
         <div className="mb-8">
           <h1 className="text-3xl font-bold">Marketing Connections</h1>
           <p className="text-muted-foreground">
             Connect your social media and advertising platforms
           </p>
         </div>

         <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
           {PLATFORMS.map((platform) => {
             const connection = getConnection(platform.id);
             const connected = isConnected(platform.id);

             return (
               <Card key={platform.id}>
                 <CardHeader>
                   <div className="flex items-center justify-between">
                     <div className="flex items-center gap-2">
                       <span className="text-2xl">{platform.icon}</span>
                       <CardTitle>{platform.name}</CardTitle>
                     </div>
                     {connected && (
                       <Badge variant="success">Connected</Badge>
                     )}
                   </div>
                   {connection && (
                     <CardDescription>
                       {connection.account_name}
                     </CardDescription>
                   )}
                 </CardHeader>
                 <CardContent>
                   {connected ? (
                     <Button
                       variant="outline"
                       className="w-full"
                       onClick={() => disconnectPlatform(platform.id)}
                     >
                       Disconnect
                     </Button>
                   ) : (
                     <Button
                       className="w-full"
                       onClick={() => connectPlatform(platform.id)}
                       disabled={platform.disabled}
                     >
                       {platform.disabled ? 'Coming Soon' : 'Connect'}
                     </Button>
                   )}
                 </CardContent>
               </Card>
             );
           })}
         </div>
       </div>
     );
   }
   ```

2. **Add Navigation Link**
   ```typescript
   // apps/dashboard/components/layout/sidebar.tsx
   // Add to navigation items:
   {
     title: 'Marketing',
     href: '/dashboard/marketing',
     icon: TrendingUp,
   }
   ```

3. **Handle OAuth Callback in UI**
   ```typescript
   // apps/dashboard/app/dashboard/marketing/page.tsx
   useEffect(() => {
     // Check for OAuth callback params
     const params = new URLSearchParams(window.location.search);
     const connection = params.get('connection');
     const platform = params.get('platform');
     const message = params.get('message');

     if (connection === 'success') {
       toast.success(`Successfully connected ${platform}!`);
       loadConnections();
       // Clean URL
       window.history.replaceState({}, '', '/dashboard/marketing');
     } else if (connection === 'error') {
       toast.error(`Failed to connect: ${message}`);
       window.history.replaceState({}, '', '/dashboard/marketing');
     }
   }, []);
   ```

#### Deliverables
- [ ] Marketing connections page in dashboard
- [ ] OAuth flow working from UI
- [ ] Connection status display
- [ ] Connect/disconnect functionality

---

### Days 13-14: Testing, Documentation & Deployment

#### Tasks
1. **Write Unit Tests**
   ```python
   # tests/test_oauth.py
   import pytest
   from app.oauth.manager import OAuthManager, Platform
   from app.oauth.token_store import TokenStore

   @pytest.mark.asyncio
   async def test_state_token_generation():
       token_store = TokenStore()
       state = token_store.generate_state_token(
           user_id="user123",
           store_id="store456",
           platform="meta",
       )
       assert len(state) > 20

       # Verify
       data = await token_store.verify_state_token(state)
       assert data["user_id"] == "user123"
       assert data["store_id"] == "store456"

   @pytest.mark.asyncio
   async def test_token_encryption():
       token_store = TokenStore()
       tokens = {"access_token": "secret123", "refresh_token": "refresh456"}

       await token_store.store_tokens(
           user_id="user123",
           store_id="store456",
           platform="meta",
           tokens=tokens,
           account_info={"id": "123", "name": "Test User"},
       )

       retrieved = await token_store.get_tokens("user123", "store456", "meta")
       assert retrieved["tokens"]["access_token"] == "secret123"
   ```

2. **Create Dockerfile**
   ```dockerfile
   # services/growth-engine/Dockerfile
   FROM python:3.11-slim

   WORKDIR /app

   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt

   COPY . .

   EXPOSE 8003

   CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8003"]
   ```

3. **Create Docker Compose**
   ```yaml
   # services/growth-engine/docker-compose.yml
   version: '3.8'

   services:
     growth-engine:
       build: .
       ports:
         - "8003:8003"
       environment:
         - DATABASE_URL=${DATABASE_URL}
         - REDIS_URL=${REDIS_URL}
         - TOKEN_ENCRYPTION_KEY=${TOKEN_ENCRYPTION_KEY}
         - META_APP_ID=${META_APP_ID}
         - META_APP_SECRET=${META_APP_SECRET}
         - TIKTOK_CLIENT_KEY=${TIKTOK_CLIENT_KEY}
         - TIKTOK_CLIENT_SECRET=${TIKTOK_CLIENT_SECRET}
       depends_on:
         - postgres
         - redis
   ```

4. **Write Documentation**
   ```markdown
   # Growth Engine - OAuth Integration

   ## Overview
   Secure OAuth 2.0 integration for marketing platforms.

   ## Supported Platforms
   - Meta (Facebook & Instagram)
   - TikTok
   - Google Ads (Coming Soon)
   - Pinterest (Coming Soon)
   - Twitter/X (Coming Soon)

   ## Setup

   ### Environment Variables
   ```bash
   # Meta
   META_APP_ID=your_app_id
   META_APP_SECRET=your_app_secret

   # TikTok
   TIKTOK_CLIENT_KEY=your_client_key
   TIKTOK_CLIENT_SECRET=your_client_secret

   # Security
   TOKEN_ENCRYPTION_KEY=your_32_byte_key
   ```

   ### Running the Service
   ```bash
   cd services/growth-engine
   docker-compose up -d
   ```

   ## API Endpoints

   ### Connect Platform
   ```
   GET /api/v1/oauth/connect/{platform}?user_id={id}&store_id={id}
   ```

   ### OAuth Callback
   ```
   GET /api/v1/oauth/callback/{platform}?code={code}&state={state}
   ```

   ### Disconnect Platform
   ```
   DELETE /api/v1/oauth/disconnect/{platform}?user_id={id}&store_id={id}
   ```

   ### Get Connections
   ```
   GET /api/v1/oauth/connections/{store_id}
   ```

   ## Security
   - Tokens encrypted using Fernet (AES-128)
   - State tokens expire in 10 minutes
   - Tokens stored in Redis + PostgreSQL
   - Automatic token refresh handling
   ```

5. **Create .env.example**
   ```bash
   # services/growth-engine/.env.example
   # API Configuration
   API_URL=http://localhost:8003
   ALLOWED_ORIGINS=["http://localhost:3000"]

   # Database
   DATABASE_URL=postgresql://user:pass@localhost:5432/zyenta
   REDIS_URL=redis://localhost:6379/0

   # Security (Generate with: python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")
   TOKEN_ENCRYPTION_KEY=your_32_byte_base64_key_here

   # Meta (Facebook/Instagram)
   META_APP_ID=
   META_APP_SECRET=

   # TikTok
   TIKTOK_CLIENT_KEY=
   TIKTOK_CLIENT_SECRET=

   # Google (Coming Soon)
   GOOGLE_CLIENT_ID=
   GOOGLE_CLIENT_SECRET=

   # Pinterest (Coming Soon)
   PINTEREST_APP_ID=
   PINTEREST_APP_SECRET=

   # Twitter (Coming Soon)
   TWITTER_CLIENT_ID=
   TWITTER_CLIENT_SECRET=
   ```

#### Deliverables
- [ ] Unit tests passing
- [ ] Integration tests passing
- [ ] Docker setup complete
- [ ] Documentation written
- [ ] OAuth flow end-to-end tested

---

## Definition of Done

### Technical Requirements
- [ ] Growth Engine FastAPI service running on port 8003
- [ ] OAuth flows working for Meta and TikTok
- [ ] Secure token encryption/decryption functional
- [ ] Token refresh mechanism implemented
- [ ] Database schema migrated
- [ ] Redis integration working
- [ ] All unit tests passing (>80% coverage)
- [ ] Integration tests passing

### UI Requirements
- [ ] Marketing connections page accessible
- [ ] OAuth connect buttons functional
- [ ] Connection status displayed correctly
- [ ] Disconnect functionality working
- [ ] Success/error messages showing

### Security Requirements
- [ ] Tokens encrypted at rest
- [ ] State tokens expire after 10 minutes
- [ ] OAuth callback validates state
- [ ] Environment secrets not committed
- [ ] HTTPS enforced for production

### Documentation
- [ ] API endpoints documented
- [ ] Environment setup guide complete
- [ ] Security practices documented
- [ ] Troubleshooting guide available

---

## Testing Checklist

### Unit Tests
- [ ] State token generation and verification
- [ ] Token encryption/decryption
- [ ] OAuth URL generation
- [ ] Token refresh logic

### Integration Tests
- [ ] Meta OAuth full flow (requires test app)
- [ ] TikTok OAuth full flow (requires test app)
- [ ] Token storage and retrieval
- [ ] Connection revocation

### Manual Testing
- [ ] Connect Meta account from dashboard
- [ ] Connect TikTok account from dashboard
- [ ] Verify connections display correctly
- [ ] Disconnect and verify revocation
- [ ] Test token refresh on expiration
- [ ] Test error handling for invalid state

---

## Risk Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| OAuth app approval delays | Medium | High | Apply for app approvals early in sprint |
| Platform API changes | Low | High | Version pin SDKs, monitor changelogs |
| Token security breach | Low | Critical | Encryption, secure storage, regular key rotation |
| Callback URL mismatches | Medium | Medium | Strict validation, environment-specific configs |
| Rate limiting issues | Low | Medium | Implement exponential backoff |

---

## Success Metrics

- Meta OAuth flow completes in < 30 seconds
- TikTok OAuth flow completes in < 30 seconds
- Token encryption/decryption < 10ms
- 100% of test OAuth flows successful
- Zero token leakage in logs or errors
- Dashboard loads connections in < 500ms

---

## Next Sprint Preview

**Sprint 2: Social Agent Core** will build upon this OAuth foundation to:
- Implement AI content planning
- Build caption writer with LLM
- Create hashtag optimizer
- Implement posting scheduler
- Build platform clients (TikTok, Instagram)

The OAuth system created in this sprint is **critical infrastructure** that all future social and ads features will depend on.
