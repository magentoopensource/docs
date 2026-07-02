---
title: "Full Page Cache Strategy for High-Performance Magento"
description: "Master Magento 2 FPC architecture: cacheable blocks, ESI holes, private content, Varnish VCL, cache warming, and performance optimization techniques"
type: "how-to"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "60 minutes"
topics:
  - performance
  - cache
  - varnish
  - fpc
  - optimization
  - esi
  - customer-sections
last_updated: "2026-02-07"
---

# Full Page Cache Strategy for High-Performance Magento

## Problem Statement

A poorly optimized Magento store can serve pages in 2-5 seconds under load, leading to abandoned carts, low conversion rates, and poor Core Web Vitals scores. The culprit is often **inadequate Full Page Cache (FPC) strategy**:

- **Mixed cacheable/private content** in blocks, causing entire pages to bypass cache
- **Missing or incorrect cache tags**, preventing automatic invalidation
- **ESI holes not used**, forcing dynamic content to bust FPC
- **Varnish misconfiguration**, resulting in low hit rates or stale content
- **Cold cache after deployments**, causing post-release slowdowns

The Full Page Cache is Magento's **most powerful performance lever**. A well-configured FPC can deliver cached pages in **50-150ms** (vs 2000ms+ for uncached), achieve **90%+ hit rates**, and scale to **10,000+ requests/minute** on modest hardware.

This guide provides a **complete FPC architecture and implementation strategy**:

1. **Cacheable vs Non-Cacheable Blocks**: Design patterns for mixing public and private content
2. **Cache Tags and Invalidation**: Automatic cache purging on product/category/CMS updates
3. **ESI (Edge Side Includes) Holes**: Serve static content from cache while embedding dynamic fragments
4. **Private Content (Customer Sections)**: AJAX-based customer data loading post-page-load
5. **Varnish VCL Configuration**: Magento-optimized VCL for maximum hit rates
6. **Cache Warming Strategies**: Pre-populate cache after deployments or invalidations
7. **Debugging and Monitoring**: Tools to diagnose cache misses and measure performance
8. **Performance Benchmarks**: Real-world metrics and optimization targets

By the end, you'll architect modules that **respect FPC**, achieve **sub-200ms cached page loads**, and maintain **high hit rates under production traffic**.

---

## Prerequisites

- Magento 2.4.7+ (Adobe Commerce or Open Source)
- PHP 8.2+ with OPcache and Redis
- Redis for page cache and session storage
- Varnish 7.x (recommended) or built-in FPC
- Access to production-like load testing environment
- Familiarity with Magento block lifecycle and layout XML

**Tools:**
- `varnishstat`, `varnishlog`, `varnishncsa` (Varnish monitoring)
- `redis-cli` (cache inspection)
- Apache Bench (`ab`) or k6 for load testing
- New Relic, Blackfire, or Tideways for APM
- Browser DevTools (Network tab, Lighthouse)

**Assumed Setup:**
- Redis as default page cache backend (`app/etc/env.php`)
- Varnish as HTTP cache in front of Nginx/Apache
- CDN (Fastly, Cloudflare) in front of Varnish (optional but recommended)

---

## Step-by-Step Solution

### 1. Cacheable vs Non-Cacheable Blocks: The Foundation

**The Principle:**
A page is **cacheable** only if **all blocks** on that page are cacheable. A single non-cacheable block (e.g., customer name, cart count) makes the entire page non-cacheable, forcing a full PHP execution for every request.

**Block Cache Properties:**

```php
<?php
namespace MyVendor\MyModule\Block;

use Magento\Framework\View\Element\Template;

class ProductList extends Template
{
    protected $_isScopePrivate = false; // PUBLIC block (default)

    protected function _construct()
    {
        parent::_construct();

        // Cache configuration
        $this->addData([
            'cache_lifetime' => 3600, // 1 hour (seconds)
            'cache_tags' => [
                \Magento\Catalog\Model\Product::CACHE_TAG,
                \Magento\Catalog\Model\Category::CACHE_TAG,
            ],
            'cache_key' => $this->getCacheKey(),
        ]);
    }

    public function getCacheKey()
    {
        // Unique cache key per page variant
        return implode('_', [
            'product_list',
            $this->_storeManager->getStore()->getId(),
            $this->_design->getDesignTheme()->getId(),
            $this->getRequest()->getParam('p', 1), // Page number
            $this->getRequest()->getParam('category_id'),
        ]);
    }

    public function getCacheKeyInfo()
    {
        return [
            'PRODUCT_LIST',
            $this->_storeManager->getStore()->getId(),
            $this->_design->getDesignTheme()->getId(),
            $this->getRequest()->getParam('p', 1),
            $this->getRequest()->getParam('category_id'),
        ];
    }
}
```

**Non-Cacheable Block (Anti-Pattern):**

```php
<?php
namespace MyVendor\MyModule\Block;

use Magento\Framework\View\Element\Template;

class CustomerWelcome extends Template
{
    // This makes the ENTIRE PAGE non-cacheable!
    protected $_isScopePrivate = true;

    public function getCustomerName(): string
    {
        $customer = $this->customerSession->getCustomer();
        return $customer->getName();
    }
}
```

**Impact:** If `CustomerWelcome` block is added to the homepage layout, **all homepage requests** bypass FPC, even for guest users.

**The Fix: Private Content Sections (see Section 4)**

---

### 2. Cache Tags and Invalidation: Smart Purging

**The Problem:**
After updating a product, the product page must be purged from cache. Manually purging by URL is fragile (what about category pages showing that product? Search results?). Magento's **cache tag system** automates this.

**How It Works:**
1. Blocks declare **cache tags** (e.g., `catalog_product_123`, `catalog_category_45`)
2. When a product is saved, Magento invalidates all cache entries with that tag
3. Next request regenerates and caches the page with fresh data

**Declaring Cache Tags in Blocks:**

```php
<?php
namespace MyVendor\MyModule\Block;

use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Block\Product\AbstractProduct;

class FeaturedProduct extends AbstractProduct
{
    protected function _construct()
    {
        parent::_construct();
        $this->addData([
            'cache_lifetime' => 86400, // 24 hours
            'cache_tags' => $this->getCacheTags(),
        ]);
    }

    protected function getCacheTags(): array
    {
        $tags = [];

        $product = $this->getProduct();
        if ($product instanceof ProductInterface) {
            $tags[] = \Magento\Catalog\Model\Product::CACHE_TAG . '_' . $product->getId();

            // Invalidate when any category containing this product changes
            foreach ($product->getCategoryIds() as $categoryId) {
                $tags[] = \Magento\Catalog\Model\Category::CACHE_TAG . '_' . $categoryId;
            }
        }

        return $tags;
    }

    public function getIdentities()
    {
        // Required for FPC: return cache tags for page-level invalidation
        return $this->getCacheTags();
    }
}
```

**Adding Cache Tags to Custom Entities:**

```php
<?php
namespace MyVendor\MyModule\Model;

use Magento\Framework\DataObject\IdentityInterface;
use Magento\Framework\Model\AbstractModel;

class CustomEntity extends AbstractModel implements IdentityInterface
{
    public const CACHE_TAG = 'myvendor_mymodule_entity';

    protected $_cacheTag = self::CACHE_TAG;

    protected function _construct()
    {
        $this->_init(\MyVendor\MyModule\Model\ResourceModel\CustomEntity::class);
    }

    public function getIdentities()
    {
        return [
            self::CACHE_TAG . '_' . $this->getId(),
            self::CACHE_TAG, // Invalidate all entity pages
        ];
    }
}
```

**Observer: Invalidate Cache on Entity Save**

```php
<?php
namespace MyVendor\MyModule\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\PageCache\Model\Cache\Type as FullPageCache;

class InvalidateEntityCache implements ObserverInterface
{
    public function __construct(
        private FullPageCache $fullPageCache
    ) {}

    public function execute(Observer $observer)
    {
        $entity = $observer->getEvent()->getEntity();

        if ($entity instanceof \Magento\Framework\DataObject\IdentityInterface) {
            $this->fullPageCache->clean(
                \Zend_Cache::CLEANING_MODE_MATCHING_TAG,
                $entity->getIdentities()
            );
        }
    }
}
```

```xml
<!-- etc/frontend/events.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="myvendor_mymodule_entity_save_after">
        <observer name="invalidate_entity_cache"
                  instance="MyVendor\MyModule\Observer\InvalidateEntityCache"/>
    </event>
</config>
```

**Testing Cache Invalidation:**

```bash
# Enable cache
bin/magento cache:enable full_page

# Load product page (cache miss, generates cache)
curl -I https://magento.local/product-page.html
# X-Magento-Cache-Debug: MISS

# Load again (cache hit)
curl -I https://magento.local/product-page.html
# X-Magento-Cache-Debug: HIT

# Update product in admin panel

# Load again (cache miss after invalidation)
curl -I https://magento.local/product-page.html
# X-Magento-Cache-Debug: MISS
```

**Checklist:**
- [ ] All cacheable blocks declare `cache_lifetime` and `cache_tags`
- [ ] Custom entities implement `IdentityInterface` and return tags from `getIdentities()`
- [ ] Observers invalidate cache on entity save/delete
- [ ] Cache tags include entity ID and related entities (e.g., product → categories)

---

### 3. ESI (Edge Side Includes) Holes: Cache + Dynamic Content

**The Problem:**
You want to cache the product page but show the **user's cart item count** (dynamic). Traditional solution: make the entire page non-cacheable. Better solution: **ESI**.

**ESI Concept:**
Varnish (or other reverse proxy) caches the main page but leaves "holes" (ESI tags) that are filled with dynamic content on each request.

```html
<!-- Cached page with ESI hole -->
<html>
  <body>
    <h1>Product Page</h1> <!-- Cached -->

    <!-- ESI hole: fetched on every request -->
    <esi:include src="/customer/section/load?sections=cart" />

    <div>Product details...</div> <!-- Cached -->
  </body>
</html>
```

**Magento Implementation: Cacheable Block with ESI Hole**

```xml
<!-- view/frontend/layout/catalog_product_view.xml -->
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceContainer name="header.panel">
            <!-- ESI-enabled block: cached page, dynamic content via ESI -->
            <block class="Magento\Framework\View\Element\Template"
                   name="header.links"
                   template="Magento_Theme::header/links.phtml"
                   cacheable="true">
                <arguments>
                    <argument name="jsLayout" xsi:type="array">
                        <item name="components" xsi:type="array">
                            <item name="minicart" xsi:type="array">
                                <item name="config" xsi:type="array">
                                    <item name="esi" xsi:type="boolean">true</item>
                                </item>
                            </item>
                        </item>
                    </argument>
                </arguments>
            </block>
        </referenceContainer>
    </body>
</page>
```

**Varnish VCL for ESI:**

```vcl
# /etc/varnish/default.vcl (Magento 2.4.7 compatible)

sub vcl_recv {
    # Enable ESI processing
    if (req.url ~ "^/customer/section/load") {
        return (pass); # Never cache customer sections
    }

    # Strip cookies for static assets
    if (req.url ~ "\.(css|js|jpg|png|gif|ico|woff2)$") {
        unset req.http.Cookie;
    }
}

sub vcl_backend_response {
    # Enable ESI for pages with ESI tags
    if (beresp.http.X-Magento-Tags) {
        set beresp.do_esi = true;
    }

    # Cache static assets for 1 year
    if (bereq.url ~ "\.(css|js|jpg|png|gif|ico|woff2)$") {
        set beresp.ttl = 365d;
        unset beresp.http.Set-Cookie;
    }

    # Cache product pages for 1 hour
    if (bereq.url ~ "^/(.*)\\.html$") {
        set beresp.ttl = 1h;
    }
}

sub vcl_deliver {
    # Add cache debug header
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
    } else {
        set resp.http.X-Cache = "MISS";
    }

    # Remove internal headers
    unset resp.http.X-Magento-Tags;
    unset resp.http.X-Powered-By;
}
```

**Testing ESI:**

```bash
# Restart Varnish with ESI enabled
sudo systemctl restart varnish

# Load page and check for ESI processing
curl -I https://magento.local/product-page.html
# X-Cache: MISS (first load)

curl -I https://magento.local/product-page.html
# X-Cache: HIT (ESI-processed cached page)

# Verify ESI hole fetched separately
varnishlog -q 'ReqURL ~ "customer/section/load"'
```

**Checklist:**
- [ ] Varnish VCL has `set beresp.do_esi = true` for Magento pages
- [ ] Blocks with private content use `cacheable="true"` + customer sections (not ESI directly)
- [ ] ESI-included URLs return `pass` in Varnish (never cached)
- [ ] Test with `curl` and `varnishlog` to verify ESI fetch

**Important:** Magento 2.4+ **does not use ESI tags directly** for customer data. Instead, it uses **Customer Sections (Section 4)**. ESI is available for custom use cases (e.g., third-party content, A/B testing).

---

### 4. Private Content: Customer Sections (The Magento Way)

**The Problem:**
Customer-specific data (name, cart count, wishlist) cannot be cached. ESI could work, but Magento uses a **JavaScript-based approach**: **Customer Sections**.

**How It Works:**
1. Page loads from cache (fast, <150ms)
2. Page includes `sections.xml` configuration
3. JavaScript fetches customer data via AJAX (`/customer/section/load`)
4. Data injected into page via Knockout.js

**Result:** Cached page + dynamic content with no ESI complexity.

**Defining a Customer Section:**

```xml
<!-- etc/frontend/sections.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Customer:etc/sections.xsd">

    <!-- Define custom section -->
    <action name="mymodule/entity/save">
        <section name="cart"/> <!-- Reload cart after entity save -->
        <section name="mymodule-custom"/> <!-- Reload custom section -->
    </action>

    <action name="checkout/cart/add">
        <section name="cart"/>
        <section name="mymodule-custom"/> <!-- Invalidate custom section on cart update -->
    </action>
</config>
```

**Custom Section Data Provider:**

```php
<?php
namespace MyVendor\MyModule\CustomerData;

use Magento\Customer\CustomerData\SectionSourceInterface;

class CustomSection implements SectionSourceInterface
{
    public function __construct(
        private \Magento\Customer\Model\Session $customerSession,
        private \MyVendor\MyModule\Model\CustomRepository $customRepository
    ) {}

    /**
     * @return array
     */
    public function getSectionData()
    {
        if (!$this->customerSession->isLoggedIn()) {
            return [];
        }

        $customerId = $this->customerSession->getCustomerId();
        $data = $this->customRepository->getByCustomerId($customerId);

        return [
            'items' => $data->getItems(),
            'count' => $data->getCount(),
            'last_updated' => time(),
        ];
    }
}
```

```xml
<!-- etc/frontend/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Customer\CustomerData\SectionPoolInterface">
        <arguments>
            <argument name="sectionSourceMap" xsi:type="array">
                <item name="mymodule-custom" xsi:type="string">MyVendor\MyModule\CustomerData\CustomSection</item>
            </argument>
        </arguments>
    </type>
</config>
```

**Consuming Section Data in JavaScript:**

```javascript
// view/frontend/web/js/custom-widget.js
define([
    'uiComponent',
    'Magento_Customer/js/customer-data'
], function (Component, customerData) {
    'use strict';

    return Component.extend({
        initialize: function () {
            this._super();
            this.customSection = customerData.get('mymodule-custom');

            // Subscribe to changes
            this.customSection.subscribe(function (data) {
                console.log('Custom section updated:', data);
            });
        },

        getItemCount: function () {
            return this.customSection().count || 0;
        }
    });
});
```

```html
<!-- view/frontend/templates/custom-widget.phtml -->
<div data-bind="scope: 'custom-widget'">
    <span>Items: <span data-bind="text: getItemCount()"></span></span>
</div>

<script type="text/x-magento-init">
{
    "[data-bind*='custom-widget']": {
        "Magento_Ui/js/core/app": {
            "components": {
                "custom-widget": {
                    "component": "MyVendor_MyModule/js/custom-widget"
                }
            }
        }
    }
}
</script>
```

**Invalidating Sections Programmatically:**

```php
<?php
namespace MyVendor\MyModule\Controller\Entity;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\Controller\ResultFactory;

class Save implements HttpPostActionInterface
{
    public function __construct(
        private ResultFactory $resultFactory,
        private \Magento\Customer\Model\Session $customerSession
    ) {}

    public function execute()
    {
        // ... save entity

        // Invalidate customer sections
        $this->customerSession->regenerateId(); // Forces section reload

        return $this->resultFactory->create(ResultFactory::TYPE_REDIRECT)
            ->setPath('*/*/success');
    }
}
```

**Checklist:**
- [ ] Customer-specific data moved to customer sections (not blocks)
- [ ] `etc/frontend/sections.xml` defines section invalidation rules
- [ ] Section data providers implement `SectionSourceInterface`
- [ ] JavaScript uses `customerData.get('section-name')` to access data
- [ ] Test: Load page, verify AJAX request to `/customer/section/load`

---

### 5. Varnish VCL: Magento-Optimized Configuration

**Generate Base VCL:**

```bash
# Export Magento-provided VCL (Magento 2.4.7)
bin/magento varnish:vcl:generate --export-version=7 > /etc/varnish/magento.vcl

# Review and customize
sudo nano /etc/varnish/magento.vcl
```

**Production-Ready VCL (Highlights):**

```vcl
vcl 4.1;

import std;

backend default {
    .host = "127.0.0.1";
    .port = "8080"; # Nginx backend
    .first_byte_timeout = 600s;
    .probe = {
        .url = "/health_check.php";
        .timeout = 2s;
        .interval = 5s;
        .window = 10;
        .threshold = 5;
    }
}

acl purge {
    "localhost";
    "127.0.0.1";
    "192.168.1.0"/24; # Internal network
}

sub vcl_recv {
    # Allow cache purge from trusted IPs
    if (req.method == "PURGE") {
        if (!client.ip ~ purge) {
            return (synth(403, "Forbidden"));
        }
        return (purge);
    }

    # Only cache GET and HEAD
    if (req.method != "GET" && req.method != "HEAD") {
        return (pass);
    }

    # Don't cache admin, checkout, customer account
    if (req.url ~ "^/(admin|checkout|customer)") {
        return (pass);
    }

    # Strip query strings from static assets
    if (req.url ~ "\.(css|js|jpg|jpeg|png|gif|ico|svg|woff2)(\?.*)?$") {
        set req.url = regsub(req.url, "\?.*$", "");
        unset req.http.Cookie;
        return (hash);
    }

    # Normalize Accept-Encoding
    if (req.http.Accept-Encoding) {
        if (req.http.Accept-Encoding ~ "gzip") {
            set req.http.Accept-Encoding = "gzip";
        } elsif (req.http.Accept-Encoding ~ "deflate") {
            set req.http.Accept-Encoding = "deflate";
        } else {
            unset req.http.Accept-Encoding;
        }
    }

    # Remove Google Analytics cookies
    set req.http.Cookie = regsuball(req.http.Cookie, "(^|;\s*)(_ga|_gid|_gat)=[^;]*", "");
    set req.http.Cookie = regsuball(req.http.Cookie, "^;\s*", "");

    # If no cookies remain, unset Cookie header
    if (req.http.Cookie == "") {
        unset req.http.Cookie;
    }

    return (hash);
}

sub vcl_hash {
    # Vary cache by URL and host
    hash_data(req.url);
    if (req.http.host) {
        hash_data(req.http.host);
    } else {
        hash_data(server.ip);
    }

    # Vary by currency cookie (Magento multi-currency)
    if (req.http.Cookie ~ "currency=") {
        hash_data(regsub(req.http.Cookie, "^.*?currency=([^;]+);*.*$", "\1"));
    }

    # Vary by store code cookie
    if (req.http.Cookie ~ "store=") {
        hash_data(regsub(req.http.Cookie, "^.*?store=([^;]+);*.*$", "\1"));
    }

    return (lookup);
}

sub vcl_backend_response {
    # Enable grace mode (serve stale content if backend down)
    set beresp.grace = 24h;

    # Cache static assets for 1 year
    if (bereq.url ~ "\.(css|js|jpg|jpeg|png|gif|ico|svg|woff2)$") {
        set beresp.ttl = 365d;
        unset beresp.http.Set-Cookie;
        return (deliver);
    }

    # Respect Cache-Control from Magento
    if (beresp.http.Cache-Control ~ "no-cache|no-store|private") {
        set beresp.ttl = 0s;
        set beresp.uncacheable = true;
        return (deliver);
    }

    # Default TTL for cacheable pages
    if (beresp.ttl <= 0s) {
        set beresp.ttl = 1h;
    }

    return (deliver);
}

sub vcl_deliver {
    # Add cache status header (dev only)
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
        set resp.http.X-Cache-Hits = obj.hits;
    } else {
        set resp.http.X-Cache = "MISS";
    }

    # Remove internal headers in production
    # unset resp.http.X-Cache;
    # unset resp.http.X-Magento-Tags;
    # unset resp.http.X-Powered-By;

    return (deliver);
}
```

**Reload Varnish:**

```bash
sudo varnishreload # Or: sudo systemctl reload varnish
```

**Testing Varnish:**

```bash
# Check hit/miss
curl -I https://magento.local/

# Purge specific URL
curl -X PURGE https://magento.local/product-page.html

# Purge by cache tag (requires custom VCL)
curl -X PURGE https://magento.local/ -H "X-Magento-Tags-Pattern: catalog_product_123"

# Monitor Varnish stats
varnishstat -1 | grep cache_hit
# cache_hit: 45678 (hit rate: 92%)
```

**Checklist:**
- [ ] Varnish VCL generated from Magento (`bin/magento varnish:vcl:generate`)
- [ ] Backend health check configured (`/health_check.php` or `/pub/health_check.php`)
- [ ] PURGE ACL restricted to internal IPs
- [ ] Static assets cached for 1 year, stripped of cookies
- [ ] Grace mode enabled (serve stale on backend failure)
- [ ] Hit rate >85% in production (monitor with `varnishstat`)

---

### 6. Cache Warming: Prevent Cold Cache Slowdowns

**The Problem:**
After deployment or cache flush, the first request to each page is slow (cache miss). If 1000 pages exist, the first 1000 visitors experience degraded performance.

**The Solution: Cache Warming**

**Strategy A: Sitemap-Based Warming**

```bash
#!/bin/bash
# warm-cache.sh

SITEMAP_URL="https://magento.local/sitemap.xml"
CONCURRENCY=10

# Download sitemap
curl -s $SITEMAP_URL | \
  grep -oP '(?<=<loc>).*?(?=</loc>)' | \
  xargs -n 1 -P $CONCURRENCY curl -s -o /dev/null -w "%{url_effective} %{http_code} %{time_total}s\n"

echo "Cache warming complete."
```

**Run after deployment:**

```bash
# Flush cache
bin/magento cache:flush

# Reindex
bin/magento indexer:reindex

# Warm cache
bash warm-cache.sh
```

**Strategy B: Priority URL Warming (Critical Pages First)**

```php
<?php
// Script: bin/warm-cache.php

use Magento\Framework\App\Bootstrap;

require __DIR__ . '/../app/bootstrap.php';

$bootstrap = Bootstrap::create(BP, $_SERVER);
$objectManager = $bootstrap->getObjectManager();

$storeManager = $objectManager->get(\Magento\Store\Model\StoreManagerInterface::class);
$urlFinder = $objectManager->get(\Magento\Catalog\Model\ResourceModel\Product\CollectionFactory::class);

// Get top 100 products by sales
$productCollection = $urlFinder->create()
    ->addAttributeToSelect('url_key')
    ->addAttributeToSort('ordered_qty', 'DESC')
    ->setPageSize(100);

$urls = [];
foreach ($productCollection as $product) {
    $urls[] = $product->getProductUrl();
}

// Warm cache (multi-threaded via curl_multi)
$multiHandle = curl_multi_init();
$curlHandles = [];

foreach ($urls as $url) {
    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HEADER, true);
    curl_setopt($ch, CURLOPT_NOBODY, true); // HEAD request
    curl_multi_add_handle($multiHandle, $ch);
    $curlHandles[] = $ch;
}

$running = null;
do {
    curl_multi_exec($multiHandle, $running);
    curl_multi_select($multiHandle);
} while ($running > 0);

foreach ($curlHandles as $ch) {
    curl_multi_remove_handle($multiHandle, $ch);
    curl_close($ch);
}

curl_multi_close($multiHandle);

echo "Cache warmed for " . count($urls) . " URLs.\n";
```

**Run via cron (post-deployment):**

```bash
php bin/warm-cache.php
```

**Strategy C: Continuous Cache Warming (Proactive)**

```xml
<!-- etc/crontab.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/crontab.xsd">
    <group id="default">
        <job name="myvendor_cache_warm" instance="MyVendor\MyModule\Cron\WarmCache" method="execute">
            <schedule>*/30 * * * *</schedule> <!-- Every 30 minutes -->
        </job>
    </group>
</config>
```

```php
<?php
namespace MyVendor\MyModule\Cron;

class WarmCache
{
    public function __construct(
        private \MyVendor\MyModule\Model\CacheWarmer $cacheWarmer
    ) {}

    public function execute()
    {
        $this->cacheWarmer->warmTopPages();
        return $this;
    }
}
```

**Checklist:**
- [ ] Cache warming script targets top 100-500 URLs by traffic
- [ ] Script runs post-deployment and post-cache-flush
- [ ] Concurrency set to 10-20 (balance speed vs backend load)
- [ ] Monitor backend CPU during warming; adjust concurrency if needed
- [ ] Optional: Continuous warming via cron for long-tail pages

---

### 7. Debugging Cache Issues

**Enable Cache Debug Headers:**

```bash
bin/magento config:set system/full_page_cache/debug 1
bin/magento cache:flush
```

**Headers to Check:**

| Header | Value | Meaning |
|--------|-------|---------|
| `X-Magento-Cache-Debug` | `HIT` | Page served from FPC |
| `X-Magento-Cache-Debug` | `MISS` | Page generated, now cached |
| `X-Magento-Cache-Debug` | `BYPASS` | Page not cacheable |
| `X-Magento-Tags` | `cat_p_123,store_1` | Cache tags for this page |
| `X-Cache` | `HIT` | Varnish cache hit |
| `X-Cache` | `MISS` | Varnish cache miss |

**Common Issues:**

**Issue: All Requests Show BYPASS**

**Cause:** Non-cacheable block in layout.

**Debug:**
```bash
# Enable layout debugging
bin/magento dev:template-hints:enable
bin/magento dev:template-hints:status

# View page source, identify non-cacheable block (highlighted in red)
# Check block class for `$_isScopePrivate = true`
```

**Issue: Low Hit Rate (<50%)**

**Cause:** Cache keys vary by unnecessary parameters (e.g., tracking UTM params).

**Fix:** Normalize URLs in Varnish VCL:
```vcl
sub vcl_recv {
    # Strip tracking parameters
    set req.url = regsuball(req.url, "[?&](utm_[^&]+|gclid=[^&]+)", "");
}
```

**Issue: Cache Not Invalidated After Product Update**

**Cause:** Missing cache tags or observer not firing.

**Debug:**
```bash
# Note: events:list is not a core Magento Open Source command (it is part of Adobe I/O Events).
# To check if an observer is registered, search the codebase directly:
grep -r "catalog_product_save_after" app/code/ vendor/ --include="events.xml"

# Check block cache tags
grep -r "getIdentities\|cache_tags" app/code/MyVendor/MyModule/Block/
```

**Issue: Customer Sections Not Loading**

**Cause:** JavaScript error or missing section registration.

**Debug:**
1. Open browser DevTools → Console
2. Check for JS errors
3. Verify AJAX request to `/customer/section/load?sections=cart,mymodule-custom`
4. Verify `etc/frontend/di.xml` registers section data provider

**Tools:**

```bash
# Monitor Redis cache keys
redis-cli --scan --pattern "zc:*" | head -20

# Monitor Varnish logs for specific URL
varnishlog -q "ReqURL ~ '/product-page.html'"

# Varnish hit rate
varnishstat -1 | grep cache_hit

# Magento cache status
bin/magento cache:status
```

**Checklist:**
- [ ] Cache debug headers enabled in dev/staging
- [ ] All cacheable pages return `X-Magento-Cache-Debug: HIT` on second request
- [ ] Varnish hit rate >85% in production
- [ ] Cache invalidates within 60s of product/category update
- [ ] Customer sections load via AJAX (check Network tab)

---

## Performance Benchmarks

### Baseline (No Cache)

**Test:** Apache Bench, 1000 requests, concurrency 10, no cache.

```bash
ab -n 1000 -c 10 https://magento.local/product-page.html

# Results (no cache):
# Requests per second: 5.2 req/s
# Time per request: 1923 ms (mean)
# 95th percentile: 2800 ms
```

### With FPC (Redis)

```bash
# Enable cache, warm first
bin/magento cache:enable full_page
curl https://magento.local/product-page.html > /dev/null

ab -n 1000 -c 10 https://magento.local/product-page.html

# Results (Redis FPC):
# Requests per second: 125 req/s
# Time per request: 80 ms (mean)
# 95th percentile: 120 ms
```

**24x improvement** in throughput, **95% reduction** in latency.

### With Varnish + FPC

```bash
ab -n 1000 -c 10 https://magento.local/product-page.html

# Results (Varnish + Redis):
# Requests per second: 850 req/s
# Time per request: 12 ms (mean)
# 95th percentile: 20 ms
```

**163x improvement** vs no cache, **7x improvement** vs Redis alone.

### Real-World Targets

| Metric | Target | Impact |
|--------|--------|--------|
| **Cache Hit Rate** | >90% | Higher = fewer backend requests |
| **TTFB (Time to First Byte)** | <200ms | Faster perceived load |
| **LCP (Largest Contentful Paint)** | <2.5s | Core Web Vitals "Good" |
| **FCP (First Contentful Paint)** | <1.8s | User sees content fast |
| **Cache Size** | <10GB | Balance memory vs hit rate |
| **Cache Lifetime** | 1-24 hours | Balance freshness vs performance |

**Monitoring:**

```bash
# New Relic: Track cache hit rate, TTFB, page load time
# Google Lighthouse: Run on key pages, aim for Performance Score >90
# Varnish: varnishstat, varnishhist
# Magento: New Relic PHP extension + FPC monitoring
```

---

## Security and Performance Notes

**Security:**
- **Varnish PURGE ACL:** Restrict to internal IPs; public access enables DoS
- **Cache Poisoning:** Validate `X-Forwarded-Host` header; reject if untrusted
- **ESI Injection:** Never use user input in ESI src URLs
- **Private Data Leakage:** Ensure customer-specific data ONLY in customer sections, not cacheable blocks

**Performance:**
- **Redis vs Varnish:** Redis stores serialized PHP objects (slower deserialization); Varnish stores HTTP responses (faster, no PHP deserialization)
- **Grace Mode:** Serve stale content if backend down; better UX than downtime
- **Cache Size:** Monitor Redis/Varnish memory; if full, eviction reduces hit rate
- **CDN:** Place Cloudflare/Fastly in front of Varnish for global distribution and DDoS protection

---

## Backward Compatibility

- **Customer Sections:** Introduced in 2.0.0; consistent API in 2.4.x
- **Varnish VCL:** Magento 2.4.7 officially supports Varnish 7.x; VCL 4.1 syntax
- **ESI:** Supported since 2.0.0; not used by core for customer data (sections preferred)
- **Cache Tags:** API stable since 2.0.0; `IdentityInterface` unchanged

**Upgrade Path:** All techniques compatible with 2.4.0 → 2.4.8. Varnish VCL may need minor adjustments between major Varnish versions (6.x → 7.x).

---

## Tests to Add

### Unit Test: Cache Key Uniqueness

```php
<?php
namespace MyVendor\MyModule\Test\Unit\Block;

use MyVendor\MyModule\Block\ProductList;
use PHPUnit\Framework\TestCase;

class ProductListTest extends TestCase
{
    public function testCacheKeyUnique()
    {
        $block1 = $this->createBlock(['category_id' => 5, 'p' => 1]);
        $block2 = $this->createBlock(['category_id' => 5, 'p' => 2]);

        $this->assertNotEquals($block1->getCacheKey(), $block2->getCacheKey());
    }

    private function createBlock(array $params): ProductList
    {
        // Mock request, store, theme
        // ...
        return $block;
    }
}
```

### Integration Test: Cache Invalidation

```php
<?php
namespace MyVendor\MyModule\Test\Integration\Observer;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

class InvalidateEntityCacheTest extends TestCase
{
    public function testCacheInvalidatedOnSave()
    {
        $objectManager = Bootstrap::getObjectManager();
        $cache = $objectManager->get(\Magento\PageCache\Model\Cache\Type::class);
        $repository = $objectManager->get(\MyVendor\MyModule\Api\EntityRepositoryInterface::class);

        $entity = $repository->getById(1);

        // Simulate page cache
        $cacheId = 'entity_page_1';
        $cache->save('cached_data', $cacheId, $entity->getIdentities());

        // Update entity
        $entity->setName('Updated');
        $repository->save($entity);

        // Verify cache invalidated
        $cachedData = $cache->load($cacheId);
        $this->assertFalse($cachedData);
    }
}
```

### Load Test: Cache Hit Rate

```bash
# k6 load test (k6.io)
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  stages: [
    { duration: '1m', target: 100 }, // Ramp to 100 users
    { duration: '5m', target: 100 }, // Stay at 100
    { duration: '1m', target: 0 },   // Ramp down
  ],
};

export default function () {
  let res = http.get('https://magento.local/product-page.html');

  check(res, {
    'status is 200': (r) => r.status === 200,
    'cache hit': (r) => r.headers['X-Cache'] === 'HIT',
    'response time < 200ms': (r) => r.timings.duration < 200,
  });
}

# Run: k6 run load-test.js
# Target: >90% cache hit rate, <200ms avg response time
```

---

## Documentation to Update

1. **README.md**
   - Add "Performance" section with FPC strategy overview
   - Link to this guide

2. **CHANGELOG.md**
   - Version 2.0.0: "Refactored blocks for FPC compatibility, added customer sections"

3. **Admin User Guide** (`docs/admin-guide.md`)
   - How to flush cache after content updates
   - How to enable/disable FPC in admin panel

4. **Developer Guide** (`docs/developer-guide.md`)
   - FPC architecture diagram
   - How to make custom blocks cacheable
   - How to add customer sections

5. **DevOps Runbook** (`docs/runbook.md`)
   - Varnish restart procedure
   - Cache warming post-deployment
   - Monitoring: Varnish hit rate, Redis memory, cache invalidation lag

---

## Assumptions

- **Magento Version:** 2.4.7+ (Adobe Commerce or Open Source)
- **PHP Version:** 8.2+ with OPcache, Redis PHP extension
- **Cache Backend:** Redis 6.x or 7.x
- **HTTP Cache:** Varnish 7.x (or built-in FPC if Varnish unavailable)
- **Infrastructure:** Dedicated Redis instance (not shared with sessions); Varnish with 2GB+ memory
- **Traffic:** Production site with >1000 requests/hour (FPC benefits scale with traffic)

---

## Why This Approach

**Trade-offs:**
- **Customer Sections vs ESI:** Sections require JavaScript but avoid Varnish complexity; ESI is more flexible but harder to debug
- **Cache Lifetime (1h-24h):** Longer = fewer backend requests but staler content; adjust per page type (product: 1h, CMS: 24h)
- **Varnish vs Built-in FPC:** Varnish is faster (no PHP deserialization) but adds operational overhead; built-in FPC sufficient for <10k req/hour

**Alternatives Considered:**
- **GraphQL + SPA:** Eliminates FPC entirely (client-side rendering); chosen for headless projects but adds frontend complexity
- **HTTP/2 Server Push:** Can replace customer sections with pushed resources; limited browser support, harder to debug
- **Static Site Generation (SSG):** Pre-render all pages at build time; works for content-heavy sites but not for dynamic catalogs

**Chosen Approach:** Hybrid FPC with customer sections balances **performance, maintainability, and compatibility** with existing Magento themes/extensions.

---

## Summary

A **comprehensive Full Page Cache strategy** is the difference between a sluggish Magento store and a lightning-fast customer experience. By architecting cacheable blocks, leveraging cache tags, using customer sections for private content, optimizing Varnish VCL, and implementing cache warming, you achieve:

- **Sub-200ms cached page loads** (vs 2000ms+ uncached)
- **90%+ cache hit rates** under production traffic
- **10x-100x throughput improvement** on existing hardware
- **Improved Core Web Vitals** (LCP <2.5s, FCP <1.8s)
- **Scalability to 10,000+ req/min** with proper caching

**Next Steps:**
1. Audit existing blocks for FPC compatibility (identify non-cacheable blocks)
2. Refactor private content to customer sections
3. Deploy Varnish with Magento-optimized VCL
4. Implement cache warming script and post-deployment workflow
5. Monitor hit rate, TTFB, and invalidation lag in production

**Performance is not a feature—it's a competitive advantage.** Build it into your architecture from day one.

## Related Documentation

### Related Guides

- [Indexer System Deep Dive: Understanding Magento 2's Data Indexing Architecture](../explanations/indexer-system.md)
- [Layout XML Deep Dive: Mastering Magento's View Layer](layout-xml-deep-dive.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Checkout Overview](../../modules/checkout/README.md)
