# Lumiere Media & Marketing — Automation Playbook

**Owner:** TJ | **VPS:** 100.123.255.8 | **Goal:** 10-20 dental/cosmetic clients in 6 months
**Current state:** 2 clients, full VPS with Claude Code, PM2, cron, Telegram bot, Puppeteer
**Philosophy:** Automate everything. Free APIs first, pay only when free fails.

---

## 1. Client Acquisition Automation

This is the engine. Without clients, nothing else matters. Every hour spent here has 10x the ROI of anything in sections 2 or 3.

---

### 1.1 Lead Generation

#### 1.1.1 Dental Practice Lead Scraper

**What it does:** Scrapes Google Maps, Jameda, Doctolib, Gelbe Seiten, and Yelp for dental clinics and cosmetic practices across German cities. Extracts: practice name, address, phone, email, website URL, Instagram handle, Facebook page, Google review count, average rating, number of photos, last review date.

**How Claude Code builds it:**
- Python script (`~/products/lumiere/tools/lead-scraper/scraper.py`) using Puppeteer via `pyppeteer` or direct `requests` + `BeautifulSoup` for static pages.
- **Google Maps:** Use Puppeteer to search "Zahnarzt [city]" and "Kosmetikstudio [city]", scroll through results, extract business cards. Google Maps renders JS, so Puppeteer is required. Extract the Google Places data from the page DOM — name, rating, review count, address, phone, website link.
- **Jameda:** `requests` + BS4. Search URL pattern: `https://www.jameda.de/zahnaerzte/[city]/`. Parse result cards for name, address, rating, profile link. Follow profile link to get website/phone.
- **Doctolib:** `requests` + BS4. Search: `https://www.doctolib.de/zahnarzt/[city]`. Parse practitioner cards.
- **Gelbe Seiten:** `requests` + BS4. Search: `https://www.gelbeseiten.de/zahnarzt/[city]`. Parse listings.
- **Instagram discovery:** For each practice with a website, scrape the website for Instagram links (`instagram.com/` in href or page text). If no link found, search Instagram via `https://www.instagram.com/web/search/topsearch/?query=[practice-name]-[city]` (may need login cookies).
- **Email extraction:** Scrape practice websites for `mailto:` links and email patterns. Also check impressum pages (German legal requirement — almost always has contact email).
- Store everything in SQLite database (`~/products/lumiere/data/leads.db`) with tables: `leads`, `lead_scores`, `outreach_history`.
- Cron job: runs weekly on Sunday night, scrapes new cities or re-scrapes existing ones for new practices.
- Output: CSV export + Telegram notification with count of new leads found.

```
# Target cities (start with these, expand later):
CITIES = [
    "berlin", "hamburg", "muenchen", "koeln", "frankfurt",
    "duesseldorf", "stuttgart", "dortmund", "essen", "leipzig",
    "bremen", "dresden", "hannover", "nuernberg", "duisburg",
    # Add 50+ more as you scale
]
```

**Rate limiting strategy:**
- 2-3 second delay between requests
- Rotate User-Agent strings
- Use residential proxy only if blocked (start without)
- Respect robots.txt for non-critical sources
- Run during off-hours (2-4 AM via cron)

**Effort:** 8-12 hours (scraper + database + deduplication logic)
**Priority:** P0 — build this week
**Revenue impact:** This is the top of the funnel. Without leads, nothing else works. 500+ leads per city = thousands of potential clients.

---

#### 1.1.2 Social Media Audit Bot

**What it does:** Takes a dental practice's Instagram handle (or website URL) and generates a personalized PDF audit report. This is the free lead magnet that converts cold leads into warm conversations. The report shows: what they're doing wrong, what competitors are doing better, and exactly what Lumiere would do differently — with mockup examples.

**How Claude Code builds it:**
- Python script (`~/products/lumiere/tools/audit-bot/audit.py`)
- **Instagram analysis** (no API needed — public profile scraping):
  - Profile: followers, following, bio quality, link in bio, highlights
  - Last 12 posts: posting frequency, engagement rate (likes+comments/followers), content types (photo/video/carousel), caption quality, hashtag usage
  - Reels: views, frequency
  - Use Puppeteer to load `instagram.com/[handle]/` and extract data from the page JSON (`window._sharedData` or the GraphQL endpoint)
- **Website analysis:**
  - Mobile responsiveness (Puppeteer viewport test)
  - Load speed (basic timing)
  - Has booking widget? Has social links? Has blog?
  - SSL certificate check
- **Google Business analysis:**
  - Review count and average rating
  - Response rate to reviews
  - Photo count
  - Last update date
- **Competitor comparison:**
  - Find 2-3 competitor practices in the same city from our leads database
  - Compare metrics side by side
- **PDF generation:**
  - Use `weasyprint` or `reportlab` to generate a branded PDF
  - Lumiere branding: logo, colors, professional layout
  - Sections: Executive Summary, Instagram Audit, Website Audit, Google Presence, Competitor Comparison, "What We'd Do" (3 specific recommendations with mockup content ideas)
  - Include a CTA: "Book a free 15-minute strategy call"
  - Save to `~/products/lumiere/data/audits/[practice-name]-audit.pdf`

**Automation flow:**
1. Lead scraper finds new practice
2. Audit bot auto-generates report for high-scored leads
3. Report attached to outreach email
4. Telegram notification: "Audit generated for [Practice Name] — score: 85/100"

**Effort:** 12-16 hours (scraping + analysis logic + PDF template + branding)
**Priority:** P0 — build this week (alongside scraper)
**Revenue impact:** This is the conversion weapon. A personalized audit showing their specific weaknesses makes it nearly impossible to ignore. Expected response rate: 15-25% vs 2-5% for generic cold email.

---

#### 1.1.3 Lead Scoring Engine

**What it does:** Automatically scores every lead from 0-100 based on how likely they are to become a paying client. High-score leads get immediate outreach; low-score leads go into nurture sequences.

**How Claude Code builds it:**
- Python module (`~/products/lumiere/tools/lead-scorer/scorer.py`)
- Scoring criteria (each weighted):

| Signal | Points | Why |
|--------|--------|-----|
| Has website but NO Instagram | +25 | They know they need online presence but haven't figured out social |
| Has Instagram but inactive (30+ days no post) | +20 | They tried and gave up — perfect for "let us handle it" |
| Low Google reviews (<20) | +15 | Needs visibility help |
| No Instagram highlights | +10 | Low-effort social presence |
| Competitor in same city is very active | +15 | They're losing to someone doing it right |
| Website has no booking widget | +10 | Missing basic digital infrastructure |
| Has video content on website | +10 | Already invested in video — upsell editing services |
| Multiple locations | +15 | Bigger budget, more content needed |
| Recently opened (new Google listing) | +10 | New practices need clients fast |
| Bad engagement rate (<1%) | +10 | Posting but failing — needs expert help |
| Premium/aesthetic focus (cosmetic, Invisalign, implants) | +15 | Higher margins = bigger marketing budget |
| Has Facebook but no Instagram | +10 | Stuck on old platform |

- Runs automatically after each scraper pass
- Updates `lead_scores` table in SQLite
- Telegram daily digest: "Top 10 new leads today: [list with scores]"

**Effort:** 3-4 hours
**Priority:** P0 — build alongside scraper
**Revenue impact:** Focus energy on the leads most likely to convert. A scored list of 50 hot leads is worth more than an unsorted list of 500.

---

### 1.2 Outreach Automation

#### 1.2.1 Cold Email Sequences

**What it does:** Sends personalized multi-step email sequences to leads, with the audit report attached. Each email is customized with the practice's name, specific audit findings, and local competitor references.

**How Claude Code builds it:**
- Python email sender (`~/products/lumiere/tools/outreach/email-sender.py`)
- **SMTP setup:** Use a transactional email service with a free tier:
  - **Option 1 (recommended):** Brevo (formerly Sendinblue) — 300 emails/day free. API-based, good deliverability.
  - **Option 2:** Mailgun — 100 emails/day free for 3 months.
  - **Option 3:** Custom SMTP via a cheap domain email (Zoho free tier — 5 users free, SMTP access).
  - Set up SPF, DKIM, DMARC records on the lumiere domain for deliverability.
- **Email sequence (4 touches over 14 days):**

**Email 1 (Day 0) — The Audit:**
```
Subject: [Praxisname] — Ihre Social-Media-Analyse (kostenlos)
Body: Personalized intro → "Wir haben Ihre Online-Pr??senz analysiert" → 
      2-3 key findings from audit → PDF attached → CTA: free call
```

**Email 2 (Day 3) — The Competitor Angle:**
```
Subject: Was [Konkurrent in der Stadt] anders macht
Body: Reference specific competitor doing social media well → 
      "In 30 Tagen k??nnten Sie das auch" → CTA: free call
```

**Email 3 (Day 7) — The Case Study:**
```
Subject: Wie eine Zahnarztpraxis 40% mehr Anfragen bekam
Body: Mini case study (use your existing client results) → 
      Social proof → CTA: free call
```

**Email 4 (Day 14) — The Breakup:**
```
Subject: Soll ich aufh??ren zu schreiben?
Body: Short, direct → "Kein Problem wenn kein Interesse" → 
      Last chance CTA → Creates urgency
```

- **Template engine:** Jinja2 templates with variables: `{{practice_name}}`, `{{city}}`, `{{competitor_name}}`, `{{audit_finding_1}}`, `{{audit_finding_2}}`, `{{review_count}}`, `{{instagram_handle}}`
- **Database tracking:** `outreach_history` table tracks: lead_id, email_step, sent_at, opened (if using tracking pixel), replied
- **Cron job:** Runs every morning at 9 AM. Checks which leads need which email step. Sends in batches (50/day to stay under free tier limits and avoid spam flags).
- **Reply detection:** If using IMAP, check inbox for replies from leads and auto-mark them as "responded" in DB. Notify TJ on Telegram.

**Legal compliance (DSGVO/GDPR):**
- B2B cold email to business addresses is legal in Germany under UWG ??7 if: the email is relevant to their business, you include opt-out, and you don't email personal addresses.
- Always use info@/kontakt@/praxis@ addresses, never personal ones.
- Include unsubscribe link in every email.
- Log opt-outs in the database and never re-contact.

**Effort:** 8-12 hours (SMTP setup + templates + sequence logic + tracking + cron)
**Priority:** P0 — build in week 2
**Revenue impact:** Direct revenue driver. 50 emails/day ?? 14-day sequence = 700 touches/month. At 5% response rate = 35 conversations/month. At 20% close rate = 7 new clients/month.

---

#### 1.2.2 Instagram DM Outreach Scripts

**What it does:** Pre-generates personalized DM scripts for each high-scored lead so TJ can copy-paste them into Instagram. Not fully automated (Instagram bans automated DMs aggressively), but 90% of the work is done.

**How Claude Code builds it:**
- Python script (`~/products/lumiere/tools/outreach/dm-generator.py`)
- Uses Claude API (via DeepSeek for cost savings) to generate personalized DM scripts based on:
  - Practice name and city
  - Their recent Instagram content (or lack thereof)
  - Specific audit findings
  - Local competitor reference
- Generates 3 DM variants per lead (different tones: professional, casual, direct)
- Outputs to a daily file: `~/products/lumiere/data/outreach/dm-scripts-YYYY-MM-DD.md`
- Also sends via Telegram: "Today's DM scripts ready — 15 leads to message"

**DM template structure:**
```
DM 1 (Opening):
"Hi [Name]! Ich bin auf Ihre Praxis in [Stadt] gesto??en. 
Ihr letzter Instagram-Post ist von [Datum] ??? darf ich Ihnen 
zeigen, wie ??hnliche Praxen mit 2-3 Posts pro Woche 
ihre Patientenanfragen verdoppelt haben? Komplett kostenlos."

DM 2 (If no reply after 3 days):
"Kurze Frage: nutzen Sie Social Media aktiv f??r die 
Patientengewinnung? Ich habe eine kostenlose Analyse 
f??r [Praxisname] erstellt ??? soll ich sie schicken?"
```

**Effort:** 3-4 hours
**Priority:** P1 — build in week 2
**Revenue impact:** Instagram DMs have 30-50% open rate vs 20% for email. Complements email outreach for leads who don't respond to email.

---

#### 1.2.3 Follow-Up Reminder System

**What it does:** Tracks all outreach (email, DM, calls) and sends Telegram reminders when a lead hasn't responded after 3, 5, and 7 days. Prevents leads from falling through the cracks.

**How Claude Code builds it:**
- Python script (`~/products/lumiere/tools/outreach/followup-tracker.py`)
- Reads from `outreach_history` table in SQLite
- Cron job: runs daily at 8 AM
- Logic:
  - Day 3 no response: Telegram notification "Follow up with [Practice] — try DM"
  - Day 5 no response: Telegram notification "Last chance for [Practice] — try phone"
  - Day 7 no response: Mark as "cold" — move to monthly nurture list
  - Reply detected: Telegram celebration notification + mark as "warm lead"
- Weekly summary every Monday: "This week: X leads contacted, Y responded, Z calls booked"

**Effort:** 2-3 hours
**Priority:** P1 — build in week 2
**Revenue impact:** Most sales happen on follow-up 2-4, not the first touch. This ensures TJ never forgets to follow up.

---

#### 1.2.4 LinkedIn Outreach Automation

**What it does:** Semi-automated LinkedIn outreach to dental practice owners and office managers. Generates personalized connection request notes and follow-up messages.

**How Claude Code builds it:**
- Python script (`~/products/lumiere/tools/outreach/linkedin-generator.py`)
- **Lead enrichment:** For each lead in the database, search LinkedIn for the practice owner. Use Puppeteer to search `site:linkedin.com "[practice name]" zahnarzt [city]` on Google.
- **Connection request note generator:** Claude/DeepSeek generates a 300-character personalized note per lead.
- **Message sequence after connection:**
  - Day 1: Thank for connecting + soft value offer
  - Day 3: Share a relevant dental social media tip (position as expert)
  - Day 7: Offer free audit
- Output: daily markdown file with copy-paste messages + LinkedIn profile URLs
- Telegram notification: "LinkedIn outreach ready — 10 profiles to connect with today"

**Note:** Do NOT use LinkedIn automation tools (they get accounts banned). This generates the text; TJ sends it manually. 5-10 minutes/day of copy-pasting.

**Effort:** 3-4 hours
**Priority:** P2 — build when email + DM outreach is running
**Revenue impact:** LinkedIn targets decision-makers directly. Higher conversion rate than cold email for B2B services, but lower volume.

---

### 1.3 Inbound Lead Capture

#### 1.3.1 Website AI Chatbot

**What it does:** An AI-powered chatbot embedded on the Lumiere website that qualifies leads 24/7. Answers questions about services and pricing, collects contact info, and books free strategy calls.

**How Claude Code builds it:**
- Lightweight chat widget (`~/products/lumiere/tools/chatbot/`)
- **Frontend:** Vanilla JS chat widget (~200 lines) injected into the GitHub Pages site via a `<script>` tag. Floating button bottom-right, opens chat panel.
- **Backend:** Small Express.js or Python Flask server on the VPS (PM2-managed, port 3101).
- **AI backend:** DeepSeek API (free/cheap) with a system prompt that:
  - Knows Lumiere's services, pricing tiers (EUR 500/1000/3000)
  - Qualifies leads by asking: practice type, city, current social media status, budget
  - Offers to schedule a free call
  - Collects name + email + phone
- **Lead capture:** When chatbot collects contact info, writes to `leads.db` + sends Telegram notification to TJ
- **Conversation logging:** All chats saved to `~/products/lumiere/data/chatbot-logs/` for review

**Effort:** 6-8 hours (widget + API backend + system prompt tuning)
**Priority:** P1 — build in week 2-3
**Revenue impact:** Captures leads who visit the website outside business hours. Even 1-2 qualified leads/month from the chatbot pays for itself.

---

#### 1.3.2 Formspree to Telegram Notification

**What it does:** When someone fills out the contact form on the Lumiere website, TJ gets an instant Telegram notification with all lead details so he can respond within minutes.

**How Claude Code builds it:**
- **Option A (Formspree webhook):** Formspree Pro ($8/mo) supports webhooks. Webhook hits a small endpoint on TJ's VPS that forwards to Telegram.
- **Option B (free, recommended):** Replace Formspree with a custom form handler:
  - Tiny Flask/Express endpoint on VPS (port 3102, behind nginx)
  - Form POST goes to `https://lumiere-api.example.com/contact` (or use the VPS IP directly)
  - Endpoint validates, stores in `leads.db`, sends Telegram notification via Bot API
  - Responds with a thank-you page redirect

**Telegram notification format:**
```
???? New Lumiere Lead!
Name: Dr. M??ller
Email: mueller@zahnarzt-berlin.de
Phone: +49 170 1234567
Practice: Zahnarztpraxis M??ller
City: Berlin
Message: "Interesse an Social Media Management"
---
Score: 78/100 (auto-calculated)
Audit: [generating...]
```

- Auto-triggers audit bot for the new lead
- Adds to email outreach sequence if no response in 24h

**Effort:** 2-3 hours
**Priority:** P0 — build in week 1 (quick win)
**Revenue impact:** Speed to respond is the #1 factor in converting inbound leads. Responding in 5 minutes vs 5 hours = 10x higher conversion.

---

#### 1.3.3 Google Business Profile Automation

**What it does:** Auto-posts updates to Lumiere's own Google Business Profile and monitors client review responses.

**How Claude Code builds it:**
- **Google Business Profile API** (free): Use the GBP API to:
  - Post weekly updates (generated by Claude/DeepSeek): dental social media tips, client success stories, service highlights
  - Monitor new reviews and draft responses
- Python script (`~/products/lumiere/tools/gbp/gbp-manager.py`)
- Cron: weekly post on Tuesday morning, daily review check
- OAuth setup required (one-time via Google Cloud Console — free tier)

**Effort:** 4-6 hours (OAuth setup is the time sink)
**Priority:** P2 — build when core acquisition is running
**Revenue impact:** Improves Lumiere's own Google visibility. Long-term SEO play. Low urgency but compounds over time.

---

#### 1.3.4 Content Marketing Funnel

**What it does:** Auto-generates and schedules blog posts and LinkedIn articles about dental social media tips. Positions TJ as the expert, drives organic inbound leads.

**How Claude Code builds it:**
- **Blog posts:** Claude/DeepSeek generates weekly articles on topics like:
  - "5 Instagram-Fehler, die jede Zahnarztpraxis macht"
  - "Vorher-Nachher-Fotos: So nutzen Sie sie richtig (DSGVO-konform)"
  - "Warum Ihre Zahnarztpraxis TikTok braucht (ja, wirklich)"
- Python script (`~/products/lumiere/tools/content-marketing/blog-generator.py`)
- Generates markdown article + meta description + social sharing snippet
- Auto-publishes to GitHub Pages blog section (git commit + push)
- Cross-posts to LinkedIn via LinkedIn API (or generates copy-paste text)
- Cron: generates 1 article/week, publishes every Wednesday

**LinkedIn posting:**
- Repurpose blog posts into shorter LinkedIn posts
- Generate 3 LinkedIn posts/week (tips, insights, mini case studies)
- Output to `~/products/lumiere/data/content/linkedin-posts-YYYY-MM-DD.md`
- Telegram reminder: "LinkedIn posts ready for today — 3 posts to publish"

**Effort:** 6-8 hours (generator + blog template + publishing pipeline)
**Priority:** P1 — build in week 3
**Revenue impact:** Organic inbound is the highest-quality lead source. Takes 2-3 months to compound but becomes a free lead engine.

---

## 2. Client Delivery Automation

Once TJ signs clients, speed and consistency of delivery determines retention. Target: each client takes max 4-5 hours/month to service.

---

### 2.1 Content Creation Pipeline

#### 2.1.1 PostForge Batch Content Generation

**What it does:** Uses the existing PostForge tool to generate a full month of social media content for each client in one batch. Outputs: 12-16 posts with captions, hashtags, posting schedule, and content briefs for any photos/videos needed.

**How Claude Code builds it:**
- Wrapper script (`~/products/lumiere/tools/delivery/batch-content.py`) that:
  1. Reads client profile from `~/products/lumiere/data/clients/[client-slug]/profile.json` (practice type, brand voice, services, target audience, city, competitors)
  2. Generates content calendar for the month: mix of post types (educational, behind-the-scenes, testimonial, before/after, team intro, seasonal, promotion)
  3. Calls PostForge for each post: caption in German, 20 relevant hashtags, posting time recommendation
  4. Outputs: `content-calendar-YYYY-MM.md` + individual post files
  5. Sends Telegram summary: "Content for [Client] ready — 16 posts for March"
- Run monthly, triggered by cron on the 25th of each month (prep for next month)

**Content mix template (per month):**
```
- 4x Educational tips (dental hygiene, treatment explainers)
- 3x Behind-the-scenes (team, equipment, office tour)
- 2x Patient testimonials/reviews (anonymized or with consent)
- 2x Before/after (treatment results)
- 2x Seasonal/trending (holidays, awareness days)
- 1x Promotion/offer
- 2x Reels/video content briefs
```

**Effort:** 4-6 hours (wrapper + client profile system + calendar logic)
**Priority:** P1 — build in week 3
**Revenue impact:** Reduces per-client delivery time from 8-10 hours to 2-3 hours. Enables scaling to 10+ clients without burning out.

---

#### 2.1.2 AICut Video Editing Pipeline

**What it does:** Client sends a long video (practice tour, treatment explanation, team intro). AICut automatically creates 3-5 short-form clips (15-60 seconds) optimized for Instagram Reels and TikTok.

**How Claude Code builds it:**
- Integration script (`~/products/lumiere/tools/delivery/video-pipeline.py`)
- Flow:
  1. Client uploads video to a shared folder (Google Drive or simple upload endpoint on VPS)
  2. Fabienne transcribes the video
  3. Claude/DeepSeek analyzes transcript to identify the best 3-5 clip-worthy moments
  4. AICut cuts the clips with: captions burned in, intro/outro branding, music suggestion
  5. Output: clips saved to client folder + Telegram notification
- Semi-automated: Claude selects moments, TJ reviews cuts before sending to client

**Effort:** 6-8 hours (integration between Fabienne + AICut + clip selection logic)
**Priority:** P1 — build in week 3-4
**Revenue impact:** Video content gets 2-3x engagement on Instagram. Offering video editing as part of the package justifies the EUR 1000+ tier.

---

#### 2.1.3 Fabienne Transcription Integration

**What it does:** Client sends a video or voice memo. Fabienne auto-transcribes it and generates: social media captions, blog post draft, and quote graphics text.

**How Claude Code builds it:**
- Already built (Fabienne is running on PM2). Needs a client-facing integration:
  - Simple upload page per client (`https://[vps-ip]:3000/upload/[client-slug]`)
  - Or: monitor a shared Google Drive folder per client
- Post-transcription hook: automatically pipe transcript to PostForge for caption generation
- Output: transcript + 3-5 social media captions + 1 blog post draft

**Effort:** 2-3 hours (integration hook + client upload endpoint)
**Priority:** P1 — build in week 3
**Revenue impact:** Turns a 5-minute client voice memo into a week of content. Massively reduces the creative bottleneck.

---

#### 2.1.4 Design Template Library

**What it does:** Pre-built Canva-compatible design templates for the most common dental content types. TJ (or eventually a VA) just swaps in the client's photos and branding.

**How Claude Code builds it:**
- This is NOT a code task — it's a Canva task. But Claude Code helps by:
  - Generating the template specifications: dimensions, color palette per client, font recommendations, layout descriptions
  - Creating an HTML/CSS version of each template that can be screenshotted or exported as images (using Puppeteer)
  - Script (`~/products/lumiere/tools/delivery/template-generator.py`) that generates branded image templates using `Pillow` (Python imaging library)

**Template types needed:**
```
1. Before/After split (portrait, 1080x1350)
2. Dental tip carousel (1080x1080, 5 slides)
3. Team member spotlight (1080x1350)
4. Patient testimonial quote (1080x1080)
5. Treatment explainer (1080x1350)
6. Story template — poll/question (1080x1920)
7. Story template — announcement (1080x1920)
8. Reel cover (1080x1920)
```

**Effort:** 8-12 hours (template design + generation script)
**Priority:** P2 — build when client count reaches 3+
**Revenue impact:** Templates make content creation 3x faster. Essential for scaling past 5 clients.

---

#### 2.1.5 Social Media Scheduling

**What it does:** Auto-publishes approved content to Instagram, Facebook, and TikTok at optimal times.

**How Claude Code builds it:**
- **Option A (recommended for now):** Use Buffer free tier (3 channels, 10 posts scheduled). Upload manually via Buffer UI — fast enough for 2-3 clients.
- **Option B (for 5+ clients):** Later.com free tier (1 social set, 30 posts/month).
- **Option C (for 10+ clients, build custom):**
  - Meta Graph API (free) for Instagram/Facebook publishing
  - Python script that reads approved content from client folder and publishes via API
  - Cron job: publishes scheduled posts at the time specified in the content calendar
  - Requires: Facebook App approval, Instagram Business account per client, Meta API token per client
  - Script: `~/products/lumiere/tools/delivery/auto-publisher.py`

**API setup for Meta publishing:**
1. Create Facebook Developer App
2. Request `pages_manage_posts`, `instagram_basic`, `instagram_content_publish` permissions
3. Each client connects their Instagram Business account to a Facebook Page
4. Store per-client tokens in `~/products/lumiere/data/clients/[slug]/meta-token.json`
5. Refresh tokens automatically (they expire every 60 days)

**Effort:** 2 hours (Buffer setup) or 12-16 hours (custom Meta API integration)
**Priority:** P2 (Buffer now) / P1 (custom integration when at 5+ clients)
**Revenue impact:** Eliminates the manual posting step entirely. At 10+ clients with 4 posts/week each, that's 160+ posts/month. Manual posting would take 8+ hours/month; automation takes 0.

---

### 2.2 Client Reporting

#### 2.2.1 Automated Monthly Reports

**What it does:** On the 1st of each month, automatically generates a branded PDF report for each client showing: follower growth, engagement rates, top-performing posts, reach/impressions, comparison to previous month, and recommendations for next month.

**How Claude Code builds it:**
- Python script (`~/products/lumiere/tools/reporting/monthly-report.py`)
- **Data sources:**
  - Instagram Insights API (via Meta Graph API — requires Business account)
  - Facebook Insights API
  - Google Analytics (if client has GA on their website)
  - Manual: Google review count tracking (scraped monthly)
- **Report sections:**
  1. Executive Summary (2-3 key wins)
  2. Follower Growth (chart)
  3. Engagement Rate (chart + comparison to industry average: dental = 1-3%)
  4. Top 3 Posts (screenshots + why they worked)
  5. Reach & Impressions
  6. Month-over-Month Comparison
  7. Recommendations for Next Month
  8. Upcoming Content Preview
- **PDF generation:** `weasyprint` or `reportlab` with Lumiere branding
- **Charts:** `matplotlib` or `plotly` for clean data visualizations
- Cron: 1st of every month, 6 AM. Generates all client reports. Telegram: "Monthly reports ready for X clients"
- Auto-emails report to client (using SMTP) with a personal note

**Effort:** 12-16 hours (API integration + chart generation + PDF template)
**Priority:** P1 — build in week 4
**Revenue impact:** Professional reports justify the monthly fee. Clients who see clear ROI data renew. Churn drops from ~30% to ~10% with good reporting.

---

#### 2.2.2 Competitor Tracking

**What it does:** Monthly automated analysis of each client's top 3 local competitors' social media. Shows what competitors are doing, what's working for them, and opportunities the client should capitalize on.

**How Claude Code builds it:**
- Python script (`~/products/lumiere/tools/reporting/competitor-tracker.py`)
- For each client, store competitor Instagram handles in their profile
- Monthly scrape: follower count change, posting frequency, engagement rate, content types, top posts
- DeepSeek/Claude generates a 1-page competitor analysis summary
- Included as an addendum to the monthly report
- Cron: runs on the 28th of each month (before monthly report generation)

**Effort:** 4-6 hours (extends existing scraping infrastructure)
**Priority:** P2 — build when client base is 3+
**Revenue impact:** Unique value-add that most agencies don't provide. Creates "we need to keep up with competitors" urgency that prevents clients from canceling.

---

### 2.3 Client Communication

#### 2.3.1 Onboarding Automation

**What it does:** When a new client signs up, automatically sends a welcome email sequence, onboarding questionnaire, and collects everything needed to start content creation.

**How Claude Code builds it:**
- Python script (`~/products/lumiere/tools/client-mgmt/onboarding.py`)
- **Onboarding sequence (auto-triggered when TJ marks a deal as "won" in the CRM):**

**Day 0 — Welcome email:**
- Congratulations message
- What to expect (timeline, process, communication)
- Link to onboarding questionnaire (Google Form or Typeform free)
- Request: brand guidelines, logo, photos, Instagram login, Facebook Page admin access

**Day 1 — Questionnaire follow-up:**
- If questionnaire not completed, Telegram reminder to TJ + auto email reminder to client

**Day 3 — Brand guide + content strategy draft:**
- Claude/DeepSeek generates a brand voice guide based on questionnaire answers
- First content strategy draft auto-generated
- Sent to client for review

**Day 5 — First content preview:**
- 3 sample posts generated using PostForge with client's brand voice
- Sent for approval
- Once approved, content creation begins at full speed

**Questionnaire fields:**
```
- Practice name, full address
- Services offered (checkboxes)
- Target patients (families, cosmetic, implants, etc.)
- Brand voice preference (professional, friendly, luxury, modern)
- Competitor practices they admire
- Content preferences (video ok? Team photos ok? Patient photos ok?)
- Seasonal promotions/events
- Instagram login (or Business account manager access)
- Facebook Page admin access
- Logo files + brand colors
```

**Effort:** 4-6 hours (email templates + questionnaire + auto-generation logic)
**Priority:** P1 — build in week 4
**Revenue impact:** Professional onboarding = professional first impression. Reduces "I don't know what to send you" back-and-forth from 2 weeks to 3 days.

---

#### 2.3.2 Content Approval Workflow

**What it does:** Client reviews content in a simple shared interface. They can approve, request changes, or reject with one click. Approved content auto-queues for publishing.

**How Claude Code builds it:**
- **Simple approach (recommended for now):** Shared Google Drive folder per client
  - Folder structure: `Pending Review/` → `Approved/` → `Published/`
  - TJ uploads content to `Pending Review/`
  - Client moves to `Approved/` (or comments with changes)
  - Auto-publisher watches `Approved/` folder
- **Advanced approach (build later):**
  - Simple web UI on VPS (port 3103)
  - Client logs in, sees content calendar
  - Approve/reject/comment buttons per post
  - Approved posts auto-queue for scheduling
  - Built with Flask + simple auth (token-based, one token per client)

**Effort:** 2 hours (Google Drive approach) or 8-12 hours (custom web UI)
**Priority:** P2 (Google Drive now) / P1 (custom UI at 5+ clients)
**Revenue impact:** Reduces approval bottleneck from days to hours. Content gets published faster, results come faster, client is happier.

---

## 3. Internal Operations Automation

Keep the business running smoothly with minimal administrative overhead.

---

#### 3.1 Invoice Generation

**What it does:** Auto-generates monthly invoices for each client on the 1st of the month. Branded PDF with Lumiere details, client details, service description, and payment info.

**How Claude Code builds it:**
- Python script (`~/products/lumiere/tools/admin/invoice-generator.py`)
- Uses `reportlab` or `weasyprint` to generate branded PDF invoices
- Client data from `~/products/lumiere/data/clients/[slug]/billing.json`:
  ```json
  {
    "company": "Zahnarztpraxis M??ller",
    "address": "Hauptstra??e 1, 10115 Berlin",
    "email": "mueller@zahnarzt-berlin.de",
    "plan": "growth",
    "monthly_amount": 1000,
    "currency": "EUR",
    "tax_id": "DE123456789",
    "payment_method": "bank_transfer",
    "iban": "..."
  }
  ```
- German invoice requirements (Pflichtangaben): Lumiere company info, Steuernummer, invoice number (sequential), date, service description, net amount, USt (19%), gross amount, payment terms
- Auto-increments invoice numbers: `LUM-2026-001`, `LUM-2026-002`, etc.
- Cron: 1st of month, 8 AM. Generates all invoices. Telegram: "Invoices generated for X clients"
- Auto-emails invoice to client
- Saves to `~/products/lumiere/data/invoices/YYYY/[client-slug]-LUM-YYYY-NNN.pdf`

**Effort:** 4-6 hours
**Priority:** P1 — build in week 4
**Revenue impact:** Saves 30 minutes per invoice per month. At 10 clients, that's 5 hours/month saved. More importantly, ensures you never forget to invoice.

---

#### 3.2 Time Tracking

**What it does:** Simple automated logging of time spent per client. TJ logs time via Telegram command; system tracks and reports monthly totals.

**How Claude Code builds it:**
- Telegram bot command: `/time [client] [hours] [task]`
  - Example: `/time mueller 1.5 content creation`
  - Stores in SQLite: `time_entries` table (date, client_slug, hours, task, timestamp)
- Weekly Telegram digest: "This week: 12h total. Mueller 4h, Schmidt 3h, ..."
- Monthly summary in invoice data: hours spent per client
- Helps identify unprofitable clients (>8h/month on a EUR 500/month plan)

**Effort:** 2-3 hours
**Priority:** P2 — build when at 3+ clients
**Revenue impact:** Data-driven pricing decisions. Know exactly which clients are profitable and which need to be upsold or dropped.

---

#### 3.3 CRM (Lead & Client Tracker)

**What it does:** Central SQLite database + simple Telegram interface for tracking every lead from first contact to paying client to renewal.

**How Claude Code builds it:**
- The SQLite database (`leads.db`) is already part of the scraper design. Extend it with:
  
**Tables:**
```sql
leads (id, name, practice_name, city, email, phone, website, 
       instagram, score, status, source, created_at, updated_at)

-- status: new, contacted, responded, call_booked, proposal_sent, 
--         won, lost, nurture

outreach_history (id, lead_id, channel, step, sent_at, 
                  opened_at, replied_at, notes)

clients (id, lead_id, plan, monthly_amount, start_date, 
         contract_end, status, onboarding_complete)

tasks (id, client_id, description, due_date, status, hours)

invoices (id, client_id, number, amount, sent_at, paid_at)
```

- **Telegram commands:**
  - `/leads` — show top 10 scored leads
  - `/pipeline` — show leads by status (funnel view)
  - `/clients` — show active clients
  - `/update [lead] [status]` — move lead through pipeline
  - `/revenue` — monthly revenue summary

- **Dashboard (optional, later):** Simple Flask web page on port 3104 showing pipeline visualization, revenue chart, client health scores

**Effort:** 6-8 hours (database schema + Telegram commands + pipeline logic)
**Priority:** P1 — build in week 2 (needed as soon as outreach starts)
**Revenue impact:** Without a CRM, leads fall through cracks. A simple pipeline view ensures every lead gets followed up and every opportunity is tracked.

---

## 4. Implementation Roadmap

### Week 1 — Lead Engine (Foundation)

| Day | Task | Hours | Deliverable |
|-----|------|-------|-------------|
| Mon | Set up project structure, SQLite database schema, client data models | 3h | Database + folder structure |
| Mon | Build Formspree/custom form → Telegram notification | 2h | Inbound leads notify instantly |
| Tue | Build Google Maps scraper (Puppeteer) | 4h | Scraper for top 5 cities |
| Wed | Build Jameda + Gelbe Seiten scrapers | 4h | Multi-source lead data |
| Wed | Build lead scoring engine | 3h | Auto-scored leads in DB |
| Thu | Build social media audit bot (Instagram scraping) | 6h | Instagram analysis per lead |
| Fri | Build audit PDF generator (branding + layout) | 6h | Branded PDF audit reports |
| Fri | Test full pipeline: scrape → score → audit → notification | 2h | End-to-end working |

**Week 1 output:** 500+ scored leads in the database. Auto-generated audit PDFs for top 50. Telegram notifying TJ of every new high-score lead and inbound form submission.

---

### Week 2 — Outreach Machine

| Day | Task | Hours | Deliverable |
|-----|------|-------|-------------|
| Mon | Set up transactional email (Brevo SMTP + domain DNS) | 3h | Email infrastructure ready |
| Mon | Build email templates (4-step sequence, German) | 3h | Personalized email templates |
| Tue | Build email sender + sequence logic + cron | 4h | Auto-sending email sequences |
| Tue | Build DM script generator | 3h | Daily DM scripts via Telegram |
| Wed | Build follow-up reminder system | 3h | Telegram follow-up alerts |
| Wed | Build CRM database + Telegram commands | 4h | Pipeline tracking via Telegram |
| Thu | Build LinkedIn message generator | 3h | LinkedIn outreach scripts |
| Thu | Test full outreach pipeline end-to-end | 2h | Scrape → score → audit → email → track |
| Fri | Launch: send first 50 outreach emails | 2h | First real leads contacted |

**Week 2 output:** Automated email sequences running. CRM tracking all leads. TJ spending 30 min/day on DM + LinkedIn outreach. First responses coming in.

---

### Week 3 — Content Delivery Pipeline

| Day | Task | Hours | Deliverable |
|-----|------|-------|-------------|
| Mon | Build PostForge batch content wrapper | 4h | One-click monthly content generation |
| Mon | Build client profile system (JSON configs per client) | 2h | Standardized client data |
| Tue | Build Fabienne integration (client upload → transcript → captions) | 3h | Video-to-content pipeline |
| Tue | Build AICut integration (clip selection + cutting) | 4h | Auto short-form video creation |
| Wed | Build content marketing blog generator | 4h | Weekly blog posts auto-generated |
| Wed | Set up Buffer/scheduling for existing clients | 2h | Content auto-scheduled |
| Thu | Build chatbot backend (DeepSeek-powered) | 4h | 24/7 lead qualification |
| Thu | Build chatbot frontend widget | 3h | Chat widget on Lumiere website |
| Fri | Test all delivery pipelines with real client content | 3h | Full delivery automation working |

**Week 3 output:** Content creation down to 2-3 hours per client per month. Video pipeline working. Chatbot live on website. Blog publishing weekly.

---

### Week 4 — Reporting, CRM & Polish

| Day | Task | Hours | Deliverable |
|-----|------|-------|-------------|
| Mon | Build automated monthly report (data collection + charts) | 6h | Instagram/Facebook analytics pulled |
| Tue | Build report PDF generator (branded template) | 4h | Beautiful client reports |
| Tue | Build competitor tracking script | 3h | Monthly competitor analysis |
| Wed | Build invoice generator | 4h | Auto-invoicing on the 1st |
| Wed | Build onboarding automation (email sequence + questionnaire) | 3h | New client onboarding in 3 days |
| Thu | Build time tracking (Telegram commands) | 2h | Per-client hour tracking |
| Thu | Polish CRM: add /pipeline, /revenue, /clients commands | 3h | Full Telegram CRM |
| Fri | End-to-end test: simulate new lead → client → first month delivery | 3h | Everything works together |
| Fri | Document all cron jobs, API keys, processes | 2h | Operations manual |

**Week 4 output:** Full automation stack operational. Monthly reports auto-generated. Invoicing automated. CRM tracking everything. TJ can manage 10 clients spending <20 hours/month total.

---

## Quick Reference: All Cron Jobs

| Schedule | Script | What it does |
|----------|--------|--------------|
| Sun 2 AM | `lead-scraper/scraper.py` | Scrape new leads from all sources |
| Daily 6 AM | `lead-scorer/scorer.py` | Re-score all leads |
| Daily 8 AM | `outreach/followup-tracker.py` | Send follow-up reminders |
| Daily 9 AM | `outreach/email-sender.py` | Send next email in sequence |
| Daily 9 AM | `outreach/dm-generator.py` | Generate DM scripts for today |
| Tue 8 AM | `gbp/gbp-manager.py` | Post Google Business update |
| Wed 6 AM | `content-marketing/blog-generator.py` | Generate + publish blog post |
| 25th 6 AM | `delivery/batch-content.py` | Generate next month's content per client |
| 28th 6 AM | `reporting/competitor-tracker.py` | Run competitor analysis |
| 1st 6 AM | `reporting/monthly-report.py` | Generate client reports |
| 1st 8 AM | `admin/invoice-generator.py` | Generate + send invoices |

---

## Quick Reference: All Telegram Commands

| Command | What it does |
|---------|--------------|
| `/leads` | Show top 10 scored new leads |
| `/pipeline` | Show leads by funnel stage |
| `/clients` | Show active clients + health |
| `/update [lead] [status]` | Move lead through pipeline |
| `/revenue` | Monthly revenue summary |
| `/time [client] [hours] [task]` | Log time entry |
| `/audit [instagram-handle]` | Trigger manual audit report |
| `/content [client]` | Generate content batch for client |
| `/report [client]` | Generate monthly report for client |

---

## Quick Reference: All PM2 Services

| PM2 Name | Port | Script |
|----------|------|--------|
| `lumiere-chatbot` | 3101 | `tools/chatbot/server.py` |
| `lumiere-forms` | 3102 | `tools/forms/server.py` |
| `lumiere-dashboard` | 3104 | `tools/dashboard/app.py` |
| `lumiere-email` | — | `tools/outreach/email-sender.py` (cron, not persistent) |

---

## Priority Matrix Summary

| Priority | Items | Combined effort | Expected impact |
|----------|-------|----------------|-----------------|
| **P0** (this week) | Lead scraper, Audit bot, Lead scoring, Form→Telegram, Email sequences | ~40h | Generates first leads and conversations |
| **P1** (this month) | DM generator, Follow-ups, CRM, Chatbot, PostForge batch, Fabienne integration, Reports, Invoicing, Onboarding, Content marketing | ~55h | Full acquisition + delivery pipeline |
| **P2** (when needed) | LinkedIn outreach, GBP automation, Templates, Competitor tracking, Time tracking, Custom scheduling, Dashboard | ~45h | Scaling and optimization |

---

## Tech Stack Summary

| Component | Technology | Cost |
|-----------|-----------|------|
| Scraping | Puppeteer + requests + BeautifulSoup | Free |
| Database | SQLite | Free |
| Email sending | Brevo free tier (300/day) | Free |
| AI text generation | DeepSeek API (via existing key) | ~EUR 1-2/month |
| PDF generation | weasyprint or reportlab | Free |
| Charts | matplotlib | Free |
| Web backend | Flask (Python) | Free |
| Chat widget | Vanilla JS | Free |
| Scheduling | PM2 + cron | Free (VPS already paid) |
| Social publishing | Buffer free tier → Meta API | Free |
| Notifications | Telegram Bot API | Free |
| Version control | Git on VPS | Free |

**Total additional cost: EUR 0-5/month.** Everything runs on the existing VPS.

---

*Last updated: 2026-04-15*
*This playbook is the operating manual for Lumiere's growth engine. Execute week by week. Every automation here is buildable by Claude Code on TJ's VPS.*
