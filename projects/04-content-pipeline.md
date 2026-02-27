# AI Content Pipeline (Video to Multi Platform)

## Overview

Automated content repurposing system. Client records one video. The platform transcribes it, generates a full week of derivative content (blog post, social clips, email newsletter, podcast audio), processes the video (captions, clips, aspect ratios), and posts everything to connected social accounts after client approval.

## Problem It Solves

Small business owners and consultants know they should create content but don't have the time or team to turn one piece of content into 10. They record a video, post it to YouTube, and that's it. This system takes that same video and automatically generates content for every platform they're on, formatted correctly for each one.

## How It Works

### Processing Pipeline

```
1. Brand Kit generates LLM Index (company profile for AI context)
   |
2. AI suggests topic + generates two script versions
   |-- Version A: Structured/educational approach
   |-- Version B: Conversational/story driven approach
   |
3. Client reviews scripts via magic link (no login required)
   |
4. Client records video
   |
5. Whisper transcription (local, no API cost)
   |-- Full transcript with timestamps
   |-- Auto queues content generation on completion
   |
6. Claude content generation (transcript + brand context)
   |-- YouTube: title, description, tags, thumbnail brief
   |-- Blog post: 1,000-2,000 words with headers and CTAs
   |-- Social posts: 13 platforms with per platform character limits
   |-- Email newsletter (Growth/Scale plans)
   |
7. FFmpeg video processing
   |-- Short form clip extraction with format conversion
   |-- Aspect ratio conversion (16:9, 9:16, 1:1)
   |-- Text overlays with custom positioning and timing
   |-- Audio normalization
   |
8. Client review via magic link (approve/edit/request revision per item)
   |
9. Scheduled posting (Late API, 10+ platforms)
```

### Brand Kit to AI Pipeline

Every AI generation call receives a "LLM Index," a compiled brand profile that gives the AI full business context. The index is cached and auto regenerates hourly.

```php
function mvk_generate_llm_index($company_uid, $site_id = null) {
    // Compiles markdown-formatted brand profile:
    // - Company name, industry, description, website
    // - Brand voice, tone, custom instructions
    // - Target audience
    // - Seed topics and topics to avoid
    // - Connected social platforms + posting schedule
    // - Content statistics (videos, pieces, projects)
    // - Recent transcripts (last 5)
    // - Recent videos (last 20)
    // - Recent content (last 30)
    // - Active projects
    // - Content type + platform distribution

    // Cached in mvk_companies.llm_index, regenerates every hour
}
```

### Script Generation from Brand Profile

AI generates two script versions from a content brief plus the full brand context. Client picks which approach they want to record.

```python
def generate_scripts(brief_text, llm_index, brand_data=None, target_platforms=None):
    """
    Generate two video script versions from content brief + brand profile.
    Version A: Structured/educational approach
    Version B: Conversational/story-driven approach
    """
    # Build system prompt with full brand context from LLM Index
    # Includes: brand voice, target audience, topics, avoid topics

    # Returns structured JSON:
    # {
    #   "script_version_a": "<h2>Hook</h2><p>...",
    #   "script_version_b": "<h2>Hook</h2><p>...",
    #   "brief_analysis": {
    #     "key_topics": [...],
    #     "suggested_title": "...",
    #     "estimated_duration_minutes": 5,
    #     "target_audience_fit": "..."
    #   }
    # }
```

### Content Generation with Platform Constraints

Each platform gets content formatted to its specific character limits. Content that exceeds limits is truncated at sentence boundaries.

```python
PLATFORM_CHAR_LIMITS = {
    'twitter': 280,
    'bluesky': 300,
    'snapchat': 160,
    'threads': 500,
    'googlebusiness': 1500,
    'instagram': 2200,
    'tiktok': 2200,
    'linkedin': 3000,
    'telegram': 4096,
    'pinterest': 500,
    'reddit': 40000,
    'facebook': 63206,
}

def generate_content(transcript, llm_index, content_type, platform=None, segments=None):
    """
    Generate platform-specific content from video transcript + brand profile.
    Content types: youtube_meta, blog_post, social_caption, thumbnail_brief, newsletter
    """
    # Model selection based on content type:
    # - Sonnet for long form (blog, newsletter, scripts)
    # - Haiku for short form (social captions, metadata)

    # Validates character limits per platform
    # Truncates at sentence boundary if over limit
    # Logs warnings for oversized content
```

### Magic Link Approval System

Clients review and approve content without logging in. Crypto random token, configurable expiry, per item approval workflow.

```php
// Generate magic link for content review
$token = bin2hex(random_bytes(32));  // 64-character crypto-random token

// REST endpoints (no authentication required, token validates):
// POST /mvk/v1/approvals/generate           -> Generate magic link
// GET  /mvk/v1/approvals/validate/{token}   -> Load content for review
// POST /mvk/v1/approvals/{token}/approve-item  -> Approve single item
// POST /mvk/v1/approvals/{token}/revise-item   -> Request revision with notes
// POST /mvk/v1/approvals/{token}/approve-all   -> Bulk approve

// Features:
// - 48 hour default expiry (configurable 1-168 hours)
// - Per item status: review, approved, revision_requested, posted
// - TipTap rich text editor for blog/newsletter edits
// - Character count validation for social posts
// - Progress tracking (approved/pending/revision counts)
// - Email notifications on approval and revision requests
```

### Prompt Template System

Markdown based prompt templates with YAML frontmatter for model selection and variable substitution.

```yaml
---
id: blog_post
name: Blog Post Generator
group: content
model: claude-sonnet-4-20250514
max_tokens: 8192
output_format: json
---
Your prompt body here...
Use {{variable}} for substitution
Use {{#if variable}}...{{/if}} for conditionals
```

Generation modes control cost:
- `test` mode: Always Haiku (low cost during development)
- `live` mode: Sonnet for long form content, Haiku for everything else

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
| Transcription | faster-whisper (local, medium model, no API cost) |
| Content AI | Claude Sonnet (long form), Claude Haiku (short form) |
| Video Processing | FFmpeg (server side) |
| Image Generation | Together.ai FLUX (thumbnail concepts) |
| Social Posting | Late API ($49/mo, 50 profiles, unlimited posts) |
| Payments | Stripe + Stripe Connect (partner revenue share) |
| Storage | Local VPS (540GB) + Cloudflare R2 (archive) |
| Server | Liquid Web Cloud VPS (6 core, 24GB RAM, Ubuntu 24.04) |

## Key Technical Decisions

- **Local Whisper over API:** VPS has 24GB RAM, easily runs faster-whisper medium model. Saves ~$0.006/min in API costs. At scale (50+ videos/month), the savings are significant. Auto queues content generation on transcription completion.
- **Server side FFmpeg over client side:** Video processing is CPU intensive. Client browsers can't handle it reliably. Server side processing with job queue ensures consistent output quality.
- **Late API over native platform APIs:** One integration covers 10+ platforms. $49/month flat rate vs managing OAuth for each platform individually.
- **Magic links over login walls:** Clients are busy. They're not going to create an account and log in to approve a blog post. A secure token link in their email gets them reviewing content in one click.
- **Prompt templates over hardcoded prompts:** Markdown files with YAML frontmatter. Easy to iterate on prompts without code changes. Model selection per template lets you optimize cost vs quality per content type.

## Status

Basic version operational. Transcription, content generation, and video processing pipeline working. Magic link approval system built. Brand kit to AI pipeline generating LLM indexes. Dashboard and content calendar in development. Late API account active for multi platform posting. Reusing substantial code from existing projects (WordPress multisite, dashboard framework, calendar, Stripe billing).
