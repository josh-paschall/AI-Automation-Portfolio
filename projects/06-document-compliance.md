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

### Encryption (HIPAA Ready)

```
AES 256 GCM encryption for all sensitive data
    |-- Key rotation support
    |-- Signed URLs for secure file access (time limited)
    |-- Files stored outside web root
    |-- Decryption only on authenticated request
```

### Audit Logging

Every action logged with:
- Entity type and ID (which vendor, which document)
- Action performed (upload, view, download, delete, edit)
- User who performed it
- IP address and user agent
- Full JSON details of what changed
- Timestamp

This audit trail satisfies HIPAA requirements for access logging on protected data.

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
