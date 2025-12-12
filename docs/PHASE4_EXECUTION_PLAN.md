# Phase 4: "Expansion" - Production Execution Plan

## Overview

This document outlines the complete execution plan for building Zyenta's Phase 4: **Expansion**. This phase extends the platform's autonomous capabilities with SEO content generation and influencer marketing automation.

### Phase 4 Objectives
- [ ] Build SEO Blog Writing Agent
- [ ] Build Influencer Outreach Agent
- [ ] Launch "Full Autopilot" tier

### Prerequisites
- Phase 1-3 fully operational
- Growth Hub generating organic and paid traffic
- Established store analytics baseline
- Content generation pipeline proven

---

## 1. SEO Blog Writing Agent

### 1.1 Agent Architecture

```
services/growth-engine/app/agents/seo/
├── __init__.py
├── agent.py                    # SEO Agent orchestrator
├── keyword_researcher.py       # Keyword discovery and analysis
├── content_strategist.py       # Content planning and strategy
├── article_writer.py           # AI article generation
├── seo_optimizer.py            # On-page SEO optimization
├── internal_linker.py          # Internal linking automation
├── schema_generator.py         # Structured data generation
└── performance_tracker.py      # SEO performance monitoring
```

### 1.2 SEO Agent Orchestrator

```python
# services/growth-engine/app/agents/seo/agent.py
from typing import List, Optional, Dict, Any
from dataclasses import dataclass
from datetime import datetime, timedelta
from enum import Enum
from app.agents.base import BaseMarketingAgent
from app.agents.seo.keyword_researcher import KeywordResearcher
from app.agents.seo.content_strategist import ContentStrategist
from app.agents.seo.article_writer import ArticleWriter
from app.agents.seo.seo_optimizer import SEOOptimizer
from app.agents.seo.internal_linker import InternalLinker
from app.agents.seo.schema_generator import SchemaGenerator


class ArticleStatus(str, Enum):
    DRAFT = "draft"
    REVIEW = "review"
    SCHEDULED = "scheduled"
    PUBLISHED = "published"


@dataclass
class KeywordData:
    keyword: str
    search_volume: int
    difficulty: float  # 0-100
    cpc: float
    intent: str  # informational, commercial, transactional, navigational
    trend: str  # rising, stable, declining
    related_keywords: List[str]


@dataclass
class ContentPlan:
    store_id: str
    keywords: List[KeywordData]
    articles: List[Dict[str, Any]]
    clusters: List[Dict[str, Any]]  # Topic clusters
    publishing_schedule: List[Dict[str, Any]]


@dataclass
class Article:
    id: str
    store_id: str
    title: str
    slug: str
    meta_description: str
    content: str  # HTML content
    excerpt: str
    target_keyword: str
    secondary_keywords: List[str]
    word_count: int
    reading_time: int
    schema_markup: Dict[str, Any]
    internal_links: List[Dict[str, str]]
    status: ArticleStatus
    scheduled_date: Optional[datetime]
    published_date: Optional[datetime]


class SEOAgent(BaseMarketingAgent):
    """
    Autonomous SEO Content Agent.
    Handles keyword research, content strategy, article writing, and optimization.
    """

    def __init__(self):
        super().__init__()
        self.keyword_researcher = KeywordResearcher()
        self.content_strategist = ContentStrategist()
        self.article_writer = ArticleWriter()
        self.seo_optimizer = SEOOptimizer()
        self.internal_linker = InternalLinker()
        self.schema_generator = SchemaGenerator()

    async def create_content_strategy(
        self,
        store_id: str,
        months: int = 3,
        articles_per_month: int = 8,
    ) -> ContentPlan:
        """Create a comprehensive SEO content strategy"""

        # Get store data
        store = await self.get_store_data(store_id)
        products = await self.get_store_products(store_id)
        existing_content = await self._get_existing_content(store_id)

        # Research keywords
        keywords = await self.keyword_researcher.research(
            niche=store.niche,
            products=products,
            competitor_domains=await self._get_competitors(store.niche),
            existing_keywords=self._extract_existing_keywords(existing_content),
        )

        # Create topic clusters
        clusters = await self.content_strategist.create_clusters(
            keywords=keywords,
            niche=store.niche,
            products=products,
        )

        # Plan articles
        articles = await self.content_strategist.plan_articles(
            clusters=clusters,
            total_articles=months * articles_per_month,
            existing_content=existing_content,
        )

        # Create publishing schedule
        schedule = self._create_publishing_schedule(
            articles=articles,
            months=months,
            articles_per_month=articles_per_month,
        )

        content_plan = ContentPlan(
            store_id=store_id,
            keywords=keywords,
            articles=articles,
            clusters=clusters,
            publishing_schedule=schedule,
        )

        # Save plan
        await self._save_content_plan(content_plan)

        return content_plan

    async def generate_article(
        self,
        store_id: str,
        planned_article: Dict[str, Any],
    ) -> Article:
        """Generate a full SEO-optimized article"""

        store = await self.get_store_data(store_id)
        products = await self.get_store_products(store_id)
        existing_content = await self._get_existing_content(store_id)

        # Generate article content
        raw_content = await self.article_writer.write(
            topic=planned_article["topic"],
            target_keyword=planned_article["target_keyword"],
            secondary_keywords=planned_article["secondary_keywords"],
            outline=planned_article.get("outline"),
            tone=store.brandIdentity.get("tone", "professional"),
            products=self._get_relevant_products(
                products, planned_article["target_keyword"]
            ),
            word_count_target=planned_article.get("word_count", 1500),
        )

        # Optimize for SEO
        optimized_content = await self.seo_optimizer.optimize(
            content=raw_content,
            target_keyword=planned_article["target_keyword"],
            secondary_keywords=planned_article["secondary_keywords"],
        )

        # Add internal links
        linked_content = await self.internal_linker.add_links(
            content=optimized_content["content"],
            store_id=store_id,
            existing_content=existing_content,
            products=products,
            current_keyword=planned_article["target_keyword"],
        )

        # Generate schema markup
        schema = await self.schema_generator.generate(
            article_type="BlogPosting",
            title=optimized_content["title"],
            description=optimized_content["meta_description"],
            content=linked_content,
            author=store.name,
            publisher=store.name,
        )

        # Calculate metrics
        word_count = len(linked_content.split())
        reading_time = max(1, word_count // 200)  # ~200 words per minute

        article = Article(
            id=self.generate_id(),
            store_id=store_id,
            title=optimized_content["title"],
            slug=self._generate_slug(optimized_content["title"]),
            meta_description=optimized_content["meta_description"],
            content=linked_content,
            excerpt=optimized_content["excerpt"],
            target_keyword=planned_article["target_keyword"],
            secondary_keywords=planned_article["secondary_keywords"],
            word_count=word_count,
            reading_time=reading_time,
            schema_markup=schema,
            internal_links=optimized_content.get("internal_links", []),
            status=ArticleStatus.DRAFT,
            scheduled_date=planned_article.get("scheduled_date"),
            published_date=None,
        )

        # Save article
        await self._save_article(article)

        return article

    async def publish_article(self, article_id: str) -> Dict[str, Any]:
        """Publish an article to the store's blog"""

        article = await self._get_article(article_id)
        if not article:
            raise ValueError(f"Article not found: {article_id}")

        # Publish to storefront CMS
        result = await self._publish_to_cms(article)

        # Update article status
        article.status = ArticleStatus.PUBLISHED
        article.published_date = datetime.utcnow()
        await self._update_article(article)

        # Submit to search engines
        await self._submit_to_search_engines(
            url=result["url"],
            store_id=article.store_id,
        )

        return {
            "success": True,
            "url": result["url"],
            "published_at": article.published_date,
        }

    async def get_seo_analytics(
        self,
        store_id: str,
        days: int = 30,
    ) -> Dict[str, Any]:
        """Get SEO performance analytics"""

        articles = await self._get_published_articles(store_id)

        # Get search console data
        search_data = await self._get_search_console_data(store_id, days)

        # Get page analytics
        page_analytics = await self._get_page_analytics(store_id, articles, days)

        # Calculate aggregate metrics
        total_impressions = sum(p.get("impressions", 0) for p in search_data)
        total_clicks = sum(p.get("clicks", 0) for p in search_data)
        avg_position = (
            sum(p.get("position", 0) * p.get("impressions", 0) for p in search_data)
            / total_impressions
            if total_impressions > 0
            else 0
        )

        # Identify top performing content
        top_articles = sorted(
            page_analytics,
            key=lambda x: x.get("organic_traffic", 0),
            reverse=True,
        )[:10]

        # Identify keyword opportunities
        opportunities = await self._identify_opportunities(search_data)

        return {
            "overview": {
                "total_articles": len(articles),
                "total_impressions": total_impressions,
                "total_clicks": total_clicks,
                "avg_ctr": (total_clicks / total_impressions * 100) if total_impressions > 0 else 0,
                "avg_position": avg_position,
            },
            "articles": page_analytics,
            "top_performing": top_articles,
            "keyword_rankings": search_data,
            "opportunities": opportunities,
        }

    async def optimize_existing_content(
        self,
        article_id: str,
    ) -> Dict[str, Any]:
        """Optimize existing content based on performance data"""

        article = await self._get_article(article_id)
        if not article:
            raise ValueError(f"Article not found: {article_id}")

        # Get current performance
        performance = await self._get_article_performance(article_id)

        # Analyze content gaps
        gaps = await self.seo_optimizer.analyze_gaps(
            content=article.content,
            target_keyword=article.target_keyword,
            competitor_content=await self._get_competitor_content(article.target_keyword),
        )

        # Generate optimization recommendations
        recommendations = await self.seo_optimizer.get_recommendations(
            article=article,
            performance=performance,
            gaps=gaps,
        )

        # Optionally auto-apply optimizations
        if recommendations.get("auto_apply"):
            optimized = await self._apply_optimizations(article, recommendations)
            return {
                "optimized": True,
                "changes": optimized["changes"],
                "recommendations": recommendations,
            }

        return {
            "optimized": False,
            "recommendations": recommendations,
        }

    def _create_publishing_schedule(
        self,
        articles: List[Dict],
        months: int,
        articles_per_month: int,
    ) -> List[Dict[str, Any]]:
        """Create an optimal publishing schedule"""

        schedule = []
        current_date = datetime.utcnow()

        # Distribute articles evenly
        days_between = 30 // articles_per_month

        for i, article in enumerate(articles):
            publish_date = current_date + timedelta(days=i * days_between)

            # Prefer Tuesday/Thursday publishing
            while publish_date.weekday() not in [1, 3]:  # Tue, Thu
                publish_date += timedelta(days=1)

            schedule.append({
                "article_id": article.get("id"),
                "topic": article["topic"],
                "target_keyword": article["target_keyword"],
                "scheduled_date": publish_date,
                "priority": article.get("priority", "medium"),
            })

        return schedule

    async def _submit_to_search_engines(self, url: str, store_id: str):
        """Submit URL to search engines for indexing"""

        # Google Indexing API
        try:
            await self._submit_to_google(url, store_id)
        except Exception as e:
            self.logger.warning(f"Google indexing submission failed: {e}")

        # Bing Webmaster API
        try:
            await self._submit_to_bing(url, store_id)
        except Exception as e:
            self.logger.warning(f"Bing indexing submission failed: {e}")
```

### 1.3 Keyword Researcher

```python
# services/growth-engine/app/agents/seo/keyword_researcher.py
from typing import List, Dict, Any, Optional
from dataclasses import dataclass
import httpx
from app.services.llm import LLMService


@dataclass
class KeywordData:
    keyword: str
    search_volume: int
    difficulty: float
    cpc: float
    intent: str
    trend: str
    related_keywords: List[str]


class KeywordResearcher:
    """AI-powered keyword research for SEO"""

    def __init__(self):
        self.llm = LLMService()
        # Integration with keyword tools (e.g., Ahrefs, SEMrush APIs)
        self.ahrefs_api_key = None  # Configure if available
        self.semrush_api_key = None

    async def research(
        self,
        niche: str,
        products: List[Dict],
        competitor_domains: List[str],
        existing_keywords: List[str],
        target_count: int = 100,
    ) -> List[KeywordData]:
        """Conduct comprehensive keyword research"""

        all_keywords = []

        # 1. Generate seed keywords using AI
        seed_keywords = await self._generate_seed_keywords(niche, products)

        # 2. Expand keywords using tools/AI
        expanded = await self._expand_keywords(seed_keywords, niche)

        # 3. Get competitor keywords
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
        - Location-based keywords (if applicable)
        - Year-based keywords (2024, this year)

        Return as a JSON array of keywords.
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

        # If we have API access to SEO tools, use them
        if self.ahrefs_api_key:
            return await self._get_ahrefs_keywords(domains)
        elif self.semrush_api_key:
            return await self._get_semrush_keywords(domains)

        # Fallback to AI-based competitor analysis
        prompt = f"""
        Analyze these e-commerce competitors and suggest keywords they might rank for:
        {domains}

        Generate 30 likely keyword opportunities based on typical e-commerce SEO.
        Return as a JSON array of keywords.
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
        """Get search metrics for keywords"""

        # If we have API access, use real data
        if self.ahrefs_api_key or self.semrush_api_key:
            return await self._fetch_real_metrics(keywords)

        # Fallback to AI estimation
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

            Return as JSON array with objects for each keyword.
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

### 1.4 Article Writer

```python
# services/growth-engine/app/agents/seo/article_writer.py
from typing import List, Dict, Any, Optional
from app.services.llm import LLMService


class ArticleWriter:
    """AI-powered SEO article writer"""

    def __init__(self):
        self.llm = LLMService()

    async def write(
        self,
        topic: str,
        target_keyword: str,
        secondary_keywords: List[str],
        outline: Optional[List[Dict]] = None,
        tone: str = "professional",
        products: List[Dict] = None,
        word_count_target: int = 1500,
    ) -> Dict[str, Any]:
        """Write a complete SEO-optimized article"""

        # Generate outline if not provided
        if not outline:
            outline = await self._generate_outline(
                topic=topic,
                target_keyword=target_keyword,
                secondary_keywords=secondary_keywords,
                word_count_target=word_count_target,
            )

        # Write introduction
        intro = await self._write_introduction(
            topic=topic,
            target_keyword=target_keyword,
            tone=tone,
        )

        # Write each section
        sections = []
        for section in outline:
            section_content = await self._write_section(
                heading=section["heading"],
                key_points=section.get("key_points", []),
                target_keyword=target_keyword,
                secondary_keywords=secondary_keywords,
                tone=tone,
                products=products,
            )
            sections.append(section_content)

        # Write conclusion
        conclusion = await self._write_conclusion(
            topic=topic,
            target_keyword=target_keyword,
            tone=tone,
        )

        # Combine into full article
        full_content = self._assemble_article(intro, sections, conclusion, outline)

        # Generate meta content
        title = await self._generate_title(topic, target_keyword)
        meta_description = await self._generate_meta_description(
            topic, target_keyword, full_content[:500]
        )
        excerpt = await self._generate_excerpt(topic, full_content[:1000])

        return {
            "title": title,
            "meta_description": meta_description,
            "excerpt": excerpt,
            "content": full_content,
            "outline": outline,
        }

    async def _generate_outline(
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

    async def _write_introduction(
        self,
        topic: str,
        target_keyword: str,
        tone: str,
    ) -> str:
        """Write engaging introduction"""

        prompt = f"""
        Write an engaging introduction (150-200 words) for an article about: "{topic}"

        Requirements:
        1. Start with a hook that captures attention
        2. Identify the reader's problem or interest
        3. Include the keyword "{target_keyword}" naturally in the first paragraph
        4. Preview what the article will cover
        5. Use a {tone} tone

        Write in HTML format with <p> tags.
        """

        response = await self.llm.generate(prompt=prompt)
        return response

    async def _write_section(
        self,
        heading: str,
        key_points: List[str],
        target_keyword: str,
        secondary_keywords: List[str],
        tone: str,
        products: List[Dict] = None,
    ) -> str:
        """Write a content section"""

        product_context = ""
        if products:
            product_context = f"""
            If relevant, naturally mention these products:
            {[p['title'] for p in products[:3]]}
            """

        prompt = f"""
        Write a detailed section for the heading: "{heading}"

        Key points to cover:
        {key_points}

        Requirements:
        1. Write 200-300 words
        2. Include "{target_keyword}" once naturally
        3. Include 1-2 of these secondary keywords: {secondary_keywords[:5]}
        4. Use a {tone} tone
        5. Include specific examples or data where relevant
        6. Use bullet points or numbered lists where appropriate
        {product_context}

        Write in HTML format with appropriate tags (<p>, <ul>, <li>, etc.).
        Do NOT include the heading tag - just the content.
        """

        response = await self.llm.generate(prompt=prompt)
        return response

    async def _write_conclusion(
        self,
        topic: str,
        target_keyword: str,
        tone: str,
    ) -> str:
        """Write conclusion with call to action"""

        prompt = f"""
        Write a conclusion (100-150 words) for an article about: "{topic}"

        Requirements:
        1. Summarize key takeaways
        2. Include "{target_keyword}" one final time
        3. Add a clear call to action
        4. Use a {tone} tone
        5. End on an engaging note

        Write in HTML format with <p> tags.
        """

        response = await self.llm.generate(prompt=prompt)
        return response

    async def _generate_title(
        self,
        topic: str,
        target_keyword: str,
    ) -> str:
        """Generate SEO-optimized title"""

        prompt = f"""
        Generate an SEO-optimized title for an article about: "{topic}"

        Requirements:
        1. Include "{target_keyword}" near the beginning
        2. Keep under 60 characters
        3. Make it compelling and click-worthy
        4. Use power words where appropriate

        Return only the title text, nothing else.
        """

        response = await self.llm.generate(prompt=prompt)
        return response.strip()

    async def _generate_meta_description(
        self,
        topic: str,
        target_keyword: str,
        content_preview: str,
    ) -> str:
        """Generate meta description"""

        prompt = f"""
        Generate an SEO meta description for an article about: "{topic}"

        Content preview: {content_preview}

        Requirements:
        1. Include "{target_keyword}"
        2. Keep between 150-160 characters
        3. Include a call to action
        4. Make it compelling for search results

        Return only the meta description text.
        """

        response = await self.llm.generate(prompt=prompt)
        return response.strip()[:160]

    async def _generate_excerpt(
        self,
        topic: str,
        content_preview: str,
    ) -> str:
        """Generate article excerpt"""

        prompt = f"""
        Generate a 50-word excerpt summarizing this article about "{topic}":

        {content_preview}

        Make it engaging and informative.
        """

        response = await self.llm.generate(prompt=prompt)
        return response.strip()

    def _assemble_article(
        self,
        intro: str,
        sections: List[str],
        conclusion: str,
        outline: List[Dict],
    ) -> str:
        """Assemble full article HTML"""

        html = f"""
        <div class="article-content">
            <div class="introduction">
                {intro}
            </div>
        """

        for i, section in enumerate(sections):
            heading = outline[i]["heading"]
            level = outline[i].get("level", 2)

            html += f"""
            <section>
                <h{level}>{heading}</h{level}>
                {section}
            </section>
            """

        html += f"""
            <div class="conclusion">
                <h2>Conclusion</h2>
                {conclusion}
            </div>
        </div>
        """

        return html
```

---

## 2. Influencer Outreach Agent

### 2.1 Agent Architecture

```
services/growth-engine/app/agents/influencer/
├── __init__.py
├── agent.py                    # Influencer Agent orchestrator
├── discovery.py                # Influencer discovery
├── analyzer.py                 # Profile analysis
├── matcher.py                  # Brand-influencer matching
├── outreach.py                 # Outreach automation
├── campaign_manager.py         # Campaign management
├── performance_tracker.py      # ROI tracking
└── platforms/
    ├── __init__.py
    ├── instagram.py
    ├── tiktok.py
    ├── youtube.py
    └── twitter.py
```

### 2.2 Influencer Agent Orchestrator

```python
# services/growth-engine/app/agents/influencer/agent.py
from typing import List, Optional, Dict, Any
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from decimal import Decimal
from app.agents.base import BaseMarketingAgent
from app.agents.influencer.discovery import InfluencerDiscovery
from app.agents.influencer.analyzer import ProfileAnalyzer
from app.agents.influencer.matcher import BrandMatcher
from app.agents.influencer.outreach import OutreachManager
from app.agents.influencer.campaign_manager import CampaignManager
from app.agents.influencer.performance_tracker import PerformanceTracker


class InfluencerTier(str, Enum):
    NANO = "nano"           # 1K-10K followers
    MICRO = "micro"         # 10K-100K followers
    MID = "mid"             # 100K-500K followers
    MACRO = "macro"         # 500K-1M followers
    MEGA = "mega"           # 1M+ followers


class CampaignType(str, Enum):
    PRODUCT_REVIEW = "product_review"
    SPONSORED_POST = "sponsored_post"
    AFFILIATE = "affiliate"
    BRAND_AMBASSADOR = "brand_ambassador"
    GIVEAWAY = "giveaway"
    TAKEOVER = "takeover"


class OutreachStatus(str, Enum):
    PENDING = "pending"
    SENT = "sent"
    OPENED = "opened"
    REPLIED = "replied"
    INTERESTED = "interested"
    NEGOTIATING = "negotiating"
    ACCEPTED = "accepted"
    DECLINED = "declined"


@dataclass
class Influencer:
    id: str
    platform: str
    username: str
    display_name: str
    bio: str
    followers: int
    engagement_rate: float
    tier: InfluencerTier
    niche_categories: List[str]
    location: Optional[str]
    email: Optional[str]
    estimated_rate: Decimal
    profile_url: str
    avatar_url: str
    recent_posts: List[Dict[str, Any]]
    audience_demographics: Dict[str, Any]
    brand_affinity_score: float  # 0-100


@dataclass
class InfluencerCampaign:
    id: str
    store_id: str
    name: str
    type: CampaignType
    budget: Decimal
    spent: Decimal
    influencers: List[Dict[str, Any]]
    products: List[str]
    deliverables: List[Dict[str, Any]]
    start_date: datetime
    end_date: datetime
    performance: Dict[str, Any]


class InfluencerAgent(BaseMarketingAgent):
    """
    Autonomous Influencer Marketing Agent.
    Handles discovery, outreach, campaign management, and ROI tracking.
    """

    def __init__(self):
        super().__init__()
        self.discovery = InfluencerDiscovery()
        self.analyzer = ProfileAnalyzer()
        self.matcher = BrandMatcher()
        self.outreach = OutreachManager()
        self.campaign_manager = CampaignManager()
        self.performance_tracker = PerformanceTracker()

    async def discover_influencers(
        self,
        store_id: str,
        platform: str = "instagram",
        tier: InfluencerTier = InfluencerTier.MICRO,
        count: int = 50,
    ) -> List[Influencer]:
        """Discover relevant influencers for the store"""

        store = await self.get_store_data(store_id)
        products = await self.get_store_products(store_id, limit=10)

        # Generate search parameters
        search_params = await self._generate_search_params(
            niche=store.niche,
            products=products,
            tier=tier,
        )

        # Discover influencers on platform
        raw_profiles = await self.discovery.search(
            platform=platform,
            keywords=search_params["keywords"],
            hashtags=search_params["hashtags"],
            locations=search_params.get("locations"),
            follower_range=self._get_follower_range(tier),
            count=count * 2,  # Get more to filter
        )

        # Analyze profiles
        analyzed = []
        for profile in raw_profiles:
            analysis = await self.analyzer.analyze(
                platform=platform,
                username=profile["username"],
            )
            if analysis:
                analyzed.append({**profile, **analysis})

        # Score and match to brand
        matched = await self.matcher.match(
            influencers=analyzed,
            brand_niche=store.niche,
            brand_values=store.brandIdentity.get("values", []),
            target_audience=store.settings.get("target_audience", {}),
        )

        # Convert to Influencer objects
        influencers = [
            Influencer(
                id=self.generate_id(),
                platform=platform,
                username=m["username"],
                display_name=m.get("display_name", m["username"]),
                bio=m.get("bio", ""),
                followers=m["followers"],
                engagement_rate=m.get("engagement_rate", 0),
                tier=tier,
                niche_categories=m.get("categories", []),
                location=m.get("location"),
                email=m.get("email"),
                estimated_rate=self._estimate_rate(m["followers"], tier, platform),
                profile_url=m["profile_url"],
                avatar_url=m.get("avatar_url", ""),
                recent_posts=m.get("recent_posts", []),
                audience_demographics=m.get("demographics", {}),
                brand_affinity_score=m.get("brand_score", 0),
            )
            for m in matched[:count]
        ]

        # Save discovered influencers
        await self._save_influencers(store_id, influencers)

        return influencers

    async def create_campaign(
        self,
        store_id: str,
        name: str,
        campaign_type: CampaignType,
        budget: Decimal,
        influencer_ids: List[str],
        product_ids: List[str],
        duration_days: int = 30,
    ) -> InfluencerCampaign:
        """Create an influencer marketing campaign"""

        store = await self.get_store_data(store_id)
        influencers = await self._get_influencers(influencer_ids)
        products = await self._get_products(product_ids)

        # Generate campaign brief
        brief = await self._generate_campaign_brief(
            store=store,
            campaign_type=campaign_type,
            products=products,
            influencers=influencers,
        )

        # Create deliverables based on campaign type
        deliverables = self._create_deliverables(
            campaign_type=campaign_type,
            influencers=influencers,
            products=products,
        )

        campaign = InfluencerCampaign(
            id=self.generate_id(),
            store_id=store_id,
            name=name,
            type=campaign_type,
            budget=budget,
            spent=Decimal("0"),
            influencers=[
                {
                    "id": inf.id,
                    "username": inf.username,
                    "platform": inf.platform,
                    "status": "pending",
                    "agreed_rate": None,
                }
                for inf in influencers
            ],
            products=product_ids,
            deliverables=deliverables,
            start_date=datetime.utcnow(),
            end_date=datetime.utcnow() + timedelta(days=duration_days),
            performance={},
        )

        # Save campaign
        await self._save_campaign(campaign)

        return campaign

    async def send_outreach(
        self,
        campaign_id: str,
        influencer_id: str,
        personalized: bool = True,
    ) -> Dict[str, Any]:
        """Send outreach to an influencer"""

        campaign = await self._get_campaign(campaign_id)
        influencer = await self._get_influencer(influencer_id)
        store = await self.get_store_data(campaign.store_id)
        products = await self._get_products(campaign.products)

        # Generate personalized outreach message
        if personalized:
            message = await self.outreach.generate_message(
                influencer=influencer,
                store=store,
                campaign_type=campaign.type,
                products=products,
            )
        else:
            message = await self._get_template_message(campaign.type)

        # Send outreach
        result = await self.outreach.send(
            influencer=influencer,
            subject=f"Collaboration Opportunity with {store.name}",
            message=message,
            attachments=await self._prepare_attachments(store, products),
        )

        # Update campaign status
        await self._update_influencer_status(
            campaign_id=campaign_id,
            influencer_id=influencer_id,
            status=OutreachStatus.SENT,
            outreach_id=result["id"],
        )

        return result

    async def track_campaign_performance(
        self,
        campaign_id: str,
    ) -> Dict[str, Any]:
        """Track campaign performance and ROI"""

        campaign = await self._get_campaign(campaign_id)

        # Get content from influencers
        content = await self.performance_tracker.get_campaign_content(
            campaign_id=campaign_id,
            influencers=campaign.influencers,
        )

        # Calculate metrics
        total_reach = sum(c.get("reach", 0) for c in content)
        total_engagement = sum(c.get("engagement", 0) for c in content)
        total_clicks = sum(c.get("clicks", 0) for c in content)
        total_conversions = sum(c.get("conversions", 0) for c in content)

        # Calculate revenue attribution
        revenue = await self.performance_tracker.calculate_revenue(
            campaign_id=campaign_id,
            tracking_links=campaign.deliverables,
        )

        # Calculate ROI
        roi = (
            (revenue - float(campaign.spent)) / float(campaign.spent) * 100
            if campaign.spent > 0
            else 0
        )

        performance = {
            "reach": total_reach,
            "engagement": total_engagement,
            "engagement_rate": (
                total_engagement / total_reach * 100 if total_reach > 0 else 0
            ),
            "clicks": total_clicks,
            "conversions": total_conversions,
            "conversion_rate": (
                total_conversions / total_clicks * 100 if total_clicks > 0 else 0
            ),
            "revenue": revenue,
            "spent": float(campaign.spent),
            "roi": roi,
            "cost_per_engagement": (
                float(campaign.spent) / total_engagement if total_engagement > 0 else 0
            ),
            "cost_per_conversion": (
                float(campaign.spent) / total_conversions
                if total_conversions > 0
                else 0
            ),
            "content": content,
        }

        # Update campaign performance
        await self._update_campaign_performance(campaign_id, performance)

        return performance

    async def get_recommendations(
        self,
        store_id: str,
    ) -> Dict[str, Any]:
        """Get AI recommendations for influencer marketing"""

        store = await self.get_store_data(store_id)
        past_campaigns = await self._get_store_campaigns(store_id)
        discovered_influencers = await self._get_discovered_influencers(store_id)

        prompt = f"""
        Analyze influencer marketing for this {store.niche} e-commerce store:

        Past campaigns: {len(past_campaigns)}
        Average ROI: {self._calculate_avg_roi(past_campaigns)}%
        Discovered influencers: {len(discovered_influencers)}

        Top performing influencers: {self._get_top_influencers(past_campaigns)}

        Provide recommendations for:
        1. Optimal influencer tier to target
        2. Best campaign types for this niche
        3. Suggested budget allocation
        4. Content format recommendations
        5. Timing and frequency suggestions

        Return as JSON.
        """

        response = await self.llm.generate(
            prompt=prompt,
            response_format={"type": "json_object"},
        )

        return response

    def _get_follower_range(self, tier: InfluencerTier) -> tuple:
        """Get follower range for tier"""
        ranges = {
            InfluencerTier.NANO: (1000, 10000),
            InfluencerTier.MICRO: (10000, 100000),
            InfluencerTier.MID: (100000, 500000),
            InfluencerTier.MACRO: (500000, 1000000),
            InfluencerTier.MEGA: (1000000, float("inf")),
        }
        return ranges.get(tier, (10000, 100000))

    def _estimate_rate(
        self,
        followers: int,
        tier: InfluencerTier,
        platform: str,
    ) -> Decimal:
        """Estimate influencer rate based on metrics"""

        # Base rates by tier (per post)
        base_rates = {
            InfluencerTier.NANO: 50,
            InfluencerTier.MICRO: 250,
            InfluencerTier.MID: 1000,
            InfluencerTier.MACRO: 5000,
            InfluencerTier.MEGA: 20000,
        }

        # Platform multipliers
        platform_multipliers = {
            "instagram": 1.0,
            "tiktok": 0.8,
            "youtube": 2.0,
            "twitter": 0.6,
        }

        base = base_rates.get(tier, 250)
        multiplier = platform_multipliers.get(platform, 1.0)

        return Decimal(str(base * multiplier))

    def _create_deliverables(
        self,
        campaign_type: CampaignType,
        influencers: List[Influencer],
        products: List[Dict],
    ) -> List[Dict[str, Any]]:
        """Create deliverables for campaign"""

        deliverables = []

        for influencer in influencers:
            if campaign_type == CampaignType.PRODUCT_REVIEW:
                deliverables.append({
                    "influencer_id": influencer.id,
                    "type": "video_review",
                    "description": "In-depth product review video",
                    "platform": influencer.platform,
                    "status": "pending",
                })
            elif campaign_type == CampaignType.SPONSORED_POST:
                deliverables.append({
                    "influencer_id": influencer.id,
                    "type": "feed_post",
                    "description": "Sponsored feed post featuring product",
                    "platform": influencer.platform,
                    "status": "pending",
                })
                deliverables.append({
                    "influencer_id": influencer.id,
                    "type": "story",
                    "description": "3 story slides with swipe-up link",
                    "platform": influencer.platform,
                    "status": "pending",
                })
            elif campaign_type == CampaignType.AFFILIATE:
                deliverables.append({
                    "influencer_id": influencer.id,
                    "type": "affiliate_content",
                    "description": "Content with unique affiliate link",
                    "platform": influencer.platform,
                    "tracking_link": self._generate_affiliate_link(influencer),
                    "status": "pending",
                })
            elif campaign_type == CampaignType.GIVEAWAY:
                deliverables.append({
                    "influencer_id": influencer.id,
                    "type": "giveaway_post",
                    "description": "Giveaway announcement post",
                    "platform": influencer.platform,
                    "status": "pending",
                })

        return deliverables
```

### 2.3 Influencer Discovery

```python
# services/growth-engine/app/agents/influencer/discovery.py
from typing import List, Dict, Any, Optional
import httpx
from app.services.llm import LLMService


class InfluencerDiscovery:
    """Discover influencers across platforms"""

    def __init__(self):
        self.llm = LLMService()
        # Platform API clients would be initialized here
        self.instagram_client = None
        self.tiktok_client = None

    async def search(
        self,
        platform: str,
        keywords: List[str],
        hashtags: List[str],
        locations: Optional[List[str]] = None,
        follower_range: tuple = (10000, 100000),
        count: int = 100,
    ) -> List[Dict[str, Any]]:
        """Search for influencers matching criteria"""

        if platform == "instagram":
            return await self._search_instagram(
                keywords, hashtags, locations, follower_range, count
            )
        elif platform == "tiktok":
            return await self._search_tiktok(
                keywords, hashtags, locations, follower_range, count
            )
        elif platform == "youtube":
            return await self._search_youtube(
                keywords, hashtags, locations, follower_range, count
            )

        return []

    async def _search_instagram(
        self,
        keywords: List[str],
        hashtags: List[str],
        locations: Optional[List[str]],
        follower_range: tuple,
        count: int,
    ) -> List[Dict[str, Any]]:
        """Search Instagram for influencers"""

        profiles = []

        # Search by hashtags
        for hashtag in hashtags[:10]:
            hashtag_profiles = await self._get_hashtag_top_posts(hashtag)
            profiles.extend(hashtag_profiles)

        # Search by keywords in bio
        for keyword in keywords[:5]:
            keyword_profiles = await self._search_by_bio(keyword)
            profiles.extend(keyword_profiles)

        # Filter by follower count
        filtered = [
            p
            for p in profiles
            if follower_range[0] <= p.get("followers", 0) <= follower_range[1]
        ]

        # Deduplicate
        seen = set()
        unique = []
        for p in filtered:
            if p["username"] not in seen:
                seen.add(p["username"])
                unique.append(p)

        return unique[:count]

    async def _search_tiktok(
        self,
        keywords: List[str],
        hashtags: List[str],
        locations: Optional[List[str]],
        follower_range: tuple,
        count: int,
    ) -> List[Dict[str, Any]]:
        """Search TikTok for influencers"""

        profiles = []

        # Similar logic for TikTok
        # Would use TikTok Creator Marketplace API or scraping

        return profiles[:count]

    async def _search_youtube(
        self,
        keywords: List[str],
        hashtags: List[str],
        locations: Optional[List[str]],
        follower_range: tuple,
        count: int,
    ) -> List[Dict[str, Any]]:
        """Search YouTube for influencers"""

        # Use YouTube Data API
        profiles = []

        async with httpx.AsyncClient() as client:
            for keyword in keywords[:5]:
                response = await client.get(
                    "https://www.googleapis.com/youtube/v3/search",
                    params={
                        "key": self.youtube_api_key,
                        "q": keyword,
                        "type": "channel",
                        "maxResults": 20,
                        "part": "snippet",
                    },
                )

                if response.status_code == 200:
                    data = response.json()
                    for item in data.get("items", []):
                        channel_id = item["id"]["channelId"]
                        channel_info = await self._get_youtube_channel(channel_id)
                        if channel_info:
                            profiles.append(channel_info)

        # Filter by subscriber count
        filtered = [
            p
            for p in profiles
            if follower_range[0] <= p.get("followers", 0) <= follower_range[1]
        ]

        return filtered[:count]

    async def _get_hashtag_top_posts(self, hashtag: str) -> List[Dict[str, Any]]:
        """Get top posts and creators from a hashtag"""

        # Would use Instagram Graph API or third-party service
        # Placeholder implementation
        return []

    async def _search_by_bio(self, keyword: str) -> List[Dict[str, Any]]:
        """Search profiles by bio keyword"""

        # Would use Instagram search or third-party service
        return []

    async def _get_youtube_channel(self, channel_id: str) -> Optional[Dict[str, Any]]:
        """Get YouTube channel details"""

        async with httpx.AsyncClient() as client:
            response = await client.get(
                "https://www.googleapis.com/youtube/v3/channels",
                params={
                    "key": self.youtube_api_key,
                    "id": channel_id,
                    "part": "snippet,statistics",
                },
            )

            if response.status_code == 200:
                data = response.json()
                if data.get("items"):
                    item = data["items"][0]
                    return {
                        "username": item["snippet"]["customUrl"],
                        "display_name": item["snippet"]["title"],
                        "bio": item["snippet"]["description"],
                        "followers": int(item["statistics"].get("subscriberCount", 0)),
                        "avatar_url": item["snippet"]["thumbnails"]["default"]["url"],
                        "profile_url": f"https://youtube.com/channel/{channel_id}",
                    }

        return None
```

### 2.4 Outreach Manager

```python
# services/growth-engine/app/agents/influencer/outreach.py
from typing import Dict, Any, List, Optional
from app.services.llm import LLMService
from app.services.email import EmailService


class OutreachManager:
    """Manage influencer outreach"""

    def __init__(self):
        self.llm = LLMService()
        self.email_service = EmailService()

    async def generate_message(
        self,
        influencer: Any,
        store: Any,
        campaign_type: str,
        products: List[Dict],
    ) -> str:
        """Generate personalized outreach message"""

        prompt = f"""
        Write a personalized influencer outreach email.

        Influencer details:
        - Name: {influencer.display_name}
        - Username: @{influencer.username}
        - Platform: {influencer.platform}
        - Bio: {influencer.bio}
        - Niche: {', '.join(influencer.niche_categories)}
        - Followers: {influencer.followers:,}

        Brand details:
        - Name: {store.name}
        - Niche: {store.niche}
        - Description: {store.description}

        Products to potentially feature:
        {[p['title'] for p in products[:3]]}

        Campaign type: {campaign_type}

        Requirements:
        1. Start with a genuine compliment about their content
        2. Reference something specific from their profile
        3. Explain why they're a good fit for the brand
        4. Clearly state the collaboration opportunity
        5. Include next steps
        6. Keep professional but friendly tone
        7. Keep under 200 words

        Write the email body only (no subject line).
        """

        response = await self.llm.generate(prompt=prompt)
        return response

    async def send(
        self,
        influencer: Any,
        subject: str,
        message: str,
        attachments: List[Dict] = None,
    ) -> Dict[str, Any]:
        """Send outreach email"""

        if not influencer.email:
            # Try DM approach if no email
            return await self._send_dm(influencer, message)

        result = await self.email_service.send(
            to=influencer.email,
            subject=subject,
            body=message,
            attachments=attachments,
            track_opens=True,
            track_clicks=True,
        )

        return {
            "id": result["message_id"],
            "method": "email",
            "sent_at": result["sent_at"],
        }

    async def _send_dm(
        self,
        influencer: Any,
        message: str,
    ) -> Dict[str, Any]:
        """Send direct message on platform"""

        # Would integrate with platform APIs for DMs
        # Currently, return placeholder for manual action

        return {
            "id": None,
            "method": "dm_required",
            "platform": influencer.platform,
            "username": influencer.username,
            "message": message,
            "action_required": "manual_dm",
        }

    async def generate_followup(
        self,
        influencer: Any,
        original_message: str,
        days_since_sent: int,
    ) -> str:
        """Generate follow-up message"""

        prompt = f"""
        Write a follow-up email to an influencer who hasn't responded.

        Original outreach was {days_since_sent} days ago.

        Influencer: @{influencer.username} on {influencer.platform}

        Original message summary:
        {original_message[:200]}

        Requirements:
        1. Be polite and not pushy
        2. Add new value or information
        3. Include a soft deadline or urgency
        4. Keep under 100 words

        Write the follow-up body only.
        """

        response = await self.llm.generate(prompt=prompt)
        return response

    async def handle_reply(
        self,
        influencer_id: str,
        reply_content: str,
    ) -> Dict[str, Any]:
        """Process and categorize reply"""

        prompt = f"""
        Analyze this influencer reply and determine the next action:

        Reply: {reply_content}

        Categorize as one of:
        - interested: They want to collaborate
        - questions: They have questions before deciding
        - negotiating: They're negotiating terms
        - declined: They declined the opportunity
        - unclear: Need more clarification

        Also extract:
        - Requested rate (if mentioned)
        - Specific questions
        - Availability/timeline
        - Concerns

        Return as JSON.
        """

        response = await self.llm.generate(
            prompt=prompt,
            response_format={"type": "json_object"},
        )

        # Generate suggested response
        if response.get("category") == "questions":
            response["suggested_reply"] = await self._generate_answer_response(
                response.get("questions", [])
            )
        elif response.get("category") == "negotiating":
            response["suggested_reply"] = await self._generate_negotiation_response(
                response.get("requested_rate")
            )

        return response
```

---

## 3. Database Schema Extensions

### 3.1 New Prisma Models

```prisma
// packages/database/prisma/schema.prisma (additions)

// ==================== SEO CONTENT ====================

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
  wordCount       Int
  readingTime     Int

  schemaMarkup    Json?
  internalLinks   Json?     // Array of {text, url}

  status          ArticleStatus @default(DRAFT)
  scheduledDate   DateTime?
  publishedDate   DateTime?
  publishedUrl    String?

  // Performance
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

// ==================== INFLUENCER MARKETING ====================

model Influencer {
  id              String    @id @default(cuid())
  storeId         String    // Discovery context
  store           Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)

  platform        String
  username        String
  displayName     String
  bio             String?   @db.Text
  email           String?

  followers       Int
  engagementRate  Float
  tier            InfluencerTier

  nicheCategories String[]
  location        String?

  estimatedRate   Decimal   @db.Decimal(10, 2)
  profileUrl      String
  avatarUrl       String?

  recentPosts     Json?     // Array of post data
  demographics    Json?     // Audience demographics
  brandScore      Float     @default(0)

  // Outreach tracking
  outreaches      InfluencerOutreach[]
  campaigns       InfluencerCampaignParticipation[]

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@unique([platform, username])
  @@index([storeId])
  @@index([tier])
}

enum InfluencerTier {
  NANO
  MICRO
  MID
  MACRO
  MEGA
}

model InfluencerCampaign {
  id              String    @id @default(cuid())
  storeId         String
  store           Store     @relation(fields: [storeId], references: [id], onDelete: Cascade)

  name            String
  type            CampaignType
  status          InfluencerCampaignStatus @default(DRAFT)

  budget          Decimal   @db.Decimal(10, 2)
  spent           Decimal   @db.Decimal(10, 2) @default(0)

  startDate       DateTime
  endDate         DateTime

  productIds      String[]
  deliverables    Json      // Array of deliverable specs
  brief           String?   @db.Text

  // Performance
  reach           Int       @default(0)
  engagement      Int       @default(0)
  clicks          Int       @default(0)
  conversions     Int       @default(0)
  revenue         Decimal   @db.Decimal(10, 2) @default(0)

  participants    InfluencerCampaignParticipation[]

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([storeId])
}

enum CampaignType {
  PRODUCT_REVIEW
  SPONSORED_POST
  AFFILIATE
  BRAND_AMBASSADOR
  GIVEAWAY
  TAKEOVER
}

enum InfluencerCampaignStatus {
  DRAFT
  RECRUITING
  ACTIVE
  COMPLETED
  CANCELLED
}

model InfluencerCampaignParticipation {
  id              String    @id @default(cuid())
  campaignId      String
  campaign        InfluencerCampaign @relation(fields: [campaignId], references: [id], onDelete: Cascade)
  influencerId    String
  influencer      Influencer @relation(fields: [influencerId], references: [id])

  status          ParticipationStatus @default(PENDING)
  agreedRate      Decimal?  @db.Decimal(10, 2)
  paymentStatus   String?

  deliverables    Json?     // Status of each deliverable
  contentLinks    String[]  // Links to created content

  // Performance
  reach           Int       @default(0)
  engagement      Int       @default(0)
  clicks          Int       @default(0)
  conversions     Int       @default(0)

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@unique([campaignId, influencerId])
}

enum ParticipationStatus {
  PENDING
  CONTACTED
  NEGOTIATING
  ACCEPTED
  DECLINED
  ACTIVE
  COMPLETED
}

model InfluencerOutreach {
  id              String    @id @default(cuid())
  influencerId    String
  influencer      Influencer @relation(fields: [influencerId], references: [id], onDelete: Cascade)
  campaignId      String?

  method          String    // email, dm
  subject         String?
  message         String    @db.Text

  status          OutreachStatus @default(SENT)
  sentAt          DateTime  @default(now())
  openedAt        DateTime?
  repliedAt       DateTime?
  replyContent    String?   @db.Text

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([influencerId])
  @@index([status])
}

enum OutreachStatus {
  PENDING
  SENT
  OPENED
  REPLIED
  INTERESTED
  NEGOTIATING
  ACCEPTED
  DECLINED
}
```

---

## 4. Implementation Order

### Sprint 1: SEO Foundation (Weeks 1-2)
1. Implement keyword researcher
2. Build content strategist
3. Create topic clustering
4. Setup content plan management
5. Build dashboard UI for SEO planning

### Sprint 2: Article Generation (Weeks 3-4)
1. Implement article writer with AI
2. Build SEO optimizer
3. Create internal linking system
4. Implement schema generator
5. Build article editor UI

### Sprint 3: SEO Publishing & Analytics (Weeks 5-6)
1. Implement CMS integration
2. Build search engine submission
3. Create SEO analytics dashboard
4. Implement content optimization suggestions
5. Build performance tracking

### Sprint 4: Influencer Discovery (Weeks 7-8)
1. Implement multi-platform discovery
2. Build profile analyzer
3. Create brand matcher
4. Build influencer database UI
5. Implement tier categorization

### Sprint 5: Influencer Outreach (Weeks 9-10)
1. Implement outreach manager
2. Build message personalization
3. Create campaign management
4. Implement reply handling
5. Build negotiation workflows

### Sprint 6: Integration & Analytics (Weeks 11-12)
1. Build influencer campaign tracking
2. Implement ROI calculation
3. Create unified marketing dashboard
4. Performance optimization
5. Documentation and launch

---

## 5. Success Metrics

### SEO Metrics
- [ ] Content plan generated in < 5 minutes
- [ ] Article generation time < 10 minutes
- [ ] SEO score > 80 for generated content
- [ ] Organic traffic increase > 50% in 90 days
- [ ] Keyword rankings improvement for 60% of targets

### Influencer Metrics
- [ ] Discovery time < 2 minutes per platform
- [ ] Outreach response rate > 15%
- [ ] Campaign ROI > 3x
- [ ] Influencer match accuracy > 70%
- [ ] Content approval rate > 80%

---

## 6. Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| Content quality inconsistency | High | Human review option, quality scoring |
| SEO penalties for AI content | High | Unique value-add, E-E-A-T guidelines |
| Influencer fraud | Medium | Engagement authenticity checking |
| Platform API limitations | Medium | Multi-source data, fallback strategies |
| Email deliverability | Medium | Warm-up domains, personalization |
| Low outreach response | Medium | A/B test messages, optimal timing |

---

*Phase 4 completes the autonomous marketing ecosystem with SEO content automation and influencer marketing, enabling true "full autopilot" e-commerce operations.*
