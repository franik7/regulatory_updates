# RegWave
**Regulatory Intelligence Platform for Financial Services Compliance**

RegWave aggregates, classifies, and summarizes regulatory updates from major US regulators — 
helping compliance teams stay current without manually monitoring dozens of sources.

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

> **💰 Cost: $0 under current free-tier limits. The system is designed to operate entirely within free tiers, though higher volume or API usage may require upgrades.**

---

## 🏗️ Architecture

```
8 Regulatory Agencies → 10 Workflows
        ↓
n8n Workflows (10 source flows + 1 error handler)
   ├── Manual crawling & parsing (9 flows)
   └── Tavily search API (1 flow — FFIEC only)
        ↓
AI Classification & Summarization
(Groq llama-3.3-70b + OpenRouter Gemma fallback)
        ↓
Supabase (PostgreSQL Database)
        ↓
Caddy Reverse Proxy (HTTPS + auto SSL)
        ↓
Custom Domain (HTTPS secured, no port number)
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

The platform monitors **8 major U.S. regulatory agencies** across **10 automated workflows**. FDIC and the Federal Reserve each have two separate feeds (news and speeches), resulting in 10 workflows from 8 agencies:

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
- Fetches and parses regulatory content from all 8 agencies across 10 workflows
- Calls AI APIs for classification and summarization
- Checks Supabase for duplicates before processing
- Writes processed articles to the database
- Sends Telegram alerts when any workflow fails

**Deployment:** Self-hosted in Docker on Oracle Cloud free tier VPS, accessed securely via Caddy reverse proxy over HTTPS.

---

### 🔹 Caddy — Reverse Proxy & HTTPS
**What it is:** A lightweight, open-source web server that automatically handles SSL certificate provisioning and renewal.

**What it does in this project:**
- Sits in front of n8n and handles all incoming HTTPS traffic
- Automatically obtained a free SSL certificate from Let's Encrypt on first startup
- Auto-renews the certificate before expiry — zero manual maintenance ever
- Proxies all HTTPS requests to n8n running locally on port 5678
- Port 5678 is no longer exposed publicly — all traffic routes through Caddy on ports 80/443

**Why Caddy instead of Nginx:**
- Zero-config SSL — just specify the domain name and it handles everything
- No manual certificate renewal commands
- Minimal resource usage (~13MB RAM)

**Entire Caddy configuration:**
```
your-subdomain.your-domain.com {
    reverse_proxy localhost:5678
}
```
That's the complete config. Caddy handles SSL issuance, renewal, and proxying automatically.

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

**Note on credential security:** The Supabase **anon key** is used inside n8n workflow code for duplicate-checking (read-only). This is safe — the anon key is designed for client-side use and security is enforced via Row Level Security policies. The **service role key** (full admin access) is stored only in n8n's encrypted credentials manager and never hardcoded in workflow code.

---

### 🔹 Oracle Cloud — Backend Hosting
**What it is:** Oracle Cloud Infrastructure's Always Free tier provides a permanent, always-on virtual server.

**Specs used:**
- VM.Standard.E2.1.Micro (1 OCPU, 1GB RAM)
- Ubuntu 22.04 LTS
- 50GB boot volume
- 4GB swap file

**What runs on it:**
- n8n (Docker container)
- Caddy (reverse proxy + SSL termination)
- All 10 source workflows + error handler

---

### 🔹 Netlify — Frontend Hosting
**What it is:** A free static site hosting platform with GitHub auto-deployment.

**What it hosts:**
- The compliance review web interface
- Connected directly to Supabase for data and auth
- Auto-deploys on every GitHub push

**Important:** Netlify connects directly to Supabase — it does NOT connect to n8n. n8n writes data to Supabase on its own schedule. Netlify independently reads from and writes decisions to the same Supabase database.

---

### 🔹 AI Models — Classification & Summarization

**Primary:** Groq API — `llama-3.3-70b-versatile`
- Fast inference, free tier with generous limits
- Primary model for all 10 workflows

**Fallback:** OpenRouter — `google/gemma-3-12b-it:free`
- Activates automatically if Groq hits rate limits
- Different company/infrastructure = fully independent rate limits
- Free tier on OpenRouter

**Why dual providers:** If Groq experiences rate limits or downtime, OpenRouter automatically takes over — no manual intervention needed. Both providers serve large, capable models for free.

**What the AI does for each article:**
1. Classifies into exactly ONE category:
   - BSA/AML/KYC Fines & Enforcement
   - Regulatory Updates
   - Regulator Speeches & Testimony
   - Industry News & Trends
2. Writes a 2-sentence plain-English summary for compliance officers

**Enhanced AI Classification Prompt v2:**
Rewrote the n8n classification prompt with explicit decision rules, keyword anchors, and a priority hierarchy to reduce misclassification. Added a KEY TEST heuristic ("Does this article tell a compliance officer they must do something new?") to better distinguish true regulatory updates from program announcements, statistics, and industry news.

Manual reclassification — users can now correct AI-assigned categories from the UI; original AI classification is retained in the database for audit purposes and flagged visually in the review queue

---

## ⚠️ Failure Handling

The system is designed to fail safely and visibly:

- Duplicate articles → skipped automatically  
- AI failures → fallback provider is triggered  
- Source errors → workflow fails and sends Telegram alert  
- Parsing failures → article is skipped (no partial data stored)  

A global error workflow ensures **no silent failures**.

---

### 🔹 Telegram — Error Monitoring
A dedicated n8n error handler workflow sends real-time alerts to a Telegram bot whenever any workflow fails, including:
- Which workflow failed
- Which node failed
- The error message

This ensures failures are immediately visible and actionable.

---

## ⚠️ Known Limitations

- URL-based deduplication assumes one URL per unique article  
- HTML parsing depends on regulator website structure (can break if layout changes)  
- FFIEC relies on Tavily search rather than direct scraping  
- AI classification is not deterministic and requires human validation  

These are acceptable trade-offs for a lightweight, free-tier system.

---

## 🔄 Workflows (n8n) — Detailed

There are **10 source workflows** plus **1 error handler**, all running in a single n8n instance.

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
- `FDIC Speeches - Classify` — AI Agent with Groq primary + OpenRouter fallback
- `FDIC Speeches - Output Parser` — Structured JSON output parser
- `FDIC Speeches - Save to Supabase` — Inserts row into `items` table

---

### Workflow 2 — FDIC Press Releases
**Trigger:** 5:05 AM ET daily
**Source:** `https://www.fdic.gov/news/press-releases?year={current_year}`
**Method:** Same pattern as FDIC Speeches but targets the press releases section

---

### Workflow 3 — FinCEN
**Trigger:** 5:10 AM ET daily
**Source:** FinCEN news section on Treasury website
**Method:** HTTP fetch → HTML extraction → parsing

---

### Workflow 4 — OCC
**Trigger:** 5:15 AM ET daily
**Source:** OCC news RSS feed
**Method:** RSS feed → XML to JSON → filter valid articles → fetch full content → clean

**RSS-specific nodes:**
- `OCC - Parse RSS Feed` — Converts RSS XML to structured JSON
- `OCC - Filter Valid Articles` — Removes items without proper dates or URLs

---

### Workflow 5 — Treasury
**Trigger:** 5:20 AM ET daily
**Source:** U.S. Treasury press releases
**Method:** HTML crawl → CSS extraction → parse

---

### Workflow 6 — FINRA
**Trigger:** 5:25 AM ET daily
**Source:** FINRA rules guidance and notices
**Method:** HTML crawl → extract → parse → clean

---

### Workflow 7 — Federal Reserve News
**Trigger:** 5:30 AM ET daily
**Source:** Federal Reserve news RSS
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
- `FFIEC - Tavily Search` — Queries Tavily for FFIEC press releases for the current year
- `FFIEC - Parse Search Results` — Filters results to current year, valid URLs, excludes homepage entries
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

The system uses a **two-layer deduplication strategy**:

1. **Pre-processing deduplication (workflow level)**  
   Each workflow checks whether an article URL already exists before sending it to AI.  
   This prevents unnecessary API calls and reduces token usage.

2. **Database-level protection (Supabase)**  
   The `url` field is enforced as unique, ensuring no duplicate records are ever stored, even if multiple workflows run concurrently.

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
**Data:** Direct Supabase API connection (does NOT connect to n8n)

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

#### Step 1 — Create Oracle Cloud Account
- Sign up at cloud.oracle.com (free tier)
- Select **VM.Standard.E2.1.Micro** (Always Free eligible)
- Choose **Ubuntu 22.04**
- Generate SSH key pair and save both keys securely
- Note your instance's public IP address

#### Step 2 — Connect via SSH
```bash
ssh -i "your-key.pem" ubuntu@YOUR_ORACLE_IP
```

#### Step 3 — Update the server first
```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
sudo apt clean
```

#### Step 4 — Install Docker
```bash
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu
# Disconnect and reconnect SSH for the group change to take effect
exit
ssh -i "your-key.pem" ubuntu@YOUR_ORACLE_IP
```

#### Step 5 — Configure 4GB Swap (critical for 1GB RAM)
```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

#### Step 6 — RAM Optimization (run in order)

**Remove Snap** — biggest win, frees ~150MB RAM:
```bash
sudo systemctl stop snapd
sudo systemctl disable snapd
sudo apt remove --purge snapd -y
sudo apt autoremove -y
sudo apt-mark hold snapd
```
> ⚠️ This also removes oracle-cloud-agent which runs via snap. Oracle's monitoring dashboard stops working but your instance continues running 100% normally. For a self-managed server this is acceptable and worth the RAM savings.

**Disable multipathd** — frees ~27MB, not needed on single-disk VPS:
```bash
sudo systemctl stop multipathd
sudo systemctl disable multipathd
sudo systemctl mask multipathd
```

**Disable crash reporters** — frees ~15MB, pointless on a server:
```bash
sudo systemctl stop apport
sudo systemctl disable apport
sudo systemctl stop whoopsie
sudo systemctl disable whoopsie
```

**Limit journald log memory** — prevents logs from eating unbounded RAM:
```bash
sudo nano /etc/systemd/journald.conf
# Find and set these two lines:
# SystemMaxUse=50M
# RuntimeMaxUse=50M
sudo systemctl restart systemd-journald
```

> **Important — do NOT limit Node.js memory:** Adding `--max-old-space-size` flags to cap n8n's memory causes workflow crashes instead of graceful degradation. The 4GB swap file already handles memory overflow safely — this is the correct approach.

#### Step 7 — Set up RAM logging for monitoring
```bash
echo "*/1 * * * * ubuntu free -m >> /home/ubuntu/ram_log.txt" | sudo tee -a /etc/crontab
sudo systemctl restart cron
```
Check RAM history anytime:
```bash
cat ~/ram_log.txt
```

#### Step 8 — Deploy n8n via Docker
```bash
mkdir n8n && cd n8n
nano docker-compose.yml
```

Paste this configuration:
```yaml
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

Save with `Ctrl+X → Y → Enter`, then:
```bash
docker-compose up -d
docker ps  # verify n8n shows "Up"
```

#### Step 9 — Open Ubuntu's internal firewall
```bash
sudo iptables -I INPUT -p tcp --dport 5678 -j ACCEPT
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

#### Step 10 — Open Oracle Cloud Security List ports
In Oracle Cloud Console → your instance → Networking → Subnet → Default Security List → Add Ingress Rules:

| Port | Protocol | Purpose |
|------|----------|---------|
| 5678 | TCP | Initial n8n access (can be closed after Caddy is set up) |
| 80 | TCP | Required for Let's Encrypt SSL certificate verification |
| 443 | TCP | HTTPS traffic via Caddy |

#### Step 11 — Set up Custom Domain

**Create DNS A record:**
- In your domain registrar or hosting control panel (e.g., cPanel → Zone Editor)
- Add an **A record**: `your-subdomain.your-domain.com` → your Oracle public IP
- Wait 5–30 minutes for DNS propagation

#### Step 12 — Install and Configure Caddy (HTTPS)

**Install Caddy:**
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy -y
```

**Configure Caddy:**
```bash
sudo bash -c 'cat > /etc/caddy/Caddyfile << EOF
your-subdomain.your-domain.com {
    reverse_proxy localhost:5678
}
EOF'
```

**Open ports 80 and 443 in Ubuntu firewall:**
```bash
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 443 -j ACCEPT
sudo netfilter-persistent save
```

**Start Caddy:**
```bash
sudo systemctl restart caddy
sudo systemctl enable caddy
sudo systemctl status caddy
```
Look for `validations succeeded` and `successfully downloaded` in the logs — this confirms Let's Encrypt issued the free SSL certificate automatically.

**Verify HTTPS is working:**
- Open `https://your-subdomain.your-domain.com` in a browser
- You should see the n8n login screen with a 🔒 padlock and no port number

**Optional but recommended — close port 5678 from public access:**
```bash
sudo iptables -D INPUT -p tcp --dport 5678 -j ACCEPT
sudo netfilter-persistent save
```
Also remove the port 5678 Ingress Rule from Oracle Cloud Security List. Caddy now handles all traffic — port 5678 no longer needs to be public.

#### Step 13 — Import workflows and configure credentials
- Access n8n at your HTTPS domain
- Go to **Settings → Credentials** and add:
  - Supabase (URL + service role key)
  - Groq API key
  - OpenRouter API key
  - Tavily API key
  - Telegram bot token
- Import each workflow JSON via **Workflows → Import from file**
- Activate each workflow with the toggle switch

---

### Frontend (Netlify)

1. Push frontend code to GitHub
2. Connect repo to Netlify
3. Set environment variables in Netlify dashboard:
   - `SUPABASE_URL`
   - `SUPABASE_ANON_KEY`
4. Deploy — Netlify auto-builds on every GitHub push

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

### Server RAM Optimizations (Oracle Free Tier)

| Optimization | RAM Saved | Safe? | Notes |
|---|---|---|---|
| Remove Snap | ~150MB | ✅ | Largest single win |
| Remove oracle-cloud-agent | ~50MB | ✅ | Removed automatically with snap; Oracle metrics stop but instance runs fine |
| Disable multipathd | ~27MB | ✅ | Multi-path disk manager, not needed on single-disk VPS |
| Disable apport + whoopsie | ~15MB | ✅ | Ubuntu crash reporters, useless on a server |
| Limit journald to 50MB | variable | ✅ | Prevents log files from consuming unbounded RAM |
| 4GB swap file | prevents crashes | ✅ | Emergency RAM overflow buffer; n8n spills into swap instead of crashing |
| `restart: always` in Docker | reliability | ✅ | n8n auto-restarts after any crash or server reboot |
| RAM logging cron job | visibility | ✅ | Logs `free -m` every minute to `~/ram_log.txt` |
| Do NOT limit Node.js memory | n/a | ✅ | Swap handles overflow; capping Node causes workflow crashes |

**Before vs. after optimization:**
```
Before: 544MB used | 256MB available | 352MB swap in use
After:  283MB used | 522MB available |   0MB swap in use
```

### Network & Security Improvements

| Change | Benefit |
|---|---|
| Caddy reverse proxy | Handles SSL automatically, hides n8n port from public internet |
| Let's Encrypt SSL certificate | Free, auto-renewing — zero maintenance |
| Port 5678 closed publicly | All traffic routes through HTTPS on port 443 only |
| Custom domain with HTTPS | Accessible from corporate networks that block raw IP addresses |

### Workflow Optimizations

- **Staggered triggers** — workflows run every 5 minutes from 5:00–5:45 AM to avoid simultaneous API hammering
- **Deduplication before AI** — AI is only called for genuinely new articles (saves API costs and rate limits)
- **Dynamic year in URLs** — all listing URLs use `new Date().getFullYear()` expression so no code changes are needed each January
- **Dual AI provider** — Groq primary + OpenRouter fallback prevents rate limit failures; two independent providers with separate limits
- **Anon key for deduplication** — uses Supabase anon key (safe for read-only operations) in workflow code; service role key stored only in n8n encrypted credentials

---

## 🔗 Integrations Summary

| Integration | Purpose | Cost |
|-------------|---------|------|
| **n8n** (self-hosted) | Workflow automation engine | Free |
| **Oracle Cloud** | VPS hosting for n8n | Free (Always Free tier) |
| **Caddy** | Reverse proxy + automatic SSL | Free (open source) |
| **Let's Encrypt** | SSL certificate authority | Free |
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
- ✅ **8 agencies, 10 workflows** — comprehensive U.S. financial regulator coverage
- ✅ **HTTPS secured** — custom domain with auto-renewing SSL certificate via Let's Encrypt
- ✅ **Workplace accessible** — HTTPS domain works on corporate networks that block raw IP addresses
- ✅ **AI-powered** — automatic classification and plain-English summaries
- ✅ **Duplicate-free** — built-in deduplication prevents redundant data
- ✅ **Fault-tolerant** — dual AI providers + Telegram error alerts + Docker auto-restart
- ✅ **Audit-ready** — all decisions stored with timestamps and user attribution
- ✅ **Role-based** — structured Doer → Reviewer workflow
- ✅ **Real-time monitoring** — Telegram alerts for any workflow failures
- ✅ **RAM optimized** — server reduced from 544MB to 283MB RAM usage through targeted service removal

---

## 📊 Data Volume

As of early 2026:
- **130+ articles** collected and processed
- Growing daily across all 8 agencies (10 workflows)
- All stored in Supabase with full text, AI summaries, and categories

---

## 🚀 Future Improvements

- Email/SMS digest alerts for new high-priority items
- Analytics dashboard (article volume by source, category trends)
- Multi-tenant support for multiple compliance teams
- Integration with existing compliance management systems
- Mobile app for on-the-go review
- Expanded international regulator coverage
- Upgrade Oracle VM to A1.Flex (4 OCPU / 24GB RAM) when free tier capacity becomes available

---

## 📋 Business Summary

This platform continuously monitors all major U.S. financial regulators, automatically collects and AI-processes regulatory updates every morning, and routes them through a structured compliance review workflow. Analysts (Doers) review AI-summarized articles and make initial decisions; Reviewers validate those decisions. Every action is permanently stored for audit purposes.

The system is secured with HTTPS via a custom domain — accessible from any network including corporate environments — and eliminates hours of daily manual monitoring across 8 agencies at zero ongoing cost.

---

*Built with n8n · Supabase · Oracle Cloud · Caddy · Let's Encrypt · Netlify · Groq · OpenRouter · Tavily · Telegram*
