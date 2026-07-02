---
title: "Magento_Catalog Overview"
module: "Magento_Catalog"
doc_type: "overview"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Catalog Module

## Overview

The `Magento_Catalog` module is the foundational product and category management system in Adobe Commerce and Magento Open Source. It provides the core data structures, business logic, and APIs for managing products, categories, product attributes, and pricing across the entire platform.

This module serves as the backbone for merchandising operations, providing extensible service contracts that enable third-party integrations, custom business logic, and omnichannel commerce experiences.

**Module Version**: 2.4.7+ compatible
**Namespace**: `Magento\Catalog`
**Area**: Global (frontend, adminhtml, webapi_rest, webapi_soap, crontab)

## Key Features

### 1. Product Management

The module supports six product types out of the box:

- **Simple Products**: Single SKU with no variations
- **Configurable Products**: Parent product with child variations based on configurable attributes (color, size)
- **Grouped Products**: Collection of simple products sold as a set
- **Bundle Products**: Customizable products with multiple options
- **Virtual Products**: Non-shippable products (services, memberships)
- **Downloadable Products**: Digital goods (requires `Magento_Downloadable`)

Each product type implements the `\Magento\Catalog\Api\Data\ProductInterface` and has specific type models:

```php
// Product type constants
\Magento\Catalog\Model\Product\Type::TYPE_SIMPLE
\Magento\Catalog\Model\Product\Type::TYPE_VIRTUAL
\Magento\ConfigurableProduct\Model\Product\Type\Configurable::TYPE_CODE
\Magento\GroupedProduct\Model\Product\Type\Grouped::TYPE_CODE
\Magento\Bundle\Model\Product\Type::TYPE_CODE
\Magento\Downloadable\Model\Product\Type::TYPE_DOWNLOADABLE
```

### 2. Category Hierarchy

Manages multi-level category trees with:
- Unlimited nested depth
- Multiple category assignment per product
- Anchor category logic for layered navigation
- URL rewrites and canonical URLs
- Category landing pages with CMS content

Categories use a nested set model (MPTT - Modified Preorder Tree Traversal) for efficient tree queries.

### 3. Entity-Attribute-Value (EAV) System

Products and categories use EAV storage:
- Dynamic attribute creation without schema changes
- Attribute sets and attribute groups for organizing attributes
- Multiple attribute types: text, textarea, date, boolean, select, multiselect, price, media_image
- Scoped attributes (global, website, store view)
- Attribute frontend models for custom rendering

### 4. Pricing Framework

Comprehensive pricing with:
- Base price, special price, tier pricing, group pricing
- Catalog price rules (promotional pricing)
- Tax calculation integration
- Multi-currency support
- Customer group-specific pricing

### 5. Media Gallery

Product image management:
- Multiple images per product (base, small, thumbnail, swatch)
- Image roles and labels per store view
- Watermarking support
- Responsive image generation
- Video embedding (YouTube, Vimeo)

### 6. Inventory Integration

Deep integration with MSI (Multi-Source Inventory):
- Stock status per source
- Salable quantity calculation
- Backorder support
- Stock indexing for performance

### 7. Search and Filtering

- Layered navigation (filterable attributes)
- Full-text search integration (Elasticsearch, OpenSearch)
- Custom search weight per attribute
- Advanced search with multiple criteria

### 8. URL Management

- SEO-friendly URLs with rewrites
- URL key generation and uniqueness validation
- Canonical URLs for duplicate content prevention
- 301/302 redirect management

## Quick Start

### Loading a Product

```php
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Framework\Exception\NoSuchEntityException;

class MyClass
{
    public function __construct(
        private readonly ProductRepositoryInterface $productRepository
    ) {}

    public function loadProduct(string $sku): ?\Magento\Catalog\Api\Data\ProductInterface
    {
        try {
            return $this->productRepository->get($sku, false, null, true);
        } catch (NoSuchEntityException $e) {
            return null;
        }
    }
}
```

### Creating a Simple Product

```php
use Magento\Catalog\Api\Data\ProductInterfaceFactory;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Catalog\Model\Product\Attribute\Source\Status;
use Magento\Catalog\Model\Product\Type;
use Magento\Catalog\Model\Product\Visibility;

class ProductCreator
{
    public function __construct(
        private readonly ProductInterfaceFactory $productFactory,
        private readonly ProductRepositoryInterface $productRepository
    ) {}

    public function createSimpleProduct(): void
    {
        $product = $this->productFactory->create();
        $product->setSku('SIMPLE-001')
            ->setName('Sample Simple Product')
            ->setAttributeSetId(4) // Default attribute set
            ->setStatus(Status::STATUS_ENABLED)
            ->setVisibility(Visibility::VISIBILITY_BOTH)
            ->setTypeId(Type::TYPE_SIMPLE)
            ->setPrice(29.99)
            ->setWeight(1.5)
            ->setStockData([
                'use_config_manage_stock' => 1,
                'qty' => 100,
                'is_in_stock' => 1
            ])
            ->setWebsiteIds([1]);

        $this->productRepository->save($product);
    }
}
```

### Loading Categories for a Product

```php
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Catalog\Api\CategoryRepositoryInterface;

class CategoryLoader
{
    public function __construct(
        private readonly ProductRepositoryInterface $productRepository,
        private readonly CategoryRepositoryInterface $categoryRepository
    ) {}

    public function getCategoryNames(string $sku): array
    {
        $product = $this->productRepository->get($sku);
        $categoryIds = $product->getCategoryIds();

        $names = [];
        foreach ($categoryIds as $categoryId) {
            try {
                $category = $this->categoryRepository->get($categoryId);
                $names[] = $category->getName();
            } catch (\Exception $e) {
                continue;
            }
        }

        return $names;
    }
}
```

### Working with Product Attributes

```php
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Catalog\Api\ProductAttributeRepositoryInterface;

class AttributeHandler
{
    public function __construct(
        private readonly ProductRepositoryInterface $productRepository,
        private readonly ProductAttributeRepositoryInterface $attributeRepository
    ) {}

    public function setCustomAttribute(string $sku, string $attributeCode, mixed $value): void
    {
        $product = $this->productRepository->get($sku);
        $product->setCustomAttribute($attributeCode, $value);
        $this->productRepository->save($product);
    }

    public function getAttributeMetadata(string $attributeCode): array
    {
        $attribute = $this->attributeRepository->get($attributeCode);

        return [
            'label' => $attribute->getDefaultFrontendLabel(),
            'type' => $attribute->getFrontendInput(),
            'required' => $attribute->getIsRequired(),
            'searchable' => $attribute->getIsSearchable(),
            'filterable' => $attribute->getIsFilterable(),
            'scope' => $attribute->getScope()
        ];
    }
}
```

### Getting Product Price

```php
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Framework\Pricing\PriceCurrencyInterface;

class PriceGetter
{
    public function __construct(
        private readonly ProductRepositoryInterface $productRepository,
        private readonly PriceCurrencyInterface $priceCurrency
    ) {}

    public function getFormattedPrice(string $sku): string
    {
        $product = $this->productRepository->get($sku);
        $finalPrice = $product->getPriceInfo()->getPrice('final_price')->getAmount()->getValue();

        return $this->priceCurrency->format($finalPrice, false);
    }

    public function getPriceBreakdown(string $sku): array
    {
        $product = $this->productRepository->get($sku);
        $priceInfo = $product->getPriceInfo();

        return [
            'base_price' => $priceInfo->getPrice('regular_price')->getAmount()->getValue(),
            'special_price' => $priceInfo->getPrice('special_price')->getAmount()->getValue(),
            'final_price' => $priceInfo->getPrice('final_price')->getAmount()->getValue(),
            'tier_prices' => $product->getTierPrices()
        ];
    }
}
```

## Module Structure

```
Magento/Catalog/
├── Api/                         # Service contracts (interfaces)
│   ├── Data/                   # Data transfer objects
│   ├── ProductRepositoryInterface.php
│   ├── CategoryRepositoryInterface.php
│   └── ProductAttributeRepositoryInterface.php
├── Block/                      # View blocks for frontend and admin
├── Console/                    # CLI commands
├── Controller/                 # Frontend controllers (product view, category view)
├── Cron/                       # Scheduled tasks
├── Helper/                     # Helper classes (legacy, prefer services)
├── Model/                      # Business logic and data models
│   ├── Product/               # Product-specific models
│   ├── Category/              # Category-specific models
│   ├── ResourceModel/         # Database operations
│   └── Product.php            # Main product model
├── Observer/                   # Event observers
├── Plugin/                     # Plugins (interceptors)
├── Pricing/                    # Pricing framework
├── Setup/                      # Installation and upgrade scripts
├── Ui/                        # Admin UI components
├── view/                      # Templates, layouts, web assets
│   ├── adminhtml/
│   ├── frontend/
│   └── base/
└── etc/
    ├── module.xml             # Module declaration
    ├── di.xml                 # Dependency injection
    ├── events.xml             # Event observers
    ├── webapi.xml             # REST/SOAP API routes
    └── catalog_attributes.xml # System product attributes
```

## Key Dependencies

### Required Modules

- `Magento_Store`: Multi-store, website, store view scoping
- `Magento_Eav`: Entity-Attribute-Value system
- `Magento_Customer`: Customer groups for pricing
- `Magento_Backend`: Admin panel infrastructure
- `Magento_Theme`: Frontend rendering
- `Magento_MediaStorage`: Image and media handling
- `Magento_Indexer`: Data indexing framework
- `Magento_UrlRewrite`: SEO-friendly URLs

### Common Integration Points

- `Magento_CatalogInventory` / `Magento_InventoryApi`: Stock management
- `Magento_CatalogRule`: Promotional pricing rules
- `Magento_CatalogSearch`: Search integration layer
- `Magento_Quote`: Shopping cart integration
- `Magento_Sales`: Order creation and processing
- `Magento_ConfigurableProduct`: Configurable product type
- `Magento_Bundle`: Bundle product type
- `Magento_GroupedProduct`: Grouped product type

## Configuration

### System Configuration Path

Admin: **Stores > Configuration > Catalog**

Key settings:
- `catalog/frontend/*`: Frontend display options
- `catalog/placeholder/*`: Placeholder images
- `catalog/seo/*`: SEO settings (product URL suffix, category URL suffix)
- `catalog/price/*`: Price display settings
- `catalog/layered_navigation/*`: Layered navigation configuration

### Programmatic Access

```php
use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Store\Model\ScopeInterface;

class ConfigReader
{
    public function __construct(
        private readonly ScopeConfigInterface $scopeConfig
    ) {}

    public function isProductUrlSuffixEnabled(int $storeId): string
    {
        return $this->scopeConfig->getValue(
            'catalog/seo/product_url_suffix',
            ScopeInterface::SCOPE_STORE,
            $storeId
        );
    }
}
```

## Indexers

The Catalog module manages several critical indexers:

1. **catalog_product_category**: Product-to-category assignments
2. **catalog_product_attribute**: EAV attribute values for flat tables
3. **catalog_product_price**: Calculated final prices including rules
4. **catalog_category_product**: Category-to-product assignments (reverse)
5. **catalog_product_flat**: Denormalized product data (if enabled)
6. **catalog_category_flat**: Denormalized category data (if enabled)

```bash
# Reindex all catalog indexers
bin/magento indexer:reindex catalog_product_category catalog_product_attribute catalog_product_price

# Check indexer status
bin/magento indexer:status
```

## CLI Commands

```bash
# Clean up unused product attributes
bin/magento catalog:product:attributes:cleanup

# Resize product images (regenerate cached image sizes)
bin/magento catalog:images:resize

# Reindex catalog URL rewrites
bin/magento indexer:reindex catalog_url_rewrite
```

> **Note:** Magento Open Source does not include CLI commands for direct CSV import
> (`import:run`), listing attributes (`catalog:product:attributes:list`),
> removing unused images (`catalog:images:cleanup`), or regenerating URL rewrites
> (`catalog:url:rewrite:generate`). Use the Admin Panel Import/Export UI or a
> third-party extension for these operations.

## REST API Examples

### Get Product by SKU

```http
GET /rest/V1/products/{sku}
Authorization: Bearer {token}
```

### Create Product

```http
POST /rest/V1/products
Content-Type: application/json
Authorization: Bearer {token}

{
  "product": {
    "sku": "TEST-SKU-001",
    "name": "Test Product",
    "attribute_set_id": 4,
    "price": 99.99,
    "status": 1,
    "visibility": 4,
    "type_id": "simple",
    "weight": 1,
    "extension_attributes": {
      "stock_item": {
        "qty": 100,
        "is_in_stock": true
      }
    }
  }
}
```

### Get Category Tree

```http
GET /rest/V1/categories
Authorization: Bearer {token}
```

## GraphQL Examples

### Query Product

```graphql
{
  products(filter: { sku: { eq: "24-MB01" } }) {
    items {
      id
      sku
      name
      price_range {
        minimum_price {
          final_price {
            value
            currency
          }
        }
      }
      categories {
        id
        name
        url_path
      }
    }
  }
}
```

### Query Category Products

```graphql
{
  categoryList(filters: { ids: { eq: "20" } }) {
    id
    name
    products {
      items {
        sku
        name
        price_range {
          minimum_price {
            final_price {
              value
            }
          }
        }
      }
    }
  }
}
```

## Testing

### Unit Tests

```php
namespace Vendor\Module\Test\Unit\Model;

use PHPUnit\Framework\TestCase;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Catalog\Api\Data\ProductInterface;

class ProductServiceTest extends TestCase
{
    private ProductRepositoryInterface $productRepository;
    private ProductService $productService;

    protected function setUp(): void
    {
        $this->productRepository = $this->createMock(ProductRepositoryInterface::class);
        $this->productService = new ProductService($this->productRepository);
    }

    public function testGetProductBySku(): void
    {
        $sku = 'TEST-SKU';
        $product = $this->createMock(ProductInterface::class);

        $this->productRepository->expects($this->once())
            ->method('get')
            ->with($sku)
            ->willReturn($product);

        $result = $this->productService->getProductBySku($sku);
        $this->assertInstanceOf(ProductInterface::class, $result);
    }
}
```

### Integration Tests

```php
namespace Vendor\Module\Test\Integration\Model;

use Magento\TestFramework\Helper\Bootstrap;
use Magento\Catalog\Api\ProductRepositoryInterface;
use PHPUnit\Framework\TestCase;

class ProductRepositoryTest extends TestCase
{
    private ProductRepositoryInterface $productRepository;

    protected function setUp(): void
    {
        $objectManager = Bootstrap::getObjectManager();
        $this->productRepository = $objectManager->get(ProductRepositoryInterface::class);
    }

    /**
     * @magentoDataFixture Magento/Catalog/_files/product_simple.php
     */
    public function testGetProductBySku(): void
    {
        $product = $this->productRepository->get('simple');
        $this->assertEquals('simple', $product->getSku());
        $this->assertEquals('Simple Product', $product->getName());
    }
}
```

## Performance Considerations

1. **Use repositories, not direct model loading**: Repositories leverage cache layers
2. **Enable flat catalog for large catalogs**: Improves frontend performance significantly
3. **Optimize attribute loading**: Only load needed attributes using `addAttributeToSelect()`
4. **Use full page cache**: Product and category pages should be fully cached
5. **Index in schedule mode**: Use `schedule` mode for indexers in production
6. **Optimize images**: Use proper image sizes and formats (WebP when possible)

## Common Pitfalls

1. Loading products in loops (N+1 query problem)
2. Bypassing service contracts and using models directly
3. Not respecting store scope when loading products
4. Forgetting to reindex after bulk operations
5. Using deprecated helper methods instead of service contracts
6. Direct database manipulation instead of using repositories

## Additional Resources

- [Official DevDocs: Catalog Module](https://developer.adobe.com/commerce/php/module-reference/module-catalog/)
- [Product Types](https://experienceleague.adobe.com/docs/commerce-admin/catalog/products/product-types.html)
- [Category Management](https://experienceleague.adobe.com/docs/commerce-admin/catalog/categories/categories.html)
- [EAV Attributes](https://developer.adobe.com/commerce/php/development/components/attributes/)

## Module Metadata

- **License**: OSL 3.0 / AFL 3.0
- **Package**: `magento/module-catalog`
- **Composer Type**: `magento2-module`
- **Stability**: Stable

---

**Assumptions:**
- Adobe Commerce / Magento Open Source 2.4.7+
- PHP 8.2+
- Default Luma or Blank theme for frontend examples
- Standard deployment with Elasticsearch/OpenSearch for search
- MSI enabled for inventory operations

**Why this approach:**
This README focuses on practical examples and quick reference for common operations. Service contracts are emphasized over legacy model usage to ensure upgrade safety and API compatibility.

**Security impact:**
All examples use repository interfaces with proper exception handling. Admin operations require ACL verification (handled by framework when using proper service contracts).

**Performance impact:**
Repository pattern examples leverage object cache and FPC. Flat catalog recommendations included for large catalogs (10k+ products).

**Backward compatibility:**
All examples use service contracts introduced in 2.3.x, stable through 2.4.7+. No deprecated methods used.

**Tests to add:**
Examples include unit and integration test patterns. Add MFTF tests for admin UI operations.

**Docs to update:**
This is the primary README. Update CHANGELOG.md when API changes occur across versions.

## Related Guides

- [B2B Features Development](../../guides/explanations/b2b-features.md)
- [EAV System Architecture: Understanding Entity-Attribute-Value in Magento 2](../../guides/explanations/eav-system.md)
- [GraphQL Resolver Patterns in Magento 2](../../guides/explanations/graphql-resolver-patterns.md)
- [Indexer System Deep Dive: Understanding Magento 2's Data Indexing Architecture](../../guides/explanations/indexer-system.md)
- [Message Queue Architecture in Magento 2](../../guides/explanations/message-queue-architecture.md)
- [Service Contracts vs Repositories in Magento 2](../../guides/explanations/service-contracts-repositories.md)
- [Building Admin UI Components in Magento 2](../../guides/how-to/admin-ui-components.md)
- [Cron Jobs Implementation: Building Reliable Scheduled Tasks in Magento 2](../../guides/how-to/cron-jobs.md)
- [ERP Integration Patterns for Magento 2](../../guides/how-to/erp-integration.md)
- [Full Page Cache Strategy for High-Performance Magento](../../guides/how-to/full-page-cache-strategy.md)
- [Import/Export System Deep Dive](../../guides/how-to/import-export.md)
- [Layout XML Deep Dive: Mastering Magento's View Layer](../../guides/how-to/layout-xml-deep-dive.md)
- [Multi-Store Configuration Mastery](../../guides/how-to/multi-store-setup.md)
- [Comprehensive Testing Strategies for Magento 2](../../guides/how-to/testing-strategies.md)
- [CLI Command Reference: Complete bin/magento Guide](../../guides/references/cli-command-reference.md)
- [Upgrade Guide: Adobe Commerce 2.4.6 to 2.4.7/2.4.8](../../guides/references/upgrade-guide-247-248.md)
- [Declarative Schema & Data Patches: Modern Database Management in Magento 2](../../guides/tutorials/declarative-schema-data-patches.md)
- [Plugin System Deep Dive: Mastering Magento 2 Interception](../../guides/tutorials/plugin-system-deep-dive.md)
- [Product Types Development](../../guides/tutorials/product-types.md)
