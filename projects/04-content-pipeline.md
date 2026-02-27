# AI Content Pipeline (Video to Multi Platform)

## Overview

Automated content repurposing system. Client records one video. The platform transcribes it, generates a full week of derivative content (blog post, social clips, email newsletter, podcast audio), processes the video (captions, clips, aspect ratios), and posts everything to connected social accounts after client approval.

## Problem It Solves

Small business owners and consultants know they should create content but don't have the time or team to turn one piece of content into 10. They record a video, post it to YouTube, and that's it. This system takes that same video and automatically generates content for every platform they're on, formatted correctly for each one.

## How It Works

### Processing Pipeline

```
1. Client uploads video (chunked upload, direct to server)
   |
2. Whisper transcription (local, no API cost)
   |-- Full transcript with timestamps
   |-- Speaker diarization (future)
   |
3. Claude content generation (from transcript)
   |-- YouTube: title, description, tags, thumbnail brief
   |-- Blog post: full article with headers, links, CTAs
   |-- Social posts: platform specific (Instagram, Facebook, TikTok, LinkedIn, X)
   |-- Email newsletter: summary + key takeaways
   |-- Podcast: intro/outro script for audio only version
   |
4. FFmpeg video processing
   |-- Caption burn in (styled SRT/ASS subtitles)
   |-- Short form clip extraction (best moments)
   |-- Aspect ratio conversion (16:9 -> 9:16 for Reels/TikTok, 1:1 for feed)
   |-- Audio normalization
   |-- Intro/outro concatenation
   |
5. Client review (content calendar UI)
   |-- Preview all generated content
   |-- Edit text with TipTap rich text editor
   |-- Approve or request changes per piece
   |
6. Scheduled posting (Late API)
   |-- YouTube, Instagram, Facebook, TikTok, LinkedIn, X
   |-- Google Business Profile, Pinterest, Reddit, Bluesky
   |-- Staggered across the week per content calendar
```

### In Browser Video Editor

Before processing, clients can:
- Trim start/end points
- Mark clip segments for short form extraction
- Preview caption placement
- Select aspect ratio outputs

Editor uses video.js with frame accurate seeking. Edit decisions are saved as a job manifest (JSON) that FFmpeg processes server side. No client side rendering.

### Content Calendar

FullCalendar.js based view showing:
- Scheduled posts across all platforms
- Color coded by platform and content type
- Drag to reschedule
- Click to preview/edit
- Status indicators (draft, approved, posted, failed)

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Dashboard | WordPress Multisite, Alpine.js, TailAdmin CSS |
| Rich Text Editor | TipTap (ProseMirror based) |
| Video Player | video.js (frame accurate seeking) |
| Calendar | FullCalendar.js |
| Backend API | PHP (WordPress REST API) |
| Processing Engine | Python 3.11, Flask |
| Transcription | OpenAI Whisper (local, medium model, no API cost) |
| Content AI | Claude Sonnet |
| Video Processing | FFmpeg (server side) |
| Image Generation | Together.ai FLUX (thumbnail concepts) |
| Social Posting | Late API ($49/mo, 50 profiles, unlimited posts) |
| Payments | Stripe + Stripe Connect (partner revenue share) |
| Storage | Local VPS (540GB) + Cloudflare R2 (archive) |
| Server | Liquid Web Cloud VPS (6 core, 24GB RAM, Ubuntu 24.04) |

## Business Model

- Partnership with a sales partner who brings clients (25% revenue share via Stripe Connect)
- Pricing: $497/mo (Starter), $997/mo (Growth), $1,497/mo (Scale)
- 3 founding clients from partner's existing manual workflow (day one revenue)
- Platform reuses ~60-80% of code from existing projects (onboarding, dashboard, calendar, billing)

## Key Technical Decisions

- **Local Whisper over API:** VPS has 24GB RAM, easily runs Whisper medium model. Saves ~$0.006/min in API costs. At scale (50+ videos/month), the savings are significant.
- **Server side FFmpeg over client side:** Video processing is CPU intensive. Client browsers can't handle it reliably. Server side processing with job queue ensures consistent output quality.
- **Late API over native platform APIs:** One integration covers 10+ platforms. $49/month flat rate vs managing OAuth for each platform individually.
- **TipTap over basic textarea:** Clients need to edit blog posts and social captions with formatting. TipTap gives them a proper editor without building one from scratch.

## Status

Basic version operational. Transcription, content generation, and video processing pipeline working. Dashboard and approval workflow in development. Late API account active for multi platform posting. Partnership agreement in place with founding clients ready. Reusing substantial code from existing projects (WordPress multisite, dashboard framework, calendar, Stripe billing).
