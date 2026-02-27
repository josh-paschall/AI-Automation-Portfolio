# Multi Brand Social Scanner and Content Intelligence System

## Overview

Two systems on the same infrastructure: (1) a multi brand Reddit engagement scanner that scores conversations for opportunity, drafts persona matched responses, and queues for human approval, and (2) a content intelligence mining system that extracts questions, complaints, sentiment, keywords, location data, and competitive mentions from Reddit communities to feed SEO and content strategy.

## Problem It Solves

**Engagement side:** Target customers post frustrations, ask for recommendations, and discuss problems your product solves across dozens of subreddits. Manually scrolling all of them is a 2+ hour daily time sink. This system automates the discovery and first draft, reducing it to 5 minutes of queue review.

**Content intelligence side:** Reddit is full of real questions, complaints, and recommendations that map directly to blog topics, FAQ pages, and keyword targets. But extracting that data manually is tedious. The mining system scans target communities and produces structured content briefs, FAQ lists, sentiment reports, keyword analysis, and competitive intelligence automatically.

## Architecture

```
CLI Commands
    |
    v
scanner.py (mode-aware: engage vs datamine)
    |
    ├── Late API: search_reddit(query, subreddit)
    ├── Late API: get_feed(subreddit, sort, limit)
    |
    v
database.py (MySQL, dedup, rate limiting)
    |
    ├── is_post_seen() -> skip duplicates
    ├── save_post() -> store with scores
    |
    v
Scoring Engine (per-mode)
    |
    ├── score_post() -> engagement opportunities
    |   ├── Frustration signals (+3)
    |   ├── Shopping signals (+3)
    |   ├── Technical question signals (+2)
    |   ├── Service business context (+2)
    |   ├── Engagement/recency modifiers
    |   └── Returns: score, matched_keywords, opportunity_type
    |
    └── score_post_datamine() -> content intelligence
        ├── Question format detection (+3)
        ├── Complaint/recommendation signals (+3)
        ├── Price/cost discussion (+2)
        ├── Location mentions (+3)
        ├── Competitive mentions (+2)
        └── Returns: score, matched_keywords, intel_type
    |
    v
Output Generation
    |
    ├── output.py -> engagement queue + tracking markdown
    └── datamine_output.py -> structured intelligence
        ├── Content briefs (questions -> blog topics)
        ├── FAQ intelligence
        ├── Sentiment reports
        ├── Keyword/language analysis
        ├── Location intelligence
        └── Competitor mentions
```

## Multi Brand Configuration

Each brand has its own targeting, keywords, and persona profile. Brands are stored in the database, not hardcoded.

```sql
CREATE TABLE sa_brands (
    id INT AUTO_INCREMENT PRIMARY KEY,
    brand_key VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    brand_type ENUM('engage', 'monitor', 'datamine') NOT NULL,
    positioning JSON,
    live_features JSON,
    in_development JSON,
    focus_locations JSON,
    active TINYINT(1) DEFAULT 1
);

CREATE TABLE sa_targets (
    id INT AUTO_INCREMENT PRIMARY KEY,
    brand_key VARCHAR(50) NOT NULL,
    subreddit VARCHAR(100) NOT NULL,
    tier ENUM('primary', 'secondary', 'tertiary') DEFAULT 'primary',
    keywords JSON,
    search_queries JSON,
    UNIQUE KEY uniq_brand_sub (brand_key, subreddit)
);
```

Scan commands:
```bash
python3 scanner.py brand-a primary       # scan one brand, primary subs
python3 scanner.py all primary           # scan all engagement brands
python3 scanner.py datamine-site primary # scan one datamine site
python3 scanner.py datamine primary      # scan all datamine sites
```

## Code: Opportunity Scoring Engine

```python
def score_post(post, keywords, brand_key=None):
    """
    Score a post based on keyword matches and engagement signals.
    Returns (score, matched_keywords, opportunity_type).
    """
    title = (post.get("title") or "").lower()
    body = (post.get("selftext") or "").lower()
    text = f"{title} {body}"
    score = 0
    matched = []

    # Keyword matching
    for kw in keywords:
        if kw.lower() in text:
            matched.append(kw)
            score += 1

    # Frustration signals (+3)
    frustration_words = [
        "hate", "terrible", "awful", "garbage", "worst",
        "frustrated", "nightmare", "broken", "leaving",
        "switching", "bloated", "overpriced", "waste of money",
        "cancelling", "done with", "fed up"
    ]
    for word in frustration_words:
        if word in text:
            score += 3
            matched.append(f"frustration:{word}")
            break

    # Shopping signals (+3)
    shopping_words = [
        "alternative", "what should i use", "looking for",
        "recommend", "best crm", "which crm", "moving from",
        "switching to", "replace", "instead of", "comparison"
    ]
    for word in shopping_words:
        if word in text:
            score += 3
            matched.append(f"shopping:{word}")
            break

    # Engagement and recency modifiers
    if post.get("numComments", 0) >= 10:
        score += 1
    elif post.get("numComments", 0) < 3:
        score -= 1

    created = post.get("createdUtc", 0)
    if created:
        age_hours = (datetime.now(timezone.utc).timestamp() - created) / 3600
        if age_hours < 24:
            score += 1
        elif age_hours > 168:
            score -= 2

    # Classify opportunity type from matched signals
    opp_type = "general"
    for m in matched:
        if m.startswith("frustration:"):
            opp_type = "frustration"
            break
        elif m.startswith("shopping:"):
            opp_type = "shopping"
            break
        elif m.startswith("tech:"):
            opp_type = "question"

    return score, matched, opp_type
```

## Code: Content Intelligence Scoring

Separate scoring model for data mining mode. Not looking for engagement opportunities. Looking for content that informs SEO strategy.

```python
def score_post_datamine(post, keywords, site_key=None):
    """
    Score a post for content intelligence value.
    Questions become blog topics, complaints become FAQ entries,
    location mentions inform geo targeting, competitor mentions
    feed competitive analysis.
    """
    text = f"{(post.get('title') or '').lower()} {(post.get('selftext') or '').lower()}"
    score = 0
    matched = []

    # Question format detection (+3) - these become content briefs
    question_patterns = [
        "how do i", "how can i", "anyone know",
        "what should i", "can someone recommend",
        "best way to", "how much does", "is it worth"
    ]
    for pattern in question_patterns:
        if pattern in text:
            score += 3
            matched.append(f"question:{pattern}")
            break

    # Complaint/frustration (+3) - sentiment data
    # Recommendation requests (+3) - competitive intel
    # Price/cost discussions (+2) - market data
    # Location mentions (+3) - geo targeting
    # ... (additional signal categories)

    return score, matched, intel_type
```

## Code: Content Intelligence Output

The data mining output generates structured markdown files per category:

```python
def generate_content_briefs(posts, site_key, site_config, output_dir):
    """
    Generate content briefs from question-format posts.
    Each brief maps a real Reddit question to a potential
    blog post or landing page topic.
    """
    question_posts = [p for p in posts
                      if any(m.startswith("question:") for m in json.loads(p["keyword_matches"] or "[]"))]

    # Sort by opportunity score descending
    question_posts.sort(key=lambda p: p["opportunity_score"], reverse=True)

    lines = [f"# Content Briefs - {site_config['name']}\n"]
    lines.append(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M')}\n")
    lines.append(f"Source: Reddit scan of {len(question_posts)} question-format posts\n\n")

    for post in question_posts[:50]:  # Top 50
        lines.append(f"### {post['title']}")
        lines.append(f"- **Sub:** r/{post['subreddit']} | **Score:** {post['opportunity_score']}")
        lines.append(f"- **Comments:** {post['comment_count']} | **URL:** {post['url']}")
        matches = json.loads(post["keyword_matches"] or "[]")
        if matches:
            lines.append(f"- **Signals:** {', '.join(matches)}")
        lines.append("")

    write_output(output_dir, "content-briefs.md", "\n".join(lines))
```

## Persona System (Architecture Level)

Each brand gets a persona profile that controls how draft responses are generated. The system uses engagement escalation levels:

| Level | Approach | When |
|-------|----------|------|
| 1 | Helpful comment, no mention of product | Building karma, answering questions |
| 2 | Share relevant experience, soft mention | Someone has a problem you've solved |
| 3 | Direct response, open about what you're building | Someone is explicitly shopping |
| 4 | Flag for human takeover | Serious interest, DM conversation |

Persona profiles include: voice/tone rules, platform specific calibration, words to use and avoid, response templates by scenario type, and strict rules about what never to claim. Details are kept private per brand.

## Rate Limiting

```
- Max 1 comment per 20 minutes
- Max 3 comments per hour, 10 per day
- Max 1 per subreddit per hour (spread across subs)
- Never reply twice in the same thread
- No posting between 2am and 7am
- 60 min cooldown if a reply gets downvoted
```

## Results (First Scan)

| Site | Posts Scanned | Content Briefs | FAQ Items |
|------|--------------|----------------|-----------|
| Service Directory A | 1,345 | 570 | Extracted |
| Service Directory B | 1,375 | 616 | Extracted |
| Professional Directory C | 1,143 | 409 | Extracted |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | Python 3.12 |
| Social API | Late API (Reddit search, feed, comments) |
| Database | MySQL (shared with WordPress, sa_ prefix) |
| Output | Markdown files synced to Dropbox |
| Server | Liquid Web Cloud VPS (Ubuntu 24.04) |

## Status

Scanner and data mining system operational. Running scans across 5 engagement brands and 3 datamine sites. Content intelligence pipeline producing structured output. WordPress admin plugin with feed, conversations, brand management, and SMS integration built. Engagement reply posting in development.
