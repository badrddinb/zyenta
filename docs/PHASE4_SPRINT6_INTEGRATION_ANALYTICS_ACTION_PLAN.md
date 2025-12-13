# Phase 4 - Sprint 6: Integration & Analytics - Action Plan

## Sprint Overview

**Duration:** 2 weeks (14 days)
**Team Size:** 4-5 developers (2 backend, 2 frontend, 1 DevOps)
**Sprint Goal:** Integrate all Phase 4 components, build unified marketing dashboard, implement comprehensive ROI tracking, and prepare for production deployment.

### Objectives
- [ ] Integrate SEO and Influencer marketing systems
- [ ] Build unified marketing analytics dashboard
- [ ] Implement multi-channel ROI attribution
- [ ] Create performance comparison and benchmarking tools
- [ ] Build budget optimization recommendations
- [ ] Complete production readiness tasks (performance, security, monitoring)
- [ ] Finalize documentation and training materials

### Prerequisites
- All Phase 4 Sprints 1-5 completed
- SEO content system operational (keyword research, article generation, publishing)
- Influencer marketing functional (discovery, outreach, campaigns)
- Analytics data collecting across all systems
- Database performance optimized

---

## Database Models

### Prisma Schema Extensions

```prisma
model MarketingChannel {
  id              String   @id @default(cuid())
  storeId         String
  type            String   // seo, influencer, social, ads
  name            String

  // Budgets
  monthlyBudget   Decimal  @db.Decimal(10, 2)
  spentThisMonth  Decimal  @db.Decimal(10, 2) @default(0)

  // Status
  status          String   @default("active") // active, paused, archived
  createdAt       DateTime @default(now())

  store           Store    @relation(fields: [storeId], references: [id], onDelete: Cascade)

  @@index([storeId, type])
}

model UnifiedMetric {
  id              String   @id @default(cuid())
  storeId         String
  channel         String   // seo, influencer, social, ads, overall
  source          String?  // Specific source (article ID, campaign ID, etc.)

  // Timeframe
  date            DateTime
  period          String   // daily, weekly, monthly

  // Traffic Metrics
  sessions        Int      @default(0)
  pageviews       Int      @default(0)
  uniqueVisitors  Int      @default(0)
  avgSessionDuration Decimal? @db.Decimal(10, 2)
  bounceRate      Decimal? @db.Decimal(5, 2)

  // Engagement Metrics
  clicks          Int      @default(0)
  impressions     Int      @default(0)
  engagementRate  Decimal? @db.Decimal(5, 2)

  // Conversion Metrics
  conversions     Int      @default(0)
  conversionRate  Decimal? @db.Decimal(5, 2)
  revenue         Decimal  @db.Decimal(10, 2) @default(0)

  // Cost Metrics
  cost            Decimal  @db.Decimal(10, 2) @default(0)
  cpc             Decimal? @db.Decimal(10, 4)
  cpa             Decimal? @db.Decimal(10, 2)
  roas            Decimal? @db.Decimal(10, 2)

  createdAt       DateTime @default(now())

  store           Store    @relation(fields: [storeId], references: [id], onDelete: Cascade)

  @@unique([storeId, channel, date, period, source])
  @@index([storeId, channel, date])
  @@index([storeId, date])
}

model Attribution {
  id              String   @id @default(cuid())
  storeId         String
  orderId         String?  // If attributed to an order

  // Customer Journey
  firstTouch      Json     // {channel: "seo", source: "article_123", timestamp: "..."}
  lastTouch       Json
  allTouches      Json[]   // Array of all touchpoints

  // Attribution Model Results
  linearValue     Json     // {seo: 50, influencer: 50}
  timeDecayValue  Json
  lastTouchValue  Json

  // Conversion Details
  convertedAt     DateTime
  revenue         Decimal  @db.Decimal(10, 2)
  profit          Decimal? @db.Decimal(10, 2)

  createdAt       DateTime @default(now())

  store           Store    @relation(fields: [storeId], references: [id], onDelete: Cascade)

  @@index([storeId, convertedAt])
}

model PerformanceBenchmark {
  id              String   @id @default(cuid())
  channel         String
  metric          String   // conversion_rate, roas, cpa, engagement_rate
  industry        String   @default("general")

  // Benchmark Ranges
  poor            Decimal  @db.Decimal(10, 2)
  average         Decimal  @db.Decimal(10, 2)
  good            Decimal  @db.Decimal(10, 2)
  excellent       Decimal  @db.Decimal(10, 2)

  updatedAt       DateTime @updatedAt

  @@unique([channel, metric, industry])
}

model MarketingInsight {
  id              String   @id @default(cuid())
  storeId         String
  type            String   // opportunity, warning, recommendation
  category        String   // budget, performance, content, campaign

  // Insight Details
  title           String
  description     String
  priority        String   // low, medium, high, critical
  impact          Decimal? @db.Decimal(10, 2) // Estimated revenue impact

  // Suggested Actions
  suggestedAction String?
  actionUrl       String?

  // Status
  status          String   @default("active") // active, dismissed, actioned
  createdAt       DateTime @default(now())
  dismissedAt     DateTime?
  actionedAt      DateTime?

  store           Store    @relation(fields: [storeId], references: [id], onDelete: Cascade)

  @@index([storeId, status, priority])
}
```

---

## Day-by-Day Breakdown

### **Days 1-2: System Integration**

**Backend Tasks:**
1. Build marketing integration orchestrator
2. Create unified event tracking system
3. Implement cross-system data synchronization
4. Add webhook connectors between SEO and Influencer systems
5. Build unified content calendar

**Data Sync:**
- SEO articles to unified calendar
- Influencer campaign deliverables to calendar
- Analytics data aggregation
- Revenue attribution linkage

**Testing:**
- Test data sync between systems
- Verify event tracking
- Test calendar integration

### **Days 3-4: Unified Metrics Collection**

**Backend Tasks:**
1. Build unified metrics aggregator
2. Create daily metrics rollup job
3. Implement real-time metrics updates
4. Add data warehouse integration
5. Build metrics API

**Metrics Sources:**
- SEO: Article views, organic traffic, conversions
- Influencer: Campaign reach, engagements, conversions
- Social: Post performance
- Ads: Campaign metrics

**Testing:**
- Test metrics collection from all sources
- Verify aggregation accuracy
- Test API endpoints

### **Days 5-6: ROI Attribution Engine**

**Backend Tasks:**
1. Build multi-touch attribution system
2. Implement attribution models (linear, time-decay, last-touch)
3. Create customer journey tracker
4. Add revenue attribution logic
5. Build profit calculation

**Attribution Models:**
- First Touch
- Last Touch
- Linear (equal credit)
- Time Decay (more recent gets more credit)
- Position-Based (40-20-40)

**Testing:**
- Test attribution with sample journeys
- Verify revenue calculations
- Test different attribution models

### **Days 7-8: Unified Dashboard Backend**

**Backend Tasks:**
1. Build dashboard data aggregator
2. Create caching layer for dashboard metrics
3. Implement real-time dashboard updates
4. Add date range filtering
5. Build comparison endpoints (vs previous period)

**Dashboard Metrics:**
- Overall marketing performance
- Channel-by-channel breakdown
- ROI and ROAS
- Traffic sources
- Conversion funnels
- Budget utilization

**Testing:**
- Test dashboard data accuracy
- Verify caching behavior
- Test real-time updates

### **Days 9-10: Performance Comparison & Benchmarking**

**Backend Tasks:**
1. Build performance comparison service
2. Implement industry benchmarking
3. Create channel efficiency scorer
4. Add trend analysis
5. Build performance predictions

**Comparisons:**
- SEO vs Influencer performance
- This month vs last month
- Store performance vs industry benchmarks
- Campaign performance over time

**Testing:**
- Test comparison calculations
- Verify benchmark data
- Test prediction accuracy

### **Days 11-12: Budget Optimizer & Insights**

**Backend Tasks:**
1. Build budget optimizer
2. Create AI-powered insights generator
3. Implement opportunity detector
4. Add warning system for underperforming channels
5. Build recommendation engine

**Optimizations:**
- Reallocate budget from low-ROAS to high-ROAS channels
- Identify underutilized budgets
- Detect declining performance
- Recommend scaling opportunities

**Testing:**
- Test budget optimization logic
- Verify insight generation
- Test recommendation accuracy

### **Days 13-14: Dashboard UI**

**Frontend Tasks:**
1. Create unified marketing dashboard page
2. Build overview stats cards
3. Add channel performance charts
4. Create ROI attribution visualizations
5. Build budget allocation UI
6. Add insights/recommendations panel
7. Create export functionality

**UI Components:**
- Overview metrics (revenue, ROAS, conversions)
- Channel comparison charts
- Attribution breakdown pie charts
- Timeline views
- Budget utilization bars
- Insight cards

**Testing:**
- Test UI with various data sets
- Verify chart rendering
- Test responsive design
- Test export functionality

---

## Core Components Implementation

### 1. Marketing Integration Orchestrator

```python
# services/marketing/integration.py

from typing import Dict, List, Any
from datetime import datetime, timedelta
from prisma import Prisma
from decimal import Decimal

class MarketingIntegrationOrchestrator:
    """Orchestrate integration between marketing systems"""

    def __init__(self, prisma: Prisma):
        self.prisma = prisma

    async def sync_all_data(self, store_id: str) -> Dict[str, Any]:
        """Sync all marketing data for unified view"""
        results = {
            'seo_synced': 0,
            'influencer_synced': 0,
            'metrics_updated': 0,
        }

        # Sync SEO data
        results['seo_synced'] = await self._sync_seo_metrics(store_id)

        # Sync Influencer data
        results['influencer_synced'] = await self._sync_influencer_metrics(store_id)

        # Update unified metrics
        results['metrics_updated'] = await self._update_unified_metrics(store_id)

        return results

    async def _sync_seo_metrics(self, store_id: str) -> int:
        """Sync SEO article metrics to unified system"""
        # Get articles published in last 30 days
        cutoff = datetime.utcnow() - timedelta(days=30)

        articles = await self.prisma.blogarticle.find_many(
            where={
                'storeId': store_id,
                'status': 'published',
                'publishedAt': {'gte': cutoff}
            }
        )

        synced = 0
        for article in articles:
            # Aggregate article metrics
            await self._create_unified_metric(
                store_id=store_id,
                channel='seo',
                source=f"article_{article.id}",
                date=datetime.utcnow().date(),
                period='daily',
                pageviews=article.pageviews or 0,
                uniqueVisitors=article.uniqueVisitors or 0,
                avgSessionDuration=Decimal(str(article.avgTimeOnPage or 0)),
                bounceRate=Decimal(str(article.bounceRate or 0)),
                # Additional metrics would be fetched from analytics service
            )
            synced += 1

        return synced

    async def _sync_influencer_metrics(self, store_id: str) -> int:
        """Sync influencer campaign metrics to unified system"""
        # Get active campaigns
        campaigns = await self.prisma.influencercampaign.find_many(
            where={
                'storeId': store_id,
                'status': {'in': ['active', 'completed']}
            },
            include={
                'outreach': True
            }
        )

        synced = 0
        for campaign in campaigns:
            # Calculate campaign metrics
            total_sent = len([o for o in campaign.outreach if o.status == 'sent'])
            total_opened = len([o for o in campaign.outreach if o.openedAt])
            total_replied = len([o for o in campaign.outreach if o.repliedAt])

            await self._create_unified_metric(
                store_id=store_id,
                channel='influencer',
                source=f"campaign_{campaign.id}",
                date=datetime.utcnow().date(),
                period='daily',
                impressions=total_sent,
                clicks=total_opened,
                engagementRate=Decimal(str((total_replied / total_sent * 100) if total_sent > 0 else 0)),
                cost=campaign.budgetAllocated,
            )
            synced += 1

        return synced

    async def _create_unified_metric(
        self,
        store_id: str,
        channel: str,
        source: str,
        date: Any,
        period: str,
        **metrics
    ):
        """Create or update unified metric"""
        # Convert date to datetime if it's a date object
        if isinstance(date, datetime.date) and not isinstance(date, datetime):
            date = datetime.combine(date, datetime.min.time())

        # Upsert unified metric
        await self.prisma.unifiedmetric.upsert(
            where={
                'storeId_channel_date_period_source': {
                    'storeId': store_id,
                    'channel': channel,
                    'date': date,
                    'period': period,
                    'source': source,
                }
            },
            create={
                'storeId': store_id,
                'channel': channel,
                'source': source,
                'date': date,
                'period': period,
                **metrics
            },
            update=metrics
        )

    async def _update_unified_metrics(self, store_id: str) -> int:
        """Update aggregated unified metrics"""
        today = datetime.utcnow().date()

        # Aggregate daily metrics by channel
        channels = ['seo', 'influencer', 'social', 'ads']

        updated = 0
        for channel in channels:
            # Get all metrics for this channel today
            metrics = await self.prisma.unifiedmetric.find_many(
                where={
                    'storeId': store_id,
                    'channel': channel,
                    'date': {'gte': datetime.combine(today, datetime.min.time())},
                    'period': 'daily'
                }
            )

            if metrics:
                # Aggregate
                total_sessions = sum(m.sessions for m in metrics)
                total_revenue = sum(m.revenue for m in metrics)
                total_cost = sum(m.cost for m in metrics)
                total_conversions = sum(m.conversions for m in metrics)

                # Create channel-level aggregate
                await self._create_unified_metric(
                    store_id=store_id,
                    channel=channel,
                    source=None,  # No specific source for aggregates
                    date=today,
                    period='daily',
                    sessions=total_sessions,
                    revenue=total_revenue,
                    cost=total_cost,
                    conversions=total_conversions,
                    roas=Decimal(str((total_revenue / total_cost) if total_cost > 0 else 0)),
                )

                updated += 1

        return updated

    async def get_unified_calendar(
        self,
        store_id: str,
        start_date: datetime,
        end_date: datetime,
    ) -> List[Dict[str, Any]]:
        """Get unified content calendar"""
        calendar_items = []

        # Get SEO articles
        articles = await self.prisma.blogarticle.find_many(
            where={
                'storeId': store_id,
                'publishedAt': {'gte': start_date, 'lte': end_date}
            }
        )

        for article in articles:
            calendar_items.append({
                'type': 'seo_article',
                'id': article.id,
                'title': article.title,
                'date': article.publishedAt,
                'status': article.status,
                'channel': 'SEO',
            })

        # Get influencer campaign deliverables
        contracts = await self.prisma.influencercontract.find_many(
            where={
                'storeId': store_id,
                'startDate': {'lte': end_date},
                'endDate': {'gte': start_date}
            },
            include={
                'deliveryTracking': True
            }
        )

        for contract in contracts:
            for delivery in contract.deliveryTracking:
                if delivery.postDate and start_date <= delivery.postDate <= end_date:
                    calendar_items.append({
                        'type': 'influencer_delivery',
                        'id': delivery.id,
                        'title': f"{delivery.deliverableType} on {delivery.platform}",
                        'date': delivery.postDate,
                        'status': delivery.status,
                        'channel': 'Influencer',
                    })

        # Sort by date
        calendar_items.sort(key=lambda x: x['date'])

        return calendar_items
```

### 2. ROI Attribution Engine

```python
# services/marketing/attribution.py

from typing import List, Dict, Any
from datetime import datetime, timedelta
from decimal import Decimal
from prisma import Prisma

class ROIAttributionEngine:
    """Multi-touch attribution for marketing ROI"""

    def __init__(self, prisma: Prisma):
        self.prisma = prisma

    async def attribute_conversion(
        self,
        store_id: str,
        order_id: str,
        customer_journey: List[Dict[str, Any]],
        revenue: Decimal,
    ) -> Dict[str, Any]:
        """
        Attribute conversion across multiple touchpoints

        Args:
            store_id: Store ID
            order_id: Order ID
            customer_journey: List of touchpoints [{channel, source, timestamp}, ...]
            revenue: Conversion revenue

        Returns:
            Attribution results for different models
        """
        if not customer_journey:
            return {}

        # Apply different attribution models
        linear = self._linear_attribution(customer_journey, revenue)
        time_decay = self._time_decay_attribution(customer_journey, revenue)
        last_touch = self._last_touch_attribution(customer_journey, revenue)

        # Store attribution
        attribution = await self.prisma.attribution.create(
            data={
                'storeId': store_id,
                'orderId': order_id,
                'firstTouch': customer_journey[0],
                'lastTouch': customer_journey[-1],
                'allTouches': customer_journey,
                'linearValue': linear,
                'timeDecayValue': time_decay,
                'lastTouchValue': last_touch,
                'convertedAt': datetime.utcnow(),
                'revenue': revenue,
            }
        )

        return {
            'attribution_id': attribution.id,
            'models': {
                'linear': linear,
                'time_decay': time_decay,
                'last_touch': last_touch,
            }
        }

    def _linear_attribution(
        self,
        journey: List[Dict],
        revenue: Decimal,
    ) -> Dict[str, Decimal]:
        """Equal credit to all touchpoints"""
        credit_per_touch = revenue / len(journey)

        result = {}
        for touch in journey:
            channel = touch['channel']
            result[channel] = result.get(channel, Decimal(0)) + credit_per_touch

        return {k: float(v) for k, v in result.items()}

    def _time_decay_attribution(
        self,
        journey: List[Dict],
        revenue: Decimal,
    ) -> Dict[str, Decimal]:
        """More credit to recent touchpoints"""
        # Use exponential decay with half-life of 7 days
        conversion_time = datetime.utcnow()

        weights = []
        for touch in journey:
            touch_time = datetime.fromisoformat(touch['timestamp'])
            days_ago = (conversion_time - touch_time).days
            weight = 2 ** (-days_ago / 7)
            weights.append(weight)

        total_weight = sum(weights)

        result = {}
        for touch, weight in zip(journey, weights):
            channel = touch['channel']
            credit = (weight / total_weight) * revenue
            result[channel] = result.get(channel, Decimal(0)) + Decimal(str(credit))

        return {k: float(v) for k, v in result.items()}

    def _last_touch_attribution(
        self,
        journey: List[Dict],
        revenue: Decimal,
    ) -> Dict[str, Decimal]:
        """All credit to last touchpoint"""
        last_touch = journey[-1]
        return {last_touch['channel']: float(revenue)}

    async def get_channel_attribution(
        self,
        store_id: str,
        start_date: datetime,
        end_date: datetime,
        model: str = 'linear',
    ) -> Dict[str, Decimal]:
        """Get total attributed revenue by channel"""
        attributions = await self.prisma.attribution.find_many(
            where={
                'storeId': store_id,
                'convertedAt': {'gte': start_date, 'lte': end_date}
            }
        )

        # Aggregate by channel based on selected model
        channel_revenue = {}

        for attr in attributions:
            if model == 'linear':
                values = attr.linearValue
            elif model == 'time_decay':
                values = attr.timeDecayValue
            else:
                values = attr.lastTouchValue

            for channel, value in values.items():
                channel_revenue[channel] = channel_revenue.get(channel, Decimal(0)) + Decimal(str(value))

        return channel_revenue

    async def calculate_channel_roi(
        self,
        store_id: str,
        channel: str,
        start_date: datetime,
        end_date: datetime,
    ) -> Dict[str, Decimal]:
        """Calculate ROI for a specific channel"""
        # Get attributed revenue
        attribution = await self.get_channel_attribution(
            store_id=store_id,
            start_date=start_date,
            end_date=end_date,
            model='linear'
        )

        revenue = attribution.get(channel, Decimal(0))

        # Get channel costs
        metrics = await self.prisma.unifiedmetric.find_many(
            where={
                'storeId': store_id,
                'channel': channel,
                'date': {'gte': start_date, 'lte': end_date}
            }
        )

        total_cost = sum(m.cost for m in metrics)

        # Calculate ROI
        roi = ((revenue - total_cost) / total_cost * 100) if total_cost > 0 else Decimal(0)
        roas = (revenue / total_cost) if total_cost > 0 else Decimal(0)

        return {
            'revenue': revenue,
            'cost': total_cost,
            'profit': revenue - total_cost,
            'roi': roi,
            'roas': roas,
        }
```

### 3. Performance Comparison Service

```python
# services/marketing/performance_comparison.py

from typing import Dict, List, Any
from datetime import datetime, timedelta
from decimal import Decimal
from prisma import Prisma

class PerformanceComparisonService:
    """Compare performance across channels and time periods"""

    def __init__(self, prisma: Prisma):
        self.prisma = prisma

    async def compare_channels(
        self,
        store_id: str,
        start_date: datetime,
        end_date: datetime,
    ) -> Dict[str, Any]:
        """Compare performance across all marketing channels"""
        channels = ['seo', 'influencer', 'social', 'ads']

        comparison = {}
        for channel in channels:
            metrics = await self.prisma.unifiedmetric.find_many(
                where={
                    'storeId': store_id,
                    'channel': channel,
                    'date': {'gte': start_date, 'lte': end_date},
                    'source': None  # Only aggregated metrics
                }
            )

            if metrics:
                total_revenue = sum(m.revenue for m in metrics)
                total_cost = sum(m.cost for m in metrics)
                total_conversions = sum(m.conversions for m in metrics)
                total_sessions = sum(m.sessions for m in metrics)

                comparison[channel] = {
                    'revenue': float(total_revenue),
                    'cost': float(total_cost),
                    'profit': float(total_revenue - total_cost),
                    'conversions': total_conversions,
                    'sessions': total_sessions,
                    'roas': float(total_revenue / total_cost) if total_cost > 0 else 0,
                    'cpa': float(total_cost / total_conversions) if total_conversions > 0 else 0,
                    'conversion_rate': float(total_conversions / total_sessions * 100) if total_sessions > 0 else 0,
                }

        # Calculate overall
        overall_revenue = sum(c['revenue'] for c in comparison.values())
        overall_cost = sum(c['cost'] for c in comparison.values())

        comparison['overall'] = {
            'revenue': overall_revenue,
            'cost': overall_cost,
            'profit': overall_revenue - overall_cost,
            'roas': overall_revenue / overall_cost if overall_cost > 0 else 0,
        }

        return comparison

    async def compare_periods(
        self,
        store_id: str,
        current_start: datetime,
        current_end: datetime,
        previous_start: datetime,
        previous_end: datetime,
    ) -> Dict[str, Any]:
        """Compare current period vs previous period"""
        current = await self.compare_channels(store_id, current_start, current_end)
        previous = await self.compare_channels(store_id, previous_start, previous_end)

        comparison = {}
        for channel in current.keys():
            if channel in previous:
                curr = current[channel]
                prev = previous[channel]

                comparison[channel] = {
                    'current': curr,
                    'previous': prev,
                    'change': {
                        'revenue': self._calculate_change(curr['revenue'], prev['revenue']),
                        'cost': self._calculate_change(curr['cost'], prev['cost']),
                        'profit': self._calculate_change(curr['profit'], prev['profit']),
                        'roas': self._calculate_change(curr['roas'], prev['roas']),
                    }
                }

        return comparison

    def _calculate_change(self, current: float, previous: float) -> Dict[str, Any]:
        """Calculate percentage change"""
        if previous == 0:
            return {'absolute': current, 'percentage': 100 if current > 0 else 0}

        absolute = current - previous
        percentage = (absolute / previous) * 100

        return {
            'absolute': absolute,
            'percentage': percentage,
            'direction': 'up' if absolute > 0 else 'down' if absolute < 0 else 'flat'
        }

    async def get_benchmarks(
        self,
        channel: str,
        industry: str = "general",
    ) -> Dict[str, Any]:
        """Get industry benchmarks for a channel"""
        benchmarks = {}

        metrics = ['conversion_rate', 'roas', 'cpa', 'engagement_rate']

        for metric in metrics:
            benchmark = await self.prisma.performancebenchmark.find_first(
                where={
                    'channel': channel,
                    'metric': metric,
                    'industry': industry
                }
            )

            if benchmark:
                benchmarks[metric] = {
                    'poor': float(benchmark.poor),
                    'average': float(benchmark.average),
                    'good': float(benchmark.good),
                    'excellent': float(benchmark.excellent),
                }

        return benchmarks

    async def compare_to_benchmarks(
        self,
        store_id: str,
        channel: str,
        start_date: datetime,
        end_date: datetime,
        industry: str = "general",
    ) -> Dict[str, Any]:
        """Compare store performance to industry benchmarks"""
        # Get store metrics
        metrics = await self.prisma.unifiedmetric.find_many(
            where={
                'storeId': store_id,
                'channel': channel,
                'date': {'gte': start_date, 'lte': end_date},
                'source': None
            }
        )

        if not metrics:
            return {}

        # Calculate averages
        total_revenue = sum(m.revenue for m in metrics)
        total_cost = sum(m.cost for m in metrics)
        total_conversions = sum(m.conversions for m in metrics)
        total_sessions = sum(m.sessions for m in metrics)

        store_metrics = {
            'conversion_rate': float(total_conversions / total_sessions * 100) if total_sessions > 0 else 0,
            'roas': float(total_revenue / total_cost) if total_cost > 0 else 0,
            'cpa': float(total_cost / total_conversions) if total_conversions > 0 else 0,
        }

        # Get benchmarks
        benchmarks = await self.get_benchmarks(channel, industry)

        # Compare
        comparison = {}
        for metric, value in store_metrics.items():
            if metric in benchmarks:
                bench = benchmarks[metric]
                if value >= bench['excellent']:
                    rating = 'excellent'
                elif value >= bench['good']:
                    rating = 'good'
                elif value >= bench['average']:
                    rating = 'average'
                else:
                    rating = 'poor'

                comparison[metric] = {
                    'value': value,
                    'rating': rating,
                    'benchmarks': bench,
                }

        return comparison
```

### 4. Budget Optimizer

```python
# services/marketing/budget_optimizer.py

from typing import Dict, List
from datetime import datetime, timedelta
from decimal import Decimal
from prisma import Prisma
from openai import AsyncOpenAI

class BudgetOptimizer:
    """Optimize marketing budget allocation"""

    def __init__(self, prisma: Prisma, openai_client: AsyncOpenAI):
        self.prisma = prisma
        self.openai = openai_client

    async def optimize_budget(
        self,
        store_id: str,
        total_budget: Decimal,
    ) -> Dict[str, Any]:
        """
        Optimize budget allocation across channels

        Uses historical ROAS to recommend budget distribution
        """
        # Get last 30 days performance
        end_date = datetime.utcnow()
        start_date = end_date - timedelta(days=30)

        # Get channel performance
        channels = ['seo', 'influencer', 'social', 'ads']
        performance = {}

        for channel in channels:
            metrics = await self.prisma.unifiedmetric.find_many(
                where={
                    'storeId': store_id,
                    'channel': channel,
                    'date': {'gte': start_date, 'lte': end_date},
                    'source': None
                }
            )

            if metrics:
                total_revenue = sum(m.revenue for m in metrics)
                total_cost = sum(m.cost for m in metrics)
                roas = total_revenue / total_cost if total_cost > 0 else Decimal(0)

                performance[channel] = {
                    'revenue': float(total_revenue),
                    'cost': float(total_cost),
                    'roas': float(roas),
                }

        # Calculate recommended allocation
        recommendations = await self._calculate_allocation(
            performance=performance,
            total_budget=float(total_budget),
        )

        return {
            'total_budget': float(total_budget),
            'current_performance': performance,
            'recommendations': recommendations,
        }

    async def _calculate_allocation(
        self,
        performance: Dict[str, Dict],
        total_budget: float,
    ) -> Dict[str, float]:
        """Calculate optimal budget allocation"""
        # Weight channels by ROAS
        total_roas = sum(p['roas'] for p in performance.values())

        if total_roas == 0:
            # Equal distribution if no ROAS data
            per_channel = total_budget / len(performance)
            return {channel: per_channel for channel in performance.keys()}

        # Allocate proportionally to ROAS
        allocation = {}
        for channel, perf in performance.items():
            weight = perf['roas'] / total_roas
            allocation[channel] = total_budget * weight

        return allocation

    async def generate_insights(
        self,
        store_id: str,
    ) -> List[Dict[str, Any]]:
        """Generate actionable marketing insights"""
        insights = []

        # Get last 30 days vs previous 30 days
        today = datetime.utcnow()
        current_start = today - timedelta(days=30)
        previous_start = today - timedelta(days=60)
        previous_end = current_start

        channels = ['seo', 'influencer']

        for channel in channels:
            current_metrics = await self.prisma.unifiedmetric.find_many(
                where={
                    'storeId': store_id,
                    'channel': channel,
                    'date': {'gte': current_start, 'lte': today}
                }
            )

            previous_metrics = await self.prisma.unifiedmetric.find_many(
                where={
                    'storeId': store_id,
                    'channel': channel,
                    'date': {'gte': previous_start, 'lte': previous_end}
                }
            )

            if current_metrics and previous_metrics:
                current_revenue = sum(m.revenue for m in current_metrics)
                previous_revenue = sum(m.revenue for m in previous_metrics)

                # Declining performance
                if current_revenue < previous_revenue * 0.8:  # 20% decline
                    insight = await self.prisma.marketinginsight.create(
                        data={
                            'storeId': store_id,
                            'type': 'warning',
                            'category': 'performance',
                            'title': f"{channel.upper()} revenue declined 20%+",
                            'description': f"Revenue from {channel} has decreased significantly. Consider reviewing your strategy.",
                            'priority': 'high',
                            'impact': float(previous_revenue - current_revenue),
                            'suggestedAction': f"Review {channel} performance and adjust strategy",
                        }
                    )
                    insights.append(insight)

                # Strong growth
                elif current_revenue > previous_revenue * 1.5:  # 50% growth
                    insight = await self.prisma.marketinginsight.create(
                        data={
                            'storeId': store_id,
                            'type': 'opportunity',
                            'category': 'performance',
                            'title': f"{channel.upper()} revenue up 50%+",
                            'description': f"{channel} is performing exceptionally. Consider increasing budget.",
                            'priority': 'high',
                            'impact': float(current_revenue - previous_revenue),
                            'suggestedAction': f"Increase {channel} budget by 25%",
                        }
                    )
                    insights.append(insight)

        return insights
```

---

## API Endpoints

```python
# routers/marketing_analytics.py

from fastapi import APIRouter, Depends, HTTPException, Query
from typing import Optional
from datetime import datetime, timedelta
from decimal import Decimal

from ..dependencies import get_current_user, get_prisma
from ..services.marketing.integration import MarketingIntegrationOrchestrator
from ..services.marketing.attribution import ROIAttributionEngine
from ..services.marketing.performance_comparison import PerformanceComparisonService
from ..services.marketing.budget_optimizer import BudgetOptimizer

router = APIRouter(prefix="/api/marketing/analytics", tags=["marketing-analytics"])

@router.get("/dashboard")
async def get_marketing_dashboard(
    start_date: Optional[datetime] = None,
    end_date: Optional[datetime] = None,
    current_user = Depends(get_current_user),
    comparison_service: PerformanceComparisonService = Depends(),
):
    """Get unified marketing dashboard data"""
    if not end_date:
        end_date = datetime.utcnow()
    if not start_date:
        start_date = end_date - timedelta(days=30)

    # Get channel comparison
    comparison = await comparison_service.compare_channels(
        store_id=current_user.store_id,
        start_date=start_date,
        end_date=end_date,
    )

    # Get previous period comparison
    period_length = (end_date - start_date).days
    previous_end = start_date
    previous_start = previous_end - timedelta(days=period_length)

    period_comparison = await comparison_service.compare_periods(
        store_id=current_user.store_id,
        current_start=start_date,
        current_end=end_date,
        previous_start=previous_start,
        previous_end=previous_end,
    )

    return {
        'success': True,
        'data': {
            'channels': comparison,
            'vs_previous_period': period_comparison,
            'date_range': {
                'start': start_date.isoformat(),
                'end': end_date.isoformat(),
            }
        }
    }

@router.get("/attribution")
async def get_attribution_data(
    start_date: Optional[datetime] = None,
    end_date: Optional[datetime] = None,
    model: str = Query('linear', regex='^(linear|time_decay|last_touch)$'),
    current_user = Depends(get_current_user),
    attribution_engine: ROIAttributionEngine = Depends(),
):
    """Get attribution data"""
    if not end_date:
        end_date = datetime.utcnow()
    if not start_date:
        start_date = end_date - timedelta(days=30)

    attribution = await attribution_engine.get_channel_attribution(
        store_id=current_user.store_id,
        start_date=start_date,
        end_date=end_date,
        model=model,
    )

    return {
        'success': True,
        'data': {
            'model': model,
            'attribution': {k: float(v) for k, v in attribution.items()},
        }
    }

@router.get("/roi/{channel}")
async def get_channel_roi(
    channel: str,
    start_date: Optional[datetime] = None,
    end_date: Optional[datetime] = None,
    current_user = Depends(get_current_user),
    attribution_engine: ROIAttributionEngine = Depends(),
):
    """Get ROI for specific channel"""
    if not end_date:
        end_date = datetime.utcnow()
    if not start_date:
        start_date = end_date - timedelta(days=30)

    roi = await attribution_engine.calculate_channel_roi(
        store_id=current_user.store_id,
        channel=channel,
        start_date=start_date,
        end_date=end_date,
    )

    return {
        'success': True,
        'data': {
            'channel': channel,
            **{k: float(v) for k, v in roi.items()}
        }
    }

@router.get("/benchmarks/{channel}")
async def get_channel_benchmarks(
    channel: str,
    industry: str = "general",
    start_date: Optional[datetime] = None,
    end_date: Optional[datetime] = None,
    current_user = Depends(get_current_user),
    comparison_service: PerformanceComparisonService = Depends(),
):
    """Get channel performance vs benchmarks"""
    if not end_date:
        end_date = datetime.utcnow()
    if not start_date:
        start_date = end_date - timedelta(days=30)

    comparison = await comparison_service.compare_to_benchmarks(
        store_id=current_user.store_id,
        channel=channel,
        start_date=start_date,
        end_date=end_date,
        industry=industry,
    )

    return {
        'success': True,
        'data': comparison,
    }

@router.get("/budget/optimize")
async def optimize_budget(
    total_budget: Decimal,
    current_user = Depends(get_current_user),
    optimizer: BudgetOptimizer = Depends(),
):
    """Get budget optimization recommendations"""
    recommendations = await optimizer.optimize_budget(
        store_id=current_user.store_id,
        total_budget=total_budget,
    )

    return {
        'success': True,
        'data': recommendations,
    }

@router.get("/insights")
async def get_insights(
    status: Optional[str] = None,
    current_user = Depends(get_current_user),
    prisma = Depends(get_prisma),
    optimizer: BudgetOptimizer = Depends(),
):
    """Get marketing insights"""
    # Generate fresh insights
    await optimizer.generate_insights(current_user.store_id)

    # Fetch insights
    where = {'storeId': current_user.store_id}
    if status:
        where['status'] = status

    insights = await prisma.marketinginsight.find_many(
        where=where,
        order_by=[
            {'priority': 'desc'},
            {'createdAt': 'desc'}
        ],
        take=50,
    )

    return {
        'success': True,
        'data': insights,
    }

@router.get("/calendar")
async def get_unified_calendar(
    start_date: datetime,
    end_date: datetime,
    current_user = Depends(get_current_user),
    orchestrator: MarketingIntegrationOrchestrator = Depends(),
):
    """Get unified content calendar"""
    calendar = await orchestrator.get_unified_calendar(
        store_id=current_user.store_id,
        start_date=start_date,
        end_date=end_date,
    )

    return {
        'success': True,
        'data': calendar,
    }
```

---

## UI Components

### Unified Marketing Dashboard

```typescript
// app/marketing/dashboard/page.tsx

'use client';

import { useState, useEffect } from 'react';
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card';
import { Tabs, TabsList, TabsTrigger, TabsContent } from '@/components/ui/tabs';
import { LineChart, Line, BarChart, Bar, PieChart, Pie, Cell, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';
import { TrendingUp, TrendingDown, DollarSign, Target, Users, Activity } from 'lucide-react';

export default function MarketingDashboard() {
    const [data, setData] = useState<any>(null);
    const [loading, setLoading] = useState(true);
    const [dateRange, setDateRange] = useState('30d');

    useEffect(() => {
        loadDashboard();
    }, [dateRange]);

    const loadDashboard = async () => {
        const response = await fetch(`/api/marketing/analytics/dashboard?days=${dateRange}`);
        const result = await response.json();

        if (result.success) {
            setData(result.data);
        }

        setLoading(false);
    };

    if (loading) return <div>Loading...</div>;

    const { channels, vs_previous_period } = data;

    return (
        <div className="container mx-auto p-6">
            <h1 className="text-3xl font-bold mb-6">Marketing Dashboard</h1>

            {/* Overview Stats */}
            <div className="grid grid-cols-4 gap-4 mb-6">
                <StatsCard
                    title="Total Revenue"
                    value={`$${channels.overall.revenue.toLocaleString()}`}
                    change={vs_previous_period?.overall?.change?.revenue}
                    icon={<DollarSign className="h-6 w-6" />}
                />
                <StatsCard
                    title="Marketing Spend"
                    value={`$${channels.overall.cost.toLocaleString()}`}
                    change={vs_previous_period?.overall?.change?.cost}
                    icon={<Target className="h-6 w-6" />}
                />
                <StatsCard
                    title="ROAS"
                    value={channels.overall.roas.toFixed(2) + 'x'}
                    change={vs_previous_period?.overall?.change?.roas}
                    icon={<TrendingUp className="h-6 w-6" />}
                />
                <StatsCard
                    title="Profit"
                    value={`$${channels.overall.profit.toLocaleString()}`}
                    change={vs_previous_period?.overall?.change?.profit}
                    icon={<Activity className="h-6 w-6" />}
                />
            </div>

            {/* Channel Breakdown */}
            <div className="grid grid-cols-2 gap-6 mb-6">
                <Card>
                    <CardHeader>
                        <CardTitle>Revenue by Channel</CardTitle>
                    </CardHeader>
                    <CardContent>
                        <ResponsiveContainer width="100%" height={300}>
                            <PieChart>
                                <Pie
                                    data={Object.entries(channels).filter(([k]) => k !== 'overall').map(([channel, data]: any) => ({
                                        name: channel.toUpperCase(),
                                        value: data.revenue
                                    }))}
                                    dataKey="value"
                                    nameKey="name"
                                    cx="50%"
                                    cy="50%"
                                    outerRadius={100}
                                    label
                                >
                                    {Object.keys(channels).filter(k => k !== 'overall').map((entry, index) => (
                                        <Cell key={`cell-${index}`} fill={COLORS[index % COLORS.length]} />
                                    ))}
                                </Pie>
                                <Tooltip />
                            </PieChart>
                        </ResponsiveContainer>
                    </CardContent>
                </Card>

                <Card>
                    <CardHeader>
                        <CardTitle>ROAS by Channel</CardTitle>
                    </CardHeader>
                    <CardContent>
                        <ResponsiveContainer width="100%" height={300}>
                            <BarChart data={Object.entries(channels).filter(([k]) => k !== 'overall').map(([channel, data]: any) => ({
                                channel: channel.toUpperCase(),
                                roas: data.roas
                            }))}>
                                <CartesianGrid strokeDasharray="3 3" />
                                <XAxis dataKey="channel" />
                                <YAxis />
                                <Tooltip />
                                <Bar dataKey="roas" fill="#8884d8" />
                            </BarChart>
                        </ResponsiveContainer>
                    </CardContent>
                </Card>
            </div>

            {/* Channel Details */}
            <Tabs defaultValue="seo" className="mb-6">
                <TabsList>
                    <TabsTrigger value="seo">SEO</TabsTrigger>
                    <TabsTrigger value="influencer">Influencer</TabsTrigger>
                    <TabsTrigger value="social">Social</TabsTrigger>
                    <TabsTrigger value="ads">Ads</TabsTrigger>
                </TabsList>

                {Object.entries(channels).filter(([k]) => k !== 'overall').map(([channel, data]: any) => (
                    <TabsContent key={channel} value={channel}>
                        <ChannelDetailView channel={channel} data={data} comparison={vs_previous_period?.[channel]} />
                    </TabsContent>
                ))}
            </Tabs>
        </div>
    );
}

function StatsCard({ title, value, change, icon }: any) {
    const isPositive = change?.direction === 'up';
    const isNegative = change?.direction === 'down';

    return (
        <Card>
            <CardContent className="p-6">
                <div className="flex items-center justify-between">
                    <div>
                        <p className="text-sm text-gray-500">{title}</p>
                        <p className="text-2xl font-bold mt-1">{value}</p>
                        {change && (
                            <div className={`flex items-center mt-2 text-sm ${isPositive ? 'text-green-600' : isNegative ? 'text-red-600' : 'text-gray-600'}`}>
                                {isPositive ? <TrendingUp className="h-4 w-4 mr-1" /> : isNegative ? <TrendingDown className="h-4 w-4 mr-1" /> : null}
                                <span>{change.percentage.toFixed(1)}%</span>
                            </div>
                        )}
                    </div>
                    <div className="text-gray-400">{icon}</div>
                </div>
            </CardContent>
        </Card>
    );
}

function ChannelDetailView({ channel, data, comparison }: any) {
    return (
        <div className="grid grid-cols-4 gap-4">
            <MetricCard label="Revenue" value={`$${data.revenue.toLocaleString()}`} change={comparison?.change?.revenue} />
            <MetricCard label="Cost" value={`$${data.cost.toLocaleString()}`} change={comparison?.change?.cost} />
            <MetricCard label="ROAS" value={`${data.roas.toFixed(2)}x`} change={comparison?.change?.roas} />
            <MetricCard label="Profit" value={`$${data.profit.toLocaleString()}`} change={comparison?.change?.profit} />
            <MetricCard label="Conversions" value={data.conversions} />
            <MetricCard label="Sessions" value={data.sessions.toLocaleString()} />
            <MetricCard label="CPA" value={`$${data.cpa.toFixed(2)}`} />
            <MetricCard label="Conv. Rate" value={`${data.conversion_rate.toFixed(2)}%`} />
        </div>
    );
}

function MetricCard({ label, value, change }: any) {
    return (
        <Card>
            <CardContent className="p-4">
                <p className="text-sm text-gray-500">{label}</p>
                <p className="text-xl font-bold mt-1">{value}</p>
                {change && (
                    <p className={`text-sm mt-1 ${change.direction === 'up' ? 'text-green-600' : change.direction === 'down' ? 'text-red-600' : 'text-gray-600'}`}>
                        {change.percentage > 0 ? '+' : ''}{change.percentage.toFixed(1)}%
                    </p>
                )}
            </CardContent>
        </Card>
    );
}

const COLORS = ['#0088FE', '#00C49F', '#FFBB28', '#FF8042'];
```

---

## Celery Workers

```python
# workers/marketing_analytics.py

from celery import shared_task
from datetime import datetime, timedelta
from prisma import Prisma

@shared_task
def sync_marketing_data():
    """
    Sync all marketing data daily
    Runs at 1 AM daily
    """
    prisma = Prisma()
    prisma.connect()

    try:
        from services.marketing.integration import MarketingIntegrationOrchestrator

        orchestrator = MarketingIntegrationOrchestrator(prisma)

        # Get all active stores
        stores = prisma.store.find_many(where={'status': 'active'})

        for store in stores:
            try:
                results = orchestrator.sync_all_data(store.id)
                print(f"Synced store {store.id}: {results}")
            except Exception as e:
                print(f"Error syncing store {store.id}: {e}")

    finally:
        prisma.disconnect()

@shared_task
def generate_daily_insights():
    """
    Generate insights for all stores
    Runs at 2 AM daily
    """
    prisma = Prisma()
    prisma.connect()

    try:
        from services.marketing.budget_optimizer import BudgetOptimizer
        from openai import AsyncOpenAI

        optimizer = BudgetOptimizer(prisma, AsyncOpenAI())

        stores = prisma.store.find_many(where={'status': 'active'})

        for store in stores:
            try:
                insights = optimizer.generate_insights(store.id)
                print(f"Generated {len(insights)} insights for store {store.id}")
            except Exception as e:
                print(f"Error generating insights for {store.id}: {e}")

    finally:
        prisma.disconnect()
```

---

## Production Readiness Checklist

### Performance Optimization
- [ ] Database queries optimized with proper indexes
- [ ] Redis caching implemented for frequently accessed data
- [ ] API response times < 500ms for 95th percentile
- [ ] Frontend bundle size optimized
- [ ] Image optimization and lazy loading
- [ ] Database connection pooling configured
- [ ] CDN configured for static assets

### Security
- [ ] All API endpoints require authentication
- [ ] Rate limiting implemented on all endpoints
- [ ] Input validation on all user inputs
- [ ] SQL injection prevention verified
- [ ] XSS protection enabled
- [ ] CORS properly configured
- [ ] Secrets stored in environment variables
- [ ] HTTPS enforced
- [ ] Security headers configured

### Monitoring & Logging
- [ ] Error tracking (Sentry/Rollbar) configured
- [ ] Application performance monitoring (New Relic/DataDog) set up
- [ ] Log aggregation (Elasticsearch/Splunk) configured
- [ ] Uptime monitoring (Pingdom/UptimeRobot) enabled
- [ ] Alert rules configured for critical errors
- [ ] Dashboard for system health metrics
- [ ] Database performance monitoring

### Testing
- [ ] Unit tests passing (>80% coverage)
- [ ] Integration tests passing
- [ ] End-to-end tests passing
- [ ] Load testing completed (handle 1000 req/sec)
- [ ] Security audit completed
- [ ] Browser compatibility tested
- [ ] Mobile responsiveness verified

### Documentation
- [ ] API documentation complete
- [ ] User guide written
- [ ] Admin documentation ready
- [ ] Deployment guide finalized
- [ ] Troubleshooting guide created
- [ ] Database schema documented
- [ ] Architecture diagrams updated

---

## Definition of Done

### Technical Requirements
- [ ] All systems integrated and data syncing
- [ ] Unified dashboard functional with real-time updates
- [ ] ROI attribution accurate across all channels
- [ ] Performance benchmarking operational
- [ ] Budget optimization recommendations working
- [ ] Insights generation automated
- [ ] Production deployment ready
- [ ] All API endpoints tested and documented

### Quality Checks
- [ ] Dashboard loads in < 2 seconds
- [ ] Data sync latency < 5 minutes
- [ ] Attribution accuracy > 95%
- [ ] Zero critical bugs
- [ ] Security audit passed
- [ ] Load testing passed (1000 concurrent users)
- [ ] Mobile responsiveness verified

### Documentation
- [ ] User guides complete with screenshots
- [ ] API documentation finalized
- [ ] Admin documentation ready
- [ ] Training materials prepared
- [ ] Video tutorials recorded

---

## Testing Checklist

### Integration Tests
- [ ] SEO to unified metrics sync
- [ ] Influencer to unified metrics sync
- [ ] Attribution calculation accuracy
- [ ] Cross-channel analytics aggregation
- [ ] Dashboard data loading
- [ ] Budget optimization logic

### Performance Tests
- [ ] Dashboard loads with 10K metrics in < 2s
- [ ] API endpoints respond in < 500ms
- [ ] Data sync completes in < 5 minutes
- [ ] Handle 1000 concurrent dashboard users

### UI Tests
- [ ] Dashboard renders all charts correctly
- [ ] Date range filters work
- [ ] Channel tabs function properly
- [ ] Export functionality works
- [ ] Mobile view responsive

---

## Success Metrics

### Integration
- **Target:** 100% data sync success rate
- **Target:** < 5 minute sync latency
- **Target:** Zero attribution errors

### Performance
- **Target:** Dashboard load < 2 seconds (p95)
- **Target:** API response < 500ms (p95)
- **Target:** Support 1000 concurrent users

### ROI Accuracy
- **Target:** 95%+ attribution accuracy
- **Target:** < 1% revenue reconciliation variance

### User Adoption
- **Target:** 90%+ of users view dashboard weekly
- **Target:** 70%+ of users act on insights
- **Target:** 80%+ satisfaction score

---

## Phase 4 Completion

Upon completion of Sprint 6, Phase 4 (Expansion) will be fully operational with:

1. ✅ **SEO Foundation** - AI-powered keyword research and content planning
2. ✅ **Article Generation** - Automated SEO content creation with optimization
3. ✅ **SEO Publishing & Analytics** - CMS integration and Search Console tracking
4. ✅ **Influencer Discovery** - Multi-platform influencer search with AI matching
5. ✅ **Influencer Outreach** - Automated campaign management and personalization
6. ✅ **Integration & Analytics** - Unified marketing dashboard with ROI tracking

**Platform Status:** Zyenta is now a complete "Full Autopilot" e-commerce platform with autonomous Genesis (creation), Operations (fulfillment), and Growth (marketing) engines!

**Key Achievements:**
- Complete autonomous store creation (Phase 1)
- Full order fulfillment automation (Phase 2)
- Multi-channel social media marketing (Phase 3)
- SEO and influencer marketing automation (Phase 4)
- Unified marketing analytics and ROI tracking

**Next Phase:** Future expansions could include:
- Email marketing automation
- Affiliate program management
- Customer retention and loyalty systems
- Advanced AI personalization
- Multi-store management
- White-label reseller capabilities
