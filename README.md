# AI Automation Portfolio

Real systems I've built and operate. Not tutorials, not toy projects. These are production workflows handling live traffic, real customers, and actual business operations.

## What I Do

I find manual processes and turn them into automated systems. My sweet spot is the full loop: identify the inefficiency, prototype a solution, wire up the APIs, deploy it, and keep it running. Most of my work connects AI/LLM services with existing business tools through custom Python backends and webhook orchestration.

**Stack:** Python, Flask, PHP, JavaScript, REST APIs, webhooks, Claude/OpenAI APIs, Twilio, Stripe, WordPress, MySQL, FFmpeg

---

## Projects

### 1. AI Voice and SMS Agent System + Full CRM Platform

Multi channel AI communication platform with a complete CRM backend. 22 custom database tables, 83+ REST API endpoints, AI estimate drafting, Stripe payments, and SMTP2GO transactional email.

**What it does:**
- AI answers inbound calls via Twilio, has a natural conversation, collects information
- Transcribes and summarizes every call using Whisper and Claude
- Routes leads into a CRM pipeline based on which business number was dialed
- Sends automated SMS follow ups after calls
- Two way SMS with AI responses and human handoff detection
- Click to call outbound dialing from a CRM dashboard

**Tech:** Python/Flask, Twilio (voice + SMS), Claude API, OpenAI Whisper, Retell AI, custom webhook orchestration, SQLite/MySQL

**Architecture:**
```
Inbound Call (Twilio)
    |
    v
Voice AI Agent (Retell + Claude)
    |
    v
Call Recording + Transcription (Whisper)
    |
    v
Lead Creation + CRM Pipeline Assignment
    |
    v
Automated Follow Up (SMS/Email)
    |
    v
Notification to Business Owner
```

[Full project details](projects/01-voice-sms-agent.md)

---

### 2. Multi Brand Social Scanner and Content Intelligence System

Two systems: (1) multi brand Reddit engagement scanner with AI scoring and persona matched response drafting, and (2) content intelligence mining that extracts questions, complaints, sentiment, keywords, and competitive mentions from Reddit to feed SEO strategy.

**What it does:**
- Scans target subreddits on a schedule using Late API
- AI scores each post/comment for opportunity signals (frustration, shopping for alternatives, technical questions)
- Drafts persona matched responses using a detailed voice profile
- Flask dashboard with approve/edit/skip workflow
- Throttled posting with rate limiting (no spam, no bans)
- Tracks engagement and reply performance

**Tech:** Python/Flask, Late API (Reddit read/write), Claude API (scoring + drafting), MySQL, WordPress plugin

**Architecture:**
```
Cron (every 30 min)
    |
    v
Late API: Search + Feed Pull
    |
    v
Dedup + Keyword Filter (MySQL)
    |
    v
Scoring Engine: Opportunity Score + Signal Classification
    |
    v
Review Queue (WordPress Admin Plugin)
    |
    v
Human: Approve / Edit / Skip
    |
    v
Late API: Post Comment (throttled)
```

[Full project details](projects/02-social-scanner.md)

---

### 3. SMS Review Automation with AI Intent Analysis

Automated SMS conversation engine that collects customer reviews through text messages, analyzes sentiment with AI, and posts verified reviews to a website.

**What it does:**
- Business owner enters customer name and phone number
- System sends first SMS, handles full multi step conversation
- State machine tracks each customer through consent, rating, and open ended questions
- AI analyzes responses for intent and sentiment in real time
- Positive reviews get routed to Google review links
- Full conversation log pushed to WordPress via REST API
- Opt out handling (STOP/UNSUBSCRIBE keywords)

**Tech:** Python/Flask, httpSMS/Twilio, OpenAI API, WordPress REST API, SQLite, bidirectional webhooks

**Architecture:**
```
Business submits customer info (WordPress form)
    |
    v
Flask API receives webhook, stores state, sends first SMS
    |
    v
Customer replies -> State machine walks through questions
    |
    v
AI analyzes responses, generates review text
    |
    v
Review posted to WordPress via REST API
```

[Full project details](projects/03-sms-review-automation.md)

---

### 4. AI Content Pipeline (Video to Multi Platform)

One video in, a full week of content out. Automated pipeline that turns a single video recording into YouTube content, blog posts, social media clips, email newsletters, and podcast audio.

**What it does:**
- Brand Kit compiles business profile into an LLM Index (auto cached, hourly refresh)
- AI generates topic suggestions and two script versions from brand context
- Client reviews scripts via magic link (no login required)
- Client records video, Whisper transcribes locally (no API cost)
- Claude generates all derivative content (blog, 13 platform social posts, newsletter) with per platform character limits
- Client approves/edits/requests revisions via magic link before anything posts
- Multi platform posting via Late API (YouTube, Instagram, Facebook, TikTok, LinkedIn, X, and more)

**Tech:** Python/Flask, faster-whisper (local), Claude API, FFmpeg, Late API, WordPress, Stripe Connect, Alpine.js

**Architecture:**
```
Brand Kit -> LLM Index (compiled business profile)
    |
    v
AI Script Generation (2 versions: structured + conversational)
    |
    v
Client Review via Magic Link
    |
    v
Video Recording + Whisper Transcription (local)
    |
    v
Claude Content Generation (transcript + brand context)
    |-- YouTube metadata
    |-- Blog post (1,000-2,000 words)
    |-- Social posts (13 platforms, per platform char limits)
    |-- Email newsletter
    |
    v
FFmpeg Processing (clips, aspect ratios, text overlays)
    |
    v
Client Approval via Magic Link (per item approve/edit/revise)
    |
    v
Late API: Post to All Platforms
```

[Full project details](projects/04-content-pipeline.md)

---

### 5. WordPress Multisite SaaS with Auto Provisioning

Production SaaS platform that auto provisions client websites on payment. Handles the full lifecycle: payment, site creation, DNS, SSL, onboarding, subscription management, and teardown.

**What it does:**
- Customer pays through Stripe/WooCommerce, site clones automatically
- Template site cloned with serialization safe URL replacement (handles Elementor/ACF data)
- Subdomain provisioned via RunCloud API, DNS via Cloudflare API, SSL automatic
- Custom domain mapping with sunrise.php (no plugins)
- Member dashboard for site management, subscription status, custom domain setup
- Subscription enforcement with grace periods and site locking
- 14+ custom Elementor widgets for directory functionality

**Tech:** PHP, WordPress Multisite, WooCommerce, Stripe, Cloudflare API, RunCloud API, Elementor Pro, ACF Pro, custom REST API

[Full project details](projects/05-multisite-saas.md)

---

### 6. Automated Document Generation and Compliance Tracking

Built for a healthcare compliance platform handling protected health information (PHI). PDF generation from form submissions with e signatures, vendor compliance monitoring with automated reminders, and HIPAA compliant security (encryption + audit logging).

**What it does:**
- Dynamic form submissions generate branded PDFs via TCPDF
- Canvas based e signature capture with signer tracking
- Vendor compliance dashboard tracks W9s, insurance certificates, license expiration
- Automated reminder emails when documents approach expiration
- Non compliant vendors flagged in admin dashboard
- AES 256 GCM encryption with key rotation for PHI data
- HIPAA compliant audit logging (every action on sensitive data tracked with IP, user agent, details)

**Tech:** PHP, WordPress, TCPDF, SMTP2GO, AES 256 GCM encryption, custom REST API, Alpine.js

[Full project details](projects/06-document-compliance.md)

