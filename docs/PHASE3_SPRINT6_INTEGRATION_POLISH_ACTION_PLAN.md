# Phase 3 - Sprint 6: Integration & Polish - Action Plan

## Sprint Overview

**Duration:** 2 weeks
**Team Size:** 4-5 developers (2 backend, 2 frontend, 1 DevOps)
**Sprint Goal:** Integrate all Phase 3 components (OAuth, Social Agent, Ads Agent) into a unified Growth Hub, build cross-agent workflows, implement advanced analytics, optimize system performance, and prepare for production deployment.

### Objectives
- [ ] Build unified Growth Hub dashboard
- [ ] Implement cross-agent workflows (Social → Ads)
- [ ] Create advanced analytics and ROI attribution
- [ ] Build predictive insights system
- [ ] Optimize system performance and costs
- [ ] Complete security audit
- [ ] Set up monitoring and alerting
- [ ] Finalize documentation

### Prerequisites
- Phase 3 Sprints 1-5 completed
- OAuth system functional
- Social Agent operational (TikTok, Instagram, Pinterest, Twitter)
- Ads Agent operational (Meta, Google, TikTok)
- All platform integrations tested

---

## Day-by-Day Breakdown

### Days 1-3: Unified Growth Hub Dashboard

#### Tasks
1. **Build Growth Hub Main Dashboard**
   ```typescript
   // app/dashboard/growth/page.tsx
   'use client';

   import { useState, useEffect } from 'react';
   import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
   import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
   import OverviewMetrics from '@/components/growth/OverviewMetrics';
   import PlatformConnections from '@/components/growth/PlatformConnections';
   import ChannelPerformance from '@/components/growth/ChannelPerformance';
   import ROIAnalytics from '@/components/growth/ROIAnalytics';

   export default function GrowthHubPage() {
     const [overview, setOverview] = useState<any>(null);
     const [loading, setLoading] = useState(true);

     useEffect(() => {
       fetchOverview();
     }, []);

     const fetchOverview = async () => {
       const response = await fetch('/api/growth/overview');
       const data = await response.json();
       setOverview(data);
       setLoading(false);
     };

     return (
       <div className="container mx-auto p-6">
         <div className="flex justify-between items-center mb-6">
           <h1 className="text-3xl font-bold">Growth Hub</h1>
           <PlatformConnections />
         </div>

         {/* Key Metrics Overview */}
         <OverviewMetrics overview={overview} />

         {/* Detailed Tabs */}
         <Tabs defaultValue="overview" className="mt-6">
           <TabsList>
             <TabsTrigger value="overview">Overview</TabsTrigger>
             <TabsTrigger value="social">Social Media</TabsTrigger>
             <TabsTrigger value="ads">Paid Ads</TabsTrigger>
             <TabsTrigger value="analytics">Analytics</TabsTrigger>
             <TabsTrigger value="automation">Automation</TabsTrigger>
           </TabsList>

           <TabsContent value="overview">
             <ChannelPerformance data={overview?.channels} />
           </TabsContent>

           <TabsContent value="social">
             <SocialMediaManager />
           </TabsContent>

           <TabsContent value="ads">
             <AdsManager />
           </TabsContent>

           <TabsContent value="analytics">
             <ROIAnalytics />
           </TabsContent>

           <TabsContent value="automation">
             <AutomationWorkflows />
           </TabsContent>
         </Tabs>
       </div>
     );
   }
   ```

2. **Overview Metrics Component**
   ```typescript
   // components/growth/OverviewMetrics.tsx
   'use client';

   import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
   import { ArrowUpIcon, ArrowDownIcon } from '@radix-ui/react-icons';

   interface OverviewMetricsProps {
     overview: any;
   }

   export default function OverviewMetrics({ overview }: OverviewMetricsProps) {
     if (!overview) return null;

     const metrics = [
       {
         label: 'Total Revenue',
         value: `$${overview.total_revenue.toFixed(2)}`,
         change: overview.revenue_change,
         trend: overview.revenue_trend,
       },
       {
         label: 'Marketing Spend',
         value: `$${overview.total_spend.toFixed(2)}`,
         change: overview.spend_change,
         trend: overview.spend_trend,
       },
       {
         label: 'Overall ROAS',
         value: `${overview.overall_roas.toFixed(2)}x`,
         change: overview.roas_change,
         trend: overview.roas_trend,
       },
       {
         label: 'Total Conversions',
         value: overview.total_conversions.toLocaleString(),
         change: overview.conversions_change,
         trend: overview.conversions_trend,
       },
       {
         label: 'Social Engagement',
         value: overview.total_engagement.toLocaleString(),
         change: overview.engagement_change,
         trend: overview.engagement_trend,
       },
       {
         label: 'Total Reach',
         value: overview.total_reach.toLocaleString(),
         change: overview.reach_change,
         trend: overview.reach_trend,
       },
     ];

     return (
       <div className="grid grid-cols-3 gap-4">
         {metrics.map((metric, idx) => (
           <Card key={idx}>
             <CardHeader className="pb-2">
               <CardTitle className="text-sm font-medium text-gray-500">
                 {metric.label}
               </CardTitle>
             </CardHeader>
             <CardContent>
               <div className="text-2xl font-bold">{metric.value}</div>
               <div className="flex items-center mt-1 text-sm">
                 {metric.trend === 'up' ? (
                   <ArrowUpIcon className="text-green-500 mr-1" />
                 ) : (
                   <ArrowDownIcon className="text-red-500 mr-1" />
                 )}
                 <span className={metric.trend === 'up' ? 'text-green-500' : 'text-red-500'}>
                   {Math.abs(metric.change).toFixed(1)}%
                 </span>
                 <span className="text-gray-500 ml-1">vs last week</span>
               </div>
             </CardContent>
           </Card>
         ))}
       </div>
     );
   }
   ```

3. **Platform Connections Manager**
   ```typescript
   // components/growth/PlatformConnections.tsx
   'use client';

   import { useState, useEffect } from 'react';
   import { Button } from '@/components/ui/button';
   import { Badge } from '@/components/ui/badge';
   import {
     DropdownMenu,
     DropdownMenuContent,
     DropdownMenuItem,
     DropdownMenuTrigger,
   } from '@/components/ui/dropdown-menu';

   export default function PlatformConnections() {
     const [connections, setConnections] = useState<any[]>([]);

     useEffect(() => {
       fetchConnections();
     }, []);

     const fetchConnections = async () => {
       const response = await fetch('/api/oauth/connections');
       const data = await response.json();
       setConnections(data.connections);
     };

     const connectPlatform = async (platform: string) => {
       const response = await fetch('/api/oauth/authorize', {
         method: 'POST',
         headers: { 'Content-Type': 'application/json' },
         body: JSON.stringify({ platform }),
       });

       const data = await response.json();
       window.location.href = data.authorization_url;
     };

     const connectedPlatforms = connections.filter(c => c.status === 'active');
     const availablePlatforms = ['tiktok', 'instagram', 'pinterest', 'twitter', 'meta', 'google'];

     return (
       <div className="flex items-center gap-2">
         <div className="flex gap-1">
           {connectedPlatforms.map((connection) => (
             <Badge key={connection.id} variant="default" className="capitalize">
               {connection.platform}
             </Badge>
           ))}
         </div>

         <DropdownMenu>
           <DropdownMenuTrigger asChild>
             <Button variant="outline" size="sm">
               + Connect Platform
             </Button>
           </DropdownMenuTrigger>
           <DropdownMenuContent>
             {availablePlatforms
               .filter(p => !connectedPlatforms.find(c => c.platform === p))
               .map((platform) => (
                 <DropdownMenuItem
                   key={platform}
                   onClick={() => connectPlatform(platform)}
                   className="capitalize"
                 >
                   {platform}
                 </DropdownMenuItem>
               ))}
           </DropdownMenuContent>
         </DropdownMenu>
       </div>
     );
   }
   ```

#### Deliverables
- [ ] Unified Growth Hub dashboard
- [ ] Overview metrics display
- [ ] Platform connection manager
- [ ] Navigation between Social/Ads sections

---

### Days 4-6: Cross-Agent Workflows

#### Tasks
1. **Content-to-Ad Pipeline**
   ```python
   # app/agents/growth/content_to_ad_pipeline.py
   from typing import List, Dict, Any
   from datetime import datetime
   from app.agents.social import ContentPlanner
   from app.agents.ads import CampaignManager, CreativeGenerator

   class ContentToAdPipeline:
       """Pipeline to promote top-performing social content as paid ads"""

       def __init__(self):
           self.content_planner = ContentPlanner()
           self.campaign_manager = CampaignManager()
           self.creative_generator = CreativeGenerator()

       async def identify_top_performers(
           self,
           store_id: str,
           days: int = 7,
           min_engagement: int = 1000,
       ) -> List[Any]:
           """Find top-performing social posts to promote"""

           from app.db.prisma import prisma

           # Get recent posts with high engagement
           posts = await prisma.socialpost.find_many(
               where={
                   "storeId": store_id,
                   "publishedAt": {
                       "gte": datetime.now() - timedelta(days=days),
                   },
                   "status": "published",
               },
               order_by={"engagement": "desc"},
           )

           # Filter by engagement threshold
           top_posts = [p for p in posts if p.engagement >= min_engagement]

           return top_posts

       async def promote_to_ads(
           self,
           post_id: str,
           budget: int,
           platforms: List[str] = ["meta", "tiktok"],
       ) -> Dict[str, Any]:
           """Convert a high-performing social post into a paid ad campaign"""

           from app.db.prisma import prisma

           # Get post details
           post = await prisma.socialpost.find_unique(
               where={"id": post_id},
               include={"store": True},
           )

           if not post:
               raise ValueError(f"Post {post_id} not found")

           # Create campaigns on each platform
           campaigns = []

           for platform in platforms:
               # Generate ad creative from social post
               creative = await self.creative_generator.generate_from_social_post(
                   platform=platform,
                   image_url=post.imageUrl,
                   video_url=post.videoUrl,
                   caption=post.caption,
                   product_url=post.linkUrl,
               )

               # Create campaign
               campaign = await self.campaign_manager.create_campaign(
                   store_id=post.storeId,
                   name=f"Promoted: {post.caption[:50]}...",
                   platform=platform,
                   objective="conversions",
                   daily_budget=budget,
                   creative_ids=[creative.id],
               )

               campaigns.append(campaign)

               # Link post to campaign
               await prisma.socialpost.update(
                   where={"id": post_id},
                   data={
                       "promotedAsCampaignIds": {
                           "push": campaign.id,
                       }
                   }
               )

           return {
               "post_id": post_id,
               "campaigns": campaigns,
               "total_budget": budget * len(platforms),
           }

       async def auto_promote_workflow(
           self,
           store_id: str,
           budget_per_campaign: int = 2000,  # $20
           platforms: List[str] = ["meta"],
       ):
           """Automatically promote top performers"""

           # Find top posts
           top_posts = await self.identify_top_performers(
               store_id=store_id,
               days=7,
               min_engagement=1000,
           )

           results = []

           # Promote top 3 posts
           for post in top_posts[:3]:
               # Check if already promoted
               if post.promotedAsCampaignIds:
                   continue

               try:
                   result = await self.promote_to_ads(
                       post_id=post.id,
                       budget=budget_per_campaign,
                       platforms=platforms,
                   )
                   results.append(result)
               except Exception as e:
                   print(f"Error promoting post {post.id}: {e}")

           return results
   ```

2. **Product Launch Automation**
   ```python
   # app/agents/growth/product_launch_automation.py
   from typing import Dict, Any
   from app.agents.social import ContentPlanner, SocialPublisher
   from app.agents.ads import CampaignManager

   class ProductLaunchAutomation:
       """Automated multi-channel product launch workflow"""

       def __init__(self):
           self.content_planner = ContentPlanner()
           self.social_publisher = SocialPublisher()
           self.campaign_manager = CampaignManager()

       async def launch_product(
           self,
           store_id: str,
           product_id: str,
           social_budget_days: int = 7,
           ad_budget: int = 5000,  # $50/day
       ) -> Dict[str, Any]:
           """Execute full product launch across social and paid channels"""

           from app.db.prisma import prisma

           # Get product details
           product = await prisma.product.find_unique(
               where={"id": product_id},
               include={"store": True},
           )

           if not product:
               raise ValueError(f"Product {product_id} not found")

           launch_results = {}

           # Step 1: Create social media content calendar
           social_posts = await self.content_planner.create_product_campaign(
               product_id=product_id,
               days=social_budget_days,
               posts_per_day=2,
               platforms=["tiktok", "instagram", "pinterest"],
           )

           launch_results["social_posts"] = len(social_posts)

           # Step 2: Schedule organic posts
           for post in social_posts:
               await self.social_publisher.schedule_post(post)

           # Step 3: Create paid ad campaigns
           campaigns = []

           for platform in ["meta", "google"]:
               campaign = await self.campaign_manager.create_product_campaign(
                   store_id=store_id,
                   product_id=product_id,
                   platform=platform,
                   daily_budget=ad_budget,
                   duration_days=14,
               )
               campaigns.append(campaign)

           launch_results["ad_campaigns"] = campaigns

           # Step 4: Create launch tracking record
           launch = await prisma.productlaunch.create(
               data={
                   "productId": product_id,
                   "storeId": store_id,
                   "socialPosts": social_posts,
                   "adCampaigns": campaigns,
                   "status": "active",
                   "launchedAt": datetime.now(),
               }
           )

           launch_results["launch_id"] = launch.id

           return launch_results
   ```

3. **Celery Worker for Automation**
   ```python
   # app/workers/growth_automation_worker.py
   from celery import shared_task
   from app.agents.growth import ContentToAdPipeline, ProductLaunchAutomation

   @shared_task
   async def auto_promote_top_content():
       """Automatically promote top-performing content to ads"""

       from app.db.prisma import prisma

       # Get all active stores
       stores = await prisma.store.find_many(
           where={"status": "active"}
       )

       pipeline = ContentToAdPipeline()

       for store in stores:
           try:
               await pipeline.auto_promote_workflow(
                   store_id=store.id,
                   budget_per_campaign=2000,
                   platforms=["meta"],
               )
           except Exception as e:
               print(f"Error auto-promoting for store {store.id}: {e}")

   @shared_task
   async def monitor_product_launches():
       """Monitor and optimize active product launches"""

       from app.db.prisma import prisma

       launches = await prisma.productlaunch.find_many(
           where={"status": "active"}
       )

       for launch in launches:
           # Check performance and adjust budgets
           # Implementation here...
           pass
   ```

#### Deliverables
- [ ] Content-to-ad pipeline functional
- [ ] Product launch automation working
- [ ] Automated workflows running
- [ ] Cross-agent coordination tested

---

### Days 7-9: Advanced Analytics & ROI Attribution

#### Tasks
1. **ROI Attribution System**
   ```python
   # app/analytics/roi_attribution.py
   from typing import Dict, Any, List
   from datetime import datetime, timedelta
   from decimal import Decimal

   class ROIAttributionEngine:
       """Multi-touch attribution for revenue across channels"""

       def __init__(self):
           pass

       async def calculate_attribution(
           self,
           store_id: str,
           start_date: datetime,
           end_date: datetime,
       ) -> Dict[str, Any]:
           """Calculate revenue attribution across all marketing channels"""

           from app.db.prisma import prisma

           # Get all orders in period
           orders = await prisma.order.find_many(
               where={
                   "storeId": store_id,
                   "createdAt": {
                       "gte": start_date,
                       "lte": end_date,
                   },
                   "status": "completed",
               }
           )

           # Get all marketing activities
           social_posts = await prisma.socialpost.find_many(
               where={
                   "storeId": store_id,
                   "publishedAt": {
                       "gte": start_date - timedelta(days=30),  # Look back 30 days
                       "lte": end_date,
                   },
                   "status": "published",
               }
           )

           ad_campaigns = await prisma.adcampaign.find_many(
               where={
                   "storeId": store_id,
                   "createdAt": {
                       "gte": start_date - timedelta(days=30),
                       "lte": end_date,
                   },
                   "status": {"in": ["ACTIVE", "PAUSED"]},
               }
           )

           # Attribute revenue
           attribution = {
               "total_revenue": Decimal("0"),
               "channels": {
                   "organic_social": Decimal("0"),
                   "paid_ads": Decimal("0"),
                   "direct": Decimal("0"),
                   "other": Decimal("0"),
               },
               "platform_breakdown": {},
           }

           for order in orders:
               revenue = Decimal(str(order.total))
               attribution["total_revenue"] += revenue

               # Check referrer/utm params
               if order.utmSource:
                   if order.utmMedium == "social":
                       attribution["channels"]["organic_social"] += revenue
                       platform = order.utmSource
                       attribution["platform_breakdown"][platform] = \
                           attribution["platform_breakdown"].get(platform, Decimal("0")) + revenue
                   elif order.utmMedium == "cpc":
                       attribution["channels"]["paid_ads"] += revenue
                       platform = order.utmSource
                       attribution["platform_breakdown"][platform] = \
                           attribution["platform_breakdown"].get(platform, Decimal("0")) + revenue
                   else:
                       attribution["channels"]["other"] += revenue
               else:
                   attribution["channels"]["direct"] += revenue

           # Calculate ROI for each channel
           attribution["roi"] = {}

           # Social ROI (organic - minimal cost)
           attribution["roi"]["organic_social"] = {
               "revenue": float(attribution["channels"]["organic_social"]),
               "cost": 0,  # Organic has no direct cost
               "roi": float("inf") if attribution["channels"]["organic_social"] > 0 else 0,
           }

           # Paid Ads ROI
           total_ad_spend = sum(c.spend for c in ad_campaigns)
           ad_revenue = attribution["channels"]["paid_ads"]
           attribution["roi"]["paid_ads"] = {
               "revenue": float(ad_revenue),
               "cost": float(total_ad_spend),
               "roi": float((ad_revenue - Decimal(str(total_ad_spend))) / Decimal(str(total_ad_spend))) if total_ad_spend > 0 else 0,
               "roas": float(ad_revenue / Decimal(str(total_ad_spend))) if total_ad_spend > 0 else 0,
           }

           return attribution

       async def get_customer_journey(
           self,
           order_id: str,
       ) -> List[Dict[str, Any]]:
           """Trace the customer journey leading to a purchase"""

           from app.db.prisma import prisma

           order = await prisma.order.find_unique(
               where={"id": order_id},
               include={"customer": True},
           )

           if not order:
               return []

           # Build customer touchpoint timeline
           touchpoints = []

           # Check social media interactions
           # Check ad clicks
           # Check website visits
           # Implementation depends on tracking system

           return touchpoints
   ```

2. **Predictive Insights Engine**
   ```python
   # app/analytics/predictive_insights.py
   from typing import Dict, Any
   import statistics
   from datetime import datetime, timedelta

   class PredictiveInsightsEngine:
       """ML-powered predictions for marketing performance"""

       def __init__(self):
           pass

       async def predict_campaign_performance(
           self,
           campaign_id: str,
       ) -> Dict[str, Any]:
           """Predict campaign performance based on historical data"""

           from app.db.prisma import prisma

           campaign = await prisma.adcampaign.find_unique(
               where={"id": campaign_id}
           )

           if not campaign:
               return {}

           # Get historical campaigns with similar attributes
           similar_campaigns = await prisma.adcampaign.find_many(
               where={
                   "storeId": campaign.storeId,
                   "platform": campaign.platform,
                   "objective": campaign.objective,
                   "status": {"in": ["ACTIVE", "PAUSED", "COMPLETED"]},
               }
           )

           if len(similar_campaigns) < 3:
               return {"prediction": "insufficient_data"}

           # Calculate average metrics
           roas_values = [c.roas for c in similar_campaigns if c.roas]
           conversion_rates = [
               c.conversions / c.clicks if c.clicks > 0 else 0
               for c in similar_campaigns
           ]

           predicted_roas = statistics.mean(roas_values) if roas_values else 0
           predicted_cvr = statistics.mean(conversion_rates)

           # Estimate revenue
           predicted_spend = float(campaign.dailyBudget) * 30
           predicted_revenue = predicted_spend * predicted_roas

           return {
               "prediction": "success",
               "predicted_roas": predicted_roas,
               "predicted_cvr": predicted_cvr * 100,
               "predicted_monthly_spend": predicted_spend,
               "predicted_monthly_revenue": predicted_revenue,
               "confidence": min(len(similar_campaigns) / 10, 1.0),
           }

       async def recommend_budget_allocation(
           self,
           store_id: str,
           total_budget: Decimal,
       ) -> Dict[str, Any]:
           """Recommend optimal budget allocation across channels"""

           from app.db.prisma import prisma

           # Get historical performance by platform
           campaigns = await prisma.adcampaign.find_many(
               where={
                   "storeId": store_id,
                   "status": {"in": ["ACTIVE", "PAUSED", "COMPLETED"]},
               }
           )

           platform_performance = {}

           for campaign in campaigns:
               platform = campaign.platform
               if platform not in platform_performance:
                   platform_performance[platform] = {
                       "spend": 0,
                       "revenue": 0,
                       "roas": 0,
                       "count": 0,
                   }

               platform_performance[platform]["spend"] += float(campaign.spend or 0)
               platform_performance[platform]["revenue"] += float(campaign.revenue or 0)
               platform_performance[platform]["count"] += 1

           # Calculate average ROAS per platform
           for platform in platform_performance:
               perf = platform_performance[platform]
               if perf["spend"] > 0:
                   perf["roas"] = perf["revenue"] / perf["spend"]

           # Allocate budget proportionally to ROAS
           total_roas = sum(p["roas"] for p in platform_performance.values())

           recommendations = {}
           for platform, perf in platform_performance.items():
               if total_roas > 0:
                   allocation = (perf["roas"] / total_roas) * total_budget
               else:
                   allocation = total_budget / len(platform_performance)

               recommendations[platform] = {
                   "recommended_budget": float(allocation),
                   "expected_roas": perf["roas"],
                   "expected_revenue": float(allocation) * perf["roas"],
               }

           return recommendations
   ```

3. **Analytics Dashboard Component**
   ```typescript
   // components/growth/ROIAnalytics.tsx
   'use client';

   import { useState, useEffect } from 'react';
   import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
   import { PieChart, Pie, Cell, ResponsiveContainer, Legend, Tooltip } from 'recharts';

   export default function ROIAnalytics() {
     const [attribution, setAttribution] = useState<any>(null);
     const [predictions, setPredictions] = useState<any>(null);

     useEffect(() => {
       fetchAnalytics();
     }, []);

     const fetchAnalytics = async () => {
       const attrResponse = await fetch('/api/analytics/attribution');
       const attrData = await attrResponse.json();
       setAttribution(attrData);

       const predResponse = await fetch('/api/analytics/predictions');
       const predData = await predResponse.json();
       setPredictions(predData);
     };

     const channelData = attribution ? [
       { name: 'Organic Social', value: parseFloat(attribution.channels.organic_social) },
       { name: 'Paid Ads', value: parseFloat(attribution.channels.paid_ads) },
       { name: 'Direct', value: parseFloat(attribution.channels.direct) },
       { name: 'Other', value: parseFloat(attribution.channels.other) },
     ] : [];

     const COLORS = ['#0088FE', '#00C49F', '#FFBB28', '#FF8042'];

     return (
       <div className="space-y-6">
         {/* Revenue Attribution */}
         <Card>
           <CardHeader>
             <CardTitle>Revenue Attribution by Channel</CardTitle>
           </CardHeader>
           <CardContent>
             <ResponsiveContainer width="100%" height={300}>
               <PieChart>
                 <Pie
                   data={channelData}
                   cx="50%"
                   cy="50%"
                   labelLine={false}
                   label={({ name, percent }) => `${name} ${(percent * 100).toFixed(0)}%`}
                   outerRadius={80}
                   fill="#8884d8"
                   dataKey="value"
                 >
                   {channelData.map((entry, index) => (
                     <Cell key={`cell-${index}`} fill={COLORS[index % COLORS.length]} />
                   ))}
                 </Pie>
                 <Tooltip />
                 <Legend />
               </PieChart>
             </ResponsiveContainer>
           </CardContent>
         </Card>

         {/* ROI Breakdown */}
         <Card>
           <CardHeader>
             <CardTitle>ROI by Channel</CardTitle>
           </CardHeader>
           <CardContent>
             <div className="space-y-4">
               {attribution && Object.entries(attribution.roi).map(([channel, data]: [string, any]) => (
                 <div key={channel} className="border-b pb-4">
                   <h4 className="font-semibold capitalize">{channel.replace('_', ' ')}</h4>
                   <div className="grid grid-cols-3 gap-4 mt-2">
                     <div>
                       <div className="text-sm text-gray-500">Revenue</div>
                       <div className="text-lg font-bold">${data.revenue.toFixed(2)}</div>
                     </div>
                     <div>
                       <div className="text-sm text-gray-500">Cost</div>
                       <div className="text-lg font-bold">${data.cost.toFixed(2)}</div>
                     </div>
                     <div>
                       <div className="text-sm text-gray-500">ROI</div>
                       <div className="text-lg font-bold">
                         {data.roas ? `${data.roas.toFixed(2)}x` : 'Organic'}
                       </div>
                     </div>
                   </div>
                 </div>
               ))}
             </div>
           </CardContent>
         </Card>

         {/* Predictions */}
         {predictions && (
           <Card>
             <CardHeader>
               <CardTitle>Budget Recommendations</CardTitle>
             </CardHeader>
             <CardContent>
               <div className="space-y-4">
                 {Object.entries(predictions.recommendations || {}).map(([platform, rec]: [string, any]) => (
                   <div key={platform} className="border rounded-lg p-4">
                     <h4 className="font-semibold capitalize">{platform}</h4>
                     <div className="grid grid-cols-3 gap-4 mt-2">
                       <div>
                         <div className="text-sm text-gray-500">Recommended Budget</div>
                         <div className="text-lg font-bold">${rec.recommended_budget.toFixed(2)}</div>
                       </div>
                       <div>
                         <div className="text-sm text-gray-500">Expected ROAS</div>
                         <div className="text-lg font-bold">{rec.expected_roas.toFixed(2)}x</div>
                       </div>
                       <div>
                         <div className="text-sm text-gray-500">Expected Revenue</div>
                         <div className="text-lg font-bold">${rec.expected_revenue.toFixed(2)}</div>
                       </div>
                     </div>
                   </div>
                 ))}
               </div>
             </CardContent>
           </Card>
         )}
       </div>
     );
   }
   ```

#### Deliverables
- [ ] ROI attribution system functional
- [ ] Multi-touch attribution tracking
- [ ] Predictive insights engine working
- [ ] Analytics dashboard complete

---

### Days 10-11: System Optimization & Performance

#### Tasks
1. **Database Query Optimization**
   ```python
   # app/db/optimizations.py
   """
   Database query optimizations for Growth Hub
   """

   # Add indexes to frequently queried fields
   """
   CREATE INDEX idx_social_post_store_published ON "SocialPost" ("storeId", "publishedAt");
   CREATE INDEX idx_social_post_engagement ON "SocialPost" ("engagement");
   CREATE INDEX idx_ad_campaign_store_status ON "AdCampaign" ("storeId", "status");
   CREATE INDEX idx_ad_campaign_roas ON "AdCampaign" ("roas");
   CREATE INDEX idx_order_store_created ON "Order" ("storeId", "createdAt");
   CREATE INDEX idx_platform_connection_store ON "PlatformConnection" ("storeId", "platform");
   """

   # Use query batching for analytics
   async def batch_fetch_campaign_insights(campaign_ids: List[str]):
       """Batch fetch insights for multiple campaigns"""
       # Implementation with connection pooling
       pass
   ```

2. **Caching Layer**
   ```python
   # app/cache/growth_cache.py
   from redis import Redis
   import json
   from typing import Any, Optional
   from datetime import timedelta

   class GrowthCache:
       """Redis cache for Growth Hub data"""

       def __init__(self):
           self.redis = Redis(
               host=settings.REDIS_HOST,
               port=settings.REDIS_PORT,
               db=1,  # Separate DB for growth data
           )

       async def get_overview(self, store_id: str) -> Optional[Dict]:
           """Get cached overview metrics"""
           key = f"growth:overview:{store_id}"
           data = self.redis.get(key)
           return json.loads(data) if data else None

       async def set_overview(self, store_id: str, data: Dict, ttl: int = 300):
           """Cache overview metrics (5 min TTL)"""
           key = f"growth:overview:{store_id}"
           self.redis.setex(
               key,
               ttl,
               json.dumps(data),
           )

       async def get_attribution(self, store_id: str) -> Optional[Dict]:
           """Get cached attribution data"""
           key = f"growth:attribution:{store_id}"
           data = self.redis.get(key)
           return json.loads(data) if data else None

       async def set_attribution(self, store_id: str, data: Dict, ttl: int = 3600):
           """Cache attribution (1 hour TTL)"""
           key = f"growth:attribution:{store_id}"
           self.redis.setex(
               key,
               ttl,
               json.dumps(data),
           )

       async def invalidate_store(self, store_id: str):
           """Invalidate all cache for a store"""
           pattern = f"growth:*:{store_id}"
           keys = self.redis.keys(pattern)
           if keys:
               self.redis.delete(*keys)
   ```

3. **API Response Time Optimization**
   ```python
   # app/api/growth/overview.py (optimized)
   from fastapi import APIRouter, Depends, BackgroundTasks
   from app.auth import get_current_user
   from app.cache.growth_cache import GrowthCache
   from app.analytics import ROIAttributionEngine

   router = APIRouter()
   cache = GrowthCache()

   @router.get("/overview")
   async def get_growth_overview(
       background_tasks: BackgroundTasks,
       user = Depends(get_current_user),
   ):
       """Get growth hub overview with caching"""

       store_id = user.storeId

       # Try cache first
       cached = await cache.get_overview(store_id)
       if cached:
           return cached

       # Fetch fresh data
       overview = await compute_overview(store_id)

       # Cache in background
       background_tasks.add_task(
           cache.set_overview,
           store_id,
           overview,
       )

       return overview

   async def compute_overview(store_id: str) -> Dict:
       """Compute overview metrics (heavy operation)"""
       # Complex aggregations here
       pass
   ```

#### Deliverables
- [ ] Database indexes added
- [ ] Redis caching implemented
- [ ] API response times < 200ms
- [ ] Query performance optimized

---

### Days 12-13: Security Audit & Monitoring

#### Tasks
1. **Security Hardening**
   ```python
   # app/security/growth_security.py
   """
   Security measures for Growth Hub
   """

   # Rate limiting for OAuth callbacks
   from fastapi import Request
   from slowapi import Limiter
   from slowapi.util import get_remote_address

   limiter = Limiter(key_func=get_remote_address)

   @router.post("/oauth/callback")
   @limiter.limit("5/minute")
   async def oauth_callback(request: Request):
       """Rate-limited OAuth callback"""
       pass

   # Encrypt sensitive platform tokens
   from cryptography.fernet import Fernet

   class TokenEncryption:
       def __init__(self):
           self.cipher = Fernet(settings.ENCRYPTION_KEY)

       def encrypt_token(self, token: str) -> str:
           return self.cipher.encrypt(token.encode()).decode()

       def decrypt_token(self, encrypted: str) -> str:
           return self.cipher.decrypt(encrypted.encode()).decode()

   # Validate webhook signatures
   import hmac
   import hashlib

   def verify_webhook_signature(
       payload: bytes,
       signature: str,
       secret: str,
   ) -> bool:
       """Verify webhook signature"""
       expected = hmac.new(
           secret.encode(),
           payload,
           hashlib.sha256,
       ).hexdigest()
       return hmac.compare_digest(expected, signature)
   ```

2. **Monitoring Setup**
   ```python
   # app/monitoring/growth_metrics.py
   from prometheus_client import Counter, Histogram, Gauge
   import logging

   # Metrics
   oauth_connections = Counter(
       'growth_oauth_connections_total',
       'Total OAuth connections',
       ['platform', 'status']
   )

   social_posts_published = Counter(
       'growth_social_posts_published_total',
       'Total social posts published',
       ['platform']
   )

   ad_campaigns_created = Counter(
       'growth_ad_campaigns_created_total',
       'Total ad campaigns created',
       ['platform']
   )

   api_request_duration = Histogram(
       'growth_api_request_duration_seconds',
       'API request duration',
       ['endpoint']
   )

   active_campaigns = Gauge(
       'growth_active_campaigns',
       'Number of active ad campaigns'
   )

   # Logging
   logger = logging.getLogger('growth_hub')

   def log_campaign_created(campaign_id: str, platform: str):
       logger.info(f"Campaign created: {campaign_id} on {platform}")
       ad_campaigns_created.labels(platform=platform).inc()

   def log_oauth_connected(platform: str, success: bool):
       status = "success" if success else "failure"
       logger.info(f"OAuth connection: {platform} - {status}")
       oauth_connections.labels(platform=platform, status=status).inc()
   ```

3. **Error Tracking**
   ```python
   # app/monitoring/error_tracking.py
   import sentry_sdk
   from sentry_sdk.integrations.fastapi import FastApiIntegration
   from sentry_sdk.integrations.celery import CeleryIntegration

   sentry_sdk.init(
       dsn=settings.SENTRY_DSN,
       integrations=[
           FastApiIntegration(),
           CeleryIntegration(),
       ],
       traces_sample_rate=0.1,
       environment=settings.ENVIRONMENT,
   )

   # Custom error handling
   class GrowthHubError(Exception):
       """Base exception for Growth Hub"""
       pass

   class PlatformConnectionError(GrowthHubError):
       """Platform connection failed"""
       pass

   class CampaignCreationError(GrowthHubError):
       """Campaign creation failed"""
       pass
   ```

#### Deliverables
- [ ] Rate limiting implemented
- [ ] Token encryption enabled
- [ ] Webhook signature verification
- [ ] Prometheus metrics exported
- [ ] Sentry error tracking active

---

### Day 14: Documentation & Final Testing

#### Tasks
1. **API Documentation**
   ```python
   # Update FastAPI docs with examples

   @router.post("/campaigns", response_model=CampaignResponse)
   async def create_campaign(
       campaign: CampaignCreate,
       user = Depends(get_current_user),
   ):
       """
       Create a new ad campaign.

       ## Example Request:
       ```json
       {
         "name": "Summer Sale 2025",
         "platform": "meta",
         "objective": "conversions",
         "daily_budget": 5000,
         "creative_ids": ["creative_123"]
       }
       ```

       ## Example Response:
       ```json
       {
         "id": "campaign_456",
         "name": "Summer Sale 2025",
         "status": "PAUSED",
         "created_at": "2025-06-01T00:00:00Z"
       }
       ```
       """
       pass
   ```

2. **User Documentation**
   ```markdown
   # docs/GROWTH_HUB_USER_GUIDE.md

   # Growth Hub User Guide

   ## Overview
   The Growth Hub is your command center for managing all marketing activities across social media and paid advertising platforms.

   ## Getting Started

   ### 1. Connect Your Platforms
   - Click "Connect Platform" in the Growth Hub
   - Select a platform (TikTok, Instagram, Meta Ads, etc.)
   - Authorize the OAuth connection
   - Platform will appear in your connected platforms list

   ### 2. Launch Your First Social Campaign
   - Navigate to Growth Hub → Social Media
   - Click "Create Content Calendar"
   - Select platforms and duration
   - Review AI-generated content
   - Click "Schedule All Posts"

   ### 3. Create Your First Ad Campaign
   - Navigate to Growth Hub → Paid Ads
   - Click "New Campaign"
   - Select platform and objective
   - Set daily budget
   - Choose or generate creatives
   - Launch campaign

   ## Advanced Features

   ### Auto-Promote Top Content
   Enable automatic promotion of high-performing social posts to paid ads.

   ### Product Launch Automation
   Automate multi-channel product launches with one click.

   ### ROI Analytics
   Track revenue attribution across all marketing channels.
   ```

3. **Integration Testing**
   ```python
   # tests/integration/test_growth_hub.py
   import pytest
   from fastapi.testclient import TestClient

   @pytest.mark.integration
   async def test_complete_growth_workflow(client: TestClient):
       """Test complete growth hub workflow"""

       # 1. Connect OAuth platform
       response = client.post("/api/oauth/authorize", json={
           "platform": "tiktok"
       })
       assert response.status_code == 200

       # 2. Create social content calendar
       response = client.post("/api/social/calendar", json={
           "days": 7,
           "posts_per_day": 2,
           "platforms": ["tiktok"]
       })
       assert response.status_code == 200
       calendar = response.json()

       # 3. Publish post
       post_id = calendar["posts"][0]["id"]
       response = client.post(f"/api/social/publish/{post_id}")
       assert response.status_code == 200

       # 4. Create ad campaign
       response = client.post("/api/ads/campaigns", json={
           "name": "Test Campaign",
           "platform": "meta",
           "objective": "conversions",
           "daily_budget": 1000
       })
       assert response.status_code == 200
       campaign = response.json()

       # 5. Get analytics
       response = client.get("/api/analytics/attribution")
       assert response.status_code == 200

       # 6. Verify overview
       response = client.get("/api/growth/overview")
       assert response.status_code == 200
       overview = response.json()
       assert "total_revenue" in overview
   ```

#### Deliverables
- [ ] API documentation complete
- [ ] User guide written
- [ ] Integration tests passing
- [ ] All endpoints documented

---

## Definition of Done

### Technical Requirements
- [ ] Unified Growth Hub dashboard deployed
- [ ] Cross-agent workflows operational
- [ ] ROI attribution system functional
- [ ] Predictive insights working
- [ ] Performance optimizations complete
- [ ] Security audit passed
- [ ] Monitoring and alerting active
- [ ] All tests passing (95%+ coverage)

### Integration Requirements
- [ ] OAuth system integrated
- [ ] Social Agent integrated
- [ ] Ads Agent integrated
- [ ] Analytics pipeline working
- [ ] Automation workflows running

### Production Readiness
- [ ] Security hardening complete
- [ ] Rate limiting enabled
- [ ] Error tracking configured
- [ ] Monitoring dashboards set up
- [ ] Documentation finalized
- [ ] Performance benchmarks met

---

## Testing Checklist

### Integration Testing
- [ ] OAuth → Social workflow
- [ ] OAuth → Ads workflow
- [ ] Social → Ads promotion pipeline
- [ ] Product launch automation
- [ ] ROI attribution calculation
- [ ] Predictive insights generation

### Performance Testing
- [ ] API response times < 200ms
- [ ] Database query performance
- [ ] Cache hit rate > 80%
- [ ] Worker processing times
- [ ] Concurrent user load testing

### Security Testing
- [ ] OAuth token encryption
- [ ] Webhook signature verification
- [ ] Rate limiting effectiveness
- [ ] API authentication
- [ ] SQL injection prevention
- [ ] XSS prevention

### User Acceptance Testing
- [ ] Dashboard usability
- [ ] Platform connections work
- [ ] Content creation flow
- [ ] Campaign management
- [ ] Analytics accuracy

---

## Success Metrics

### Integration Success
- **Target:** All agents working together seamlessly
- **Measurement:** Zero integration errors in production

### Performance
- **Target:** API response < 200ms (p95)
- **Measurement:** Prometheus metrics

### User Experience
- **Target:** < 5 clicks to launch product campaign
- **Measurement:** User flow analysis

### Automation
- **Target:** 90% of tasks automated
- **Measurement:** Manual intervention rate

### ROI Accuracy
- **Target:** 95%+ attribution accuracy
- **Measurement:** Compare with platform reporting

---

## Production Deployment Checklist

### Pre-Deployment
- [ ] All environment variables configured
- [ ] Database migrations tested
- [ ] Redis cache warmed
- [ ] SSL certificates installed
- [ ] DNS records updated
- [ ] CDN configured

### Deployment
- [ ] Blue-green deployment strategy
- [ ] Database backup created
- [ ] Health checks passing
- [ ] Monitoring alerts configured
- [ ] Rollback plan ready

### Post-Deployment
- [ ] Smoke tests passed
- [ ] Performance metrics normal
- [ ] Error rates < 0.1%
- [ ] User acceptance testing
- [ ] Documentation updated

---

## Phase 3 Completion

Upon completion of Sprint 6, Phase 3 (The Growth Hub) will be fully operational with:

1. ✅ **OAuth Foundation** - Secure platform connections
2. ✅ **Social Agent Core** - AI-powered social media automation
3. ✅ **Social Agent Advanced** - Multi-platform publishing
4. ✅ **Ads Agent Core** - Meta Ads campaign management
5. ✅ **Ads Agent Advanced** - Multi-platform ads (Google, TikTok)
6. ✅ **Integration & Polish** - Unified Growth Hub

**Next Phase:** Phase 4 - Expansion (SEO, Influencer Outreach, Advanced Features)
