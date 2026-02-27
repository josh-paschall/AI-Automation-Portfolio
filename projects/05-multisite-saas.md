# WordPress Multisite SaaS with Auto Provisioning

## Overview

Production SaaS platform running on WordPress Multisite. Customer pays through Stripe, site is auto provisioned (cloned from template, subdomain created, DNS configured, SSL issued, onboarding email sent). Includes a member dashboard, admin network panel, subscription enforcement, and custom domain mapping. ~16K lines PHP in the core plugin.

## Problem It Solves

Launching a SaaS on WordPress Multisite requires solving a chain of infrastructure problems: site cloning without data corruption, DNS automation, SSL provisioning, subscription lifecycle management, and multi tenant isolation. This platform handles all of it so each new customer gets a fully configured site within minutes of payment.

## How It Works

### Provisioning Pipeline

```
Customer Purchase (Stripe/WooCommerce)
    |
    v
Payment confirmed via webhook
    |
    v
Async clone scheduled (WP-Cron, prevents timeout)
    |
    v
Template site cloned
    |-- Pages, posts, menus, media
    |-- Serialization safe URL replacement
    |-- Elementor/ACF data preserved correctly
    |
    v
Subdomain created (RunCloud API)
    |
    v
DNS record added (Cloudflare API)
    |
    v
SSL provisioned (automatic via Cloudflare)
    |
    v
Onboarding email sent
    |
    v
Dashboard ready for customer
```

### Race Condition Prevention

```php
public function schedule_site_clone_on_activation($subscription) {
    // Skip renewals (only clone on first activation)
    if (get_post_meta($subscription->get_id(), '_mdk_cloned_site_id', true)) return;

    // Prevent duplicate runs
    if (get_post_meta($subscription->get_id(), '_mwsk_clone_status', true) === 'provisioning') return;

    update_post_meta($subscription->get_id(), '_mwsk_clone_status', 'provisioning');
    wp_schedule_single_event(time(), 'mwsk_process_site_clone',
        array($subscription->get_id()));
    spawn_cron();
}
```

### Custom Domain Mapping

Customers can use their own domain. No plugins needed.

```
Add Domain -> pending_dns -> pending_ssl -> verification -> ready -> active
```

Uses custom `sunrise.php` for domain resolution at the WordPress bootstrap level. Cloudflare API handles DNS verification and SSL. RunCloud API adds the domain to the server configuration.

### Subscription Enforcement

- 7 day grace period on failed payment (site stays active, warning shown)
- 30 day grace period on cancellation (site accessible but read only)
- After grace period: site deactivated, admin notified
- Reactivation on payment resume (no data loss)

## Plugin Architecture (~16K Lines PHP)

```
multisite-saas-plugin/
├── includes/
│   ├── class-mwsk-site-cloner.php           # Clone orchestration
│   ├── class-mwsk-subscription-enforcer.php # Grace periods, locking
│   ├── class-mwsk-custom-domain-manager.php # Cloudflare + RunCloud
│   ├── class-mwsk-domain-manager.php        # sunrise.php mapping
│   └── class-mwsk-checkout-fields.php       # Subdomain at checkout
├── core/
│   ├── url-replacement.php    # Serialization safe search/replace
│   └── media-clone.php        # Background media copying
└── templates/woocommerce/     # Custom checkout + provisioning status
```

### Serialization Safe URL Replacement

WordPress stores serialized data (Elementor layouts, ACF fields, widget settings). A naive find/replace breaks serialized strings because the character count changes. The URL replacement engine:

1. Detects serialized strings
2. Unserializes them
3. Replaces URLs at the value level
4. Re serializes with correct byte counts
5. Handles nested serialization (serialized data inside serialized data)

### Custom Elementor Widgets (14+)

Two categories built for directory functionality:

**Universal widgets:** Grid layouts, map views, search/filter, dashboard components
**Single listing widgets:** About section, services, locations, reviews, contact forms

Each widget:
- Full Elementor controls panel (layout, typography, colors, content source)
- Pulls from custom post types and ACF fields
- Context aware (archive widgets vs page widgets)
- Shortcode fallbacks for non Elementor pages

## Member Dashboard

Custom dashboard (not wp-admin) built with TailAdmin CSS:
- Site management and settings
- Subscription status and billing
- Custom domain setup wizard
- Analytics overview
- Content management

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Platform | WordPress Multisite (subdomain architecture) |
| Payments | WooCommerce + WooCommerce Subscriptions + Stripe |
| Server Management | RunCloud API |
| DNS/SSL | Cloudflare for SaaS API |
| Dashboard | Custom PHP + TailAdmin CSS + Alpine.js |
| Page Builder | Elementor Pro (14+ custom widgets) |
| Custom Fields | ACF Pro |
| Hosting | Liquid Web Cloud VPS |

## Key Technical Decisions

- **Async cloning via WP-Cron:** Large template sites take 30+ seconds to clone. Doing this synchronously would timeout the checkout flow. WP-Cron with `spawn_cron()` runs it in the background.
- **Custom sunrise.php over domain mapping plugins:** Plugins like Mercator add overhead on every request. Custom sunrise.php does one database lookup at bootstrap, no plugin loading required.
- **Serialization safe replacement over wp-cli search-replace:** Needed more control over the replacement process, especially for nested Elementor data structures.
- **RunCloud API over manual server config:** Automated subdomain and domain creation. No SSH needed for routine provisioning.

## Status

Built and deployed. Core provisioning pipeline operational (payment to site creation to DNS to SSL). CRM features (lead pipeline, estimates, invoicing, AI voice) actively being added. 14+ custom Elementor widgets in production.
