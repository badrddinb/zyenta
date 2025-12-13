# Phase 3 - Sprint 3: Social Agent Advanced - Action Plan

## Sprint Overview

**Duration:** 2 weeks
**Team Size:** 3-4 developers (2 backend, 2 frontend)
**Sprint Goal:** Extend the Social Agent with Pinterest and Twitter support, build a comprehensive dashboard UI for content calendar management, and implement analytics aggregation across all platforms.

### Objectives
- [ ] Add Pinterest platform client for Pin publishing
- [ ] Add Twitter/X platform client for tweet publishing
- [ ] Build content calendar UI in dashboard
- [ ] Implement analytics aggregation system
- [ ] Create social media dashboard with insights
- [ ] Add post preview and editing functionality
- [ ] Implement bulk scheduling interface
- [ ] Build automated publishing worker

### Prerequisites
- Phase 3 Sprint 1 (OAuth Foundation) complete
- Phase 3 Sprint 2 (Social Agent Core) complete
- OAuth working for all platforms
- TikTok and Instagram clients operational
- Content calendar generation working

---

## Day-by-Day Breakdown

### Days 1-3: Pinterest & Twitter Platform Clients

#### Tasks
1. **Implement Pinterest Platform Client**
   ```python
   # app/agents/social/platforms/pinterest.py
   from typing import List, Dict, Any
   from datetime import datetime
   import httpx
   from .base import BasePlatformClient

   class PinterestClient(BasePlatformClient):
       """Pinterest API client for Pin publishing"""

       BASE_URL = "https://api.pinterest.com/v5"

       def __init__(self, tokens: Dict[str, Any]):
           self.access_token = tokens["access_token"]

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
           For Pinterest, we don't pre-upload media.
           We provide the URL directly when creating the Pin.
           Return the media URL as-is.
           """
           return media_url

       async def create_post(
           self,
           caption: str,
           hashtags: List[str],
           media_ids: List[str],
       ) -> Dict[str, Any]:
           """Create a Pin on Pinterest"""

           if not media_ids:
               raise ValueError("Pinterest requires at least one image")

           # Get default board (or first board)
           boards = await self._get_boards()
           if not boards:
               raise ValueError("No Pinterest boards found. Create a board first.")

           board_id = boards[0]["id"]

           # Format description with hashtags
           full_description = self.format_caption(caption, hashtags)

           # Create Pin
           response = await self._request(
               "POST",
               "/pins",
               json={
                   "board_id": board_id,
                   "title": caption[:100],  # Pinterest title limit
                   "description": full_description[:500],  # Description limit
                   "media_source": {
                       "source_type": "image_url",
                       "url": media_ids[0],  # First media URL
                   },
                   "link": caption.split("http")[-1] if "http" in caption else None,
               },
           )

           pin_id = response["id"]

           return {
               "id": pin_id,
               "status": "published",
               "url": f"https://www.pinterest.com/pin/{pin_id}/",
           }

       async def _get_boards(self) -> List[Dict[str, Any]]:
           """Get user's Pinterest boards"""

           response = await self._request(
               "GET",
               "/boards",
               params={"page_size": 25},
           )

           return response.get("items", [])

       async def get_insights(
           self,
           start_date: datetime,
           end_date: datetime,
       ) -> Dict[str, Any]:
           """Get Pinterest account analytics"""

           # Get user analytics
           response = await self._request(
               "GET",
               "/user_account/analytics",
               params={
                   "start_date": start_date.strftime("%Y-%m-%d"),
                   "end_date": end_date.strftime("%Y-%m-%d"),
                   "metric_types": "IMPRESSION,PIN_CLICK,OUTBOUND_CLICK,SAVE",
               },
           )

           summary = response.get("all", {})
           metrics = summary.get("daily_metrics", [])

           total_impressions = sum(m.get("data_status", {}).get("IMPRESSION", 0) for m in metrics)
           total_clicks = sum(m.get("data_status", {}).get("PIN_CLICK", 0) for m in metrics)
           total_saves = sum(m.get("data_status", {}).get("SAVE", 0) for m in metrics)

           return {
               "impressions": total_impressions,
               "clicks": total_clicks,
               "saves": total_saves,
               "engagement_rate": (total_clicks + total_saves) / total_impressions * 100 if total_impressions > 0 else 0,
           }

       async def get_post_analytics(self, post_id: str) -> Dict[str, Any]:
           """Get analytics for a specific Pin"""

           response = await self._request(
               "GET",
               f"/pins/{post_id}/analytics",
               params={
                   "metric_types": "IMPRESSION,PIN_CLICK,OUTBOUND_CLICK,SAVE",
               },
           )

           metrics = response.get("all", {}).get("lifetime_metrics", {})

           return {
               "impressions": metrics.get("IMPRESSION", 0),
               "clicks": metrics.get("PIN_CLICK", 0),
               "outbound_clicks": metrics.get("OUTBOUND_CLICK", 0),
               "saves": metrics.get("SAVE", 0),
           }
   ```

2. **Implement Twitter/X Platform Client**
   ```python
   # app/agents/social/platforms/twitter.py
   from typing import List, Dict, Any
   from datetime import datetime
   import httpx
   from .base import BasePlatformClient

   class TwitterClient(BasePlatformClient):
       """Twitter/X API client for tweet publishing"""

       BASE_URL = "https://api.twitter.com/2"

       def __init__(self, tokens: Dict[str, Any]):
           self.access_token = tokens["access_token"]

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
           Upload media to Twitter.
           Uses Twitter's media upload API (v1.1).
           """

           # Download media first
           async with httpx.AsyncClient() as client:
               media_response = await client.get(media_url)
               media_data = media_response.content

           # Upload to Twitter (using v1.1 API for media)
           upload_url = "https://upload.twitter.com/1.1/media/upload.json"

           headers = {
               "Authorization": f"Bearer {self.access_token}",
           }

           async with httpx.AsyncClient() as client:
               response = await client.post(
                   upload_url,
                   headers=headers,
                   files={"media": media_data},
               )
               response.raise_for_status()
               data = response.json()

           return data["media_id_string"]

       async def create_post(
           self,
           caption: str,
           hashtags: List[str],
           media_ids: List[str],
       ) -> Dict[str, Any]:
           """Create a tweet on Twitter"""

           # Format tweet with hashtags
           full_text = self.format_caption(caption, hashtags)

           # Ensure under 280 characters
           if len(full_text) > 280:
               # Truncate caption, keep hashtags if possible
               hashtag_str = ' '.join([f'#{tag}' for tag in hashtags[:2]])  # Keep max 2 hashtags
               available_length = 280 - len(hashtag_str) - 2  # -2 for spacing
               truncated_caption = caption[:available_length-3] + "..."
               full_text = f"{truncated_caption}\n\n{hashtag_str}"

           # Create tweet data
           tweet_data = {
               "text": full_text,
           }

           # Add media if present
           if media_ids:
               tweet_data["media"] = {
                   "media_ids": media_ids[:4]  # Twitter max 4 images
               }

           # Create tweet
           response = await self._request(
               "POST",
               "/tweets",
               json=tweet_data,
           )

           tweet_id = response["data"]["id"]

           return {
               "id": tweet_id,
               "status": "published",
               "url": f"https://twitter.com/i/web/status/{tweet_id}",
           }

       async def get_insights(
           self,
           start_date: datetime,
           end_date: datetime,
       ) -> Dict[str, Any]:
           """Get Twitter account metrics"""

           # Get authenticated user info
           user_response = await self._request(
               "GET",
               "/users/me",
               params={
                   "user.fields": "public_metrics",
               },
           )

           user_data = user_response.get("data", {})
           metrics = user_data.get("public_metrics", {})

           return {
               "followers": metrics.get("followers_count", 0),
               "following": metrics.get("following_count", 0),
               "tweets": metrics.get("tweet_count", 0),
           }

       async def get_post_analytics(self, post_id: str) -> Dict[str, Any]:
           """Get analytics for a specific tweet"""

           response = await self._request(
               "GET",
               f"/tweets/{post_id}",
               params={
                   "tweet.fields": "public_metrics,non_public_metrics,organic_metrics",
               },
           )

           tweet_data = response.get("data", {})
           public_metrics = tweet_data.get("public_metrics", {})
           organic_metrics = tweet_data.get("organic_metrics", {})

           return {
               "impressions": organic_metrics.get("impression_count", 0),
               "likes": public_metrics.get("like_count", 0),
               "retweets": public_metrics.get("retweet_count", 0),
               "replies": public_metrics.get("reply_count", 0),
               "url_clicks": organic_metrics.get("url_link_clicks", 0),
               "profile_clicks": organic_metrics.get("user_profile_clicks", 0),
           }
   ```

3. **Update Platform Factory**
   ```python
   # app/agents/social/platforms/__init__.py (update)
   from .pinterest import PinterestClient
   from .twitter import TwitterClient

   def get_platform_client(
       platform: str,
       tokens: Dict[str, Any],
   ) -> BasePlatformClient:
       """Factory function to get platform client"""

       clients = {
           "tiktok": TikTokClient,
           "meta": InstagramClient,
           "instagram": InstagramClient,
           "pinterest": PinterestClient,
           "twitter": TwitterClient,
       }

       client_class = clients.get(platform)
       if not client_class:
           raise ValueError(f"Unsupported platform: {platform}")

       return client_class(tokens)
   ```

4. **Test Platform Clients**
   ```python
   # tests/test_platform_clients.py
   import pytest
   from app.agents.social.platforms import get_platform_client

   @pytest.mark.asyncio
   async def test_pinterest_client():
       tokens = {"access_token": "test_token"}
       client = get_platform_client("pinterest", tokens)

       assert client is not None
       assert hasattr(client, 'create_post')

   @pytest.mark.asyncio
   async def test_twitter_client():
       tokens = {"access_token": "test_token"}
       client = get_platform_client("twitter", tokens)

       assert client is not None
       assert hasattr(client, 'create_post')

       # Test caption truncation
       long_caption = "A" * 300
       hashtags = ["test1", "test2", "test3"]
       formatted = client.format_caption(long_caption[:250], hashtags)
       assert len(formatted) <= 280
   ```

#### Deliverables
- [ ] Pinterest client implemented and tested
- [ ] Twitter client implemented and tested
- [ ] Both platforms registered in factory
- [ ] Media upload working for both platforms
- [ ] Post creation functional

---

### Days 4-7: Content Calendar UI

#### Tasks
1. **Create Content Calendar Page**
   ```typescript
   // apps/dashboard/app/dashboard/social/calendar/page.tsx
   'use client';

   import { useState, useEffect } from 'react';
   import { Calendar } from '@/components/ui/calendar';
   import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
   import { Button } from '@/components/ui/button';
   import { Badge } from '@/components/ui/badge';
   import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
   import { CalendarDays, Eye, Edit, Trash2, Send } from 'lucide-react';
   import { format, startOfMonth, endOfMonth, eachDayOfInterval } from 'date-fns';

   interface Post {
     id: string;
     platform: string;
     contentType: string;
     caption: string;
     hashtags: string[];
     mediaUrls: string[];
     scheduledTime: string;
     status: string;
   }

   interface ContentCalendar {
     id: string;
     startDate: string;
     endDate: string;
     totalPosts: number;
     posts: Post[];
   }

   export default function SocialCalendarPage() {
     const [calendar, setCalendar] = useState<ContentCalendar | null>(null);
     const [selectedDate, setSelectedDate] = useState<Date>(new Date());
     const [loading, setLoading] = useState(true);
     const [generating, setGenerating] = useState(false);
     const [view, setView] = useState<'calendar' | 'list'>('calendar');

     const storeId = 'store_123'; // Get from session/context

     useEffect(() => {
       loadCalendar();
     }, []);

     async function loadCalendar() {
       try {
         const response = await fetch(
           `http://localhost:8003/api/v1/social/calendar/${storeId}`
         );

         if (response.ok) {
           const data = await response.json();
           setCalendar(data);
         }
       } catch (error) {
         console.error('Failed to load calendar:', error);
       } finally {
         setLoading(false);
       }
     }

     async function generateCalendar() {
       setGenerating(true);
       try {
         const response = await fetch(
           `http://localhost:8003/api/v1/social/calendar/generate?store_id=${storeId}&days=30&posts_per_day=2`,
           { method: 'POST' }
         );

         if (response.ok) {
           await loadCalendar();
         }
       } catch (error) {
         console.error('Failed to generate calendar:', error);
       } finally {
         setGenerating(false);
       }
     }

     function getPostsForDate(date: Date): Post[] {
       if (!calendar) return [];

       return calendar.posts.filter((post) => {
         const postDate = new Date(post.scheduledTime);
         return (
           postDate.getDate() === date.getDate() &&
           postDate.getMonth() === date.getMonth() &&
           postDate.getFullYear() === date.getFullYear()
         );
       });
     }

     function getSelectedDatePosts(): Post[] {
       return getPostsForDate(selectedDate);
     }

     const platformIcons = {
       tiktok: '🎵',
       instagram: '📷',
       meta: '📱',
       pinterest: '📌',
       twitter: '🐦',
     };

     const statusColors = {
       DRAFT: 'secondary',
       SCHEDULED: 'default',
       PUBLISHING: 'warning',
       PUBLISHED: 'success',
       FAILED: 'destructive',
     };

     return (
       <div className="container mx-auto py-8">
         <div className="flex justify-between items-center mb-8">
           <div>
             <h1 className="text-3xl font-bold">Content Calendar</h1>
             <p className="text-muted-foreground">
               Manage your social media content schedule
             </p>
           </div>

           <div className="flex gap-2">
             <Button
               variant="outline"
               onClick={() => setView(view === 'calendar' ? 'list' : 'calendar')}
             >
               {view === 'calendar' ? 'List View' : 'Calendar View'}
             </Button>
             <Button onClick={generateCalendar} disabled={generating}>
               {generating ? 'Generating...' : 'Generate Calendar'}
             </Button>
           </div>
         </div>

         {!calendar && !loading && (
           <Card>
             <CardContent className="py-12 text-center">
               <CalendarDays className="w-16 h-16 mx-auto mb-4 text-muted-foreground" />
               <h3 className="text-xl font-semibold mb-2">No Content Calendar Yet</h3>
               <p className="text-muted-foreground mb-4">
                 Generate a 30-day content calendar to get started
               </p>
               <Button onClick={generateCalendar}>Generate Calendar</Button>
             </CardContent>
           </Card>
         )}

         {calendar && (
           <Tabs value={view} onValueChange={(v) => setView(v as 'calendar' | 'list')}>
             <TabsList>
               <TabsTrigger value="calendar">Calendar View</TabsTrigger>
               <TabsTrigger value="list">List View</TabsTrigger>
             </TabsList>

             <TabsContent value="calendar" className="mt-6">
               <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
                 {/* Calendar */}
                 <Card className="lg:col-span-2">
                   <CardHeader>
                     <CardTitle>
                       {format(selectedDate, 'MMMM yyyy')}
                     </CardTitle>
                     <CardDescription>
                       {calendar.totalPosts} posts scheduled
                     </CardDescription>
                   </CardHeader>
                   <CardContent>
                     <Calendar
                       mode="single"
                       selected={selectedDate}
                       onSelect={(date) => date && setSelectedDate(date)}
                       className="rounded-md border"
                       modifiers={{
                         hasPost: (date) => getPostsForDate(date).length > 0,
                       }}
                       modifiersClassNames={{
                         hasPost: 'bg-primary/10 font-bold',
                       }}
                     />
                   </CardContent>
                 </Card>

                 {/* Posts for selected date */}
                 <Card>
                   <CardHeader>
                     <CardTitle>
                       {format(selectedDate, 'MMM d, yyyy')}
                     </CardTitle>
                     <CardDescription>
                       {getSelectedDatePosts().length} posts
                     </CardDescription>
                   </CardHeader>
                   <CardContent>
                     <div className="space-y-4">
                       {getSelectedDatePosts().length === 0 && (
                         <p className="text-muted-foreground text-sm">
                           No posts scheduled for this date
                         </p>
                       )}

                       {getSelectedDatePosts().map((post) => (
                         <Card key={post.id}>
                           <CardContent className="pt-4">
                             <div className="flex items-start justify-between mb-2">
                               <div className="flex items-center gap-2">
                                 <span className="text-xl">
                                   {platformIcons[post.platform as keyof typeof platformIcons]}
                                 </span>
                                 <Badge variant={statusColors[post.status as keyof typeof statusColors]}>
                                   {post.status}
                                 </Badge>
                               </div>
                               <span className="text-sm text-muted-foreground">
                                 {format(new Date(post.scheduledTime), 'h:mm a')}
                               </span>
                             </div>

                             <p className="text-sm line-clamp-2 mb-3">
                               {post.caption}
                             </p>

                             <div className="flex gap-1">
                               <Button size="sm" variant="ghost">
                                 <Eye className="w-4 h-4" />
                               </Button>
                               <Button size="sm" variant="ghost">
                                 <Edit className="w-4 h-4" />
                               </Button>
                               <Button size="sm" variant="ghost">
                                 <Trash2 className="w-4 h-4" />
                               </Button>
                             </div>
                           </CardContent>
                         </Card>
                       ))}
                     </div>
                   </CardContent>
                 </Card>
               </div>
             </TabsContent>

             <TabsContent value="list" className="mt-6">
               <Card>
                 <CardHeader>
                   <CardTitle>All Scheduled Posts</CardTitle>
                   <CardDescription>
                     {calendar.totalPosts} total posts
                   </CardDescription>
                 </CardHeader>
                 <CardContent>
                   <div className="space-y-2">
                     {calendar.posts.slice(0, 50).map((post) => (
                       <div
                         key={post.id}
                         className="flex items-center justify-between p-4 border rounded-lg hover:bg-accent"
                       >
                         <div className="flex-1">
                           <div className="flex items-center gap-3 mb-1">
                             <span className="text-xl">
                               {platformIcons[post.platform as keyof typeof platformIcons]}
                             </span>
                             <span className="font-medium">
                               {format(new Date(post.scheduledTime), 'MMM d, yyyy • h:mm a')}
                             </span>
                             <Badge variant={statusColors[post.status as keyof typeof statusColors]} size="sm">
                               {post.status}
                             </Badge>
                           </div>
                           <p className="text-sm text-muted-foreground line-clamp-1">
                             {post.caption}
                           </p>
                         </div>

                         <div className="flex gap-1 ml-4">
                           <Button size="sm" variant="ghost">
                             <Eye className="w-4 h-4" />
                           </Button>
                           <Button size="sm" variant="ghost">
                             <Edit className="w-4 h-4" />
                           </Button>
                           <Button size="sm" variant="ghost">
                             <Send className="w-4 h-4" />
                           </Button>
                         </div>
                       </div>
                     ))}
                   </div>
                 </CardContent>
               </Card>
             </TabsContent>
           </Tabs>
         )}
       </div>
     );
   }
   ```

2. **Create Post Preview Modal**
   ```typescript
   // apps/dashboard/components/social/post-preview-modal.tsx
   'use client';

   import { Dialog, DialogContent, DialogDescription, DialogHeader, DialogTitle } from '@/components/ui/dialog';
   import { Badge } from '@/components/ui/badge';
   import { Button } from '@/components/ui/button';
   import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
   import { format } from 'date-fns';
   import Image from 'next/image';

   interface Post {
     id: string;
     platform: string;
     contentType: string;
     caption: string;
     hashtags: string[];
     mediaUrls: string[];
     scheduledTime: string;
     status: string;
   }

   interface PostPreviewModalProps {
     post: Post | null;
     open: boolean;
     onClose: () => void;
     onEdit?: (post: Post) => void;
     onDelete?: (postId: string) => void;
     onPublish?: (postId: string) => void;
   }

   export function PostPreviewModal({
     post,
     open,
     onClose,
     onEdit,
     onDelete,
     onPublish,
   }: PostPreviewModalProps) {
     if (!post) return null;

     const platformStyles = {
       tiktok: {
         name: 'TikTok',
         icon: '🎵',
         bgColor: 'bg-black',
         textColor: 'text-white',
       },
       instagram: {
         name: 'Instagram',
         icon: '📷',
         bgColor: 'bg-gradient-to-br from-purple-600 to-pink-500',
         textColor: 'text-white',
       },
       pinterest: {
         name: 'Pinterest',
         icon: '📌',
         bgColor: 'bg-red-600',
         textColor: 'text-white',
       },
       twitter: {
         name: 'Twitter',
         icon: '🐦',
         bgColor: 'bg-blue-400',
         textColor: 'text-white',
       },
     };

     const platform = platformStyles[post.platform as keyof typeof platformStyles] || {
       name: post.platform,
       icon: '📱',
       bgColor: 'bg-gray-800',
       textColor: 'text-white',
     };

     return (
       <Dialog open={open} onOpenChange={onClose}>
         <DialogContent className="max-w-4xl max-h-[90vh] overflow-y-auto">
           <DialogHeader>
             <DialogTitle className="flex items-center gap-2">
               <span className="text-2xl">{platform.icon}</span>
               {platform.name} Post Preview
             </DialogTitle>
             <DialogDescription>
               Scheduled for {format(new Date(post.scheduledTime), 'MMM d, yyyy • h:mm a')}
             </DialogDescription>
           </DialogHeader>

           <Tabs defaultValue="preview" className="mt-4">
             <TabsList>
               <TabsTrigger value="preview">Preview</TabsTrigger>
               <TabsTrigger value="details">Details</TabsTrigger>
             </TabsList>

             <TabsContent value="preview" className="mt-6">
               {/* Platform-specific preview */}
               <div className={`rounded-lg overflow-hidden ${platform.bgColor} ${platform.textColor} p-6`}>
                 {/* Media */}
                 {post.mediaUrls.length > 0 && (
                   <div className="mb-4 bg-white/10 rounded-lg overflow-hidden">
                     {post.contentType === 'video' ? (
                       <video
                         src={post.mediaUrls[0]}
                         controls
                         className="w-full max-h-96 object-contain"
                       />
                     ) : (
                       <div className="relative aspect-square">
                         <Image
                           src={post.mediaUrls[0]}
                           alt="Post media"
                           fill
                           className="object-cover"
                         />
                       </div>
                     )}
                   </div>
                 )}

                 {/* Caption */}
                 <div className="whitespace-pre-wrap mb-4">
                   {post.caption}
                 </div>

                 {/* Hashtags */}
                 {post.hashtags.length > 0 && (
                   <div className="flex flex-wrap gap-2">
                     {post.hashtags.map((tag) => (
                       <span key={tag} className="text-blue-300">
                         #{tag}
                       </span>
                     ))}
                   </div>
                 )}
               </div>
             </TabsContent>

             <TabsContent value="details" className="mt-6 space-y-4">
               <div>
                 <label className="text-sm font-medium">Status</label>
                 <div className="mt-1">
                   <Badge>{post.status}</Badge>
                 </div>
               </div>

               <div>
                 <label className="text-sm font-medium">Platform</label>
                 <p className="mt-1">{platform.name}</p>
               </div>

               <div>
                 <label className="text-sm font-medium">Content Type</label>
                 <p className="mt-1 capitalize">{post.contentType}</p>
               </div>

               <div>
                 <label className="text-sm font-medium">Scheduled Time</label>
                 <p className="mt-1">
                   {format(new Date(post.scheduledTime), 'EEEE, MMMM d, yyyy • h:mm a')}
                 </p>
               </div>

               <div>
                 <label className="text-sm font-medium">Caption</label>
                 <p className="mt-1 whitespace-pre-wrap border rounded-lg p-3 bg-muted">
                   {post.caption}
                 </p>
               </div>

               <div>
                 <label className="text-sm font-medium">Hashtags ({post.hashtags.length})</label>
                 <div className="mt-1 flex flex-wrap gap-2">
                   {post.hashtags.map((tag) => (
                     <Badge key={tag} variant="secondary">
                       #{tag}
                     </Badge>
                   ))}
                 </div>
               </div>

               <div>
                 <label className="text-sm font-medium">Media ({post.mediaUrls.length})</label>
                 <div className="mt-1 grid grid-cols-3 gap-2">
                   {post.mediaUrls.map((url, index) => (
                     <div key={index} className="relative aspect-square rounded-lg overflow-hidden">
                       <Image
                         src={url}
                         alt={`Media ${index + 1}`}
                         fill
                         className="object-cover"
                       />
                     </div>
                   ))}
                 </div>
               </div>
             </TabsContent>
           </Tabs>

           <div className="flex justify-end gap-2 mt-6">
             {post.status === 'DRAFT' && onPublish && (
               <Button onClick={() => onPublish(post.id)}>
                 Publish Now
               </Button>
             )}
             {onEdit && (
               <Button variant="outline" onClick={() => onEdit(post)}>
                 Edit
               </Button>
             )}
             {onDelete && (
               <Button variant="destructive" onClick={() => onDelete(post.id)}>
                 Delete
               </Button>
             )}
             <Button variant="ghost" onClick={onClose}>
               Close
             </Button>
           </div>
         </DialogContent>
       </Dialog>
     );
   }
   ```

3. **Add Navigation**
   ```typescript
   // apps/dashboard/components/layout/sidebar.tsx
   // Add to navigation items:
   {
     title: 'Social Calendar',
     href: '/dashboard/social/calendar',
     icon: CalendarDays,
   }
   ```

#### Deliverables
- [ ] Content calendar page with calendar view
- [ ] List view for all posts
- [ ] Post preview modal with platform-specific styling
- [ ] Date selection and filtering
- [ ] Calendar generation UI

---

### Days 8-10: Analytics Aggregation & Social Dashboard

#### Tasks
1. **Create Analytics Aggregation Service**
   ```python
   # app/services/analytics.py
   from typing import Dict, Any, List
   from datetime import datetime, timedelta
   from app.agents.social.platforms import get_platform_client
   from app.oauth.token_store import TokenStore

   class AnalyticsService:
       """Aggregate analytics across all social platforms"""

       def __init__(self):
           self.token_store = TokenStore()

       async def get_store_analytics(
           self,
           store_id: str,
           days: int = 30,
       ) -> Dict[str, Any]:
           """Get aggregated analytics for a store"""

           from app.services.database import prisma

           # Get all connected platforms
           connections = await prisma.platformconnection.find_many(
               where={
                   "storeId": store_id,
                   "status": "CONNECTED",
               }
           )

           end_date = datetime.utcnow()
           start_date = end_date - timedelta(days=days)

           platform_analytics = []
           total_reach = 0
           total_engagement = 0
           total_impressions = 0

           for connection in connections:
               try:
                   # Get tokens
                   tokens = await self.token_store.get_tokens(
                       connection.userId,
                       store_id,
                       connection.platform,
                   )

                   if not tokens:
                       continue

                   # Get platform client
                   client = get_platform_client(
                       connection.platform,
                       tokens["tokens"],
                   )

                   # Get insights
                   insights = await client.get_insights(
                       start_date=start_date,
                       end_date=end_date,
                   )

                   platform_analytics.append({
                       "platform": connection.platform,
                       "account_name": connection.accountName,
                       **insights,
                   })

                   # Aggregate totals
                   total_reach += insights.get("reach", 0) or insights.get("followers", 0)
                   total_impressions += insights.get("impressions", 0)
                   total_engagement += insights.get("engagement", 0) or insights.get("likes", 0)

               except Exception as e:
                   print(f"Error fetching analytics for {connection.platform}: {e}")
                   continue

           # Get post analytics
           posts = await prisma.socialpost.find_many(
               where={
                   "storeId": store_id,
                   "status": "PUBLISHED",
                   "publishedTime": {
                       "gte": start_date,
                   },
               },
               order_by={"publishedTime": "desc"},
           )

           post_analytics = []
           for post in posts[:20]:  # Top 20 recent posts
               if post.analytics:
                   post_analytics.append({
                       "id": post.id,
                       "platform": post.platform,
                       "published_time": post.publishedTime,
                       "caption": post.caption[:100] + "...",
                       **post.analytics,
                   })

           return {
               "overview": {
                   "total_reach": total_reach,
                   "total_impressions": total_impressions,
                   "total_engagement": total_engagement,
                   "engagement_rate": (total_engagement / total_impressions * 100) if total_impressions > 0 else 0,
                   "connected_platforms": len(connections),
               },
               "by_platform": platform_analytics,
               "recent_posts": post_analytics,
           }

       async def sync_post_analytics(self, post_id: str):
           """Sync analytics for a specific post"""

           from app.services.database import prisma

           post = await prisma.socialpost.find_unique(
               where={"id": post_id},
               include={"store": True},
           )

           if not post or post.status != "PUBLISHED" or not post.platformPostId:
               return

           # Get connection
           connection = await prisma.platformconnection.find_unique(
               where={
                   "storeId_platform": {
                       "storeId": post.storeId,
                       "platform": post.platform,
                   }
               }
           )

           if not connection:
               return

           # Get tokens
           tokens = await self.token_store.get_tokens(
               connection.userId,
               post.storeId,
               post.platform,
           )

           # Get platform client
           client = get_platform_client(post.platform, tokens["tokens"])

           # Fetch analytics
           analytics = await client.get_post_analytics(post.platformPostId)

           # Update post
           await prisma.socialpost.update(
               where={"id": post_id},
               data={"analytics": analytics},
           )

           return analytics
   ```

2. **Create Analytics API Endpoints**
   ```python
   # app/api/v1/analytics.py
   from fastapi import APIRouter, Query
   from app.services.analytics import AnalyticsService

   router = APIRouter(prefix="/analytics", tags=["analytics"])
   analytics_service = AnalyticsService()

   @router.get("/social/{store_id}")
   async def get_social_analytics(
       store_id: str,
       days: int = Query(30, ge=7, le=90),
   ):
       """Get social media analytics for a store"""

       analytics = await analytics_service.get_store_analytics(
           store_id=store_id,
           days=days,
       )

       return analytics

   @router.post("/sync/{post_id}")
   async def sync_post_analytics(post_id: str):
       """Sync analytics for a specific post"""

       analytics = await analytics_service.sync_post_analytics(post_id)

       return {
           "success": True,
           "analytics": analytics,
       }
   ```

3. **Include Analytics Router**
   ```python
   # app/api/v1/router.py
   from app.api.v1.analytics import router as analytics_router

   api_router.include_router(analytics_router)
   ```

4. **Create Social Dashboard**
   ```typescript
   // apps/dashboard/app/dashboard/social/page.tsx
   'use client';

   import { useState, useEffect } from 'react';
   import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
   import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
   import { TrendingUp, Users, Eye, Heart, Calendar } from 'lucide-react';
   import Link from 'next/link';
   import { Button } from '@/components/ui/button';
   import {
     Area,
     AreaChart,
     Bar,
     BarChart,
     ResponsiveContainer,
     Tooltip,
     XAxis,
     YAxis,
   } from 'recharts';

   interface Analytics {
     overview: {
       total_reach: number;
       total_impressions: number;
       total_engagement: number;
       engagement_rate: number;
       connected_platforms: number;
     };
     by_platform: Array<{
       platform: string;
       account_name: string;
       followers?: number;
       impressions?: number;
       engagement?: number;
     }>;
     recent_posts: Array<{
       id: string;
       platform: string;
       published_time: string;
       caption: string;
       likes?: number;
       comments?: number;
       shares?: number;
       views?: number;
     }>;
   }

   export default function SocialDashboardPage() {
     const [analytics, setAnalytics] = useState<Analytics | null>(null);
     const [loading, setLoading] = useState(true);
     const [timeRange, setTimeRange] = useState(30);

     const storeId = 'store_123'; // Get from context

     useEffect(() => {
       loadAnalytics();
     }, [timeRange]);

     async function loadAnalytics() {
       try {
         const response = await fetch(
           `http://localhost:8003/api/v1/analytics/social/${storeId}?days=${timeRange}`
         );
         const data = await response.json();
         setAnalytics(data);
       } catch (error) {
         console.error('Failed to load analytics:', error);
       } finally {
         setLoading(false);
       }
     }

     const platformIcons = {
       tiktok: '🎵',
       instagram: '📷',
       meta: '📱',
       pinterest: '📌',
       twitter: '🐦',
     };

     if (loading) {
       return <div>Loading...</div>;
     }

     return (
       <div className="container mx-auto py-8">
         <div className="flex justify-between items-center mb-8">
           <div>
             <h1 className="text-3xl font-bold">Social Media Dashboard</h1>
             <p className="text-muted-foreground">
               Track your social media performance
             </p>
           </div>

           <div className="flex gap-2">
             <select
               value={timeRange}
               onChange={(e) => setTimeRange(Number(e.target.value))}
               className="border rounded-lg px-3 py-2"
             >
               <option value={7}>Last 7 days</option>
               <option value={30}>Last 30 days</option>
               <option value={90}>Last 90 days</option>
             </select>

             <Link href="/dashboard/social/calendar">
               <Button>
                 <Calendar className="w-4 h-4 mr-2" />
                 View Calendar
               </Button>
             </Link>
           </div>
         </div>

         {/* Overview Stats */}
         <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-4 mb-8">
           <Card>
             <CardHeader className="flex flex-row items-center justify-between pb-2">
               <CardTitle className="text-sm font-medium">Total Reach</CardTitle>
               <Users className="h-4 w-4 text-muted-foreground" />
             </CardHeader>
             <CardContent>
               <div className="text-2xl font-bold">
                 {analytics?.overview.total_reach.toLocaleString()}
               </div>
               <p className="text-xs text-muted-foreground">
                 Across {analytics?.overview.connected_platforms} platforms
               </p>
             </CardContent>
           </Card>

           <Card>
             <CardHeader className="flex flex-row items-center justify-between pb-2">
               <CardTitle className="text-sm font-medium">Impressions</CardTitle>
               <Eye className="h-4 w-4 text-muted-foreground" />
             </CardHeader>
             <CardContent>
               <div className="text-2xl font-bold">
                 {analytics?.overview.total_impressions.toLocaleString()}
               </div>
               <p className="text-xs text-muted-foreground">
                 Total views
               </p>
             </CardContent>
           </Card>

           <Card>
             <CardHeader className="flex flex-row items-center justify-between pb-2">
               <CardTitle className="text-sm font-medium">Engagement</CardTitle>
               <Heart className="h-4 w-4 text-muted-foreground" />
             </CardHeader>
             <CardContent>
               <div className="text-2xl font-bold">
                 {analytics?.overview.total_engagement.toLocaleString()}
               </div>
               <p className="text-xs text-muted-foreground">
                 Likes, comments, shares
               </p>
             </CardContent>
           </Card>

           <Card>
             <CardHeader className="flex flex-row items-center justify-between pb-2">
               <CardTitle className="text-sm font-medium">Engagement Rate</CardTitle>
               <TrendingUp className="h-4 w-4 text-muted-foreground" />
             </CardHeader>
             <CardContent>
               <div className="text-2xl font-bold">
                 {analytics?.overview.engagement_rate.toFixed(2)}%
               </div>
               <p className="text-xs text-muted-foreground">
                 Average across platforms
               </p>
             </CardContent>
           </Card>
         </div>

         <div className="grid gap-6 md:grid-cols-2 mb-8">
           {/* Platform Performance */}
           <Card>
             <CardHeader>
               <CardTitle>Platform Performance</CardTitle>
               <CardDescription>Metrics by platform</CardDescription>
             </CardHeader>
             <CardContent>
               <div className="space-y-4">
                 {analytics?.by_platform.map((platform) => (
                   <div key={platform.platform} className="flex items-center justify-between">
                     <div className="flex items-center gap-3">
                       <span className="text-2xl">
                         {platformIcons[platform.platform as keyof typeof platformIcons]}
                       </span>
                       <div>
                         <p className="font-medium capitalize">{platform.platform}</p>
                         <p className="text-sm text-muted-foreground">
                           {platform.account_name}
                         </p>
                       </div>
                     </div>
                     <div className="text-right">
                       <p className="font-semibold">
                         {(platform.followers || 0).toLocaleString()}
                       </p>
                       <p className="text-sm text-muted-foreground">followers</p>
                     </div>
                   </div>
                 ))}
               </div>
             </CardContent>
           </Card>

           {/* Top Posts */}
           <Card>
             <CardHeader>
               <CardTitle>Top Posts</CardTitle>
               <CardDescription>Best performing content</CardDescription>
             </CardHeader>
             <CardContent>
               <div className="space-y-4">
                 {analytics?.recent_posts.slice(0, 5).map((post) => {
                   const totalEngagement = (post.likes || 0) + (post.comments || 0) + (post.shares || 0);

                   return (
                     <div key={post.id} className="flex items-start gap-3">
                       <span className="text-xl">
                         {platformIcons[post.platform as keyof typeof platformIcons]}
                       </span>
                       <div className="flex-1">
                         <p className="text-sm line-clamp-1">{post.caption}</p>
                         <div className="flex gap-4 mt-1 text-xs text-muted-foreground">
                           {post.views && <span>{post.views.toLocaleString()} views</span>}
                           <span>{totalEngagement.toLocaleString()} engagement</span>
                         </div>
                       </div>
                     </div>
                   );
                 })}
               </div>
             </CardContent>
           </Card>
         </div>
       </div>
     );
   }
   ```

#### Deliverables
- [ ] Analytics aggregation service implemented
- [ ] Analytics API endpoints working
- [ ] Social dashboard with overview metrics
- [ ] Platform performance comparison
- [ ] Top posts display

---

### Days 11-12: Publishing Worker & Bulk Actions

#### Tasks
1. **Create Publishing Worker**
   ```python
   # app/workers/posting_tasks.py
   from celery import Celery
   from datetime import datetime
   from app.agents.social.platforms import get_platform_client
   from app.oauth.token_store import TokenStore
   from app.services.database import prisma

   celery_app = Celery('growth-engine-workers')
   token_store = TokenStore()

   @celery_app.task
   async def publish_scheduled_post(post_id: str):
       """Publish a scheduled social media post"""

       # Get post from database
       post = await prisma.socialpost.find_unique(
           where={"id": post_id}
       )

       if not post:
           return {"error": "Post not found"}

       if post.status != "SCHEDULED":
           return {"error": f"Post is {post.status}, not scheduled"}

       # Update status to publishing
       await prisma.socialpost.update(
           where={"id": post_id},
           data={"status": "PUBLISHING"},
       )

       try:
           # Get connection
           connection = await prisma.platformconnection.find_unique(
               where={
                   "storeId_platform": {
                       "storeId": post.storeId,
                       "platform": post.platform,
                   }
               }
           )

           if not connection:
               raise ValueError("Platform not connected")

           # Get tokens
           tokens = await token_store.get_tokens(
               connection.userId,
               post.storeId,
               post.platform,
           )

           # Get platform client
           client = get_platform_client(post.platform, tokens["tokens"])

           # Upload media
           media_ids = []
           for media_url in post.mediaUrls:
               media_id = await client.upload_media(media_url)
               media_ids.append(media_id)

           # Create post
           result = await client.create_post(
               caption=post.caption,
               hashtags=post.hashtags,
               media_ids=media_ids,
           )

           # Update post as published
           await prisma.socialpost.update(
               where={"id": post_id},
               data={
                   "status": "PUBLISHED",
                   "publishedTime": datetime.utcnow(),
                   "platformPostId": result["id"],
                   "platformUrl": result.get("url"),
               },
           )

           return {
               "success": True,
               "platform_post_id": result["id"],
               "url": result.get("url"),
           }

       except Exception as e:
           # Update post as failed
           await prisma.socialpost.update(
               where={"id": post_id},
               data={
                   "status": "FAILED",
                   "error": str(e),
               },
           )

           return {"error": str(e)}

   @celery_app.task
   async def check_and_publish_due_posts():
       """Check for posts that are due to be published"""

       now = datetime.utcnow()

       # Find scheduled posts that are due
       posts = await prisma.socialpost.find_many(
           where={
               "status": "SCHEDULED",
               "scheduledTime": {
                   "lte": now,
               },
           },
           take=50,  # Process max 50 at a time
       )

       results = []
       for post in posts:
           result = await publish_scheduled_post.apply_async(args=[post.id])
           results.append(result)

       return {
           "processed": len(results),
           "results": results,
       }
   ```

2. **Setup Celery Beat for Scheduled Publishing**
   ```python
   # app/workers/celery_app.py
   from celery import Celery
   from celery.schedules import crontab
   from app.config import settings

   celery_app = Celery(
       'growth-engine-workers',
       broker=settings.REDIS_URL,
       backend=settings.REDIS_URL,
   )

   celery_app.conf.beat_schedule = {
       'check-scheduled-posts': {
           'task': 'app.workers.posting_tasks.check_and_publish_due_posts',
           'schedule': crontab(minute='*/5'),  # Every 5 minutes
       },
   }

   celery_app.conf.timezone = 'UTC'
   ```

3. **Add Bulk Actions API**
   ```python
   # app/api/v1/social.py (add)
   from typing import List
   from app.workers.posting_tasks import publish_scheduled_post

   @router.post("/posts/bulk/publish")
   async def bulk_publish_posts(
       post_ids: List[str],
   ):
       """Publish multiple posts at once"""

       results = []
       for post_id in post_ids:
           try:
               result = await publish_scheduled_post.apply_async(args=[post_id])
               results.append({
                   "post_id": post_id,
                   "status": "queued",
               })
           except Exception as e:
               results.append({
                   "post_id": post_id,
                   "status": "error",
                   "error": str(e),
               })

       return {
           "success": True,
           "results": results,
       }

   @router.delete("/posts/bulk/delete")
   async def bulk_delete_posts(
       post_ids: List[str],
   ):
       """Delete multiple posts at once"""

       from app.services.database import prisma

       result = await prisma.socialpost.delete_many(
           where={
               "id": {"in": post_ids},
               "status": {"in": ["DRAFT", "SCHEDULED", "FAILED"]},
           }
       )

       return {
           "success": True,
           "deleted": result,
       }

   @router.patch("/posts/bulk/reschedule")
   async def bulk_reschedule_posts(
       post_ids: List[str],
       offset_hours: int,
   ):
       """Reschedule multiple posts by offset"""

       from app.services.database import prisma
       from datetime import timedelta

       posts = await prisma.socialpost.find_many(
           where={"id": {"in": post_ids}}
       )

       for post in posts:
           if post.scheduledTime:
               new_time = post.scheduledTime + timedelta(hours=offset_hours)
               await prisma.socialpost.update(
                   where={"id": post.id},
                   data={"scheduledTime": new_time},
               )

       return {
           "success": True,
           "rescheduled": len(posts),
       }
   ```

#### Deliverables
- [ ] Celery worker for publishing posts
- [ ] Scheduled task checking for due posts
- [ ] Bulk publish API endpoint
- [ ] Bulk delete API endpoint
- [ ] Bulk reschedule API endpoint

---

### Days 13-14: Testing, Documentation & Integration

#### Tasks
1. **Write Integration Tests**
   ```python
   # tests/test_social_integration.py
   import pytest
   from datetime import datetime, timedelta
   from app.agents.social.agent import SocialAgent

   @pytest.mark.asyncio
   async def test_full_calendar_generation():
       """Test complete calendar generation flow"""
       agent = SocialAgent()

       calendar = await agent.generate_content_calendar(
           store_id="test_store_123",
           days=7,
           posts_per_day=2,
       )

       assert calendar is not None
       assert len(calendar.posts) == 14
       assert all(post.scheduled_time is not None for post in calendar.posts)
       assert all(post.caption for post in calendar.posts)
       assert all(post.hashtags for post in calendar.posts)

   @pytest.mark.asyncio
   async def test_analytics_aggregation():
       """Test analytics aggregation across platforms"""
       from app.services.analytics import AnalyticsService

       service = AnalyticsService()
       analytics = await service.get_store_analytics(
           store_id="test_store_123",
           days=30,
       )

       assert "overview" in analytics
       assert "by_platform" in analytics
       assert "recent_posts" in analytics
   ```

2. **Create Comprehensive Documentation**
   ```markdown
   # Social Agent Advanced - Documentation

   ## Features

   ### Platform Support
   - ✅ TikTok (video publishing, analytics)
   - ✅ Instagram (image/video publishing, analytics)
   - ✅ Pinterest (pin publishing, analytics)
   - ✅ Twitter (tweet publishing, analytics)

   ### Content Management
   - AI-powered 30-day content calendars
   - Platform-optimized captions
   - Intelligent hashtag optimization
   - Smart posting schedule

   ### Dashboard Features
   - Calendar view with date selection
   - List view for all posts
   - Post preview with platform-specific styling
   - Post editing and deletion
   - Bulk actions (publish, delete, reschedule)

   ### Analytics
   - Cross-platform metrics aggregation
   - Platform-specific insights
   - Top performing posts
   - Engagement rate tracking

   ## API Reference

   ### Analytics
   ```
   GET /api/v1/analytics/social/{store_id}?days=30
   ```

   ### Bulk Actions
   ```
   POST /api/v1/social/posts/bulk/publish
   Body: { "post_ids": ["id1", "id2"] }

   DELETE /api/v1/social/posts/bulk/delete
   Body: { "post_ids": ["id1", "id2"] }

   PATCH /api/v1/social/posts/bulk/reschedule
   Body: { "post_ids": ["id1", "id2"], "offset_hours": 24 }
   ```

   ## Automated Publishing

   Posts are automatically published via Celery workers:
   - Scheduled task runs every 5 minutes
   - Checks for posts due to be published
   - Publishes to respective platforms
   - Updates status and syncs platform IDs

   ### Running Workers

   ```bash
   # Start Celery worker
   celery -A app.workers.celery_app worker --loglevel=info

   # Start Celery beat (scheduler)
   celery -A app.workers.celery_app beat --loglevel=info
   ```

   ## Usage Example

   ```python
   # Generate calendar
   from app.agents.social.agent import SocialAgent

   agent = SocialAgent()
   calendar = await agent.generate_content_calendar(
       store_id="store_123",
       days=30,
       posts_per_day=2,
   )

   # Get analytics
   from app.services.analytics import AnalyticsService

   analytics_service = AnalyticsService()
   analytics = await analytics_service.get_store_analytics(
       store_id="store_123",
       days=30,
   )
   ```
   ```

3. **Update Requirements**
   ```txt
   # services/growth-engine/requirements.txt (add)
   celery==5.3.4
   celery[redis]==5.3.4
   ```

4. **Create Docker Compose for Workers**
   ```yaml
   # services/growth-engine/docker-compose.yml (update)
   services:
     growth-engine:
       # ... existing config ...

     celery-worker:
       build: .
       command: celery -A app.workers.celery_app worker --loglevel=info
       environment:
         - DATABASE_URL=${DATABASE_URL}
         - REDIS_URL=${REDIS_URL}
       depends_on:
         - redis
         - postgres

     celery-beat:
       build: .
       command: celery -A app.workers.celery_app beat --loglevel=info
       environment:
         - REDIS_URL=${REDIS_URL}
       depends_on:
         - redis
   ```

#### Deliverables
- [ ] Integration tests passing
- [ ] Documentation complete
- [ ] Docker setup for workers
- [ ] Example usage documented

---

## Definition of Done

### Technical Requirements
- [ ] Pinterest client fully functional
- [ ] Twitter client fully functional
- [ ] All 4 platforms (TikTok, Instagram, Pinterest, Twitter) working
- [ ] Content calendar UI complete with calendar and list views
- [ ] Post preview modal working
- [ ] Analytics aggregation across all platforms
- [ ] Social dashboard displaying metrics
- [ ] Publishing worker running
- [ ] Scheduled publishing working
- [ ] Bulk actions functional
- [ ] All tests passing

### UI Requirements
- [ ] Content calendar accessible from navigation
- [ ] Calendar view showing posts by date
- [ ] List view showing all posts
- [ ] Post preview with platform-specific styling
- [ ] Social dashboard with metrics
- [ ] Bulk selection and actions working

### Quality Requirements
- [ ] Workers handle errors gracefully
- [ ] Failed posts logged and retryable
- [ ] Analytics sync on schedule
- [ ] UI responsive and performant
- [ ] Loading states implemented

---

## Success Metrics

- All 4 platforms publish successfully (>95% success rate)
- Calendar loads in < 1 second
- Analytics sync completes in < 5 seconds
- Posts publish within 1 minute of scheduled time
- UI components render without lag
- Zero data loss in bulk operations

---

## Next Sprint Preview

**Sprint 4: Ads Agent Core** will implement:
- Campaign manager for paid advertising
- Creative generator for ad content
- Meta Ads platform integration
- Budget optimizer with ROAS tracking
- Campaign creation UI
- Performance tracking dashboard

---

This sprint completes the **Social Agent system** with full multi-platform support, comprehensive UI, and automated publishing capabilities.
