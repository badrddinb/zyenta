# Phase 4 - Sprint 4: Influencer Discovery - Action Plan

## Sprint Overview

**Duration:** 2 weeks (14 days)
**Team Size:** 3-4 developers (2 backend, 1 frontend, 1 AI specialist)
**Sprint Goal:** Build multi-platform influencer discovery system with AI-powered profile analysis and brand matching.

### Objectives
- [ ] Implement influencer discovery across Instagram, TikTok, YouTube, Twitter
- [ ] Build AI-powered profile analyzer with engagement metrics
- [ ] Create brand-influencer matching system with affinity scoring
- [ ] Develop influencer database UI with search and filters
- [ ] Implement tier categorization (nano, micro, macro, mega)
- [ ] Add fake follower detection and authenticity checks

### Prerequisites
- Phase 3 (Growth Hub) operational
- OAuth system ready for platform connections
- Database schema updated for influencer data
- Redis available for caching
- LLM API access (OpenAI/Anthropic) for AI analysis

---

## Database Models

### Prisma Schema Extensions

```prisma
model Influencer {
  id                String   @id @default(cuid())
  storeId           String
  platform          String   // instagram, tiktok, youtube, twitter
  platformUserId    String
  username          String
  displayName       String?
  profileUrl        String
  avatarUrl         String?
  bio               String?

  // Follower/Subscriber Stats
  followersCount    Int      @default(0)
  followingCount    Int?
  postsCount        Int?

  // Engagement Metrics
  engagementRate    Decimal  @db.Decimal(5, 2) // percentage
  avgLikes          Int?
  avgComments       Int?
  avgShares         Int?
  avgViews          Int?     // for video platforms

  // Classification
  tier              String   // nano, micro, macro, mega
  niche             String[] // categories/topics

  // Authenticity
  authenticityScore Decimal  @db.Decimal(5, 2) // 0-100
  fakeFollowerPct   Decimal? @db.Decimal(5, 2)

  // Audience Demographics
  audienceCountries Json?    // {"US": 45, "UK": 20, ...}
  audienceAgeGroups Json?    // {"18-24": 30, "25-34": 50, ...}
  audienceGenders   Json?    // {"male": 40, "female": 60}

  // Brand Matching
  brandAffinityScore Decimal? @db.Decimal(5, 2)
  contentStyle       String[]
  valueAlignment     String[]

  // Contact Info (if available)
  email             String?
  website           String?

  // Status
  status            String   @default("discovered") // discovered, contacted, interested, partnered
  discoveredAt      DateTime @default(now())
  lastAnalyzedAt    DateTime @default(now())

  // Relations
  store             Store    @relation(fields: [storeId], references: [id], onDelete: Cascade)
  metrics           InfluencerMetric[]
  campaigns         InfluencerCampaign[]
  outreach          InfluencerOutreach[]

  @@unique([storeId, platform, platformUserId])
  @@index([storeId, tier])
  @@index([storeId, engagementRate])
  @@index([storeId, brandAffinityScore])
}

model InfluencerMetric {
  id              String   @id @default(cuid())
  influencerId    String

  // Snapshot metrics
  followersCount  Int
  engagementRate  Decimal  @db.Decimal(5, 2)
  avgLikes        Int?
  avgComments     Int?
  avgViews        Int?

  recordedAt      DateTime @default(now())

  influencer      Influencer @relation(fields: [influencerId], references: [id], onDelete: Cascade)

  @@index([influencerId, recordedAt])
}

model InfluencerSearch {
  id              String   @id @default(cuid())
  storeId         String
  platform        String
  searchType      String   // hashtag, keyword, location, similar
  query           String

  // Results
  resultsCount    Int      @default(0)
  influencersFound Int     @default(0)

  // Filters applied
  filters         Json?    // minFollowers, maxFollowers, minEngagement, etc.

  createdAt       DateTime @default(now())
  completedAt     DateTime?
  status          String   @default("pending") // pending, processing, completed, failed

  store           Store    @relation(fields: [storeId], references: [id], onDelete: Cascade)

  @@index([storeId, createdAt])
}
```

---

## Day-by-Day Breakdown

### **Days 1-2: Instagram Discovery**

**Backend Tasks:**
1. Create Instagram Business Discovery API client
2. Implement hashtag search functionality
3. Add location-based discovery
4. Build similar account discovery
5. Add rate limiting and error handling

**Database Tasks:**
1. Set up influencer tables in database
2. Create indexes for search optimization

**Testing:**
- Test hashtag search with popular hashtags
- Verify profile data extraction
- Test rate limiting behavior

### **Days 3-4: TikTok & YouTube Discovery**

**Backend Tasks:**
1. Implement TikTok creator discovery (hashtag, trending)
2. Build YouTube channel discovery (search, related channels)
3. Add video performance analysis for both platforms
4. Implement pagination for large result sets

**Testing:**
- Test TikTok hashtag and sound searches
- Verify YouTube search and channel data
- Test video metrics extraction

### **Day 5: Twitter/X Discovery**

**Backend Tasks:**
1. Implement Twitter profile discovery
2. Add follower/following graph analysis
3. Build hashtag and keyword search
4. Add tweet engagement analysis

**Testing:**
- Test Twitter search with various queries
- Verify profile metric extraction

### **Days 6-7: Profile Analyzer**

**Backend Tasks:**
1. Build engagement rate calculator (platform-specific formulas)
2. Implement content analysis (post frequency, types)
3. Add audience demographics parser
4. Create authenticity checker (follower growth patterns, engagement consistency)
5. Build niche categorization using AI

**AI Tasks:**
1. Create LLM prompts for content analysis
2. Build niche detection from bio and posts
3. Implement content quality scorer

**Testing:**
- Test engagement calculations against known profiles
- Verify authenticity scoring
- Test AI categorization accuracy

### **Days 8-9: Fake Follower Detection**

**Backend Tasks:**
1. Build follower growth pattern analyzer
2. Implement engagement consistency checker
3. Add comment quality analysis (spam detection)
4. Create composite authenticity score
5. Build benchmarking against industry averages

**Algorithm Implementation:**
- Sudden follower spikes detection
- Engagement drop-off patterns
- Bot-like comment detection
- Follower-to-engagement ratio analysis

**Testing:**
- Test with known authentic accounts
- Test with suspicious accounts
- Verify scoring thresholds

### **Days 10-11: Brand Matching System**

**Backend Tasks:**
1. Build brand affinity scorer using AI
2. Implement audience overlap calculator
3. Add content style matcher
4. Create value alignment detector
5. Build composite match score

**AI Tasks:**
1. Create prompts for brand-influencer fit analysis
2. Build content style classifier
3. Implement value extraction from bios and posts

**Testing:**
- Test matching with various brand types
- Verify scoring accuracy
- Test edge cases (no match, perfect match)

### **Day 12: Tier Categorization**

**Backend Tasks:**
1. Implement tier classification logic:
   - Nano: 1K-10K followers
   - Micro: 10K-100K followers
   - Macro: 100K-1M followers
   - Mega: 1M+ followers
2. Add platform-specific tier adjustments
3. Build tier-based recommendation system

**Testing:**
- Test tier assignment for various follower counts
- Verify platform-specific rules

### **Days 13-14: Influencer Database UI**

**Frontend Tasks:**
1. Create influencer discovery dashboard
2. Build search interface with filters
3. Add profile detail modal/page
4. Implement save and organize features
5. Build export to campaign functionality
6. Add metrics visualization (charts)

**Testing:**
- Test search and filter combinations
- Verify profile display with all data
- Test save and export features
- Check responsive design

---

## Core Components Implementation

### 1. Influencer Discovery Service

```python
# services/influencer/discovery.py

from typing import List, Dict, Optional, Any
from datetime import datetime, timedelta
from decimal import Decimal
import asyncio
import httpx
from prisma import Prisma
from redis import Redis

class InfluencerDiscovery:
    """Multi-platform influencer discovery service"""

    def __init__(
        self,
        prisma: Prisma,
        redis: Redis,
        instagram_client: 'InstagramDiscoveryClient',
        tiktok_client: 'TikTokDiscoveryClient',
        youtube_client: 'YouTubeDiscoveryClient',
        twitter_client: 'TwitterDiscoveryClient',
    ):
        self.prisma = prisma
        self.redis = redis
        self.clients = {
            'instagram': instagram_client,
            'tiktok': tiktok_client,
            'youtube': youtube_client,
            'twitter': twitter_client,
        }

    async def discover(
        self,
        store_id: str,
        platform: str,
        search_type: str,
        query: str,
        filters: Optional[Dict[str, Any]] = None,
        limit: int = 50,
    ) -> Dict[str, Any]:
        """
        Discover influencers on specified platform

        Args:
            store_id: Store ID
            platform: instagram, tiktok, youtube, twitter
            search_type: hashtag, keyword, location, similar
            query: Search query
            filters: Optional filters (minFollowers, maxFollowers, etc.)
            limit: Max results to return

        Returns:
            Discovery results with influencer profiles
        """
        # Create search record
        search = await self.prisma.influencersearch.create(
            data={
                'storeId': store_id,
                'platform': platform,
                'searchType': search_type,
                'query': query,
                'filters': filters or {},
                'status': 'processing',
            }
        )

        try:
            # Get platform client
            client = self.clients.get(platform)
            if not client:
                raise ValueError(f"Unsupported platform: {platform}")

            # Execute search
            profiles = await client.search(
                search_type=search_type,
                query=query,
                limit=limit,
            )

            # Apply filters
            if filters:
                profiles = self._apply_filters(profiles, filters)

            # Save influencers to database
            saved_count = 0
            for profile in profiles:
                try:
                    await self._save_influencer(
                        store_id=store_id,
                        platform=platform,
                        profile=profile,
                    )
                    saved_count += 1
                except Exception as e:
                    print(f"Error saving influencer: {e}")
                    continue

            # Update search record
            await self.prisma.influencersearch.update(
                where={'id': search.id},
                data={
                    'status': 'completed',
                    'resultsCount': len(profiles),
                    'influencersFound': saved_count,
                    'completedAt': datetime.utcnow(),
                }
            )

            return {
                'search_id': search.id,
                'platform': platform,
                'results_count': len(profiles),
                'influencers_found': saved_count,
                'profiles': profiles,
            }

        except Exception as e:
            # Update search as failed
            await self.prisma.influencersearch.update(
                where={'id': search.id},
                data={
                    'status': 'failed',
                    'completedAt': datetime.utcnow(),
                }
            )
            raise

    def _apply_filters(
        self,
        profiles: List[Dict],
        filters: Dict[str, Any],
    ) -> List[Dict]:
        """Apply filters to profile results"""
        filtered = profiles

        # Follower count filters
        if 'minFollowers' in filters:
            filtered = [
                p for p in filtered
                if p.get('followers_count', 0) >= filters['minFollowers']
            ]

        if 'maxFollowers' in filters:
            filtered = [
                p for p in filtered
                if p.get('followers_count', 0) <= filters['maxFollowers']
            ]

        # Engagement rate filter
        if 'minEngagement' in filters:
            filtered = [
                p for p in filtered
                if p.get('engagement_rate', 0) >= filters['minEngagement']
            ]

        # Niche filter
        if 'niches' in filters:
            target_niches = set(filters['niches'])
            filtered = [
                p for p in filtered
                if set(p.get('niche', [])) & target_niches
            ]

        return filtered

    async def _save_influencer(
        self,
        store_id: str,
        platform: str,
        profile: Dict[str, Any],
    ) -> 'Influencer':
        """Save or update influencer in database"""
        # Check if already exists
        existing = await self.prisma.influencer.find_first(
            where={
                'storeId': store_id,
                'platform': platform,
                'platformUserId': profile['id'],
            }
        )

        # Determine tier
        tier = self._classify_tier(
            platform=platform,
            followers=profile.get('followers_count', 0),
        )

        data = {
            'username': profile['username'],
            'displayName': profile.get('display_name'),
            'profileUrl': profile['profile_url'],
            'avatarUrl': profile.get('avatar_url'),
            'bio': profile.get('bio'),
            'followersCount': profile.get('followers_count', 0),
            'followingCount': profile.get('following_count'),
            'postsCount': profile.get('posts_count'),
            'engagementRate': Decimal(str(profile.get('engagement_rate', 0))),
            'avgLikes': profile.get('avg_likes'),
            'avgComments': profile.get('avg_comments'),
            'avgShares': profile.get('avg_shares'),
            'avgViews': profile.get('avg_views'),
            'tier': tier,
            'niche': profile.get('niche', []),
            'lastAnalyzedAt': datetime.utcnow(),
        }

        if existing:
            # Update existing
            influencer = await self.prisma.influencer.update(
                where={'id': existing.id},
                data=data,
            )
        else:
            # Create new
            influencer = await self.prisma.influencer.create(
                data={
                    **data,
                    'storeId': store_id,
                    'platform': platform,
                    'platformUserId': profile['id'],
                }
            )

        return influencer

    def _classify_tier(self, platform: str, followers: int) -> str:
        """Classify influencer tier based on follower count"""
        if followers >= 1_000_000:
            return 'mega'
        elif followers >= 100_000:
            return 'macro'
        elif followers >= 10_000:
            return 'micro'
        else:
            return 'nano'

    async def discover_batch(
        self,
        store_id: str,
        searches: List[Dict[str, Any]],
    ) -> List[Dict[str, Any]]:
        """Run multiple searches in parallel"""
        tasks = [
            self.discover(
                store_id=store_id,
                platform=search['platform'],
                search_type=search['search_type'],
                query=search['query'],
                filters=search.get('filters'),
                limit=search.get('limit', 50),
            )
            for search in searches
        ]

        results = await asyncio.gather(*tasks, return_exceptions=True)

        return [
            r if not isinstance(r, Exception) else {'error': str(r)}
            for r in results
        ]
```

### 2. Instagram Discovery Client

```python
# services/influencer/clients/instagram.py

import httpx
from typing import List, Dict, Optional
from datetime import datetime

class InstagramDiscoveryClient:
    """Instagram Business Discovery API client"""

    def __init__(self, access_token: str):
        self.access_token = access_token
        self.base_url = "https://graph.facebook.com/v18.0"

    async def search(
        self,
        search_type: str,
        query: str,
        limit: int = 50,
    ) -> List[Dict]:
        """
        Search for Instagram profiles

        Search types:
        - hashtag: Search by hashtag
        - location: Search by location
        - similar: Find similar accounts
        """
        if search_type == 'hashtag':
            return await self._search_by_hashtag(query, limit)
        elif search_type == 'location':
            return await self._search_by_location(query, limit)
        elif search_type == 'similar':
            return await self._find_similar(query, limit)
        else:
            raise ValueError(f"Unknown search type: {search_type}")

    async def _search_by_hashtag(
        self,
        hashtag: str,
        limit: int,
    ) -> List[Dict]:
        """Search for top posts by hashtag and extract creators"""
        hashtag = hashtag.lstrip('#')

        async with httpx.AsyncClient() as client:
            # Get hashtag ID
            response = await client.get(
                f"{self.base_url}/ig_hashtag_search",
                params={
                    'user_id': 'me',  # Business account
                    'q': hashtag,
                    'access_token': self.access_token,
                }
            )
            response.raise_for_status()
            data = response.json()

            if not data.get('data'):
                return []

            hashtag_id = data['data'][0]['id']

            # Get recent media for hashtag
            response = await client.get(
                f"{self.base_url}/{hashtag_id}/recent_media",
                params={
                    'user_id': 'me',
                    'fields': 'id,media_type,media_url,permalink,like_count,comments_count',
                    'limit': limit,
                    'access_token': self.access_token,
                }
            )
            response.raise_for_status()
            media = response.json().get('data', [])

            # Extract unique creators and get their profiles
            creator_ids = set()
            for item in media:
                # Get media owner
                owner_response = await client.get(
                    f"{self.base_url}/{item['id']}",
                    params={
                        'fields': 'owner',
                        'access_token': self.access_token,
                    }
                )
                if owner_response.status_code == 200:
                    owner_data = owner_response.json()
                    if 'owner' in owner_data:
                        creator_ids.add(owner_data['owner']['id'])

            # Get full profiles
            profiles = []
            for creator_id in list(creator_ids)[:limit]:
                profile = await self._get_profile(creator_id)
                if profile:
                    profiles.append(profile)

            return profiles

    async def _get_profile(self, user_id: str) -> Optional[Dict]:
        """Get full Instagram profile data"""
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{self.base_url}/{user_id}",
                params={
                    'fields': 'id,username,name,biography,followers_count,follows_count,media_count,profile_picture_url',
                    'access_token': self.access_token,
                }
            )

            if response.status_code != 200:
                return None

            data = response.json()

            # Get recent media for engagement calculation
            media_response = await client.get(
                f"{self.base_url}/{user_id}/media",
                params={
                    'fields': 'like_count,comments_count',
                    'limit': 12,
                    'access_token': self.access_token,
                }
            )

            engagement_data = {'avg_likes': 0, 'avg_comments': 0, 'engagement_rate': 0}
            if media_response.status_code == 200:
                media = media_response.json().get('data', [])
                if media:
                    total_likes = sum(m.get('like_count', 0) for m in media)
                    total_comments = sum(m.get('comments_count', 0) for m in media)
                    avg_likes = total_likes / len(media)
                    avg_comments = total_comments / len(media)

                    # Engagement rate = (avg likes + avg comments) / followers * 100
                    followers = data.get('followers_count', 1)
                    engagement_rate = ((avg_likes + avg_comments) / followers) * 100 if followers > 0 else 0

                    engagement_data = {
                        'avg_likes': int(avg_likes),
                        'avg_comments': int(avg_comments),
                        'engagement_rate': round(engagement_rate, 2),
                    }

            return {
                'id': data['id'],
                'username': data['username'],
                'display_name': data.get('name'),
                'bio': data.get('biography'),
                'profile_url': f"https://instagram.com/{data['username']}",
                'avatar_url': data.get('profile_picture_url'),
                'followers_count': data.get('followers_count', 0),
                'following_count': data.get('follows_count'),
                'posts_count': data.get('media_count'),
                **engagement_data,
            }

    async def _search_by_location(
        self,
        location: str,
        limit: int,
    ) -> List[Dict]:
        """Search for influencers by location"""
        # Implementation depends on location API availability
        # This is a placeholder
        return []

    async def _find_similar(
        self,
        username: str,
        limit: int,
    ) -> List[Dict]:
        """Find similar accounts (based on followers/engagement)"""
        # This would typically use Instagram's graph API
        # to find accounts that share followers
        return []
```

### 3. TikTok Discovery Client

```python
# services/influencer/clients/tiktok.py

import httpx
from typing import List, Dict, Optional

class TikTokDiscoveryClient:
    """TikTok creator discovery client"""

    def __init__(self, access_token: str):
        self.access_token = access_token
        self.base_url = "https://open.tiktokapis.com/v2"

    async def search(
        self,
        search_type: str,
        query: str,
        limit: int = 50,
    ) -> List[Dict]:
        """Search for TikTok creators"""
        if search_type == 'hashtag':
            return await self._search_by_hashtag(query, limit)
        elif search_type == 'keyword':
            return await self._search_by_keyword(query, limit)
        else:
            raise ValueError(f"Unknown search type: {search_type}")

    async def _search_by_hashtag(
        self,
        hashtag: str,
        limit: int,
    ) -> List[Dict]:
        """Search for creators using specific hashtag"""
        hashtag = hashtag.lstrip('#')

        async with httpx.AsyncClient() as client:
            # Search videos by hashtag
            response = await client.post(
                f"{self.base_url}/research/video/query/",
                headers={
                    'Authorization': f'Bearer {self.access_token}',
                    'Content-Type': 'application/json',
                },
                json={
                    'query': {
                        'and': [
                            {'field_name': 'hashtag_name', 'operation': 'EQ', 'field_values': [hashtag]}
                        ]
                    },
                    'max_count': limit,
                }
            )

            if response.status_code != 200:
                return []

            data = response.json()
            videos = data.get('data', {}).get('videos', [])

            # Extract unique creators
            creator_usernames = set()
            for video in videos:
                if 'username' in video:
                    creator_usernames.add(video['username'])

            # Get full creator profiles
            profiles = []
            for username in list(creator_usernames)[:limit]:
                profile = await self._get_profile(username)
                if profile:
                    profiles.append(profile)

            return profiles

    async def _get_profile(self, username: str) -> Optional[Dict]:
        """Get TikTok creator profile"""
        async with httpx.AsyncClient() as client:
            # Get user info
            response = await client.get(
                f"{self.base_url}/research/user/info/",
                headers={'Authorization': f'Bearer {self.access_token}'},
                params={'username': username}
            )

            if response.status_code != 200:
                return None

            data = response.json().get('data', {}).get('user', {})

            # Get recent videos for engagement
            videos_response = await client.post(
                f"{self.base_url}/research/video/query/",
                headers={
                    'Authorization': f'Bearer {self.access_token}',
                    'Content-Type': 'application/json',
                },
                json={
                    'query': {
                        'and': [
                            {'field_name': 'username', 'operation': 'EQ', 'field_values': [username]}
                        ]
                    },
                    'max_count': 10,
                }
            )

            engagement_data = {'avg_likes': 0, 'avg_views': 0, 'avg_shares': 0, 'engagement_rate': 0}
            if videos_response.status_code == 200:
                videos = videos_response.json().get('data', {}).get('videos', [])
                if videos:
                    total_likes = sum(v.get('like_count', 0) for v in videos)
                    total_views = sum(v.get('view_count', 0) for v in videos)
                    total_shares = sum(v.get('share_count', 0) for v in videos)

                    avg_likes = total_likes / len(videos)
                    avg_views = total_views / len(videos)
                    avg_shares = total_shares / len(videos)

                    # Engagement rate = avg likes / avg views * 100
                    engagement_rate = (avg_likes / avg_views * 100) if avg_views > 0 else 0

                    engagement_data = {
                        'avg_likes': int(avg_likes),
                        'avg_views': int(avg_views),
                        'avg_shares': int(avg_shares),
                        'avg_comments': int(sum(v.get('comment_count', 0) for v in videos) / len(videos)),
                        'engagement_rate': round(engagement_rate, 2),
                    }

            return {
                'id': data.get('id'),
                'username': username,
                'display_name': data.get('display_name'),
                'bio': data.get('bio_description'),
                'profile_url': f"https://tiktok.com/@{username}",
                'avatar_url': data.get('avatar_url'),
                'followers_count': data.get('follower_count', 0),
                'following_count': data.get('following_count'),
                'posts_count': data.get('video_count'),
                **engagement_data,
            }

    async def _search_by_keyword(
        self,
        keyword: str,
        limit: int,
    ) -> List[Dict]:
        """Search by keyword in video descriptions"""
        # Similar to hashtag search but with keyword query
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/research/video/query/",
                headers={
                    'Authorization': f'Bearer {self.access_token}',
                    'Content-Type': 'application/json',
                },
                json={
                    'query': {
                        'and': [
                            {'field_name': 'keyword', 'operation': 'IN', 'field_values': [keyword]}
                        ]
                    },
                    'max_count': limit,
                }
            )

            if response.status_code != 200:
                return []

            videos = response.json().get('data', {}).get('videos', [])

            creator_usernames = set(v['username'] for v in videos if 'username' in v)

            profiles = []
            for username in list(creator_usernames)[:limit]:
                profile = await self._get_profile(username)
                if profile:
                    profiles.append(profile)

            return profiles
```

### 4. Profile Analyzer

```python
# services/influencer/analyzer.py

from typing import Dict, List, Optional, Any
from decimal import Decimal
from datetime import datetime, timedelta
import statistics
from openai import AsyncOpenAI

class ProfileAnalyzer:
    """Analyze influencer profiles for authenticity and quality"""

    def __init__(self, openai_client: AsyncOpenAI):
        self.openai = openai_client

    async def analyze(
        self,
        profile: Dict[str, Any],
        recent_posts: Optional[List[Dict]] = None,
    ) -> Dict[str, Any]:
        """
        Comprehensive profile analysis

        Returns:
            {
                'authenticity_score': float,  # 0-100
                'fake_follower_pct': float,   # 0-100
                'content_quality': float,      # 0-100
                'niche': List[str],
                'audience_demographics': Dict,
            }
        """
        # Calculate authenticity score
        authenticity = await self._calculate_authenticity(profile, recent_posts)

        # Detect fake followers
        fake_follower_pct = await self._detect_fake_followers(profile, recent_posts)

        # Analyze content quality
        content_quality = await self._analyze_content_quality(recent_posts) if recent_posts else 50.0

        # Categorize niche
        niche = await self._categorize_niche(profile, recent_posts)

        return {
            'authenticity_score': authenticity,
            'fake_follower_pct': fake_follower_pct,
            'content_quality': content_quality,
            'niche': niche,
        }

    async def _calculate_authenticity(
        self,
        profile: Dict,
        recent_posts: Optional[List[Dict]],
    ) -> float:
        """
        Calculate authenticity score based on multiple factors

        Factors:
        - Follower-to-engagement ratio
        - Follower growth patterns
        - Comment quality
        - Engagement consistency
        """
        score = 100.0

        # Check follower-to-engagement ratio
        followers = profile.get('followers_count', 0)
        engagement_rate = profile.get('engagement_rate', 0)

        # Typical engagement rates by follower count
        # Micro: 3-6%, Macro: 1-3%, Mega: <1%
        if followers < 10000:
            expected_min = 3.0
        elif followers < 100000:
            expected_min = 1.5
        else:
            expected_min = 0.5

        if engagement_rate < expected_min * 0.3:  # Less than 30% of expected
            score -= 30
        elif engagement_rate < expected_min * 0.5:  # Less than 50% of expected
            score -= 15

        # Check engagement consistency
        if recent_posts and len(recent_posts) >= 5:
            likes = [p.get('like_count', 0) for p in recent_posts]
            if likes:
                # High variance suggests bought engagement
                mean_likes = statistics.mean(likes)
                std_dev = statistics.stdev(likes) if len(likes) > 1 else 0
                coefficient_of_variation = (std_dev / mean_likes) if mean_likes > 0 else 0

                if coefficient_of_variation > 1.5:  # Very high variance
                    score -= 20
                elif coefficient_of_variation > 1.0:
                    score -= 10

        # Check follower-to-following ratio
        following = profile.get('following_count', 0)
        if following > 0 and followers > 0:
            ratio = followers / following
            if ratio < 0.1:  # Following way more than followers
                score -= 15

        return max(0.0, min(100.0, score))

    async def _detect_fake_followers(
        self,
        profile: Dict,
        recent_posts: Optional[List[Dict]],
    ) -> float:
        """
        Estimate percentage of fake followers

        Indicators:
        - Low engagement relative to followers
        - Sudden follower spikes
        - Comment spam patterns
        """
        fake_pct = 0.0

        followers = profile.get('followers_count', 0)
        engagement_rate = profile.get('engagement_rate', 0)

        # Engagement rate benchmark
        if followers > 0:
            if followers < 10000:
                expected_engagement = 4.0  # 4%
            elif followers < 100000:
                expected_engagement = 2.0  # 2%
            else:
                expected_engagement = 1.0  # 1%

            # Calculate fake follower estimate based on engagement gap
            engagement_gap = max(0, expected_engagement - engagement_rate)
            fake_pct = min(50.0, engagement_gap * 10)  # Max 50% from this factor

        # TODO: Add follower growth pattern analysis
        # This would require historical data

        return round(fake_pct, 2)

    async def _analyze_content_quality(
        self,
        recent_posts: List[Dict],
    ) -> float:
        """Use AI to analyze content quality"""
        if not recent_posts:
            return 50.0  # Neutral score

        # Sample up to 5 recent posts
        sample_posts = recent_posts[:5]

        # Extract captions/descriptions
        captions = [
            p.get('caption', '') or p.get('description', '')
            for p in sample_posts
        ]

        prompt = f"""Analyze the quality of these social media posts.
Rate the overall content quality from 0-100 based on:
- Creativity and originality
- Professionalism
- Value to audience
- Consistency in style

Posts:
{chr(10).join(f"{i+1}. {caption[:200]}" for i, caption in enumerate(captions) if caption)}

Respond with only a number from 0-100."""

        try:
            response = await self.openai.chat.completions.create(
                model="gpt-4",
                messages=[
                    {"role": "system", "content": "You are a social media content analyst."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.3,
            )

            score_text = response.choices[0].message.content.strip()
            score = float(score_text)
            return max(0.0, min(100.0, score))

        except Exception as e:
            print(f"Error analyzing content quality: {e}")
            return 50.0

    async def _categorize_niche(
        self,
        profile: Dict,
        recent_posts: Optional[List[Dict]],
    ) -> List[str]:
        """Use AI to categorize influencer niche"""
        bio = profile.get('bio', '')

        post_samples = []
        if recent_posts:
            post_samples = [
                p.get('caption', '') or p.get('description', '')
                for p in recent_posts[:5]
            ]

        prompt = f"""Categorize this influencer's niche based on their bio and recent posts.
Choose up to 3 most relevant categories from:
Fashion, Beauty, Fitness, Food, Travel, Tech, Gaming, Lifestyle, Business, Education,
Parenting, Pets, Health, Sports, Music, Art, Photography, Comedy, DIY, Home

Bio: {bio[:300]}

Recent posts:
{chr(10).join(f"- {post[:200]}" for post in post_samples if post)}

Respond with only the category names, comma-separated."""

        try:
            response = await self.openai.chat.completions.create(
                model="gpt-4",
                messages=[
                    {"role": "system", "content": "You are a social media categorization expert."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.3,
            )

            categories_text = response.choices[0].message.content.strip()
            categories = [c.strip() for c in categories_text.split(',')]
            return categories[:3]  # Max 3 categories

        except Exception as e:
            print(f"Error categorizing niche: {e}")
            return ['Lifestyle']  # Default category
```

### 5. Brand Matcher

```python
# services/influencer/brand_matcher.py

from typing import Dict, List, Any
from decimal import Decimal
from openai import AsyncOpenAI
from prisma import Prisma

class BrandMatcher:
    """Match influencers with brand based on affinity and alignment"""

    def __init__(
        self,
        prisma: Prisma,
        openai_client: AsyncOpenAI,
    ):
        self.prisma = prisma
        self.openai = openai_client

    async def calculate_match_score(
        self,
        store_id: str,
        influencer_id: str,
    ) -> Dict[str, Any]:
        """
        Calculate brand-influencer match score

        Returns:
            {
                'overall_score': float,      # 0-100
                'affinity_score': float,     # 0-100
                'audience_overlap': float,   # 0-100
                'content_style_match': float,# 0-100
                'value_alignment': float,    # 0-100
            }
        """
        # Get store info
        store = await self.prisma.store.find_unique(
            where={'id': store_id}
        )

        # Get influencer
        influencer = await self.prisma.influencer.find_unique(
            where={'id': influencer_id}
        )

        if not store or not influencer:
            raise ValueError("Store or influencer not found")

        # Calculate individual scores
        affinity_score = await self._calculate_affinity(store, influencer)
        audience_overlap = await self._calculate_audience_overlap(store, influencer)
        content_style = await self._calculate_content_style_match(store, influencer)
        value_alignment = await self._calculate_value_alignment(store, influencer)

        # Weighted overall score
        overall_score = (
            affinity_score * 0.4 +
            audience_overlap * 0.3 +
            content_style * 0.2 +
            value_alignment * 0.1
        )

        # Update influencer with brand affinity score
        await self.prisma.influencer.update(
            where={'id': influencer_id},
            data={'brandAffinityScore': Decimal(str(overall_score))}
        )

        return {
            'overall_score': round(overall_score, 2),
            'affinity_score': round(affinity_score, 2),
            'audience_overlap': round(audience_overlap, 2),
            'content_style_match': round(content_style, 2),
            'value_alignment': round(value_alignment, 2),
        }

    async def _calculate_affinity(
        self,
        store: Any,
        influencer: Any,
    ) -> float:
        """Use AI to determine brand affinity"""
        prompt = f"""Analyze the fit between this brand and influencer.
Rate the brand affinity from 0-100 based on:
- Niche alignment
- Brand relevance
- Audience match
- Content compatibility

Brand:
- Name: {store.name}
- Description: {store.description or 'N/A'}
- Niche: {store.niche or 'General'}

Influencer:
- Username: {influencer.username}
- Bio: {influencer.bio or 'N/A'}
- Niche: {', '.join(influencer.niche or [])}
- Followers: {influencer.followersCount}
- Engagement: {influencer.engagementRate}%

Respond with only a number from 0-100."""

        try:
            response = await self.openai.chat.completions.create(
                model="gpt-4",
                messages=[
                    {"role": "system", "content": "You are a brand-influencer matching expert."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.3,
            )

            score_text = response.choices[0].message.content.strip()
            score = float(score_text)
            return max(0.0, min(100.0, score))

        except Exception as e:
            print(f"Error calculating affinity: {e}")
            return 50.0

    async def _calculate_audience_overlap(
        self,
        store: Any,
        influencer: Any,
    ) -> float:
        """Calculate audience demographic overlap"""
        # This would compare store's customer demographics
        # with influencer's audience demographics
        # For now, return a placeholder

        score = 50.0  # Baseline

        # Check follower tier match
        # E.g., stores targeting young adults might prefer micro influencers
        if influencer.tier in ['micro', 'macro']:
            score += 20

        return min(100.0, score)

    async def _calculate_content_style_match(
        self,
        store: Any,
        influencer: Any,
    ) -> float:
        """Match content style with brand aesthetic"""
        # Placeholder - would analyze actual content
        return 50.0

    async def _calculate_value_alignment(
        self,
        store: Any,
        influencer: Any,
    ) -> float:
        """Check brand values alignment"""
        # Placeholder - would analyze brand values vs influencer messaging
        return 50.0

    async def find_best_matches(
        self,
        store_id: str,
        limit: int = 20,
        min_score: float = 60.0,
    ) -> List[Dict[str, Any]]:
        """Find top influencer matches for a store"""
        # Get all influencers for this store
        influencers = await self.prisma.influencer.find_many(
            where={'storeId': store_id},
            order_by={'engagementRate': 'desc'},
            take=100,  # Analyze top 100 by engagement
        )

        # Calculate match scores
        matches = []
        for influencer in influencers:
            try:
                scores = await self.calculate_match_score(store_id, influencer.id)
                if scores['overall_score'] >= min_score:
                    matches.append({
                        'influencer': influencer,
                        **scores,
                    })
            except Exception as e:
                print(f"Error scoring influencer {influencer.id}: {e}")
                continue

        # Sort by overall score
        matches.sort(key=lambda x: x['overall_score'], reverse=True)

        return matches[:limit]
```

---

## API Endpoints

```python
# routers/influencer.py

from fastapi import APIRouter, Depends, HTTPException, Query
from typing import List, Optional
from pydantic import BaseModel
from decimal import Decimal

from ..dependencies import get_current_user, get_prisma
from ..services.influencer.discovery import InfluencerDiscovery
from ..services.influencer.analyzer import ProfileAnalyzer
from ..services.influencer.brand_matcher import BrandMatcher

router = APIRouter(prefix="/api/influencer", tags=["influencer"])

# Request/Response Models

class DiscoveryRequest(BaseModel):
    platform: str  # instagram, tiktok, youtube, twitter
    search_type: str  # hashtag, keyword, location, similar
    query: str
    filters: Optional[dict] = None
    limit: int = 50

class InfluencerFilters(BaseModel):
    platform: Optional[str] = None
    tier: Optional[str] = None
    min_followers: Optional[int] = None
    max_followers: Optional[int] = None
    min_engagement: Optional[float] = None
    niches: Optional[List[str]] = None
    min_affinity_score: Optional[float] = None

# Endpoints

@router.post("/discover")
async def discover_influencers(
    request: DiscoveryRequest,
    current_user = Depends(get_current_user),
    discovery_service: InfluencerDiscovery = Depends(),
):
    """Discover influencers on specified platform"""
    try:
        results = await discovery_service.discover(
            store_id=current_user.store_id,
            platform=request.platform,
            search_type=request.search_type,
            query=request.query,
            filters=request.filters,
            limit=request.limit,
        )

        return {
            'success': True,
            'data': results,
        }
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/list")
async def list_influencers(
    platform: Optional[str] = None,
    tier: Optional[str] = None,
    min_followers: Optional[int] = None,
    max_followers: Optional[int] = None,
    min_engagement: Optional[float] = None,
    niches: Optional[str] = None,  # comma-separated
    min_affinity_score: Optional[float] = None,
    page: int = Query(1, ge=1),
    per_page: int = Query(20, ge=1, le=100),
    current_user = Depends(get_current_user),
    prisma = Depends(get_prisma),
):
    """List influencers with filters"""
    # Build where clause
    where = {'storeId': current_user.store_id}

    if platform:
        where['platform'] = platform

    if tier:
        where['tier'] = tier

    if min_followers:
        where['followersCount'] = {'gte': min_followers}

    if max_followers:
        if 'followersCount' in where:
            where['followersCount']['lte'] = max_followers
        else:
            where['followersCount'] = {'lte': max_followers}

    if min_engagement:
        where['engagementRate'] = {'gte': Decimal(str(min_engagement))}

    if niches:
        niche_list = [n.strip() for n in niches.split(',')]
        where['niche'] = {'hasSome': niche_list}

    if min_affinity_score:
        where['brandAffinityScore'] = {'gte': Decimal(str(min_affinity_score))}

    # Get total count
    total = await prisma.influencer.count(where=where)

    # Get page of results
    influencers = await prisma.influencer.find_many(
        where=where,
        order_by={'brandAffinityScore': 'desc'},
        skip=(page - 1) * per_page,
        take=per_page,
    )

    return {
        'success': True,
        'data': {
            'influencers': influencers,
            'pagination': {
                'page': page,
                'per_page': per_page,
                'total': total,
                'pages': (total + per_page - 1) // per_page,
            }
        }
    }

@router.get("/{influencer_id}")
async def get_influencer(
    influencer_id: str,
    current_user = Depends(get_current_user),
    prisma = Depends(get_prisma),
):
    """Get influencer details"""
    influencer = await prisma.influencer.find_first(
        where={
            'id': influencer_id,
            'storeId': current_user.store_id,
        },
        include={
            'metrics': {
                'order_by': {'recordedAt': 'desc'},
                'take': 30,  # Last 30 data points
            }
        }
    )

    if not influencer:
        raise HTTPException(status_code=404, detail="Influencer not found")

    return {
        'success': True,
        'data': influencer,
    }

@router.post("/{influencer_id}/analyze")
async def analyze_influencer(
    influencer_id: str,
    current_user = Depends(get_current_user),
    prisma = Depends(get_prisma),
    analyzer: ProfileAnalyzer = Depends(),
):
    """Re-analyze influencer profile"""
    influencer = await prisma.influencer.find_first(
        where={
            'id': influencer_id,
            'storeId': current_user.store_id,
        }
    )

    if not influencer:
        raise HTTPException(status_code=404, detail="Influencer not found")

    # Analyze profile
    profile_data = {
        'followers_count': influencer.followersCount,
        'engagement_rate': float(influencer.engagementRate),
        'bio': influencer.bio,
    }

    analysis = await analyzer.analyze(profile_data)

    # Update influencer
    updated = await prisma.influencer.update(
        where={'id': influencer_id},
        data={
            'authenticityScore': Decimal(str(analysis['authenticity_score'])),
            'fakeFollowerPct': Decimal(str(analysis['fake_follower_pct'])),
            'niche': analysis['niche'],
            'lastAnalyzedAt': datetime.utcnow(),
        }
    )

    return {
        'success': True,
        'data': {
            'influencer': updated,
            'analysis': analysis,
        }
    }

@router.post("/{influencer_id}/match")
async def calculate_brand_match(
    influencer_id: str,
    current_user = Depends(get_current_user),
    brand_matcher: BrandMatcher = Depends(),
):
    """Calculate brand-influencer match score"""
    try:
        scores = await brand_matcher.calculate_match_score(
            store_id=current_user.store_id,
            influencer_id=influencer_id,
        )

        return {
            'success': True,
            'data': scores,
        }
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/matches/best")
async def get_best_matches(
    limit: int = Query(20, ge=1, le=100),
    min_score: float = Query(60.0, ge=0, le=100),
    current_user = Depends(get_current_user),
    brand_matcher: BrandMatcher = Depends(),
):
    """Get best influencer matches for current store"""
    matches = await brand_matcher.find_best_matches(
        store_id=current_user.store_id,
        limit=limit,
        min_score=min_score,
    )

    return {
        'success': True,
        'data': matches,
    }

@router.get("/searches/history")
async def get_search_history(
    page: int = Query(1, ge=1),
    per_page: int = Query(20, ge=1, le=100),
    current_user = Depends(get_current_user),
    prisma = Depends(get_prisma),
):
    """Get discovery search history"""
    total = await prisma.influencersearch.count(
        where={'storeId': current_user.store_id}
    )

    searches = await prisma.influencersearch.find_many(
        where={'storeId': current_user.store_id},
        order_by={'createdAt': 'desc'},
        skip=(page - 1) * per_page,
        take=per_page,
    )

    return {
        'success': True,
        'data': {
            'searches': searches,
            'pagination': {
                'page': page,
                'per_page': per_page,
                'total': total,
                'pages': (total + per_page - 1) // per_page,
            }
        }
    }
```

---

## UI Components

### Influencer Discovery Dashboard

```typescript
// app/influencer/discover/page.tsx

'use client';

import { useState } from 'react';
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Select, SelectTrigger, SelectValue, SelectContent, SelectItem } from '@/components/ui/select';
import { Badge } from '@/components/ui/badge';
import { Search, Filter, Download } from 'lucide-react';

export default function InfluencerDiscoveryPage() {
    const [platform, setPlatform] = useState('instagram');
    const [searchType, setSearchType] = useState('hashtag');
    const [query, setQuery] = useState('');
    const [results, setResults] = useState<any[]>([]);
    const [loading, setLoading] = useState(false);

    const handleDiscover = async () => {
        setLoading(true);

        try {
            const response = await fetch('/api/influencer/discover', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    platform,
                    search_type: searchType,
                    query,
                    limit: 50,
                }),
            });

            const data = await response.json();

            if (data.success) {
                setResults(data.data.profiles || []);
            }
        } catch (error) {
            console.error('Discovery error:', error);
        } finally {
            setLoading(false);
        }
    };

    return (
        <div className="container mx-auto p-6">
            <h1 className="text-3xl font-bold mb-6">Influencer Discovery</h1>

            {/* Search Form */}
            <Card className="mb-6">
                <CardHeader>
                    <CardTitle>Discover Influencers</CardTitle>
                </CardHeader>
                <CardContent>
                    <div className="grid grid-cols-4 gap-4">
                        <Select value={platform} onValueChange={setPlatform}>
                            <SelectTrigger>
                                <SelectValue placeholder="Platform" />
                            </SelectTrigger>
                            <SelectContent>
                                <SelectItem value="instagram">Instagram</SelectItem>
                                <SelectItem value="tiktok">TikTok</SelectItem>
                                <SelectItem value="youtube">YouTube</SelectItem>
                                <SelectItem value="twitter">Twitter</SelectItem>
                            </SelectContent>
                        </Select>

                        <Select value={searchType} onValueChange={setSearchType}>
                            <SelectTrigger>
                                <SelectValue placeholder="Search Type" />
                            </SelectTrigger>
                            <SelectContent>
                                <SelectItem value="hashtag">Hashtag</SelectItem>
                                <SelectItem value="keyword">Keyword</SelectItem>
                                <SelectItem value="location">Location</SelectItem>
                            </SelectContent>
                        </Select>

                        <Input
                            placeholder="Search query..."
                            value={query}
                            onChange={(e) => setQuery(e.target.value)}
                            className="col-span-2"
                        />
                    </div>

                    <Button
                        onClick={handleDiscover}
                        disabled={loading || !query}
                        className="mt-4"
                    >
                        <Search className="mr-2 h-4 w-4" />
                        {loading ? 'Discovering...' : 'Discover Influencers'}
                    </Button>
                </CardContent>
            </Card>

            {/* Results */}
            {results.length > 0 && (
                <div>
                    <div className="flex justify-between items-center mb-4">
                        <h2 className="text-xl font-semibold">
                            Found {results.length} influencers
                        </h2>
                        <Button variant="outline" size="sm">
                            <Download className="mr-2 h-4 w-4" />
                            Export
                        </Button>
                    </div>

                    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
                        {results.map((influencer) => (
                            <InfluencerCard
                                key={influencer.id}
                                influencer={influencer}
                            />
                        ))}
                    </div>
                </div>
            )}
        </div>
    );
}

function InfluencerCard({ influencer }: { influencer: any }) {
    const getTierColor = (tier: string) => {
        switch (tier) {
            case 'mega': return 'bg-purple-500';
            case 'macro': return 'bg-blue-500';
            case 'micro': return 'bg-green-500';
            case 'nano': return 'bg-yellow-500';
            default: return 'bg-gray-500';
        }
    };

    return (
        <Card className="hover:shadow-lg transition-shadow cursor-pointer">
            <CardContent className="p-4">
                <div className="flex items-start gap-4">
                    <img
                        src={influencer.avatar_url || '/default-avatar.png'}
                        alt={influencer.username}
                        className="w-16 h-16 rounded-full"
                    />

                    <div className="flex-1">
                        <div className="flex items-center gap-2">
                            <h3 className="font-semibold">@{influencer.username}</h3>
                            <Badge className={getTierColor(influencer.tier)}>
                                {influencer.tier}
                            </Badge>
                        </div>

                        {influencer.display_name && (
                            <p className="text-sm text-gray-600">
                                {influencer.display_name}
                            </p>
                        )}

                        <div className="grid grid-cols-2 gap-2 mt-3 text-sm">
                            <div>
                                <span className="text-gray-500">Followers:</span>
                                <span className="ml-2 font-medium">
                                    {influencer.followers_count.toLocaleString()}
                                </span>
                            </div>

                            <div>
                                <span className="text-gray-500">Engagement:</span>
                                <span className="ml-2 font-medium">
                                    {influencer.engagement_rate}%
                                </span>
                            </div>
                        </div>

                        {influencer.niche && influencer.niche.length > 0 && (
                            <div className="flex flex-wrap gap-1 mt-2">
                                {influencer.niche.slice(0, 3).map((n: string) => (
                                    <Badge key={n} variant="outline" className="text-xs">
                                        {n}
                                    </Badge>
                                ))}
                            </div>
                        )}

                        <Button variant="outline" size="sm" className="w-full mt-3">
                            View Details
                        </Button>
                    </div>
                </div>
            </CardContent>
        </Card>
    );
}
```

### Influencer Database Page

```typescript
// app/influencer/database/page.tsx

'use client';

import { useState, useEffect } from 'react';
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Select, SelectTrigger, SelectValue, SelectContent, SelectItem } from '@/components/ui/select';
import { Badge } from '@/components/ui/badge';
import { Table, TableHeader, TableBody, TableRow, TableHead, TableCell } from '@/components/ui/table';
import { Filter, SortAsc, Star, Mail } from 'lucide-react';

export default function InfluencerDatabasePage() {
    const [influencers, setInfluencers] = useState<any[]>([]);
    const [filters, setFilters] = useState({
        platform: '',
        tier: '',
        min_followers: '',
        min_engagement: '',
    });
    const [pagination, setPagination] = useState({
        page: 1,
        per_page: 20,
        total: 0,
        pages: 0,
    });

    useEffect(() => {
        loadInfluencers();
    }, [filters, pagination.page]);

    const loadInfluencers = async () => {
        const params = new URLSearchParams({
            page: pagination.page.toString(),
            per_page: pagination.per_page.toString(),
            ...(filters.platform && { platform: filters.platform }),
            ...(filters.tier && { tier: filters.tier }),
            ...(filters.min_followers && { min_followers: filters.min_followers }),
            ...(filters.min_engagement && { min_engagement: filters.min_engagement }),
        });

        const response = await fetch(`/api/influencer/list?${params}`);
        const data = await response.json();

        if (data.success) {
            setInfluencers(data.data.influencers);
            setPagination(data.data.pagination);
        }
    };

    const handleSaveInfluencer = async (influencerId: string) => {
        // Implementation for saving/starring influencer
        console.log('Save influencer:', influencerId);
    };

    const handleContactInfluencer = async (influencerId: string) => {
        // Navigate to outreach page
        window.location.href = `/influencer/outreach?influencer=${influencerId}`;
    };

    return (
        <div className="container mx-auto p-6">
            <h1 className="text-3xl font-bold mb-6">Influencer Database</h1>

            {/* Filters */}
            <Card className="mb-6">
                <CardHeader>
                    <CardTitle className="flex items-center">
                        <Filter className="mr-2 h-5 w-5" />
                        Filters
                    </CardTitle>
                </CardHeader>
                <CardContent>
                    <div className="grid grid-cols-4 gap-4">
                        <Select
                            value={filters.platform}
                            onValueChange={(value) => setFilters({ ...filters, platform: value })}
                        >
                            <SelectTrigger>
                                <SelectValue placeholder="All Platforms" />
                            </SelectTrigger>
                            <SelectContent>
                                <SelectItem value="">All Platforms</SelectItem>
                                <SelectItem value="instagram">Instagram</SelectItem>
                                <SelectItem value="tiktok">TikTok</SelectItem>
                                <SelectItem value="youtube">YouTube</SelectItem>
                                <SelectItem value="twitter">Twitter</SelectItem>
                            </SelectContent>
                        </Select>

                        <Select
                            value={filters.tier}
                            onValueChange={(value) => setFilters({ ...filters, tier: value })}
                        >
                            <SelectTrigger>
                                <SelectValue placeholder="All Tiers" />
                            </SelectTrigger>
                            <SelectContent>
                                <SelectItem value="">All Tiers</SelectItem>
                                <SelectItem value="nano">Nano (1K-10K)</SelectItem>
                                <SelectItem value="micro">Micro (10K-100K)</SelectItem>
                                <SelectItem value="macro">Macro (100K-1M)</SelectItem>
                                <SelectItem value="mega">Mega (1M+)</SelectItem>
                            </SelectContent>
                        </Select>

                        <Input
                            type="number"
                            placeholder="Min Followers"
                            value={filters.min_followers}
                            onChange={(e) => setFilters({ ...filters, min_followers: e.target.value })}
                        />

                        <Input
                            type="number"
                            placeholder="Min Engagement %"
                            value={filters.min_engagement}
                            onChange={(e) => setFilters({ ...filters, min_engagement: e.target.value })}
                        />
                    </div>
                </CardContent>
            </Card>

            {/* Table */}
            <Card>
                <CardContent className="p-0">
                    <Table>
                        <TableHeader>
                            <TableRow>
                                <TableHead>Influencer</TableHead>
                                <TableHead>Platform</TableHead>
                                <TableHead>Tier</TableHead>
                                <TableHead>Followers</TableHead>
                                <TableHead>Engagement</TableHead>
                                <TableHead>Match Score</TableHead>
                                <TableHead>Actions</TableHead>
                            </TableRow>
                        </TableHeader>
                        <TableBody>
                            {influencers.map((influencer) => (
                                <TableRow key={influencer.id}>
                                    <TableCell>
                                        <div className="flex items-center gap-3">
                                            <img
                                                src={influencer.avatarUrl || '/default-avatar.png'}
                                                alt={influencer.username}
                                                className="w-10 h-10 rounded-full"
                                            />
                                            <div>
                                                <div className="font-medium">@{influencer.username}</div>
                                                {influencer.displayName && (
                                                    <div className="text-sm text-gray-500">
                                                        {influencer.displayName}
                                                    </div>
                                                )}
                                            </div>
                                        </div>
                                    </TableCell>
                                    <TableCell>
                                        <Badge variant="outline">{influencer.platform}</Badge>
                                    </TableCell>
                                    <TableCell>
                                        <Badge>{influencer.tier}</Badge>
                                    </TableCell>
                                    <TableCell>{influencer.followersCount.toLocaleString()}</TableCell>
                                    <TableCell>{influencer.engagementRate}%</TableCell>
                                    <TableCell>
                                        {influencer.brandAffinityScore ? (
                                            <span className="font-medium text-green-600">
                                                {influencer.brandAffinityScore}
                                            </span>
                                        ) : (
                                            <span className="text-gray-400">-</span>
                                        )}
                                    </TableCell>
                                    <TableCell>
                                        <div className="flex gap-2">
                                            <Button
                                                variant="ghost"
                                                size="sm"
                                                onClick={() => handleSaveInfluencer(influencer.id)}
                                            >
                                                <Star className="h-4 w-4" />
                                            </Button>
                                            <Button
                                                variant="outline"
                                                size="sm"
                                                onClick={() => handleContactInfluencer(influencer.id)}
                                            >
                                                <Mail className="h-4 w-4 mr-1" />
                                                Contact
                                            </Button>
                                        </div>
                                    </TableCell>
                                </TableRow>
                            ))}
                        </TableBody>
                    </Table>
                </CardContent>
            </Card>

            {/* Pagination */}
            <div className="flex justify-center gap-2 mt-6">
                <Button
                    variant="outline"
                    onClick={() => setPagination({ ...pagination, page: pagination.page - 1 })}
                    disabled={pagination.page === 1}
                >
                    Previous
                </Button>
                <span className="py-2 px-4">
                    Page {pagination.page} of {pagination.pages}
                </span>
                <Button
                    variant="outline"
                    onClick={() => setPagination({ ...pagination, page: pagination.page + 1 })}
                    disabled={pagination.page === pagination.pages}
                >
                    Next
                </Button>
            </div>
        </div>
    );
}
```

---

## Celery Workers

```python
# workers/influencer_discovery.py

from celery import shared_task
from datetime import datetime, timedelta
from prisma import Prisma

@shared_task
def update_influencer_metrics():
    """
    Daily task to refresh influencer metrics
    Runs at 2 AM daily
    """
    prisma = Prisma()
    prisma.connect()

    try:
        # Get all influencers that need updating (last analyzed > 24 hours ago)
        cutoff = datetime.utcnow() - timedelta(hours=24)

        influencers = prisma.influencer.find_many(
            where={
                'lastAnalyzedAt': {'lt': cutoff}
            },
            take=100,  # Process 100 per run
        )

        for influencer in influencers:
            # Re-fetch profile data
            # This would call the appropriate platform client
            # and update metrics

            # Record metric snapshot
            prisma.influencermetric.create(
                data={
                    'influencerId': influencer.id,
                    'followersCount': influencer.followersCount,
                    'engagementRate': influencer.engagementRate,
                    'avgLikes': influencer.avgLikes,
                    'avgComments': influencer.avgComments,
                    'avgViews': influencer.avgViews,
                }
            )

            # Update last analyzed timestamp
            prisma.influencer.update(
                where={'id': influencer.id},
                data={'lastAnalyzedAt': datetime.utcnow()}
            )

        print(f"Updated metrics for {len(influencers)} influencers")

    finally:
        prisma.disconnect()

@shared_task
def cleanup_old_searches():
    """
    Weekly cleanup of old search records
    Runs on Sundays at 3 AM
    """
    prisma = Prisma()
    prisma.connect()

    try:
        # Delete searches older than 90 days
        cutoff = datetime.utcnow() - timedelta(days=90)

        result = prisma.influencersearch.delete_many(
            where={'createdAt': {'lt': cutoff}}
        )

        print(f"Deleted {result} old search records")

    finally:
        prisma.disconnect()
```

---

## Definition of Done

### Technical Requirements
- [ ] Instagram discovery working with hashtag/location search
- [ ] TikTok discovery working with hashtag/keyword search
- [ ] YouTube discovery working with search/related channels
- [ ] Twitter discovery working with hashtag/keyword search
- [ ] Profile analyzer calculating engagement rates accurately
- [ ] Fake follower detection operational
- [ ] AI-powered niche categorization working
- [ ] Brand matcher calculating affinity scores
- [ ] Tier classification working for all platforms
- [ ] Database UI with search and filters functional
- [ ] Profile detail views complete
- [ ] API endpoints tested and documented

### Quality Checks
- [ ] Discovery returns relevant influencers
- [ ] Engagement calculations match platform formulas
- [ ] Authenticity scores correlate with manual checks
- [ ] Brand matching produces sensible recommendations
- [ ] UI responsive on mobile devices
- [ ] Error handling for API rate limits
- [ ] Caching implemented for repeated queries

### Documentation
- [ ] API endpoint documentation complete
- [ ] Platform-specific search limitations documented
- [ ] Engagement rate calculation formulas documented
- [ ] Tier thresholds documented

---

## Testing Checklist

### Unit Tests
- [ ] Discovery service tests for each platform
- [ ] Profile analyzer tests with known profiles
- [ ] Fake follower detection accuracy tests
- [ ] Brand matcher scoring tests
- [ ] Tier classification tests
- [ ] Filter application tests

### Integration Tests
- [ ] End-to-end discovery flow (search → save → analyze → match)
- [ ] API endpoint tests for all routes
- [ ] Database CRUD operations
- [ ] Rate limiting tests
- [ ] Error handling tests

### Platform Tests
- [ ] Instagram API integration (with valid access token)
- [ ] TikTok API integration
- [ ] YouTube API integration
- [ ] Twitter API integration
- [ ] Handle API errors gracefully

### UI Tests
- [ ] Discovery form submission
- [ ] Results display and pagination
- [ ] Filter combinations
- [ ] Profile detail modal
- [ ] Save/organize features
- [ ] Mobile responsiveness

### Performance Tests
- [ ] Discovery completes within 2 minutes
- [ ] Large result sets handled efficiently
- [ ] Database queries optimized
- [ ] Caching working correctly

---

## Success Metrics

### Discovery Performance
- **Target:** < 2 minutes per platform search
- **Target:** Find 30+ relevant influencers per search
- **Target:** 90%+ profile data completeness

### Analysis Accuracy
- **Target:** Engagement rate calculations within 5% of actual
- **Target:** Fake follower detection 80%+ accuracy
- **Target:** Niche categorization 85%+ accuracy

### Brand Matching
- **Target:** 70%+ user agreement with match scores
- **Target:** Top 10 matches include at least 7 good fits
- **Target:** Match calculation completes in < 5 seconds

### System Performance
- **Target:** API response time < 1 second
- **Target:** UI loads in < 2 seconds
- **Target:** Handle 100+ concurrent discoveries

### User Adoption
- **Target:** 80%+ of users run at least one discovery
- **Target:** Average 3+ discoveries per user per week
- **Target:** 50%+ of discovered influencers get contacted

---

## Next Sprint Preview

### Phase 4 - Sprint 5: Influencer Outreach

**Focus:** Build automated outreach system with personalized messaging and campaign management.

**Key Features:**
- AI-powered outreach email/DM templates
- Automated follow-up sequences
- Reply detection and categorization
- Negotiation workflow
- Campaign management
- Contract templates
- Payment tracking

**Prerequisites:**
- Influencer database populated
- Email/messaging integrations ready
- Contract templates prepared

**Expected Duration:** 2 weeks
