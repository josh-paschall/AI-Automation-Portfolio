# Automated Document Generation and Compliance Tracking

## Overview

Two systems working together: (1) Automated PDF generation from form submissions with e signatures, and (2) a vendor compliance tracking module that monitors document uploads, expiration dates, and sends automated reminders. Built for a healthcare compliance platform handling sensitive data with AES 256 GCM encryption.

## Problem It Solves

Organizations that manage vendors (contractors, suppliers, service providers) need to track compliance documents: W9s, insurance certificates, business licenses. These documents expire. When they expire, the vendor becomes non compliant. Tracking this manually across dozens or hundreds of vendors means things slip through the cracks. This system automates the entire lifecycle from document collection to expiration monitoring to reminder emails.

## Document Generation

### Form to PDF Pipeline

```
User fills out dynamic form (multi phase, conditional logic)
    |
    v
Form data validated and stored
    |
    v
TCPDF generates branded PDF
    |-- Company logo, headers, formatting
    |-- Merge fields populated from form data
    |-- Multi page support
    |
    v
E-signature capture (canvas based)
    |-- Finger/mouse drawn signature
    |-- Base64 encoded, stored securely
    |-- Signer IP, timestamp, user agent logged
    |
    v
Signed PDF stored + audit log entry created
```

### Dynamic Form Builder

Visual form builder with drag and drop (SortableJS), conditional logic, and multi phase support:

- **Field types:** Text, textarea, number, date, select, radio, checkbox, file upload, section headers
- **Conditional logic:** Show/hide fields based on other field values, with circular dependency detection
- **Multi phase forms:** Break long forms into steps, each phase can have a delay (e.g., "Phase 2 unlocks 30 days after Phase 1")
- **Alpine.js frontend:** Reactive state management for the builder UI (~1,500 lines JS)

## Compliance Tracking

### Document Lifecycle

```
Vendor uploads document (W9, insurance cert, license)
    |
    v
System extracts/records expiration date
    |
    v
Document stored (encrypted at rest, AES 256 GCM)
    |
    v
Monitoring loop (daily cron)
    |-- 60 days before expiration: first reminder email
    |-- 30 days before expiration: second reminder
    |-- 7 days before expiration: urgent reminder
    |-- Expired: vendor flagged as non compliant
    |
    v
Admin dashboard shows compliance status across all vendors
    |-- Green: compliant
    |-- Yellow: expiring soon
    |-- Red: expired / missing documents
```

### Compliance Dashboard

- Overview cards: total vendors, compliant %, expiring this month, overdue count
- Filterable vendor table with compliance status badges
- Click into vendor: see all documents, expiration dates, reminder history
- Bulk actions: send reminders, export compliance report, flag vendors

## Security

This platform handles protected health information (PHI) for healthcare organizations. HIPAA compliance requires encryption at rest, authenticated access controls, and a complete audit trail of every action taken on sensitive data. These aren't optional features. They're regulatory requirements.

### Code: AES 256 GCM Encryption

All sensitive data encrypted at rest using AES 256 GCM with key rotation support. The envelope format is compact and self describing: `<key_id>.<iv>.<tag>.<ciphertext>`, all URL safe base64 encoded.

```php
function tpr_encrypt($plaintext) {
    if (!tpr_crypto_has_key()) {
        return new WP_Error('tpr_no_key', 'Encryption key not configured.');
    }

    $kr = tpr_crypto_keyring();
    $kid = $kr['current_id'];
    $key = $kr['keys'][$kid];

    $pt = (string) $plaintext;
    $iv = random_bytes(12); // 96-bit IV recommended for GCM
    $tag = '';
    $ct = openssl_encrypt($pt, 'aes-256-gcm', $key, OPENSSL_RAW_DATA, $iv, $tag);

    if ($ct === false || $tag === '') {
        return new WP_Error('tpr_encrypt_failed', 'OpenSSL encryption failed.');
    }

    // Compact envelope: key_id.iv.tag.ciphertext (all base64url)
    return $kid . '.' . tpr_b64u_enc($iv) . '.' . tpr_b64u_enc($tag) . '.' . tpr_b64u_enc($ct);
}

function tpr_decrypt($envelope) {
    if (!is_string($envelope) || $envelope === '') {
        return new WP_Error('tpr_bad_input', 'Empty ciphertext.');
    }

    $parts = explode('.', $envelope);
    if (count($parts) !== 4) {
        return new WP_Error('tpr_bad_envelope', 'Ciphertext envelope malformed.');
    }

    [$kid, $iv_b64, $tag_b64, $ct_b64] = $parts;
    $iv  = tpr_b64u_dec($iv_b64);
    $tag = tpr_b64u_dec($tag_b64);
    $ct  = tpr_b64u_dec($ct_b64);

    $kr  = tpr_crypto_keyring();
    $key = $kr['keys'][$kid] ?? null;

    // If key_id not found (rotated?), try all keys as fallback
    $candidates = $key ? [$kid => $key] : $kr['keys'];

    foreach ($candidates as $test_id => $test_key) {
        $pt = openssl_decrypt($ct, 'aes-256-gcm', $test_key, OPENSSL_RAW_DATA, $iv, $tag);
        if ($pt !== false) return $pt;
    }

    return new WP_Error('tpr_decrypt_failed', 'Unable to decrypt with available keys.');
}
```

Key design decisions:
- **96 bit random IV per encryption** (NIST recommended for GCM)
- **GCM authentication tag** prevents tampering (CBC doesn't have this)
- **Key rotation support:** Key ID embedded in envelope. Old keys kept in keyring so previously encrypted data still decrypts after rotation
- **Fallback decryption:** If key ID not found, tries all keys (handles edge cases during rotation)

### Code: HIPAA Compliant Audit Logging

Every action on sensitive data is logged with full context. The audit trail is PHI safe, meaning it records what happened and who did it, but never includes the actual sensitive data values in the log.

```php
function tpr_audit(string $action, array $opts = []): void {
    global $wpdb;

    $event_uid = tpr_uuidv4();
    $user_uid = tpr_current_user_uid();

    $target_table = isset($opts['target_table'])
        ? sanitize_text_field($opts['target_table']) : null;
    $target_id = isset($opts['target_id'])
        ? intval($opts['target_id']) : null;

    $ip_address = sanitize_text_field($_SERVER['REMOTE_ADDR'] ?? '');
    $user_agent = isset($opts['user_agent'])
        ? (string) $opts['user_agent']
        : ($_SERVER['HTTP_USER_AGENT'] ?? '');

    // Process details (JSON encode if array, truncate if too large)
    $details = null;
    if (array_key_exists('details', $opts)) {
        $details = is_array($opts['details'])
            ? wp_json_encode($opts['details'])
            : (string) $opts['details'];

        if (strlen($details) > 65500) {
            $details = substr($details, 0, 65500);
        }
    }

    $wpdb->insert('tpr_audit_logs', [
        'event_uid'    => $event_uid,
        'user_uid'     => $user_uid,
        'action'       => substr($action, 0, 64),
        'target_table' => $target_table,
        'target_id'    => $target_id,
        'details'      => $details,
        'ip_address'   => $ip_address ?: null,
        'user_agent'   => $user_agent ?: null,
        'created_at'   => current_time('mysql'),
    ]);
}
```

Automatic hooks log these events without manual calls:
- `login` / `login_failed` / `logout` (authentication events)
- `rest` (REST API requests for PHI related endpoints only, logs method + route + status + parameter keys but never parameter values)

```php
// REST API audit (PHI-safe: logs keys only, never values)
add_filter('rest_request_after_callbacks', function($response, $handler, $req) {
    $route = $req->get_route();
    if (!tpr_should_audit_route($route)) return $response;

    tpr_audit('rest', [
        'details' => [
            'method'  => $req->get_method(),
            'route'   => $req->get_route(),
            'status'  => $response->get_status(),
            'qs_keys' => array_keys($req->get_params() ?: []), // Keys only, no values
        ],
    ]);

    return $response;
}, 10, 3);
```

## E-Signature System

- HTML5 canvas for signature capture
- Works on desktop (mouse) and mobile (touch/finger)
- Signature stored as base64 PNG
- Each signature records: signer name, email, IP address, user agent, timestamp
- Public signing URL (magic link) so external parties can sign without an account
- Signature embedded into generated PDF

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | PHP, WordPress |
| PDF Generation | TCPDF |
| Form Builder | Alpine.js, SortableJS, custom PHP |
| Encryption | AES 256 GCM (custom PHP module) |
| Email | SMTP2GO (reminders, notifications) |
| Signatures | HTML5 Canvas, base64 encoding |
| Audit Logging | Custom table (entity, action, details JSON, IP, user agent) |
| Dashboard | TailAdmin CSS, Alpine.js |

## Key Technical Decisions

- **AES 256 GCM over AES CBC:** GCM provides authenticated encryption (integrity + confidentiality). CBC only provides confidentiality. For healthcare/compliance data, you need both.
- **Files outside web root:** Uploaded documents are never directly accessible via URL. A PHP handler authenticates the request, decrypts if needed, and serves the file. Prevents direct file access exploits.
- **TCPDF over DOMPDF:** TCPDF handles complex layouts (multi column, embedded images, headers/footers) more reliably for form based documents. DOMPDF is better for HTML to PDF conversion.
- **Custom audit logging over plugin:** Needed entity level granularity that no WordPress audit plugin provides. Custom table with JSON details column gives maximum flexibility for compliance reporting.

## Status

Production. Running on a healthcare compliance platform with 29K+ lines of PHP. Processing real vendor documents and compliance tracking for active organizations.
