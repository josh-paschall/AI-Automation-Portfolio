# SMS Review Automation with AI Intent Analysis

## Overview

Fully automated SMS conversation engine that collects customer reviews through text messages. Business owner enters a name and phone number, the system handles the entire conversation: consent, rating, open ended questions, AI sentiment analysis, review generation, and posting to WordPress. Production system handling real customer conversations.

## Problem It Solves

Getting customer reviews is tedious. Most businesses either forget to ask or send a generic email that gets ignored. SMS has a 98% open rate. This system uses conversational SMS to walk customers through a quick review process that feels natural, not like a form. Negative reviews get filtered privately so the business gets the feedback without the public damage.

## How It Works

```
Business submits customer info (WordPress dashboard form)
    |
    v
Flask API receives webhook, creates conversation record
    |
    v
First SMS sent: "Hi [name], [business] here. Quick review?"
    |
    v
Customer replies -> State machine routes response
    |
    ├── Step 0: Consent (Yes/No/Opt-out detection)
    ├── Step 1: Rating (1-5, with AI fallback for unclear responses)
    ├── Steps 2-5: Open-ended questions
    └── Done: AI generates review text from conversation
    |
    v
Rating >= 3: Send Google review link with pre-generated text
Rating < 3: Thank customer privately, DO NOT send public link
    |
    v
Full conversation log posted to WordPress via REST API
```

## Code: AI Review Generation

```python
def generate_review_with_ai(responses, customer_name, listing_name):
    """Use GPT to generate a natural review from SMS conversation responses"""
    conversation_text = ""
    rating = "5"

    for response in responses:
        if response.get('question_number') == 0:
            continue  # Skip consent step

        question = response.get('question', '')
        answer = response.get('response', '')
        conversation_text += f"Q: {question}\nA: {answer}\n\n"

        if 'scale of 1-5' in question.lower():
            rating_match = re.search(r'[1-5]', answer)
            if rating_match:
                rating = rating_match.group()

    prompt = f"""
    Generate a natural, authentic customer review based on this
    conversation with {customer_name} about {listing_name}.

    Conversation:
    {conversation_text}

    Create a review that:
    - Sounds like a real customer wrote it
    - Incorporates their specific feedback naturally
    - Is 50-150 words
    - Maintains their tone and sentiment
    - Includes specific details they mentioned

    Respond with ONLY a JSON object:
    {{"review_title": "Brief title", "review_text": "Natural review",
      "rating": {rating}, "would_recommend": true/false}}
    """

    response = openai_client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=300,
        temperature=0.3
    )

    return json.loads(response.choices[0].message.content.strip())
```

## Code: Rating Extraction with AI Fallback

Customers don't always respond with a clean number. Some say "4 stars", some say "pretty good", some say "ehh it was ok I guess." The system tries regex first, falls back to AI for unclear responses.

```python
def parse_rating_from_response(response):
    """Parse numeric rating from customer response (0-5, including decimals)"""
    # Look for decimal numbers like 4.5, 3.2
    decimal_match = re.search(r'\b([0-5](?:\.[0-9])?)\b', response)
    if decimal_match:
        rating = float(decimal_match.group(1))
        if 0 <= rating <= 5:
            return rating

    # Look for single digits 0-5
    digit_match = re.search(r'\b([0-5])\b', response)
    if digit_match:
        return float(digit_match.group(1))

    return None  # Triggers AI fallback


def use_ai_to_extract_rating(response):
    """Use AI when customer response is unclear (e.g., 'pretty good')"""
    prompt = f"""
    Extract a numeric rating (0-5) from this customer response.

    Customer response: "{response}"

    Guidelines:
    - 0 = terrible  |  1 = very poor  |  2 = poor
    - 3 = average   |  4 = good       |  5 = excellent

    Respond with ONLY a single number between 0-5.
    """

    response = openai_client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=5, temperature=0.1
    )

    return float(response.choices[0].message.content.strip())
```

## Code: Bidirectional Webhook Security

WordPress and Flask communicate through secured webhooks. Both sides verify a shared HMAC secret before processing:

```python
# Flask verifies inbound webhooks from WordPress
def verify_webhook_secret(req):
    provided = req.headers.get('X-Webhook-Secret', '')
    return hmac.compare_digest(provided, Config.WEBHOOK_SECRET)
```

```php
// WordPress verifies callbacks from Flask
$provided = $request->get_header('X-Webhook-Secret');
if (!hash_equals($configured, $provided)) {
    return new WP_Error('invalid_secret', 'Unauthorized', ['status' => 401]);
}
```

## Code: Phone Number Normalization

```python
def normalize_phone_number(phone):
    """Normalize any phone format to E.164 (+1XXXXXXXXXX)"""
    digits = re.sub(r'\D', '', phone)

    if len(digits) == 10:
        digits = '1' + digits

    if len(digits) == 11 and digits.startswith('1'):
        return '+' + digits

    if phone.startswith('+'):
        return '+' + digits

    return phone  # Return as-is if can't normalize
```

## Conversation State Machine

| Step | System Sends | Customer Does | Handler |
|------|-------------|---------------|---------|
| 0 | Review request | Yes / No / STOP | Consent check, opt out detection |
| 1 | "Rate 1 to 5" | Number or text | Regex parse, AI fallback |
| 2+ | Open ended questions | Free form | Stored to conversation log |
| Done | Thank you + review link (if positive) | Click link | AI generates review, posts to WP |

## Negative Review Filtering

Reviews with rating <= 2 never get a public review link. The business still gets the full conversation and feedback privately through the dashboard. This protects the business while still collecting honest feedback.

## WordPress Dashboard

Member dashboard section in the directory platform:
- **Stats cards:** Total requests, awaiting response, reviews received, conversion rate
- **Request history table:** Every conversation with status, rating, timestamp
- **Submission form:** Customer name + phone, one click to start
- **Replay button:** Repost completed reviews to WordPress if the first post failed

## File Structure

```
Flask API (PythonAnywhere)          WordPress Theme
├── review_automation.py            ├── rest-api.php (webhook endpoints)
│   ├── ReviewConversationDB        ├── ajax-handlers.php
│   ├── generate_review_with_ai()   └── review-requests.php (dashboard)
│   ├── parse_rating_from_response()
│   └── normalize_phone_number()    Dashboard Section
├── httpsms_client.py               └── review-requests.php
│   ├── HttpSMSClient class             ├── Stats cards
│   ├── JWT signature verification       ├── Past requests table
│   └── Message event parsing            └── Submission form
└── .env (API keys)
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | Python 3.11, Flask |
| SMS | httpSMS (Android gateway) |
| AI | OpenAI GPT 3.5 (intent analysis, review generation) |
| CMS | WordPress (dashboard, REST API, form handling) |
| Database | CSV based conversation tracking (Flask side) |
| Hosting | PythonAnywhere (Flask), WP Engine (WordPress) |
| Email | SMTP2GO (notification emails) |

## Status

Production. Running for active directory sites. Processing real customer review conversations.
