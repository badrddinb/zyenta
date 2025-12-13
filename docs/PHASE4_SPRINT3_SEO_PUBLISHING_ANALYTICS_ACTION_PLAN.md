# Phase 4 - Sprint 3: SEO Publishing & Analytics - Action Plan

## Sprint Overview

**Duration:** 2 weeks
**Team Size:** 3-4 developers (2 backend, 1 frontend, 1 DevOps)
**Sprint Goal:** Implement CMS integration for blog publishing, search engine submission, SEO analytics dashboard with Google Search Console integration, and comprehensive content performance tracking.

### Objectives
- [ ] Integrate with storefront CMS for automated publishing
- [ ] Implement Google/Bing indexing submission
- [ ] Build SEO analytics dashboard with Search Console data
- [ ] Create content performance tracking and optimization
- [ ] Implement automated publishing workflow with scheduling

### Prerequisites
- Phase 4 Sprints 1-2 completed
- Articles generated and optimized
- Storefront blog infrastructure ready
- Google Search Console access configured

---

## Day-by-Day Breakdown

### Days 1-4: CMS Integration & Publishing System

#### Tasks
1. **Implement CMS Publisher**
   ```python
   # app/agents/seo/cms_publisher.py
   from typing import Dict, Any, Optional
   from datetime import datetime
   import httpx
   from app.db.prisma import prisma

   class CMSPublisher:
       """Publish articles to storefront CMS"""

       def __init__(self, storefront_url: str, api_key: str):
           self.storefront_url = storefront_url
           self.api_key = api_key
           self.client = httpx.AsyncClient(
               base_url=storefront_url,
               headers={"Authorization": f"Bearer {api_key}"}
           )

       async def publish_article(
           self,
           article_id: str,
       ) -> Dict[str, Any]:
           """Publish article to CMS"""

           # Get article from database
           article = await prisma.blogarticle.find_unique(
               where={"id": article_id},
               include={"store": True}
           )

           if not article:
               raise ValueError(f"Article {article_id} not found")

           # Prepare CMS payload
           payload = {
               "title": article.title,
               "slug": article.slug,
               "content": article.content,
               "excerpt": article.excerpt,
               "meta_description": article.metaDescription,
               "status": "published",
               "published_at": datetime.utcnow().isoformat(),
               "author": article.store.name,
               "schema_markup": article.schemaMarkup,
               "reading_time": article.readingTime,
           }

           # Publish to CMS
           try:
               response = await self.client.post(
                   "/api/blog/posts",
                   json=payload,
                   timeout=30.0,
               )
               response.raise_for_status()
               data = response.json()

               # Update article with published URL
               published_url = data.get("url", f"{self.storefront_url}/blog/{article.slug}")

               await prisma.blogarticle.update(
                   where={"id": article_id},
                   data={
                       "status": "PUBLISHED",
                       "publishedDate": datetime.utcnow(),
                       "publishedUrl": published_url,
                   }
               )

               return {
                   "success": True,
                   "url": published_url,
                   "cms_id": data.get("id"),
                   "published_at": datetime.utcnow(),
               }

           except httpx.HTTPError as e:
               raise Exception(f"CMS publishing failed: {str(e)}")

       async def update_article(
           self,
           article_id: str,
           cms_post_id: str,
       ) -> Dict[str, Any]:
           """Update existing article in CMS"""

           article = await prisma.blogarticle.find_unique(
               where={"id": article_id}
           )

           if not article:
               raise ValueError(f"Article {article_id} not found")

           payload = {
               "title": article.title,
               "content": article.content,
               "excerpt": article.excerpt,
               "meta_description": article.metaDescription,
               "updated_at": datetime.utcnow().isoformat(),
           }

           response = await self.client.put(
               f"/api/blog/posts/{cms_post_id}",
               json=payload,
           )
           response.raise_for_status()

           return response.json()

       async def unpublish_article(
           self,
           article_id: str,
           cms_post_id: str,
       ):
           """Unpublish article from CMS"""

           response = await self.client.delete(
               f"/api/blog/posts/{cms_post_id}"
           )
           response.raise_for_status()

           await prisma.blogarticle.update(
               where={"id": article_id},
               data={
                   "status": "ARCHIVED",
                   "publishedUrl": None,
               }
           )

       async def generate_sitemap(
           self,
           store_id: str,
       ) -> str:
           """Generate XML sitemap for all published articles"""

           articles = await prisma.blogarticle.find_many(
               where={
                   "storeId": store_id,
                   "status": "PUBLISHED",
               },
               order_by={"publishedDate": "desc"}
           )

           sitemap_entries = []
           for article in articles:
               entry = f"""
               <url>
                   <loc>{article.publishedUrl}</loc>
                   <lastmod>{article.publishedDate.strftime('%Y-%m-%d')}</lastmod>
                   <changefreq>monthly</changefreq>
                   <priority>0.8</priority>
               </url>
               """
               sitemap_entries.append(entry)

           sitemap_xml = f"""<?xml version="1.0" encoding="UTF-8"?>
           <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
               {''.join(sitemap_entries)}
           </urlset>
           """

           # Upload sitemap to CMS
           response = await self.client.put(
               "/api/sitemap.xml",
               content=sitemap_xml,
               headers={"Content-Type": "application/xml"}
           )
           response.raise_for_status()

           return sitemap_xml
   ```

2. **Image Upload Handler**
   ```python
   # app/services/image_upload_service.py
   from typing import Optional
   import httpx
   from io import BytesIO
   from PIL import Image

   class ImageUploadService:
       """Handle image uploads to CMS/CDN"""

       def __init__(self, cdn_url: str, api_key: str):
           self.cdn_url = cdn_url
           self.api_key = api_key

       async def upload_image(
           self,
           image_url: str,
           alt_text: str,
           optimize: bool = True,
       ) -> Dict[str, str]:
           """Upload and optimize image"""

           # Download image
           async with httpx.AsyncClient() as client:
               response = await client.get(image_url)
               response.raise_for_status()
               image_data = response.content

           # Optimize if requested
           if optimize:
               image_data = await self._optimize_image(image_data)

           # Upload to CDN
           files = {
               'file': ('image.jpg', image_data, 'image/jpeg'),
               'alt': (None, alt_text),
           }

           async with httpx.AsyncClient() as client:
               response = await client.post(
                   f"{self.cdn_url}/upload",
                   files=files,
                   headers={"Authorization": f"Bearer {self.api_key}"}
               )
               response.raise_for_status()
               data = response.json()

           return {
               "url": data["url"],
               "cdn_url": data["cdn_url"],
               "width": data.get("width"),
               "height": data.get("height"),
           }

       async def _optimize_image(self, image_data: bytes) -> bytes:
           """Optimize image for web"""

           img = Image.open(BytesIO(image_data))

           # Resize if too large
           max_width = 1200
           if img.width > max_width:
               ratio = max_width / img.width
               new_size = (max_width, int(img.height * ratio))
               img = img.resize(new_size, Image.Resampling.LANCZOS)

           # Convert to RGB if needed
           if img.mode != 'RGB':
               img = img.convert('RGB')

           # Save optimized
           output = BytesIO()
           img.save(output, format='JPEG', quality=85, optimize=True)
           return output.getvalue()
   ```

3. **Publishing Workflow Service**
   ```python
   # app/services/publishing_workflow_service.py
   from datetime import datetime
   from app.agents.seo import CMSPublisher
   from app.db.prisma import prisma

   class PublishingWorkflowService:
       """Manage article publishing workflow"""

       async def schedule_publication(
           self,
           article_id: str,
           scheduled_date: datetime,
       ):
           """Schedule article for future publication"""

           await prisma.blogarticle.update(
               where={"id": article_id},
               data={
                   "status": "SCHEDULED",
                   "scheduledDate": scheduled_date,
               }
           )

       async def publish_scheduled_articles(self):
           """Publish all articles scheduled for now"""

           articles = await prisma.blogarticle.find_many(
               where={
                   "status": "SCHEDULED",
                   "scheduledDate": {
                       "lte": datetime.utcnow()
                   }
               }
           )

           for article in articles:
               try:
                   await self._publish_article_workflow(article.id)
               except Exception as e:
                   print(f"Failed to publish article {article.id}: {e}")

       async def _publish_article_workflow(self, article_id: str):
           """Execute complete publishing workflow"""

           article = await prisma.blogarticle.find_unique(
               where={"id": article_id},
               include={"store": True}
           )

           # 1. Publish to CMS
           publisher = CMSPublisher(
               storefront_url=article.store.storefrontUrl,
               api_key=article.store.cmsApiKey,
           )

           result = await publisher.publish_article(article_id)

           # 2. Submit to search engines (handled in next section)
           await self._submit_to_search_engines(result["url"], article.storeId)

           # 3. Update analytics tracking
           await self._initialize_analytics_tracking(article_id)

           return result

       async def _submit_to_search_engines(self, url: str, store_id: str):
           """Submit URL to search engines - implemented in next section"""
           pass

       async def _initialize_analytics_tracking(self, article_id: str):
           """Initialize performance tracking for article"""
           await prisma.blogarticle.update(
               where={"id": article_id},
               data={
                   "impressions": 0,
                   "clicks": 0,
                   "avgPosition": None,
               }
           )
   ```

4. **Publishing API Endpoints**
   ```python
   # app/api/seo/publishing.py
   from fastapi import APIRouter, Depends, BackgroundTasks
   from app.auth import get_current_user
   from app.services.publishing_workflow_service import PublishingWorkflowService
   from datetime import datetime

   router = APIRouter()

   @router.post("/publish/{article_id}")
   async def publish_article(
       article_id: str,
       background_tasks: BackgroundTasks,
       user = Depends(get_current_user),
   ):
       """Publish article immediately"""

       service = PublishingWorkflowService()

       # Publish in background
       background_tasks.add_task(
           service._publish_article_workflow,
           article_id
       )

       return {
           "message": "Publishing queued",
           "article_id": article_id,
       }

   @router.post("/schedule/{article_id}")
   async def schedule_article(
       article_id: str,
       scheduled_date: datetime,
       user = Depends(get_current_user),
   ):
       """Schedule article for future publication"""

       service = PublishingWorkflowService()
       await service.schedule_publication(article_id, scheduled_date)

       return {
           "message": "Article scheduled",
           "scheduled_date": scheduled_date,
       }

   @router.post("/unpublish/{article_id}")
   async def unpublish_article(
       article_id: str,
       cms_post_id: str,
       user = Depends(get_current_user),
   ):
       """Unpublish article from CMS"""

       from app.db.prisma import prisma

       article = await prisma.blogarticle.find_unique(
           where={"id": article_id},
           include={"store": True}
       )

       publisher = CMSPublisher(
           storefront_url=article.store.storefrontUrl,
           api_key=article.store.cmsApiKey,
       )

       await publisher.unpublish_article(article_id, cms_post_id)

       return {"message": "Article unpublished"}
   ```

#### Deliverables
- [ ] CMS publisher implemented
- [ ] Image upload service working
- [ ] Publishing workflow automated
- [ ] Sitemap generation functional

---

### Days 5-7: Search Engine Submission & Indexing

#### Tasks
1. **Google Indexing API Integration**
   ```python
   # app/services/search_indexing_service.py
   from typing import Dict, Any
   from google.oauth2 import service_account
   from googleapiclient.discovery import build
   import httpx

   class SearchIndexingService:
       """Submit URLs to search engines for indexing"""

       def __init__(self):
           self.google_credentials = self._load_google_credentials()

       def _load_google_credentials(self):
           """Load Google service account credentials"""
           return service_account.Credentials.from_service_account_file(
               'google-credentials.json',
               scopes=['https://www.googleapis.com/auth/indexing']
           )

       async def submit_to_google(
           self,
           url: str,
           action: str = "URL_UPDATED",
       ) -> Dict[str, Any]:
           """Submit URL to Google Indexing API"""

           service = build('indexing', 'v3', credentials=self.google_credentials)

           request_body = {
               "url": url,
               "type": action,  # URL_UPDATED or URL_DELETED
           }

           try:
               response = service.urlNotifications().publish(
                   body=request_body
               ).execute()

               return {
                   "success": True,
                   "response": response,
                   "submitted_at": datetime.utcnow(),
               }

           except Exception as e:
               print(f"Google indexing failed: {e}")
               return {
                   "success": False,
                   "error": str(e),
               }

       async def submit_to_bing(
           self,
           url: str,
           api_key: str,
       ) -> Dict[str, Any]:
           """Submit URL to Bing Webmaster Tools"""

           async with httpx.AsyncClient() as client:
               response = await client.post(
                   "https://ssl.bing.com/webmaster/api.svc/json/SubmitUrlbatch",
                   params={"apikey": api_key},
                   json={
                       "siteUrl": url,
                       "urlList": [url],
                   }
               )

               if response.status_code == 200:
                   return {
                       "success": True,
                       "response": response.json(),
                   }
               else:
                   return {
                       "success": False,
                       "error": response.text,
                   }

       async def submit_sitemap(
           self,
           sitemap_url: str,
           store_id: str,
       ):
           """Submit sitemap to search engines"""

           from app.db.prisma import prisma

           store = await prisma.store.find_unique(
               where={"id": store_id}
           )

           # Submit to Google
           if store.googleSearchConsoleProperty:
               await self._submit_sitemap_to_google(
                   sitemap_url,
                   store.googleSearchConsoleProperty,
               )

           # Submit to Bing
           if store.bingWebmasterApiKey:
               await self._submit_sitemap_to_bing(
                   sitemap_url,
                   store.bingWebmasterApiKey,
               )

       async def _submit_sitemap_to_google(
           self,
           sitemap_url: str,
           property_url: str,
       ):
           """Submit sitemap to Google Search Console"""

           service = build(
               'searchconsole',
               'v1',
               credentials=self.google_credentials
           )

           service.sitemaps().submit(
               siteUrl=property_url,
               feedpath=sitemap_url,
           ).execute()

       async def _submit_sitemap_to_bing(
           self,
           sitemap_url: str,
           api_key: str,
       ):
           """Submit sitemap to Bing Webmaster Tools"""

           async with httpx.AsyncClient() as client:
               await client.post(
                   "https://ssl.bing.com/webmaster/api.svc/json/SubmitSitemap",
                   params={"apikey": api_key},
                   json={"sitemapUrl": sitemap_url}
               )
   ```

2. **Robots.txt Manager**
   ```python
   # app/services/robots_txt_service.py

   class RobotsTxtService:
       """Manage robots.txt file"""

       def generate_robots_txt(
           self,
           sitemap_url: str,
           disallow_paths: List[str] = None,
       ) -> str:
           """Generate robots.txt content"""

           disallow_paths = disallow_paths or [
               "/admin",
               "/api",
               "/checkout",
           ]

           disallow_rules = '\n'.join([f"Disallow: {path}" for path in disallow_paths])

           robots_txt = f"""User-agent: *
{disallow_rules}

Sitemap: {sitemap_url}

# Allow search engines to crawl blog
Allow: /blog/
"""

           return robots_txt

       async def upload_robots_txt(
           self,
           content: str,
           storefront_url: str,
           api_key: str,
       ):
           """Upload robots.txt to storefront"""

           async with httpx.AsyncClient() as client:
               response = await client.put(
                   f"{storefront_url}/robots.txt",
                   content=content,
                   headers={
                       "Authorization": f"Bearer {api_key}",
                       "Content-Type": "text/plain",
                   }
               )
               response.raise_for_status()
   ```

#### Deliverables
- [ ] Google Indexing API integrated
- [ ] Bing submission working
- [ ] Sitemap submission automated
- [ ] Robots.txt management functional

---

### Days 8-11: SEO Analytics Dashboard

#### Tasks
1. **Google Search Console Integration**
   ```python
   # app/services/search_console_service.py
   from google.oauth2 import service_account
   from googleapiclient.discovery import build
   from datetime import datetime, timedelta
   from typing import List, Dict, Any

   class SearchConsoleService:
       """Fetch data from Google Search Console"""

       def __init__(self):
           self.credentials = service_account.Credentials.from_service_account_file(
               'google-credentials.json',
               scopes=['https://www.googleapis.com/auth/webmasters.readonly']
           )
           self.service = build('searchconsole', 'v1', credentials=self.credentials)

       async def get_performance_data(
           self,
           property_url: str,
           start_date: datetime,
           end_date: datetime,
           dimensions: List[str] = None,
       ) -> List[Dict[str, Any]]:
           """Get performance data from Search Console"""

           dimensions = dimensions or ['page', 'query']

           request_body = {
               'startDate': start_date.strftime('%Y-%m-%d'),
               'endDate': end_date.strftime('%Y-%m-%d'),
               'dimensions': dimensions,
               'rowLimit': 25000,
           }

           response = self.service.searchanalytics().query(
               siteUrl=property_url,
               body=request_body
           ).execute()

           return response.get('rows', [])

       async def get_article_performance(
           self,
           property_url: str,
           article_url: str,
           days: int = 30,
       ) -> Dict[str, Any]:
           """Get performance data for specific article"""

           end_date = datetime.utcnow()
           start_date = end_date - timedelta(days=days)

           request_body = {
               'startDate': start_date.strftime('%Y-%m-%d'),
               'endDate': end_date.strftime('%Y-%m-%d'),
               'dimensions': ['query'],
               'dimensionFilterGroups': [{
                   'filters': [{
                       'dimension': 'page',
                       'operator': 'equals',
                       'expression': article_url,
                   }]
               }],
               'rowLimit': 100,
           }

           response = self.service.searchanalytics().query(
               siteUrl=property_url,
               body=request_body
           ).execute()

           rows = response.get('rows', [])

           # Aggregate data
           total_impressions = sum(row.get('impressions', 0) for row in rows)
           total_clicks = sum(row.get('clicks', 0) for row in rows)
           avg_position = sum(row.get('position', 0) for row in rows) / len(rows) if rows else 0
           ctr = (total_clicks / total_impressions * 100) if total_impressions > 0 else 0

           return {
               'impressions': total_impressions,
               'clicks': total_clicks,
               'ctr': ctr,
               'avg_position': avg_position,
               'top_queries': rows[:10],
           }

       async def get_keyword_rankings(
           self,
           property_url: str,
           keywords: List[str],
       ) -> Dict[str, float]:
           """Get current rankings for specific keywords"""

           end_date = datetime.utcnow()
           start_date = end_date - timedelta(days=7)

           rankings = {}

           for keyword in keywords:
               request_body = {
                   'startDate': start_date.strftime('%Y-%m-%d'),
                   'endDate': end_date.strftime('%Y-%m-%d'),
                   'dimensions': ['query'],
                   'dimensionFilterGroups': [{
                       'filters': [{
                           'dimension': 'query',
                           'operator': 'equals',
                           'expression': keyword,
                       }]
                   }],
               }

               response = self.service.searchanalytics().query(
                   siteUrl=property_url,
                   body=request_body
               ).execute()

               rows = response.get('rows', [])
               if rows:
                   rankings[keyword] = rows[0].get('position', 0)
               else:
                   rankings[keyword] = 0

           return rankings
   ```

2. **SEO Analytics Dashboard UI**
   ```typescript
   // app/dashboard/seo/analytics/page.tsx
   'use client';

   import { useState, useEffect } from 'react';
   import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
   import { LineChart, Line, BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';

   export default function SEOAnalyticsPage() {
     const [analytics, setAnalytics] = useState<any>(null);
     const [dateRange, setDateRange] = useState('30d');

     useEffect(() => {
       fetchAnalytics();
     }, [dateRange]);

     const fetchAnalytics = async () => {
       const response = await fetch(`/api/seo/analytics?range=${dateRange}`);
       const data = await response.json();
       setAnalytics(data);
     };

     if (!analytics) {
       return <div>Loading...</div>;
     }

     return (
       <div className="container mx-auto p-6">
         <h1 className="text-3xl font-bold mb-6">SEO Analytics</h1>

         {/* Overview Metrics */}
         <div className="grid grid-cols-4 gap-4 mb-6">
           <Card>
             <CardHeader>
               <CardTitle className="text-sm">Total Impressions</CardTitle>
             </CardHeader>
             <CardContent>
               <div className="text-3xl font-bold">{analytics.overview.total_impressions.toLocaleString()}</div>
               <div className="text-sm text-green-600">+{analytics.overview.impressions_change}%</div>
             </CardContent>
           </Card>

           <Card>
             <CardHeader>
               <CardTitle className="text-sm">Total Clicks</CardTitle>
             </CardHeader>
             <CardContent>
               <div className="text-3xl font-bold">{analytics.overview.total_clicks.toLocaleString()}</div>
               <div className="text-sm text-green-600">+{analytics.overview.clicks_change}%</div>
             </CardContent>
           </Card>

           <Card>
             <CardHeader>
               <CardTitle className="text-sm">Average CTR</CardTitle>
             </CardHeader>
             <CardContent>
               <div className="text-3xl font-bold">{analytics.overview.avg_ctr.toFixed(2)}%</div>
               <div className="text-sm text-gray-500">vs {analytics.overview.prev_ctr.toFixed(2)}%</div>
             </CardContent>
           </Card>

           <Card>
             <CardHeader>
               <CardTitle className="text-sm">Average Position</CardTitle>
             </CardHeader>
             <CardContent>
               <div className="text-3xl font-bold">{analytics.overview.avg_position.toFixed(1)}</div>
               <div className="text-sm text-green-600">↑ {analytics.overview.position_improvement.toFixed(1)}</div>
             </CardContent>
           </Card>
         </div>

         {/* Performance Trend */}
         <Card className="mb-6">
           <CardHeader>
             <CardTitle>Performance Trend</CardTitle>
           </CardHeader>
           <CardContent>
             <ResponsiveContainer width="100%" height={300}>
               <LineChart data={analytics.daily_performance}>
                 <CartesianGrid strokeDasharray="3 3" />
                 <XAxis dataKey="date" />
                 <YAxis yAxisId="left" />
                 <YAxis yAxisId="right" orientation="right" />
                 <Tooltip />
                 <Legend />
                 <Line yAxisId="left" type="monotone" dataKey="impressions" stroke="#8884d8" name="Impressions" />
                 <Line yAxisId="left" type="monotone" dataKey="clicks" stroke="#82ca9d" name="Clicks" />
                 <Line yAxisId="right" type="monotone" dataKey="position" stroke="#ffc658" name="Avg Position" />
               </LineChart>
             </ResponsiveContainer>
           </CardContent>
         </Card>

         {/* Top Performing Articles */}
         <Card className="mb-6">
           <CardHeader>
             <CardTitle>Top Performing Articles</CardTitle>
           </CardHeader>
           <CardContent>
             <div className="overflow-x-auto">
               <table className="min-w-full divide-y divide-gray-200">
                 <thead>
                   <tr>
                     <th>Article</th>
                     <th>Impressions</th>
                     <th>Clicks</th>
                     <th>CTR</th>
                     <th>Position</th>
                     <th>Actions</th>
                   </tr>
                 </thead>
                 <tbody>
                   {analytics.top_articles.map((article: any) => (
                     <tr key={article.id}>
                       <td className="font-medium">{article.title}</td>
                       <td>{article.impressions.toLocaleString()}</td>
                       <td>{article.clicks.toLocaleString()}</td>
                       <td>{article.ctr.toFixed(2)}%</td>
                       <td>{article.avg_position.toFixed(1)}</td>
                       <td>
                         <a href={`/seo/articles/${article.id}`} className="text-blue-600">View</a>
                       </td>
                     </tr>
                   ))}
                 </tbody>
               </table>
             </div>
           </CardContent>
         </Card>

         {/* Top Keywords */}
         <Card>
           <CardHeader>
             <CardTitle>Top Keywords</CardTitle>
           </CardHeader>
           <CardContent>
             <ResponsiveContainer width="100%" height={300}>
               <BarChart data={analytics.top_keywords}>
                 <CartesianGrid strokeDasharray="3 3" />
                 <XAxis dataKey="keyword" />
                 <YAxis />
                 <Tooltip />
                 <Legend />
                 <Bar dataKey="impressions" fill="#8884d8" />
                 <Bar dataKey="clicks" fill="#82ca9d" />
               </BarChart>
             </ResponsiveContainer>
           </CardContent>
         </Card>
       </div>
     );
   }
   ```

#### Deliverables
- [ ] Search Console integration working
- [ ] Analytics dashboard deployed
- [ ] Performance tracking functional
- [ ] Keyword ranking monitoring active

---

### Days 12-14: Performance Tracking & Optimization

#### Tasks
1. **Performance Tracker Service**
   ```python
   # app/services/seo_performance_tracker.py
   from datetime import datetime, timedelta
   from typing import List, Dict, Any
   from app.services.search_console_service import SearchConsoleService
   from app.db.prisma import prisma

   class SEOPerformanceTracker:
       """Track and analyze SEO performance"""

       def __init__(self):
           self.search_console = SearchConsoleService()

       async def update_article_metrics(
           self,
           article_id: str,
       ):
           """Update performance metrics for an article"""

           article = await prisma.blogarticle.find_unique(
               where={"id": article_id},
               include={"store": True}
           )

           if not article or not article.publishedUrl:
               return

           # Get performance data
           performance = await self.search_console.get_article_performance(
               property_url=article.store.googleSearchConsoleProperty,
               article_url=article.publishedUrl,
               days=30,
           )

           # Update database
           await prisma.blogarticle.update(
               where={"id": article_id},
               data={
                   "impressions": performance["impressions"],
                   "clicks": performance["clicks"],
                   "avgPosition": performance["avg_position"],
               }
           )

       async def identify_optimization_opportunities(
           self,
           store_id: str,
       ) -> List[Dict[str, Any]]:
           """Identify articles that need optimization"""

           articles = await prisma.blogarticle.find_many(
               where={
                   "storeId": store_id,
                   "status": "PUBLISHED",
               }
           )

           opportunities = []

           for article in articles:
               # High impressions but low CTR = optimize title/meta
               if article.impressions > 1000 and (article.clicks / article.impressions) < 0.02:
                   opportunities.append({
                       "article_id": article.id,
                       "title": article.title,
                       "type": "low_ctr",
                       "recommendation": "Optimize title and meta description",
                       "impressions": article.impressions,
                       "ctr": (article.clicks / article.impressions * 100),
                   })

               # Ranking on page 2 = optimize content
               if article.avgPosition and 11 <= article.avgPosition <= 20:
                   opportunities.append({
                       "article_id": article.id,
                       "title": article.title,
                       "type": "page_two",
                       "recommendation": "Add more depth to content and internal links",
                       "position": article.avgPosition,
                   })

               # Low impressions = keyword targeting issue
               if article.impressions < 100 and article.publishedDate < datetime.utcnow() - timedelta(days=30):
                   opportunities.append({
                       "article_id": article.id,
                       "title": article.title,
                       "type": "low_visibility",
                       "recommendation": "Review keyword targeting and improve content quality",
                       "impressions": article.impressions,
                   })

           return opportunities

       async def generate_performance_report(
           self,
           store_id: str,
           period_days: int = 30,
       ) -> Dict[str, Any]:
           """Generate comprehensive performance report"""

           articles = await prisma.blogarticle.find_many(
               where={
                   "storeId": store_id,
                   "status": "PUBLISHED",
               }
           )

           total_impressions = sum(a.impressions for a in articles)
           total_clicks = sum(a.clicks for a in articles)
           avg_ctr = (total_clicks / total_impressions * 100) if total_impressions > 0 else 0

           # Get top performers
           top_articles = sorted(articles, key=lambda a: a.clicks, reverse=True)[:10]

           # Get improvement opportunities
           opportunities = await self.identify_optimization_opportunities(store_id)

           return {
               "summary": {
                   "total_articles": len(articles),
                   "total_impressions": total_impressions,
                   "total_clicks": total_clicks,
                   "avg_ctr": avg_ctr,
                   "opportunities_count": len(opportunities),
               },
               "top_performers": [
                   {
                       "title": a.title,
                       "url": a.publishedUrl,
                       "clicks": a.clicks,
                       "impressions": a.impressions,
                   }
                   for a in top_articles
               ],
               "opportunities": opportunities,
           }
   ```

2. **Automated Performance Updates Worker**
   ```python
   # app/workers/seo_analytics_worker.py
   from celery import shared_task
   from celery.schedules import crontab
   from app.services.seo_performance_tracker import SEOPerformanceTracker
   from app.db.prisma import prisma

   @shared_task
   async def update_all_article_metrics():
       """Update metrics for all published articles"""

       tracker = SEOPerformanceTracker()

       articles = await prisma.blogarticle.find_many(
           where={"status": "PUBLISHED"}
       )

       for article in articles:
           try:
               await tracker.update_article_metrics(article.id)
           except Exception as e:
               print(f"Failed to update metrics for {article.id}: {e}")

   # Celery beat schedule
   beat_schedule = {
       'update-seo-metrics': {
           'task': 'app.workers.seo_analytics_worker.update_all_article_metrics',
           'schedule': crontab(hour=2, minute=0),  # Daily at 2 AM
       },
   }
   ```

3. **Analytics API Endpoints**
   ```python
   # app/api/seo/analytics.py
   from fastapi import APIRouter, Depends
   from app.auth import get_current_user
   from app.services.seo_performance_tracker import SEOPerformanceTracker
   from datetime import datetime, timedelta

   router = APIRouter()

   @router.get("/analytics")
   async def get_seo_analytics(
       range: str = "30d",
       user = Depends(get_current_user),
   ):
       """Get SEO analytics overview"""

       tracker = SEOPerformanceTracker()

       days = int(range.replace('d', ''))
       report = await tracker.generate_performance_report(
           store_id=user.storeId,
           period_days=days,
       )

       return report

   @router.get("/opportunities")
   async def get_optimization_opportunities(
       user = Depends(get_current_user),
   ):
       """Get optimization opportunities"""

       tracker = SEOPerformanceTracker()
       opportunities = await tracker.identify_optimization_opportunities(
           store_id=user.storeId
       )

       return {"opportunities": opportunities}

   @router.get("/article/{article_id}/performance")
   async def get_article_performance(
       article_id: str,
       user = Depends(get_current_user),
   ):
       """Get performance data for specific article"""

       from app.db.prisma import prisma

       article = await prisma.blogarticle.find_unique(
           where={"id": article_id}
       )

       return {
           "article": article,
           "performance": {
               "impressions": article.impressions,
               "clicks": article.clicks,
               "ctr": (article.clicks / article.impressions * 100) if article.impressions > 0 else 0,
               "avg_position": article.avgPosition,
           }
       }
   ```

#### Deliverables
- [ ] Performance tracking automated
- [ ] Optimization opportunities identified
- [ ] Performance reports generated
- [ ] Workers scheduled and running

---

## Definition of Done

### Technical Requirements
- [ ] CMS publishing working (<30s per article)
- [ ] Search engine submission automated
- [ ] Analytics dashboard functional with live data
- [ ] Performance tracking updating daily
- [ ] Optimization recommendations generated
- [ ] All API endpoints functional and documented

### Testing Requirements
- [ ] Unit tests for publishing service
- [ ] Integration tests for Search Console API
- [ ] UI tests for analytics dashboard
- [ ] Performance tests for batch updates
- [ ] End-to-end publishing workflow tested

### Documentation
- [ ] CMS integration guide
- [ ] Search Console setup instructions
- [ ] Analytics dashboard user guide
- [ ] Optimization recommendations guide

---

## Testing Checklist

### CMS Publishing
- [ ] Publish article to CMS successfully
- [ ] Update existing article in CMS
- [ ] Unpublish article from CMS
- [ ] Generate and upload sitemap
- [ ] Handle publishing errors gracefully
- [ ] Image upload and optimization working

### Search Engine Submission
- [ ] Submit URL to Google Indexing API
- [ ] Submit URL to Bing Webmaster Tools
- [ ] Submit sitemap to Google
- [ ] Submit sitemap to Bing
- [ ] Generate and upload robots.txt
- [ ] Handle API rate limits

### Analytics Dashboard
- [ ] Display overview metrics correctly
- [ ] Show performance trends
- [ ] List top performing articles
- [ ] Display top keywords
- [ ] Real-time data updates
- [ ] Export reports functionality

### Performance Tracking
- [ ] Update article metrics daily
- [ ] Identify low CTR opportunities
- [ ] Identify page 2 opportunities
- [ ] Identify low visibility opportunities
- [ ] Generate comprehensive reports
- [ ] Send optimization alerts

---

## Success Metrics

### Publishing Performance
- **Target:** < 30 seconds to publish article
- **Measurement:** API response time monitoring

### Indexing Speed
- **Target:** < 24 hours for Google indexing
- **Measurement:** Search Console data

### Analytics Accuracy
- **Target:** 95%+ match with Google Search Console
- **Measurement:** Data reconciliation

### User Engagement
- **Target:** 30% increase in organic traffic within 60 days
- **Measurement:** Analytics tracking

### Optimization Impact
- **Target:** 20% improvement in CTR for optimized articles
- **Measurement:** Before/after comparison

---

## Next Sprint Preview

### Phase 4 - Sprint 4: Influencer Discovery

**Focus Areas:**
1. **Multi-Platform Discovery**
   - Instagram influencer search
   - TikTok creator discovery
   - YouTube channel analysis
   - Twitter profile discovery

2. **AI Profile Analysis**
   - Engagement rate calculation
   - Audience demographics
   - Content quality assessment
   - Authenticity checking

3. **Brand Matching**
   - AI-powered affinity scoring
   - Audience overlap analysis
   - Value alignment assessment

4. **Influencer Database UI**
   - Search and filter interface
   - Profile detail views
   - Save and organize influencers

**Prerequisites:**
- Phase 4 Sprints 1-3 completed
- SEO content system fully operational
- OAuth system ready for platform connections
