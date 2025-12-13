# Phase 3 - Sprint 2: Social Agent Core - Action Plan

## Sprint Overview

**Duration:** 2 weeks
**Team Size:** 2-3 developers
**Sprint Goal:** Build the core Social Agent that autonomously creates, schedules, and publishes content to TikTok and Instagram using AI-powered content planning and caption generation.

### Objectives
- [ ] Implement AI-powered content planner
- [ ] Build LLM-based caption writer
- [ ] Create hashtag optimizer
- [ ] Implement intelligent posting scheduler
- [ ] Build TikTok platform client for content publishing
- [ ] Build Instagram platform client for content publishing
- [ ] Create content calendar generation system

### Prerequisites
- Phase 3 Sprint 1 (OAuth Foundation) complete
- OAuth connections working for Meta and TikTok
- Token management system operational
- Media Studio has images and videos available
- LLM service (OpenAI/Anthropic) configured

---

## Day-by-Day Breakdown

### Days 1-2: Social Agent Architecture & Content Planner

#### Tasks
1. **Create Social Agent Module Structure**
   ```
   services/growth-engine/app/agents/
   ├── __init__.py
   ├── base.py
   ├── social/
   │   ├── __init__.py
   │   ├── agent.py              # Main Social Agent orchestrator
   │   ├── content_planner.py    # AI content planning
   │   ├── caption_writer.py     # AI caption generation
   │   ├── hashtag_optimizer.py  # Hashtag research & optimization
   │   ├── scheduler.py          # Posting schedule optimization
   │   └── platforms/
   │       ├── __init__.py
   │       ├── base.py
   │       ├── tiktok.py
   │       ├── instagram.py
   │       ├── pinterest.py
   │       └── twitter.py
   ```

2. **Create Base Marketing Agent**
   ```python
   # app/agents/base.py
   from abc import ABC, abstractmethod
   from typing import Any, Dict, List
   import secrets

   class BaseMarketingAgent(ABC):
       """Base class for all marketing agents"""

       def __init__(self):
           self.agent_id = secrets.token_urlsafe(8)

       async def get_store_data(self, store_id: str) -> Any:
           """Fetch store information"""
           from app.services.database import prisma

           store = await prisma.store.find_unique(
               where={"id": store_id},
               include={
                   "brandIdentity": True,
                   "settings": True,
               },
           )
           return store

       async def get_store_products(
           self,
           store_id: str,
           limit: int = 20
       ) -> List[Dict]:
           """Fetch store products"""
           from app.services.database import prisma

           products = await prisma.product.find_many(
               where={"storeId": store_id, "status": "ACTIVE"},
               take=limit,
               order_by={"createdAt": "desc"},
           )
           return products

       async def get_store_media(self, store_id: str) -> List[Dict]:
           """Fetch store media assets"""
           from app.services.database import prisma

           media = await prisma.mediaasset.find_many(
               where={"storeId": store_id},
               order_by={"createdAt": "desc"},
           )
           return media

       async def get_connected_platforms(self, store_id: str) -> List[str]:
           """Get list of connected platforms for a store"""
           from app.services.database import prisma

           connections = await prisma.platformconnection.find_many(
               where={
                   "storeId": store_id,
                   "status": "CONNECTED",
               }
           )
           return [conn.platform for conn in connections]

       def generate_id(self) -> str:
           """Generate a unique ID"""
           return secrets.token_urlsafe(12)
   ```

3. **Setup LLM Service**
   ```python
   # app/services/llm.py
   from typing import Dict, Any, Optional
   from openai import AsyncOpenAI
   from anthropic import AsyncAnthropic
   from app.config import settings

   class LLMService:
       """Service for LLM interactions"""

       def __init__(self, provider: str = "openai"):
           self.provider = provider

           if provider == "openai":
               self.client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)
               self.model = "gpt-4-turbo-preview"
           elif provider == "anthropic":
               self.client = AsyncAnthropic(api_key=settings.ANTHROPIC_API_KEY)
               self.model = "claude-3-5-sonnet-20241022"

       async def generate(
           self,
           prompt: str,
           system_prompt: Optional[str] = None,
           response_format: Optional[Dict] = None,
           max_tokens: int = 2000,
           temperature: float = 0.7,
       ) -> Dict[str, Any]:
           """Generate content using LLM"""

           if self.provider == "openai":
               messages = []
               if system_prompt:
                   messages.append({"role": "system", "content": system_prompt})
               messages.append({"role": "user", "content": prompt})

               kwargs = {
                   "model": self.model,
                   "messages": messages,
                   "max_tokens": max_tokens,
                   "temperature": temperature,
               }

               if response_format:
                   kwargs["response_format"] = response_format

               response = await self.client.chat.completions.create(**kwargs)

               content = response.choices[0].message.content

               # Parse JSON if requested
               if response_format and response_format.get("type") == "json_object":
                   import json
                   return json.loads(content)

               return {"content": content}

           elif self.provider == "anthropic":
               kwargs = {
                   "model": self.model,
                   "max_tokens": max_tokens,
                   "temperature": temperature,
                   "messages": [{"role": "user", "content": prompt}],
               }

               if system_prompt:
                   kwargs["system"] = system_prompt

               response = await self.client.messages.create(**kwargs)

               content = response.content[0].text

               # Parse JSON if requested
               if response_format and response_format.get("type") == "json_object":
                   import json
                   return json.loads(content)

               return {"content": content}

       async def generate_structured(
           self,
           prompt: str,
           schema: Dict[str, Any],
           system_prompt: Optional[str] = None,
       ) -> Dict[str, Any]:
           """Generate structured JSON output"""

           full_prompt = f"{prompt}\n\nRespond with valid JSON matching this schema:\n{schema}"

           return await self.generate(
               prompt=full_prompt,
               system_prompt=system_prompt,
               response_format={"type": "json_object"},
           )
   ```

4. **Implement Content Planner**
   ```python
   # app/agents/social/content_planner.py
   from typing import List, Dict, Any
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
           products: List[Dict],
           days: int = 30,
           posts_per_day: int = 2,
       ) -> List[Dict[str, Any]]:
           """Create a comprehensive content plan"""

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

                   # Alternate between product and brand content
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
           products: List[Dict],
       ) -> List[Dict[str, Any]]:
           """Generate content themes using AI"""

           product_titles = [p.get('title', '') for p in products[:5]]

           prompt = f"""
           Generate 10 engaging content themes for a {niche} brand with a {brand_voice} voice.

           The brand sells products like:
           {', '.join(product_titles)}

           Create themes that:
           1. Showcase products naturally
           2. Provide value to the audience
           3. Build brand awareness
           4. Encourage engagement
           5. Mix promotional and educational content

           Return a JSON object with a "themes" array. Each theme should have:
           - name: Theme name
           - description: Brief description
           - content_angle: How to approach the content
           - best_for: Type of content (image, video, carousel)

           Example:
           {{
             "themes": [
               {{
                 "name": "Morning Routine Inspiration",
                 "description": "Showcase how products fit into morning routines",
                 "content_angle": "Aspirational lifestyle content",
                 "best_for": "video"
               }}
             ]
           }}
           """

           response = await self.llm.generate(
               prompt=prompt,
               response_format={"type": "json_object"},
           )

           return response.get("themes", [])

       async def _plan_product_post(
           self,
           product: Dict,
           theme: Dict,
           brand_voice: str,
       ) -> Dict[str, Any]:
           """Plan a product-focused post"""

           prompt = f"""
           Create a social media post plan for this product:

           Product: {product.get('title', '')}
           Description: {product.get('description', '')[:200]}
           Price: ${product.get('sellingPrice', 0)}

           Theme: {theme['name']}
           Brand Voice: {brand_voice}

           Return a JSON object with:
           - headline: Attention-grabbing headline
           - key_points: Array of 3 key points to highlight
           - cta: Call to action
           - hook: Opening hook for the caption
           - emotion: Primary emotion to evoke (e.g., "excitement", "curiosity")

           Make it engaging and authentic.
           """

           response = await self.llm.generate(
               prompt=prompt,
               response_format={"type": "json_object"},
           )

           return {
               "type": "product",
               "product_id": product.get("id"),
               "product": product,
               "theme": theme,
               **response,
           }

       async def _plan_brand_post(
           self,
           niche: str,
           theme: Dict,
           brand_voice: str,
       ) -> Dict[str, Any]:
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

           Make it valuable and engaging.
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
           day_of_week = day % 7

           if day_of_week in [0, 3]:  # Sunday, Wednesday
               return "video"
           elif day_of_week in [2, 5]:  # Tuesday, Friday
               return "carousel"
           else:
               return "image"
   ```

5. **Update Configuration**
   ```python
   # app/config.py (add)
   class Settings(BaseSettings):
       # ... existing settings ...

       # LLM Configuration
       OPENAI_API_KEY: str
       ANTHROPIC_API_KEY: Optional[str] = None
       LLM_PROVIDER: str = "openai"  # or "anthropic"
   ```

#### Deliverables
- [ ] Base marketing agent class created
- [ ] LLM service implemented
- [ ] Content planner generating themed content plans
- [ ] Content plan includes product and brand posts
- [ ] AI-generated themes working

---

### Days 3-4: Caption Writer & Hashtag Optimizer

#### Tasks
1. **Implement Caption Writer**
   ```python
   # app/agents/social/caption_writer.py
   from typing import Optional, Dict, Any
   from app.services.llm import LLMService

   class CaptionWriter:
       """AI-powered caption generation for social media"""

       def __init__(self):
           self.llm = LLMService()

       # Platform-specific character limits
       PLATFORM_LIMITS = {
           "tiktok": 2200,
           "instagram": 2200,
           "twitter": 280,
           "pinterest": 500,
       }

       # Platform-specific best practices
       PLATFORM_STYLES = {
           "tiktok": "casual, trendy, use emojis, hook in first 3 words",
           "instagram": "engaging, use line breaks, emojis, ask questions",
           "twitter": "concise, punchy, use threads if needed",
           "pinterest": "descriptive, keyword-rich, actionable",
       }

       async def write(
           self,
           content_type: str,
           product: Optional[Dict] = None,
           brand_voice: str = "friendly",
           platform: str = "instagram",
           call_to_action: Optional[str] = None,
           planned_content: Optional[Dict] = None,
       ) -> str:
           """Generate platform-optimized caption"""

           # Build context
           context = self._build_context(
               content_type=content_type,
               product=product,
               planned_content=planned_content,
           )

           # Get platform constraints
           max_length = self.PLATFORM_LIMITS.get(platform, 2200)
           platform_style = self.PLATFORM_STYLES.get(platform, "engaging")

           prompt = f"""
           Write a compelling social media caption for {platform}.

           Context:
           {context}

           Brand Voice: {brand_voice}
           Platform Style: {platform_style}
           Content Type: {content_type}
           Call to Action: {call_to_action or "Shop now"}

           Requirements:
           - Maximum {max_length} characters
           - Hook readers in the first line
           - Be authentic and engaging
           - Match the brand voice
           - Include the call to action naturally
           - Use emojis appropriately
           - For Instagram/TikTok: Use line breaks for readability

           Return ONLY the caption text, no JSON or extra formatting.
           """

           response = await self.llm.generate(
               prompt=prompt,
               temperature=0.8,  # More creative
           )

           caption = response.get("content", "").strip()

           # Ensure it meets length requirements
           if len(caption) > max_length:
               caption = caption[:max_length-3] + "..."

           return caption

       def _build_context(
           self,
           content_type: str,
           product: Optional[Dict],
           planned_content: Optional[Dict],
       ) -> str:
           """Build context for caption generation"""

           context_parts = []

           if planned_content:
               context_parts.append(f"Hook: {planned_content.get('hook', '')}")
               context_parts.append(f"Key Message: {planned_content.get('headline', '')}")

               if planned_content.get('key_points'):
                   points = '\n'.join(f"- {p}" for p in planned_content['key_points'])
                   context_parts.append(f"Key Points:\n{points}")

           if product:
               context_parts.append(f"Product: {product.get('title', '')}")
               context_parts.append(f"Description: {product.get('description', '')[:200]}")
               context_parts.append(f"Price: ${product.get('sellingPrice', 0)}")

           return '\n\n'.join(context_parts)

       async def write_variation(
           self,
           original_caption: str,
           platform: str,
           variation_style: str = "different",
       ) -> str:
           """Generate a variation of an existing caption"""

           prompt = f"""
           Create a {variation_style} variation of this caption for {platform}:

           Original:
           {original_caption}

           Requirements:
           - Keep the same core message
           - Use different wording and structure
           - Maintain the same tone
           - Stay within {self.PLATFORM_LIMITS.get(platform, 2200)} characters

           Return ONLY the new caption text.
           """

           response = await self.llm.generate(prompt=prompt, temperature=0.9)
           return response.get("content", "").strip()
   ```

2. **Implement Hashtag Optimizer**
   ```python
   # app/agents/social/hashtag_optimizer.py
   from typing import List, Dict, Any
   from app.services.llm import LLMService

   class HashtagOptimizer:
       """AI-powered hashtag research and optimization"""

       def __init__(self):
           self.llm = LLMService()

       # Platform-specific hashtag limits
       PLATFORM_LIMITS = {
           "instagram": 30,
           "tiktok": 20,
           "twitter": 2,  # Twitter best practice
           "pinterest": 20,
       }

       async def optimize(
           self,
           niche: str,
           caption: str,
           platform: str = "instagram",
           max_hashtags: Optional[int] = None,
       ) -> List[str]:
           """Generate optimized hashtags for content"""

           if max_hashtags is None:
               max_hashtags = self.PLATFORM_LIMITS.get(platform, 20)

           prompt = f"""
           Generate optimized hashtags for a {platform} post in the {niche} niche.

           Caption:
           {caption[:500]}

           Requirements:
           - Generate {max_hashtags} hashtags
           - Mix of popular (high reach) and niche (high engagement) hashtags
           - Include brand/campaign specific hashtags
           - Relevant to the caption and niche
           - No banned or spam hashtags
           - Follow {platform} best practices

           For Instagram/TikTok: Mix sizes (mega, macro, micro hashtags)
           For Twitter: Only 1-2 highly relevant hashtags

           Return a JSON object with:
           {{
             "hashtags": ["hashtag1", "hashtag2", ...],
             "strategy": "brief explanation of hashtag mix"
           }}

           Do NOT include the # symbol in the hashtags.
           """

           response = await self.llm.generate(
               prompt=prompt,
               response_format={"type": "json_object"},
           )

           hashtags = response.get("hashtags", [])

           # Clean and validate hashtags
           cleaned = []
           for tag in hashtags[:max_hashtags]:
               # Remove # if present
               tag = tag.strip().lstrip('#')
               # Remove spaces
               tag = tag.replace(' ', '')
               # Only alphanumeric and underscores
               if tag and tag.replace('_', '').isalnum():
                   cleaned.append(tag)

           return cleaned

       async def research_trending(
           self,
           niche: str,
           platform: str = "instagram",
       ) -> List[Dict[str, Any]]:
           """Research trending hashtags in a niche"""

           prompt = f"""
           Research current trending hashtags in the {niche} niche on {platform}.

           Provide:
           1. Top 10 trending hashtags
           2. Estimated reach for each (small/medium/large)
           3. Competition level (low/medium/high)
           4. Recommendation on usage

           Return JSON:
           {{
             "trending": [
               {{
                 "hashtag": "example",
                 "reach": "medium",
                 "competition": "low",
                 "recommendation": "use"
               }}
             ]
           }}
           """

           response = await self.llm.generate(
               prompt=prompt,
               response_format={"type": "json_object"},
           )

           return response.get("trending", [])

       def create_branded_hashtag(self, brand_name: str, campaign: str = "") -> str:
           """Create a branded hashtag"""
           # Clean brand name
           clean_brand = brand_name.replace(' ', '').replace('-', '')

           if campaign:
               clean_campaign = campaign.replace(' ', '').replace('-', '')
               return f"{clean_brand}{clean_campaign}"

           return clean_brand
   ```

3. **Add Database Models for Social Content**
   ```prisma
   # packages/database/prisma/schema.prisma (add)

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
     contentType     String    # image, video, carousel
     caption         String    @db.Text
     hashtags        String[]
     mediaUrls       String[]

     scheduledTime   DateTime?
     publishedTime   DateTime?
     platformPostId  String?
     platformUrl     String?

     status          PostStatus @default(DRAFT)
     error           String?

     analytics       Json?     # likes, comments, shares, etc.

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
   ```

4. **Run Database Migration**
   ```bash
   cd packages/database
   npx prisma migrate dev --name add_social_content_models
   npx prisma generate
   ```

#### Deliverables
- [ ] Caption writer generating platform-optimized captions
- [ ] Hashtag optimizer working
- [ ] Database models for content calendar and posts
- [ ] LLM integration tested for captions and hashtags

---

### Days 5-7: Posting Scheduler & Social Agent Orchestrator

#### Tasks
1. **Implement Posting Scheduler**
   ```python
   # app/agents/social/scheduler.py
   from typing import List, Dict, Any
   from datetime import datetime, timedelta, time
   import pytz

   class PostingScheduler:
       """Intelligent posting schedule optimization"""

       # Optimal posting times by platform (UTC)
       OPTIMAL_TIMES = {
           "instagram": [
               time(13, 0),  # 1 PM
               time(18, 0),  # 6 PM
               time(21, 0),  # 9 PM
           ],
           "tiktok": [
               time(14, 0),  # 2 PM
               time(19, 0),  # 7 PM
               time(22, 0),  # 10 PM
           ],
           "twitter": [
               time(12, 0),  # 12 PM
               time(17, 0),  # 5 PM
               time(20, 0),  # 8 PM
           ],
           "pinterest": [
               time(20, 0),  # 8 PM
               time(21, 0),  # 9 PM
           ],
       }

       # Best days by content type
       BEST_DAYS = {
           "video": [0, 3, 5],  # Mon, Thu, Sat
           "image": [1, 2, 4],  # Tue, Wed, Fri
           "carousel": [2, 4, 6],  # Wed, Fri, Sun
       }

       def __init__(self):
           pass

       async def schedule_posts(
           self,
           posts: List[Any],
           platforms: List[str],
           timezone: str = "UTC",
       ) -> List[Any]:
           """Schedule posts with optimal timing"""

           tz = pytz.timezone(timezone)
           scheduled_posts = []

           # Group posts by platform
           platform_posts = {p: [] for p in platforms}
           for post in posts:
               platform = post.platform if hasattr(post, 'platform') else platforms[0]
               platform_posts[platform].append(post)

           # Schedule each platform's posts
           start_date = datetime.now(tz)

           for platform, platform_post_list in platform_posts.items():
               optimal_times = self.OPTIMAL_TIMES.get(platform, [time(12, 0)])

               for i, post in enumerate(platform_post_list):
                   # Calculate day offset
                   day_offset = i // len(optimal_times)
                   time_index = i % len(optimal_times)

                   # Get optimal day based on content type
                   content_type = post.content_type if hasattr(post, 'content_type') else 'image'
                   preferred_days = self.BEST_DAYS.get(content_type, [0, 1, 2, 3, 4, 5, 6])

                   # Find next preferred day
                   schedule_date = start_date + timedelta(days=day_offset)
                   while schedule_date.weekday() not in preferred_days:
                       schedule_date += timedelta(days=1)

                   # Set optimal time
                   optimal_time = optimal_times[time_index]
                   schedule_datetime = schedule_date.replace(
                       hour=optimal_time.hour,
                       minute=optimal_time.minute,
                       second=0,
                       microsecond=0,
                   )

                   # Assign scheduled time
                   post.scheduled_time = schedule_datetime
                   scheduled_posts.append(post)

           # Sort by scheduled time
           scheduled_posts.sort(key=lambda p: p.scheduled_time)

           return scheduled_posts

       def get_next_slot(
           self,
           platform: str,
           existing_posts: List[datetime],
           timezone: str = "UTC",
       ) -> datetime:
           """Get next available posting slot"""

           tz = pytz.timezone(timezone)
           optimal_times = self.OPTIMAL_TIMES.get(platform, [time(12, 0)])

           current = datetime.now(tz)
           day_offset = 0

           while True:
               check_date = current + timedelta(days=day_offset)

               for optimal_time in optimal_times:
                   slot = check_date.replace(
                       hour=optimal_time.hour,
                       minute=optimal_time.minute,
                       second=0,
                       microsecond=0,
                   )

                   # Check if slot is in future and not taken
                   if slot > current and slot not in existing_posts:
                       return slot

               day_offset += 1

               # Safety check
               if day_offset > 30:
                   break

           # Fallback
           return current + timedelta(hours=2)
   ```

2. **Implement Social Agent Orchestrator**
   ```python
   # app/agents/social/agent.py
   from typing import List, Optional, Dict, Any
   from dataclasses import dataclass
   from datetime import datetime, timedelta
   from app.agents.base import BaseMarketingAgent
   from app.agents.social.content_planner import ContentPlanner
   from app.agents.social.caption_writer import CaptionWriter
   from app.agents.social.hashtag_optimizer import HashtagOptimizer
   from app.agents.social.scheduler import PostingScheduler
   from app.oauth.token_store import TokenStore

   @dataclass
   class SocialPost:
       id: str
       platform: str
       content_type: str
       caption: str
       hashtags: List[str]
       media_urls: List[str]
       scheduled_time: Optional[datetime]
       status: str

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
           """Generate a complete content calendar for the store"""

           # Get store data
           store = await self.get_store_data(store_id)
           products = await self.get_store_products(store_id)
           media_assets = await self.get_store_media(store_id)
           connected_platforms = await self.get_connected_platforms(store_id)

           if not connected_platforms:
               raise ValueError("No social platforms connected")

           # Get brand info
           niche = store.niche if hasattr(store, 'niche') else "general"
           brand_voice = "professional"
           if hasattr(store, 'brandIdentity') and store.brandIdentity:
               brand_voice = store.brandIdentity.get("tone", "professional")

           # Generate content plan using AI
           content_plan = await self.content_planner.create_plan(
               niche=niche,
               brand_voice=brand_voice,
               products=products,
               days=days,
               posts_per_day=posts_per_day,
           )

           # Create posts from plan
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
           timezone = "UTC"
           if hasattr(store, 'settings') and store.settings:
               timezone = store.settings.get("timezone", "UTC")

           scheduled_posts = await self.scheduler.schedule_posts(
               posts=posts,
               platforms=connected_platforms,
               timezone=timezone,
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
           store: Any,
           planned_content: Dict,
           media_assets: List[Dict],
           platforms: List[str],
       ) -> SocialPost:
           """Create a social post from planned content"""

           # Select appropriate media
           media_urls = self._select_media(
               planned_content["content_type"],
               planned_content.get("product_id"),
               media_assets,
           )

           # Use primary platform for post creation
           platform = platforms[0]

           # Generate caption
           caption = await self.caption_writer.write(
               content_type=planned_content["content_type"],
               product=planned_content.get("product"),
               brand_voice=store.brandIdentity.get("tone") if hasattr(store, 'brandIdentity') else "professional",
               platform=platform,
               call_to_action=planned_content.get("cta"),
               planned_content=planned_content,
           )

           # Optimize hashtags
           niche = store.niche if hasattr(store, 'niche') else "general"
           hashtags = await self.hashtag_optimizer.optimize(
               niche=niche,
               caption=caption,
               platform=platform,
           )

           return SocialPost(
               id=self.generate_id(),
               platform=platform,
               content_type=planned_content["content_type"],
               caption=caption,
               hashtags=hashtags,
               media_urls=media_urls,
               scheduled_time=None,  # Set by scheduler
               status="draft",
           )

       def _select_media(
           self,
           content_type: str,
           product_id: Optional[str],
           media_assets: List[Dict],
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
                       videos = [m for m in product_media if m.get("type") == "VIDEO"]
                       return [videos[0]["url"]] if videos else []
                   else:
                       images = [m for m in product_media if m.get("type") == "IMAGE"]
                       return [images[0]["url"]] if images else []

           # Fallback to general store media
           if content_type == "video":
               videos = [m for m in media_assets if m.get("type") == "VIDEO"]
               return [videos[0]["url"]] if videos else []
           else:
               images = [m for m in media_assets if m.get("type") == "IMAGE"]
               return [images[0]["url"]] if images else []

       async def _save_calendar(self, calendar: ContentCalendar):
           """Save content calendar to database"""
           from app.services.database import prisma

           # Create calendar
           db_calendar = await prisma.contentcalendar.create(
               data={
                   "storeId": calendar.store_id,
                   "startDate": calendar.start_date,
                   "endDate": calendar.end_date,
                   "status": "ACTIVE",
               }
           )

           # Create posts
           for post in calendar.posts:
               await prisma.socialpost.create(
                   data={
                       "calendarId": db_calendar.id,
                       "storeId": calendar.store_id,
                       "platform": post.platform,
                       "contentType": post.content_type,
                       "caption": post.caption,
                       "hashtags": post.hashtags,
                       "mediaUrls": post.media_urls,
                       "scheduledTime": post.scheduled_time,
                       "status": post.status.upper(),
                   }
               )
   ```

#### Deliverables
- [ ] Posting scheduler with optimal timing
- [ ] Social Agent orchestrator complete
- [ ] Content calendar generation working
- [ ] Posts saved to database with scheduling

---

### Days 8-10: TikTok Platform Client

#### Tasks
1. **Create Base Platform Client**
   ```python
   # app/agents/social/platforms/base.py
   from abc import ABC, abstractmethod
   from typing import List, Dict, Any
   from datetime import datetime

   class BasePlatformClient(ABC):
       """Base class for social media platform clients"""

       @abstractmethod
       async def upload_media(self, media_url: str) -> str:
           """Upload media to platform, return media ID"""
           pass

       @abstractmethod
       async def create_post(
           self,
           caption: str,
           hashtags: List[str],
           media_ids: List[str],
       ) -> Dict[str, Any]:
           """Create a post on the platform"""
           pass

       @abstractmethod
       async def get_insights(
           self,
           start_date: datetime,
           end_date: datetime,
       ) -> Dict[str, Any]:
           """Get account insights"""
           pass

       @abstractmethod
       async def get_post_analytics(self, post_id: str) -> Dict[str, Any]:
           """Get analytics for a specific post"""
           pass

       def format_caption(self, caption: str, hashtags: List[str]) -> str:
           """Format caption with hashtags"""
           if not hashtags:
               return caption

           hashtag_str = ' '.join([f'#{tag}' for tag in hashtags])
           return f"{caption}\n\n{hashtag_str}"
   ```

2. **Implement TikTok Platform Client**
   ```python
   # app/agents/social/platforms/tiktok.py
   from typing import List, Dict, Any
   from datetime import datetime
   import httpx
   import asyncio
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
                   timeout=30.0,
                   **kwargs,
               )
               response.raise_for_status()
               return response.json()

       async def upload_media(self, media_url: str) -> str:
           """
           Upload video to TikTok.
           Returns publish_id that can be used to create post.
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
           for attempt in range(max_attempts):
               await asyncio.sleep(2)

               status = await self._request(
                   "POST",
                   "/post/publish/status/fetch/",
                   json={"publish_id": publish_id},
               )

               status_code = status["data"]["status"]

               if status_code == "PUBLISH_COMPLETE":
                   return publish_id

               if status_code == "FAILED":
                   reason = status["data"].get("fail_reason", "Unknown error")
                   raise Exception(f"Upload failed: {reason}")

           raise Exception("Upload timeout - video processing took too long")

       async def create_post(
           self,
           caption: str,
           hashtags: List[str],
           media_ids: List[str],
       ) -> Dict[str, Any]:
           """
           Publish a video post to TikTok.
           media_ids should contain the publish_id from upload_media.
           """

           if not media_ids:
               raise ValueError("TikTok requires at least one video")

           # Format caption with hashtags
           full_caption = self.format_caption(caption, hashtags)

           # TikTok publish API
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
                       "source": "PULL_FROM_URL",
                       "video_url": media_ids[0],  # Assume first ID is URL
                   },
               },
           )

           return {
               "id": response["data"]["publish_id"],
               "status": "published",
               "url": f"https://www.tiktok.com/@{self.open_id}/video/{response['data']['publish_id']}",
           }

       async def get_insights(
           self,
           start_date: datetime,
           end_date: datetime,
       ) -> Dict[str, Any]:
           """Get TikTok account insights"""

           response = await self._request(
               "POST",
               "/user/info/",
               params={
                   "fields": "follower_count,following_count,likes_count,video_count",
               },
           )

           user_data = response.get("data", {}).get("user", {})

           return {
               "followers": user_data.get("follower_count", 0),
               "following": user_data.get("following_count", 0),
               "likes": user_data.get("likes_count", 0),
               "videos": user_data.get("video_count", 0),
           }

       async def get_post_analytics(self, post_id: str) -> Dict[str, Any]:
           """Get analytics for a specific TikTok video"""

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

           videos = response.get("data", {}).get("videos", [])
           if not videos:
               return {}

           video = videos[0]
           return {
               "views": video.get("view_count", 0),
               "likes": video.get("like_count", 0),
               "comments": video.get("comment_count", 0),
               "shares": video.get("share_count", 0),
               "engagement_rate": self._calculate_engagement_rate(video),
           }

       def _calculate_engagement_rate(self, video: Dict) -> float:
           """Calculate engagement rate for video"""
           views = video.get("view_count", 0)
           if views == 0:
               return 0.0

           engagements = (
               video.get("like_count", 0) +
               video.get("comment_count", 0) +
               video.get("share_count", 0)
           )

           return (engagements / views) * 100
   ```

3. **Create Platform Client Factory**
   ```python
   # app/agents/social/platforms/__init__.py
   from typing import Dict, Any
   from .base import BasePlatformClient
   from .tiktok import TikTokClient
   from .instagram import InstagramClient

   def get_platform_client(
       platform: str,
       tokens: Dict[str, Any],
   ) -> BasePlatformClient:
       """Factory function to get platform client"""

       if platform == "tiktok":
           return TikTokClient(tokens)
       elif platform == "meta" or platform == "instagram":
           return InstagramClient(tokens)
       else:
           raise ValueError(f"Unsupported platform: {platform}")
   ```

#### Deliverables
- [ ] TikTok client implemented
- [ ] Video upload to TikTok working
- [ ] Post creation to TikTok functional
- [ ] Analytics retrieval working

---

### Days 11-12: Instagram Platform Client

#### Tasks
1. **Implement Instagram Platform Client**
   ```python
   # app/agents/social/platforms/instagram.py
   from typing import List, Dict, Any
   from datetime import datetime
   import httpx
   import asyncio
   from .base import BasePlatformClient

   class InstagramClient(BasePlatformClient):
       """Instagram API client (via Facebook Graph API)"""

       BASE_URL = "https://graph.facebook.com/v18.0"

       def __init__(self, tokens: Dict[str, Any]):
           self.access_token = tokens["access_token"]
           # Instagram Business Account ID from Meta connection
           self.ig_account_id = tokens.get("ig_account_id")

           if not self.ig_account_id:
               raise ValueError("Instagram Business Account not connected")

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

       async def upload_media(self, media_url: str) -> str:
           """
           Create Instagram media container.
           Returns container ID.
           """

           # Determine if image or video
           is_video = media_url.lower().endswith(('.mp4', '.mov'))

           if is_video:
               # Create video container
               response = await self._request(
                   "POST",
                   f"/{self.ig_account_id}/media",
                   params={
                       "media_type": "VIDEO",
                       "video_url": media_url,
                   },
               )
           else:
               # Create image container
               response = await self._request(
                   "POST",
                   f"/{self.ig_account_id}/media",
                   params={
                       "image_url": media_url,
                   },
               )

           container_id = response["id"]

           # For videos, wait for processing
           if is_video:
               await self._wait_for_video_processing(container_id)

           return container_id

       async def _wait_for_video_processing(self, container_id: str):
           """Wait for Instagram video processing to complete"""

           max_attempts = 30
           for attempt in range(max_attempts):
               await asyncio.sleep(2)

               response = await self._request(
                   "GET",
                   f"/{container_id}",
                   params={"fields": "status_code"},
               )

               status = response.get("status_code")

               if status == "FINISHED":
                   return
               elif status == "ERROR":
                   raise Exception("Video processing failed")

           raise Exception("Video processing timeout")

       async def create_post(
           self,
           caption: str,
           hashtags: List[str],
           media_ids: List[str],
       ) -> Dict[str, Any]:
           """Publish post to Instagram"""

           if not media_ids:
               raise ValueError("Instagram requires at least one media item")

           # Format caption with hashtags
           full_caption = self.format_caption(caption, hashtags)

           # Publish media container
           response = await self._request(
               "POST",
               f"/{self.ig_account_id}/media_publish",
               params={
                   "creation_id": media_ids[0],
                   "caption": full_caption[:2200],  # Instagram limit
               },
           )

           post_id = response["id"]

           return {
               "id": post_id,
               "status": "published",
               "url": f"https://www.instagram.com/p/{post_id}/",
           }

       async def get_insights(
           self,
           start_date: datetime,
           end_date: datetime,
       ) -> Dict[str, Any]:
           """Get Instagram account insights"""

           # Get account metrics
           response = await self._request(
               "GET",
               f"/{self.ig_account_id}",
               params={
                   "fields": "followers_count,follows_count,media_count",
               },
           )

           # Get insights metrics
           insights_response = await self._request(
               "GET",
               f"/{self.ig_account_id}/insights",
               params={
                   "metric": "impressions,reach,profile_views,follower_count",
                   "period": "day",
                   "since": int(start_date.timestamp()),
                   "until": int(end_date.timestamp()),
               },
           )

           insights = {}
           for metric in insights_response.get("data", []):
               name = metric["name"]
               values = metric.get("values", [])
               if values:
                   insights[name] = sum(v["value"] for v in values)

           return {
               "followers": response.get("followers_count", 0),
               "following": response.get("follows_count", 0),
               "posts": response.get("media_count", 0),
               "impressions": insights.get("impressions", 0),
               "reach": insights.get("reach", 0),
               "profile_views": insights.get("profile_views", 0),
           }

       async def get_post_analytics(self, post_id: str) -> Dict[str, Any]:
           """Get analytics for a specific Instagram post"""

           response = await self._request(
               "GET",
               f"/{post_id}/insights",
               params={
                   "metric": "impressions,reach,engagement,likes,comments,shares,saves",
               },
           )

           analytics = {}
           for metric in response.get("data", []):
               name = metric["name"]
               values = metric.get("values", [])
               if values:
                   analytics[name] = values[0]["value"]

           return {
               "impressions": analytics.get("impressions", 0),
               "reach": analytics.get("reach", 0),
               "likes": analytics.get("likes", 0),
               "comments": analytics.get("comments", 0),
               "shares": analytics.get("shares", 0),
               "saves": analytics.get("saves", 0),
               "engagement": analytics.get("engagement", 0),
           }
   ```

2. **Update Platform Factory**
   ```python
   # app/agents/social/platforms/__init__.py (update)
   from .instagram import InstagramClient

   def get_platform_client(
       platform: str,
       tokens: Dict[str, Any],
   ) -> BasePlatformClient:
       """Factory function to get platform client"""

       if platform == "tiktok":
           return TikTokClient(tokens)
       elif platform == "meta" or platform == "instagram":
           return InstagramClient(tokens)
       elif platform == "pinterest":
           from .pinterest import PinterestClient
           return PinterestClient(tokens)
       elif platform == "twitter":
           from .twitter import TwitterClient
           return TwitterClient(tokens)
       else:
           raise ValueError(f"Unsupported platform: {platform}")
   ```

#### Deliverables
- [ ] Instagram client implemented
- [ ] Image upload to Instagram working
- [ ] Video upload to Instagram working
- [ ] Post creation functional
- [ ] Analytics retrieval working

---

### Days 13-14: API Endpoints, Testing & Documentation

#### Tasks
1. **Create Social Agent API Endpoints**
   ```python
   # app/api/v1/social.py
   from fastapi import APIRouter, HTTPException, Query
   from typing import Optional
   from app.agents.social.agent import SocialAgent

   router = APIRouter(prefix="/social", tags=["social"])
   social_agent = SocialAgent()

   @router.post("/calendar/generate")
   async def generate_content_calendar(
       store_id: str = Query(...),
       days: int = Query(30, ge=7, le=90),
       posts_per_day: int = Query(2, ge=1, le=5),
   ):
       """Generate a content calendar for a store"""
       try:
           calendar = await social_agent.generate_content_calendar(
               store_id=store_id,
               days=days,
               posts_per_day=posts_per_day,
           )

           return {
               "success": True,
               "calendar": {
                   "store_id": calendar.store_id,
                   "start_date": calendar.start_date,
                   "end_date": calendar.end_date,
                   "total_posts": len(calendar.posts),
                   "platforms": calendar.platforms,
               },
           }
       except Exception as e:
           raise HTTPException(status_code=400, detail=str(e))

   @router.get("/calendar/{store_id}")
   async def get_content_calendar(store_id: str):
       """Get active content calendar for a store"""
       from app.services.database import prisma

       calendar = await prisma.contentcalendar.find_first(
           where={
               "storeId": store_id,
               "status": "ACTIVE",
           },
           include={"posts": True},
       )

       if not calendar:
           raise HTTPException(status_code=404, detail="No active calendar found")

       return {
           "id": calendar.id,
           "start_date": calendar.startDate,
           "end_date": calendar.endDate,
           "total_posts": len(calendar.posts),
           "posts": [
               {
                   "id": p.id,
                   "platform": p.platform,
                   "content_type": p.contentType,
                   "caption": p.caption[:100] + "...",
                   "scheduled_time": p.scheduledTime,
                   "status": p.status,
               }
               for p in calendar.posts
           ],
       }

   @router.get("/posts/{post_id}")
   async def get_post(post_id: str):
       """Get details of a specific post"""
       from app.services.database import prisma

       post = await prisma.socialpost.find_unique(
           where={"id": post_id}
       )

       if not post:
           raise HTTPException(status_code=404, detail="Post not found")

       return {
           "id": post.id,
           "platform": post.platform,
           "content_type": post.contentType,
           "caption": post.caption,
           "hashtags": post.hashtags,
           "media_urls": post.mediaUrls,
           "scheduled_time": post.scheduledTime,
           "published_time": post.publishedTime,
           "status": post.status,
           "analytics": post.analytics,
       }

   @router.post("/posts/{post_id}/publish")
   async def publish_post(post_id: str):
       """Manually publish a post"""
       # This will be implemented in Sprint 3
       raise HTTPException(status_code=501, detail="Publishing will be available in Sprint 3")
   ```

2. **Include Router in Main API**
   ```python
   # app/api/v1/router.py
   from app.api.v1.social import router as social_router

   api_router.include_router(social_router)
   ```

3. **Write Tests**
   ```python
   # tests/test_social_agent.py
   import pytest
   from app.agents.social.content_planner import ContentPlanner
   from app.agents.social.caption_writer import CaptionWriter
   from app.agents.social.hashtag_optimizer import HashtagOptimizer

   @pytest.mark.asyncio
   async def test_content_planner():
       planner = ContentPlanner()

       content_plan = await planner.create_plan(
           niche="minimalist home decor",
           brand_voice="calm and inspiring",
           products=[
               {"id": "1", "title": "Ceramic Vase", "description": "Beautiful minimal vase", "sellingPrice": 29.99}
           ],
           days=7,
           posts_per_day=2,
       )

       assert len(content_plan) == 14
       assert content_plan[0]["type"] in ["product", "brand"]

   @pytest.mark.asyncio
   async def test_caption_writer():
       writer = CaptionWriter()

       caption = await writer.write(
           content_type="image",
           platform="instagram",
           brand_voice="friendly",
           product={"title": "Ceramic Vase", "description": "Beautiful vase"},
       )

       assert len(caption) > 0
       assert len(caption) <= 2200

   @pytest.mark.asyncio
   async def test_hashtag_optimizer():
       optimizer = HashtagOptimizer()

       hashtags = await optimizer.optimize(
           niche="home decor",
           caption="Beautiful minimalist design for your home",
           platform="instagram",
       )

       assert len(hashtags) > 0
       assert len(hashtags) <= 30
       assert all('#' not in tag for tag in hashtags)
   ```

4. **Create Documentation**
   ```markdown
   # Social Agent Documentation

   ## Overview
   The Social Agent autonomously creates, schedules, and manages social media content across platforms.

   ## Features
   - AI-powered content planning (30-day calendars)
   - Platform-optimized caption generation
   - Intelligent hashtag research
   - Optimal posting time scheduling
   - TikTok and Instagram publishing

   ## API Endpoints

   ### Generate Content Calendar
   ```
   POST /api/v1/social/calendar/generate
   Query Parameters:
     - store_id: string (required)
     - days: integer (7-90, default: 30)
     - posts_per_day: integer (1-5, default: 2)
   ```

   ### Get Content Calendar
   ```
   GET /api/v1/social/calendar/{store_id}
   ```

   ### Get Post Details
   ```
   GET /api/v1/social/posts/{post_id}
   ```

   ## Usage Example

   ```python
   from app.agents.social.agent import SocialAgent

   agent = SocialAgent()
   calendar = await agent.generate_content_calendar(
       store_id="store_123",
       days=30,
       posts_per_day=2,
   )

   print(f"Generated {len(calendar.posts)} posts")
   ```

   ## Content Planning Logic

   1. **Theme Generation**: AI creates 10 content themes based on niche
   2. **Content Mix**: Alternates between product and brand posts
   3. **Caption Writing**: Platform-specific captions with brand voice
   4. **Hashtag Optimization**: Mix of popular and niche hashtags
   5. **Smart Scheduling**: Posts at optimal times for each platform

   ## Platform Support

   | Platform | Upload | Publish | Analytics |
   |----------|--------|---------|-----------|
   | TikTok | ✅ | ✅ | ✅ |
   | Instagram | ✅ | ✅ | ✅ |
   | Pinterest | 🚧 | 🚧 | 🚧 |
   | Twitter | 🚧 | 🚧 | 🚧 |
   ```

#### Deliverables
- [ ] API endpoints for social agent created
- [ ] Unit tests passing
- [ ] Integration tests passing
- [ ] Documentation complete

---

## Definition of Done

### Technical Requirements
- [ ] Content planner generating 30-day calendars
- [ ] Caption writer creating platform-optimized captions
- [ ] Hashtag optimizer returning relevant hashtags
- [ ] Posting scheduler assigning optimal times
- [ ] TikTok client uploading and publishing videos
- [ ] Instagram client uploading images/videos and publishing
- [ ] All unit tests passing (>80% coverage)
- [ ] Database migrations applied

### Functional Requirements
- [ ] End-to-end content calendar generation working
- [ ] Posts saved to database with all metadata
- [ ] Media selection working correctly
- [ ] Captions under platform limits
- [ ] Hashtags validated and cleaned
- [ ] Schedule considers timezones

### Quality Requirements
- [ ] LLM responses validated and parsed correctly
- [ ] Error handling for platform API failures
- [ ] Token refresh handled gracefully
- [ ] No hardcoded credentials
- [ ] Logging implemented for debugging

---

## Testing Checklist

### Unit Tests
- [ ] Content planner generates correct number of posts
- [ ] Caption writer respects character limits
- [ ] Hashtag optimizer cleans and validates tags
- [ ] Scheduler assigns times within constraints
- [ ] Platform clients format requests correctly

### Integration Tests
- [ ] Full calendar generation with real store data
- [ ] TikTok upload and publish (requires test account)
- [ ] Instagram upload and publish (requires test account)
- [ ] Analytics retrieval from platforms
- [ ] Database persistence of calendar and posts

### Manual Testing
- [ ] Generate calendar via API
- [ ] Verify posts in database
- [ ] Check caption quality
- [ ] Verify hashtags are relevant
- [ ] Confirm scheduling logic

---

## Success Metrics

- Content calendar generates in < 2 minutes
- Captions are engaging and on-brand (manual review)
- Hashtags are 90%+ relevant to niche
- Scheduling spreads posts evenly across days
- Platform uploads succeed 95%+ of the time
- LLM API calls complete in < 10 seconds each

---

## Next Sprint Preview

**Sprint 3: Social Agent Advanced** will add:
- Pinterest and Twitter platform clients
- Content calendar UI in dashboard
- Post preview and editing
- Bulk scheduling interface
- Cross-platform analytics aggregation
- Actual automated publishing with worker queue

---

This sprint establishes the **core autonomous content creation engine** that will power Zyenta's social media presence. The AI-driven approach ensures high-quality, on-brand content at scale.
