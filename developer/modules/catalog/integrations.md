---
title: "Magento_Catalog Integrations"
module: "Magento_Catalog"
doc_type: "integrations"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Catalog Integrations

## Overview

The Magento_Catalog module serves as the central hub for product and category data, integrating deeply with nearly every other module in the system. This document details how Catalog integrates with Inventory, Search, Quote, Sales, and other critical subsystems.

**Target Version**: Magento 2.4.7+ / PHP 8.2+

## Architecture of Integrations

Catalog integrations follow these patterns:

1. **Service Contracts**: Other modules depend on `Magento\Catalog\Api` interfaces
2. **Events**: Catalog dispatches events that other modules observe
3. **Plugins**: Other modules intercept catalog operations
4. **Extension Attributes**: Data exchange without modifying core contracts
5. **Indexers**: Shared index tables for performance

## 1. Inventory Integration (MSI)

### Architecture

Multi-Source Inventory (MSI) extends catalog with source-aware stock management.

**Key Modules**:
- `Magento_Inventory`
- `Magento_InventoryApi`
- `Magento_InventoryCatalog`
- `Magento_InventorySales`
- `Magento_InventoryCatalogApi`

### Integration Points

#### Extension Attributes

```php
// Product interface extended with stock data
interface ProductInterface
{
    /**
     * @return \Magento\CatalogInventory\Api\Data\StockStatusInterface|null
     */
    public function getExtensionAttributes();
}
```

#### Stock Status Enrichment

```php
namespace Magento\InventoryCatalog\Plugin\Catalog\Model;

class ProductPlugin
{
    public function __construct(
        private readonly \Magento\InventorySalesApi\Api\StockResolverInterface $stockResolver,
        private readonly \Magento\InventorySalesApi\Api\GetProductSalableQtyInterface $getProductSalableQty,
        private readonly \Magento\Store\Model\StoreManagerInterface $storeManager
    ) {}

    /**
     * Add stock data to product after load
     */
    public function afterLoad(
        \Magento\Catalog\Model\Product $subject,
        $result
    ) {
        if (!$subject->hasData('is_salable')) {
            $sku = $subject->getSku();
            $websiteCode = $this->storeManager->getWebsite()->getCode();
            $stock = $this->stockResolver->execute('website', $websiteCode);

            try {
                $qty = $this->getProductSalableQty->execute($sku, $stock->getStockId());
                $subject->setData('is_salable', $qty > 0);
                $subject->setData('salable_qty', $qty);
            } catch (\Exception $e) {
                $subject->setData('is_salable', false);
                $subject->setData('salable_qty', 0);
            }
        }

        return $result;
    }
}
```

#### Configuring di.xml

```xml
<!-- Note: InventoryCatalog plugins target the ResourceModel, not the Model directly -->
<type name="Magento\Catalog\Model\ResourceModel\Product">
    <plugin name="inventory_catalog_product_source_items"
            type="Magento\InventoryCatalog\Plugin\Catalog\Model\ResourceModel\Product\CreateSourceItemsPlugin"/>
</type>
```

#### Stock Check Before Add to Cart

```php
namespace Magento\InventorySales\Model\IsProductSalableCondition;

class IsSalableWithReservationsCondition
{
    /**
     * Check if product is salable considering reservations
     */
    public function execute(string $sku, int $stockId): bool
    {
        // Get source item quantities
        $sourceItemQty = $this->getSourceItemQty->execute($sku, $stockId);

        // Subtract reservations (pending orders)
        $reservations = $this->getReservationsQuantity->execute($sku, $stockId);

        $salableQty = $sourceItemQty + $reservations;

        return $salableQty > 0;
    }
}
```

### Practical Example: Custom Stock Check

```php
namespace Vendor\Module\Service;

class StockChecker
{
    public function __construct(
        private readonly \Magento\InventorySalesApi\Api\AreProductsSalableInterface $areProductsSalable,
        private readonly \Magento\InventorySalesApi\Api\StockResolverInterface $stockResolver,
        private readonly \Magento\Store\Model\StoreManagerInterface $storeManager
    ) {}

    public function checkMultipleProducts(array $skus): array
    {
        $websiteCode = $this->storeManager->getWebsite()->getCode();
        $stockId = $this->stockResolver->execute('website', $websiteCode)->getStockId();

        $results = $this->areProductsSalable->execute($skus, $stockId);

        $status = [];
        foreach ($results as $result) {
            $status[$result->getSku()] = [
                'is_salable' => $result->isSalable(),
                'errors' => $result->getErrors()
            ];
        }

        return $status;
    }

    public function getStockDetailsBySku(string $sku): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('sku', $sku)
            ->create();

        $sourceItems = $this->sourceItemRepository->getList($searchCriteria)->getItems();

        $stockDetails = [];
        foreach ($sourceItems as $sourceItem) {
            $stockDetails[] = [
                'source_code' => $sourceItem->getSourceCode(),
                'quantity' => $sourceItem->getQuantity(),
                'status' => $sourceItem->getStatus()
            ];
        }

        return $stockDetails;
    }
}
```

### Observer: Update Stock on Product Save

```php
namespace Magento\InventoryCatalog\Observer;

class ProcessSourceItemsObserver implements ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $observer->getEvent()->getProduct();

        if ($sourceItemData = $product->getData('source_item_data')) {
            foreach ($sourceItemData as $sourceCode => $data) {
                $this->sourceItemsSave->execute([
                    'sku' => $product->getSku(),
                    'source_code' => $sourceCode,
                    'quantity' => $data['quantity'],
                    'status' => $data['status']
                ]);
            }
        }
    }
}
```

## 2. Search Integration (Elasticsearch/OpenSearch)

### Architecture

Catalog provides base data; `Magento_CatalogSearch` and `Magento_Elasticsearch` handle indexing and queries.

**Key Modules**:
- `Magento_CatalogSearch`
- `Magento_Elasticsearch` / `Magento_Elasticsearch7` / `Magento_Elasticsearch8`
- `Magento_Search`

### Integration Points

#### Product Attribute Searchability

```php
namespace Magento\Catalog\Model\ResourceModel\Eav;

class Attribute extends \Magento\Eav\Model\ResourceModel\Entity\Attribute
{
    /**
     * Attributes can be marked as searchable
     */
    public function getSearchableAttributes(): array
    {
        $connection = $this->getConnection();

        $select = $connection->select()
            ->from(['main_table' => $this->getMainTable()])
            ->join(
                ['additional_table' => $this->getTable('catalog_eav_attribute')],
                'main_table.attribute_id = additional_table.attribute_id',
                []
            )
            ->where('additional_table.is_searchable = ?', 1)
            ->where('main_table.entity_type_id = ?', $this->getEntityTypeId());

        return $connection->fetchAll($select);
    }
}
```

#### Search Indexer Data Provider

```php
namespace Magento\Elasticsearch\Model\Adapter;

class DataMapper
{
    /**
     * Map product data for Elasticsearch indexing
     */
    public function map(array $documentData, $storeId, array $context = []): array
    {
        $document = [];

        foreach ($documentData as $productId => $productData) {
            $document[$productId] = [
                'sku' => $productData['sku'],
                'name' => $productData['name'],
                'description' => $productData['description'],
                'price' => $productData['price'],
                'category_ids' => $productData['category_ids'],
                'visibility' => $productData['visibility'],
                'status' => $productData['status']
            ];

            // Add searchable attributes
            foreach ($this->searchableAttributes as $attribute) {
                $attributeCode = $attribute->getAttributeCode();
                if (isset($productData[$attributeCode])) {
                    $document[$productId][$attributeCode] = $productData[$attributeCode];
                }
            }
        }

        return $document;
    }
}
```

#### Custom Search Weight

```php
namespace Vendor\Module\Model\Search;

class CustomWeightProvider
{
    /**
     * Apply custom search weight to attributes
     */
    public function getAttributeWeight(string $attributeCode): int
    {
        $weights = [
            'sku' => 10,
            'name' => 5,
            'short_description' => 3,
            'description' => 1,
            'manufacturer' => 4
        ];

        return $weights[$attributeCode] ?? 1;
    }
}
```

#### Search Query Plugin

```php
namespace Vendor\Module\Plugin\CatalogSearch;

class SearchPlugin
{
    /**
     * Modify search query before execution
     */
    public function beforeSearch(
        \Magento\CatalogSearch\Model\ResourceModel\Fulltext\Collection $subject,
        $query
    ) {
        // Add custom filters
        $subject->addFieldToFilter('status', ['eq' => 1]);
        $subject->addFieldToFilter('visibility', ['in' => [2, 3, 4]]);

        return [$query];
    }

    /**
     * Modify search results after execution
     */
    public function afterGetItems(
        \Magento\CatalogSearch\Model\ResourceModel\Fulltext\Collection $subject,
        $result
    ) {
        // Apply custom scoring
        usort($result, function ($a, $b) {
            return $this->calculateRelevanceScore($b) - $this->calculateRelevanceScore($a);
        });

        return $result;
    }

    private function calculateRelevanceScore($product): float
    {
        $score = 0;

        // Boost new products
        $createdDate = new \DateTime($product->getCreatedAt());
        $daysSinceCreation = $createdDate->diff(new \DateTime())->days;
        if ($daysSinceCreation < 30) {
            $score += 10;
        }

        // Boost highly rated products
        if ($product->getRating() > 4) {
            $score += 5;
        }

        return $score;
    }
}
```

### Reindex Products for Search

```bash
# Full search reindex
bin/magento indexer:reindex catalogsearch_fulltext

# Check search indexer status
bin/magento indexer:status catalogsearch_fulltext
```

## 3. Quote Integration (Shopping Cart)

### Architecture

Catalog products are converted to quote items when added to cart.

**Key Modules**:
- `Magento_Quote`
- `Magento_Checkout`

### Integration Points

#### Product to Quote Item Conversion

```php
namespace Magento\Quote\Model\Quote;

class Item extends \Magento\Quote\Model\Quote\Item\AbstractItem
{
    /**
     * Initialize quote item from product
     */
    public function setProduct(\Magento\Catalog\Model\Product $product)
    {
        $this->_product = $product;

        $this->setProductId($product->getId())
            ->setProductType($product->getTypeId())
            ->setSku($product->getSku())
            ->setName($product->getName())
            ->setWeight($product->getWeight())
            ->setTaxClassId($product->getTaxClassId())
            ->setBaseCost($product->getCost());

        // Copy custom options
        if ($options = $product->getCustomOptions()) {
            foreach ($options as $option) {
                $this->addOption($option);
            }
        }

        return $this;
    }

    /**
     * Calculate item row total
     */
    public function calcRowTotal()
    {
        $qty = $this->getQty();
        $price = $this->getPrice();
        $discount = $this->getDiscountAmount();

        $rowTotal = $price * $qty - $discount;
        $this->setRowTotal($rowTotal);

        return $this;
    }
}
```

#### Add to Cart Flow

```php
namespace Magento\Quote\Model;

class Quote extends \Magento\Framework\Model\AbstractExtensibleModel implements \Magento\Quote\Api\Data\CartInterface
{
    /**
     * Add product to quote
     */
    public function addProduct(
        \Magento\Catalog\Model\Product $product,
        $request = null,
        $processMode = \Magento\Catalog\Model\Product\Type\AbstractType::PROCESS_MODE_FULL
    ) {
        if ($request === null) {
            $request = 1;
        }

        if (is_numeric($request)) {
            $request = new \Magento\Framework\DataObject(['qty' => $request]);
        }

        // Validate product
        $cartCandidate = $product->getTypeInstance()->prepareForCart($request, $product);

        if (is_string($cartCandidate)) {
            throw new \Magento\Framework\Exception\LocalizedException(__($cartCandidate));
        }

        // Check if item already exists
        $item = $this->getItemByProduct($product);

        if ($item) {
            $item->setQty($item->getQty() + $request->getQty());
        } else {
            $item = $this->itemFactory->create();
            $item->setQuote($this);
            $item->setProduct($product);
            $item->setQty($request->getQty());
            $this->addItem($item);
        }

        $this->_eventManager->dispatch(
            'sales_quote_add_item',
            ['quote_item' => $item]
        );

        return $item;
    }
}
```

#### Stock Validation Before Add to Cart

> **Note:** In Magento 2.4.7 with MSI, stock validation is handled by
> `Magento\InventorySales\Plugin\StockState\CheckQuoteItemQtyPlugin` (a plugin,
> not an observer). The class `CheckQuoteItemQtyObserver` does not exist.
> The example below shows the conceptual validation pattern:

```php
// Conceptual example — actual implementation is a plugin in module-inventory-sales
class CheckQuoteItemQtyPlugin
{
    public function execute(\Magento\Framework\Event\Observer $observer): void
    {
        /** @var \Magento\Quote\Model\Quote\Item $quoteItem */
        $quoteItem = $observer->getEvent()->getItem();

        if (!$quoteItem->getProductId()) {
            return;
        }

        $product = $quoteItem->getProduct();
        $qty = $quoteItem->getQty();

        // Check stock
        $result = $this->stockState->checkQuoteItemQty(
            $product->getId(),
            $qty,
            $product->getStore()->getWebsiteId()
        );

        if ($result->getHasError()) {
            $quoteItem->addErrorInfo(
                'cataloginventory',
                \Magento\CatalogInventory\Helper\Data::ERROR_QTY,
                $result->getMessage()
            );

            $quoteItem->getQuote()->addErrorInfo(
                'stock',
                'cataloginventory',
                \Magento\CatalogInventory\Helper\Data::ERROR_QTY,
                $result->getMessage()
            );
        }
    }
}
```

### Custom Add to Cart Logic

```php
namespace Vendor\Module\Service;

class CartService
{
    public function __construct(
        private readonly \Magento\Checkout\Model\Cart $cart,
        private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
        private readonly \Magento\Framework\DataObjectFactory $dataObjectFactory
    ) {}

    public function addProductWithCustomOptions(string $sku, int $qty, array $customOptions): void
    {
        $product = $this->productRepository->get($sku);

        $params = $this->dataObjectFactory->create([
            'data' => [
                'product' => $product->getId(),
                'qty' => $qty,
                'options' => $customOptions
            ]
        ]);

        $this->cart->addProduct($product, $params);
        $this->cart->save();
    }

    public function addMultipleProducts(array $products): void
    {
        foreach ($products as $productData) {
            $product = $this->productRepository->get($productData['sku']);

            $params = $this->dataObjectFactory->create([
                'data' => [
                    'qty' => $productData['qty']
                ]
            ]);

            $this->cart->addProduct($product, $params);
        }

        $this->cart->save();
    }
}
```

## 4. Sales Integration (Orders)

### Architecture

Quote items convert to order items when order is placed.

**Key Modules**:
- `Magento_Sales`
- `Magento_SalesRule`

### Integration Points

#### Quote Item to Order Item Conversion

```php
namespace Magento\Sales\Model\Order;

class Item extends \Magento\Sales\Model\AbstractModel
{
    /**
     * Initialize from quote item
     */
    public function setQuoteItem(\Magento\Quote\Model\Quote\Item $item)
    {
        $this->setQuoteItemId($item->getId())
            ->setProductId($item->getProductId())
            ->setProductType($item->getProductType())
            ->setSku($item->getSku())
            ->setName($item->getName())
            ->setDescription($item->getDescription())
            ->setWeight($item->getWeight())
            ->setQtyOrdered($item->getQty())
            ->setPrice($item->getPrice())
            ->setBasePrice($item->getBasePrice())
            ->setOriginalPrice($item->getOriginalPrice())
            ->setRowTotal($item->getRowTotal())
            ->setBaseRowTotal($item->getBaseRowTotal());

        return $this;
    }
}
```

#### Observer: Subtract Stock After Order

```php
namespace Magento\CatalogInventory\Observer;

class SubtractQuoteInventoryObserver implements ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer): void
    {
        /** @var \Magento\Quote\Model\Quote $quote */
        $quote = $observer->getEvent()->getQuote();

        foreach ($quote->getAllItems() as $item) {
            $product = $item->getProduct();

            if ($product && $product->getTypeId() === 'simple') {
                $this->stockManagement->registerProductSale(
                    $product->getSku(),
                    $item->getQty()
                );
            }
        }
    }
}
```

#### Observer: Update Product Sales Count

```php
namespace Vendor\Module\Observer;

class UpdateProductSalesObserver implements ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer): void
    {
        /** @var \Magento\Sales\Model\Order $order */
        $order = $observer->getEvent()->getOrder();

        foreach ($order->getAllItems() as $item) {
            $product = $this->productRepository->getById($item->getProductId());

            $currentSalesCount = (int) $product->getData('total_sales_count');
            $product->setData('total_sales_count', $currentSalesCount + $item->getQtyOrdered());

            $this->productRepository->save($product);
        }
    }
}
```

### Custom Order Processing

```php
namespace Vendor\Module\Service;

class OrderProcessor
{
    public function __construct(
        private readonly \Magento\Sales\Api\OrderRepositoryInterface $orderRepository,
        private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository
    ) {}

    /**
     * Process products after order placement
     */
    public function processOrderProducts(int $orderId): void
    {
        $order = $this->orderRepository->get($orderId);

        foreach ($order->getAllItems() as $item) {
            $product = $this->productRepository->getById($item->getProductId());

            // Update custom attributes
            $product->setData('last_sold_at', date('Y-m-d H:i:s'));

            // Increment sold count
            $soldCount = (int) $product->getData('sold_count');
            $product->setData('sold_count', $soldCount + $item->getQtyOrdered());

            $this->productRepository->save($product);
        }
    }
}
```

## 5. Catalog Rule Integration (Promotional Pricing)

### Architecture

Catalog price rules apply discounts to products before they enter the cart.

**Key Module**: `Magento_CatalogRule`

### Integration Points

#### Rule Conditions on Products

```php
namespace Magento\CatalogRule\Model\Rule\Condition;

class Product extends \Magento\Rule\Model\Condition\Product\AbstractProduct
{
    /**
     * Get product attributes that can be used in rule conditions
     */
    public function loadAttributeOptions()
    {
        $productAttributes = $this->productResource->loadAllAttributes()->getAttributesByCode();

        $attributes = [];
        foreach ($productAttributes as $attribute) {
            if ($attribute->getFrontendInput() && $attribute->getIsVisible()) {
                $attributes[$attribute->getAttributeCode()] = $attribute->getFrontendLabel();
            }
        }

        $this->setAttributeOption($attributes);
        return $this;
    }

    /**
     * Validate product against rule condition
     */
    public function validate(\Magento\Framework\Model\AbstractModel $model)
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $model->getProduct();

        if (!$product instanceof \Magento\Catalog\Model\Product) {
            $product = $this->productRepository->getById($model->getProductId());
        }

        $product->setData($this->getAttribute(), $product->getData($this->getAttribute()));

        return parent::validate($product);
    }
}
```

#### Apply Rule Price

```php
namespace Magento\CatalogRule\Model;

class ResourceModel\Rule
{
    /**
     * Get rule price for product
     */
    public function getRulePrice($date, $websiteId, $customerGroupId, $productId)
    {
        $connection = $this->getConnection();

        $select = $connection->select()
            ->from($this->getTable('catalogrule_product_price'), 'rule_price')
            ->where('rule_date = ?', $date)
            ->where('website_id = ?', $websiteId)
            ->where('customer_group_id = ?', $customerGroupId)
            ->where('product_id = ?', $productId)
            ->order('rule_price ASC')
            ->limit(1);

        return $connection->fetchOne($select);
    }
}
```

#### Observer: Apply Catalog Rules to Price

```php
namespace Magento\CatalogRule\Observer;

class ProcessFrontFinalPriceObserver implements ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $observer->getEvent()->getProduct();
        $finalPrice = $product->getData('final_price');

        $rulePrice = $this->catalogRuleResourceModel->getRulePrice(
            $this->dateTime->scopeDate($this->storeManager->getStore()->getId()),
            $this->storeManager->getStore()->getWebsiteId(),
            $this->customerSession->getCustomerGroupId(),
            $product->getId()
        );

        if ($rulePrice !== false && $rulePrice < $finalPrice) {
            $product->setFinalPrice($rulePrice);
        }
    }
}
```

### Custom Catalog Rule

```php
namespace Vendor\Module\Model\CatalogRule;

class CustomRule extends \Magento\CatalogRule\Model\Rule
{
    /**
     * Apply custom discount logic
     */
    public function calcProductPriceRule(\Magento\Catalog\Model\Product $product, $price)
    {
        $discount = parent::calcProductPriceRule($product, $price);

        // Apply additional discount for loyal customers
        if ($this->customerSession->isLoggedIn()) {
            $customer = $this->customerSession->getCustomer();
            $loyaltyTier = $customer->getData('loyalty_tier');

            $additionalDiscount = match ($loyaltyTier) {
                'gold' => 0.10,
                'platinum' => 0.15,
                default => 0
            };

            $discount = $discount * (1 - $additionalDiscount);
        }

        return $discount;
    }
}
```

## 6. URL Rewrite Integration

### Architecture

Products and categories generate SEO-friendly URLs via `Magento_UrlRewrite`.

**Key Module**: `Magento_CatalogUrlRewrite`

### Integration Points

#### Generate Product URL

```php
namespace Magento\CatalogUrlRewrite\Model;

class ProductUrlPathGenerator
{
    /**
     * Generate URL path for product
     */
    public function getUrlPath($product, $category = null)
    {
        $path = '';

        if ($category) {
            $path = $category->getUrlPath() . '/';
        }

        $path .= $product->getUrlKey();

        return $path;
    }

    /**
     * Generate URL key from product name
     */
    public function getUrlKey($product)
    {
        $urlKey = $product->getUrlKey();

        if ($urlKey === null || $urlKey === '') {
            $urlKey = $product->formatUrlKey($product->getName());
        }

        return $urlKey;
    }
}
```

#### Observer: Regenerate URLs After Save

```php
namespace Magento\CatalogUrlRewrite\Observer;

class ProductProcessUrlRewriteSavingObserver implements ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer): void
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $observer->getEvent()->getProduct();

        if ($product->dataHasChangedFor('url_key') || $product->dataHasChangedFor('visibility')) {
            $this->urlRewriteGenerator->generate($product);
        }
    }
}
```

---

**Assumptions:**
- Magento 2.4.7+ with MSI enabled
- Elasticsearch/OpenSearch configured for search
- Standard quote-to-order flow
- Catalog price rules enabled

**Why this approach:**
Service contracts enable clean module boundaries. Extension attributes provide data exchange without BC breaks. Events allow decoupled integrations. Indexers optimize cross-module queries.

**Security impact:**
Stock checks prevent overselling. Price calculations respect customer group permissions. Search respects product visibility and ACL.

**Performance impact:**
MSI adds overhead (~5ms per stock check). Search indexing is async. Quote-to-order conversion is transactional. Catalog rule index pre-calculates prices (faster than runtime).

**Backward compatibility:**
All integrations use service contracts. Extension attributes are additive (BC-safe). Events maintain backward compatibility in data structure.

**Tests to add:**
- Integration tests for full add-to-cart flow with stock validation
- Functional tests for search indexing and query
- Unit tests for price rule application
- Performance tests for multi-source stock checks

**Docs to update:**
- INTEGRATIONS.md (this file) when adding new integration patterns
- Add sequence diagrams for complex flows
- Document breaking changes in VERSION_COMPATIBILITY.md
