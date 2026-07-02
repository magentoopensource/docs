---
title: "Multi-Store Configuration Mastery"
description: "Deep dive into Magento's website/store/store view hierarchy, scope-aware configuration, multi-store themes, URL management, and performance optimization."
type: "how-to"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "50 minutes"
topics:
  - multi-store
  - scope configuration
  - url rewrites
  - shared catalogs
  - themes
  - performance
last_updated: "2026-02-07"
---

# Multi-Store Configuration Mastery

## Overview

Magento's multi-store architecture enables running multiple brands, locales, or customer segments from a single installation. Understanding the website/store/store view hierarchy and scope-aware configuration is essential for enterprise deployments.

**What you'll learn:**
- Website/Store/Store View hierarchy and scope inheritance
- Scope-aware configuration for modules and system settings
- Multi-store theme architecture and template fallback
- URL rewrite strategies for store-specific routes
- Shared vs separate catalog strategies
- Performance optimization for multi-store environments
- Database and Redis configuration for scale
- CLI commands for store management

**Prerequisites:**
- Magento 2.4.7+ (Adobe Commerce or Open Source)
- Advanced understanding of Magento configuration system
- Experience with themes and layout XML
- Knowledge of URL rewriting and routing
- Familiarity with caching strategies and indexers

---

## Store Hierarchy Architecture

### Three-Tier Hierarchy

```
Global
  ├── Website 1 (example.com)
  │   ├── Store Group 1 (Main Store)
  │   │   ├── Store View 1 (English)
  │   │   ├── Store View 2 (Spanish)
  │   │   └── Store View 3 (French)
  │   └── Store Group 2 (Wholesale)
  │       ├── Store View 4 (B2B English)
  │       └── Store View 5 (B2B German)
  └── Website 2 (brand2.com)
      └── Store Group 3 (Brand Store)
          ├── Store View 6 (English)
          └── Store View 7 (Spanish)
```

### Scope Levels

1. **Global**: Default configuration applied to all websites
2. **Website**: Shared customer accounts, order numbers, payment methods
3. **Store**: Shared root category, pricing, promotions
4. **Store View**: Locale-specific content, translations, CMS pages

### Scope Inheritance

Configuration values cascade from Global → Website → Store → Store View. Lower scopes override higher scopes.

```
Global: tax_calculation/algorithm = "TOTAL_BASE"
  └→ Website US: tax_calculation/algorithm = "TOTAL_BASE" (inherited)
      └→ Store California: tax_calculation/algorithm = "ROW_BASE" (overridden)
          └→ Store View EN-US: tax_calculation/algorithm = "ROW_BASE" (inherited)
```

---

## Creating Multi-Store Structure

### Admin Configuration

**Stores → All Stores → Create Website/Store/Store View**

```
Website: "Europe Region"
  Code: europe
  Sort Order: 10

Store Group: "European Main Store"
  Website: Europe Region
  Root Category: Default Category
  Default Store View: Europe EN

Store View: "Europe English"
  Store: European Main Store
  Code: europe_en
  Status: Enabled
  Sort Order: 10

Store View: "Europe German"
  Store: European Main Store
  Code: europe_de
  Status: Enabled
  Sort Order: 20
```

### Programmatic Store Creation

**File**: `Vendor/MultiStore/Setup/Patch/Data/CreateStoreStructure.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\MultiStore\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Store\Model\GroupFactory;
use Magento\Store\Model\ResourceModel\Group as GroupResource;
use Magento\Store\Model\ResourceModel\Store as StoreResource;
use Magento\Store\Model\ResourceModel\Website as WebsiteResource;
use Magento\Store\Model\StoreFactory;
use Magento\Store\Model\WebsiteFactory;

class CreateStoreStructure implements DataPatchInterface
{
    public function __construct(
        private readonly WebsiteFactory $websiteFactory,
        private readonly WebsiteResource $websiteResource,
        private readonly GroupFactory $groupFactory,
        private readonly GroupResource $groupResource,
        private readonly StoreFactory $storeFactory,
        private readonly StoreResource $storeResource
    ) {
    }

    public function apply(): self
    {
        // Create Website
        $website = $this->websiteFactory->create();
        $website->setCode('europe')
            ->setName('Europe Region')
            ->setSortOrder(10)
            ->setIsDefault(0);

        $this->websiteResource->save($website);

        // Create Store Group
        $group = $this->groupFactory->create();
        $group->setWebsiteId($website->getId())
            ->setName('European Main Store')
            ->setRootCategoryId(2) // Default category
            ->setDefaultStoreId(0);

        $this->groupResource->save($group);

        // Create Store Views
        $storeViews = [
            [
                'code' => 'europe_en',
                'name' => 'Europe English',
                'sort_order' => 10,
                'is_active' => 1,
            ],
            [
                'code' => 'europe_de',
                'name' => 'Europe German',
                'sort_order' => 20,
                'is_active' => 1,
            ],
        ];

        $firstStoreId = null;

        foreach ($storeViews as $storeData) {
            $store = $this->storeFactory->create();
            $store->setCode($storeData['code'])
                ->setName($storeData['name'])
                ->setWebsiteId($website->getId())
                ->setGroupId($group->getId())
                ->setSortOrder($storeData['sort_order'])
                ->setIsActive($storeData['is_active']);

            $this->storeResource->save($store);

            if ($firstStoreId === null) {
                $firstStoreId = $store->getId();
            }
        }

        // Set default store for group
        if ($firstStoreId) {
            $group->setDefaultStoreId($firstStoreId);
            $this->groupResource->save($group);
        }

        return $this;
    }

    public static function getDependencies(): array
    {
        return [];
    }

    public function getAliases(): array
    {
        return [];
    }
}
```

---

## Scope-Aware Configuration

### System Configuration with Scope

**File**: `Vendor/MultiStore/etc/adminhtml/system.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system_file.xsd">
    <system>
        <section id="multistore_config" translate="label" sortOrder="500" showInDefault="1" showInWebsite="1" showInStore="1">
            <label>Multi-Store Settings</label>
            <tab>general</tab>
            <resource>Vendor_MultiStore::config</resource>

            <group id="regional" translate="label" sortOrder="10" showInDefault="1" showInWebsite="1" showInStore="1">
                <label>Regional Settings</label>

                <!-- Global, Website, and Store scope -->
                <field id="currency_code" translate="label comment" type="select" sortOrder="10" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Default Currency</label>
                    <source_model>Magento\Config\Model\Config\Source\Locale\Currency</source_model>
                    <config_path>multistore_config/regional/currency_code</config_path>
                    <comment>Configured at website level</comment>
                </field>

                <!-- Store view specific -->
                <field id="locale" translate="label comment" type="select" sortOrder="20" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Locale</label>
                    <source_model>Magento\Config\Model\Config\Source\Locale</source_model>
                    <config_path>multistore_config/regional/locale</config_path>
                    <comment>Configured per store view</comment>
                </field>

                <!-- Website only -->
                <field id="tax_region" translate="label" type="text" sortOrder="30" showInDefault="0" showInWebsite="1" showInStore="0">
                    <label>Tax Region Code</label>
                    <config_path>multistore_config/regional/tax_region</config_path>
                    <comment>Only available at website scope</comment>
                </field>
            </group>
        </section>
    </system>
</config>
```

### Reading Scope-Aware Configuration

**File**: `Vendor/MultiStore/Model/Config.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\MultiStore\Model;

use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Store\Model\ScopeInterface;
use Magento\Store\Model\StoreManagerInterface;

class Config
{
    private const XML_PATH_CURRENCY = 'multistore_config/regional/currency_code';
    private const XML_PATH_LOCALE = 'multistore_config/regional/locale';
    private const XML_PATH_TAX_REGION = 'multistore_config/regional/tax_region';

    public function __construct(
        private readonly ScopeConfigInterface $scopeConfig,
        private readonly StoreManagerInterface $storeManager
    ) {
    }

    /**
     * Get currency for current website
     */
    public function getCurrency(?int $websiteId = null): string
    {
        if ($websiteId === null) {
            $websiteId = (int)$this->storeManager->getWebsite()->getId();
        }

        return (string)$this->scopeConfig->getValue(
            self::XML_PATH_CURRENCY,
            ScopeInterface::SCOPE_WEBSITE,
            $websiteId
        );
    }

    /**
     * Get locale for current store view
     */
    public function getLocale(?int $storeId = null): string
    {
        if ($storeId === null) {
            $storeId = (int)$this->storeManager->getStore()->getId();
        }

        return (string)$this->scopeConfig->getValue(
            self::XML_PATH_LOCALE,
            ScopeInterface::SCOPE_STORE,
            $storeId
        );
    }

    /**
     * Get tax region for website
     */
    public function getTaxRegion(?int $websiteId = null): ?string
    {
        if ($websiteId === null) {
            $websiteId = (int)$this->storeManager->getWebsite()->getId();
        }

        $value = $this->scopeConfig->getValue(
            self::XML_PATH_TAX_REGION,
            ScopeInterface::SCOPE_WEBSITE,
            $websiteId
        );

        return $value ? (string)$value : null;
    }

    /**
     * Check if value is overridden at store level
     */
    public function isStoreOverridden(string $path, int $storeId): bool
    {
        $storeValue = $this->scopeConfig->getValue(
            $path,
            ScopeInterface::SCOPE_STORE,
            $storeId
        );

        $websiteValue = $this->scopeConfig->getValue(
            $path,
            ScopeInterface::SCOPE_WEBSITE,
            $this->storeManager->getStore($storeId)->getWebsiteId()
        );

        return $storeValue !== $websiteValue;
    }
}
```

### Saving Configuration at Different Scopes

**File**: `Vendor/MultiStore/Model/ConfigWriter.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\MultiStore\Model;

use Magento\Framework\App\Config\Storage\WriterInterface;
use Magento\Framework\App\Config\ScopeConfigInterface;

class ConfigWriter
{
    public function __construct(
        private readonly WriterInterface $configWriter
    ) {
    }

    /**
     * Save config at global scope
     */
    public function saveGlobal(string $path, string $value): void
    {
        $this->configWriter->save(
            $path,
            $value,
            ScopeConfigInterface::SCOPE_TYPE_DEFAULT,
            0
        );
    }

    /**
     * Save config at website scope
     */
    public function saveWebsite(string $path, string $value, int $websiteId): void
    {
        $this->configWriter->save(
            $path,
            $value,
            \Magento\Store\Model\ScopeInterface::SCOPE_WEBSITES,
            $websiteId
        );
    }

    /**
     * Save config at store scope
     */
    public function saveStore(string $path, string $value, int $storeId): void
    {
        $this->configWriter->save(
            $path,
            $value,
            \Magento\Store\Model\ScopeInterface::SCOPE_STORES,
            $storeId
        );
    }

    /**
     * Delete config (revert to parent scope)
     */
    public function deleteStore(string $path, int $storeId): void
    {
        $this->configWriter->delete(
            $path,
            \Magento\Store\Model\ScopeInterface::SCOPE_STORES,
            $storeId
        );
    }
}
```

---

## Multi-Store Themes

### Theme Configuration Per Store View

**Stores → Configuration → Design → Theme**

```
Scope: Default Config
  Applied Theme: Magento/luma

Scope: Europe English
  Applied Theme: Vendor/europe-theme

Scope: Europe German
  Applied Theme: Vendor/europe-theme-de
```

### Theme Fallback Hierarchy

```
Vendor/europe-theme-de
  └→ Vendor/europe-theme
      └→ Magento/luma
          └→ Magento/blank
```

### Store-Specific Templates

**File**: `app/design/frontend/Vendor/europe-theme/Magento_Catalog/templates/product/view/addtocart.phtml`

```php
<?php
/**
 * Store-specific add to cart template
 * @var \Magento\Catalog\Block\Product\View $block
 */

$storeCode = $block->getStoreManager()->getStore()->getCode();
?>

<div class="product-add-form" data-store="<?= $escaper->escapeHtml($storeCode) ?>">
    <?php if ($storeCode === 'europe_de'): ?>
        <!-- German-specific content -->
        <button type="submit" class="action primary tocart">
            In den Warenkorb
        </button>
    <?php elseif ($storeCode === 'europe_en'): ?>
        <!-- English-specific content -->
        <button type="submit" class="action primary tocart">
            Add to Cart
        </button>
    <?php else: ?>
        <!-- Default -->
        <button type="submit" class="action primary tocart">
            <?= $escaper->escapeHtml(__('Add to Cart')) ?>
        </button>
    <?php endif; ?>
</div>
```

### Dynamic Theme Switching

**File**: `Vendor/MultiStore/Plugin/Theme/View/Design/ThemeExtend.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\MultiStore\Plugin\Theme\View\Design;

use Magento\Framework\View\DesignInterface;
use Magento\Store\Model\StoreManagerInterface;

class ThemeExtend
{
    private const STORE_THEME_MAP = [
        'europe_en' => 'Vendor/europe-theme',
        'europe_de' => 'Vendor/europe-theme-de',
        'us_store' => 'Vendor/us-theme',
    ];

    public function __construct(
        private readonly StoreManagerInterface $storeManager
    ) {
    }

    /**
     * Override theme based on store code
     */
    public function afterGetConfigurationDesignTheme(
        DesignInterface $subject,
        $result
    ) {
        $storeCode = $this->storeManager->getStore()->getCode();

        if (isset(self::STORE_THEME_MAP[$storeCode])) {
            return self::STORE_THEME_MAP[$storeCode];
        }

        return $result;
    }
}
```

---

## URL Management

### Base URL Configuration

**File**: `app/etc/config.php` or **Stores → Configuration → Web → Base URLs**

```php
<?php
return [
    'system' => [
        'default' => [
            'web' => [
                'unsecure' => [
                    'base_url' => 'https://default.example.com/',
                ],
                'secure' => [
                    'base_url' => 'https://default.example.com/',
                ]
            ]
        ],
        'websites' => [
            'europe' => [
                'web' => [
                    'unsecure' => [
                        'base_url' => 'https://europe.example.com/',
                    ],
                    'secure' => [
                        'base_url' => 'https://europe.example.com/',
                    ]
                ]
            ],
            'us' => [
                'web' => [
                    'unsecure' => [
                        'base_url' => 'https://us.example.com/',
                    ],
                    'secure' => [
                        'base_url' => 'https://us.example.com/',
                    ]
                ]
            ]
        ],
        'stores' => [
            'europe_en' => [
                'web' => [
                    'unsecure' => [
                        'base_url' => 'https://europe.example.com/en/',
                    ]
                ]
            ],
            'europe_de' => [
                'web' => [
                    'unsecure' => [
                        'base_url' => 'https://europe.example.com/de/',
                    ]
                ]
            ]
        ]
    ]
];
```

### Server Configuration (Nginx)

**File**: `/etc/nginx/sites-available/magento-multistore`

```nginx
upstream fastcgi_backend {
    server unix:/run/php/php8.2-fpm.sock;
}

# Europe Website
server {
    listen 80;
    server_name europe.example.com;

    set $MAGE_RUN_CODE europe;
    set $MAGE_RUN_TYPE website;

    root /var/www/magento/pub;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_pass fastcgi_backend;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param MAGE_RUN_CODE $MAGE_RUN_CODE;
        fastcgi_param MAGE_RUN_TYPE $MAGE_RUN_TYPE;
        include fastcgi_params;
    }
}

# US Website
server {
    listen 80;
    server_name us.example.com;

    set $MAGE_RUN_CODE us;
    set $MAGE_RUN_TYPE website;

    root /var/www/magento/pub;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_pass fastcgi_backend;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param MAGE_RUN_CODE $MAGE_RUN_CODE;
        fastcgi_param MAGE_RUN_TYPE $MAGE_RUN_TYPE;
        include fastcgi_params;
    }
}

# Store view by URL path
map $request_uri $store_code {
    ~^/en/ europe_en;
    ~^/de/ europe_de;
    default europe_en;
}
```

### Store-Specific URL Rewrites

**File**: `Vendor/MultiStore/Observer/CustomUrlRewrite.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\MultiStore\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\UrlRewrite\Model\UrlRewriteFactory;
use Magento\UrlRewrite\Model\ResourceModel\UrlRewrite as UrlRewriteResource;
use Magento\Store\Model\StoreManagerInterface;

class CustomUrlRewrite implements ObserverInterface
{
    public function __construct(
        private readonly UrlRewriteFactory $urlRewriteFactory,
        private readonly UrlRewriteResource $urlRewriteResource,
        private readonly StoreManagerInterface $storeManager
    ) {
    }

    /**
     * Create store-specific URL rewrite
     */
    public function execute(Observer $observer): void
    {
        $product = $observer->getEvent()->getProduct();

        foreach ($this->storeManager->getStores() as $store) {
            $urlRewrite = $this->urlRewriteFactory->create();

            $requestPath = $this->generateRequestPath($product, $store);

            $urlRewrite->setStoreId($store->getId())
                ->setEntityType('product')
                ->setEntityId($product->getId())
                ->setRequestPath($requestPath)
                ->setTargetPath('catalog/product/view/id/' . $product->getId())
                ->setIsAutogenerated(1);

            $this->urlRewriteResource->save($urlRewrite);
        }
    }

    private function generateRequestPath($product, $store): string
    {
        // Store-specific URL logic
        $urlKey = $product->getUrlKey();
        $storeCode = $store->getCode();

        return match ($storeCode) {
            'europe_de' => 'produkt/' . $urlKey . '.html',
            'europe_fr' => 'produit/' . $urlKey . '.html',
            default => 'product/' . $urlKey . '.html',
        };
    }
}
```

---

## Shared vs Separate Catalogs

### Shared Catalog (Single Root Category)

```
Website 1: example.com
  Store Group: Main Store (Root Category: Default)
    Store View: English
    Store View: Spanish

Website 2: wholesale.example.com
  Store Group: B2B Store (Root Category: Default)
    Store View: B2B English
```

**Use case:** Same products, different pricing or display

### Separate Catalogs (Different Root Categories)

```
Website 1: example.com
  Store Group: Retail (Root Category: Retail Catalog)
    Store View: English

Website 2: wholesale.example.com
  Store Group: Wholesale (Root Category: Wholesale Catalog)
    Store View: B2B English
```

**Use case:** Completely different product assortments

### Store-Specific Product Visibility

**File**: `Vendor/MultiStore/Plugin/Catalog/Product/VisibilityExtend.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\MultiStore\Plugin\Catalog\Product;

use Magento\Catalog\Model\ResourceModel\Product\Collection;
use Magento\Store\Model\StoreManagerInterface;

class VisibilityExtend
{
    public function __construct(
        private readonly StoreManagerInterface $storeManager
    ) {
    }

    /**
     * Filter products by store-specific attribute
     */
    public function afterAddStoreFilter(
        Collection $subject,
        Collection $result
    ): Collection {
        $storeId = $this->storeManager->getStore()->getId();

        // Add custom store visibility filter
        $result->addAttributeToFilter('store_visibility', [
            ['finset' => $storeId],
            ['null' => true]
        ]);

        return $result;
    }
}
```

### Store-Specific Pricing

**File**: `Vendor/MultiStore/Pricing/Price/StorePrice.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\MultiStore\Pricing\Price;

use Magento\Catalog\Model\Product;
use Magento\Framework\Pricing\Price\AbstractPrice;
use Magento\Store\Model\StoreManagerInterface;

class StorePrice extends AbstractPrice
{
    public function __construct(
        Product $saleableItem,
        $quantity,
        \Magento\Framework\Pricing\Adjustment\CalculatorInterface $calculator,
        \Magento\Framework\Pricing\PriceCurrencyInterface $priceCurrency,
        private readonly StoreManagerInterface $storeManager
    ) {
        parent::__construct($saleableItem, $quantity, $calculator, $priceCurrency);
    }

    /**
     * Get price value based on store
     */
    public function getValue(): float
    {
        $storeId = (int)$this->storeManager->getStore()->getId();
        $basePrice = (float)$this->product->getPrice();

        // Apply store-specific pricing logic
        return match ($storeId) {
            2 => $basePrice * 1.2, // Europe store: +20%
            3 => $basePrice * 0.9, // Wholesale: -10%
            default => $basePrice,
        };
    }
}
```

---

## Performance Optimization

### Redis Configuration for Multi-Store

**File**: `app/etc/env.php`

```php
<?php
return [
    'cache' => [
        'frontend' => [
            'default' => [
                'backend' => 'Cm_Cache_Backend_Redis',
                'backend_options' => [
                    'server' => '127.0.0.1',
                    'port' => '6379',
                    'database' => '0',
                    'compress_data' => '1',
                    'persistent' => '',
                ]
            ],
            'page_cache' => [
                'backend' => 'Cm_Cache_Backend_Redis',
                'backend_options' => [
                    'server' => '127.0.0.1',
                    'port' => '6379',
                    'database' => '1',
                    'compress_data' => '0',
                    'persistent' => '',
                ]
            ]
        ]
    ],
    'session' => [
        'save' => 'redis',
        'redis' => [
            'host' => '127.0.0.1',
            'port' => '6379',
            'database' => '2',
            'persistent_identifier' => 'sess-db2',
        ]
    ]
];
```

### Store-Specific Cache Tags

**File**: `Vendor/MultiStore/Model/Cache/Type/StoreCache.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\MultiStore\Model\Cache\Type;

use Magento\Framework\App\Cache\Type\FrontendPool;
use Magento\Framework\Cache\Frontend\Decorator\TagScope;

class StoreCache extends TagScope
{
    public const TYPE_IDENTIFIER = 'multistore_cache';
    public const CACHE_TAG = 'MULTISTORE';

    public function __construct(FrontendPool $cacheFrontendPool)
    {
        parent::__construct(
            $cacheFrontendPool->get(self::TYPE_IDENTIFIER),
            self::CACHE_TAG
        );
    }
}
```

**Usage:**

```php
<?php
// Cache with store-specific tag
$cacheKey = 'custom_data_store_' . $storeId;
$cacheTags = [
    \Vendor\MultiStore\Model\Cache\Type\StoreCache::CACHE_TAG,
    'store_' . $storeId
];

$cache->save($data, $cacheKey, $cacheTags, 3600);

// Clear cache for specific store
$cache->clean(\Zend_Cache::CLEANING_MODE_MATCHING_TAG, ['store_' . $storeId]);
```

### Database Optimization

**Separate indexers by store:**

```bash
# Reindex specific store
bin/magento indexer:reindex --store=europe_en

# Status per store
bin/magento indexer:status --store=europe_en
```

**Optimize queries with store filters:**

```php
<?php
// Always add store filter early in query
$collection->addStoreFilter($storeId)
    ->addAttributeToSelect('*')
    ->addAttributeToFilter('status', 1);

// NOT
$collection->addAttributeToSelect('*')
    ->addAttributeToFilter('status', 1)
    ->addStoreFilter($storeId); // Filter applied after loading all attributes
```

---

## CLI Store Management

### Useful Commands

```bash
# List all stores
bin/magento store:list

# List all websites
bin/magento store:website:list

# Create store view
# Note: store:view:create is not a core Magento CLI command.
# Create store views through the Admin Panel: Stores > All Stores > Create Store View.
# Or use the REST API:
# curl -X POST https://your-store.test/rest/V1/store/storeViews \
#   -H "Authorization: Bearer {token}" \
#   -H "Content-Type: application/json" \
#   -d '{"store_view": {"code": "europe_it", "name": "Europe Italian", "website_id": 2, "store_group_id": 2}}'

# Set base URLs
bin/magento setup:store-config:set \
    --base-url="https://italy.example.com/" \
    --base-url-secure="https://italy.example.com/" \
    --store=europe_it

# Deploy static content for specific stores
bin/magento setup:static-content:deploy en_US de_DE -s europe_en,europe_de

# Clear cache for specific store
bin/magento cache:clean --store=europe_en
```

---

## Testing Multi-Store Functionality

### Unit Test

**File**: `Vendor/MultiStore/Test/Unit/Model/ConfigTest.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\MultiStore\Test\Unit\Model;

use PHPUnit\Framework\TestCase;
use Vendor\MultiStore\Model\Config;
use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Store\Model\StoreManagerInterface;

class ConfigTest extends TestCase
{
    private Config $config;
    private $scopeConfig;
    private $storeManager;

    protected function setUp(): void
    {
        $this->scopeConfig = $this->createMock(ScopeConfigInterface::class);
        $this->storeManager = $this->createMock(StoreManagerInterface::class);

        $this->config = new Config(
            $this->scopeConfig,
            $this->storeManager
        );
    }

    public function testGetCurrencyReturnsWebsiteScopedValue(): void
    {
        $websiteId = 2;
        $expectedCurrency = 'EUR';

        $website = $this->createMock(\Magento\Store\Api\Data\WebsiteInterface::class);
        $website->method('getId')->willReturn($websiteId);

        $this->storeManager->method('getWebsite')->willReturn($website);

        $this->scopeConfig->expects($this->once())
            ->method('getValue')
            ->with(
                $this->equalTo('multistore_config/regional/currency_code'),
                $this->equalTo(\Magento\Store\Model\ScopeInterface::SCOPE_WEBSITE),
                $this->equalTo($websiteId)
            )
            ->willReturn($expectedCurrency);

        $result = $this->config->getCurrency();

        $this->assertEquals($expectedCurrency, $result);
    }
}
```

### Integration Test

**File**: `Vendor/MultiStore/Test/Integration/Model/StoreConfigTest.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\MultiStore\Test\Integration\Model;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

class StoreConfigTest extends TestCase
{
    private $objectManager;

    protected function setUp(): void
    {
        $this->objectManager = Bootstrap::getObjectManager();
    }

    /**
     * @magentoDataFixture Magento/Store/_files/second_website_with_two_stores.php
     * @magentoConfigFixture default_store multistore_config/regional/currency_code USD
     * @magentoConfigFixture fixture_second_store multistore_config/regional/currency_code EUR
     */
    public function testStoreSpecificConfiguration(): void
    {
        $config = $this->objectManager->get(\Vendor\MultiStore\Model\Config::class);

        $storeManager = $this->objectManager->get(\Magento\Store\Model\StoreManagerInterface::class);
        $defaultStore = $storeManager->getStore('default');
        $secondStore = $storeManager->getStore('fixture_second_store');

        // Test default store
        $storeManager->setCurrentStore($defaultStore->getId());
        $this->assertEquals('USD', $config->getCurrency());

        // Test second store
        $storeManager->setCurrentStore($secondStore->getId());
        $this->assertEquals('EUR', $config->getCurrency());
    }
}
```

---

## Summary

**Key Takeaways:**
- Three-tier hierarchy: Global → Website → Store → Store View
- Configuration cascades with scope inheritance; lower scopes override higher
- Themes can be assigned per store view with fallback chain
- Base URLs configured per website/store for domain/path-based separation
- Shared catalogs use same root category; separate catalogs use different roots
- Store-specific caching, indexing, and database optimization critical for performance
- MAGE_RUN_CODE and MAGE_RUN_TYPE server variables determine store context

---

## Assumptions

- **Magento Version:** 2.4.7+ (Adobe Commerce or Open Source)
- **PHP Version:** 8.2+ with sufficient memory for multi-store operations
- **Web Server:** Nginx or Apache with virtual host or path-based routing
- **Redis:** Configured for cache, session, and page cache
- **Database:** MySQL 8.0+ with optimized configuration for multi-store queries

## Why This Approach

- **Hierarchical Scopes:** Clean separation of concerns (global defaults, regional settings, locale-specific content)
- **Configuration Inheritance:** DRY principle for shared settings with store-specific overrides
- **Theme Fallback:** Reusability of base themes with store-specific customizations
- **URL Flexibility:** Domain, subdomain, or path-based store identification
- **Performance:** Store-specific caching and indexing reduce overhead

## Security Impact

- **Store Isolation:** Customers, orders, and sessions isolated by website
- **ACL:** Admin users can be restricted to specific websites/stores
- **Base URLs:** HTTPS enforcement per store; mixed content warnings avoided
- **CSRF:** Form keys scoped to store to prevent cross-store attacks
- **Secrets:** Store credentials (API keys, payment configs) isolated by scope

## Performance Impact

- **FPC:** Store-specific cache entries multiply storage requirements
- **Database:** Store filters on collections critical for query performance
- **Indexers:** Store-specific reindexing reduces full reindex time
- **Redis:** Separate databases for cache/session prevent key collisions
- **Static Content:** Deploy per locale/theme to minimize asset size

## Backward Compatibility

- **Store Structure:** Adding stores/websites doesn't break existing configuration
- **Scope Changes:** Changing scope of config field requires migration script
- **URL Changes:** URL rewrites preserve SEO when migrating to new base URLs
- **Theme Changes:** Theme inheritance ensures compatibility with parent theme updates

## Tests to Add

**Unit Tests:**
```php
testScopeInheritance()
testStoreSpecificConfig()
testThemeFallback()
```

**Integration Tests:**
```php
testMultiStoreCheckout()
testStoreSpecificPricing()
testCrossStor eNavigation()
```

**Load Tests:**
```bash
# Test multi-store performance under load
ab -n 10000 -c 100 https://europe.example.com/
ab -n 10000 -c 100 https://us.example.com/
```

## Docs to Update

- **README.md:** Multi-store setup instructions, server configuration
- **docs/STORES.md:** Store hierarchy diagram, scope reference
- **docs/DEPLOYMENT.md:** Static content deployment strategies per store
- **Admin User Guide:** Screenshots of store creation, theme assignment, base URL configuration

## Related Documentation

### Related Guides

- [Full Page Cache Strategy for High-Performance Magento](full-page-cache-strategy.md)
- [EAV System Architecture: Understanding Entity-Attribute-Value in Magento 2](../explanations/eav-system.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Customer Overview](../../modules/customer/README.md)
