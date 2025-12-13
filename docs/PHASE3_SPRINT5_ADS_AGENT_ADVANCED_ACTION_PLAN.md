# Phase 3 - Sprint 5: Ads Agent Advanced - Action Plan

## Sprint Overview

**Duration:** 2 weeks
**Team Size:** 3-4 developers (2 backend, 2 frontend)
**Sprint Goal:** Extend the Ads Agent with Google Ads and TikTok Ads platform support, implement advanced A/B testing, multi-campaign budget allocation, comprehensive ads dashboard, and automated optimization workers.

### Objectives
- [ ] Add Google Ads platform client
- [ ] Add TikTok Ads platform client
- [ ] Implement advanced A/B testing system
- [ ] Build multi-campaign budget allocation
- [ ] Create comprehensive ads dashboard
- [ ] Implement automated optimization workers
- [ ] Add campaign analytics and reporting

### Prerequisites
- Phase 3 Sprint 4 (Ads Agent Core) complete
- Meta Ads integration working
- Budget optimizer functional
- Creative generator operational
- OAuth configured for Google and TikTok

---

## Day-by-Day Breakdown

### Days 1-3: Google Ads Platform Client

#### Tasks
1. **Implement Google Ads Platform Client**
   ```python
   # app/agents/ads/platforms/google.py
   from typing import Dict, Any, List, Optional
   from datetime import datetime
   from google.ads.googleads.client import GoogleAdsClient
   from google.ads.googleads.errors import GoogleAdsException
   from .base import BaseAdsPlatformClient

   class GoogleAdsClient(BaseAdsPlatformClient):
       """Google Ads API client"""

       def __init__(self, tokens: Dict[str, Any], customer_id: str):
           self.refresh_token = tokens["refresh_token"]
           self.customer_id = customer_id

           # Initialize Google Ads client
           credentials = {
               "developer_token": settings.GOOGLE_ADS_DEVELOPER_TOKEN,
               "client_id": settings.GOOGLE_CLIENT_ID,
               "client_secret": settings.GOOGLE_CLIENT_SECRET,
               "refresh_token": self.refresh_token,
               "login_customer_id": customer_id,
           }

           self.client = GoogleAdsClient.load_from_dict(credentials)

       async def create_campaign(
           self,
           name: str,
           objective: str,
           daily_budget: int,  # in micros (1 million micros = $1)
           status: str = "PAUSED",
       ) -> Dict[str, Any]:
           """Create a Google Ads campaign"""

           campaign_service = self.client.get_service("CampaignService")
           campaign_operation = self.client.get_type("CampaignOperation")

           campaign = campaign_operation.create
           campaign.name = name
           campaign.advertising_channel_type = self._map_objective_to_channel(objective)
           campaign.status = self._map_status(status)

           # Set budget
           campaign.campaign_budget = self._create_budget(name, daily_budget)

           # Campaign settings
           campaign.network_settings.target_google_search = True
           campaign.network_settings.target_search_network = True
           campaign.network_settings.target_content_network = True

           try:
               response = campaign_service.mutate_campaigns(
                   customer_id=self.customer_id,
                   operations=[campaign_operation],
               )

               resource_name = response.results[0].resource_name
               campaign_id = resource_name.split("/")[-1]

               return {
                   "id": campaign_id,
                   "name": name,
                   "status": status,
                   "resource_name": resource_name,
               }

           except GoogleAdsException as ex:
               raise Exception(f"Google Ads error: {ex.error.message}")

       def _create_budget(self, name: str, amount_micros: int) -> str:
           """Create campaign budget"""

           budget_service = self.client.get_service("CampaignBudgetService")
           budget_operation = self.client.get_type("CampaignBudgetOperation")

           budget = budget_operation.create
           budget.name = f"{name} Budget"
           budget.amount_micros = amount_micros
           budget.delivery_method = self.client.enums.BudgetDeliveryMethodEnum.STANDARD

           response = budget_service.mutate_campaign_budgets(
               customer_id=self.customer_id,
               operations=[budget_operation],
           )

           return response.results[0].resource_name

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
           """Create an ad group (Google's equivalent of ad set)"""

           ad_group_service = self.client.get_service("AdGroupService")
           ad_group_operation = self.client.get_type("AdGroupOperation")

           ad_group = ad_group_operation.create
           ad_group.name = name
           ad_group.campaign = f"customers/{self.customer_id}/campaigns/{campaign_id}"
           ad_group.status = self.client.enums.AdGroupStatusEnum.PAUSED
           ad_group.type_ = self.client.enums.AdGroupTypeEnum.SEARCH_STANDARD

           # Set bidding strategy
           ad_group.cpc_bid_micros = daily_budget // 10  # Rough estimate

           # Apply targeting
           if targeting:
               await self._apply_targeting(ad_group, targeting)

           try:
               response = ad_group_service.mutate_ad_groups(
                   customer_id=self.customer_id,
                   operations=[ad_group_operation],
               )

               resource_name = response.results[0].resource_name
               ad_group_id = resource_name.split("/")[-1]

               return {
                   "id": ad_group_id,
                   "name": name,
                   "resource_name": resource_name,
               }

           except GoogleAdsException as ex:
               raise Exception(f"Google Ads error: {ex.error.message}")

       async def create_ad_creative(
           self,
           name: str,
           page_id: str,  # Not used for Google
           image_url: Optional[str] = None,
           video_id: Optional[str] = None,
           headline: str = "",
           description: str = "",
           call_to_action: str = "LEARN_MORE",
           link_url: str = "",
       ) -> Dict[str, Any]:
           """Create responsive search ad"""

           # For Google Ads, we'll create this when creating the actual ad
           return {
               "id": "creative_placeholder",
               "name": name,
               "headline": headline,
               "description": description,
               "link_url": link_url,
           }

       async def create_ad(
           self,
           name: str,
           ad_set_id: str,  # Ad group ID
           creative_id: str,
           status: str = "PAUSED",
       ) -> Dict[str, Any]:
           """Create a responsive search ad"""

           ad_service = self.client.get_service("AdGroupAdService")
           ad_operation = self.client.get_type("AdGroupAdOperation")

           ad_group_ad = ad_operation.create
           ad_group_ad.ad_group = f"customers/{self.customer_id}/adGroups/{ad_set_id}"
           ad_group_ad.status = self._map_status(status)

           # Create responsive search ad
           ad = ad_group_ad.ad
           ad.final_urls.append(creative_id.get("link_url", "https://example.com"))

           # Add headlines
           headline1 = self.client.get_type("AdTextAsset")
           headline1.text = creative_id.get("headline", "Default Headline")[:30]
           ad.responsive_search_ad.headlines.append(headline1)

           # Add descriptions
           description1 = self.client.get_type("AdTextAsset")
           description1.text = creative_id.get("description", "Default description")[:90]
           ad.responsive_search_ad.descriptions.append(description1)

           try:
               response = ad_service.mutate_ad_group_ads(
                   customer_id=self.customer_id,
                   operations=[ad_operation],
               )

               resource_name = response.results[0].resource_name
               ad_id = resource_name.split("/")[-1]

               return {
                   "id": ad_id,
                   "name": name,
                   "resource_name": resource_name,
               }

           except GoogleAdsException as ex:
               raise Exception(f"Google Ads error: {ex.error.message}")

       async def get_campaign_insights(
           self,
           campaign_id: str,
           start_date: datetime,
           end_date: datetime,
       ) -> Dict[str, Any]:
           """Get campaign performance insights"""

           ga_service = self.client.get_service("GoogleAdsService")

           query = f"""
               SELECT
                 campaign.id,
                 metrics.cost_micros,
                 metrics.impressions,
                 metrics.clicks,
                 metrics.conversions,
                 metrics.conversions_value,
                 metrics.average_cpc,
                 metrics.average_cpm,
                 metrics.ctr
               FROM campaign
               WHERE campaign.id = {campaign_id}
                 AND segments.date BETWEEN '{start_date.strftime('%Y-%m-%d')}'
                 AND '{end_date.strftime('%Y-%m-%d')}'
           """

           try:
               response = ga_service.search(
                   customer_id=self.customer_id,
                   query=query,
               )

               total_cost = 0
               total_impressions = 0
               total_clicks = 0
               total_conversions = 0
               total_revenue = 0

               for row in response:
                   total_cost += row.metrics.cost_micros / 1_000_000
                   total_impressions += row.metrics.impressions
                   total_clicks += row.metrics.clicks
                   total_conversions += row.metrics.conversions
                   total_revenue += row.metrics.conversions_value

               roas = total_revenue / total_cost if total_cost > 0 else 0

               return {
                   "spend": total_cost,
                   "impressions": total_impressions,
                   "clicks": total_clicks,
                   "conversions": int(total_conversions),
                   "revenue": total_revenue,
                   "cpc": total_cost / total_clicks if total_clicks > 0 else 0,
                   "cpm": (total_cost / total_impressions * 1000) if total_impressions > 0 else 0,
                   "ctr": (total_clicks / total_impressions * 100) if total_impressions > 0 else 0,
                   "roas": roas,
               }

           except GoogleAdsException as ex:
               raise Exception(f"Google Ads error: {ex.error.message}")

       async def update_campaign_status(
           self,
           campaign_id: str,
           status: str,
       ):
           """Update campaign status"""

           campaign_service = self.client.get_service("CampaignService")
           campaign_operation = self.client.get_type("CampaignOperation")

           campaign = campaign_operation.update
           campaign.resource_name = f"customers/{self.customer_id}/campaigns/{campaign_id}"
           campaign.status = self._map_status(status)

           campaign_operation.update_mask.paths.append("status")

           campaign_service.mutate_campaigns(
               customer_id=self.customer_id,
               operations=[campaign_operation],
           )

       async def update_ad_set_budget(
           self,
           ad_set_id: str,
           daily_budget: int,
       ):
           """Update ad group CPC bid"""

           ad_group_service = self.client.get_service("AdGroupService")
           ad_group_operation = self.client.get_type("AdGroupOperation")

           ad_group = ad_group_operation.update
           ad_group.resource_name = f"customers/{self.customer_id}/adGroups/{ad_set_id}"
           ad_group.cpc_bid_micros = daily_budget

           ad_group_operation.update_mask.paths.append("cpc_bid_micros")

           ad_group_service.mutate_ad_groups(
               customer_id=self.customer_id,
               operations=[ad_group_operation],
           )

       async def update_ad_set_status(
           self,
           ad_set_id: str,
           status: str,
       ):
           """Update ad group status"""

           ad_group_service = self.client.get_service("AdGroupService")
           ad_group_operation = self.client.get_type("AdGroupOperation")

           ad_group = ad_group_operation.update
           ad_group.resource_name = f"customers/{self.customer_id}/adGroups/{ad_set_id}"
           ad_group.status = self._map_status(status)

           ad_group_operation.update_mask.paths.append("status")

           ad_group_service.mutate_ad_groups(
               customer_id=self.customer_id,
               operations=[ad_group_operation],
           )

       def _map_objective_to_channel(self, objective: str):
           """Map objective to advertising channel"""
           mapping = {
               "awareness": self.client.enums.AdvertisingChannelTypeEnum.DISPLAY,
               "traffic": self.client.enums.AdvertisingChannelTypeEnum.SEARCH,
               "engagement": self.client.enums.AdvertisingChannelTypeEnum.VIDEO,
               "conversions": self.client.enums.AdvertisingChannelTypeEnum.SEARCH,
               "catalog_sales": self.client.enums.AdvertisingChannelTypeEnum.SHOPPING,
           }
           return mapping.get(objective, self.client.enums.AdvertisingChannelTypeEnum.SEARCH)

       def _map_status(self, status: str):
           """Map status to Google Ads status enum"""
           mapping = {
               "ACTIVE": self.client.enums.CampaignStatusEnum.ENABLED,
               "PAUSED": self.client.enums.CampaignStatusEnum.PAUSED,
               "ENABLED": self.client.enums.CampaignStatusEnum.ENABLED,
           }
           return mapping.get(status, self.client.enums.CampaignStatusEnum.PAUSED)

       async def _apply_targeting(self, ad_group: Any, targeting: Dict[str, Any]):
           """Apply targeting criteria to ad group"""
           # Implementation for demographics, locations, keywords, etc.
           pass
   ```

2. **Update Platform Factory**
   ```python
   # app/agents/ads/platforms/__init__.py (update)
   from .google import GoogleAdsClient

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
       elif platform == "google":
           if not ad_account_id:
               raise ValueError("Google Ads requires customer_id")
           return GoogleAdsClient(tokens, ad_account_id)
       else:
           raise ValueError(f"Unsupported ads platform: {platform}")
   ```

3. **Add Google Ads Dependencies**
   ```txt
   # services/growth-engine/requirements.txt (add)
   google-ads==23.0.0
   ```

#### Deliverables
- [ ] Google Ads client fully implemented
- [ ] Campaign creation working on Google Ads
- [ ] Performance insights retrieval functional
- [ ] Budget and status updates working

---

### Days 4-6: TikTok Ads Platform Client

#### Tasks
1. **Implement TikTok Ads Platform Client**
   ```python
   # app/agents/ads/platforms/tiktok_ads.py
   from typing import Dict, Any, List, Optional
   from datetime import datetime
   import httpx
   from .base import BaseAdsPlatformClient

   class TikTokAdsClient(BaseAdsPlatformClient):
       """TikTok Ads API client"""

       BASE_URL = "https://business-api.tiktok.com/open_api/v1.3"

       def __init__(self, tokens: Dict[str, Any], advertiser_id: str):
           self.access_token = tokens["access_token"]
           self.advertiser_id = advertiser_id

       async def _request(
           self,
           method: str,
           endpoint: str,
           **kwargs,
       ) -> Dict[str, Any]:
           """Make authenticated API request"""

           headers = {
               "Access-Token": self.access_token,
               "Content-Type": "application/json",
           }

           async with httpx.AsyncClient() as client:
               response = await client.request(
                   method,
                   f"{self.BASE_URL}{endpoint}",
                   headers=headers,
                   timeout=30.0,
                   **kwargs,
               )
               response.raise_for_status()
               data = response.json()

               if data.get("code") != 0:
                   raise Exception(f"TikTok Ads API error: {data.get('message')}")

               return data.get("data", {})

       async def create_campaign(
           self,
           name: str,
           objective: str,
           daily_budget: int,  # in cents
           status: str = "PAUSED",
       ) -> Dict[str, Any]:
           """Create a TikTok Ads campaign"""

           data = await self._request(
               "POST",
               "/campaign/create/",
               json={
                   "advertiser_id": self.advertiser_id,
                   "campaign_name": name,
                   "objective_type": self._map_objective(objective),
                   "budget_mode": "BUDGET_MODE_DAY",
                   "budget": daily_budget / 100,  # Convert cents to dollars
                   "operation_status": "DISABLE" if status == "PAUSED" else "ENABLE",
               },
           )

           return {
               "id": data["campaign_id"],
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
           """Create an ad group (TikTok's ad set)"""

           request_data = {
               "advertiser_id": self.advertiser_id,
               "campaign_id": campaign_id,
               "adgroup_name": name,
               "placement_type": "PLACEMENT_TYPE_AUTOMATIC",
               "placements": ["PLACEMENT_TIKTOK"],
               "budget_mode": "BUDGET_MODE_DAY",
               "budget": daily_budget / 100,
               "optimization_goal": self._map_optimization_goal(optimization_goal),
               "billing_event": "CPC",
               "bid_type": "BID_TYPE_NO_BID",  # Auto bid
               "operation_status": "DISABLE",
           }

           # Apply targeting
           if targeting:
               request_data.update(self._format_targeting(targeting))

           # Set schedule
           if start_time:
               request_data["schedule_start_time"] = start_time.strftime("%Y-%m-%d %H:%M:%S")
           if end_time:
               request_data["schedule_end_time"] = end_time.strftime("%Y-%m-%d %H:%M:%S")

           data = await self._request(
               "POST",
               "/adgroup/create/",
               json=request_data,
           )

           return {
               "id": data["adgroup_id"],
               "name": name,
           }

       async def create_ad_creative(
           self,
           name: str,
           page_id: str,  # Not used for TikTok
           image_url: Optional[str] = None,
           video_id: Optional[str] = None,
           headline: str = "",
           description: str = "",
           call_to_action: str = "LEARN_MORE",
           link_url: str = "",
       ) -> Dict[str, Any]:
           """Upload creative to TikTok"""

           # Upload video if provided
           if video_id:
               video_upload_id = await self._upload_video(video_id)
           elif image_url:
               video_upload_id = await self._upload_image(image_url)
           else:
               raise ValueError("Either video_id or image_url is required")

           return {
               "id": video_upload_id,
               "name": name,
               "headline": headline,
               "description": description,
               "call_to_action": call_to_action,
               "link_url": link_url,
           }

       async def create_ad(
           self,
           name: str,
           ad_set_id: str,
           creative_id: str,
           status: str = "PAUSED",
       ) -> Dict[str, Any]:
           """Create a TikTok ad"""

           data = await self._request(
               "POST",
               "/ad/create/",
               json={
                   "advertiser_id": self.advertiser_id,
                   "adgroup_id": ad_set_id,
                   "ad_name": name,
                   "ad_text": creative_id.get("description", ""),
                   "call_to_action": creative_id.get("call_to_action", "LEARN_MORE"),
                   "landing_page_url": creative_id.get("link_url", ""),
                   "video_id": creative_id.get("id"),  # Video upload ID
                   "display_name": name,
                   "operation_status": "DISABLE" if status == "PAUSED" else "ENABLE",
               },
           )

           return {
               "id": data["ad_id"],
               "name": name,
           }

       async def get_campaign_insights(
           self,
           campaign_id: str,
           start_date: datetime,
           end_date: datetime,
       ) -> Dict[str, Any]:
           """Get campaign performance insights"""

           data = await self._request(
               "GET",
               "/reports/integrated/get/",
               params={
                   "advertiser_id": self.advertiser_id,
                   "report_type": "BASIC",
                   "dimensions": ["campaign_id"],
                   "data_level": "AUCTION_CAMPAIGN",
                   "filters": [{"field": "campaign_id", "operator": "IN", "value": [campaign_id]}],
                   "metrics": [
                       "spend",
                       "impressions",
                       "clicks",
                       "cpc",
                       "cpm",
                       "ctr",
                       "conversion",
                       "total_purchase_value",
                   ],
                   "start_date": start_date.strftime("%Y-%m-%d"),
                   "end_date": end_date.strftime("%Y-%m-%d"),
               },
           )

           if not data.get("list"):
               return {}

           metrics = data["list"][0]["metrics"]

           spend = float(metrics.get("spend", 0))
           revenue = float(metrics.get("total_purchase_value", 0))
           roas = revenue / spend if spend > 0 else 0

           return {
               "spend": spend,
               "impressions": int(metrics.get("impressions", 0)),
               "clicks": int(metrics.get("clicks", 0)),
               "conversions": int(metrics.get("conversion", 0)),
               "revenue": revenue,
               "cpc": float(metrics.get("cpc", 0)),
               "cpm": float(metrics.get("cpm", 0)),
               "ctr": float(metrics.get("ctr", 0)),
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
               "/campaign/update/status/",
               json={
                   "advertiser_id": self.advertiser_id,
                   "campaign_ids": [campaign_id],
                   "operation_status": "ENABLE" if status == "ACTIVE" else "DISABLE",
               },
           )

       async def update_ad_set_budget(
           self,
           ad_set_id: str,
           daily_budget: int,
       ):
           """Update ad group budget"""

           await self._request(
               "POST",
               "/adgroup/update/",
               json={
                   "advertiser_id": self.advertiser_id,
                   "adgroup_id": ad_set_id,
                   "budget": daily_budget / 100,
               },
           )

       async def update_ad_set_status(
           self,
           ad_set_id: str,
           status: str,
       ):
           """Update ad group status"""

           await self._request(
               "POST",
               "/adgroup/update/status/",
               json={
                   "advertiser_id": self.advertiser_id,
                   "adgroup_ids": [ad_set_id],
                   "operation_status": "ENABLE" if status == "ACTIVE" else "DISABLE",
               },
           )

       async def _upload_video(self, video_url: str) -> str:
           """Upload video to TikTok"""

           data = await self._request(
               "POST",
               "/file/video/ad/upload/",
               json={
                   "advertiser_id": self.advertiser_id,
                   "video_url": video_url,
                   "upload_type": "UPLOAD_BY_URL",
               },
           )

           return data["video_id"]

       async def _upload_image(self, image_url: str) -> str:
           """Upload image to TikTok"""

           data = await self._request(
               "POST",
               "/file/image/ad/upload/",
               json={
                   "advertiser_id": self.advertiser_id,
                   "image_url": image_url,
                   "upload_type": "UPLOAD_BY_URL",
               },
           )

           return data["image_id"]

       def _map_objective(self, objective: str) -> str:
           """Map objective to TikTok objective"""
           mapping = {
               "awareness": "REACH",
               "traffic": "TRAFFIC",
               "engagement": "VIDEO_VIEWS",
               "conversions": "WEB_CONVERSIONS",
               "catalog_sales": "PRODUCT_SALES",
           }
           return mapping.get(objective, "TRAFFIC")

       def _map_optimization_goal(self, goal: str) -> str:
           """Map optimization goal to TikTok goal"""
           mapping = {
               "REACH": "REACH",
               "LINK_CLICKS": "CLICK",
               "ENGAGEMENT": "ENGAGEMENT",
               "CONVERSIONS": "CONVERT",
           }
           return mapping.get(goal, "CLICK")

       def _format_targeting(self, targeting: Dict[str, Any]) -> Dict[str, Any]:
           """Format targeting for TikTok API"""
           formatted = {}

           if "locations" in targeting:
               formatted["location_ids"] = targeting["locations"]

           if "demographics" in targeting:
               demo = targeting["demographics"]
               if "age_groups" in demo:
                   formatted["age_groups"] = demo["age_groups"]
               if "gender" in demo:
                   formatted["gender"] = demo["gender"]

           if "interests" in targeting:
               formatted["interest_category_ids"] = targeting["interests"]

           return formatted
   ```

2. **Update Platform Factory**
   ```python
   # app/agents/ads/platforms/__init__.py (update)
   from .tiktok_ads import TikTokAdsClient

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
       elif platform == "google":
           if not ad_account_id:
               raise ValueError("Google Ads requires customer_id")
           return GoogleAdsClient(tokens, ad_account_id)
       elif platform == "tiktok":
           if not ad_account_id:
               raise ValueError("TikTok Ads requires advertiser_id")
           return TikTokAdsClient(tokens, ad_account_id)
       else:
           raise ValueError(f"Unsupported ads platform: {platform}")
   ```

#### Deliverables
- [ ] TikTok Ads client implemented
- [ ] Campaign creation working on TikTok
- [ ] Video/image upload functional
- [ ] Performance tracking working

---

### Days 7-9: A/B Testing & Multi-Campaign Budget Allocation

#### Tasks
1. **Implement A/B Testing System**
   ```python
   # app/agents/ads/ab_tester.py
   from typing import List, Dict, Any
   from decimal import Decimal
   from dataclasses import dataclass
   import statistics

   @dataclass
   class ABTestResult:
       winning_creative_ids: List[str]
       losing_creative_ids: List[str]
       pause_creative_ids: List[str]
       scale_creative_ids: List[str]
       confidence: float
       summary: str

   class ABTester:
       """Advanced A/B testing for ad creatives"""

       # Statistical significance thresholds
       MIN_SAMPLE_SIZE = 100  # Minimum impressions needed
       CONFIDENCE_LEVEL = 0.90  # 90% confidence
       MIN_PERFORMANCE_DIFF = 0.20  # 20% performance difference

       def __init__(self):
           pass

       async def analyze(
           self,
           campaign_id: str,
           creatives: List[Any],
           performance: Dict[str, Any],
       ) -> ABTestResult:
           """Analyze creative performance and determine winners/losers"""

           # Calculate metrics for each creative
           creative_metrics = []
           for creative in creatives:
               if creative.impressions < self.MIN_SAMPLE_SIZE:
                   # Not enough data yet
                   continue

               metrics = self._calculate_metrics(creative)
               creative_metrics.append({
                   "id": creative.id,
                   "metrics": metrics,
                   "creative": creative,
               })

           if len(creative_metrics) < 2:
               # Need at least 2 creatives with enough data
               return ABTestResult(
                   winning_creative_ids=[],
                   losing_creative_ids=[],
                   pause_creative_ids=[],
                   scale_creative_ids=[],
                   confidence=0.0,
                   summary="Not enough data for A/B testing",
               )

           # Find best performing creative
           creative_metrics.sort(key=lambda x: x["metrics"]["ctr"], reverse=True)
           winner = creative_metrics[0]

           # Determine statistical significance
           winners = []
           losers = []
           pause = []
           scale = []

           for cm in creative_metrics:
               if cm["id"] == winner["id"]:
                   winners.append(cm["id"])
                   # Scale winner if performing well
                   if cm["metrics"]["roas"] > 2.0:
                       scale.append(cm["id"])
                   continue

               # Check if significantly worse than winner
               performance_diff = self._calculate_performance_diff(
                   winner["metrics"],
                   cm["metrics"],
               )

               if performance_diff > self.MIN_PERFORMANCE_DIFF:
                   losers.append(cm["id"])
                   # Pause if significantly underperforming
                   if performance_diff > 0.5 or cm["metrics"]["roas"] < 1.0:
                       pause.append(cm["id"])

           confidence = self._calculate_confidence(creative_metrics)

           summary = f"Winner: Creative with {winner['metrics']['ctr']:.2f}% CTR. "
           summary += f"{len(pause)} creatives recommended for pause. "
           summary += f"{len(scale)} creatives recommended for scaling."

           return ABTestResult(
               winning_creative_ids=winners,
               losing_creative_ids=losers,
               pause_creative_ids=pause,
               scale_creative_ids=scale,
               confidence=confidence,
               summary=summary,
           )

       def _calculate_metrics(self, creative: Any) -> Dict[str, float]:
           """Calculate performance metrics for a creative"""

           impressions = creative.impressions or 1
           clicks = creative.clicks or 0
           conversions = creative.conversions or 0
           spend = float(creative.spend or 0)
           revenue = float(creative.revenue or 0)

           ctr = (clicks / impressions) * 100
           cvr = (conversions / clicks * 100) if clicks > 0 else 0
           cpc = spend / clicks if clicks > 0 else 0
           cpa = spend / conversions if conversions > 0 else 999
           roas = revenue / spend if spend > 0 else 0

           return {
               "ctr": ctr,
               "cvr": cvr,
               "cpc": cpc,
               "cpa": cpa,
               "roas": roas,
               "impressions": impressions,
               "clicks": clicks,
               "conversions": conversions,
           }

       def _calculate_performance_diff(
           self,
           winner_metrics: Dict,
           challenger_metrics: Dict,
       ) -> float:
           """Calculate performance difference between two creatives"""

           # Use CTR as primary metric
           winner_ctr = winner_metrics["ctr"]
           challenger_ctr = challenger_metrics["ctr"]

           if winner_ctr == 0:
               return 0.0

           diff = (winner_ctr - challenger_ctr) / winner_ctr
           return abs(diff)

       def _calculate_confidence(self, creative_metrics: List[Dict]) -> float:
           """Calculate statistical confidence of results"""

           if len(creative_metrics) < 2:
               return 0.0

           # Use CTR variance as confidence indicator
           ctrs = [cm["metrics"]["ctr"] for cm in creative_metrics]

           if len(ctrs) < 2:
               return 0.5

           std_dev = statistics.stdev(ctrs)
           mean_ctr = statistics.mean(ctrs)

           if mean_ctr == 0:
               return 0.5

           # Lower variance = higher confidence
           coefficient_of_variation = std_dev / mean_ctr
           confidence = max(0.5, 1.0 - coefficient_of_variation)

           return min(confidence, 0.95)
   ```

2. **Implement Multi-Campaign Budget Allocation**
   ```python
   # app/agents/ads/budget_allocator.py
   from typing import List, Dict
   from decimal import Decimal

   class BudgetAllocator:
       """Intelligent budget allocation across multiple campaigns"""

       def __init__(self):
           pass

       async def allocate(
           self,
           total_budget: Decimal,
           campaigns: List[Any],
       ) -> Dict[str, Decimal]:
           """Allocate budget across campaigns based on performance"""

           allocations = {}
           total_weight = Decimal("0")

           # Calculate performance weights
           weights = {}
           for campaign in campaigns:
               if campaign.status != "ACTIVE":
                   allocations[campaign.id] = Decimal("0")
                   continue

               weight = self._calculate_weight(campaign)
               weights[campaign.id] = weight
               total_weight += weight

           # Allocate proportionally
           for campaign in campaigns:
               if campaign.status != "ACTIVE":
                   continue

               if total_weight == 0:
                   # Equal distribution if no performance data
                   allocation = total_budget / len([c for c in campaigns if c.status == "ACTIVE"])
               else:
                   weight = weights.get(campaign.id, Decimal("0"))
                   allocation = (weight / total_weight) * total_budget

               # Apply constraints
               allocation = self._apply_constraints(
                   allocation,
                   campaign,
                   total_budget,
               )

               allocations[campaign.id] = allocation

           return allocations

       def _calculate_weight(self, campaign: Any) -> Decimal:
           """Calculate performance weight for campaign"""

           roas = campaign.roas or Decimal("1.0")
           conversions = campaign.conversions or 0

           # Weight by ROAS and conversion volume
           # Higher ROAS = more budget
           # More conversions = more confidence
           weight = roas * Decimal(str(min(conversions, 100) + 1))

           return max(weight, Decimal("0.1"))

       def _apply_constraints(
           self,
           allocation: Decimal,
           campaign: Any,
           total_budget: Decimal,
       ) -> Decimal:
           """Apply min/max budget constraints"""

           # Minimum budget per campaign
           min_budget = Decimal("10.0")

           # Maximum budget (no more than 50% of total)
           max_budget = total_budget * Decimal("0.5")

           # Current campaign's max (don't increase by more than 50%)
           current_max = campaign.dailyBudget * Decimal("1.5")

           allocation = max(allocation, min_budget)
           allocation = min(allocation, max_budget)
           allocation = min(allocation, current_max)

           return allocation

       async def rebalance(
           self,
           campaigns: List[Any],
           total_budget: Decimal,
       ) -> Dict[str, Dict[str, Decimal]]:
           """Rebalance budgets based on current performance"""

           allocations = await self.allocate(total_budget, campaigns)

           changes = {}
           for campaign in campaigns:
               if campaign.id not in allocations:
                   continue

               new_budget = allocations[campaign.id]
               old_budget = campaign.dailyBudget
               change = new_budget - old_budget

               changes[campaign.id] = {
                   "old_budget": old_budget,
                   "new_budget": new_budget,
                   "change": change,
                   "change_percent": (change / old_budget * 100) if old_budget > 0 else Decimal("0"),
               }

           return changes
   ```

#### Deliverables
- [ ] A/B testing system analyzing creative performance
- [ ] Statistical significance calculations
- [ ] Multi-campaign budget allocator
- [ ] Budget rebalancing logic

---

### Days 10-12: Comprehensive Ads Dashboard

#### Tasks
1. **Build Ads Dashboard UI**
   ```typescript
   // app/dashboard/ads/page.tsx
   'use client';

   import { useState, useEffect } from 'react';
   import { Card } from '@/components/ui/card';
   import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
   import CampaignList from '@/components/ads/CampaignList';
   import CampaignDetails from '@/components/ads/CampaignDetails';
   import AdCreativeManager from '@/components/ads/AdCreativeManager';
   import BudgetOptimizer from '@/components/ads/BudgetOptimizer';
   import PerformanceCharts from '@/components/ads/PerformanceCharts';

   export default function AdsPage() {
     const [selectedCampaign, setSelectedCampaign] = useState<string | null>(null);
     const [campaigns, setCampaigns] = useState([]);

     useEffect(() => {
       fetchCampaigns();
     }, []);

     const fetchCampaigns = async () => {
       const response = await fetch('/api/ads/campaigns');
       const data = await response.json();
       setCampaigns(data.campaigns);
     };

     return (
       <div className="container mx-auto p-6">
         <h1 className="text-3xl font-bold mb-6">Ads Manager</h1>

         <Tabs defaultValue="campaigns" className="w-full">
           <TabsList>
             <TabsTrigger value="campaigns">Campaigns</TabsTrigger>
             <TabsTrigger value="creatives">Creatives</TabsTrigger>
             <TabsTrigger value="optimization">Optimization</TabsTrigger>
             <TabsTrigger value="analytics">Analytics</TabsTrigger>
           </TabsList>

           <TabsContent value="campaigns">
             <div className="grid grid-cols-3 gap-6">
               <div className="col-span-1">
                 <CampaignList
                   campaigns={campaigns}
                   onSelect={setSelectedCampaign}
                 />
               </div>
               <div className="col-span-2">
                 {selectedCampaign && (
                   <CampaignDetails campaignId={selectedCampaign} />
                 )}
               </div>
             </div>
           </TabsContent>

           <TabsContent value="creatives">
             <AdCreativeManager />
           </TabsContent>

           <TabsContent value="optimization">
             <BudgetOptimizer campaigns={campaigns} />
           </TabsContent>

           <TabsContent value="analytics">
             <PerformanceCharts campaigns={campaigns} />
           </TabsContent>
         </Tabs>
       </div>
     );
   }
   ```

2. **Campaign Performance Charts**
   ```typescript
   // components/ads/PerformanceCharts.tsx
   'use client';

   import { Line, Bar } from 'recharts';
   import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
   import { ResponsiveContainer, LineChart, BarChart, XAxis, YAxis, CartesianGrid, Tooltip, Legend } from 'recharts';

   interface PerformanceChartsProps {
     campaigns: any[];
   }

   export default function PerformanceCharts({ campaigns }: PerformanceChartsProps) {
     const [performanceData, setPerformanceData] = useState([]);
     const [dateRange, setDateRange] = useState('7d');

     useEffect(() => {
       fetchPerformanceData();
     }, [campaigns, dateRange]);

     const fetchPerformanceData = async () => {
       const response = await fetch(`/api/ads/performance?range=${dateRange}`);
       const data = await response.json();
       setPerformanceData(data.daily_metrics);
     };

     return (
       <div className="space-y-6">
         {/* ROAS Trend */}
         <Card>
           <CardHeader>
             <CardTitle>ROAS Trend</CardTitle>
           </CardHeader>
           <CardContent>
             <ResponsiveContainer width="100%" height={300}>
               <LineChart data={performanceData}>
                 <CartesianGrid strokeDasharray="3 3" />
                 <XAxis dataKey="date" />
                 <YAxis />
                 <Tooltip />
                 <Legend />
                 <Line type="monotone" dataKey="roas" stroke="#8884d8" />
               </LineChart>
             </ResponsiveContainer>
           </CardContent>
         </Card>

         {/* Spend vs Revenue */}
         <Card>
           <CardHeader>
             <CardTitle>Spend vs Revenue</CardTitle>
           </CardHeader>
           <CardContent>
             <ResponsiveContainer width="100%" height={300}>
               <BarChart data={performanceData}>
                 <CartesianGrid strokeDasharray="3 3" />
                 <XAxis dataKey="date" />
                 <YAxis />
                 <Tooltip />
                 <Legend />
                 <Bar dataKey="spend" fill="#ef4444" />
                 <Bar dataKey="revenue" fill="#22c55e" />
               </BarChart>
             </ResponsiveContainer>
           </CardContent>
         </Card>

         {/* Campaign Comparison */}
         <Card>
           <CardHeader>
             <CardTitle>Campaign Performance Comparison</CardTitle>
           </CardHeader>
           <CardContent>
             <div className="overflow-x-auto">
               <table className="min-w-full divide-y divide-gray-200">
                 <thead>
                   <tr>
                     <th>Campaign</th>
                     <th>Platform</th>
                     <th>Spend</th>
                     <th>Revenue</th>
                     <th>ROAS</th>
                     <th>Conversions</th>
                     <th>Status</th>
                   </tr>
                 </thead>
                 <tbody>
                   {campaigns.map((campaign) => (
                     <tr key={campaign.id}>
                       <td>{campaign.name}</td>
                       <td>{campaign.platform}</td>
                       <td>${campaign.spend.toFixed(2)}</td>
                       <td>${campaign.revenue.toFixed(2)}</td>
                       <td>{campaign.roas.toFixed(2)}x</td>
                       <td>{campaign.conversions}</td>
                       <td>
                         <span className={`badge ${campaign.status === 'ACTIVE' ? 'bg-green-500' : 'bg-gray-500'}`}>
                           {campaign.status}
                         </span>
                       </td>
                     </tr>
                   ))}
                 </tbody>
               </table>
             </div>
           </CardContent>
         </Card>
       </div>
     );
   }
   ```

3. **Ad Creative A/B Testing UI**
   ```typescript
   // components/ads/AdCreativeManager.tsx
   'use client';

   import { useState } from 'react';
   import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
   import { Button } from '@/components/ui/button';
   import { Badge } from '@/components/ui/badge';

   export default function AdCreativeManager() {
     const [creatives, setCreatives] = useState([]);
     const [abTests, setAbTests] = useState([]);

     const runAbTest = async (campaignId: string) => {
       const response = await fetch('/api/ads/ab-test', {
         method: 'POST',
         headers: { 'Content-Type': 'application/json' },
         body: JSON.stringify({ campaign_id: campaignId }),
       });

       const data = await response.json();
       setAbTests([...abTests, data.test]);
     };

     return (
       <div className="space-y-6">
         {/* Creative Performance */}
         <Card>
           <CardHeader>
             <CardTitle>Creative Performance</CardTitle>
           </CardHeader>
           <CardContent>
             <div className="grid grid-cols-3 gap-4">
               {creatives.map((creative) => (
                 <div key={creative.id} className="border rounded-lg p-4">
                   <img src={creative.image_url} alt={creative.name} className="w-full h-48 object-cover rounded" />
                   <h3 className="font-bold mt-2">{creative.name}</h3>
                   <div className="grid grid-cols-2 gap-2 mt-2 text-sm">
                     <div>
                       <span className="text-gray-500">CTR:</span> {creative.ctr.toFixed(2)}%
                     </div>
                     <div>
                       <span className="text-gray-500">CPC:</span> ${creative.cpc.toFixed(2)}
                     </div>
                     <div>
                       <span className="text-gray-500">Conversions:</span> {creative.conversions}
                     </div>
                     <div>
                       <span className="text-gray-500">ROAS:</span> {creative.roas.toFixed(2)}x
                     </div>
                   </div>
                   {creative.is_winner && (
                     <Badge className="mt-2 bg-green-500">Winner</Badge>
                   )}
                 </div>
               ))}
             </div>
           </CardContent>
         </Card>

         {/* A/B Test Results */}
         <Card>
           <CardHeader>
             <CardTitle>A/B Test Results</CardTitle>
           </CardHeader>
           <CardContent>
             {abTests.map((test) => (
               <div key={test.id} className="border-b pb-4 mb-4">
                 <h4 className="font-semibold">{test.campaign_name}</h4>
                 <p className="text-sm text-gray-500">{test.summary}</p>
                 <div className="mt-2">
                   <span className="text-sm">Confidence: {(test.confidence * 100).toFixed(1)}%</span>
                 </div>
                 <div className="mt-2 space-x-2">
                   {test.pause_creative_ids.length > 0 && (
                     <Button size="sm" variant="destructive">
                       Pause {test.pause_creative_ids.length} Underperforming
                     </Button>
                   )}
                   {test.scale_creative_ids.length > 0 && (
                     <Button size="sm" variant="default">
                       Scale {test.scale_creative_ids.length} Winners
                     </Button>
                   )}
                 </div>
               </div>
             ))}
           </CardContent>
         </Card>
       </div>
     );
   }
   ```

#### Deliverables
- [ ] Complete ads dashboard UI
- [ ] Performance charts and analytics
- [ ] Creative A/B testing interface
- [ ] Campaign comparison views

---

### Days 13-14: Automated Optimization Workers & Testing

#### Tasks
1. **Implement Automated Optimization Worker**
   ```python
   # app/workers/ads_optimization_worker.py
   from celery import shared_task
   from datetime import datetime, timedelta
   from app.agents.ads import CampaignManager, BudgetOptimizer, ABTester
   from app.db.prisma import prisma

   @shared_task
   def optimize_all_campaigns():
       """Run optimization for all active campaigns"""

       # Fetch all active campaigns
       campaigns = prisma.adcampaign.find_many(
           where={"status": "ACTIVE"},
           include={
               "store": True,
               "creatives": True,
           }
       )

       for campaign in campaigns:
           try:
               optimize_campaign.delay(campaign.id)
           except Exception as e:
               print(f"Error queuing optimization for campaign {campaign.id}: {e}")

   @shared_task
   async def optimize_campaign(campaign_id: str):
       """Optimize a single campaign"""

       campaign = await prisma.adcampaign.find_unique(
           where={"id": campaign_id},
           include={
               "store": True,
               "creatives": True,
           }
       )

       if not campaign:
           return

       # Get campaign manager
       manager = CampaignManager()
       optimizer = BudgetOptimizer()
       ab_tester = ABTester()

       # Fetch latest insights
       insights = await manager.get_insights(
           campaign_id=campaign_id,
           start_date=datetime.now() - timedelta(days=7),
           end_date=datetime.now(),
       )

       # Update campaign metrics
       await prisma.adcampaign.update(
           where={"id": campaign_id},
           data={
               "spend": insights["spend"],
               "impressions": insights["impressions"],
               "clicks": insights["clicks"],
               "conversions": insights["conversions"],
               "revenue": insights["revenue"],
               "roas": insights["roas"],
           }
       )

       # Run A/B testing on creatives
       if len(campaign.creatives) > 1:
           ab_result = await ab_tester.analyze(
               campaign_id=campaign_id,
               creatives=campaign.creatives,
               performance=insights,
           )

           # Pause underperforming creatives
           if ab_result.pause_creative_ids:
               await manager.pause_creatives(ab_result.pause_creative_ids)

           # Scale winning creatives
           if ab_result.scale_creative_ids:
               await manager.scale_creatives(ab_result.scale_creative_ids)

       # Optimize budget
       action = await optimizer.optimize(campaign)

       if action.action == "increase_budget":
           await manager.update_budget(
               campaign_id=campaign_id,
               new_budget=action.new_budget,
           )
       elif action.action == "decrease_budget":
           await manager.update_budget(
               campaign_id=campaign_id,
               new_budget=action.new_budget,
           )
       elif action.action == "pause_campaign":
           await manager.pause_campaign(campaign_id)

       # Log optimization
       await prisma.campaignoptimizationlog.create(
           data={
               "campaignId": campaign_id,
               "action": action.action,
               "previousBudget": campaign.dailyBudget,
               "newBudget": action.new_budget,
               "roas": insights["roas"],
               "reason": action.reason,
           }
       )

   @shared_task
   async def rebalance_budgets(store_id: str):
       """Rebalance budgets across all campaigns for a store"""

       store = await prisma.store.find_unique(
           where={"id": store_id},
           include={
               "campaigns": {
                   "where": {"status": "ACTIVE"}
               }
           }
       )

       if not store or not store.campaigns:
           return

       allocator = BudgetAllocator()

       # Get total budget
       total_budget = sum(c.dailyBudget for c in store.campaigns)

       # Rebalance
       changes = await allocator.rebalance(
           campaigns=store.campaigns,
           total_budget=total_budget,
       )

       # Apply changes
       manager = CampaignManager()
       for campaign_id, change in changes.items():
           if abs(change["change_percent"]) > 10:  # Only apply if >10% change
               await manager.update_budget(
                   campaign_id=campaign_id,
                   new_budget=change["new_budget"],
               )
   ```

2. **Configure Celery Beat Schedule**
   ```python
   # app/celery_config.py (update)
   from celery.schedules import crontab

   beat_schedule = {
       # Existing schedules...

       # Optimize campaigns every 4 hours
       'optimize-campaigns': {
           'task': 'app.workers.ads_optimization_worker.optimize_all_campaigns',
           'schedule': crontab(minute=0, hour='*/4'),
       },

       # Rebalance budgets daily at 3 AM
       'rebalance-budgets': {
           'task': 'app.workers.ads_optimization_worker.rebalance_budgets',
           'schedule': crontab(hour=3, minute=0),
       },
   }
   ```

3. **Add API Endpoints**
   ```python
   # app/api/ads/optimization.py
   from fastapi import APIRouter, Depends
   from app.auth import get_current_user
   from app.workers.ads_optimization_worker import optimize_campaign

   router = APIRouter()

   @router.post("/optimize/{campaign_id}")
   async def trigger_optimization(
       campaign_id: str,
       user = Depends(get_current_user),
   ):
       """Manually trigger campaign optimization"""

       # Queue optimization job
       task = optimize_campaign.delay(campaign_id)

       return {
           "message": "Optimization queued",
           "task_id": task.id,
       }

   @router.post("/ab-test")
   async def run_ab_test(
       campaign_id: str,
       user = Depends(get_current_user),
   ):
       """Run A/B test on campaign creatives"""

       from app.agents.ads import ABTester

       ab_tester = ABTester()

       campaign = await prisma.adcampaign.find_unique(
           where={"id": campaign_id},
           include={"creatives": True}
       )

       result = await ab_tester.analyze(
           campaign_id=campaign_id,
           creatives=campaign.creatives,
           performance={},
       )

       return {
           "test": {
               "id": campaign_id,
               "campaign_name": campaign.name,
               "winning_creative_ids": result.winning_creative_ids,
               "pause_creative_ids": result.pause_creative_ids,
               "scale_creative_ids": result.scale_creative_ids,
               "confidence": result.confidence,
               "summary": result.summary,
           }
       }
   ```

#### Deliverables
- [ ] Automated optimization workers
- [ ] Celery beat schedule configured
- [ ] Manual optimization triggers
- [ ] A/B test automation

---

## Definition of Done

### Technical Requirements
- [ ] Google Ads client fully functional
- [ ] TikTok Ads client operational
- [ ] A/B testing system analyzing creatives
- [ ] Multi-campaign budget allocation working
- [ ] Comprehensive ads dashboard deployed
- [ ] Automated optimization workers running
- [ ] All API endpoints documented

### Testing Requirements
- [ ] Unit tests for all platform clients
- [ ] Integration tests for A/B testing
- [ ] Budget allocation tests
- [ ] Dashboard UI tested across browsers
- [ ] Worker tests with mocked tasks

### Documentation
- [ ] API documentation updated
- [ ] Platform client usage guides
- [ ] Dashboard user guide
- [ ] Optimization logic documented

---

## Testing Checklist

### Google Ads Integration
- [ ] Create campaign on Google Ads
- [ ] Create ad groups with targeting
- [ ] Upload responsive search ads
- [ ] Fetch campaign insights
- [ ] Update campaign budgets
- [ ] Pause/activate campaigns

### TikTok Ads Integration
- [ ] Create TikTok campaign
- [ ] Upload video creative
- [ ] Create ad with video
- [ ] Fetch performance metrics
- [ ] Update ad group budgets
- [ ] Status management

### A/B Testing
- [ ] Test with 2 creatives (minimal)
- [ ] Test with 5+ creatives
- [ ] Verify statistical significance
- [ ] Test winner selection
- [ ] Test pause recommendations
- [ ] Test scale recommendations

### Budget Allocation
- [ ] Allocate across 3+ campaigns
- [ ] Test with different ROAS values
- [ ] Verify min/max constraints
- [ ] Test rebalancing logic
- [ ] Test equal distribution fallback

### Dashboard
- [ ] View all campaigns
- [ ] View campaign details
- [ ] Performance charts load
- [ ] A/B test results display
- [ ] Manual optimization trigger works
- [ ] Creative manager functional

### Automated Workers
- [ ] Optimization worker runs
- [ ] Budget rebalancing executes
- [ ] Logs created correctly
- [ ] Error handling works
- [ ] Retry logic functional

---

## Success Metrics

### Platform Coverage
- **Target:** Google Ads and TikTok Ads fully integrated
- **Measurement:** Successful campaign creation on both platforms

### A/B Testing Accuracy
- **Target:** 90%+ confidence in winner detection
- **Measurement:** Statistical confidence scores

### Budget Optimization
- **Target:** 15% improvement in overall ROAS
- **Measurement:** Compare ROAS before/after optimization

### Automation
- **Target:** 100% of campaigns optimized automatically
- **Measurement:** Worker execution logs

### Dashboard Usage
- **Target:** Users can manage entire ad lifecycle from UI
- **Measurement:** User feedback and task completion

---

## Next Sprint Preview

### Phase 3 - Sprint 6: Integration & Polish

**Focus Areas:**
1. **Cross-Agent Integration**
   - Social Agent → Ads Agent workflow
   - Automated product promotion
   - Content-to-ad pipeline

2. **Advanced Analytics**
   - Unified dashboard
   - ROI attribution
   - Predictive insights

3. **System Optimization**
   - Performance tuning
   - Cost optimization
   - Error handling improvements

4. **Production Readiness**
   - Security audit
   - Load testing
   - Monitoring setup
   - Documentation finalization

**Prerequisites:**
- All Phase 3 sprints (1-5) completed
- All platform integrations working
- Basic automation functional
