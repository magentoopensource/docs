---
title: "Advanced Configuration Options Reference"
description: "Complete reference of system configuration, caching, and performance settings"
type: reference
audience:
  - Magento merchants
  - New Magento users
  - Advanced users
tags:
  - reference
  - configuration
  - settings
  - performance
  - complex
last_updated: "2025-09-12T12:24:19.968Z"
---

# Advanced Configuration Options Reference

## Overview

This reference lists advanced Commerce configuration, caching, and performance settings to help merchants improve storefront speed, stability, and operating efficiency. Each entry includes where to find the setting, what it does, recommended values for production, and how to verify the impact.

Applies to: Adobe Commerce and Magento Open Source 2.4.x. Settings and paths may differ in older versions.

### Prerequisites

- Admin access with System and Stores permissions
- CLI access to the application server
- Working cron configured on the server
- For Varnish: a running Varnish service and network access
- For Redis: Redis server available if you plan to use it for cache and/or sessions

## Configuration

### Key Settings Reference

#### 1) Cache Management
- Path: Admin > System > Tools > Cache Management
- What it does: Enables/disables Magento cache types and lets you refresh caches.
- Recommended in production: Keep all cache types Enabled. Never disable Page Cache (Full Page Cache) in production.
- Actions:
  - Use "Flush Magento Cache" for application cache. Avoid "Flush Cache Storage" during business hours as it clears all storage and can spike origin load.
- Verify:
  - CLI: `bin/magento cache:status` — confirm all caches show 1 (enabled).

#### 2) Full Page Cache (FPC)
- Path: Admin > Stores > Settings > Configuration > Advanced > System > Full Page Cache
- Setting: Caching Application
  - Options: Built-in Cache | Varnish Cache
  - Recommendation: Use Varnish Cache in production for best TTFB and scalability. Use Built-in only for development or when Varnish is not available.
- Additional:
  - TTL for public content: Keep default unless you have a specific business need. Shorter TTL increases origin load; longer TTL improves hit rate.
  - Varnish configuration: After selecting Varnish Cache, use "Export VCL for Varnish" to generate the VCL for your Varnish version and deploy it to your Varnish server.
- Why choose Varnish:
  - Offloads page rendering from PHP, reduces TTFB by up to 30–50% on cache hits, and supports higher concurrent traffic on the same hardware.
  - ROI: Faster pages can lift conversion and reduce infrastructure costs by improving cache hit rates.
- Verify:
  - Check response headers for Varnish behavior (e.g., increasing `Age` header on repeat requests) and confirm improved TTFB.

#### 3) Index Management
- Path: Admin > System > Tools > Index Management
- What it does: Controls how indexers update derived data.
- Recommendation: Set all indexers to "Update by Schedule" in production to reduce admin save latency. Ensure server cron runs every minute.
- Business impact:
  - "Update by Schedule" keeps the storefront responsive and allows bulk admin updates without timeouts.
  - ROI: Reduces admin save times and errors during catalog updates; improves staff efficiency during peak merchandising cycles.
- Verify:
  - CLI: `bin/magento indexer:status` — indexers should show "Ready" when up to date.

#### 4) Application Mode
- Path: CLI only
- What it does: Controls runtime optimizations.
- Recommendation: Use Production mode in production environments for optimized performance and stability.
- Commands:
  - Show mode: `bin/magento deploy:mode:show`
  - Set production: `bin/magento deploy:mode:set production`
- Business impact:
  - Production mode compiles and serves optimized assets, decreasing page load times and reducing runtime errors.
  - ROI: Faster pages aid conversion and SEO; fewer errors lower support costs.
- Verify:
  - Re-run `bin/magento deploy:mode:show` — should read "production".

#### 5) Cron
- Path: Server crontab; optional settings at Admin > Stores > Configuration > Advanced > System > Cron
- Recommendation: Ensure the system cron invokes Magento every minute.
- Example crontab entry:
  - `* * * * * php /path/to/bin/magento cron:run >> var/log/magento.cron.log 2>&1`
- Verify:
  - Review `var/log/magento.cron.log` for recent activity and monitor indexers/jobs completing.

#### 6) Sessions and Cache Backends (Redis)
- Path: `app/etc/env.php` (server configuration)
- Recommendation: Use Redis for application cache, page cache, and sessions for better performance and resilience.
- Notes:
  - Configure Redis per Adobe documentation and schedule changes during a maintenance window.
- Verify:
  - `bin/magento cache:status` and application logs; confirm Redis connections are established and error-free.

### Change Management and Safety
- Schedule changes during low-traffic windows and notify stakeholders.
- Take a backup/snapshot before changes.
- Rollback:
  - Revert settings to prior values.
  - For Varnish, re-deploy the previous VCL.
  - For Redis, restore the previous `env.php` and clear generated caches.
- Post-change verification: Check site availability, TTFB, and error logs.

### Post-change verification checklist
- Measure homepage and key product page TTFB and LCP before/after changes.
- Confirm cache hit behavior (e.g., Varnish `Age` header increases between requests).
- Ensure indexers are "Ready" and cron logs show timely runs.
- Spot-check key customer journeys (browse, search, add to cart, checkout) for errors or slowdowns.

## See Also

- Caching
  - Cache Management: https://experienceleague.adobe.com/en/docs/commerce-admin/systems/tools/cache-management
  - Full Page Cache (Varnish): https://experienceleague.adobe.com/en/docs/commerce-operations/configuration-guide/cache/varnish
- Indexing
  - Index Management: https://experienceleague.adobe.com/en/docs/commerce-admin/systems/tools/index-management
- Runtime
  - Application modes: https://experienceleague.adobe.com/en/docs/commerce-operations/configuration-guide/cli/set-mode
  - Cron setup: https://experienceleague.adobe.com/en/docs/commerce-operations/configuration-guide/cli/configure-cron
- Storage
  - Redis configuration: https://experienceleague.adobe.com/en/docs/commerce-operations/configuration-guide/cache/redis
- Commerce Admin User Guide: https://experienceleague.adobe.com/en/docs/commerce-admin/user-guide/

---

*This reference was last updated on 2025-09-12T12:24:19.968Z. For the most current information, see the Adobe Commerce and Magento Open Source documentation at https://experienceleague.adobe.com/en/docs/commerce.*
