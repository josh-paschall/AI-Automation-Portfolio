# AI Voice and SMS Agent System

## Overview

Multi channel AI communication platform for service businesses. Each business gets a dedicated phone number that handles inbound calls with AI, sends and receives SMS, and routes everything into a CRM pipeline. The AI acts like a receptionist: answers questions, collects caller information, books appointments, and notifies the business owner with a summary.

## Problem It Solves

Small service businesses (HVAC, plumbers, contractors) miss calls constantly. They're on a job site, can't answer the phone, and the lead goes to a competitor. Existing solutions are either too expensive (ServiceTitan at $245+/tech/month) or too basic (voicemail). This system answers every call, handles the conversation intelligently, and creates a lead record before the business owner even knows someone called.

## How It Works

### Inbound Call Flow
1. Customer calls the business number (Twilio)
2. Call routes to Voice AI agent (Retell platform for MVP, custom Pipecat/LiveKit pipeline in development)
3. AI follows a business specific prompt: greets caller, asks what they need, collects contact info, checks availability
4. Call is recorded and transcribed (Whisper)
5. Claude generates a summary: who called, what they need, urgency level, next steps
6. Lead record created in CRM with full transcript, recording playback, and summary
7. Business owner gets an SMS and email notification with the summary

### SMS Flow
1. Customer texts the business number (same Twilio number)
2. AI responds based on business context and conversation history
3. If the business owner sends a manual reply from the CRM, AI backs off (human handoff detection)
4. AI stays quiet until a configurable timeout (default 2 hours), then resumes if needed
5. All messages threaded in the contact's activity timeline

### Outbound Calling (CRM Dialer)
1. Business owner clicks "Call" on a contact record in the CRM
2. Twilio calls the business owner's personal cell first
3. Once they pick up, Twilio bridges to the customer
4. Customer sees the business number on caller ID (not the owner's personal number)
5. Call recorded and logged in CRM

## Technical Details

### Voice AI Pipeline (Two Track Strategy)

**Track 1: Retell (Production MVP)**
- Retell handles the audio pipeline (STT, TTS, turn detection, interruption handling)
- Claude provides the conversational intelligence via tool calls
- Cost: ~$0.085/min bundled (Retell $0.07 + Twilio $0.015)
- Advantage: fast to deploy, handles all the hard audio engineering

**Track 2: DIY (In Development)**
- Pipecat/LiveKit for audio streaming and orchestration
- Deepgram Nova 2 for speech to text ($0.007/min)
- OpenAI TTS Standard for text to speech ($0.015/min)
- Claude Haiku for conversational logic ($0.005/min)
- Cost: ~$0.037/min total (56% cheaper than Retell)
- Both tracks use the same prompts, tools, and knowledge base. Just swapping the audio layer.

### Twilio Infrastructure
- Subaccounts per business (API provisioned, no manual setup)
- Local number provisioning with CNAM branding
- 10DLC registration for SMS compliance (automated through Twilio APIs)
- SIP trunking for voice routing
- Call recording with dual channel audio
- Number porting support (businesses keep their existing number)

### Knowledge Base Per Business
- Business name, hours, service area, services offered
- FAQ responses (configurable by business owner)
- Scheduling rules and availability
- Emergency handling (after hours routing)
- What NOT to say (no pricing over the phone, no medical advice, etc.)

### CRM Integration
- Lead created on every inbound contact (call, SMS, chat, form)
- Unified activity timeline per contact (all channels in one view)
- Pipeline stages: new, contacted, qualified, proposal, won, lost
- Automated follow up triggers based on status changes

## Database Schema (Key Tables)

```sql
-- Phone numbers provisioned per business
mdk_phone_numbers (
    id, number_uid, listing_id,
    phone_number, provider, twilio_sid, twilio_subaccount_sid,
    retell_agent_id, cnam_status, cnam_display_name,
    ten_dlc_campaign_sid, purpose, port_status,
    sms_enabled, voice_enabled, active, monthly_cost
)

-- Communication settings per business
listing_meta (
    comms_personal_cell, comms_outbound_recording,
    comms_sms_ai_enabled, comms_sms_ai_handoff_timeout,
    comms_sms_human_takeover, comms_notification_sms,
    comms_notification_email, comms_forwarding_number
)
```

## Cost Breakdown (Per Business at 200 min/month)

| Component | Retell Stack | DIY Stack |
|-----------|-------------|-----------|
| Voice AI | $14.00 | $5.40 |
| Twilio voice | $3.00 | $3.00 |
| Twilio number | $1.15 | $1.15 |
| Twilio SMS | ~$0.50 | ~$0.50 |
| Total | ~$18.65/mo | ~$10.05/mo |

## Code: AI Estimate Drafting (CRM Integration)

The CRM includes AI powered estimate drafting. A contractor adds assessment notes to a job, and Claude Haiku analyzes those notes against the price book to suggest line items with quantities and pricing.

```php
function mck_rest_ai_estimate_draft(WP_REST_Request $req) {
    global $wpdb;

    $job_uid = sanitize_text_field($req->get_param('job_uid'));

    // Load job
    $job = $wpdb->get_row($wpdb->prepare(
        "SELECT * FROM " . mck_table('mck_jobs') . " WHERE job_uid = %s", $job_uid
    ));

    // Gather context: assessment notes from activity log
    $assessment_notes = $wpdb->get_results($wpdb->prepare(
        "SELECT details, created_at FROM " . mck_table('mck_activity_log') . "
        WHERE entity_type = 'job' AND entity_uid = %s AND action = 'assessment_note'
        ORDER BY created_at ASC", $job_uid
    ));

    // Gather context: price book (active products)
    $products = $wpdb->get_results(
        "SELECT product_uid, name, category, price, unit, tier
        FROM " . mck_table('mck_products') . " WHERE active = 1
        ORDER BY category, name"
    );

    // Build prompt with job context, notes, and price book
    $system_prompt = "You are a contractor estimation assistant.
        Analyze assessment notes and suggest estimate line items
        using products from the price book when applicable.";

    $user_prompt = "## Assessment Notes\n{$notes_text}\n"
                 . "## Price Book\n{$price_book_text}\n"
                 . "Respond with JSON: {items: [{description, product_uid, "
                 . "quantity, unit_price, reasoning}]}";

    $result = mck_call_claude($system_prompt, $user_prompt);

    return rest_ensure_response([
        'suggested_items' => $parsed['items'],
        'ai_notes'        => $parsed['notes'],
        'tokens_used'     => $result['tokens_used'],
    ]);
}
```

## Code: Stripe Checkout with Webhook Verification

```php
// Create Stripe Checkout Session for invoice payment
function mck_rest_invoices_create_checkout(WP_REST_Request $req) {
    $invoice_uid = sanitize_text_field($req->get_param('invoice_uid'));

    $session = \Stripe\Checkout\Session::create([
        'payment_method_types' => ['card'],
        'line_items' => [[ 'price_data' => [
            'currency' => 'usd',
            'product_data' => ['name' => "Invoice {$invoice->invoice_number}"],
            'unit_amount' => intval($invoice->total * 100),
        ], 'quantity' => 1 ]],
        'mode' => 'payment',
        'success_url' => $success_url,
        'metadata' => ['invoice_uid' => $invoice_uid],
    ]);

    return rest_ensure_response(['checkout_url' => $session->url]);
}

// Webhook: verify Stripe signature, record payment
function mck_rest_stripe_webhook(WP_REST_Request $req) {
    $payload = $req->get_body();
    $sig_header = $_SERVER['HTTP_STRIPE_SIGNATURE'] ?? '';

    $event = \Stripe\Webhook::constructEvent(
        $payload, $sig_header, $webhook_secret
    );

    if ($event->type === 'checkout.session.completed') {
        $invoice_uid = $event->data->object->metadata->invoice_uid;
        // Record payment, update invoice status, log activity
    }
}
```

## CRM Data Architecture

22 custom database tables per tenant site (not wp_posts):

```
mck_companies     - Business entities
mck_contacts      - People linked to companies
mck_properties    - Service locations with geocoding
mck_leads         - Pipeline (new > contacted > qualified > proposal > won/lost)
mck_estimates     - Estimates with e-signature + PDF generation
mck_estimate_items - Line items with good/better/best tiers
mck_jobs          - Full lifecycle (10 status stages)
mck_job_tasks     - Checklists with time tracking
mck_invoices      - Stripe payment tracking
mck_invoice_items - Invoice line items
mck_products      - Price book with cost/margin tracking
mck_events        - Calendar with entity linking
mck_contracts     - E-signature with public signing URL
mck_activity_log  - Polymorphic audit trail
mck_email_logs    - Delivery tracking via SMTP2GO
... (22 tables total, 83+ REST API endpoints, 29 dashboard sections)
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Voice AI (MVP) | Retell AI |
| Voice AI (DIY) | Pipecat/LiveKit + Deepgram + OpenAI TTS |
| Telephony | Twilio (voice, SMS, SIP, recording) |
| LLM | Claude Sonnet (conversation), Claude Haiku (AI drafting) |
| Transcription | OpenAI Whisper (local on VPS) |
| Backend | Python/Flask (voice) + PHP/WordPress (CRM) |
| CRM | Custom 22 table schema, 83+ REST endpoints |
| Storage | Cloudflare R2 (call recordings) |
| Server | Liquid Web Cloud VPS (6 core, 24GB RAM) |

## Status

CRM platform built and operational (22 tables, 83+ REST endpoints, 29 dashboard sections, AI estimate drafting, Stripe payments, SMTP2GO email, PDF generation). Voice AI system architected with build estimates and cost analysis complete. Twilio account provisioned. Retell integration next.
