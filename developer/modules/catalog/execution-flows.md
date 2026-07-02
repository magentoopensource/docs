---
title: "Magento_Catalog Execution Flows"
module: "Magento_Catalog"
doc_type: "execution-flows"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Catalog Execution Flows

## Overview

This document details critical execution flows in the Magento_Catalog module, tracing the complete path from request to response with code-level detail. Understanding these flows is essential for debugging, customization, and performance optimization.

**Target Version**: Magento 2.4.7+ / PHP 8.2+

## 1. Product Save Flow

### Flow Diagram (Text)

```
Admin Controller
    └─> ProductRepository::save()
        ├─> beforeSave plugins
        ├─> Product::beforeSave()
        │   ├─> Validate required attributes
        │   ├─> Generate URL key if missing
        │   └─> Set default values
        ├─> ResourceModel\Product::save()
        │   ├─> Begin transaction
        │   ├─> Save to catalog_product_entity
        │   ├─> Save EAV attributes to type-specific tables
        │   ├─> Save website assignments
        │   ├─> Save category assignments
        │   ├─> Process media gallery
        │   ├─> Save product links (related, upsell, cross-sell)
        │   └─> Commit transaction
        ├─> Product::afterSave()
        │   ├─> Dispatch catalog_product_save_after event
        │   ├─> Save stock data
        │   ├─> Reindex affected indexers
        │   └─> Clear cache tags
        ├─> afterSave plugins
        └─> Return saved product
```

### Detailed Code Flow

#### Step 1: Controller Entry Point

```php
namespace Magento\Catalog\Controller\Adminhtml\Product;

class Save extends \Magento\Catalog\Controller\Adminhtml\Product
{
    public function execute()
    {
        $data = $this->getRequest()->getPostValue();

        if ($data) {
            try {
                // Initialize product
                $product = $this->initializationHelper->initialize(
                    $this->productBuilder->build($this->getRequest())
                );

                // Prepare product data from request
                $product->setData($data);

                // Save via repository
                $this->productRepository->save($product);

                $this->messageManager->addSuccessMessage(__('You saved the product.'));
                return $this->resultRedirectFactory->create()->setPath(
                    'catalog/*/edit',
                    ['id' => $product->getId()]
                );
            } catch (\Exception $e) {
                $this->messageManager->addErrorMessage($e->getMessage());
            }
        }
    }
}
```

#### Step 2: Repository Save Method

```php
namespace Magento\Catalog\Model;

class ProductRepository implements \Magento\Catalog\Api\ProductRepositoryInterface
{
    public function save(\Magento\Catalog\Api\Data\ProductInterface $product, $saveOptions = false)
    {
        $tierPrices = $product->getData('tier_price');

        // Validate product data
        $this->validateProduct($product);

        // Process media gallery
        $this->processMediaGallery($product);

        // Save main entity
        try {
            $this->resourceModel->save($product);
        } catch (\Exception $e) {
            throw new \Magento\Framework\Exception\CouldNotSaveException(
                __('Unable to save product'),
                $e
            );
        }

        // Save tier prices via separate service
        if ($tierPrices !== null && $saveOptions) {
            $this->tierPriceManagement->add($product, $tierPrices);
        }

        // Save product links (related, upsell, cross-sell)
        if ($saveOptions) {
            $this->linkManagement->setProductLinks($product);
        }

        // Clear repository cache
        unset($this->instances[$product->getSku()]);
        unset($this->instancesById[$product->getId()]);

        return $this->get($product->getSku(), false, $product->getStoreId());
    }

    private function validateProduct(\Magento\Catalog\Api\Data\ProductInterface $product): void
    {
        // SKU uniqueness
        if (!$product->getId()) {
            try {
                $existingProduct = $this->get($product->getSku());
                throw new \Magento\Framework\Exception\AlreadyExistsException(
                    __('Product with SKU "%1" already exists.', $product->getSku())
                );
            } catch (NoSuchEntityException $e) {
                // SKU is unique, continue
            }
        }

        // Validate required attributes
        $attributeSet = $this->attributeSetRepository->get($product->getAttributeSetId());
        foreach ($this->getRequiredAttributes($attributeSet) as $attribute) {
            if (!$product->getData($attribute->getAttributeCode())) {
                throw new \Magento\Framework\Exception\ValidatorException(
                    __('Attribute "%1" is required.', $attribute->getFrontendLabel())
                );
            }
        }
    }
}
```

#### Step 3: Model beforeSave Hook

```php
namespace Magento\Catalog\Model;

class Product extends \Magento\Catalog\Model\AbstractModel
{
    public function beforeSave()
    {
        // Generate URL key if not set
        if (!$this->getData('url_key')) {
            $this->setUrlKey($this->formatUrlKey($this->getName()));
        }

        // URL key uniqueness is validated by CatalogUrlRewrite observers,
        // not directly in beforeSave(). See CategoryUrlPathAutogeneratorObserver.

        // Set created_at / updated_at
        if (!$this->getId()) {
            $this->setCreatedAt($this->dateTime->gmtDate());
        }
        $this->setUpdatedAt($this->dateTime->gmtDate());

        // Validate SKU format
        $this->validateSku();

        // Process custom options
        if ($this->getOptions()) {
            $this->setHasOptions(1);
        }

        // Dispatch event
        $this->_eventManager->dispatch('catalog_product_save_before', ['product' => $this]);

        return parent::beforeSave();
    }

    private function validateSku(): void
    {
        if (!preg_match('/^[a-zA-Z0-9_-]+$/', $this->getSku())) {
            throw new \Magento\Framework\Exception\ValidatorException(
                __('SKU can only contain letters, numbers, hyphens, and underscores.')
            );
        }
    }
}
```

#### Step 4: Resource Model Save

```php
namespace Magento\Catalog\Model\ResourceModel;

class Product extends \Magento\Catalog\Model\ResourceModel\AbstractResource
{
    public function save(\Magento\Framework\Model\AbstractModel $object)
    {
        $connection = $this->getConnection();
        $connection->beginTransaction();

        try {
            // Save main entity
            parent::save($object);

            // Save website assignments
            $this->saveWebsiteIds($object);

            // Save category assignments
            $this->saveCategoryIds($object);

            // Save product links
            $this->saveProductLinks($object);

            // Save media gallery
            $this->saveMediaGallery($object);

            $connection->commit();
        } catch (\Exception $e) {
            $connection->rollBack();
            throw $e;
        }

        return $this;
    }

    protected function saveWebsiteIds($product): void
    {
        $websiteIds = $product->getWebsiteIds();
        $oldWebsiteIds = $this->getWebsiteIds($product);

        $insert = array_diff($websiteIds, $oldWebsiteIds);
        $delete = array_diff($oldWebsiteIds, $websiteIds);

        $connection = $this->getConnection();

        if ($delete) {
            $connection->delete(
                $this->getTable('catalog_product_website'),
                [
                    'product_id = ?' => $product->getId(),
                    'website_id IN (?)' => $delete
                ]
            );
        }

        if ($insert) {
            $data = [];
            foreach ($insert as $websiteId) {
                $data[] = ['product_id' => $product->getId(), 'website_id' => $websiteId];
            }
            $connection->insertMultiple($this->getTable('catalog_product_website'), $data);
        }
    }

    protected function saveCategoryIds($product): void
    {
        $categoryIds = $product->getCategoryIds();
        if ($categoryIds === null) {
            return;
        }

        $oldCategoryIds = $this->getCategoryIds($product);
        $insert = array_diff($categoryIds, $oldCategoryIds);
        $delete = array_diff($oldCategoryIds, $categoryIds);

        $connection = $this->getConnection();

        if ($delete) {
            $connection->delete(
                $this->getTable('catalog_category_product'),
                [
                    'product_id = ?' => $product->getId(),
                    'category_id IN (?)' => $delete
                ]
            );
        }

        if ($insert) {
            $data = [];
            $position = 0;
            foreach ($insert as $categoryId) {
                $data[] = [
                    'category_id' => $categoryId,
                    'product_id' => $product->getId(),
                    'position' => $position++
                ];
            }
            $connection->insertMultiple($this->getTable('catalog_category_product'), $data);
        }
    }
}
```

#### Step 5: Model afterSave Hook

```php
namespace Magento\Catalog\Model;

class Product extends \Magento\Catalog\Model\AbstractModel
{
    public function afterSave()
    {
        // Dispatch event for observers
        $this->_eventManager->dispatch(
            'catalog_product_save_after',
            ['product' => $this, 'data_object' => $this]
        );

        // Invalidate cache
        $this->cacheManager->clean($this->getIdentities());

        // Reindex
        $this->indexerProcessor->reindexRow($this->getId());

        return parent::afterSave();
    }

    public function getIdentities()
    {
        $identities = [
            self::CACHE_TAG . '_' . $this->getId(),
            self::CACHE_TAG . '_' . $this->getSku()
        ];

        // Add category cache tags
        foreach ($this->getCategoryIds() as $categoryId) {
            $identities[] = \Magento\Catalog\Model\Category::CACHE_TAG . '_' . $categoryId;
        }

        return $identities;
    }
}
```

#### Step 6: Stock Integration Observer

```php
namespace Magento\CatalogInventory\Observer;

class SaveInventoryDataObserver implements ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer)
    {
        $product = $observer->getEvent()->getProduct();

        if ($stockData = $product->getStockData()) {
            $this->stockItemRepository->save(
                $this->stockItemFactory->create()->setData($stockData)
            );
        }
    }
}
```

### Performance Considerations

1. **Transaction Scope**: Entire save wrapped in DB transaction - failure rolls back all changes
2. **Indexer Mode**: In `schedule` mode, reindex happens async via cron; `realtime` blocks save
3. **Cache Invalidation**: FPC tags cleared immediately, potentially causing cache stampede
4. **Plugin Chain**: Multiple plugins can execute - keep them lightweight
5. **Media Gallery**: Large images processed synchronously - consider queue for bulk operations

## 2. Category Tree Building Flow

### Flow Diagram (Text)

```
Category Navigation Block
    └─> Category\Tree::getTree()
        ├─> Load root category
        ├─> Build SQL query with MPTT (path field)
        │   ├─> SELECT WHERE path LIKE 'root_path/%'
        │   ├─> ORDER BY level, position
        │   └─> Filter by is_active = 1
        ├─> Execute query
        ├─> Build tree structure from flat result
        │   ├─> Create tree nodes
        │   ├─> Assign children to parents using path
        │   └─> Set product counts if anchor category
        └─> Return tree object
```

### Detailed Code Flow

#### Step 1: Frontend Block Rendering

```php
namespace Magento\Catalog\Block\Navigation;

class Navigation extends \Magento\Framework\View\Element\Template
{
    public function getHtml()
    {
        $storeId = $this->_storeManager->getStore()->getId();
        $rootCategoryId = $this->_storeManager->getStore()->getRootCategoryId();

        $category = $this->categoryRepository->get($rootCategoryId, $storeId);
        $tree = $this->getTree($category);

        return $this->_toHtml($tree);
    }

    private function getTree($parentCategory, $depth = 0)
    {
        $children = $this->categoryTreeResource->getChildren($parentCategory, true, false);

        $html = '<ul class="level-' . $depth . '">';
        foreach ($children as $child) {
            $html .= '<li class="category-item">';
            $html .= '<a href="' . $child->getUrl() . '">' . $child->getName() . '</a>';

            if ($child->hasChildren()) {
                $html .= $this->getTree($child, $depth + 1);
            }

            $html .= '</li>';
        }
        $html .= '</ul>';

        return $html;
    }
}
```

#### Step 2: Category Tree Resource

```php
namespace Magento\Catalog\Model\ResourceModel\Category;

class Tree extends \Magento\Catalog\Model\ResourceModel\Category
{
    public function getChildren($category, $recursive = true, $isActive = true)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from(['main_table' => $this->getEntityTable()], 'entity_id')
            ->where('main_table.path LIKE ?', $category->getPath() . '/%')
            ->order('main_table.level')
            ->order('main_table.position');

        if ($isActive) {
            $select->join(
                ['attr_is_active' => $this->getTable('catalog_category_entity_int')],
                'main_table.entity_id = attr_is_active.entity_id',
                []
            )
            ->where('attr_is_active.attribute_id = ?', $this->getIsActiveAttributeId())
            ->where('attr_is_active.value = ?', 1);
        }

        if (!$recursive) {
            $select->where('main_table.level = ?', $category->getLevel() + 1);
        }

        $categoryIds = $connection->fetchCol($select);

        // Load categories efficiently using collection
        $collection = $this->categoryCollectionFactory->create()
            ->addAttributeToSelect(['name', 'url_key', 'is_active', 'children_count'])
            ->addIdFilter($categoryIds)
            ->setOrder('level', 'ASC')
            ->setOrder('position', 'ASC');

        return $collection->getItems();
    }

    public function getTree($category, $levels = 3)
    {
        $storeId = $this->_storeManager->getStore()->getId();
        $connection = $this->getConnection();

        // Build nested set query
        $select = $connection->select()
            ->from(['e' => $this->getEntityTable()])
            ->where('e.path LIKE ?', $category->getPath() . '/%')
            ->where('e.level <= ?', $category->getLevel() + $levels)
            ->order('e.level')
            ->order('e.position');

        // Join name attribute
        $select->joinLeft(
            ['name_attr' => $this->getTable('catalog_category_entity_varchar')],
            'e.entity_id = name_attr.entity_id AND name_attr.attribute_id = ' . $this->getNameAttributeId()
                . ' AND name_attr.store_id = ' . $storeId,
            ['name' => 'name_attr.value']
        );

        // Join is_active attribute
        $select->joinLeft(
            ['active_attr' => $this->getTable('catalog_category_entity_int')],
            'e.entity_id = active_attr.entity_id AND active_attr.attribute_id = ' . $this->getIsActiveAttributeId()
                . ' AND active_attr.store_id = 0',
            ['is_active' => 'active_attr.value']
        );

        $select->where('COALESCE(active_attr.value, 1) = 1');

        $data = $connection->fetchAll($select);

        // Build tree from flat data
        return $this->buildTree($data, $category->getId());
    }

    private function buildTree(array $data, $parentId)
    {
        $tree = [];
        $indexed = [];

        // Index by ID
        foreach ($data as $row) {
            $indexed[$row['entity_id']] = $row;
            $indexed[$row['entity_id']]['children'] = [];
        }

        // Build parent-child relationships using path
        foreach ($indexed as $id => $row) {
            $pathIds = explode('/', $row['path']);
            $immediateParentId = $pathIds[count($pathIds) - 2] ?? null;

            if ($immediateParentId && isset($indexed[$immediateParentId])) {
                $indexed[$immediateParentId]['children'][] = &$indexed[$id];
            } elseif ($immediateParentId == $parentId) {
                $tree[] = &$indexed[$id];
            }
        }

        return $tree;
    }
}
```

#### Step 3: Anchor Category Product Count

```php
namespace Magento\Catalog\Model\ResourceModel\Category;

class Tree extends \Magento\Catalog\Model\ResourceModel\Category
{
    public function addProductCount($categories)
    {
        $categoryIds = array_keys($categories);
        $connection = $this->getConnection();

        // Get product counts from index table
        $select = $connection->select()
            ->from(
                ['index' => $this->getTable('catalog_category_product_index')],
                ['category_id', 'product_count' => 'COUNT(DISTINCT index.product_id)']
            )
            ->where('index.category_id IN (?)', $categoryIds)
            ->where('index.store_id = ?', $this->getStoreId())
            ->group('index.category_id');

        $counts = $connection->fetchPairs($select);

        foreach ($categories as $categoryId => $category) {
            if (isset($counts[$categoryId])) {
                $category->setProductCount($counts[$categoryId]);
            }

            // For anchor categories, include children product counts
            if ($category->getIsAnchor()) {
                $childrenIds = $this->getAllChildren($category);
                $childrenCounts = array_intersect_key($counts, array_flip(explode(',', $childrenIds)));
                $totalCount = array_sum($childrenCounts);
                $category->setProductCount($totalCount);
            }
        }

        return $this;
    }

    public function getAllChildren($category)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getEntityTable(), 'entity_id')
            ->where('path LIKE ?', $category->getPath() . '/%');

        return implode(',', $connection->fetchCol($select));
    }
}
```

### Performance Optimization

```php
namespace Vendor\Module\Model\ResourceModel\Category;

class TreeOptimized extends \Magento\Catalog\Model\ResourceModel\Category\Tree
{
    /**
     * Optimized tree loading with single query and in-memory tree building
     */
    public function getOptimizedTree($rootCategory, $maxDepth = 3)
    {
        $cacheKey = 'category_tree_' . $rootCategory->getId() . '_' . $maxDepth;

        // Check cache first
        if ($cached = $this->cache->load($cacheKey)) {
            return $this->serializer->unserialize($cached);
        }

        $connection = $this->getConnection();
        $storeId = $this->_storeManager->getStore()->getId();

        // Single optimized query
        $select = $connection->select()
            ->from(['e' => $this->getEntityTable()], ['entity_id', 'parent_id', 'path', 'level', 'position'])
            ->where('e.path LIKE ?', $rootCategory->getPath() . '%')
            ->where('e.level <= ?', $rootCategory->getLevel() + $maxDepth);

        // Join all needed attributes in one go
        $this->joinAttribute($select, 'name', 'varchar', $storeId);
        $this->joinAttribute($select, 'is_active', 'int', 0);
        $this->joinAttribute($select, 'url_key', 'varchar', $storeId);
        $this->joinAttribute($select, 'include_in_menu', 'int', 0);

        $select->where('COALESCE(is_active.value, 1) = 1')
            ->where('COALESCE(include_in_menu.value, 1) = 1')
            ->order('e.level ASC')
            ->order('e.position ASC');

        $rows = $connection->fetchAll($select);

        // Build tree in memory (O(n) complexity)
        $tree = $this->buildTreeFromRows($rows, $rootCategory->getId());

        // Cache for 1 hour
        $this->cache->save(
            $this->serializer->serialize($tree),
            $cacheKey,
            [\Magento\Catalog\Model\Category::CACHE_TAG],
            3600
        );

        return $tree;
    }

    private function joinAttribute($select, $alias, $type, $storeId)
    {
        $attributeId = $this->getAttribute($alias)->getId();
        $table = $this->getTable('catalog_category_entity_' . $type);

        $select->joinLeft(
            [$alias => $table],
            sprintf(
                'e.entity_id = %s.entity_id AND %s.attribute_id = %d AND %s.store_id = %d',
                $alias,
                $alias,
                $attributeId,
                $alias,
                $storeId
            ),
            [$alias => $alias . '.value']
        );
    }
}
```

## 3. Price Calculation Flow

### Flow Diagram (Text)

```
Product::getFinalPrice()
    └─> PriceInfo::getPrice('final_price')
        ├─> FinalPrice::getAmount()
        │   ├─> Calculate base price
        │   ├─> Apply special price if active
        │   ├─> Apply tier price if applicable
        │   ├─> Apply catalog price rules
        │   │   └─> Check rule conditions
        │   │       ├─> Customer group match
        │   │       ├─> Date range valid
        │   │       ├─> Website match
        │   │       └─> Calculate discount
        │   ├─> Apply custom price modifiers (plugins)
        │   └─> Return final calculated price
        └─> Return Amount object
```

### Detailed Code Flow

#### Step 1: Frontend Price Rendering

```php
namespace Magento\Catalog\Block\Product;

class View extends \Magento\Catalog\Block\Product\AbstractProduct
{
    public function getProductPrice($product)
    {
        $priceRender = $this->getLayout()->getBlock('product.price.render.default');

        if (!$priceRender) {
            return '';
        }

        return $priceRender->render(
            \Magento\Catalog\Pricing\Price\FinalPrice::PRICE_CODE,
            $product,
            [
                'display_minimal_price' => true,
                'use_link_for_as_low_as' => true,
                'zone' => \Magento\Framework\Pricing\Render::ZONE_ITEM_VIEW
            ]
        );
    }
}
```

#### Step 2: Price Info Initialization

```php
namespace Magento\Catalog\Model\Product;

class Type\Price
{
    public function getFinalPrice($qty, $product)
    {
        if ($qty === null && $product->getCalculatedFinalPrice() !== null) {
            return $product->getCalculatedFinalPrice();
        }

        // Get base price
        $finalPrice = $this->getBasePrice($product, $qty);

        // Hook for custom price modifications
        $product->setFinalPrice($finalPrice);

        // Dispatch event for observers
        $this->_eventManager->dispatch(
            'catalog_product_get_final_price',
            ['product' => $product, 'qty' => $qty]
        );

        $finalPrice = $product->getData('final_price');

        // Ensure price is not negative
        $finalPrice = max(0, $finalPrice);

        $product->setFinalPrice($finalPrice);
        $product->setCalculatedFinalPrice($finalPrice);

        return $finalPrice;
    }

    public function getBasePrice($product, $qty = null)
    {
        $price = (float) $product->getPrice();
        return $this->applySpecialPrice($product, $price);
    }

    protected function applySpecialPrice($product, $price)
    {
        $specialPrice = $product->getSpecialPrice();

        if ($specialPrice !== null && $specialPrice !== false) {
            $specialPriceFrom = $product->getSpecialFromDate();
            $specialPriceTo = $product->getSpecialToDate();

            if ($this->isValidSpecialPriceDate($specialPriceFrom, $specialPriceTo)) {
                $price = min($price, (float) $specialPrice);
            }
        }

        return $price;
    }

    private function isValidSpecialPriceDate($from, $to): bool
    {
        $now = $this->dateTime->gmtTimestamp();

        if ($from && $to) {
            return $now >= strtotime($from) && $now <= strtotime($to);
        } elseif ($from) {
            return $now >= strtotime($from);
        } elseif ($to) {
            return $now <= strtotime($to);
        }

        return true;
    }
}
```

#### Step 3: Final Price Calculation

```php
namespace Magento\Catalog\Pricing\Price;

class FinalPrice extends \Magento\Framework\Pricing\Price\AbstractPrice
{
    const PRICE_CODE = 'final_price';

    public function getValue()
    {
        if ($this->value === null) {
            // Start with regular price
            $price = $this->getBasePrice()->getValue();

            // Apply tier price discount
            $tierPrice = $this->tierPriceModel->getValue();
            if ($tierPrice !== null) {
                $price = min($price, $tierPrice);
            }

            // Apply special price
            $specialPrice = $this->specialPriceModel->getValue();
            if ($specialPrice !== false && $specialPrice < $price) {
                $price = $specialPrice;
            }

            // Apply catalog price rules
            $catalogRulePrice = $this->catalogRulePrice->getValue();
            if ($catalogRulePrice !== false && $catalogRulePrice < $price) {
                $price = $catalogRulePrice;
            }

            // Apply custom adjustments (via plugins/observers)
            $price = $this->applyAdjustments($price);

            $this->value = $price;
        }

        return max(0, $this->value);
    }

    private function applyAdjustments($price)
    {
        // Get all price adjustment modifiers
        foreach ($this->priceModifiers as $modifier) {
            $price = $modifier->modifyPrice($price, $this->product);
        }

        return $price;
    }
}
```

#### Step 4: Tier Price Calculation

```php
namespace Magento\Catalog\Pricing\Price;

class TierPrice extends \Magento\Framework\Pricing\Price\AbstractPrice
{
    public function getValue()
    {
        if ($this->value === null) {
            $prices = $this->getStoredTierPrices();
            $qty = $this->getQuantity() ?: 1;

            $applicablePrice = null;
            $matchedQty = 0;

            foreach ($prices as $tierPrice) {
                $tierQty = $tierPrice['price_qty'];

                // Find the highest tier that applies to current quantity
                if ($qty >= $tierQty && $tierQty > $matchedQty) {
                    $matchedQty = $tierQty;
                    $applicablePrice = $this->calculateTierPrice($tierPrice);
                }
            }

            $this->value = $applicablePrice;
        }

        return $this->value;
    }

    private function calculateTierPrice(array $tierPrice): float
    {
        $basePrice = $this->getBasePrice()->getValue();

        if ($tierPrice['website_id'] && $tierPrice['website_id'] != $this->getWebsiteId()) {
            return $basePrice;
        }

        if ($tierPrice['customer_group_id'] && !$this->matchesCustomerGroup($tierPrice['customer_group_id'])) {
            return $basePrice;
        }

        // Tier price can be percentage discount or fixed price
        if (isset($tierPrice['percentage_value'])) {
            return $basePrice * (1 - $tierPrice['percentage_value'] / 100);
        }

        return (float) $tierPrice['value'];
    }

    private function getStoredTierPrices(): array
    {
        $tierPrices = $this->product->getData('tier_price');

        if ($tierPrices === null) {
            $tierPrices = $this->tierPriceStorage->get($this->product);
            $this->product->setData('tier_price', $tierPrices);
        }

        return $tierPrices ?? [];
    }
}
```

#### Step 5: Catalog Rule Price

```php
namespace Magento\CatalogRule\Pricing\Price;

class CatalogRulePrice extends \Magento\Framework\Pricing\Price\AbstractPrice
{
    public function getValue()
    {
        if ($this->value === null) {
            $this->value = false;

            // Check if catalog rule applies
            $rulePrice = $this->catalogRuleResourceModel->getRulePrice(
                $this->dateTime->scopeDate($this->storeManager->getStore()->getId()),
                $this->storeManager->getStore()->getWebsiteId(),
                $this->customerSession->getCustomerGroupId(),
                $this->product->getId()
            );

            if ($rulePrice !== false) {
                $this->value = (float) $rulePrice;
            }
        }

        return $this->value;
    }
}
```

#### Step 6: Price Index Lookup (Faster Alternative)

```php
namespace Magento\Catalog\Model\ResourceModel\Product;

class Price
{
    /**
     * Get indexed final price (faster, pre-calculated)
     */
    public function getFinalPrice($product, $qty = 1)
    {
        $connection = $this->getConnection();

        $select = $connection->select()
            ->from($this->getTable('catalog_product_index_price'), 'final_price')
            ->where('entity_id = ?', $product->getId())
            ->where('customer_group_id = ?', $this->customerSession->getCustomerGroupId())
            ->where('website_id = ?', $this->storeManager->getStore()->getWebsiteId());

        $price = $connection->fetchOne($select);

        return $price !== false ? (float) $price : $product->getPrice();
    }

    /**
     * Get tier price for quantity
     */
    public function getTierPrice($product, $qty)
    {
        $connection = $this->getConnection();

        $select = $connection->select()
            ->from($this->getTable('catalog_product_index_tier_price'), 'min_price')
            ->where('entity_id = ?', $product->getId())
            ->where('customer_group_id = ?', $this->customerSession->getCustomerGroupId())
            ->where('website_id = ?', $this->storeManager->getStore()->getWebsiteId())
            ->where('qty <= ?', $qty)
            ->order('qty DESC')
            ->limit(1);

        return $connection->fetchOne($select);
    }
}
```

### Performance Optimization

```php
namespace Vendor\Module\Plugin\Pricing;

class PriceCachePlugin
{
    public function __construct(
        private readonly \Magento\Framework\App\CacheInterface $cache,
        private readonly \Magento\Framework\Serialize\SerializerInterface $serializer,
        private readonly \Magento\Customer\Model\Session $customerSession
    ) {}

    /**
     * Cache calculated prices to reduce repeated calculations
     */
    public function aroundGetValue(
        \Magento\Catalog\Pricing\Price\FinalPrice $subject,
        callable $proceed
    ) {
        $product = $subject->getProduct();
        $cacheKey = $this->getCacheKey($product);

        if ($cached = $this->cache->load($cacheKey)) {
            return (float) $cached;
        }

        $price = $proceed();

        // Cache for current request only (short TTL)
        $this->cache->save($price, $cacheKey, ['catalog_product_' . $product->getId()], 300);

        return $price;
    }

    private function getCacheKey($product): string
    {
        return sprintf(
            'product_final_price_%d_%d_%d',
            $product->getId(),
            $this->customerSession->getCustomerGroupId(),
            $product->getStoreId()
        );
    }
}
```

## 4. Product Collection Loading Flow

### Optimized Collection Loading

```php
namespace Magento\Catalog\Model\ResourceModel\Product;

class Collection extends \Magento\Catalog\Model\ResourceModel\Collection\AbstractCollection
{
    /**
     * Optimized loading with only required attributes
     */
    public function load($printQuery = false, $logQuery = false)
    {
        if ($this->isLoaded()) {
            return $this;
        }

        // Add minimal attributes for listing page
        $this->addAttributeToSelect(['name', 'sku', 'price', 'small_image', 'url_key']);

        // Filter by stock
        $this->joinField(
            'is_in_stock',
            'cataloginventory_stock_status',
            'stock_status',
            'product_id=entity_id',
            ['stock_status' => 1]
        );

        // Use price index
        $this->addPriceData();

        // Load efficiently
        parent::load($printQuery, $logQuery);

        return $this;
    }

    public function addPriceData($customerGroupId = null, $websiteId = null)
    {
        if ($customerGroupId === null) {
            $customerGroupId = $this->customerSession->getCustomerGroupId();
        }

        if ($websiteId === null) {
            $websiteId = $this->storeManager->getStore()->getWebsiteId();
        }

        $this->getSelect()->joinLeft(
            ['price_index' => $this->getTable('catalog_product_index_price')],
            'e.entity_id = price_index.entity_id'
                . ' AND price_index.customer_group_id = ' . $customerGroupId
                . ' AND price_index.website_id = ' . $websiteId,
            ['final_price', 'min_price', 'max_price']
        );

        return $this;
    }
}
```

---

**Assumptions:**
- Magento 2.4.7+ with catalog price rules enabled
- MSI for inventory (CatalogInventory deprecated)
- Customer sessions active for group-based pricing
- Indexers running in schedule mode

**Why this approach:**
Step-by-step flow documentation aids debugging and customization. Price calculation leverages index tables for performance. Category tree uses MPTT for efficient queries.

**Security impact:**
Price calculation respects customer group assignments (ACL). Admin save operations validate permissions via ACL resources.

**Performance impact:**
Product save: O(1) for main entity, O(n) for EAV attributes. Category tree: O(n) with single query using MPTT. Price calculation: O(1) with index tables, O(n) without.

**Backward compatibility:**
All flows use service contracts and events. Plugin points on repositories ensure safe extension.

**Tests to add:**
- Integration tests for complete save flow with assertions on DB state
- Unit tests for price calculation with mocked dependencies
- Functional tests for category tree rendering
- Performance tests for collection loading with large datasets

**Docs to update:**
- EXECUTION_FLOWS.md (this file) when adding new complex flows
- Sequence diagrams for visual representation
- Add caching strategies to PERFORMANCE.md
