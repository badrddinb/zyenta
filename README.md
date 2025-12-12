# 🚀 Zyenta (Project Agentis)

**The World's First "Zero-Effort" Autonomous AI-Commerce Engine.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) [![Status: Alpha](https://img.shields.io/badge/Status-Alpha-red)]() [![Stack: T3](https://img.shields.io/badge/Stack-Next.js%20%7C%20FastAPI%20%7C%20K8s-blue)]()

## 📖 Executive Summary

**Zyenta** is an agentic SaaS platform that democratizes e-commerce by acting as a fully autonomous digital workforce. It is not just a store builder; it is a **CEO, COO, and CMO in the cloud.**

Unlike traditional platforms where the user must design, write, and manage, Zyenta operates on a **"Zero-Effort"** principle. From a single user prompt (e.g., *"Cyberpunk home decor"*), Zyenta's swarm of AI agents autonomously:

1.  **Builds** a brand and sources high-potential dropshipping products.
2.  **Generates** professional media (images & video) and copywriting.
3.  **Operates** the store 24/7 (inventory sync, fulfillment).
4.  **Markets** the brand by plugging into the user's external channels (TikTok, Meta, Google).

---

## ⚡ The 3 Core Pillars (Architecture)

Zyenta is built on three distinct, interacting engines powered by a microservices architecture.

### 🏛 Pillar 1: The Genesis Engine (Creator)
*The creative factory that builds the asset.*

* **🤖 Brand Agent:** Generates brand identity, logo, color palette, and deploys a headless storefront.
* **🕵️ Product Scout Agent:** Connects to supplier APIs (AliExpress, CJ Dropshipping, Spocket) to identify 10-20 "winning" products based on real-time sales data.
* **🎥 AI Media Studio Agent:**
    * **Image Processing:** Removes watermarks/backgrounds from supplier photos. Generates professional, consistency-tuned lifestyle backgrounds (e.g., "marble desk," "cozy living room").
    * **Video Generation:** Creates 15-30s engaging product showcase videos and animations for social media and ads.
* **✍️ Copywriter Agent:** Generates SEO-optimized product titles, persuasive long-form descriptions, meta tags, and legal pages (Privacy/Refund).

### ⚙️ Pillar 2: The Operations Engine (COO)
*The automation layer that runs the business 24/7.*

* **🔄 Inventory & Price Sync Agent:** A high-frequency background worker that monitors supplier stock levels. Automatically updates the storefront to prevent out-of-stock sales and adjusts pricing to maintain margins.
* **📦 Fulfillment Orchestrator:** Event-driven automation:
    1.  Detects `payment.success` webhook.
    2.  Autonomously places the order with the supplier.
    3.  Pays using the client's pre-funded account.
    4.  Retrieves tracking numbers and syncs them to the customer.
* **💳 Payment Orchestrator:** Manages the integration where the **Client is the Merchant of Record (MoR)**. Wires the storefront directly to the client's Stripe/PayPal API keys.

### 📈 Pillar 3: The Growth Engine (CMO)
*The "Pluggable" Marketing Hub. Clients connect their own accounts (BYOC - Bring Your Own Channel), and agents take over.*

* **📱 Organic Social Agent:**
    * **Connectors:** TikTok, Instagram, Pinterest, X.
    * **Action:** Repurposes Pillar 1 media (Videos/Images) into a 30-day content calendar. Posts autonomously to build brand presence.
* **💰 Paid Ads Agent:**
    * **Connectors:** Meta Ads, Google Ads, TikTok Ads.
    * **Action:** Takes a daily budget (e.g., $20). Creates, launches, and A/B tests campaigns. Monitors ROAS (Return on Ad Spend) 24/7 to scale winners and kill losers.
* **📧 Retention Agent:** Connects to Mailchimp/Klaviyo to automate Abandoned Cart and Welcome flows using generated assets.

---

## 🛠 Tech Stack

Zyenta uses an **Event-Driven Microservices** architecture to handle the complexity of concurrent agents.

| Component | Tech Choice | Rationale |
| :--- | :--- | :--- |
| **Frontend** | Next.js 14, Tailwind, Shadcn UI | SEO, Server-Side Rendering, and high-performance dashboard. |
| **AI Backend** | Python (FastAPI) | Native support for PyTorch, LangChain, and Image/Video processing libs. |
| **Ops Backend** | Node.js (NestJS) or Go | Handling high-concurrency webhooks and 24/7 sync jobs. |
| **Database** | PostgreSQL | Robust relational data for multi-tenant user/store schemas. |
| **Message Bus** | RabbitMQ / Kafka | Critical for decoupling agents (e.g., `StoreCreated` event triggers `MarketingAgent`). |
| **Queue/Cache** | Redis | Job queues for long-running AI tasks (Video Gen) and caching. |
| **Infra** | Docker, K8s, Terraform | Scalable container orchestration. |

---

## 🔁 The User Workflow

1.  **Onboarding:** User signs up and connects their Stripe (for revenue) and Supplier Account (for fulfillment).
2.  **Prompt:** User enters a single niche: *"Minimalist Japanese Office Supplies."*
3.  **Genesis (T+10 mins):** Zyenta scouts products, generates media/video, writes copy, and deploys the live store.
4.  **Activation:** User connects marketing channels (e.g., TikTok, Facebook) in the "Growth Hub."
5.  **Autonomy:**
    * **Traffic:** Paid Ads Agent launches campaigns; Social Agent posts content.
    * **Sales:** Orders flow in.
    * **Fulfillment:** Ops Engine buys from supplier and ships to customer.
    * **Profit:** Net profit lands in User's Stripe account.

---

## 🗺️ Roadmap

### Phase 1: The "Creator" MVP 🚧
- [ ] Build Genesis Engine (Scout, Brand, Copy).
- [ ] Build Basic Media Studio (Images only).
- [ ] Implement MoR Payment Architecture (Client Stripe Connect).
- [ ] Launch "Store Builder" tier.

### Phase 2: The "Ops" Foundation
- [ ] Implement 24/7 Inventory/Price Sync (The "Zero-Effort" guarantee).
- [ ] Build Automated Fulfillment Orchestrator.
- [ ] Add Video Generation to Media Studio.

### Phase 3: The "Growth" Hub (Current Focus)
- [ ] Build "Pluggable" OAuth system for external platforms.
- [ ] Launch **Social Agent** (Organic TikTok/Insta).
- [ ] Launch **Ads Agent** (Meta/Google with Budget Management).

### Phase 4: Expansion
- [ ] SEO Blog Writing Agent.
- [ ] Influencer Outreach Agent.

---

## ⚠️ Liability & Disclaimer

**Zyenta is a SaaS Platform, not a Retailer.**
* **Merchant of Record:** The End User is the legal Merchant of Record.
* **Ad Spend:** All ad budgets are charged directly to the User's ad accounts.
* **Product Liability:** Zyenta facilitates the data connection to suppliers; the User acts as the retailer.

---

## 💻 Repository Structure

```bash
zyenta/
├── apps/
│   ├── dashboard/          # Next.js Client Dashboard
│   ├── storefront/         # Headless Storefront Template
├── services/
│   ├── genesis-engine/     # Python: Brand, Copy, Scout Agents
│   ├── media-studio/       # Python: Image/Video Generation Pipeline
│   ├── ops-engine/         # Node.js: Fulfillment, Sync, Webhooks
│   ├── growth-engine/      # Python: Social & Ads Managers
├── packages/
│   ├── database/           # Prisma/Postgres Schema
│   ├── queue/              # Redis/BullMQ config
│   ├── ui/                 # Shared React Components
├── docs/                   # Project Documentation
│   ├── PHASE1_EXECUTION_PLAN.md
│   ├── PHASE2_EXECUTION_PLAN.md
│   ├── PHASE3_EXECUTION_PLAN.md
│   ├── PHASE4_EXECUTION_PLAN.md
│   ├── PHASE2_SPRINT1_INVENTORY_SYNC_ACTION_PLAN.md
│   ├── SPRINT1_FOUNDATION_ACTION_PLAN.md
│   ├── SPRINT2_GENESIS_ENGINE_ACTION_PLAN.md
│   ├── SPRINT3_MEDIA_STUDIO_ACTION_PLAN.md
│   ├── SPRINT4_DASHBOARD_MVP_ACTION_PLAN.md
│   ├── SPRINT5_STOREFRONT_PAYMENTS_ACTION_PLAN.md
│   └── SPRINT6_INTEGRATION_POLISH_ACTION_PLAN.md
├── infra/                  # Terraform & K8s Manifests
├── LICENSE                 # MIT License
└── README.md
```

## 🤝 Contributing

We are currently in Closed Alpha. To request access to the codebase or contribute to the core agent logic, please contact the maintainers.

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

Built by the Zyenta Team.
