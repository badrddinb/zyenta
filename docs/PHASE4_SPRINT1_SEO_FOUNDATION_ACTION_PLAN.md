# Phase 4 - Sprint 1: SEO Foundation - Action Plan

## Sprint Overview

**Duration:** 2 weeks
**Team Size:** 3-4 developers (2 backend, 1 frontend, 1 AI/ML specialist)
**Sprint Goal:** Build the foundational SEO infrastructure including keyword research, content strategy, topic clustering, and content planning management system.

### Objectives
- [ ] Implement AI-powered keyword researcher
- [ ] Build content strategist with topic clustering
- [ ] Create content plan management system
- [ ] Develop SEO planning dashboard UI
- [ ] Integrate with SEO data providers

### Prerequisites
- Phase 3 (Growth Hub) fully operational
- LLM service infrastructure ready
- Database schema updated with SEO models
- Admin dashboard framework in place

---

## Day-by-Day Breakdown

### Days 1-3: AI-Powered Keyword Research System

#### Tasks
1. **Implement Keyword Researcher Core**
   ```python
   # app/agents/seo/keyword_researcher.py
   from typing import List, Dict, Any, Optional
   from dataclasses import dataclass
   import httpx
   from app.services.llm import LLMService
   from app.db.prisma import prisma

   @dataclass
   class KeywordData:
       keyword: str
       search_volume: int
       difficulty: float  # 0-100
       cpc: float
       intent: str  # informational, commercial, transactional, navigational
       trend: str  # rising, stable, declining
       related_keywords: List[str]

   class KeywordResearcher:
       """AI-powered keyword research for SEO"""

       def __init__(self):
           self.llm = LLMService()
           self.serp_api_key = settings.SERP_API_KEY

       async def research(
           self,
           niche: str,
           products: List[Dict],
           competitor_domains: List[str],
           existing_keywords: List[str],
           target_count: int = 100,
       ) -> List[KeywordData]:
           """Conduct comprehensive keyword research"""

           # 1. Generate seed keywords using AI
           seed_keywords = await self._generate_seed_keywords(niche, products)

           # 2. Expand keywords using AI
           expanded = await self._expand_keywords(seed_keywords, niche)

           # 3. Analyze competitors
           competitor_keywords = await self._analyze_competitors(competitor_domains)

           # 4. Combine and deduplicate
           combined = list(set(seed_keywords + expanded + competitor_keywords))

           # 5. Filter out existing keywords
           new_keywords = [k for k in combined if k not in existing_keywords]

           # 6. Get metrics for keywords
           keyword_data = await self._get_keyword_metrics(new_keywords)

           # 7. Score and rank keywords
           scored = self._score_keywords(keyword_data, niche)

           # 8. Select top keywords
           top_keywords = sorted(scored, key=lambda x: x["score"], reverse=True)[:target_count]

           return [
               KeywordData(
                   keyword=k["keyword"],
                   search_volume=k["search_volume"],
                   difficulty=k["difficulty"],
                   cpc=k["cpc"],
                   intent=k["intent"],
                   trend=k["trend"],
                   related_keywords=k["related"],
               )
               for k in top_keywords
           ]

       async def _generate_seed_keywords(
           self,
           niche: str,
           products: List[Dict],
       ) -> List[str]:
           """Generate seed keywords using AI"""

           product_titles = [p["title"] for p in products[:20]]

           prompt = f"""
           Generate 50 SEO seed keywords for an e-commerce store in the {niche} niche.

           The store sells products like:
           {product_titles}

           Generate keywords across these categories:
           1. Product-focused keywords (what people search to buy)
           2. Informational keywords (how-to, guides, comparisons)
           3. Problem-focused keywords (pain points the products solve)
           4. Long-tail keywords (specific searches with buying intent)

           Return as a JSON array of keywords.
           Example: {{"keywords": ["keyword 1", "keyword 2", ...]}}
           """

           response = await self.llm.generate(
               prompt=prompt,
               response_format={"type": "json_object"},
           )

           return response.get("keywords", [])

       async def _expand_keywords(
           self,
           seed_keywords: List[str],
           niche: str,
       ) -> List[str]:
           """Expand seed keywords to find more opportunities"""

           prompt = f"""
           Given these seed keywords for a {niche} store:
           {seed_keywords[:20]}

           Generate 100 related long-tail keyword variations including:
           - Question-based keywords (how, what, why, when, where)
           - Comparison keywords (vs, alternative, best)
           - Modifier keywords (cheap, best, top, review)
           - Year-based keywords (2025, latest)

           Return as a JSON array.
           Example: {{"keywords": ["keyword 1", "keyword 2", ...]}}
           """

           response = await self.llm.generate(
               prompt=prompt,
               response_format={"type": "json_object"},
           )

           return response.get("keywords", [])

       async def _analyze_competitors(
           self,
           domains: List[str],
       ) -> List[str]:
           """Analyze competitor keywords"""

           prompt = f"""
           Analyze these e-commerce competitors and suggest keywords they might rank for:
           {domains}

           Generate 30 likely keyword opportunities based on typical e-commerce SEO.
           Return as a JSON array.
           Example: {{"keywords": ["keyword 1", "keyword 2", ...]}}
           """

           response = await self.llm.generate(
               prompt=prompt,
               response_format={"type": "json_object"},
           )

           return response.get("keywords", [])

       async def _get_keyword_metrics(
           self,
           keywords: List[str],
       ) -> List[Dict[str, Any]]:
           """Get search metrics for keywords using AI estimation"""

           keyword_metrics = []

           for batch in self._batch(keywords, 20):
               prompt = f"""
               Estimate SEO metrics for these keywords:
               {batch}

               For each keyword, provide realistic estimates for:
               - search_volume (monthly searches, integer)
               - difficulty (0-100, higher = harder to rank)
               - cpc (cost per click in USD)
               - intent (informational, commercial, transactional, navigational)
               - trend (rising, stable, declining)
               - related (3-5 related keywords)

               Return as JSON array with objects for each keyword.
               Example: {{
                   "keywords": [
                       {{
                           "keyword": "example keyword",
                           "search_volume": 1000,
                           "difficulty": 45,
                           "cpc": 1.25,
                           "intent": "informational",
                           "trend": "stable",
                           "related": ["related 1", "related 2"]
                       }}
                   ]
               }}
               """

               response = await self.llm.generate(
                   prompt=prompt,
                   response_format={"type": "json_object"},
               )

               keyword_metrics.extend(response.get("keywords", []))

           return keyword_metrics

       def _score_keywords(
           self,
           keyword_data: List[Dict],
           niche: str,
       ) -> List[Dict]:
           """Score keywords based on opportunity"""

           for kw in keyword_data:
               # Base score from search volume (log scale)
               volume_score = min(100, kw.get("search_volume", 0) / 100)

               # Difficulty penalty (lower difficulty = higher score)
               difficulty = kw.get("difficulty", 50)
               difficulty_score = (100 - difficulty) / 100

               # Intent bonus (transactional > commercial > informational)
               intent_scores = {
                   "transactional": 1.5,
                   "commercial": 1.3,
                   "informational": 1.0,
                   "navigational": 0.5,
               }
               intent_multiplier = intent_scores.get(kw.get("intent", "informational"), 1.0)

               # Trend bonus
               trend_scores = {"rising": 1.2, "stable": 1.0, "declining": 0.8}
               trend_multiplier = trend_scores.get(kw.get("trend", "stable"), 1.0)

               # Calculate final score
               kw["score"] = (
                   volume_score * difficulty_score * intent_multiplier * trend_multiplier
               )

           return keyword_data

       def _batch(self, items: List, size: int):
           """Split list into batches"""
           for i in range(0, len(items), size):
               yield items[i : i + size]
   ```

2. **Add Keyword Management API**
   ```python
   # app/api/seo/keywords.py
   from fastapi import APIRouter, Depends
   from app.auth import get_current_user
   from app.agents.seo import KeywordResearcher

   router = APIRouter()

   @router.post("/research")
   async def research_keywords(
       niche: str,
       target_count: int = 100,
       user = Depends(get_current_user),
   ):
       """Research keywords for a store"""

       from app.db.prisma import prisma

       store = await prisma.store.find_unique(
           where={"userId": user.id},
           include={"products": {"take": 20}}
       )

       # Get existing keywords
       existing_articles = await prisma.blogarticle.find_many(
           where={"storeId": store.id},
           select={"targetKeyword": True}
       )
       existing_keywords = [a.targetKeyword for a in existing_articles]

       # Research keywords
       researcher = KeywordResearcher()
       keywords = await researcher.research(
           niche=niche or store.niche,
           products=store.products,
           competitor_domains=[],
           existing_keywords=existing_keywords,
           target_count=target_count,
       )

       return {
           "keywords": [
               {
                   "keyword": k.keyword,
                   "search_volume": k.search_volume,
                   "difficulty": k.difficulty,
                   "cpc": k.cpc,
                   "intent": k.intent,
                   "trend": k.trend,
                   "related_keywords": k.related_keywords,
               }
               for k in keywords
           ]
       }
   ```

#### Deliverables
- [ ] Keyword researcher implemented
- [ ] AI seed keyword generation working
- [ ] Keyword expansion functional
- [ ] Keyword scoring system operational

---

### Days 4-6: Content Strategy & Topic Clustering

#### Tasks
1. **Implement Content Strategist**
   ```python
   # app/agents/seo/content_strategist.py
   from typing import List, Dict, Any
   from app.services.llm import LLMService

   class ContentStrategist:
       """AI-powered content strategy and topic clustering"""

       def __init__(self):
           self.llm = LLMService()

       async def create_clusters(
           self,
           keywords: List[Any],
           niche: str,
           products: List[Dict],
       ) -> List[Dict[str, Any]]:
           """Create topic clusters from keywords"""

           # Group keywords by topic using AI
           keyword_list = [k.keyword for k in keywords]

           prompt = f"""
           Organize these SEO keywords into topic clusters for a {niche} e-commerce blog.

           Keywords:
           {keyword_list}

           Create 5-10 topic clusters where:
           1. Each cluster has a main pillar topic
           2. Each cluster contains 5-15 related keywords
           3. Clusters should cover the buyer's journey (awareness, consideration, decision)
           4. Consider search intent grouping

           Return as JSON with this structure:
           {{
               "clusters": [
                   {{
                       "name": "Cluster name",
                       "pillar_keyword": "main keyword",
                       "intent": "informational/commercial/transactional",
                       "keywords": ["keyword1", "keyword2", ...],
                       "description": "Brief description of cluster focus"
                   }}
               ]
           }}
           """

           response = await self.llm.generate(
               prompt=prompt,
               response_format={"type": "json_object"},
           )

           return response.get("clusters", [])

       async def plan_articles(
           self,
           clusters: List[Dict],
           total_articles: int,
           existing_content: List[Any],
       ) -> List[Dict[str, Any]]:
           """Plan articles from topic clusters"""

           articles = []

           # Distribute articles across clusters
           articles_per_cluster = total_articles // len(clusters)

           for cluster in clusters:
               cluster_keywords = cluster["keywords"]

               prompt = f"""
               Plan {articles_per_cluster} blog articles for this topic cluster:

               Cluster: {cluster["name"]}
               Pillar Keyword: {cluster["pillar_keyword"]}
               Intent: {cluster["intent"]}
               Keywords: {cluster_keywords}

               For each article, specify:
               - topic (article title idea)
               - target_keyword (primary keyword)
               - secondary_keywords (2-3 supporting keywords)
               - content_type (guide, listicle, how-to, comparison, review)
               - word_count (1000-2500)
               - priority (high, medium, low based on opportunity)

               Return as JSON array:
               {{
                   "articles": [
                       {{
                           "topic": "Article title",
                           "target_keyword": "keyword",
                           "secondary_keywords": ["kw1", "kw2"],
                           "content_type": "guide",
                           "word_count": 1500,
                           "priority": "high"
                       }}
                   ]
               }}
               """

               response = await self.llm.generate(
                   prompt=prompt,
                   response_format={"type": "json_object"},
               )

               cluster_articles = response.get("articles", [])

               # Add cluster metadata
               for article in cluster_articles:
                   article["cluster"] = cluster["name"]
                   article["pillar_keyword"] = cluster["pillar_keyword"]

               articles.extend(cluster_articles)

           return articles[:total_articles]

       async def generate_outline(
           self,
           topic: str,
           target_keyword: str,
           secondary_keywords: List[str],
           word_count_target: int,
       ) -> List[Dict]:
           """Generate article outline"""

           prompt = f"""
           Create a detailed article outline for the topic: "{topic}"

           Target keyword: {target_keyword}
           Secondary keywords to include: {secondary_keywords}
           Target word count: {word_count_target}

           Requirements:
           1. Include an engaging introduction section
           2. Create 4-7 main sections with H2 headings
           3. Add 2-3 subsections (H3) where appropriate
           4. Naturally incorporate keywords in headings
           5. Include a conclusion section
           6. Consider FAQ section for common questions

           Return as JSON with this structure:
           {{
               "outline": [
                   {{
                       "heading": "H2 heading text",
                       "level": 2,
                       "key_points": ["point 1", "point 2"],
                       "target_words": 200,
                       "subsections": [
                           {{
                               "heading": "H3 subheading",
                               "level": 3,
                               "key_points": ["point 1"]
                           }}
                       ]
                   }}
               ]
           }}
           """

           response = await self.llm.generate(
               prompt=prompt,
               response_format={"type": "json_object"},
           )

           return response.get("outline", [])
   ```

2. **Create Content Plan Management**
   ```python
   # app/services/content_plan_service.py
   from datetime import datetime, timedelta
   from typing import List, Dict, Any
   from decimal import Decimal

   class ContentPlanService:
       """Manage SEO content plans"""

       async def create_plan(
           self,
           store_id: str,
           months: int = 3,
           articles_per_month: int = 8,
       ) -> Any:
           """Create a comprehensive content plan"""

           from app.db.prisma import prisma
           from app.agents.seo import KeywordResearcher, ContentStrategist

           store = await prisma.store.find_unique(
               where={"id": store_id},
               include={"products": {"take": 20}}
           )

           # Research keywords
           researcher = KeywordResearcher()
           keywords = await researcher.research(
               niche=store.niche,
               products=store.products,
               competitor_domains=[],
               existing_keywords=[],
               target_count=100,
           )

           # Create topic clusters
           strategist = ContentStrategist()
           clusters = await strategist.create_clusters(
               keywords=keywords,
               niche=store.niche,
               products=store.products,
           )

           # Plan articles
           total_articles = months * articles_per_month
           articles = await strategist.plan_articles(
               clusters=clusters,
               total_articles=total_articles,
               existing_content=[],
           )

           # Create publishing schedule
           schedule = self._create_publishing_schedule(
               articles=articles,
               months=months,
               articles_per_month=articles_per_month,
           )

           # Save content plan
           content_plan = await prisma.contentplan.create(
               data={
                   "storeId": store_id,
                   "startDate": datetime.utcnow(),
                   "endDate": datetime.utcnow() + timedelta(days=months * 30),
                   "status": "active",
                   "keywords": {
                       "keywords": [
                           {
                               "keyword": k.keyword,
                               "search_volume": k.search_volume,
                               "difficulty": k.difficulty,
                               "intent": k.intent,
                           }
                           for k in keywords
                       ]
                   },
                   "clusters": {"clusters": clusters},
                   "schedule": {"schedule": schedule},
               }
           )

           return content_plan

       def _create_publishing_schedule(
           self,
           articles: List[Dict],
           months: int,
           articles_per_month: int,
       ) -> List[Dict[str, Any]]:
           """Create optimal publishing schedule"""

           schedule = []
           current_date = datetime.utcnow()
           days_between = 30 // articles_per_month

           for i, article in enumerate(articles):
               publish_date = current_date + timedelta(days=i * days_between)

               # Prefer Tuesday/Thursday publishing
               while publish_date.weekday() not in [1, 3]:  # Tue, Thu
                   publish_date += timedelta(days=1)

               schedule.append({
                   "topic": article["topic"],
                   "target_keyword": article["target_keyword"],
                   "scheduled_date": publish_date.isoformat(),
                   "priority": article.get("priority", "medium"),
                   "cluster": article.get("cluster"),
               })

           return schedule
   ```

#### Deliverables
- [ ] Topic clustering system working
- [ ] Content strategist operational
- [ ] Article planning functional
- [ ] Publishing schedule generator ready

---

### Days 7-9: SEO Planning Dashboard UI

#### Tasks
1. **Build SEO Dashboard Main Page**
   ```typescript
   // app/dashboard/seo/page.tsx
   'use client';

   import { useState, useEffect } from 'react';
   import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
   import { Button } from '@/components/ui/button';
   import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
   import KeywordResearch from '@/components/seo/KeywordResearch';
   import ContentPlan from '@/components/seo/ContentPlan';
   import TopicClusters from '@/components/seo/TopicClusters';
   import ArticleList from '@/components/seo/ArticleList';

   export default function SEODashboard() {
     const [contentPlan, setContentPlan] = useState<any>(null);
     const [loading, setLoading] = useState(false);

     const createContentPlan = async () => {
       setLoading(true);
       const response = await fetch('/api/seo/content-plan', {
         method: 'POST',
         headers: { 'Content-Type': 'application/json' },
         body: JSON.stringify({
           months: 3,
           articles_per_month: 8,
         }),
       });

       const data = await response.json();
       setContentPlan(data.plan);
       setLoading(false);
     };

     return (
       <div className="container mx-auto p-6">
         <div className="flex justify-between items-center mb-6">
           <h1 className="text-3xl font-bold">SEO Content Hub</h1>
           {!contentPlan && (
             <Button onClick={createContentPlan} disabled={loading}>
               {loading ? 'Creating Plan...' : 'Create Content Plan'}
             </Button>
           )}
         </div>

         <Tabs defaultValue="plan" className="w-full">
           <TabsList>
             <TabsTrigger value="plan">Content Plan</TabsTrigger>
             <TabsTrigger value="keywords">Keyword Research</TabsTrigger>
             <TabsTrigger value="clusters">Topic Clusters</TabsTrigger>
             <TabsTrigger value="articles">Articles</TabsTrigger>
           </TabsList>

           <TabsContent value="plan">
             <ContentPlan plan={contentPlan} />
           </TabsContent>

           <TabsContent value="keywords">
             <KeywordResearch />
           </TabsContent>

           <TabsContent value="clusters">
             <TopicClusters clusters={contentPlan?.clusters} />
           </TabsContent>

           <TabsContent value="articles">
             <ArticleList />
           </TabsContent>
         </Tabs>
       </div>
     );
   }
   ```

2. **Keyword Research UI Component**
   ```typescript
   // components/seo/KeywordResearch.tsx
   'use client';

   import { useState } from 'react';
   import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
   import { Button } from '@/components/ui/button';
   import { Input } from '@/components/ui/input';
   import { Badge } from '@/components/ui/badge';

   export default function KeywordResearch() {
     const [niche, setNiche] = useState('');
     const [keywords, setKeywords] = useState<any[]>([]);
     const [loading, setLoading] = useState(false);

     const researchKeywords = async () => {
       setLoading(true);
       const response = await fetch('/api/seo/keywords/research', {
         method: 'POST',
         headers: { 'Content-Type': 'application/json' },
         body: JSON.stringify({ niche, target_count: 50 }),
       });

       const data = await response.json();
       setKeywords(data.keywords);
       setLoading(false);
     };

     const getIntentColor = (intent: string) => {
       const colors = {
         transactional: 'bg-green-500',
         commercial: 'bg-blue-500',
         informational: 'bg-yellow-500',
         navigational: 'bg-gray-500',
       };
       return colors[intent as keyof typeof colors] || 'bg-gray-500';
     };

     const getDifficultyColor = (difficulty: number) => {
       if (difficulty < 30) return 'text-green-600';
       if (difficulty < 60) return 'text-yellow-600';
       return 'text-red-600';
     };

     return (
       <Card>
         <CardHeader>
           <CardTitle>Keyword Research</CardTitle>
         </CardHeader>
         <CardContent>
           <div className="flex gap-4 mb-6">
             <Input
               placeholder="Enter niche or leave empty to use store niche"
               value={niche}
               onChange={(e) => setNiche(e.target.value)}
             />
             <Button onClick={researchKeywords} disabled={loading}>
               {loading ? 'Researching...' : 'Research Keywords'}
             </Button>
           </div>

           {keywords.length > 0 && (
             <div className="overflow-x-auto">
               <table className="min-w-full divide-y divide-gray-200">
                 <thead>
                   <tr>
                     <th className="text-left">Keyword</th>
                     <th className="text-left">Volume</th>
                     <th className="text-left">Difficulty</th>
                     <th className="text-left">Intent</th>
                     <th className="text-left">Trend</th>
                     <th className="text-left">CPC</th>
                   </tr>
                 </thead>
                 <tbody>
                   {keywords.map((kw, idx) => (
                     <tr key={idx} className="border-b">
                       <td className="py-2 font-medium">{kw.keyword}</td>
                       <td>{kw.search_volume.toLocaleString()}</td>
                       <td className={getDifficultyColor(kw.difficulty)}>
                         {kw.difficulty.toFixed(0)}
                       </td>
                       <td>
                         <Badge className={`${getIntentColor(kw.intent)} text-white`}>
                           {kw.intent}
                         </Badge>
                       </td>
                       <td>{kw.trend}</td>
                       <td>${kw.cpc.toFixed(2)}</td>
                     </tr>
                   ))}
                 </tbody>
               </table>
             </div>
           )}
         </CardContent>
       </Card>
     );
   }
   ```

3. **Topic Clusters Visualization**
   ```typescript
   // components/seo/TopicClusters.tsx
   'use client';

   import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
   import { Badge } from '@/components/ui/badge';

   interface TopicClustersProps {
     clusters: any;
   }

   export default function TopicClusters({ clusters }: TopicClustersProps) {
     if (!clusters?.clusters) {
       return (
         <Card>
           <CardContent className="p-6">
             <p className="text-gray-500">No topic clusters yet. Create a content plan first.</p>
           </CardContent>
         </Card>
       );
     }

     return (
       <div className="grid grid-cols-2 gap-4">
         {clusters.clusters.map((cluster: any, idx: number) => (
           <Card key={idx}>
             <CardHeader>
               <CardTitle className="text-lg">{cluster.name}</CardTitle>
               <Badge className="w-fit mt-2">{cluster.intent}</Badge>
             </CardHeader>
             <CardContent>
               <div className="mb-4">
                 <div className="text-sm text-gray-500 mb-1">Pillar Keyword</div>
                 <div className="font-semibold">{cluster.pillar_keyword}</div>
               </div>

               <div className="mb-4">
                 <div className="text-sm text-gray-500 mb-2">Description</div>
                 <p className="text-sm">{cluster.description}</p>
               </div>

               <div>
                 <div className="text-sm text-gray-500 mb-2">Keywords ({cluster.keywords.length})</div>
                 <div className="flex flex-wrap gap-1">
                   {cluster.keywords.slice(0, 10).map((kw: string, i: number) => (
                     <span key={i} className="text-xs bg-gray-100 px-2 py-1 rounded">
                       {kw}
                     </span>
                   ))}
                   {cluster.keywords.length > 10 && (
                     <span className="text-xs text-gray-500 px-2 py-1">
                       +{cluster.keywords.length - 10} more
                     </span>
                   )}
                 </div>
               </div>
             </CardContent>
           </Card>
         ))}
       </div>
     );
   }
   ```

#### Deliverables
- [ ] SEO dashboard UI deployed
- [ ] Keyword research interface functional
- [ ] Topic clusters visualization working
- [ ] Content plan display complete

---

### Days 10-12: Database Integration & API Endpoints

#### Tasks
1. **Add Prisma Models**
   ```prisma
   // packages/database/prisma/schema.prisma (additions)

   model ContentPlan {
     id          String    @id @default(cuid())
     storeId     String
     store       Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)

     startDate   DateTime
     endDate     DateTime
     status      String    @default("active")

     keywords    Json      // Array of keyword data
     clusters    Json      // Topic clusters
     schedule    Json      // Publishing schedule

     articles    BlogArticle[]

     createdAt   DateTime  @default(now())
     updatedAt   DateTime  @updatedAt

     @@index([storeId])
   }

   model BlogArticle {
     id              String    @id @default(cuid())
     storeId         String
     store           Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)
     contentPlanId   String?
     contentPlan     ContentPlan? @relation(fields: [contentPlanId], references: [id])

     title           String
     slug            String
     metaDescription String
     content         String    @db.Text
     excerpt         String

     targetKeyword   String
     secondaryKeywords String[]
     wordCount       Int       @default(0)
     readingTime     Int       @default(0)

     schemaMarkup    Json?
     internalLinks   Json?

     status          ArticleStatus @default(DRAFT)
     scheduledDate   DateTime?
     publishedDate   DateTime?
     publishedUrl    String?

     impressions     Int       @default(0)
     clicks          Int       @default(0)
     avgPosition     Float?

     createdAt       DateTime  @default(now())
     updatedAt       DateTime  @updatedAt

     @@unique([storeId, slug])
     @@index([storeId])
     @@index([status])
   }

   enum ArticleStatus {
     DRAFT
     REVIEW
     SCHEDULED
     PUBLISHED
     ARCHIVED
   }
   ```

2. **Content Plan API Endpoints**
   ```python
   # app/api/seo/content_plan.py
   from fastapi import APIRouter, Depends
   from app.auth import get_current_user
   from app.services.content_plan_service import ContentPlanService

   router = APIRouter()

   @router.post("/content-plan")
   async def create_content_plan(
       months: int = 3,
       articles_per_month: int = 8,
       user = Depends(get_current_user),
   ):
       """Create a comprehensive SEO content plan"""

       service = ContentPlanService()
       plan = await service.create_plan(
           store_id=user.storeId,
           months=months,
           articles_per_month=articles_per_month,
       )

       return {"plan": plan}

   @router.get("/content-plan/{plan_id}")
   async def get_content_plan(
       plan_id: str,
       user = Depends(get_current_user),
   ):
       """Get content plan details"""

       from app.db.prisma import prisma

       plan = await prisma.contentplan.find_unique(
           where={"id": plan_id},
           include={"articles": True}
       )

       return {"plan": plan}

   @router.get("/content-plans")
   async def list_content_plans(
       user = Depends(get_current_user),
   ):
       """List all content plans for user's store"""

       from app.db.prisma import prisma

       plans = await prisma.contentplan.find_many(
           where={"storeId": user.storeId},
           include={"_count": {"select": {"articles": True}}},
           order_by={"createdAt": "desc"},
       )

       return {"plans": plans}
   ```

3. **Run Database Migration**
   ```bash
   # Generate and apply migration
   npx prisma migrate dev --name add_seo_models
   npx prisma generate
   ```

#### Deliverables
- [ ] Database schema updated
- [ ] Prisma models generated
- [ ] API endpoints functional
- [ ] Data persistence working

---

### Days 13-14: Testing & Documentation

#### Tasks
1. **Integration Tests**
   ```python
   # tests/integration/test_seo_foundation.py
   import pytest
   from fastapi.testclient import TestClient

   @pytest.mark.integration
   async def test_keyword_research_workflow(client: TestClient):
       """Test complete keyword research workflow"""

       # Research keywords
       response = client.post("/api/seo/keywords/research", json={
           "niche": "minimalist home decor",
           "target_count": 50
       })
       assert response.status_code == 200
       data = response.json()
       assert len(data["keywords"]) <= 50
       assert all(k["search_volume"] >= 0 for k in data["keywords"])

   @pytest.mark.integration
   async def test_content_plan_creation(client: TestClient):
       """Test content plan creation"""

       response = client.post("/api/seo/content-plan", json={
           "months": 3,
           "articles_per_month": 8
       })
       assert response.status_code == 200
       plan = response.json()["plan"]

       assert plan["keywords"] is not None
       assert plan["clusters"] is not None
       assert plan["schedule"] is not None

   @pytest.mark.integration
   async def test_topic_clustering(client: TestClient):
       """Test topic clustering functionality"""

       # Create content plan first
       response = client.post("/api/seo/content-plan", json={
           "months": 1,
           "articles_per_month": 4
       })
       plan = response.json()["plan"]

       # Verify clusters
       clusters = plan["clusters"]["clusters"]
       assert len(clusters) >= 3
       assert all("pillar_keyword" in c for c in clusters)
       assert all("keywords" in c for c in clusters)
   ```

2. **User Documentation**
   ```markdown
   # docs/SEO_FOUNDATION_GUIDE.md

   # SEO Foundation User Guide

   ## Overview
   The SEO Foundation system provides AI-powered keyword research, content strategy, and planning for your store's blog.

   ## Getting Started

   ### 1. Create a Content Plan
   - Navigate to Dashboard → SEO
   - Click "Create Content Plan"
   - Choose duration (1-6 months)
   - Set articles per month (4-12 recommended)
   - Click "Generate Plan"

   ### 2. Review Keyword Research
   - View discovered keywords in the Keywords tab
   - Filter by search volume, difficulty, or intent
   - Export keywords for external use

   ### 3. Explore Topic Clusters
   - View topic clusters in the Clusters tab
   - Each cluster groups related keywords
   - Clusters guide your content strategy

   ### 4. Review Article Plan
   - Check the scheduled articles in the Plan tab
   - View publication schedule
   - Adjust priorities as needed

   ## Best Practices

   - Start with 6-8 articles per month
   - Focus on informational content first
   - Target low-difficulty, high-volume keywords
   - Build topic clusters around your products
   - Publish consistently (2x per week recommended)
   ```

#### Deliverables
- [ ] Integration tests passing
- [ ] Unit test coverage > 80%
- [ ] User documentation complete
- [ ] API documentation updated

---

## Definition of Done

### Technical Requirements
- [ ] Keyword researcher generating 100+ keywords
- [ ] Content strategist creating topic clusters
- [ ] Content plan management system operational
- [ ] SEO dashboard UI deployed
- [ ] Database models created and migrated
- [ ] All API endpoints functional

### Testing Requirements
- [ ] Unit tests for keyword research
- [ ] Unit tests for content strategy
- [ ] Integration tests for full workflow
- [ ] UI component tests
- [ ] Performance tests (keyword research < 60s)

### Documentation
- [ ] User guide written
- [ ] API documentation complete
- [ ] Code comments added
- [ ] Architecture documented

---

## Testing Checklist

### Keyword Research
- [ ] Generate seed keywords for test niche
- [ ] Expand keywords successfully
- [ ] Score keywords correctly
- [ ] Filter duplicate keywords
- [ ] Handle empty results gracefully

### Content Strategy
- [ ] Create topic clusters from keywords
- [ ] Distribute keywords across clusters
- [ ] Plan articles from clusters
- [ ] Generate publishing schedule
- [ ] Handle various niche types

### UI Functionality
- [ ] Create content plan from dashboard
- [ ] View keyword research results
- [ ] Display topic clusters
- [ ] Navigate between tabs
- [ ] Responsive on mobile devices

### API Endpoints
- [ ] POST /api/seo/keywords/research
- [ ] POST /api/seo/content-plan
- [ ] GET /api/seo/content-plan/{id}
- [ ] GET /api/seo/content-plans
- [ ] Error handling for invalid inputs

---

## Success Metrics

### Performance
- **Target:** Keyword research completes in < 60 seconds
- **Measurement:** API response time monitoring

### Quality
- **Target:** 80%+ of keywords have search intent classified
- **Measurement:** Keyword data completeness

### User Experience
- **Target:** Content plan creation < 2 minutes
- **Measurement:** User flow analytics

### Coverage
- **Target:** 5-10 topic clusters per plan
- **Measurement:** Cluster count per generated plan

---

## Next Sprint Preview

### Phase 4 - Sprint 2: Article Generation

**Focus Areas:**
1. **AI Article Writer**
   - LLM-powered article generation
   - SEO-optimized content creation
   - Multi-section composition

2. **SEO Optimizer**
   - On-page SEO analysis
   - Keyword density optimization
   - Meta tag generation

3. **Internal Linking**
   - Automated link suggestions
   - Product integration
   - Related content discovery

4. **Article Editor UI**
   - Rich text editor
   - SEO scoring
   - Preview functionality

**Prerequisites:**
- Sprint 1 (SEO Foundation) completed
- Content plans created
- Keywords researched
- Topic clusters defined
