# 📊 Regulatory Updates AI Platform
### BSA / AML / KYC / ABC / Fraud & Regulatory Intelligence System

---

## 🧭 Overview

This platform **fully automates** the collection, AI-processing, and structured review of regulatory updates and enforcement actions from all major U.S. financial regulators. It monitors for:

- **BSA** — Bank Secrecy Act
- **AML** — Anti-Money Laundering
- **KYC** — Know Your Customer
- **ABC** — Anti-Bribery & Corruption
- **Fraud** fines & enforcement actions
- **Regulatory developments** across financial services

The system runs **completely automatically**, every morning before business hours, and routes processed data through a structured compliance review workflow with defined user roles.

> **💰 Cost: $0. The entire stack runs on free tiers — permanently.**

---

## 🏗️ Architecture

```
9 Regulatory Sources
        ↓
n8n Workflows (11 automated flows)
   ├── Manual crawling & parsing (10 flows)
   └── Tavily search API (1 flow — FFIEC only)
        ↓
AI Classification & Summarization
(Groq llama-3.3-70b + OpenRouter Gemma fallback)
        ↓
Supabase (PostgreSQL Database)
        ↓
Web Frontend (Netlify)
        ↓
User Review Workflow
(Doer → Reviewer → Stored decisions)
        ↓
Telegram Error Alerts (real-time)
```

---

## 🏛️ Regulatory Sources

The platform monitors **9 major U.S. regulatory bodies**, covering both news and speeches where applicable:

| Source | Type | Collection Method |
|--------|------|-------------------|
| **FDIC** | News | Manual crawl |
| **FDIC** | Speeches | Manual crawl |
| **FinCEN** | News & Guidance | Manual crawl |
| **OCC** | News & Enforcement | RSS + parse |
| **U.S. Treasury** | News | Manual crawl |
| **FINRA** | News & Enforcement | Manual crawl |
| **Federal Reserve** | News | RSS + parse |
| **Federal Reserve** | Speeches | RSS + parse |
| **SEC** | Press Releases | RSS + parse |
| **FFIEC** | Press Releases | Tavily search |

> **Note:** Tavily is used **exclusively** for FFIEC. All other sources use direct crawling, HTML parsing, and content cleaning pipelines.

---

## ⚙️ Core Components

### 🔹 n8n — Automation Engine
**What it is:** An open-source workflow automation platform (like Zapier but self-hosted and free).

**What it does in this project:**
- Triggers all workflows on a daily schedule (5:00–5:45 AM ET, staggered every 5 minutes)
- Fetches and parses regulatory content from all 9 sources
- Calls AI APIs for classification and summarization
- Checks Supabase for duplicates before processing
- Writes processed articles to the database
- Sends Telegram alerts when any workflow fails

**Deployment:** Self-hosted in Docker on Oracle Cloud free tier VPS.

---

### 🔹 Tavily — AI-Optimized Search (FFIEC only)
**What it is:** A search engine built specifically for AI/LLM workflows. Unlike traditional web scraping, Tavily returns clean, structured JSON results.

**Why used for FFIEC specifically:**
- FFIEC's website structure is complex to scrape reliably
- Tavily returns pre-structured results (title, URL, date, content) without custom parsing logic
- Reduces code complexity for this one source

**What it returns:**
```json
{
  "title": "FFIEC Releases Revised BSA/AML Examination Manual",
  "url": "https://www.ffiec.gov/press/...",
  "published_date": "2026-04-01",
  "content": "..."
}
```

---

### 🔹 Supabase — Database & Authentication
**What it is:** An open-source Firebase alternative built on PostgreSQL.

**What it stores:**
- All processed regulatory articles
- AI-generated categories and summaries
- Full article text
- Source metadata (regulator, date, URL)
- User decisions (approve/decline/unsure) with notes
- Decision history and timestamps

**Why Supabase:**
- Free tier is generous (500MB database, unlimited API calls)
- Built-in authentication for multi-user login
- Row-Level Security (RLS) for role-based data access
- Real-time API accessible from the frontend

---

### 🔹 Oracle Cloud — Backend Hosting
**What it is:** Oracle Cloud Infrastructure's Always Free tier provides a permanent, always-on virtual server.

**Specs used:**
- VM.Standard.E2.1.Micro (1 OCPU, 1GB RAM)
- Ubuntu 22.04 LTS
- 50GB boot volume
- 4GB swap file (configured for low-RAM optimization)

**Optimizations applied:**
- Disabled unnecessary Ubuntu services
- Configured 4GB swap to extend effective memory
- Docker with `restart: always` for automatic recovery
- RAM usage logging via cron job for monitoring
- UptimeRobot monitoring for uptime alerts

**What runs on it:**
- n8n (Docker container)
- All 11 automated workflows

---

### 🔹 Netlify — Frontend Hosting
**What it is:** A free static site hosting platform with GitHub auto-deployment.

**What it hosts:**
- The compliance review web interface
- Connected directly to Supabase for data and auth
- Auto-deploys on every GitHub push

---

### 🔹 AI Models — Classification & Summarization

**Primary:** Groq API — `llama-3.3-70b-versatile`
- Fast inference
- Free tier with generous limits
- Primary model for all workflows

**Fallback:** OpenRouter — `google/gemma-3-12b-it:free`
- Activates automatically if Groq hits rate limits
- Different provider = independent rate limits
- Free tier on OpenRouter

**What the AI does for each article:**
1. Classifies into exactly ONE category:
   - BSA/AML/KYC Fines & Enforcement
   - Regulatory Updates
   - Regulator Speeches & Testimony
   - Industry News & Trends
2. Writes a 2-sentence plain-English summary for compliance officers

**Prompt structure:**
```
Classify this article into exactly ONE of these categories:
- BSA/AML/KYC Fines & Enforcement
- Regulatory Updates
- Regulator Speeches & Testimony
- Industry News & Trends

Also write a 2-sentence plain-English summary for a compliance officer.

Respond ONLY as valid JSON: {"category": "...", "summary": "..."}
```

---

### 🔹 Telegram — Error Monitoring
A dedicated n8n error handler workflow sends real-time alerts to a Telegram bot whenever any workflow fails, including:
- Which workflow failed
- Which node failed
- The error message

This eliminates silent failures and removes the need to manually check execution logs.

---

## 🔄 Workflows (n8n) — Detailed

There are **11 automated workflows** plus **1 error handler**, all running in a single n8n instance.

---

### Workflow Architecture (all flows share this pattern):

```
Schedule Trigger
      ↓
Fetch Listing Page / RSS Feed / Tavily Search
      ↓
Extract & Parse Rows (HTML / XML / JSON)
      ↓
Filter Duplicates (check Supabase — skip if URL exists)
      ↓
Fetch Full Article Content
      ↓
Merge metadata + article HTML
      ↓
Clean & Normalize Article Text
      ↓
AI Agent (classify + summarize)
      ↓
Save to Supabase
```

---

### Workflow 1 — FDIC Speeches
**Trigger:** 5:00 AM ET daily
**Source:** `https://www.fdic.gov/news/speeches?year={current_year}`
**Method:** HTTP fetch → HTML extraction (CSS selectors) → regex parsing
**Nodes:**
- `FDIC Speeches - Daily Trigger` — Schedule trigger
- `FDIC Speeches - Fetch Listing` — HTTP GET to FDIC speeches page
- `FDIC Speeches - Extract Rows` — HTML node extracts `.views-row` elements
- `FDIC Speeches - Parse Rows` — Code node: extracts title, URL, date via regex; resolves relative URLs; decodes HTML entities
- `FDIC Speeches - Filter Duplicates` — Checks Supabase for existing URLs; skips already-stored articles
- `FDIC Speeches - Fetch Article` — HTTP GET to each individual article URL
- `FDIC Speeches - Merge` — Combines article HTML with metadata from upstream
- `FDIC Speeches - Clean Article` — Code node: strips scripts/styles/SVGs, extracts `<main>` content, removes boilerplate (social share buttons, email addresses, contact sections), normalizes whitespace
- `FDIC Speeches - Classify` — AI Agent with Groq + OpenRouter fallback
- `FDIC Speeches - Output Parser` — Structured JSON output parser
- `FDIC Speeches - Save to Supabase` — Inserts row into `items` table

---

### Workflow 2 — FDIC Press Releases
**Trigger:** 5:05 AM ET daily
**Source:** `https://www.fdic.gov/news/press-releases?year={current_year}`
**Method:** Same as FDIC Speeches but targets press releases section
**Key difference:** URL includes antibot parameters that must be included in the request

---

### Workflow 3 — FinCEN
**Trigger:** 5:10 AM ET daily
**Source:** `https://home.treasury.gov` (FinCEN section)
**Method:** HTTP fetch → HTML extraction → parsing

---

### Workflow 4 — OCC
**Trigger:** 5:15 AM ET daily
**Source:** `https://www.occ.gov` news RSS feed
**Method:** RSS feed → XML to JSON → filter valid articles → fetch full content → clean

**RSS-specific nodes:**
- `OCC - Parse RSS Feed` — Converts RSS XML to structured JSON
- `OCC - Filter Valid Articles` — Removes items without proper dates or URLs

---

### Workflow 5 — Treasury
**Trigger:** 5:20 AM ET daily
**Source:** `https://home.treasury.gov/news/press-releases`
**Method:** HTML crawl → CSS extraction → parse

---

### Workflow 6 — FINRA
**Trigger:** 5:25 AM ET daily
**Source:** `https://www.finra.org/rules-guidance/notices`
**Method:** HTML crawl → extract → parse → clean

---

### Workflow 7 — Federal Reserve News
**Trigger:** 5:30 AM ET daily
**Source:** `https://www.federalreserve.gov` RSS
**Method:** RSS → parse → fetch articles → clean

---

### Workflow 8 — Federal Reserve Speeches
**Trigger:** 5:35 AM ET daily
**Source:** Federal Reserve speeches RSS feed
**Method:** RSS → parse → fetch → clean

---

### Workflow 9 — SEC
**Trigger:** 5:40 AM ET daily
**Source:** SEC press releases RSS feed
**Method:** RSS → parse → fetch → clean

---

### Workflow 10 — FFIEC (Tavily-powered)
**Trigger:** 5:45 AM ET daily
**Source:** FFIEC press releases via Tavily search API
**Method:** Tavily search → structured results → filter new → Tavily extract → AI clean → classify

**Why Tavily here:** FFIEC's website structure makes direct scraping unreliable. Tavily provides pre-structured, clean results directly.

**FFIEC-specific nodes:**
- `FFIEC - Tavily Search` — Queries Tavily for `site:ffiec.gov press releases {current_year}`
- `FFIEC - Parse Search Results` — Filters results to current year, valid URLs, excludes homepage
- `FFIEC - Tavily Extract` — Extracts full article content via Tavily's extract endpoint
- `FFIEC - Filter Duplicates` — Supabase duplicate check
- `FFIEC - Clean Article` — Strips markdown artifacts, removes boilerplate, normalizes text

---

### Workflow 11 — Error Handler (Global)
**Trigger:** Any workflow failure
**What it does:** Catches errors from all 10 source workflows and sends a Telegram message with:
- Workflow name
- Node that failed
- Error message
- Timestamp

---

## 🔁 Deduplication Logic

Every workflow checks Supabase before processing any article:

```javascript
const response = await this.helpers.httpRequest({
  method: 'GET',
  url: `https://[project].supabase.co/rest/v1/items?select=url&url=eq.${url}`,
  headers: { 'apikey': '[anon_key]' }
});

if (Array.isArray(response) && response.length === 0) {
  // New article — process it
}
// Existing article — skip silently
```

Only articles with URLs **not already in the database** are processed further. This prevents duplicate AI calls and database entries.

---

## 🗃️ Database Schema (Supabase)

### `items` table
| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `title` | text | Article title |
| `url` | text | Source URL (unique) |
| `source` | text | Regulator name (e.g., "FDIC") |
| `date_published` | date | Publication date |
| `category` | text | AI-assigned category |
| `summary` | text | AI-generated 2-sentence summary |
| `full_text` | text | Cleaned article content |
| `created_at` | timestamp | When added to DB |

### `decisions` table (frontend)
| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `item_id` | uuid | FK to items |
| `user_id` | uuid | FK to auth.users |
| `decision` | text | approve / decline / unsure |
| `notes` | text | User notes |
| `created_at` | timestamp | Decision timestamp |
| `updated_at` | timestamp | Last modified |

---

## 👥 User Roles & Review Workflow

### 🟢 Doer
- Sees **ALL** incoming articles
- For each article must select:
  - ✅ **Approve** — relevant, should be escalated
  - ❌ **Decline** — not relevant
  - ❓ **Unsure** — needs second opinion
- Can add notes to any item
- Primary focus: approve or unsure (decline is available but secondary)

### 🔵 Reviewer
- Sees **ONLY** items marked `Approve` or `Unsure` by Doer
- Cannot see `Declined` items
- Can add their own notes
- Validates Doer decisions

### 🟣 Approver *(optional extension)*
- Final authority
- Confirms or overrides decisions
- Full audit trail

### Business Rule
For each of the 6 regulatory categories, the Doer must make a decision. The system enforces that `Approve` and `Unsure` items flow to the Reviewer queue. All decisions are stored permanently with timestamps and user attribution.

---

## 💻 Frontend (Web Interface)

**Hosting:** Netlify (free, auto-deploys from GitHub)
**Auth:** Supabase Auth (email/password, multi-user)
**Data:** Direct Supabase API connection

### Features:
- Multi-user login with role-based access
- Dashboard styled similar to FCL compliance interface
- Article list with category badges and AI summaries
- Per-article decision panel (Approve / Decline / Unsure buttons)
- Notes field per article
- Filters by: Category, Status, Date range, Source
- Reviewer queue (filtered view — approved/unsure only)
- Fully responsive (works on mobile)

---

## 📦 Deployment Guide

### Backend (Oracle Cloud)

1. **Create Oracle Cloud account** (free tier)
   - Select VM.Standard.E2.1.Micro (Always Free)
   - Ubuntu 22.04
   - Generate SSH key pair

2. **Connect via SSH:**
```bash
ssh -i "your-key.pem" ubuntu@YOUR_IP
```

3. **Install Docker:**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

4. **Configure swap (critical for 1GB RAM):**
```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

5. **Deploy n8n:**
```yaml
# docker-compose.yml
version: '3'
services:
  n8n:
    image: n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - GENERIC_TIMEZONE=America/New_York
      - N8N_SECURE_COOKIE=false
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

```bash
docker-compose up -d
```

6. **Open firewall port 5678** in Oracle Security Lists

7. **Access n8n** at `http://YOUR_IP:5678`

8. **Import workflows** and configure credentials:
   - Supabase (URL + service role key)
   - Groq API key
   - OpenRouter API key
   - Tavily API key
   - Telegram bot token

---

### Frontend (Netlify)

1. Push frontend code to GitHub
2. Connect repo to Netlify
3. Set environment variables:
   - `SUPABASE_URL`
   - `SUPABASE_ANON_KEY`
4. Deploy — Netlify auto-builds on every push

---

### Database (Supabase)

1. Create free Supabase project
2. Run schema migrations (items + decisions tables)
3. Configure Row Level Security:
   - Doers see all items
   - Reviewers see only approved/unsure items
4. Create user accounts and assign roles

---

## 🔧 Optimizations

### Server Optimizations (Oracle Free Tier)
- **4GB swap file** — extends effective memory from 1GB to 5GB
- **`restart: always`** in Docker — auto-recovers from crashes
- **RAM logging cron** — logs memory usage every minute for monitoring
- Disabled unnecessary Ubuntu services to reduce memory footprint
- UptimeRobot monitoring — alerts if n8n goes offline

### Workflow Optimizations
- **Staggered triggers** — workflows run every 5 minutes from 5:00–5:45 AM to avoid simultaneous API hammering
- **Deduplication before AI** — AI is only called for genuinely new articles (saves API costs and rate limits)
- **Year filter in URLs** — all listing URLs include `?year={currentYear}` dynamically so no code changes needed each January
- **Dual AI provider** — Groq primary + OpenRouter fallback prevents rate limit failures
- **Anon key for deduplication** — uses Supabase anon key (safe for this read-only operation) to avoid hardcoding service role key

---

## 🔗 Integrations Summary

| Integration | Purpose | Cost |
|-------------|---------|------|
| **n8n** (self-hosted) | Workflow automation engine | Free |
| **Oracle Cloud** | VPS hosting for n8n | Free (Always Free tier) |
| **Supabase** | PostgreSQL database + Auth | Free tier |
| **Netlify** | Frontend hosting | Free tier |
| **Groq** | Primary AI (llama-3.3-70b) | Free tier |
| **OpenRouter** | Fallback AI (gemma-3-12b) | Free tier |
| **Tavily** | FFIEC search (1 workflow only) | Free tier |
| **Telegram** | Error monitoring alerts | Free |

---

## 📈 Key Advantages

- ✅ **TOTALLY FREE** — entire stack runs on free tiers permanently
- ✅ **Fully automated** — runs every morning without human intervention
- ✅ **9 regulatory sources** — comprehensive U.S. financial regulator coverage
- ✅ **AI-powered** — automatic classification and plain-English summaries
- ✅ **Duplicate-free** — built-in deduplication prevents redundant data
- ✅ **Fault-tolerant** — dual AI providers + error alerts + auto-restart
- ✅ **Audit-ready** — all decisions stored with timestamps and user attribution
- ✅ **Role-based** — structured Doer → Reviewer workflow
- ✅ **Real-time monitoring** — Telegram alerts for any failures

---

## 📊 Data Volume

As of early 2026:
- **130+ articles** collected and processed
- Growing daily across all 9 regulatory sources
- All stored in Supabase with full text, AI summaries, and categories

---

## 🚀 Future Improvements

- Email/SMS digest alerts for new high-priority items
- Analytics dashboard (article volume by source, category trends)
- Multi-tenant support for multiple compliance teams
- Integration with existing compliance management systems
- Mobile app for on-the-go review
- Expanded international regulator coverage

---

## 📋 Business Summary

This platform continuously monitors all major U.S. financial regulators, automatically collects and AI-processes regulatory updates every morning, and routes them through a structured compliance review workflow. Analysts (Doers) review AI-summarized articles and make initial decisions; Reviewers validate those decisions. Every action is permanently stored for audit purposes.

The system eliminates hours of daily manual monitoring, ensures nothing is missed across 9 regulatory sources, and provides a clear audit trail of all compliance decisions — at zero cost.

---

*Built with n8n · Supabase · Oracle Cloud · Netlify · Groq · OpenRouter · Tavily · Telegram*
