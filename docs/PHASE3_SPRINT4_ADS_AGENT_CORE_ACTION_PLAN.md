# Phase 3 - Sprint 4: Ads Agent Core - Action Plan

## Sprint Overview

**Duration:** 2 weeks
**Team Size:** 3-4 developers (2 backend, 2 frontend)
**Sprint Goal:** Build the core Ads Agent that autonomously creates, manages, and optimizes paid advertising campaigns on Meta (Facebook/Instagram) with AI-powered creative generation and budget optimization.

### Objectives
- [ ] Implement campaign manager for ad campaign orchestration
- [ ] Build AI-powered creative generator for ad content
- [ ] Create Meta Ads platform client for campaign management
- [ ] Implement budget optimizer with ROAS tracking
- [ ] Build campaign creation UI in dashboard
- [ ] Add performance tracking and analytics
- [ ] Create ad campaign database models

### Prerequisites
- Phase 3 Sprint 1 (OAuth Foundation) complete
- Meta OAuth working with ads permissions
- LLM service configured
- Media assets available from Media Studio
- Store products and branding ready

---

## Day-by-Day Breakdown

### Days 1-2: Ads Agent Architecture & Database Models

#### Tasks
1. **Create Ads Agent Module Structure**
   ```
   services/growth-engine/app/agents/
   ├── ads/
   │   ├── __init__.py
   │   ├── agent.py              # Main Ads Agent orchestrator
   │   ├── campaign_manager.py   # Campaign lifecycle management
   │   ├── creative_generator.py # AI-powered ad creative generation
   │   ├── budget_optimizer.py   # Budget allocation & ROAS optimization
   │   ├── ab_tester.py          # A/B testing and performance analysis
   │   └── platforms/
   │       ├── __init__.py
   │       ├── base.py
   │       ├── meta.py           # Meta (Facebook/Instagram) Ads
   │       ├── google.py         # Google Ads (Sprint 5)
   │       └── tiktok.py         # TikTok Ads (Sprint 5)
   ```

2. **Add Database Models for Ad Campaigns**
   ```prisma
   // packages/database/prisma/schema.prisma (add)

   model AdCampaign {
     id              String    @id @default(cuid())
     storeId         String
     store           Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)

     platform        String    // meta, google, tiktok
     platformCampaignId String?

     name            String
     objective       CampaignObjective
     status          CampaignStatus @default(DRAFT)

     dailyBudget     Decimal   @db.Decimal(10, 2)
     totalBudget     Decimal?  @db.Decimal(10, 2)
     spentAmount     Decimal   @db.Decimal(10, 2) @default(0)

     startDate       DateTime
     endDate         DateTime?

     targeting       Json      // Demographics, interests, behaviors
     creatives       AdCreative[]

     // Performance metrics
     impressions     Int       @default(0)
     clicks          Int       @default(0)
     conversions     Int       @default(0)
     revenue         Decimal   @db.Decimal(10, 2) @default(0)
     cpc             Decimal   @db.Decimal(10, 4) @default(0)
     cpm             Decimal   @db.Decimal(10, 4) @default(0)
     ctr             Decimal   @db.Decimal(10, 4) @default(0)
     roas            Decimal   @db.Decimal(10, 4) @default(0)

     autoOptimize    Boolean   @default(true)
     lastOptimizedAt DateTime?

     createdAt       DateTime  @default(now())
     updatedAt       DateTime  @updatedAt

     @@index([storeId])
     @@index([status])
     @@index([platform])
   }

   enum CampaignObjective {
     AWARENESS
     TRAFFIC
     ENGAGEMENT
     CONVERSIONS
     CATALOG_SALES
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
     platformAdSetId    String?

     name            String
     type            String    // image, video, carousel
     headline        String
     description     String    @db.Text
     callToAction    String
     mediaUrls       String[]
     linkUrl         String

     status          CreativeStatus @default(ACTIVE)

     // A/B test tracking
     impressions     Int       @default(0)
     clicks          Int       @default(0)
     conversions     Int       @default(0)
     spend           Decimal   @db.Decimal(10, 2) @default(0)
     revenue         Decimal   @db.Decimal(10, 2) @default(0)

     createdAt       DateTime  @default(now())
     updatedAt       DateTime  @updatedAt

     @@index([campaignId])
     @@index([status])
   }

   enum CreativeStatus {
     ACTIVE
     PAUSED
     TESTING
     WINNER
     LOSER
   }

   model CampaignOptimizationLog {
     id              String    @id @default(cuid())
     campaignId      String

     action          OptimizationAction
     previousValue   Json?
     newValue        Json?
     reason          String
     confidence      Decimal   @db.Decimal(5, 4)

     metrics         Json?     // Snapshot of metrics at time of optimization

     createdAt       DateTime  @default(now())

     @@index([campaignId])
     @@index([createdAt])
   }

   enum OptimizationAction {
     BUDGET_INCREASE
     BUDGET_DECREASE
     BUDGET_MAINTAIN
     CREATIVE_PAUSED
     CREATIVE_SCALED
     CAMPAIGN_PAUSED
     TARGETING_ADJUSTED
   }
   ```

3. **Run Database Migration**
   ```bash
   cd packages/database
   npx prisma migrate dev --name add_ad_campaigns
   npx prisma generate
   ```

4. **Create Ads Agent Base Classes**
   ```python
   # app/agents/ads/__init__.py
   from enum import Enum

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
       REJECTED = "rejected"
       ERROR = "error"
   ```

#### Deliverables
- [ ] Ads agent module structure created
- [ ] Database models for campaigns and creatives
- [ ] Migration applied successfully
- [ ] Base enums and types defined

---

### Days 3-5: Creative Generator & Campaign Manager

#### Tasks
1. **Implement Creative Generator**
   ```python
   # app/agents/ads/creative_generator.py
   from typing import List, Dict, Any, Optional
   from dataclasses import dataclass
   from app.services.llm import LLMService
   from app.agents.ads import CampaignObjective

   @dataclass
   class AdCreative:
       name: str
       type: str  # image, video, carousel
       headline: str
       description: str
       call_to_action: str
       media_urls: List[str]
       link_url: str

   class CreativeGenerator:
       """AI-powered ad creative generation"""

       def __init__(self):
           self.llm = LLMService()

       # Platform CTA options
       CTA_OPTIONS = {
           "meta": [
               "SHOP_NOW",
               "LEARN_MORE",
               "SIGN_UP",
               "DOWNLOAD",
               "GET_OFFER",
               "CONTACT_US",
               "BOOK_NOW",
               "APPLY_NOW",
           ],
           "google": [
               "shop",
               "learn_more",
               "sign_up",
               "download",
               "get_quote",
           ],
       }

       async def generate(
           self,
           store: Any,
           products: List[Dict],
           media_assets: List[Dict],
           platform: str,
           objective: CampaignObjective,
           variations: int = 3,
       ) -> List[AdCreative]:
           """Generate ad creative variations"""

           # Select products for ads
           featured_products = products[:5]  # Top 5 products

           # Generate creative concepts
           concepts = await self._generate_concepts(
               store=store,
               products=featured_products,
               objective=objective,
               variations=variations,
           )

           # Create creatives from concepts
           creatives = []
           for i, concept in enumerate(concepts):
               creative = await self._create_creative(
                   concept=concept,
                   store=store,
                   media_assets=media_assets,
                   platform=platform,
                   variation_number=i + 1,
               )
               creatives.append(creative)

           return creatives

       async def _generate_concepts(
           self,
           store: Any,
           products: List[Dict],
           objective: CampaignObjective,
           variations: int,
       ) -> List[Dict[str, Any]]:
           """Generate creative concepts using AI"""

           niche = store.niche if hasattr(store, 'niche') else "general"
           brand_name = store.name if hasattr(store, 'name') else "Store"

           product_info = "\n".join([
               f"- {p.get('title', '')}: ${p.get('sellingPrice', 0)} - {p.get('description', '')[:100]}"
               for p in products
           ])

           prompt = f"""
           Create {variations} distinct ad creative concepts for a {objective.value} campaign.

           Brand: {brand_name}
           Niche: {niche}
           Products:
           {product_info}

           For each concept, provide:
           1. Target angle (what makes this unique)
           2. Headline (attention-grabbing, 40 chars max)
           3. Primary text (ad copy, 125 chars max)
           4. Visual suggestion (what image/video to use)
           5. Call to action
           6. Emotion to evoke

           Requirements:
           - Make each concept unique and test different angles
           - Focus on benefits, not features
           - Use urgency and social proof where appropriate
           - Align with {objective.value} objective

           Return JSON:
           {{
             "concepts": [
               {{
                 "angle": "...",
                 "headline": "...",
                 "primary_text": "...",
                 "visual": "...",
                 "cta": "...",
                 "emotion": "..."
               }}
             ]
           }}
           """

           response = await self.llm.generate(
               prompt=prompt,
               response_format={"type": "json_object"},
               temperature=0.8,  # More creative
           )

           return response.get("concepts", [])

       async def _create_creative(
           self,
           concept: Dict[str, Any],
           store: Any,
           media_assets: List[Dict],
           platform: str,
           variation_number: int,
       ) -> AdCreative:
           """Create ad creative from concept"""

           # Select media based on visual suggestion
           media_urls = self._select_media(
               visual_hint=concept.get("visual", ""),
               media_assets=media_assets,
           )

           # Get store URL
           store_url = f"https://{store.domain}" if hasattr(store, 'domain') else "https://example.com"

           # Map CTA to platform-specific format
           cta = self._map_cta(concept.get("cta", "Shop Now"), platform)

           return AdCreative(
               name=f"{concept.get('angle', 'Concept')} - Variation {variation_number}",
               type="image" if media_urls and not media_urls[0].endswith('.mp4') else "video",
               headline=concept.get("headline", "")[:40],
               description=concept.get("primary_text", "")[:125],
               call_to_action=cta,
               media_urls=media_urls,
               link_url=store_url,
           )

       def _select_media(
           self,
           visual_hint: str,
           media_assets: List[Dict],
       ) -> List[str]:
           """Select appropriate media for creative"""

           if not media_assets:
               return []

           # Prefer videos for engagement/awareness
           # Prefer product images for conversions
           if "video" in visual_hint.lower():
               videos = [m for m in media_assets if m.get("type") == "VIDEO"]
               if videos:
                   return [videos[0]["url"]]

           # Default to first available image
           images = [m for m in media_assets if m.get("type") == "IMAGE"]
           if images:
               return [images[0]["url"]]

           return []

       def _map_cta(self, cta_text: str, platform: str) -> str:
           """Map CTA text to platform-specific format"""

           cta_mapping = {
               "shop now": "SHOP_NOW",
               "learn more": "LEARN_MORE",
               "sign up": "SIGN_UP",
               "download": "DOWNLOAD",
               "get offer": "GET_OFFER",
               "book now": "BOOK_NOW",
           }

           cta_lower = cta_text.lower()
           for key, value in cta_mapping.items():
               if key in cta_lower:
                   return value

           return "LEARN_MORE"  # Default
   ```

2. **Implement Campaign Manager**
   ```python
   # app/agents/ads/campaign_manager.py
   from typing import Dict, Any, Optional
   from datetime import datetime
   from decimal import Decimal
   from app.agents.ads.platforms import get_ads_platform_client
   from app.oauth.token_store import TokenStore

   class CampaignManager:
       """Manages ad campaign lifecycle across platforms"""

       def __init__(self):
           self.token_store = TokenStore()

       async def launch(
           self,
           campaign: Any,
           store_id: str,
       ) -> Dict[str, Any]:
           """Launch a campaign on the ad platform"""

           # Get platform connection
           from app.services.database import prisma

           connection = await prisma.platformconnection.find_unique(
               where={
                   "storeId_platform": {
                       "storeId": store_id,
                       "platform": campaign.platform,
                   }
               }
           )

           if not connection:
               raise ValueError(f"Platform not connected: {campaign.platform}")

           # Get tokens
           tokens = await self.token_store.get_tokens(
               connection.userId,
               store_id,
               campaign.platform,
           )

           # Get platform client
           client = get_ads_platform_client(
               campaign.platform,
               tokens["tokens"],
           )

           # Create campaign on platform
           platform_campaign = await client.create_campaign(
               name=campaign.name,
               objective=campaign.objective,
               daily_budget=int(campaign.daily_budget * 100),  # Convert to cents
               status="PAUSED",  # Start paused for review
           )

           # Create ad sets with targeting
           ad_set_results = []
           for creative in campaign.creatives:
               ad_set = await client.create_ad_set(
                   campaign_id=platform_campaign["id"],
                   name=f"{campaign.name} - {creative.name}",
                   daily_budget=int(campaign.daily_budget * 100 / len(campaign.creatives)),
                   targeting=campaign.targeting,
                   optimization_goal=self._get_optimization_goal(campaign.objective),
               )

               # Create ad creative
               platform_creative = await client.create_ad_creative(
                   name=creative.name,
                   page_id=connection.metadata.get("page_id"),  # For Meta
                   image_url=creative.media_urls[0] if creative.media_urls else None,
                   headline=creative.headline,
                   description=creative.description,
                   call_to_action=creative.call_to_action,
                   link_url=creative.link_url,
               )

               # Create ad
               ad = await client.create_ad(
                   name=creative.name,
                   ad_set_id=ad_set["id"],
                   creative_id=platform_creative["id"],
                   status="PAUSED",
               )

               ad_set_results.append({
                   "ad_set_id": ad_set["id"],
                   "creative_id": platform_creative["id"],
                   "ad_id": ad["id"],
               })

               # Update creative with platform IDs
               await prisma.adcreative.update(
                   where={"id": creative.id},
                   data={
                       "platformCreativeId": platform_creative["id"],
                       "platformAdSetId": ad_set["id"],
                   },
               )

           # Update campaign with platform ID
           await prisma.adcampaign.update(
               where={"id": campaign.id},
               data={
                   "platformCampaignId": platform_campaign["id"],
                   "status": "PENDING_REVIEW",
               },
           )

           return {
               "campaign_id": platform_campaign["id"],
               "ad_sets": ad_set_results,
               "status": "pending_review",
           }

       async def activate_campaign(self, campaign_id: str):
           """Activate a campaign"""

           from app.services.database import prisma

           campaign = await prisma.adcampaign.find_unique(
               where={"id": campaign_id}
           )

           if not campaign or not campaign.platformCampaignId:
               raise ValueError("Campaign not found or not launched")

           # Get connection
           connection = await prisma.platformconnection.find_unique(
               where={
                   "storeId_platform": {
                       "storeId": campaign.storeId,
                       "platform": campaign.platform,
                   }
               }
           )

           tokens = await self.token_store.get_tokens(
               connection.userId,
               campaign.storeId,
               campaign.platform,
           )

           client = get_ads_platform_client(campaign.platform, tokens["tokens"])

           # Activate campaign
           await client.update_campaign_status(
               campaign.platformCampaignId,
               "ACTIVE",
           )

           # Update database
           await prisma.adcampaign.update(
               where={"id": campaign_id},
               data={"status": "ACTIVE"},
           )

       async def pause_campaign(self, campaign_id: str):
           """Pause a campaign"""

           from app.services.database import prisma

           campaign = await prisma.adcampaign.find_unique(
               where={"id": campaign_id}
           )

           if not campaign or not campaign.platformCampaignId:
               raise ValueError("Campaign not found or not launched")

           connection = await prisma.platformconnection.find_unique(
               where={
                   "storeId_platform": {
                       "storeId": campaign.storeId,
                       "platform": campaign.platform,
                   }
               }
           )

           tokens = await self.token_store.get_tokens(
               connection.userId,
               campaign.storeId,
               campaign.platform,
           )

           client = get_ads_platform_client(campaign.platform, tokens["tokens"])

           await client.update_campaign_status(
               campaign.platformCampaignId,
               "PAUSED",
           )

           await prisma.adcampaign.update(
               where={"id": campaign_id},
               data={"status": "PAUSED"},
           )

       async def get_performance(
           self,
           campaign_id: str,
           platform: str,
       ) -> Dict[str, Any]:
           """Get campaign performance metrics"""

           from app.services.database import prisma

           campaign = await prisma.adcampaign.find_unique(
               where={"id": campaign_id}
           )

           if not campaign or not campaign.platformCampaignId:
               return {}

           connection = await prisma.platformconnection.find_unique(
               where={
                   "storeId_platform": {
                       "storeId": campaign.storeId,
                       "platform": platform,
                   }
               }
           )

           tokens = await self.token_store.get_tokens(
               connection.userId,
               campaign.storeId,
               platform,
           )

           client = get_ads_platform_client(platform, tokens["tokens"])

           # Get insights from platform
           insights = await client.get_campaign_insights(
               campaign_id=campaign.platformCampaignId,
               start_date=campaign.startDate,
               end_date=datetime.utcnow(),
           )

           # Update campaign metrics
           await prisma.adcampaign.update(
               where={"id": campaign_id},
               data={
                   "impressions": insights.get("impressions", 0),
                   "clicks": insights.get("clicks", 0),
                   "conversions": insights.get("conversions", 0),
                   "spentAmount": Decimal(str(insights.get("spend", 0))),
                   "revenue": Decimal(str(insights.get("revenue", 0))),
                   "cpc": Decimal(str(insights.get("cpc", 0))),
                   "cpm": Decimal(str(insights.get("cpm", 0))),
                   "ctr": Decimal(str(insights.get("ctr", 0))),
                   "roas": Decimal(str(insights.get("roas", 0))),
               },
           )

           return insights

       def _get_optimization_goal(self, objective: str) -> str:
           """Map campaign objective to optimization goal"""

           mapping = {
               "awareness": "REACH",
               "traffic": "LINK_CLICKS",
               "engagement": "ENGAGEMENT",
               "conversions": "CONVERSIONS",
               "catalog_sales": "CONVERSIONS",
           }

           return mapping.get(objective, "LINK_CLICKS")

       async def adjust_budget(
           self,
           campaign_id: str,
           new_budget: Decimal,
       ):
           """Adjust campaign budget"""

           from app.services.database import prisma

           campaign = await prisma.adcampaign.find_unique(
               where={"id": campaign_id},
               include={"creatives": True},
           )

           if not campaign:
               raise ValueError("Campaign not found")

           connection = await prisma.platformconnection.find_unique(
               where={
                   "storeId_platform": {
                       "storeId": campaign.storeId,
                       "platform": campaign.platform,
                   }
               }
           )

           tokens = await self.token_store.get_tokens(
               connection.userId,
               campaign.storeId,
               campaign.platform,
           )

           client = get_ads_platform_client(campaign.platform, tokens["tokens"])

           # Update budget for each ad set
           budget_per_creative = new_budget / len(campaign.creatives)
           for creative in campaign.creatives:
               if creative.platformAdSetId:
                   await client.update_ad_set_budget(
                       creative.platformAdSetId,
                       int(budget_per_creative * 100),
                   )

           # Update database
           await prisma.adcampaign.update(
               where={"id": campaign_id},
               data={"dailyBudget": new_budget},
           )

       async def pause_creative(self, campaign_id: str, creative_id: str):
           """Pause an underperforming creative"""

           from app.services.database import prisma

           creative = await prisma.adcreative.find_unique(
               where={"id": creative_id}
           )

           if not creative or not creative.platformAdSetId:
               return

           campaign = await prisma.adcampaign.find_unique(
               where={"id": campaign_id}
           )

           connection = await prisma.platformconnection.find_unique(
               where={
                   "storeId_platform": {
                       "storeId": campaign.storeId,
                       "platform": campaign.platform,
                   }
               }
           )

           tokens = await self.token_store.get_tokens(
               connection.userId,
               campaign.storeId,
               campaign.platform,
           )

           client = get_ads_platform_client(campaign.platform, tokens["tokens"])

           # Pause ad set
           await client.update_ad_set_status(
               creative.platformAdSetId,
               "PAUSED",
           )

           # Update database
           await prisma.adcreative.update(
               where={"id": creative_id},
               data={"status": "PAUSED"},
           )

       async def increase_budget(
           self,
           campaign_id: str,
           creative_id: str,
           increase_percent: int,
       ):
           """Increase budget for winning creative"""

           from app.services.database import prisma

           creative = await prisma.adcreative.find_unique(
               where={"id": creative_id}
           )

           campaign = await prisma.adcampaign.find_unique(
               where={"id": campaign_id},
               include={"creatives": True},
           )

           if not creative or not creative.platformAdSetId:
               return

           # Calculate new budget
           current_budget = campaign.dailyBudget / len(campaign.creatives)
           new_budget = current_budget * (1 + increase_percent / 100)

           connection = await prisma.platformconnection.find_unique(
               where={
                   "storeId_platform": {
                       "storeId": campaign.storeId,
                       "platform": campaign.platform,
                   }
               }
           )

           tokens = await self.token_store.get_tokens(
               connection.userId,
               campaign.storeId,
               campaign.platform,
           )

           client = get_ads_platform_client(campaign.platform, tokens["tokens"])

           await client.update_ad_set_budget(
               creative.platformAdSetId,
               int(new_budget * 100),
           )
   ```

#### Deliverables
- [ ] Creative generator creating AI-powered ad variations
- [ ] Campaign manager handling full campaign lifecycle
- [ ] Campaign launch to platform working
- [ ] Budget adjustment functionality

---

### Days 6-8: Meta Ads Platform Client

#### Tasks
1. **Create Base Ads Platform Client**
   ```python
   # app/agents/ads/platforms/base.py
   from abc import ABC, abstractmethod
   from typing import Dict, Any, Optional
   from datetime import datetime

   class BaseAdsPlatformClient(ABC):
       """Base class for advertising platform clients"""

       @abstractmethod
       async def create_campaign(
           self,
           name: str,
           objective: str,
           daily_budget: int,
           status: str = "PAUSED",
       ) -> Dict[str, Any]:
           """Create an ad campaign"""
           pass

       @abstractmethod
       async def create_ad_set(
           self,
           campaign_id: str,
           name: str,
           daily_budget: int,
           targeting: Dict[str, Any],
           optimization_goal: str,
           start_time: Optional[datetime] = None,
           end_time: Optional[datetime] = None,
       ) -> Dict[str, Any]:
           """Create an ad set within a campaign"""
           pass

       @abstractmethod
       async def create_ad_creative(
           self,
           name: str,
           page_id: str,
           image_url: Optional[str] = None,
           video_id: Optional[str] = None,
           headline: str = "",
           description: str = "",
           call_to_action: str = "LEARN_MORE",
           link_url: str = "",
       ) -> Dict[str, Any]:
           """Create an ad creative"""
           pass

       @abstractmethod
       async def create_ad(
           self,
           name: str,
           ad_set_id: str,
           creative_id: str,
           status: str = "PAUSED",
       ) -> Dict[str, Any]:
           """Create an ad"""
           pass

       @abstractmethod
       async def get_campaign_insights(
           self,
           campaign_id: str,
           start_date: datetime,
           end_date: datetime,
       ) -> Dict[str, Any]:
           """Get campaign performance insights"""
           pass

       @abstractmethod
       async def update_campaign_status(
           self,
           campaign_id: str,
           status: str,
       ):
           """Update campaign status"""
           pass

       @abstractmethod
       async def update_ad_set_budget(
           self,
           ad_set_id: str,
           daily_budget: int,
       ):
           """Update ad set budget"""
           pass

       @abstractmethod
       async def update_ad_set_status(
           self,
           ad_set_id: str,
           status: str,
       ):
           """Update ad set status"""
           pass
   ```

2. **Implement Meta Ads Client**
   ```python
   # app/agents/ads/platforms/meta.py
   from typing import Dict, Any, List, Optional
   from datetime import datetime
   import httpx
   from .base import BaseAdsPlatformClient

   class MetaAdsClient(BaseAdsPlatformClient):
       """Meta (Facebook/Instagram) Ads API client"""

       BASE_URL = "https://graph.facebook.com/v18.0"

       def __init__(self, tokens: Dict[str, Any], ad_account_id: str):
           self.access_token = tokens["access_token"]
           self.ad_account_id = ad_account_id

       async def _request(
           self,
           method: str,
           endpoint: str,
           **kwargs,
       ) -> Dict[str, Any]:
           """Make authenticated API request"""

           params = kwargs.get("params", {})
           params["access_token"] = self.access_token
           kwargs["params"] = params

           async with httpx.AsyncClient() as client:
               response = await client.request(
                   method,
                   f"{self.BASE_URL}{endpoint}",
                   timeout=30.0,
                   **kwargs,
               )
               response.raise_for_status()
               return response.json()

       async def create_campaign(
           self,
           name: str,
           objective: str,
           daily_budget: int,
           status: str = "PAUSED",
       ) -> Dict[str, Any]:
           """Create a Meta ad campaign"""

           objective_map = {
               "awareness": "OUTCOME_AWARENESS",
               "traffic": "OUTCOME_TRAFFIC",
               "engagement": "OUTCOME_ENGAGEMENT",
               "conversions": "OUTCOME_SALES",
               "catalog_sales": "OUTCOME_SALES",
           }

           response = await self._request(
               "POST",
               f"/act_{self.ad_account_id}/campaigns",
               params={
                   "name": name,
                   "objective": objective_map.get(objective, "OUTCOME_TRAFFIC"),
                   "status": status,
                   "special_ad_categories": "[]",
               },
           )

           return {
               "id": response["id"],
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
           start_time: Optional[datetime] = None,
           end_time: Optional[datetime] = None,
       ) -> Dict[str, Any]:
           """Create an ad set"""

           params = {
               "name": name,
               "campaign_id": campaign_id,
               "daily_budget": daily_budget,
               "billing_event": "IMPRESSIONS",
               "optimization_goal": optimization_goal,
               "targeting": targeting,
               "status": "PAUSED",
           }

           if start_time:
               params["start_time"] = start_time.isoformat()
           if end_time:
               params["end_time"] = end_time.isoformat()

           response = await self._request(
               "POST",
               f"/act_{self.ad_account_id}/adsets",
               params=params,
           )

           return {
               "id": response["id"],
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
           call_to_action: str = "LEARN_MORE",
           link_url: str = "",
       ) -> Dict[str, Any]:
           """Create an ad creative"""

           object_story_spec = {
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
           }

           if image_url:
               # Upload image
               image_hash = await self._upload_image(image_url)
               object_story_spec["link_data"]["image_hash"] = image_hash
           elif video_id:
               object_story_spec["video_data"] = {
                   "video_id": video_id,
                   "call_to_action": {
                       "type": call_to_action,
                       "value": {"link": link_url},
                   },
               }

           response = await self._request(
               "POST",
               f"/act_{self.ad_account_id}/adcreatives",
               params={
                   "name": name,
                   "object_story_spec": object_story_spec,
               },
           )

           return {
               "id": response["id"],
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

           response = await self._request(
               "POST",
               f"/act_{self.ad_account_id}/ads",
               params={
                   "name": name,
                   "adset_id": ad_set_id,
                   "creative": f'{{"creative_id": "{creative_id}"}}',
                   "status": status,
               },
           )

           return {
               "id": response["id"],
               "name": name,
           }

       async def get_campaign_insights(
           self,
           campaign_id: str,
           start_date: datetime,
           end_date: datetime,
       ) -> Dict[str, Any]:
           """Get campaign performance insights"""

           response = await self._request(
               "GET",
               f"/{campaign_id}/insights",
               params={
                   "fields": "spend,impressions,clicks,reach,actions,action_values,cpc,cpm,ctr",
                   "time_range": f'{{"since":"{start_date.strftime("%Y-%m-%d")}","until":"{end_date.strftime("%Y-%m-%d")}"}}',
               },
           )

           if not response.get("data"):
               return {}

           data = response["data"][0]

           # Parse conversions and revenue
           conversions = 0
           revenue = 0
           for action in data.get("actions", []):
               if action["action_type"] == "purchase":
                   conversions = int(action["value"])

           for value in data.get("action_values", []):
               if value["action_type"] == "purchase":
                   revenue = float(value["value"])

           spend = float(data.get("spend", 0))
           roas = revenue / spend if spend > 0 else 0

           return {
               "spend": spend,
               "impressions": int(data.get("impressions", 0)),
               "clicks": int(data.get("clicks", 0)),
               "reach": int(data.get("reach", 0)),
               "conversions": conversions,
               "revenue": revenue,
               "cpc": float(data.get("cpc", 0)),
               "cpm": float(data.get("cpm", 0)),
               "ctr": float(data.get("ctr", 0)),
               "roas": roas,
           }

       async def update_campaign_status(
           self,
           campaign_id: str,
           status: str,
       ):
           """Update campaign status"""

           await self._request(
               "POST",
               f"/{campaign_id}",
               params={"status": status},
           )

       async def update_ad_set_budget(
           self,
           ad_set_id: str,
           daily_budget: int,
       ):
           """Update ad set daily budget"""

           await self._request(
               "POST",
               f"/{ad_set_id}",
               params={"daily_budget": daily_budget},
           )

       async def update_ad_set_status(
           self,
           ad_set_id: str,
           status: str,
       ):
           """Update ad set status"""

           await self._request(
               "POST",
               f"/{ad_set_id}",
               params={"status": status},
           )

       async def _upload_image(self, image_url: str) -> str:
           """Upload image to Facebook"""

           # Download image
           async with httpx.AsyncClient() as client:
               response = await client.get(image_url)
               image_data = response.content

           # Save temporarily
           import tempfile
           import os

           with tempfile.NamedTemporaryFile(suffix=".jpg", delete=False) as f:
               f.write(image_data)
               temp_path = f.name

           try:
               # Upload to Facebook
               with open(temp_path, 'rb') as image_file:
                   files = {'file': image_file}
                   async with httpx.AsyncClient() as client:
                       response = await client.post(
                           f"{self.BASE_URL}/act_{self.ad_account_id}/adimages",
                           params={"access_token": self.access_token},
                           files=files,
                       )
                       response.raise_for_status()
                       data = response.json()

                   # Get hash from response
                   images = data.get("images", {})
                   if images:
                       first_key = list(images.keys())[0]
                       return images[first_key]["hash"]

               raise ValueError("Failed to upload image")

           finally:
               os.unlink(temp_path)
   ```

3. **Create Platform Factory**
   ```python
   # app/agents/ads/platforms/__init__.py
   from typing import Dict, Any
   from .base import BaseAdsPlatformClient
   from .meta import MetaAdsClient

   def get_ads_platform_client(
       platform: str,
       tokens: Dict[str, Any],
       ad_account_id: Optional[str] = None,
   ) -> BaseAdsPlatformClient:
       """Factory function to get ads platform client"""

       if platform == "meta":
           if not ad_account_id:
               raise ValueError("Meta Ads requires ad_account_id")
           return MetaAdsClient(tokens, ad_account_id)
       else:
           raise ValueError(f"Unsupported ads platform: {platform}")
   ```

#### Deliverables
- [ ] Meta Ads client fully implemented
- [ ] Campaign creation working on Meta
- [ ] Ad set and creative creation functional
- [ ] Performance insights retrieval working
- [ ] Budget and status updates functional

---

### Days 9-10: Budget Optimizer

#### Tasks
1. **Implement Budget Optimizer**
   ```python
   # app/agents/ads/budget_optimizer.py
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

       # ROAS thresholds
       ROAS_EXCELLENT = Decimal("3.0")
       ROAS_GOOD = Decimal("2.0")
       ROAS_ACCEPTABLE = Decimal("1.0")
       ROAS_POOR = Decimal("0.5")

       # Budget change limits
       MAX_DAILY_BUDGET_INCREASE = Decimal("0.20")  # 20%
       MIN_LEARNING_SPEND = Decimal("50.0")
       MIN_LEARNING_DAYS = 3

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
           if spend < self.MIN_LEARNING_SPEND or days_running < self.MIN_LEARNING_DAYS:
               return OptimizationResult(
                   action=BudgetAction.MAINTAIN,
                   new_budget=None,
                   reason="Campaign still in learning phase - need more data",
                   confidence=0.5,
               )

           # Excellent performance - scale up aggressively
           if roas >= self.ROAS_EXCELLENT:
               increase = min(
                   campaign.dailyBudget * self.MAX_DAILY_BUDGET_INCREASE,
                   Decimal("100.0"),  # Max $100 increase
               )
               return OptimizationResult(
                   action=BudgetAction.INCREASE,
                   new_budget=campaign.dailyBudget + increase,
                   reason=f"Excellent ROAS of {roas:.2f}x - scaling budget by ${increase}",
                   confidence=0.9,
               )

           # Good performance - moderate increase
           if roas >= self.ROAS_GOOD:
               increase = campaign.dailyBudget * Decimal("0.10")
               return OptimizationResult(
                   action=BudgetAction.INCREASE,
                   new_budget=campaign.dailyBudget + increase,
                   reason=f"Good ROAS of {roas:.2f}x - gradual scaling by ${increase}",
                   confidence=0.75,
               )

           # Acceptable performance - maintain
           if roas >= self.ROAS_ACCEPTABLE:
               return OptimizationResult(
                   action=BudgetAction.MAINTAIN,
                   new_budget=None,
                   reason=f"Acceptable ROAS of {roas:.2f}x - maintaining current budget",
                   confidence=0.7,
               )

           # Poor performance - pause campaign
           if roas < self.ROAS_POOR:
               return OptimizationResult(
                   action=BudgetAction.PAUSE,
                   new_budget=Decimal("0"),
                   reason=f"Poor ROAS of {roas:.2f}x - pausing campaign to prevent losses",
                   confidence=0.85,
               )

           # Suboptimal - reduce budget
           decrease = campaign.dailyBudget * Decimal("0.20")
           return OptimizationResult(
               action=BudgetAction.DECREASE,
               new_budget=campaign.dailyBudget - decrease,
               reason=f"Suboptimal ROAS of {roas:.2f}x - reducing budget by ${decrease}",
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
               if campaign.status != "ACTIVE":
                   allocations[campaign.id] = Decimal("0")
                   continue

               roas = campaign.roas or Decimal("1.0")

               # Weight by ROAS (higher ROAS = more budget)
               weight = max(roas, Decimal("0.1"))
               campaign._temp_weight = weight
               total_weight += weight

           # Allocate based on weights
           for campaign in campaigns:
               if campaign.status != "ACTIVE":
                   continue

               weight = getattr(campaign, "_temp_weight", Decimal("1"))
               allocation = (weight / total_weight) * total_budget if total_weight > 0 else Decimal("0")

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

           current_budget = campaign.dailyBudget
           scale_factor = float(new_budget / current_budget) if current_budget > 0 else 1

           # Diminishing returns after 2x budget
           if scale_factor > 2:
               scale_factor = 2 + (scale_factor - 2) * 0.5

           current_roas = campaign.roas or Decimal("1.0")
           current_conversions = campaign.conversions or 0
           current_revenue = campaign.revenue or Decimal("0")

           return {
               "forecasted_spend": float(new_budget * days),
               "forecasted_conversions": int(current_conversions * scale_factor * (days / 7)),
               "forecasted_revenue": float(current_revenue * Decimal(str(scale_factor)) * (Decimal(str(days)) / Decimal("7"))),
               "forecasted_roas": float(current_roas * Decimal("0.9")),  # Slight decrease due to scale
               "confidence": 0.7 if scale_factor < 1.5 else 0.5,
           }
   ```

#### Deliverables
- [ ] Budget optimizer with ROAS-based decisions
- [ ] Campaign performance forecasting
- [ ] Multi-campaign budget allocation
- [ ] Optimization confidence scoring

---

### Days 11-12: Campaign Creation UI

#### Tasks
1. **Create Campaign Creation Page**
   ```typescript
   // apps/dashboard/app/dashboard/ads/create/page.tsx
   'use client';

   import { useState } from 'react';
   import { useRouter } from 'next/navigation';
   import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
   import { Button } from '@/components/ui/button';
   import { Input } from '@/components/ui/input';
   import { Label } from '@/components/ui/label';
   import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
   import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
   import { Slider } from '@/components/ui/slider';
   import { ArrowLeft, Rocket } from 'lucide-react';
   import Link from 'next/link';

   export default function CreateCampaignPage() {
     const router = useRouter();
     const [step, setStep] = useState(1);
     const [loading, setLoading] = useState(false);

     // Form state
     const [campaignName, setCampaignName] = useState('');
     const [platform, setPlatform] = useState('meta');
     const [objective, setObjective] = useState('conversions');
     const [dailyBudget, setDailyBudget] = useState(50);
     const [duration, setDuration] = useState(7);

     const storeId = 'store_123'; // Get from context

     async function createCampaign() {
       setLoading(true);
       try {
         const response = await fetch(
           `http://localhost:8003/api/v1/ads/campaigns/create`,
           {
             method: 'POST',
             headers: { 'Content-Type': 'application/json' },
             body: JSON.stringify({
               store_id: storeId,
               platform,
               objective,
               daily_budget: dailyBudget,
               duration_days: duration,
               name: campaignName || undefined,
             }),
           }
         );

         if (response.ok) {
           const data = await response.json();
           router.push(`/dashboard/ads/campaigns/${data.campaign_id}`);
         }
       } catch (error) {
         console.error('Failed to create campaign:', error);
       } finally {
         setLoading(false);
       }
     }

     return (
       <div className="container mx-auto py-8 max-w-4xl">
         <div className="mb-8">
           <Link href="/dashboard/ads">
             <Button variant="ghost" size="sm">
               <ArrowLeft className="w-4 h-4 mr-2" />
               Back to Campaigns
             </Button>
           </Link>
           <h1 className="text-3xl font-bold mt-4">Create Ad Campaign</h1>
           <p className="text-muted-foreground">
             Launch an AI-powered advertising campaign
           </p>
         </div>

         <div className="grid gap-6">
           {/* Step 1: Campaign Basics */}
           <Card>
             <CardHeader>
               <CardTitle>Campaign Settings</CardTitle>
               <CardDescription>
                 Configure your campaign objective and budget
               </CardDescription>
             </CardHeader>
             <CardContent className="space-y-6">
               <div>
                 <Label htmlFor="name">Campaign Name (Optional)</Label>
                 <Input
                   id="name"
                   placeholder="Auto-generated if left empty"
                   value={campaignName}
                   onChange={(e) => setCampaignName(e.target.value)}
                 />
               </div>

               <div>
                 <Label htmlFor="platform">Platform</Label>
                 <Select value={platform} onValueChange={setPlatform}>
                   <SelectTrigger>
                     <SelectValue />
                   </SelectTrigger>
                   <SelectContent>
                     <SelectItem value="meta">📱 Meta (Facebook & Instagram)</SelectItem>
                     <SelectItem value="google" disabled>🔍 Google Ads (Coming Soon)</SelectItem>
                     <SelectItem value="tiktok" disabled>🎵 TikTok Ads (Coming Soon)</SelectItem>
                   </SelectContent>
                 </Select>
               </div>

               <div>
                 <Label htmlFor="objective">Campaign Objective</Label>
                 <Select value={objective} onValueChange={setObjective}>
                   <SelectTrigger>
                     <SelectValue />
                   </SelectTrigger>
                   <SelectContent>
                     <SelectItem value="awareness">Brand Awareness</SelectItem>
                     <SelectItem value="traffic">Website Traffic</SelectItem>
                     <SelectItem value="engagement">Engagement</SelectItem>
                     <SelectItem value="conversions">Conversions (Sales)</SelectItem>
                     <SelectItem value="catalog_sales">Catalog Sales</SelectItem>
                   </SelectContent>
                 </Select>
                 <p className="text-sm text-muted-foreground mt-1">
                   {objective === 'conversions' && 'Optimized for purchases and revenue'}
                   {objective === 'traffic' && 'Drive visitors to your store'}
                   {objective === 'awareness' && 'Reach the maximum number of people'}
                   {objective === 'engagement' && 'Maximize likes, comments, and shares'}
                 </p>
               </div>

               <div>
                 <Label>Daily Budget: ${dailyBudget}</Label>
                 <Slider
                   value={[dailyBudget]}
                   onValueChange={(value) => setDailyBudget(value[0])}
                   min={10}
                   max={500}
                   step={5}
                   className="mt-2"
                 />
                 <div className="flex justify-between text-sm text-muted-foreground mt-1">
                   <span>$10</span>
                   <span>$500</span>
                 </div>
               </div>

               <div>
                 <Label>Campaign Duration: {duration} days</Label>
                 <Slider
                   value={[duration]}
                   onValueChange={(value) => setDuration(value[0])}
                   min={7}
                   max={90}
                   step={1}
                   className="mt-2"
                 />
                 <div className="flex justify-between text-sm text-muted-foreground mt-1">
                   <span>7 days</span>
                   <span>90 days</span>
                 </div>
                 <p className="text-sm text-muted-foreground mt-2">
                   Total budget: ${dailyBudget * duration}
                 </p>
               </div>
             </CardContent>
           </Card>

           {/* Campaign Summary */}
           <Card>
             <CardHeader>
               <CardTitle>Campaign Summary</CardTitle>
             </CardHeader>
             <CardContent>
               <div className="space-y-2 text-sm">
                 <div className="flex justify-between">
                   <span className="text-muted-foreground">Platform:</span>
                   <span className="font-medium">Meta (Facebook & Instagram)</span>
                 </div>
                 <div className="flex justify-between">
                   <span className="text-muted-foreground">Objective:</span>
                   <span className="font-medium capitalize">{objective}</span>
                 </div>
                 <div className="flex justify-between">
                   <span className="text-muted-foreground">Daily Budget:</span>
                   <span className="font-medium">${dailyBudget}</span>
                 </div>
                 <div className="flex justify-between">
                   <span className="text-muted-foreground">Duration:</span>
                   <span className="font-medium">{duration} days</span>
                 </div>
                 <div className="flex justify-between border-t pt-2 mt-2">
                   <span className="font-medium">Total Budget:</span>
                   <span className="font-bold">${dailyBudget * duration}</span>
                 </div>
               </div>

               <div className="mt-6 p-4 bg-muted rounded-lg">
                 <p className="text-sm">
                   <strong>What happens next:</strong>
                 </p>
                 <ul className="text-sm mt-2 space-y-1 list-disc list-inside">
                   <li>AI will generate 3 ad creative variations</li>
                   <li>Targeting optimized for your niche</li>
                   <li>Campaign will start in "Paused" status for review</li>
                   <li>You can activate when ready</li>
                 </ul>
               </div>

               <Button
                 className="w-full mt-6"
                 size="lg"
                 onClick={createCampaign}
                 disabled={loading}
               >
                 {loading ? (
                   'Creating Campaign...'
                 ) : (
                   <>
                     <Rocket className="w-4 h-4 mr-2" />
                     Create Campaign
                   </>
                 )}
               </Button>
             </CardContent>
           </Card>
         </div>
       </div>
     );
   }
   ```

2. **Create Campaigns List Page**
   ```typescript
   // apps/dashboard/app/dashboard/ads/page.tsx
   'use client';

   import { useState, useEffect } from 'react';
   import Link from 'next/link';
   import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
   import { Button } from '@/components/ui/button';
   import { Badge } from '@/components/ui/badge';
   import { Plus, TrendingUp, DollarSign, Eye, MousePointerClick } from 'lucide-react';

   interface Campaign {
     id: string;
     name: string;
     platform: string;
     objective: string;
     status: string;
     dailyBudget: number;
     spentAmount: number;
     impressions: number;
     clicks: number;
     conversions: number;
     roas: number;
   }

   export default function AdsCampaignsPage() {
     const [campaigns, setCampaigns] = useState<Campaign[]>([]);
     const [loading, setLoading] = useState(true);

     const storeId = 'store_123';

     useEffect(() => {
       loadCampaigns();
     }, []);

     async function loadCampaigns() {
       try {
         const response = await fetch(
           `http://localhost:8003/api/v1/ads/campaigns?store_id=${storeId}`
         );
         const data = await response.json();
         setCampaigns(data.campaigns || []);
       } catch (error) {
         console.error('Failed to load campaigns:', error);
       } finally {
         setLoading(false);
       }
     }

     const statusColors = {
       DRAFT: 'secondary',
       PENDING_REVIEW: 'warning',
       ACTIVE: 'success',
       PAUSED: 'default',
       COMPLETED: 'secondary',
       REJECTED: 'destructive',
       ERROR: 'destructive',
     };

     const totalSpend = campaigns.reduce((sum, c) => sum + c.spentAmount, 0);
     const totalConversions = campaigns.reduce((sum, c) => sum + c.conversions, 0);
     const avgRoas = campaigns.length > 0
       ? campaigns.reduce((sum, c) => sum + c.roas, 0) / campaigns.length
       : 0;

     return (
       <div className="container mx-auto py-8">
         <div className="flex justify-between items-center mb-8">
           <div>
             <h1 className="text-3xl font-bold">Ad Campaigns</h1>
             <p className="text-muted-foreground">
               Manage your advertising campaigns
             </p>
           </div>

           <Link href="/dashboard/ads/create">
             <Button>
               <Plus className="w-4 h-4 mr-2" />
               Create Campaign
             </Button>
           </Link>
         </div>

         {/* Overview Stats */}
         <div className="grid gap-4 md:grid-cols-4 mb-8">
           <Card>
             <CardHeader className="flex flex-row items-center justify-between pb-2">
               <CardTitle className="text-sm font-medium">Total Spend</CardTitle>
               <DollarSign className="h-4 w-4 text-muted-foreground" />
             </CardHeader>
             <CardContent>
               <div className="text-2xl font-bold">
                 ${totalSpend.toFixed(2)}
               </div>
             </CardContent>
           </Card>

           <Card>
             <CardHeader className="flex flex-row items-center justify-between pb-2">
               <CardTitle className="text-sm font-medium">Conversions</CardTitle>
               <TrendingUp className="h-4 w-4 text-muted-foreground" />
             </CardHeader>
             <CardContent>
               <div className="text-2xl font-bold">{totalConversions}</div>
             </CardContent>
           </Card>

           <Card>
             <CardHeader className="flex flex-row items-center justify-between pb-2">
               <CardTitle className="text-sm font-medium">Average ROAS</CardTitle>
               <TrendingUp className="h-4 w-4 text-muted-foreground" />
             </CardHeader>
             <CardContent>
               <div className="text-2xl font-bold">
                 {avgRoas.toFixed(2)}x
               </div>
             </CardContent>
           </Card>

           <Card>
             <CardHeader className="flex flex-row items-center justify-between pb-2">
               <CardTitle className="text-sm font-medium">Active Campaigns</CardTitle>
             </CardHeader>
             <CardContent>
               <div className="text-2xl font-bold">
                 {campaigns.filter(c => c.status === 'ACTIVE').length}
               </div>
             </CardContent>
           </Card>
         </div>

         {/* Campaigns List */}
         {campaigns.length === 0 && !loading && (
           <Card>
             <CardContent className="py-12 text-center">
               <h3 className="text-xl font-semibold mb-2">No campaigns yet</h3>
               <p className="text-muted-foreground mb-4">
                 Create your first ad campaign to start driving sales
               </p>
               <Link href="/dashboard/ads/create">
                 <Button>Create Campaign</Button>
               </Link>
             </CardContent>
           </Card>
         )}

         <div className="space-y-4">
           {campaigns.map((campaign) => (
             <Card key={campaign.id}>
               <CardContent className="py-6">
                 <div className="flex items-start justify-between">
                   <div className="flex-1">
                     <div className="flex items-center gap-3 mb-2">
                       <h3 className="text-lg font-semibold">{campaign.name}</h3>
                       <Badge variant={statusColors[campaign.status as keyof typeof statusColors]}>
                         {campaign.status}
                       </Badge>
                       <span className="text-sm text-muted-foreground capitalize">
                         {campaign.objective}
                       </span>
                     </div>

                     <div className="grid grid-cols-4 gap-4 mt-4">
                       <div>
                         <p className="text-sm text-muted-foreground">Spend</p>
                         <p className="text-lg font-semibold">
                           ${campaign.spentAmount.toFixed(2)}
                         </p>
                         <p className="text-xs text-muted-foreground">
                           of ${campaign.dailyBudget}/day
                         </p>
                       </div>

                       <div>
                         <p className="text-sm text-muted-foreground">Impressions</p>
                         <p className="text-lg font-semibold">
                           {campaign.impressions.toLocaleString()}
                         </p>
                       </div>

                       <div>
                         <p className="text-sm text-muted-foreground">Conversions</p>
                         <p className="text-lg font-semibold">
                           {campaign.conversions}
                         </p>
                       </div>

                       <div>
                         <p className="text-sm text-muted-foreground">ROAS</p>
                         <p className="text-lg font-semibold">
                           {campaign.roas.toFixed(2)}x
                         </p>
                       </div>
                     </div>
                   </div>

                   <div className="ml-4">
                     <Link href={`/dashboard/ads/campaigns/${campaign.id}`}>
                       <Button variant="outline">View Details</Button>
                     </Link>
                   </div>
                 </div>
               </CardContent>
             </Card>
           ))}
         </div>
       </div>
     );
   }
   ```

3. **Add Navigation**
   ```typescript
   // apps/dashboard/components/layout/sidebar.tsx
   // Add to navigation items:
   {
     title: 'Ad Campaigns',
     href: '/dashboard/ads',
     icon: TrendingUp,
   }
   ```

#### Deliverables
- [ ] Campaign creation UI with budget and objective selection
- [ ] Campaigns list page with overview metrics
- [ ] Campaign status badges and performance display
- [ ] Navigation to ads section

---

### Days 13-14: API Endpoints, Testing & Documentation

#### Tasks
1. **Create Ads Agent API Endpoints**
   ```python
   # app/api/v1/ads.py
   from fastapi import APIRouter, HTTPException, Query
   from typing import Optional
   from decimal import Decimal
   from app.agents.ads.agent import AdsAgent
   from app.agents.ads import CampaignObjective

   router = APIRouter(prefix="/ads", tags=["ads"])
   ads_agent = AdsAgent()

   @router.post("/campaigns/create")
   async def create_campaign(
       store_id: str = Query(...),
       platform: str = Query(...),
       objective: CampaignObjective = Query(...),
       daily_budget: float = Query(..., ge=10, le=10000),
       duration_days: int = Query(7, ge=1, le=90),
       name: Optional[str] = None,
   ):
       """Create a new ad campaign"""
       try:
           campaign = await ads_agent.create_campaign(
               store_id=store_id,
               platform=platform,
               objective=objective,
               daily_budget=Decimal(str(daily_budget)),
               duration_days=duration_days,
               auto_optimize=True,
           )

           return {
               "success": True,
               "campaign_id": campaign.id,
               "name": campaign.name,
               "status": campaign.status,
           }
       except Exception as e:
           raise HTTPException(status_code=400, detail=str(e))

   @router.get("/campaigns")
   async def get_campaigns(
       store_id: str = Query(...),
       status: Optional[str] = None,
   ):
       """Get all campaigns for a store"""
       from app.services.database import prisma

       where = {"storeId": store_id}
       if status:
           where["status"] = status

       campaigns = await prisma.adcampaign.find_many(
           where=where,
           include={"creatives": True},
           order_by={"createdAt": "desc"},
       )

       return {
           "campaigns": [
               {
                   "id": c.id,
                   "name": c.name,
                   "platform": c.platform,
                   "objective": c.objective,
                   "status": c.status,
                   "dailyBudget": float(c.dailyBudget),
                   "spentAmount": float(c.spentAmount),
                   "impressions": c.impressions,
                   "clicks": c.clicks,
                   "conversions": c.conversions,
                   "roas": float(c.roas),
                   "creatives_count": len(c.creatives),
               }
               for c in campaigns
           ]
       }

   @router.get("/campaigns/{campaign_id}")
   async def get_campaign(campaign_id: str):
       """Get campaign details"""
       from app.services.database import prisma

       campaign = await prisma.adcampaign.find_unique(
           where={"id": campaign_id},
           include={"creatives": True},
       )

       if not campaign:
           raise HTTPException(status_code=404, detail="Campaign not found")

       return {
           "id": campaign.id,
           "name": campaign.name,
           "platform": campaign.platform,
           "objective": campaign.objective,
           "status": campaign.status,
           "dailyBudget": float(campaign.dailyBudget),
           "totalBudget": float(campaign.totalBudget) if campaign.totalBudget else None,
           "spentAmount": float(campaign.spentAmount),
           "startDate": campaign.startDate,
           "endDate": campaign.endDate,
           "targeting": campaign.targeting,
           "performance": {
               "impressions": campaign.impressions,
               "clicks": campaign.clicks,
               "conversions": campaign.conversions,
               "revenue": float(campaign.revenue),
               "cpc": float(campaign.cpc),
               "cpm": float(campaign.cpm),
               "ctr": float(campaign.ctr),
               "roas": float(campaign.roas),
           },
           "creatives": [
               {
                   "id": cr.id,
                   "name": cr.name,
                   "type": cr.type,
                   "headline": cr.headline,
                   "status": cr.status,
                   "impressions": cr.impressions,
                   "clicks": cr.clicks,
                   "conversions": cr.conversions,
               }
               for cr in campaign.creatives
           ],
       }

   @router.post("/campaigns/{campaign_id}/activate")
   async def activate_campaign(campaign_id: str):
       """Activate a campaign"""
       from app.agents.ads.campaign_manager import CampaignManager

       manager = CampaignManager()
       await manager.activate_campaign(campaign_id)

       return {"success": True, "message": "Campaign activated"}

   @router.post("/campaigns/{campaign_id}/pause")
   async def pause_campaign(campaign_id: str):
       """Pause a campaign"""
       from app.agents.ads.campaign_manager import CampaignManager

       manager = CampaignManager()
       await manager.pause_campaign(campaign_id)

       return {"success": True, "message": "Campaign paused"}

   @router.post("/campaigns/{campaign_id}/optimize")
   async def optimize_campaign(campaign_id: str):
       """Run optimization on a campaign"""
       result = await ads_agent.optimize_campaign(campaign_id)

       return {
           "success": True,
           "optimization": result,
       }
   ```

2. **Include Ads Router**
   ```python
   # app/api/v1/router.py
   from app.api/v1.ads import router as ads_router

   api_router.include_router(ads_router)
   ```

3. **Write Tests**
   ```python
   # tests/test_ads_agent.py
   import pytest
   from decimal import Decimal
   from app.agents.ads.creative_generator import CreativeGenerator
   from app.agents.ads.budget_optimizer import BudgetOptimizer, BudgetAction

   @pytest.mark.asyncio
   async def test_creative_generator():
       generator = CreativeGenerator()

       # Mock data
       store = type('Store', (), {'niche': 'home decor', 'name': 'Test Store'})()
       products = [
           {'id': '1', 'title': 'Ceramic Vase', 'description': 'Beautiful vase', 'sellingPrice': 29.99}
       ]
       media_assets = [
           {'url': 'https://example.com/image.jpg', 'type': 'IMAGE'}
       ]

       creatives = await generator.generate(
           store=store,
           products=products,
           media_assets=media_assets,
           platform='meta',
           objective='conversions',
           variations=3,
       )

       assert len(creatives) == 3
       assert all(c.headline for c in creatives)
       assert all(c.description for c in creatives)

   def test_budget_optimizer():
       optimizer = BudgetOptimizer()

       # Mock campaign
       campaign = type('Campaign', (), {
           'dailyBudget': Decimal('50'),
           'status': 'ACTIVE',
       })()

       # Test excellent performance
       performance = {
           'spend': 100,
           'revenue': 350,
           'conversions': 10,
           'days_running': 5,
       }

       result = optimizer.optimize(campaign, performance)
       assert result.action == BudgetAction.INCREASE
       assert result.confidence > 0.8

       # Test poor performance
       performance = {
           'spend': 100,
           'revenue': 30,
           'conversions': 1,
           'days_running': 5,
       }

       result = optimizer.optimize(campaign, performance)
       assert result.action == BudgetAction.PAUSE
   ```

4. **Create Documentation**
   ```markdown
   # Ads Agent Core - Documentation

   ## Overview
   The Ads Agent autonomously creates, manages, and optimizes paid advertising campaigns with AI-powered creative generation and budget optimization.

   ## Features

   ### Campaign Management
   - AI-generated ad creatives (3 variations per campaign)
   - Automated targeting based on store niche
   - Budget optimization with ROAS tracking
   - Multi-objective support (awareness, traffic, conversions, etc.)

   ### Supported Platforms
   - ✅ Meta (Facebook & Instagram) Ads
   - 🚧 Google Ads (Sprint 5)
   - 🚧 TikTok Ads (Sprint 5)

   ### Budget Optimization
   - ROAS-based decision making
   - Automatic budget scaling for winners
   - Pause underperforming campaigns
   - Multi-campaign budget allocation

   ## API Reference

   ### Create Campaign
   ```
   POST /api/v1/ads/campaigns/create
   Query Parameters:
     - store_id: string (required)
     - platform: string (required) - "meta", "google", "tiktok"
     - objective: string (required) - "awareness", "traffic", "conversions"
     - daily_budget: number (10-10000)
     - duration_days: number (1-90, default: 7)
     - name: string (optional)
   ```

   ### Get Campaigns
   ```
   GET /api/v1/ads/campaigns?store_id={id}&status={status}
   ```

   ### Get Campaign Details
   ```
   GET /api/v1/ads/campaigns/{campaign_id}
   ```

   ### Activate/Pause Campaign
   ```
   POST /api/v1/ads/campaigns/{campaign_id}/activate
   POST /api/v1/ads/campaigns/{campaign_id}/pause
   ```

   ### Optimize Campaign
   ```
   POST /api/v1/ads/campaigns/{campaign_id}/optimize
   ```

   ## Usage Example

   ```python
   from app.agents.ads.agent import AdsAgent
   from app.agents.ads import CampaignObjective
   from decimal import Decimal

   agent = AdsAgent()

   # Create campaign
   campaign = await agent.create_campaign(
       store_id="store_123",
       platform="meta",
       objective=CampaignObjective.CONVERSIONS,
       daily_budget=Decimal("50"),
       duration_days=7,
   )

   print(f"Campaign created: {campaign.name}")
   print(f"Budget: ${campaign.dailyBudget}/day")
   print(f"Creatives: {len(campaign.creatives)}")
   ```

   ## Budget Optimization Thresholds

   | ROAS | Action | Description |
   |------|--------|-------------|
   | ≥3.0x | Increase 20% | Excellent - scale aggressively |
   | ≥2.0x | Increase 10% | Good - gradual scaling |
   | ≥1.0x | Maintain | Acceptable - monitor |
   | <1.0x | Decrease 20% | Suboptimal - reduce spend |
   | <0.5x | Pause | Poor - stop losses |

   ## Creative Generation

   The AI generates 3 creative variations testing different angles:
   1. **Variation 1**: Benefit-focused
   2. **Variation 2**: Social proof / urgency
   3. **Variation 3**: Problem-solution

   Each creative includes:
   - Attention-grabbing headline (≤40 chars)
   - Compelling ad copy (≤125 chars)
   - Call-to-action button
   - Optimized media selection
   ```

#### Deliverables
- [ ] API endpoints for campaign management
- [ ] Unit tests for creative generator and optimizer
- [ ] Integration tests for campaign creation
- [ ] Complete documentation

---

## Definition of Done

### Technical Requirements
- [ ] Campaign manager creating campaigns on Meta
- [ ] Creative generator producing 3 variations
- [ ] Meta Ads client fully functional
- [ ] Budget optimizer making ROAS-based decisions
- [ ] All database models migrated
- [ ] API endpoints working
- [ ] Tests passing (>80% coverage)

### UI Requirements
- [ ] Campaign creation page functional
- [ ] Campaigns list showing performance
- [ ] Campaign activation/pause working
- [ ] Budget and objective selection working

### Functional Requirements
- [ ] End-to-end campaign creation working
- [ ] Campaigns launching on Meta Ads
- [ ] Performance tracking updating correctly
- [ ] Budget optimization logic sound
- [ ] Creative variations unique and compelling

---

## Testing Checklist

### Unit Tests
- [ ] Creative generator produces valid creatives
- [ ] Budget optimizer calculates correct actions
- [ ] Campaign manager formats requests properly
- [ ] Meta Ads client handles API correctly

### Integration Tests
- [ ] Full campaign creation flow
- [ ] Campaign launch to Meta (requires test ad account)
- [ ] Performance sync from Meta
- [ ] Budget adjustments applied
- [ ] Creative pause/activation

### Manual Testing
- [ ] Create campaign via UI
- [ ] Verify campaign in database
- [ ] Check Meta Ads Manager for campaign
- [ ] Activate campaign and verify status
- [ ] Monitor performance updates

---

## Success Metrics

- Campaign creation completes in < 3 minutes
- AI creatives pass manual quality review
- Budget optimizer makes logical decisions
- Campaigns launch successfully to Meta 95%+ of time
- ROAS tracking accurate within 5%
- UI loads and responds smoothly

---

## Next Sprint Preview

**Sprint 5: Ads Agent Advanced** will add:
- Google Ads platform integration
- TikTok Ads platform integration
- Advanced A/B testing system
- Multi-campaign budget allocation
- Comprehensive ads dashboard
- Automated optimization workers

---

This sprint establishes the **core autonomous advertising engine** that will drive paid traffic and sales for Zyenta stores with AI-powered campaign management and optimization.
