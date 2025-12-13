# Phase 4 - Sprint 2: Article Generation - Action Plan

## Sprint Overview

**Duration:** 2 weeks
**Team Size:** 3-4 developers (2 backend, 1 frontend, 1 AI/ML specialist)
**Sprint Goal:** Build the AI-powered article generation system with SEO optimization, internal linking, schema markup, and article editor UI.

### Objectives
- [ ] Implement AI article writer with LLM
- [ ] Build SEO optimizer for on-page optimization
- [ ] Create internal linking automation system
- [ ] Implement schema markup generator
- [ ] Develop article editor UI with preview

### Prerequisites
- Phase 4 Sprint 1 (SEO Foundation) completed
- Content plans created with keywords
- Topic clusters defined
- LLM service operational

---

## Day-by-Day Breakdown

### Days 1-4: AI-Powered Article Writer

#### Tasks
1. **Implement Article Writer Core**
   ```python
   # app/agents/seo/article_writer.py
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

2. **Article Generation Service**
   ```python
   # app/services/article_service.py
   from datetime import datetime
   from typing import Dict, Any
   from app.agents.seo import ArticleWriter, SEOOptimizer, InternalLinker, SchemaGenerator

   class ArticleService:
       """Service for managing article generation"""

       def __init__(self):
           self.writer = ArticleWriter()
           self.optimizer = SEOOptimizer()
           self.linker = InternalLinker()
           self.schema_gen = SchemaGenerator()

       async def generate_article(
           self,
           store_id: str,
           planned_article: Dict[str, Any],
       ) -> Any:
           """Generate a full SEO-optimized article"""

           from app.db.prisma import prisma

           store = await prisma.store.find_unique(
               where={"id": store_id},
               include={"products": {"take": 10}}
           )

           # Generate article content
           raw_content = await self.writer.write(
               topic=planned_article["topic"],
               target_keyword=planned_article["target_keyword"],
               secondary_keywords=planned_article["secondary_keywords"],
               outline=planned_article.get("outline"),
               tone=store.brandIdentity.get("tone", "professional"),
               products=store.products,
               word_count_target=planned_article.get("word_count", 1500),
           )

           # Optimize for SEO
           optimized_content = await self.optimizer.optimize(
               content=raw_content,
               target_keyword=planned_article["target_keyword"],
               secondary_keywords=planned_article["secondary_keywords"],
           )

           # Get existing content for internal linking
           existing_articles = await prisma.blogarticle.find_many(
               where={
                   "storeId": store_id,
                   "status": "PUBLISHED"
               }
           )

           # Add internal links
           linked_content = await self.linker.add_links(
               content=optimized_content["content"],
               store_id=store_id,
               existing_content=existing_articles,
               products=store.products,
               current_keyword=planned_article["target_keyword"],
           )

           # Generate schema markup
           schema = await self.schema_gen.generate(
               article_type="BlogPosting",
               title=optimized_content["title"],
               description=optimized_content["meta_description"],
               content=linked_content,
               author=store.name,
               publisher=store.name,
           )

           # Calculate metrics
           word_count = len(linked_content.split())
           reading_time = max(1, word_count // 200)

           # Save article
           article = await prisma.blogarticle.create(
               data={
                   "storeId": store_id,
                   "title": optimized_content["title"],
                   "slug": self._generate_slug(optimized_content["title"]),
                   "metaDescription": optimized_content["meta_description"],
                   "content": linked_content,
                   "excerpt": optimized_content["excerpt"],
                   "targetKeyword": planned_article["target_keyword"],
                   "secondaryKeywords": planned_article["secondary_keywords"],
                   "wordCount": word_count,
                   "readingTime": reading_time,
                   "schemaMarkup": schema,
                   "status": "DRAFT",
               }
           )

           return article

       def _generate_slug(self, title: str) -> str:
           """Generate URL-friendly slug"""
           import re
           slug = title.lower()
           slug = re.sub(r'[^a-z0-9\s-]', '', slug)
           slug = re.sub(r'[\s_]+', '-', slug)
           slug = slug.strip('-')
           return slug[:100]
   ```

#### Deliverables
- [ ] Article writer generating full articles
- [ ] Outline generation working
- [ ] Section writing functional
- [ ] Meta content generation operational

---

### Days 5-7: SEO Optimizer & Internal Linking

#### Tasks
1. **Implement SEO Optimizer**
   ```python
   # app/agents/seo/seo_optimizer.py
   from typing import Dict, Any, List
   from bs4 import BeautifulSoup

   class SEOOptimizer:
       """On-page SEO optimization"""

       async def optimize(
           self,
           content: Dict[str, Any],
           target_keyword: str,
           secondary_keywords: List[str],
       ) -> Dict[str, Any]:
           """Optimize content for SEO"""

           # Parse HTML
           soup = BeautifulSoup(content["content"], 'html.parser')

           # Optimize title
           title = self._optimize_title(
               content["title"],
               target_keyword,
           )

           # Optimize headings
           self._optimize_headings(soup, target_keyword, secondary_keywords)

           # Optimize keyword density
           self._optimize_keyword_density(soup, target_keyword, secondary_keywords)

           # Add image alt tags
           self._optimize_images(soup, target_keyword)

           return {
               "title": title,
               "meta_description": content["meta_description"],
               "excerpt": content["excerpt"],
               "content": str(soup),
               "seo_score": self._calculate_seo_score(
                   soup, title, content["meta_description"], target_keyword, secondary_keywords
               ),
           }

       def _optimize_title(self, title: str, keyword: str) -> str:
           """Optimize title for SEO"""
           if keyword.lower() not in title.lower():
               return f"{keyword.title()} - {title}"
           return title[:60]

       def _optimize_headings(
           self,
           soup: BeautifulSoup,
           target_keyword: str,
           secondary_keywords: List[str],
       ):
           """Ensure keywords in headings"""
           h2_tags = soup.find_all('h2')

           if h2_tags and target_keyword.lower() not in ' '.join([h.text.lower() for h in h2_tags]):
               # Add keyword to first H2 if not present
               if h2_tags:
                   first_h2 = h2_tags[0]
                   first_h2.string = f"{target_keyword.title()}: {first_h2.text}"

       def _optimize_keyword_density(
           self,
           soup: BeautifulSoup,
           target_keyword: str,
           secondary_keywords: List[str],
       ):
           """Check and adjust keyword density"""
           text = soup.get_text()
           words = text.split()
           total_words = len(words)

           # Target keyword density: 1-2%
           keyword_count = text.lower().count(target_keyword.lower())
           target_density = 0.015  # 1.5%
           target_count = int(total_words * target_density)

           # Note: In production, might add keywords to content if needed
           # For now, just calculate and return score

       def _optimize_images(self, soup: BeautifulSoup, keyword: str):
           """Add alt tags to images"""
           images = soup.find_all('img')
           for i, img in enumerate(images):
               if not img.get('alt'):
                   img['alt'] = f"{keyword} - Image {i+1}"

       def _calculate_seo_score(
           self,
           soup: BeautifulSoup,
           title: str,
           meta_desc: str,
           target_keyword: str,
           secondary_keywords: List[str],
       ) -> int:
           """Calculate SEO score (0-100)"""
           score = 0

           # Title optimization (20 points)
           if target_keyword.lower() in title.lower():
               score += 20

           # Meta description (10 points)
           if target_keyword.lower() in meta_desc.lower():
               score += 10

           # Heading structure (20 points)
           h1_count = len(soup.find_all('h1'))
           h2_count = len(soup.find_all('h2'))
           if h1_count == 1:
               score += 10
           if h2_count >= 3:
               score += 10

           # Keyword in headings (15 points)
           headings_text = ' '.join([h.text for h in soup.find_all(['h1', 'h2', 'h3'])])
           if target_keyword.lower() in headings_text.lower():
               score += 15

           # Content length (15 points)
           word_count = len(soup.get_text().split())
           if word_count >= 1000:
               score += 15
           elif word_count >= 500:
               score += 10

           # Secondary keywords (10 points)
           content_lower = soup.get_text().lower()
           secondary_found = sum(1 for kw in secondary_keywords if kw.lower() in content_lower)
           score += min(10, secondary_found * 3)

           # Images with alt tags (10 points)
           images = soup.find_all('img')
           images_with_alt = len([img for img in images if img.get('alt')])
           if images and images_with_alt == len(images):
               score += 10

           return min(100, score)
   ```

2. **Implement Internal Linker**
   ```python
   # app/agents/seo/internal_linker.py
   from typing import List, Dict, Any
   from bs4 import BeautifulSoup
   import re

   class InternalLinker:
       """Automated internal linking system"""

       async def add_links(
           self,
           content: str,
           store_id: str,
           existing_content: List[Any],
           products: List[Dict],
           current_keyword: str,
       ) -> str:
           """Add internal links to content"""

           soup = BeautifulSoup(content, 'html.parser')
           text_content = soup.get_text()

           # Find linking opportunities
           links_to_add = []

           # Link to existing articles
           for article in existing_content[:10]:
               if article.targetKeyword.lower() != current_keyword.lower():
                   # Check if keyword appears in content
                   if article.targetKeyword.lower() in text_content.lower():
                       links_to_add.append({
                           "text": article.targetKeyword,
                           "url": f"/blog/{article.slug}",
                           "type": "article",
                       })

           # Link to products
           for product in products[:5]:
               product_name = product.get("title", "")
               if product_name.lower() in text_content.lower():
                   links_to_add.append({
                       "text": product_name,
                       "url": f"/products/{product.get('slug')}",
                       "type": "product",
                   })

           # Add links to content (limit to 5)
           for link_data in links_to_add[:5]:
               self._add_link_to_content(soup, link_data)

           return str(soup)

       def _add_link_to_content(self, soup: BeautifulSoup, link_data: Dict):
           """Add a single link to content"""
           text_to_link = link_data["text"]

           # Find first occurrence in paragraphs
           paragraphs = soup.find_all('p')
           for p in paragraphs:
               if text_to_link.lower() in p.get_text().lower():
                   # Replace first occurrence with link
                   p_html = str(p)
                   pattern = re.compile(re.escape(text_to_link), re.IGNORECASE)
                   new_html = pattern.sub(
                       f'<a href="{link_data["url"]}">{text_to_link}</a>',
                       p_html,
                       count=1
                   )
                   new_p = BeautifulSoup(new_html, 'html.parser').p
                   p.replace_with(new_p)
                   break
   ```

3. **Implement Schema Generator**
   ```python
   # app/agents/seo/schema_generator.py
   from typing import Dict, Any
   from datetime import datetime

   class SchemaGenerator:
       """Generate structured data / schema markup"""

       async def generate(
           self,
           article_type: str,
           title: str,
           description: str,
           content: str,
           author: str,
           publisher: str,
       ) -> Dict[str, Any]:
           """Generate schema markup for article"""

           # Calculate word count for reading time
           word_count = len(content.split())

           schema = {
               "@context": "https://schema.org",
               "@type": article_type,
               "headline": title,
               "description": description,
               "author": {
                   "@type": "Organization",
                   "name": author,
               },
               "publisher": {
                   "@type": "Organization",
                   "name": publisher,
               },
               "datePublished": datetime.utcnow().isoformat(),
               "dateModified": datetime.utcnow().isoformat(),
               "wordCount": word_count,
               "articleBody": content[:1000],  # Preview
           }

           return schema
   ```

#### Deliverables
- [ ] SEO optimizer functional
- [ ] SEO scoring system working
- [ ] Internal linking automated
- [ ] Schema markup generated

---

### Days 8-10: Article Editor UI

#### Tasks
1. **Build Article Editor Component**
   ```typescript
   // components/seo/ArticleEditor.tsx
   'use client';

   import { useState, useEffect } from 'react';
   import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
   import { Button } from '@/components/ui/button';
   import { Input } from '@/components/ui/input';
   import { Textarea } from '@/components/ui/textarea';
   import { Badge } from '@/components/ui/badge';
   import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
   import dynamic from 'next/dynamic';

   const RichTextEditor = dynamic(() => import('@/components/ui/rich-text-editor'), {
     ssr: false,
   });

   interface ArticleEditorProps {
     articleId?: string;
   }

   export default function ArticleEditor({ articleId }: ArticleEditorProps) {
     const [article, setArticle] = useState<any>(null);
     const [loading, setLoading] = useState(false);
     const [seoScore, setSeoScore] = useState(0);

     useEffect(() => {
       if (articleId) {
         fetchArticle();
       }
     }, [articleId]);

     const fetchArticle = async () => {
       const response = await fetch(`/api/seo/articles/${articleId}`);
       const data = await response.json();
       setArticle(data.article);
       setSeoScore(data.article.seoScore || 0);
     };

     const generateArticle = async (plannedArticle: any) => {
       setLoading(true);
       const response = await fetch('/api/seo/articles/generate', {
         method: 'POST',
         headers: { 'Content-Type': 'application/json' },
         body: JSON.stringify({ planned_article: plannedArticle }),
       });

       const data = await response.json();
       setArticle(data.article);
       setSeoScore(data.article.seoScore || 0);
       setLoading(false);
     };

     const saveArticle = async () => {
       const response = await fetch(`/api/seo/articles/${article.id}`, {
         method: 'PUT',
         headers: { 'Content-Type': 'application/json' },
         body: JSON.stringify(article),
       });

       return response.json();
     };

     const getSeoScoreColor = (score: number) => {
       if (score >= 80) return 'text-green-600';
       if (score >= 60) return 'text-yellow-600';
       return 'text-red-600';
     };

     return (
       <div className="space-y-6">
         {/* SEO Score Card */}
         <Card>
           <CardContent className="p-6">
             <div className="flex justify-between items-center">
               <div>
                 <div className="text-sm text-gray-500">SEO Score</div>
                 <div className={`text-4xl font-bold ${getSeoScoreColor(seoScore)}`}>
                   {seoScore}/100
                 </div>
               </div>
               <div className="text-right">
                 <Button onClick={saveArticle} className="mr-2">Save Draft</Button>
                 <Button variant="default">Publish</Button>
               </div>
             </div>
           </CardContent>
         </Card>

         <Tabs defaultValue="editor" className="w-full">
           <TabsList>
             <TabsTrigger value="editor">Editor</TabsTrigger>
             <TabsTrigger value="preview">Preview</TabsTrigger>
             <TabsTrigger value="seo">SEO Analysis</TabsTrigger>
           </TabsList>

           <TabsContent value="editor">
             <Card>
               <CardHeader>
                 <Input
                   placeholder="Article Title"
                   value={article?.title || ''}
                   onChange={(e) => setArticle({...article, title: e.target.value})}
                   className="text-2xl font-bold border-none"
                 />
               </CardHeader>
               <CardContent>
                 <div className="space-y-4">
                   <div>
                     <label className="text-sm text-gray-500">Meta Description</label>
                     <Textarea
                       placeholder="Meta description (150-160 characters)"
                       value={article?.metaDescription || ''}
                       onChange={(e) => setArticle({...article, metaDescription: e.target.value})}
                       maxLength={160}
                     />
                     <div className="text-xs text-gray-400 mt-1">
                       {article?.metaDescription?.length || 0}/160 characters
                     </div>
                   </div>

                   <div>
                     <label className="text-sm text-gray-500">Target Keyword</label>
                     <Input
                       placeholder="Primary keyword"
                       value={article?.targetKeyword || ''}
                       onChange={(e) => setArticle({...article, targetKeyword: e.target.value})}
                     />
                   </div>

                   <div>
                     <label className="text-sm text-gray-500">Content</label>
                     <RichTextEditor
                       value={article?.content || ''}
                       onChange={(content: string) => setArticle({...article, content})}
                     />
                   </div>
                 </div>
               </CardContent>
             </Card>
           </TabsContent>

           <TabsContent value="preview">
             <Card>
               <CardContent className="p-6">
                 <article className="prose max-w-none">
                   <h1>{article?.title}</h1>
                   <div dangerouslySetInnerHTML={{ __html: article?.content || '' }} />
                 </article>
               </CardContent>
             </Card>
           </TabsContent>

           <TabsContent value="seo">
             <SEOAnalysis article={article} score={seoScore} />
           </TabsContent>
         </Tabs>
       </div>
     );
   }
   ```

2. **SEO Analysis Component**
   ```typescript
   // components/seo/SEOAnalysis.tsx
   'use client';

   import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
   import { CheckCircle, XCircle, AlertCircle } from 'lucide-react';

   interface SEOAnalysisProps {
     article: any;
     score: number;
   }

   export default function SEOAnalysis({ article, score }: SEOAnalysisProps) {
     const checks = [
       {
         name: 'Title Length',
         passing: article?.title?.length > 0 && article?.title?.length <= 60,
         message: article?.title?.length > 60 ? 'Title too long' : 'Title length good',
       },
       {
         name: 'Meta Description',
         passing: article?.metaDescription?.length >= 150 && article?.metaDescription?.length <= 160,
         message: 'Meta description optimal length',
       },
       {
         name: 'Keyword in Title',
         passing: article?.title?.toLowerCase().includes(article?.targetKeyword?.toLowerCase()),
         message: 'Target keyword in title',
       },
       {
         name: 'Word Count',
         passing: article?.wordCount >= 1000,
         message: `${article?.wordCount || 0} words`,
       },
       {
         name: 'Readability',
         passing: article?.readingTime <= 10,
         message: `${article?.readingTime || 0} min read`,
       },
     ];

     return (
       <Card>
         <CardHeader>
           <CardTitle>SEO Analysis</CardTitle>
         </CardHeader>
         <CardContent>
           <div className="space-y-4">
             {checks.map((check, idx) => (
               <div key={idx} className="flex items-center justify-between p-3 border rounded">
                 <div className="flex items-center gap-3">
                   {check.passing ? (
                     <CheckCircle className="text-green-500" size={20} />
                   ) : (
                     <XCircle className="text-red-500" size={20} />
                   )}
                   <div>
                     <div className="font-medium">{check.name}</div>
                     <div className="text-sm text-gray-500">{check.message}</div>
                   </div>
                 </div>
               </div>
             ))}
           </div>
         </CardContent>
       </Card>
     );
   }
   ```

#### Deliverables
- [ ] Article editor UI functional
- [ ] Rich text editing working
- [ ] Preview mode operational
- [ ] SEO analysis display complete

---

### Days 11-12: API Endpoints & Integration

#### Tasks
1. **Article Management API**
   ```python
   # app/api/seo/articles.py
   from fastapi import APIRouter, Depends
   from app.auth import get_current_user
   from app.services.article_service import ArticleService

   router = APIRouter()

   @router.post("/generate")
   async def generate_article(
       planned_article: dict,
       user = Depends(get_current_user),
   ):
       """Generate a new article from plan"""

       service = ArticleService()
       article = await service.generate_article(
           store_id=user.storeId,
           planned_article=planned_article,
       )

       return {"article": article}

   @router.get("/{article_id}")
   async def get_article(
       article_id: str,
       user = Depends(get_current_user),
   ):
       """Get article by ID"""

       from app.db.prisma import prisma

       article = await prisma.blogarticle.find_unique(
           where={"id": article_id}
       )

       return {"article": article}

   @router.put("/{article_id}")
   async def update_article(
       article_id: str,
       title: str,
       content: str,
       meta_description: str,
       user = Depends(get_current_user),
   ):
       """Update article"""

       from app.db.prisma import prisma

       article = await prisma.blogarticle.update(
           where={"id": article_id},
           data={
               "title": title,
               "content": content,
               "metaDescription": meta_description,
           }
       )

       return {"article": article}

   @router.get("/")
   async def list_articles(
       status: str = None,
       user = Depends(get_current_user),
   ):
       """List articles"""

       from app.db.prisma import prisma

       where_clause = {"storeId": user.storeId}
       if status:
           where_clause["status"] = status

       articles = await prisma.blogarticle.find_many(
           where=where_clause,
           order_by={"createdAt": "desc"},
       )

       return {"articles": articles}
   ```

#### Deliverables
- [ ] Article generation API working
- [ ] CRUD operations functional
- [ ] List and filter working
- [ ] Integration with UI complete

---

### Days 13-14: Testing & Documentation

#### Tasks
1. **Integration Tests**
   ```python
   # tests/integration/test_article_generation.py
   import pytest

   @pytest.mark.integration
   async def test_article_generation_workflow(client):
       """Test complete article generation workflow"""

       planned_article = {
           "topic": "Best Minimalist Home Decor Ideas",
           "target_keyword": "minimalist home decor",
           "secondary_keywords": ["minimalist design", "modern home"],
           "word_count": 1500,
       }

       # Generate article
       response = client.post("/api/seo/articles/generate", json={
           "planned_article": planned_article
       })
       assert response.status_code == 200
       article = response.json()["article"]

       # Verify article structure
       assert article["title"]
       assert article["content"]
       assert article["metaDescription"]
       assert article["wordCount"] >= 1000
       assert article["targetKeyword"] == planned_article["target_keyword"]

   @pytest.mark.integration
   async def test_seo_optimization(client):
       """Test SEO optimization"""

       # Generate article first
       response = client.post("/api/seo/articles/generate", json={
           "planned_article": {
               "topic": "Test Topic",
               "target_keyword": "test keyword",
               "secondary_keywords": ["secondary"],
               "word_count": 1000,
           }
       })
       article = response.json()["article"]

       # Verify SEO elements
       assert "test keyword" in article["title"].lower()
       assert "test keyword" in article["metaDescription"].lower()
   ```

#### Deliverables
- [ ] All tests passing
- [ ] Documentation complete
- [ ] Performance benchmarks met

---

## Definition of Done

### Technical Requirements
- [ ] Article generation producing 1500+ word articles
- [ ] SEO optimization achieving 80+ scores
- [ ] Internal linking adding 3-5 links per article
- [ ] Schema markup generated correctly
- [ ] Editor UI fully functional

### Testing Requirements
- [ ] Unit tests for all components
- [ ] Integration tests for workflow
- [ ] Performance tests (< 2 min generation)
- [ ] SEO score validation

### Documentation
- [ ] User guide updated
- [ ] API documentation complete
- [ ] Code examples provided

---

## Success Metrics

### Article Quality
- **Target:** 80%+ SEO score on generated articles
- **Measurement:** SEO scoring system

### Generation Speed
- **Target:** Article generation < 2 minutes
- **Measurement:** API response times

### User Satisfaction
- **Target:** 90%+ approval rate for generated content
- **Measurement:** User feedback and edit rates

---

## Next Sprint Preview

### Phase 4 - Sprint 3: SEO Publishing & Analytics

**Focus Areas:**
1. CMS integration for publishing
2. Search engine submission
3. SEO analytics dashboard
4. Content performance tracking

**Prerequisites:**
- Sprint 2 completed
- Articles generated and optimized
- Editor functional
