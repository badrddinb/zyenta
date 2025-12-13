# Phase 4 - Sprint 5: Influencer Outreach - Action Plan

## Sprint Overview

**Duration:** 2 weeks (14 days)
**Team Size:** 3-4 developers (2 backend, 1 frontend, 1 AI specialist)
**Sprint Goal:** Build automated influencer outreach system with AI-powered personalization, campaign management, and reply handling.

### Objectives
- [ ] Implement automated email and DM outreach system
- [ ] Build AI message personalization engine
- [ ] Create campaign management system with deliverables tracking
- [ ] Implement reply detection and categorization
- [ ] Build negotiation workflow with rate management
- [ ] Add contract generation and tracking
- [ ] Create follow-up automation sequences

### Prerequisites
- Sprint 4 (Influencer Discovery) completed
- Influencer database populated with contacts
- Email service configured (SendGrid/AWS SES)
- Social media DM APIs set up
- LLM API access for personalization

---

## Database Models

### Prisma Schema Extensions

```prisma
model InfluencerCampaign {
  id                String   @id @default(cuid())
  storeId           String
  name              String
  description       String?

  // Campaign Details
  objective         String   // awareness, engagement, conversions, ugc
  budget            Decimal  @db.Decimal(10, 2)
  budgetAllocated   Decimal  @db.Decimal(10, 2) @default(0)

  // Deliverables
  deliverables      Json     // [{type: "post", platform: "instagram", quantity: 3}, ...]

  // Timeline
  startDate         DateTime?
  endDate           DateTime?

  // Targeting
  targetTiers       String[] // nano, micro, macro, mega
  targetNiches      String[]
  minEngagement     Decimal? @db.Decimal(5, 2)
  maxRate           Decimal? @db.Decimal(10, 2)

  // Status
  status            String   @default("draft") // draft, active, paused, completed
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  // Relations
  store             Store    @relation(fields: [storeId], references: [id], onDelete: Cascade)
  outreach          InfluencerOutreach[]
  contracts         InfluencerContract[]

  @@index([storeId, status])
}

model InfluencerOutreach {
  id                String   @id @default(cuid())
  campaignId        String?
  influencerId      String
  storeId           String

  // Outreach Details
  channel           String   // email, instagram_dm, tiktok_dm
  subject           String?
  message           String
  personalizedData  Json?    // Data used for personalization

  // Tracking
  sentAt            DateTime?
  openedAt          DateTime?
  clickedAt         DateTime?
  repliedAt         DateTime?

  // Status
  status            String   @default("draft") // draft, scheduled, sent, opened, replied, interested, declined, no_response

  // Reply
  replyText         String?
  replyCategory     String?  // interested, questions, declined, auto_reply, spam
  replySentiment    String?  // positive, neutral, negative

  // Follow-ups
  followUpCount     Int      @default(0)
  lastFollowUpAt    DateTime?
  nextFollowUpAt    DateTime?

  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  // Relations
  campaign          InfluencerCampaign? @relation(fields: [campaignId], references: [id], onDelete: SetNull)
  influencer        Influencer @relation(fields: [influencerId], references: [id], onDelete: Cascade)
  store             Store    @relation(fields: [storeId], references: [id], onDelete: Cascade)

  @@index([campaignId, status])
  @@index([influencerId])
  @@index([storeId, sentAt])
}

model InfluencerContract {
  id                String   @id @default(cuid())
  campaignId        String
  influencerId      String
  storeId           String

  // Contract Terms
  deliverables      Json     // [{type: "post", platform: "instagram", quantity: 2, dueDate: "..."}, ...]
  compensation      Decimal  @db.Decimal(10, 2)
  compensationType  String   // flat_fee, per_post, commission, product

  // Timeline
  startDate         DateTime
  endDate           DateTime

  // Content Guidelines
  contentGuidelines String?
  mustInclude       String[] // hashtags, mentions, links
  mustAvoid         String[]

  // Legal
  exclusivity       Boolean  @default(false)
  usageRights       String?  // perpetual, limited, none
  contractUrl       String?  // Link to signed contract

  // Status
  status            String   @default("draft") // draft, sent, signed, active, completed, cancelled
  sentAt            DateTime?
  signedAt          DateTime?

  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  // Relations
  campaign          InfluencerCampaign @relation(fields: [campaignId], references: [id], onDelete: Cascade)
  influencer        Influencer @relation(fields: [influencerId], references: [id], onDelete: Cascade)
  store             Store    @relation(fields: [storeId], references: [id], onDelete: Cascade)
  deliveryTracking  ContentDelivery[]

  @@index([campaignId, status])
  @@index([influencerId])
}

model ContentDelivery {
  id              String   @id @default(cuid())
  contractId      String

  // Deliverable Info
  deliverableType String   // post, story, reel, video
  platform        String

  // Content Details
  contentUrl      String?
  postDate        DateTime?

  // Performance (if trackable)
  views           Int?
  likes           Int?
  comments        Int?
  shares          Int?
  clicks          Int?

  // Approval
  status          String   @default("pending") // pending, submitted, approved, rejected, published
  submittedAt     DateTime?
  approvedAt      DateTime?

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  contract        InfluencerContract @relation(fields: [contractId], references: [id], onDelete: Cascade)

  @@index([contractId, status])
}

model OutreachTemplate {
  id              String   @id @default(cuid())
  storeId         String
  name            String

  // Template Content
  subject         String?
  body            String   // With {{placeholders}}
  channel         String   // email, instagram_dm, tiktok_dm

  // Template Type
  templateType    String   // initial_outreach, follow_up_1, follow_up_2, interested_response

  // Variables
  variables       String[] // List of available placeholders

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  store           Store    @relation(fields: [storeId], references: [id], onDelete: Cascade)

  @@index([storeId, templateType])
}
```

---

## Day-by-Day Breakdown

### **Days 1-2: Email Outreach System**

**Backend Tasks:**
1. Set up email service integration (SendGrid/AWS SES)
2. Build email composer with template support
3. Implement tracking pixels for opens/clicks
4. Add email scheduling system
5. Create outreach database models

**Email Service Setup:**
- Configure SMTP/API credentials
- Set up webhook endpoints for delivery/bounce tracking
- Implement email validation
- Add rate limiting to comply with sending limits

**Testing:**
- Test email delivery to various providers
- Verify tracking pixel functionality
- Test bounce handling
- Validate rate limiting

### **Days 3-4: DM Integration**

**Backend Tasks:**
1. Integrate Instagram Direct Message API
2. Integrate TikTok Direct Message API
3. Build unified messaging interface
4. Implement DM status tracking
5. Add error handling for API limitations

**Testing:**
- Test DM sending on both platforms
- Verify rate limits respected
- Test error handling for blocked users

### **Days 5-6: Message Personalization Engine**

**Backend Tasks:**
1. Build AI message personalizer
2. Create profile analysis for compliments
3. Implement brand-product matching
4. Add personality/tone adjustment
5. Build template variable system

**AI Tasks:**
1. Create prompts for personalized greetings
2. Build content compliment generator
3. Implement brand alignment messaging
4. Add product recommendation logic

**Testing:**
- Test personalization quality with various influencers
- Verify all template variables replaced
- Check message tone consistency

### **Day 7: Template Management**

**Backend Tasks:**
1. Build template CRUD system
2. Create default templates for different scenarios
3. Implement template preview functionality
4. Add variable validation

**Frontend Tasks:**
1. Create template management UI
2. Add template editor with variable insertion
3. Build preview functionality

### **Days 8-9: Campaign Management**

**Backend Tasks:**
1. Build campaign creation system
2. Implement influencer selection for campaigns
3. Add deliverable specification
4. Create budget tracking
5. Build timeline management

**Frontend Tasks:**
1. Create campaign creation wizard
2. Build influencer selection interface
3. Add deliverable configuration
4. Create campaign dashboard

**Testing:**
- Test campaign creation flow
- Verify budget calculations
- Test influencer assignment

### **Days 10-11: Reply Detection & Categorization**

**Backend Tasks:**
1. Set up email reply webhook handling
2. Implement DM reply polling/webhooks
3. Build AI reply categorizer
4. Add sentiment analysis
5. Create notification system for replies

**AI Tasks:**
1. Create prompts for reply categorization
2. Build sentiment analyzer
3. Implement interest level detector

**Testing:**
- Test reply detection from emails
- Test reply detection from DMs
- Verify categorization accuracy
- Test notification delivery

### **Days 12-13: Negotiation & Contract System**

**Backend Tasks:**
1. Build negotiation tracker
2. Implement rate management
3. Create contract generator
4. Add contract template system
5. Build e-signature integration (optional)

**Frontend Tasks:**
1. Create negotiation interface
2. Build contract generation UI
3. Add contract tracking dashboard

**Testing:**
- Test contract generation with various terms
- Verify rate calculations
- Test contract PDF generation

### **Day 14: Follow-up Automation**

**Backend Tasks:**
1. Build follow-up scheduler
2. Implement auto-follow-up logic (3, 7, 14 days)
3. Add smart follow-up based on behavior
4. Create follow-up templates

**Celery Tasks:**
1. Daily job to process scheduled follow-ups
2. Smart follow-up trigger based on opens without replies

**Testing:**
- Test follow-up scheduling
- Verify follow-up timing logic
- Test follow-up template personalization

---

## Core Components Implementation

### 1. Outreach Manager

```python
# services/influencer/outreach_manager.py

from typing import List, Dict, Optional, Any
from datetime import datetime, timedelta
from decimal import Decimal
from prisma import Prisma
from openai import AsyncOpenAI

class OutreachManager:
    """Manage influencer outreach campaigns"""

    def __init__(
        self,
        prisma: Prisma,
        email_service: 'EmailService',
        dm_service: 'DMService',
        personalizer: 'MessagePersonalizer',
    ):
        self.prisma = prisma
        self.email_service = email_service
        self.dm_service = dm_service
        self.personalizer = personalizer

    async def send_outreach(
        self,
        campaign_id: Optional[str],
        influencer_id: str,
        store_id: str,
        channel: str,
        template_id: Optional[str] = None,
        custom_message: Optional[str] = None,
        schedule_for: Optional[datetime] = None,
    ) -> Dict[str, Any]:
        """
        Send outreach message to influencer

        Args:
            campaign_id: Campaign ID (optional)
            influencer_id: Influencer to contact
            store_id: Store ID
            channel: email, instagram_dm, tiktok_dm
            template_id: Template to use (optional)
            custom_message: Custom message (optional)
            schedule_for: Schedule for later (optional)

        Returns:
            Outreach record
        """
        # Get influencer
        influencer = await self.prisma.influencer.find_unique(
            where={'id': influencer_id}
        )

        if not influencer:
            raise ValueError("Influencer not found")

        # Get store
        store = await self.prisma.store.find_unique(
            where={'id': store_id}
        )

        # Prepare message
        if template_id:
            template = await self.prisma.outreachtemplate.find_unique(
                where={'id': template_id}
            )
            if not template:
                raise ValueError("Template not found")

            # Personalize template
            subject, message = await self.personalizer.personalize_template(
                template=template,
                influencer=influencer,
                store=store,
            )
        elif custom_message:
            subject = None
            message = custom_message
        else:
            # Generate default personalized message
            subject, message = await self.personalizer.generate_message(
                influencer=influencer,
                store=store,
                channel=channel,
            )

        # Create outreach record
        outreach = await self.prisma.influenceroutreach.create(
            data={
                'campaignId': campaign_id,
                'influencerId': influencer_id,
                'storeId': store_id,
                'channel': channel,
                'subject': subject,
                'message': message,
                'status': 'scheduled' if schedule_for else 'draft',
            }
        )

        # Send immediately or schedule
        if schedule_for:
            # Schedule for later (handled by celery worker)
            await self.prisma.influenceroutreach.update(
                where={'id': outreach.id},
                data={'nextFollowUpAt': schedule_for}
            )
        else:
            # Send now
            await self._send_message(outreach.id)

        return outreach

    async def _send_message(self, outreach_id: str) -> bool:
        """Actually send the message"""
        outreach = await self.prisma.influenceroutreach.find_unique(
            where={'id': outreach_id},
            include={
                'influencer': True,
                'store': True,
            }
        )

        if not outreach:
            raise ValueError("Outreach not found")

        try:
            if outreach.channel == 'email':
                # Send email
                result = await self.email_service.send_email(
                    to_email=outreach.influencer.email,
                    to_name=outreach.influencer.displayName or outreach.influencer.username,
                    from_email=outreach.store.supportEmail or outreach.store.email,
                    from_name=outreach.store.name,
                    subject=outreach.subject,
                    body=outreach.message,
                    tracking_id=outreach.id,
                )

                # Update status
                await self.prisma.influenceroutreach.update(
                    where={'id': outreach_id},
                    data={
                        'status': 'sent',
                        'sentAt': datetime.utcnow(),
                        'nextFollowUpAt': datetime.utcnow() + timedelta(days=3),
                    }
                )

                return True

            elif outreach.channel in ['instagram_dm', 'tiktok_dm']:
                # Send DM
                platform = outreach.channel.split('_')[0]
                result = await self.dm_service.send_dm(
                    platform=platform,
                    recipient_id=outreach.influencer.platformUserId,
                    message=outreach.message,
                )

                await self.prisma.influenceroutreach.update(
                    where={'id': outreach_id},
                    data={
                        'status': 'sent',
                        'sentAt': datetime.utcnow(),
                        'nextFollowUpAt': datetime.utcnow() + timedelta(days=3),
                    }
                )

                return True

        except Exception as e:
            # Update with error
            await self.prisma.influenceroutreach.update(
                where={'id': outreach_id},
                data={'status': 'failed'}
            )
            raise

        return False

    async def send_bulk_outreach(
        self,
        campaign_id: str,
        influencer_ids: List[str],
        template_id: str,
    ) -> Dict[str, Any]:
        """Send outreach to multiple influencers"""
        results = {
            'sent': 0,
            'failed': 0,
            'scheduled': 0,
        }

        campaign = await self.prisma.influencercampaign.find_unique(
            where={'id': campaign_id}
        )

        for influencer_id in influencer_ids:
            try:
                # Determine best channel
                influencer = await self.prisma.influencer.find_unique(
                    where={'id': influencer_id}
                )

                channel = 'email' if influencer.email else f"{influencer.platform}_dm"

                # Send outreach
                await self.send_outreach(
                    campaign_id=campaign_id,
                    influencer_id=influencer_id,
                    store_id=campaign.storeId,
                    channel=channel,
                    template_id=template_id,
                )

                results['sent'] += 1

            except Exception as e:
                print(f"Error sending to {influencer_id}: {e}")
                results['failed'] += 1

        return results

    async def send_follow_up(
        self,
        outreach_id: str,
        message: Optional[str] = None,
    ) -> bool:
        """Send follow-up message"""
        outreach = await self.prisma.influenceroutreach.find_unique(
            where={'id': outreach_id},
            include={
                'influencer': True,
                'store': True,
            }
        )

        if not outreach:
            raise ValueError("Outreach not found")

        # Check if reply already received
        if outreach.status in ['replied', 'interested', 'declined']:
            return False

        # Generate follow-up message if not provided
        if not message:
            message = await self.personalizer.generate_follow_up(
                influencer=outreach.influencer,
                store=outreach.store,
                previous_message=outreach.message,
                follow_up_number=outreach.followUpCount + 1,
            )

        # Create new outreach for follow-up
        follow_up = await self.prisma.influenceroutreach.create(
            data={
                'campaignId': outreach.campaignId,
                'influencerId': outreach.influencerId,
                'storeId': outreach.storeId,
                'channel': outreach.channel,
                'subject': f"Re: {outreach.subject}" if outreach.subject else None,
                'message': message,
                'status': 'draft',
            }
        )

        # Send follow-up
        await self._send_message(follow_up.id)

        # Update original outreach
        await self.prisma.influenceroutreach.update(
            where={'id': outreach_id},
            data={
                'followUpCount': outreach.followUpCount + 1,
                'lastFollowUpAt': datetime.utcnow(),
                'nextFollowUpAt': None,  # Clear next follow-up
            }
        )

        return True
```

### 2. Message Personalizer

```python
# services/influencer/message_personalizer.py

from typing import Dict, Optional, Tuple
from openai import AsyncOpenAI

class MessagePersonalizer:
    """AI-powered message personalization"""

    def __init__(self, openai_client: AsyncOpenAI):
        self.openai = openai_client

    async def personalize_template(
        self,
        template: Any,
        influencer: Any,
        store: Any,
    ) -> Tuple[Optional[str], str]:
        """Personalize template with influencer/store data"""
        # Replace basic placeholders
        subject = template.subject or ""
        body = template.body

        replacements = {
            '{{influencer_name}}': influencer.displayName or influencer.username,
            '{{influencer_username}}': influencer.username,
            '{{store_name}}': store.name,
            '{{followers_count}}': f"{influencer.followersCount:,}",
            '{{engagement_rate}}': f"{influencer.engagementRate}%",
            '{{niche}}': ', '.join(influencer.niche or []),
        }

        for placeholder, value in replacements.items():
            subject = subject.replace(placeholder, str(value))
            body = body.replace(placeholder, str(value))

        # Use AI for smart personalization
        body = await self._add_ai_personalization(
            message=body,
            influencer=influencer,
            store=store,
        )

        return subject if subject else None, body

    async def _add_ai_personalization(
        self,
        message: str,
        influencer: Any,
        store: Any,
    ) -> str:
        """Add AI-generated personalized touches"""
        # If message has {{ai_compliment}} placeholder
        if '{{ai_compliment}}' in message:
            compliment = await self._generate_compliment(influencer)
            message = message.replace('{{ai_compliment}}', compliment)

        # If message has {{brand_alignment}}
        if '{{brand_alignment}}' in message:
            alignment = await self._generate_brand_alignment(influencer, store)
            message = message.replace('{{brand_alignment}}', alignment)

        return message

    async def _generate_compliment(self, influencer: Any) -> str:
        """Generate personalized compliment"""
        prompt = f"""Generate a genuine, specific compliment for this influencer based on their profile.
Keep it professional and relevant to potential brand partnerships.

Influencer:
- Username: {influencer.username}
- Bio: {influencer.bio or 'N/A'}
- Niche: {', '.join(influencer.niche or [])}
- Followers: {influencer.followersCount:,}
- Engagement: {influencer.engagementRate}%

Write a 1-2 sentence compliment. Be specific, not generic."""

        try:
            response = await self.openai.chat.completions.create(
                model="gpt-4",
                messages=[
                    {"role": "system", "content": "You are an expert at crafting authentic influencer outreach."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.7,
            )

            return response.choices[0].message.content.strip()

        except Exception as e:
            print(f"Error generating compliment: {e}")
            return f"I've been following your {influencer.niche[0] if influencer.niche else 'content'} content"

    async def _generate_brand_alignment(
        self,
        influencer: Any,
        store: Any,
    ) -> str:
        """Explain why brand and influencer are a good fit"""
        prompt = f"""Explain in 1-2 sentences why this brand and influencer would be a great partnership.
Focus on audience alignment and shared values.

Brand:
- Name: {store.name}
- Description: {store.description or 'N/A'}
- Niche: {store.niche or 'General'}

Influencer:
- Username: {influencer.username}
- Niche: {', '.join(influencer.niche or [])}
- Followers: {influencer.followersCount:,}

Be specific and authentic."""

        try:
            response = await self.openai.chat.completions.create(
                model="gpt-4",
                messages=[
                    {"role": "system", "content": "You are an expert at influencer marketing."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.7,
            )

            return response.choices[0].message.content.strip()

        except Exception as e:
            print(f"Error generating alignment: {e}")
            return f"Your audience aligns perfectly with our {store.niche or 'brand'} values"

    async def generate_message(
        self,
        influencer: Any,
        store: Any,
        channel: str,
    ) -> Tuple[Optional[str], str]:
        """Generate complete personalized message from scratch"""
        prompt = f"""Write a professional influencer outreach message for a brand partnership.

Brand:
- Name: {store.name}
- Description: {store.description or 'N/A'}

Influencer:
- Name: {influencer.displayName or influencer.username}
- Platform: {influencer.platform}
- Followers: {influencer.followersCount:,}
- Niche: {', '.join(influencer.niche or [])}

Requirements:
- Professional but friendly tone
- Compliment their content (be specific if possible)
- Explain why they're a good fit
- Propose collaboration
- Include clear call-to-action
- Keep it concise (150-200 words)
{'- Include subject line' if channel == 'email' else ''}

Format:
{'Subject: [subject line]' if channel == 'email' else ''}
{'' if channel == 'email' else 'Message:'}
[message body]"""

        try:
            response = await self.openai.chat.completions.create(
                model="gpt-4",
                messages=[
                    {"role": "system", "content": "You are an expert at influencer outreach."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.7,
            )

            content = response.choices[0].message.content.strip()

            # Parse subject and body
            if channel == 'email' and 'Subject:' in content:
                lines = content.split('\n')
                subject_line = next((l for l in lines if l.startswith('Subject:')), None)
                subject = subject_line.replace('Subject:', '').strip() if subject_line else None

                body_start = content.find('\n\n')
                body = content[body_start:].strip() if body_start > 0 else content
            else:
                subject = None
                body = content

            return subject, body

        except Exception as e:
            print(f"Error generating message: {e}")
            # Fallback template
            return (
                f"Partnership Opportunity with {store.name}" if channel == 'email' else None,
                f"Hi {influencer.displayName or influencer.username},\n\n"
                f"I love your {influencer.niche[0] if influencer.niche else 'content'}! "
                f"I think your audience would be a perfect fit for {store.name}. "
                f"Would you be interested in collaborating?\n\nLooking forward to hearing from you!"
            )

    async def generate_follow_up(
        self,
        influencer: Any,
        store: Any,
        previous_message: str,
        follow_up_number: int,
    ) -> str:
        """Generate follow-up message"""
        tone = "polite and friendly" if follow_up_number == 1 else "brief and respectful"

        prompt = f"""Write a follow-up message for an influencer who hasn't responded.

Previous message:
{previous_message[:300]}

Follow-up number: {follow_up_number}
Tone: {tone}

Requirements:
- Keep it short (50-75 words)
- Be polite, not pushy
- Add value or new information if possible
- Include clear call-to-action

Message:"""

        try:
            response = await self.openai.chat.completions.create(
                model="gpt-4",
                messages=[
                    {"role": "system", "content": "You are an expert at follow-up outreach."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.7,
            )

            return response.choices[0].message.content.strip()

        except Exception as e:
            print(f"Error generating follow-up: {e}")
            if follow_up_number == 1:
                return f"Hi {influencer.displayName or influencer.username}, just wanted to follow up on my previous message. Would love to explore a partnership. Let me know if you're interested!"
            else:
                return f"Hi again! Still interested in collaborating. No pressure - just wanted to check in one last time."
```

### 3. Reply Handler

```python
# services/influencer/reply_handler.py

from typing import Dict, Optional
from datetime import datetime
from openai import AsyncOpenAI
from prisma import Prisma

class ReplyHandler:
    """Handle and categorize influencer replies"""

    def __init__(
        self,
        prisma: Prisma,
        openai_client: AsyncOpenAI,
    ):
        self.prisma = prisma
        self.openai = openai_client

    async def process_reply(
        self,
        outreach_id: str,
        reply_text: str,
    ) -> Dict[str, any]:
        """Process incoming reply"""
        # Get outreach
        outreach = await self.prisma.influenceroutreach.find_unique(
            where={'id': outreach_id}
        )

        if not outreach:
            raise ValueError("Outreach not found")

        # Categorize reply
        category = await self._categorize_reply(reply_text)
        sentiment = await self._analyze_sentiment(reply_text)

        # Update outreach
        updated = await self.prisma.influenceroutreach.update(
            where={'id': outreach_id},
            data={
                'status': self._get_status_from_category(category),
                'replyText': reply_text,
                'replyCategory': category,
                'replySentiment': sentiment,
                'repliedAt': datetime.utcnow(),
                'nextFollowUpAt': None,  # Clear scheduled follow-ups
            }
        )

        # Generate suggested response
        suggested_response = await self._generate_response_suggestion(
            original_message=outreach.message,
            reply=reply_text,
            category=category,
        )

        return {
            'outreach': updated,
            'category': category,
            'sentiment': sentiment,
            'suggested_response': suggested_response,
        }

    async def _categorize_reply(self, reply_text: str) -> str:
        """Categorize reply using AI"""
        prompt = f"""Categorize this influencer reply into ONE of these categories:
- interested: They're interested in the partnership
- questions: They have questions or need more info
- declined: They're not interested
- rate_negotiation: They're discussing rates/compensation
- auto_reply: This is an automated out-of-office or similar
- spam: This is spam or irrelevant

Reply:
{reply_text}

Respond with ONLY the category name."""

        try:
            response = await self.openai.chat.completions.create(
                model="gpt-4",
                messages=[
                    {"role": "system", "content": "You are an expert at categorizing messages."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.3,
            )

            category = response.choices[0].message.content.strip().lower()

            # Validate category
            valid_categories = ['interested', 'questions', 'declined', 'rate_negotiation', 'auto_reply', 'spam']
            if category not in valid_categories:
                return 'questions'  # Default

            return category

        except Exception as e:
            print(f"Error categorizing reply: {e}")
            return 'questions'

    async def _analyze_sentiment(self, reply_text: str) -> str:
        """Analyze sentiment"""
        prompt = f"""Analyze the sentiment of this message. Respond with ONE word: positive, neutral, or negative.

Message:
{reply_text}

Sentiment:"""

        try:
            response = await self.openai.chat.completions.create(
                model="gpt-4",
                messages=[
                    {"role": "system", "content": "You are a sentiment analyzer."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.3,
            )

            sentiment = response.choices[0].message.content.strip().lower()

            if sentiment not in ['positive', 'neutral', 'negative']:
                return 'neutral'

            return sentiment

        except Exception as e:
            print(f"Error analyzing sentiment: {e}")
            return 'neutral'

    def _get_status_from_category(self, category: str) -> str:
        """Map category to outreach status"""
        mapping = {
            'interested': 'interested',
            'questions': 'replied',
            'declined': 'declined',
            'rate_negotiation': 'interested',
            'auto_reply': 'sent',
            'spam': 'replied',
        }
        return mapping.get(category, 'replied')

    async def _generate_response_suggestion(
        self,
        original_message: str,
        reply: str,
        category: str,
    ) -> Optional[str]:
        """Generate suggested response"""
        if category in ['declined', 'spam', 'auto_reply']:
            return None  # No response needed

        prompt = f"""Generate a professional response to this influencer reply.

Original outreach:
{original_message[:200]}

Their reply:
{reply}

Category: {category}

Requirements:
- Professional and friendly
- Address their specific points/questions
- Move toward next steps
- Keep it concise (100-150 words)

Response:"""

        try:
            response = await self.openai.chat.completions.create(
                model="gpt-4",
                messages=[
                    {"role": "system", "content": "You are an expert at influencer communications."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.7,
            )

            return response.choices[0].message.content.strip()

        except Exception as e:
            print(f"Error generating response: {e}")
            return None
```

### 4. Contract Generator

```python
# services/influencer/contract_generator.py

from typing import Dict, List
from datetime import datetime, timedelta
from decimal import Decimal
from prisma import Prisma
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, Table
from reportlab.lib.styles import getSampleStyleSheet
from io import BytesIO

class ContractGenerator:
    """Generate influencer contracts"""

    def __init__(self, prisma: Prisma):
        self.prisma = prisma

    async def create_contract(
        self,
        campaign_id: str,
        influencer_id: str,
        deliverables: List[Dict],
        compensation: Decimal,
        compensation_type: str,
        start_date: datetime,
        end_date: datetime,
        content_guidelines: Optional[str] = None,
        must_include: Optional[List[str]] = None,
        must_avoid: Optional[List[str]] = None,
        exclusivity: bool = False,
        usage_rights: str = "limited",
    ) -> 'InfluencerContract':
        """Create contract"""
        campaign = await self.prisma.influencercampaign.find_unique(
            where={'id': campaign_id}
        )

        contract = await self.prisma.influencercontract.create(
            data={
                'campaignId': campaign_id,
                'influencerId': influencer_id,
                'storeId': campaign.storeId,
                'deliverables': deliverables,
                'compensation': compensation,
                'compensationType': compensation_type,
                'startDate': start_date,
                'endDate': end_date,
                'contentGuidelines': content_guidelines,
                'mustInclude': must_include or [],
                'mustAvoid': must_avoid or [],
                'exclusivity': exclusivity,
                'usageRights': usage_rights,
                'status': 'draft',
            }
        )

        return contract

    async def generate_pdf(
        self,
        contract_id: str,
    ) -> bytes:
        """Generate contract PDF"""
        contract = await self.prisma.influencercontract.find_unique(
            where={'id': contract_id},
            include={
                'campaign': True,
                'influencer': True,
                'store': True,
            }
        )

        if not contract:
            raise ValueError("Contract not found")

        # Create PDF
        buffer = BytesIO()
        doc = SimpleDocTemplate(buffer, pagesize=letter)
        styles = getSampleStyleSheet()
        story = []

        # Title
        title = Paragraph(
            f"<b>Influencer Partnership Agreement</b>",
            styles['Title']
        )
        story.append(title)
        story.append(Spacer(1, 12))

        # Parties
        parties = Paragraph(
            f"This agreement is between {contract.store.name} (\"Brand\") and "
            f"{contract.influencer.displayName or contract.influencer.username} (\"Influencer\").",
            styles['Normal']
        )
        story.append(parties)
        story.append(Spacer(1, 12))

        # Campaign details
        campaign_details = Paragraph(
            f"<b>Campaign:</b> {contract.campaign.name}<br/>"
            f"<b>Duration:</b> {contract.startDate.strftime('%Y-%m-%d')} to {contract.endDate.strftime('%Y-%m-%d')}<br/>"
            f"<b>Compensation:</b> ${contract.compensation} ({contract.compensationType})",
            styles['Normal']
        )
        story.append(campaign_details)
        story.append(Spacer(1, 12))

        # Deliverables
        story.append(Paragraph("<b>Deliverables:</b>", styles['Heading2']))
        deliverables_data = [['Type', 'Platform', 'Quantity', 'Due Date']]
        for d in contract.deliverables:
            deliverables_data.append([
                d['type'],
                d['platform'],
                str(d['quantity']),
                d.get('dueDate', 'TBD')
            ])

        deliverables_table = Table(deliverables_data)
        story.append(deliverables_table)
        story.append(Spacer(1, 12))

        # Content guidelines
        if contract.contentGuidelines:
            guidelines = Paragraph(
                f"<b>Content Guidelines:</b><br/>{contract.contentGuidelines}",
                styles['Normal']
            )
            story.append(guidelines)
            story.append(Spacer(1, 12))

        # Must include/avoid
        if contract.mustInclude:
            must_include = Paragraph(
                f"<b>Must Include:</b> {', '.join(contract.mustInclude)}",
                styles['Normal']
            )
            story.append(must_include)
            story.append(Spacer(1, 12))

        if contract.mustAvoid:
            must_avoid = Paragraph(
                f"<b>Must Avoid:</b> {', '.join(contract.mustAvoid)}",
                styles['Normal']
            )
            story.append(must_avoid)
            story.append(Spacer(1, 12))

        # Terms
        terms = Paragraph(
            f"<b>Additional Terms:</b><br/>"
            f"Exclusivity: {'Yes' if contract.exclusivity else 'No'}<br/>"
            f"Usage Rights: {contract.usageRights}",
            styles['Normal']
        )
        story.append(terms)
        story.append(Spacer(1, 24))

        # Signatures
        signatures = Paragraph(
            "Signatures:<br/><br/>"
            "_________________________<br/>"
            f"{contract.store.name}<br/><br/><br/>"
            "_________________________<br/>"
            f"{contract.influencer.displayName or contract.influencer.username}",
            styles['Normal']
        )
        story.append(signatures)

        # Build PDF
        doc.build(story)
        buffer.seek(0)

        return buffer.getvalue()
```

---

## API Endpoints

```python
# routers/influencer_outreach.py

from fastapi import APIRouter, Depends, HTTPException, Query, UploadFile, File
from typing import List, Optional
from pydantic import BaseModel
from datetime import datetime
from decimal import Decimal

from ..dependencies import get_current_user, get_prisma
from ..services.influencer.outreach_manager import OutreachManager
from ..services.influencer.reply_handler import ReplyHandler
from ..services.influencer.contract_generator import ContractGenerator

router = APIRouter(prefix="/api/influencer/outreach", tags=["influencer-outreach"])

# Request Models

class OutreachRequest(BaseModel):
    campaign_id: Optional[str] = None
    influencer_id: str
    channel: str  # email, instagram_dm, tiktok_dm
    template_id: Optional[str] = None
    custom_message: Optional[str] = None
    schedule_for: Optional[datetime] = None

class BulkOutreachRequest(BaseModel):
    campaign_id: str
    influencer_ids: List[str]
    template_id: str

class CampaignCreate(BaseModel):
    name: str
    description: Optional[str] = None
    objective: str
    budget: Decimal
    deliverables: List[dict]
    target_tiers: List[str]
    target_niches: List[str]
    min_engagement: Optional[Decimal] = None
    max_rate: Optional[Decimal] = None
    start_date: Optional[datetime] = None
    end_date: Optional[datetime] = None

class ReplyUpdate(BaseModel):
    reply_text: str

class ContractCreate(BaseModel):
    campaign_id: str
    influencer_id: str
    deliverables: List[dict]
    compensation: Decimal
    compensation_type: str
    start_date: datetime
    end_date: datetime
    content_guidelines: Optional[str] = None
    must_include: Optional[List[str]] = None
    must_avoid: Optional[List[str]] = None
    exclusivity: bool = False
    usage_rights: str = "limited"

# Endpoints

@router.post("/send")
async def send_outreach(
    request: OutreachRequest,
    current_user = Depends(get_current_user),
    outreach_manager: OutreachManager = Depends(),
):
    """Send outreach to influencer"""
    try:
        result = await outreach_manager.send_outreach(
            campaign_id=request.campaign_id,
            influencer_id=request.influencer_id,
            store_id=current_user.store_id,
            channel=request.channel,
            template_id=request.template_id,
            custom_message=request.custom_message,
            schedule_for=request.schedule_for,
        )

        return {
            'success': True,
            'data': result,
        }
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.post("/bulk")
async def send_bulk_outreach(
    request: BulkOutreachRequest,
    current_user = Depends(get_current_user),
    outreach_manager: OutreachManager = Depends(),
):
    """Send outreach to multiple influencers"""
    try:
        results = await outreach_manager.send_bulk_outreach(
            campaign_id=request.campaign_id,
            influencer_ids=request.influencer_ids,
            template_id=request.template_id,
        )

        return {
            'success': True,
            'data': results,
        }
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/list")
async def list_outreach(
    campaign_id: Optional[str] = None,
    status: Optional[str] = None,
    page: int = Query(1, ge=1),
    per_page: int = Query(20, ge=1, le=100),
    current_user = Depends(get_current_user),
    prisma = Depends(get_prisma),
):
    """List outreach messages"""
    where = {'storeId': current_user.store_id}

    if campaign_id:
        where['campaignId'] = campaign_id

    if status:
        where['status'] = status

    total = await prisma.influenceroutreach.count(where=where)

    outreach = await prisma.influenceroutreach.find_many(
        where=where,
        include={
            'influencer': True,
            'campaign': True,
        },
        order_by={'sentAt': 'desc'},
        skip=(page - 1) * per_page,
        take=per_page,
    )

    return {
        'success': True,
        'data': {
            'outreach': outreach,
            'pagination': {
                'page': page,
                'per_page': per_page,
                'total': total,
                'pages': (total + per_page - 1) // per_page,
            }
        }
    }

@router.post("/{outreach_id}/reply")
async def handle_reply(
    outreach_id: str,
    request: ReplyUpdate,
    current_user = Depends(get_current_user),
    reply_handler: ReplyHandler = Depends(),
):
    """Process influencer reply"""
    try:
        result = await reply_handler.process_reply(
            outreach_id=outreach_id,
            reply_text=request.reply_text,
        )

        return {
            'success': True,
            'data': result,
        }
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.post("/{outreach_id}/follow-up")
async def send_follow_up(
    outreach_id: str,
    current_user = Depends(get_current_user),
    outreach_manager: OutreachManager = Depends(),
):
    """Send follow-up message"""
    try:
        result = await outreach_manager.send_follow_up(outreach_id=outreach_id)

        return {
            'success': True,
            'data': {'sent': result},
        }
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

# Campaign Endpoints

@router.post("/campaigns")
async def create_campaign(
    request: CampaignCreate,
    current_user = Depends(get_current_user),
    prisma = Depends(get_prisma),
):
    """Create campaign"""
    campaign = await prisma.influencercampaign.create(
        data={
            'storeId': current_user.store_id,
            'name': request.name,
            'description': request.description,
            'objective': request.objective,
            'budget': request.budget,
            'deliverables': request.deliverables,
            'targetTiers': request.target_tiers,
            'targetNiches': request.target_niches,
            'minEngagement': request.min_engagement,
            'maxRate': request.max_rate,
            'startDate': request.start_date,
            'endDate': request.end_date,
            'status': 'draft',
        }
    )

    return {
        'success': True,
        'data': campaign,
    }

@router.get("/campaigns")
async def list_campaigns(
    status: Optional[str] = None,
    page: int = Query(1, ge=1),
    per_page: int = Query(20, ge=1, le=100),
    current_user = Depends(get_current_user),
    prisma = Depends(get_prisma),
):
    """List campaigns"""
    where = {'storeId': current_user.store_id}

    if status:
        where['status'] = status

    total = await prisma.influencercampaign.count(where=where)

    campaigns = await prisma.influencercampaign.find_many(
        where=where,
        order_by={'createdAt': 'desc'},
        skip=(page - 1) * per_page,
        take=per_page,
    )

    return {
        'success': True,
        'data': {
            'campaigns': campaigns,
            'pagination': {
                'page': page,
                'per_page': per_page,
                'total': total,
                'pages': (total + per_page - 1) // per_page,
            }
        }
    }

# Contract Endpoints

@router.post("/contracts")
async def create_contract(
    request: ContractCreate,
    current_user = Depends(get_current_user),
    contract_generator: ContractGenerator = Depends(),
):
    """Create contract"""
    try:
        contract = await contract_generator.create_contract(
            campaign_id=request.campaign_id,
            influencer_id=request.influencer_id,
            deliverables=request.deliverables,
            compensation=request.compensation,
            compensation_type=request.compensation_type,
            start_date=request.start_date,
            end_date=request.end_date,
            content_guidelines=request.content_guidelines,
            must_include=request.must_include,
            must_avoid=request.must_avoid,
            exclusivity=request.exclusivity,
            usage_rights=request.usage_rights,
        )

        return {
            'success': True,
            'data': contract,
        }
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/contracts/{contract_id}/pdf")
async def get_contract_pdf(
    contract_id: str,
    current_user = Depends(get_current_user),
    contract_generator: ContractGenerator = Depends(),
):
    """Generate contract PDF"""
    try:
        pdf_bytes = await contract_generator.generate_pdf(contract_id)

        from fastapi.responses import Response

        return Response(
            content=pdf_bytes,
            media_type="application/pdf",
            headers={
                "Content-Disposition": f"attachment; filename=contract_{contract_id}.pdf"
            }
        )
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))
```

---

## UI Components

### Campaign Management Dashboard

```typescript
// app/influencer/campaigns/page.tsx

'use client';

import { useState, useEffect } from 'react';
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { Plus, Users, Mail, CheckCircle } from 'lucide-react';

export default function CampaignsPage() {
    const [campaigns, setCampaigns] = useState<any[]>([]);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        loadCampaigns();
    }, []);

    const loadCampaigns = async () => {
        const response = await fetch('/api/influencer/outreach/campaigns');
        const data = await response.json();

        if (data.success) {
            setCampaigns(data.data.campaigns);
        }

        setLoading(false);
    };

    return (
        <div className="container mx-auto p-6">
            <div className="flex justify-between items-center mb-6">
                <h1 className="text-3xl font-bold">Influencer Campaigns</h1>
                <Button>
                    <Plus className="mr-2 h-4 w-4" />
                    New Campaign
                </Button>
            </div>

            {/* Campaign Stats */}
            <div className="grid grid-cols-4 gap-4 mb-6">
                <Card>
                    <CardContent className="p-4">
                        <div className="flex items-center justify-between">
                            <div>
                                <p className="text-sm text-gray-500">Active Campaigns</p>
                                <p className="text-2xl font-bold">
                                    {campaigns.filter(c => c.status === 'active').length}
                                </p>
                            </div>
                            <Users className="h-8 w-8 text-blue-500" />
                        </div>
                    </CardContent>
                </Card>

                <Card>
                    <CardContent className="p-4">
                        <div className="flex items-center justify-between">
                            <div>
                                <p className="text-sm text-gray-500">Outreach Sent</p>
                                <p className="text-2xl font-bold">247</p>
                            </div>
                            <Mail className="h-8 w-8 text-green-500" />
                        </div>
                    </CardContent>
                </Card>

                <Card>
                    <CardContent className="p-4">
                        <div className="flex items-center justify-between">
                            <div>
                                <p className="text-sm text-gray-500">Response Rate</p>
                                <p className="text-2xl font-bold">18%</p>
                            </div>
                            <CheckCircle className="h-8 w-8 text-purple-500" />
                        </div>
                    </CardContent>
                </Card>

                <Card>
                    <CardContent className="p-4">
                        <div className="flex items-center justify-between">
                            <div>
                                <p className="text-sm text-gray-500">Total Budget</p>
                                <p className="text-2xl font-bold">$12.5K</p>
                            </div>
                        </div>
                    </CardContent>
                </Card>
            </div>

            {/* Campaigns List */}
            <div className="grid grid-cols-1 gap-4">
                {campaigns.map((campaign) => (
                    <CampaignCard key={campaign.id} campaign={campaign} />
                ))}
            </div>
        </div>
    );
}

function CampaignCard({ campaign }: { campaign: any }) {
    const getStatusColor = (status: string) => {
        switch (status) {
            case 'active': return 'bg-green-500';
            case 'draft': return 'bg-gray-500';
            case 'paused': return 'bg-yellow-500';
            case 'completed': return 'bg-blue-500';
            default: return 'bg-gray-500';
        }
    };

    return (
        <Card>
            <CardContent className="p-6">
                <div className="flex justify-between items-start">
                    <div className="flex-1">
                        <div className="flex items-center gap-3 mb-2">
                            <h3 className="text-xl font-semibold">{campaign.name}</h3>
                            <Badge className={getStatusColor(campaign.status)}>
                                {campaign.status}
                            </Badge>
                        </div>

                        {campaign.description && (
                            <p className="text-gray-600 mb-4">{campaign.description}</p>
                        )}

                        <div className="grid grid-cols-4 gap-4 text-sm">
                            <div>
                                <span className="text-gray-500">Budget:</span>
                                <span className="ml-2 font-medium">${campaign.budget}</span>
                            </div>
                            <div>
                                <span className="text-gray-500">Allocated:</span>
                                <span className="ml-2 font-medium">${campaign.budgetAllocated}</span>
                            </div>
                            <div>
                                <span className="text-gray-500">Outreach Sent:</span>
                                <span className="ml-2 font-medium">24</span>
                            </div>
                            <div>
                                <span className="text-gray-500">Responses:</span>
                                <span className="ml-2 font-medium">5 (21%)</span>
                            </div>
                        </div>
                    </div>

                    <div className="flex gap-2">
                        <Button variant="outline" size="sm">View Details</Button>
                        <Button size="sm">Send Outreach</Button>
                    </div>
                </div>
            </CardContent>
        </Card>
    );
}
```

---

## Celery Workers

```python
# workers/outreach_automation.py

from celery import shared_task
from datetime import datetime, timedelta
from prisma import Prisma

@shared_task
def send_scheduled_outreach():
    """
    Send scheduled outreach messages
    Runs every hour
    """
    prisma = Prisma()
    prisma.connect()

    try:
        # Get outreach scheduled for now or earlier
        now = datetime.utcnow()

        scheduled = prisma.influenceroutreach.find_many(
            where={
                'status': 'scheduled',
                'nextFollowUpAt': {'lte': now}
            },
            take=50,  # Process 50 at a time
        )

        for outreach in scheduled:
            try:
                # Send via OutreachManager
                # (This would be imported and used here)
                pass
            except Exception as e:
                print(f"Error sending outreach {outreach.id}: {e}")

        print(f"Processed {len(scheduled)} scheduled outreach")

    finally:
        prisma.disconnect()

@shared_task
def send_auto_follow_ups():
    """
    Send automated follow-ups
    Runs daily at 9 AM
    """
    prisma = Prisma()
    prisma.connect()

    try:
        now = datetime.utcnow()

        # Get outreach needing follow-up
        need_follow_up = prisma.influenceroutreach.find_many(
            where={
                'status': {'in': ['sent', 'opened']},
                'nextFollowUpAt': {'lte': now},
                'followUpCount': {'lt': 2},  # Max 2 follow-ups
            },
            take=100,
        )

        for outreach in need_follow_up:
            try:
                # Send follow-up via OutreachManager
                pass
            except Exception as e:
                print(f"Error sending follow-up for {outreach.id}: {e}")

        print(f"Sent {len(need_follow_up)} follow-ups")

    finally:
        prisma.disconnect()

@shared_task
def process_email_webhooks():
    """
    Process email tracking webhooks (opens, clicks)
    Runs every 5 minutes
    """
    # This would process queued webhook events
    # from SendGrid/AWS SES and update outreach records
    pass
```

---

## Definition of Done

### Technical Requirements
- [ ] Email service integrated and sending
- [ ] DM integration working for Instagram and TikTok
- [ ] AI message personalization functional
- [ ] Template management system complete
- [ ] Campaign creation and management working
- [ ] Reply detection and categorization operational
- [ ] Contract generation working
- [ ] Follow-up automation functional
- [ ] Tracking (opens, clicks, replies) working
- [ ] API endpoints tested

### Quality Checks
- [ ] Personalized messages sound natural and authentic
- [ ] Reply categorization accurate (>85%)
- [ ] Follow-ups sent at appropriate intervals
- [ ] Contracts include all required terms
- [ ] Email deliverability high (>95%)
- [ ] No spam flags triggered

### Documentation
- [ ] API endpoint documentation complete
- [ ] Email/DM best practices documented
- [ ] Template creation guide written
- [ ] Campaign setup workflow documented

---

## Testing Checklist

### Unit Tests
- [ ] Message personalizer tests
- [ ] Reply categorizer tests
- [ ] Contract generator tests
- [ ] Template variable replacement tests

### Integration Tests
- [ ] End-to-end outreach flow
- [ ] Bulk outreach processing
- [ ] Reply webhook handling
- [ ] Follow-up automation
- [ ] Contract PDF generation

### Platform Tests
- [ ] Email delivery (SendGrid/SES)
- [ ] Instagram DM sending
- [ ] TikTok DM sending
- [ ] Tracking pixel functionality
- [ ] Webhook processing

### UI Tests
- [ ] Campaign creation wizard
- [ ] Template editor
- [ ] Outreach inbox
- [ ] Contract review

---

## Success Metrics

### Outreach Performance
- **Target:** > 15% response rate
- **Target:** < 2% bounce rate
- **Target:** > 20% open rate (emails)

### Personalization Quality
- **Target:** > 80% of messages sound personalized (user survey)
- **Target:** All template variables correctly replaced

### Campaign Efficiency
- **Target:** < 10 minutes to set up campaign
- **Target:** Bulk outreach to 50+ influencers in < 5 minutes

### Reply Handling
- **Target:** > 85% reply categorization accuracy
- **Target:** < 1 hour average reply processing time

### Automation
- **Target:** Follow-ups sent within 1 hour of scheduled time
- **Target:** 100% of scheduled messages sent

---

## Next Sprint Preview

### Phase 4 - Sprint 6: Integration & Analytics

**Focus:** Integrate SEO and Influencer systems, build unified marketing dashboard, and implement ROI tracking.

**Key Features:**
- Unified marketing dashboard
- Cross-system analytics
- ROI attribution
- Performance comparison (SEO vs Influencer)
- Budget optimization recommendations
- Reporting and exports

**Prerequisites:**
- Phase 4 Sprints 1-5 completed
- All systems operational
- Analytics data collecting

**Expected Duration:** 2 weeks
