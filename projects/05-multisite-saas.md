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

WordPress stores serialized data (Elementor layouts, ACF fields, widget settings). A naive SQL `REPLACE()` breaks serialized strings because PHP serialization includes string length markers. Change the URL, the length marker is wrong, and the entire serialized blob corrupts.

**The problem:**
```
Before: s:45:"https://template.site.com/image.jpg"
After SQL REPLACE: s:45:"https://newsite.com/image.jpg"
                   ^^^^^ length is now wrong = corrupt data
```

**The solution:** Recursive unserialize, replace at value level, re serialize (PHP recalculates correct lengths automatically).

```php
private function recursive_unserialize_replace($search, $replace, $data, $serialised = false) {
    try {
        // If data is serialized, unserialize it first
        if (is_string($data) && ($unserialized = @unserialize($data)) !== false) {
            $data = $this->recursive_unserialize_replace($search, $replace, $unserialized, true);
        } elseif (is_array($data)) {
            $_tmp = array();
            foreach ($data as $key => $value) {
                $_tmp[$key] = $this->recursive_unserialize_replace($search, $replace, $value, false);
            }
            $data = $_tmp;
            unset($_tmp);
        } elseif (is_object($data)) {
            if (get_class($data) === '__PHP_Incomplete_Class') {
                return $data;
            }
            $_tmp = $data;
            $props = get_object_vars($data);
            foreach ($props as $key => $value) {
                $_tmp->$key = $this->recursive_unserialize_replace($search, $replace, $value, false);
            }
            $data = $_tmp;
            unset($_tmp);
        } elseif (is_string($data)) {
            $data = str_replace($search, $replace, $data);
        }

        // Re-serialize with correct byte counts
        if ($serialised) {
            return serialize($data);
        }
    } catch (Exception $e) {
        error_log('recursive_unserialize_replace error: ' . $e->getMessage());
    }

    return $data;
}
```

The database operation finds matching rows, runs each value through the recursive replacer, and only updates rows where the value actually changed:

```php
private function serialized_search_replace($table, $column, $search, $replace) {
    global $wpdb;

    $primary_key = $this->get_primary_key($table);
    if (!$primary_key) return 0;

    $rows = $wpdb->get_results($wpdb->prepare(
        "SELECT `{$primary_key}`, `{$column}` FROM `{$table}` WHERE `{$column}` LIKE %s",
        '%' . $wpdb->esc_like($search) . '%'
    ));

    $updated = 0;
    foreach ($rows as $row) {
        $original_value = $row->$column;
        $new_value = $this->recursive_unserialize_replace($search, $replace, $original_value);

        if ($new_value !== $original_value) {
            $wpdb->update($table, array($column => $new_value),
                array($primary_key => $row->{$primary_key}));
            $updated++;
        }
    }

    return $updated;
}
```

Handles replacement across: post_content, post_excerpt, guid, all postmeta (Elementor `_elementor_data`, ACF fields, widget settings), options table, and both normal and escaped URLs (for JSON stored in Elementor).

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
