---
title: "Service Contracts vs Repositories in Magento 2"
description: "Developer guide: Service Contracts vs Repositories in Magento 2"
type: "explanation"
tier: 1
difficulty: "intermediate"
version: "2.4.7+"
estimated_time: "18 minutes"
topics:
  - service-contracts
  - repositories
  - api-design
  - backward-compatibility
  - upgrade-safety
last_updated: "2026-02-07"
---

# Service Contracts vs Repositories in Magento 2

## Introduction

One of the most critical architectural decisions in Magento 2 was the introduction of **service contracts**—stable, versioned APIs marked with `@api` annotations. This shift from direct model manipulation to repository-mediated persistence fundamentally changed how we build upgrade-safe, extensible Magento modules.

Understanding the distinction between service contracts and repositories—and when to use each—is essential for any Magento developer building production-ready extensions. This guide explores the conceptual foundations, architectural trade-offs, and practical patterns that govern these core abstractions.

### What You'll Learn

- The purpose and guarantees of `@api` service contracts
- How the Repository pattern abstracts persistence
- When to use repositories vs direct model access
- SearchCriteriaBuilder for complex filtering
- Extension attributes vs custom attributes
- API stability and backward compatibility guarantees
- Real-world examples from core modules
- Design trade-offs and antipatterns to avoid

---

## What Are Service Contracts?

### Definition

A **service contract** in Magento 2 is a set of PHP interfaces in the `Api` namespace of a module that define stable public APIs. These interfaces are marked with the `@api` annotation and carry a promise: **the method signatures will remain backward-compatible across minor and patch releases**.

```php
/**
 * Customer repository interface.
 *
 * @api
 * @since 100.0.2
 */
interface CustomerRepositoryInterface
{
    /**
     * Retrieve customer.
     *
     * @param int $customerId
     * @return \Magento\Customer\Api\Data\CustomerInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function getById($customerId);

    /**
     * Create or update a customer.
     *
     * @param \Magento\Customer\Api\Data\CustomerInterface $customer
     * @param string $passwordHash
     * @return \Magento\Customer\Api\Data\CustomerInterface
     * @throws \Magento\Framework\Exception\InputException
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function save(
        \Magento\Customer\Api\Data\CustomerInterface $customer,
        $passwordHash = null
    );

    /**
     * Retrieve customers matching specified search criteria.
     *
     * @param \Magento\Framework\Api\SearchCriteriaInterface $searchCriteria
     * @return \Magento\Customer\Api\Data\CustomerSearchResultsInterface
     */
    public function getList(
        \Magento\Framework\Api\SearchCriteriaInterface $searchCriteria
    );

    /**
     * Delete customer.
     *
     * @param \Magento\Customer\Api\Data\CustomerInterface $customer
     * @return bool
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function delete(\Magento\Customer\Api\Data\CustomerInterface $customer);
}
```

### Key Characteristics

1. **Backward Compatibility (BC) Policy**: Method signatures cannot change in ways that break existing calls (adding required parameters, changing return types, removing methods).

2. **Versioning via @since**: The `@since` docblock tag indicates when the interface was introduced, enabling tracking across releases.

3. **Data Interfaces (DTOs)**: Service contracts accept and return **Data Interfaces**—immutable value objects that represent entities without behavior.

4. **Exception Contracts**: Well-defined exception hierarchy (`NoSuchEntityException`, `InputException`, `LocalizedException`) ensures predictable error handling.

5. **REST/SOAP/GraphQL Exposure**: `@api` interfaces can be automatically exposed as web services via `webapi.xml`.

### Why Service Contracts Matter

#### Before Service Contracts (Magento 1.x)

```php
// Magento 1 approach - fragile, upgrade-unsafe
$customer = Mage::getModel('customer/customer')->load($customerId);
$customer->setFirstname('John');
$customer->save();
```

**Problems:**
- Direct coupling to implementation (Models, ResourceModels)
- Methods could be added/removed/changed between versions
- No clear API surface for third-party modules or web services
- Difficult to mock for testing
- Plugins couldn't reliably intercept persistence logic

#### With Service Contracts (Magento 2)

```php
// Magento 2 approach - stable, upgrade-safe
public function __construct(
    \Magento\Customer\Api\CustomerRepositoryInterface $customerRepository
) {
    $this->customerRepository = $customerRepository;
}

public function updateCustomer(int $customerId): void
{
    $customer = $this->customerRepository->getById($customerId);
    $customer->setFirstname('John');
    $this->customerRepository->save($customer);
}
```

**Benefits:**
- Guaranteed stability: method signatures won't break
- Testable: inject mocks via DI
- Pluginable: intercept `save()`, `getById()` with plugins
- Web API ready: expose via REST/GraphQL without additional code
- Clear separation: business logic vs persistence vs presentation

---

## The Repository Pattern in Magento 2

### Pattern Overview

The **Repository pattern** mediates between the domain layer (business logic) and data mapping layers (ResourceModels, Collections). It provides a **collection-like interface** for accessing domain objects while encapsulating storage details.

```
┌─────────────────────────────────────────────────────────┐
│                    Service Layer                        │
│              (Business Logic / Use Cases)               │
└────────────────────┬────────────────────────────────────┘
                     │ depends on
                     ▼
┌─────────────────────────────────────────────────────────┐
│             Repository Interface (@api)                 │
│  - getById(int $id)                                     │
│  - save(DataInterface $entity)                          │
│  - delete(DataInterface $entity)                        │
│  - getList(SearchCriteriaInterface $criteria)           │
└────────────────────┬────────────────────────────────────┘
                     │ implemented by
                     ▼
┌─────────────────────────────────────────────────────────┐
│          Repository Implementation (Model/)             │
│  - Coordinates ResourceModel, Factory, SearchResults    │
│  - Handles caching, validation, events                  │
│  - Maps between Models and Data Interfaces              │
└────────────────────┬────────────────────────────────────┘
                     │ uses
                     ▼
┌─────────────────────────────────────────────────────────┐
│            ResourceModel / Collection                   │
│         (Database Abstraction Layer)                    │
└─────────────────────────────────────────────────────────┘
```

### Core Repository Methods

Every Magento repository follows a consistent pattern:

| Method | Purpose | Returns |
|--------|---------|---------|
| `getById(int $id)` | Fetch single entity by primary key | `DataInterface` |
| `save(DataInterface $entity)` | Create or update entity | `DataInterface` |
| `delete(DataInterface $entity)` | Remove entity | `bool` |
| `deleteById(int $id)` | Remove entity by ID | `bool` |
| `getList(SearchCriteriaInterface $criteria)` | Fetch multiple entities with filters, sorting, pagination | `SearchResultsInterface` |

### Example: ProductRepository

```php
namespace Magento\Catalog\Api;

/**
 * @api
 */
interface ProductRepositoryInterface
{
    /**
     * @param \Magento\Catalog\Api\Data\ProductInterface $product
     * @param bool $saveOptions
     * @return \Magento\Catalog\Api\Data\ProductInterface
     * @throws \Magento\Framework\Exception\InputException
     * @throws \Magento\Framework\Exception\StateException
     * @throws \Magento\Framework\Exception\CouldNotSaveException
     */
    public function save(
        \Magento\Catalog\Api\Data\ProductInterface $product,
        $saveOptions = false
    );

    /**
     * @param string $sku
     * @param bool $editMode
     * @param int|null $storeId
     * @param bool $forceReload
     * @return \Magento\Catalog\Api\Data\ProductInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function get($sku, $editMode = false, $storeId = null, $forceReload = false);

    /**
     * @param int $productId
     * @param bool $editMode
     * @param int|null $storeId
     * @param bool $forceReload
     * @return \Magento\Catalog\Api\Data\ProductInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function getById($productId, $editMode = false, $storeId = null, $forceReload = false);

    /**
     * @param \Magento\Framework\Api\SearchCriteriaInterface $searchCriteria
     * @return \Magento\Catalog\Api\Data\ProductSearchResultsInterface
     */
    public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria);

    /**
     * @param \Magento\Catalog\Api\Data\ProductInterface $product
     * @return bool
     * @throws \Magento\Framework\Exception\StateException
     */
    public function delete(\Magento\Catalog\Api\Data\ProductInterface $product);

    /**
     * @param string $sku
     * @return bool
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     * @throws \Magento\Framework\Exception\StateException
     */
    public function deleteById($sku);
}
```

**Key Details:**
- `get($sku)` vs `getById($productId)`: Products have composite identities
- `$editMode`, `$storeId`, `$forceReload`: control caching and scope
- `save()` returns the updated entity (with generated IDs, timestamps)

---

## When to Use Repositories vs Direct Model Access

### Use Repositories When:

#### 1. Building for Upgrade Safety
```php
// RIGHT: Upgrade-safe, BC-guaranteed
public function __construct(
    \Magento\Catalog\Api\ProductRepositoryInterface $productRepository
) {
    $this->productRepository = $productRepository;
}

public function execute(string $sku): void
{
    $product = $this->productRepository->get($sku);
    // Business logic
}
```

**Why:** The `@api` interface guarantees the method signature won't change. Your code will survive Magento upgrades from 2.4.7 → 2.4.8 → 2.4.9.

#### 2. Exposing Business Logic as Web APIs
```xml
<!-- webapi.xml -->
<route url="/V1/products/:sku" method="GET">
    <service class="Magento\Catalog\Api\ProductRepositoryInterface" method="get"/>
    <resources>
        <resource ref="Magento_Catalog::products"/>
    </resources>
</route>
```

Service contracts map directly to REST/SOAP/GraphQL endpoints with zero additional code.

#### 3. Writing Testable Code
```php
// Unit test - mock the repository
public function testUpdateProduct(): void
{
    $productMock = $this->createMock(\Magento\Catalog\Api\Data\ProductInterface::class);
    $repositoryMock = $this->createMock(\Magento\Catalog\Api\ProductRepositoryInterface::class);

    $repositoryMock->expects($this->once())
        ->method('get')
        ->with('TEST-SKU')
        ->willReturn($productMock);

    $service = new MyProductService($repositoryMock);
    $service->execute('TEST-SKU');
}
```

Mocking interfaces is trivial; mocking concrete Models with 50 dependencies is not.

#### 4. Enabling Plugin Interception
```php
// Plugin to log every product save
class LogProductSave
{
    public function afterSave(
        \Magento\Catalog\Api\ProductRepositoryInterface $subject,
        \Magento\Catalog\Api\Data\ProductInterface $result
    ): \Magento\Catalog\Api\Data\ProductInterface {
        $this->logger->info('Product saved: ' . $result->getSku());
        return $result;
    }
}
```

Plugins on repository methods give you centralized interception points; plugins on Models are fragmented.

### Use Direct Model Access When:

#### 1. Working in Admin Grids or Collections (Performance)
```php
// Admin product grid - use collection for efficiency
$collection = $this->collectionFactory->create();
$collection->addAttributeToSelect(['name', 'sku', 'price']);
$collection->addFieldToFilter('status', ['eq' => Status::STATUS_ENABLED]);
$collection->setPageSize(20);
```

**Why:** `getList()` with SearchCriteria is convenient but adds overhead. For admin grids where you need raw DB control (joins, custom columns), collections are still appropriate.

#### 2. Bulk Operations
```php
// Bulk price update - direct ResourceModel
$connection = $this->resourceConnection->getConnection();
$connection->update(
    $this->resourceConnection->getTableName('catalog_product_entity_decimal'),
    ['value' => new \Zend_Db_Expr('value * 1.10')],
    ['attribute_id = ?' => $priceAttributeId]
);
```

**Why:** Repository `save()` calls trigger events, validation, indexing. For bulk updates, direct SQL is orders of magnitude faster.

#### 3. Custom Queries Not Supported by SearchCriteria
```php
// Complex join not expressible in SearchCriteria
$collection = $this->collectionFactory->create();
$collection->getSelect()
    ->joinLeft(
        ['sales' => $this->resourceConnection->getTableName('sales_order_item')],
        'e.entity_id = sales.product_id',
        ['total_sold' => 'SUM(sales.qty_ordered)']
    )
    ->group('e.entity_id')
    ->having('total_sold > ?', 100);
```

**Why:** SearchCriteria supports filters, sorting, pagination—but not arbitrary SQL. Use collections when you need joins, subqueries, or aggregates.

### Decision Matrix

| Scenario | Use Repository | Use Model/Collection |
|----------|----------------|----------------------|
| CRUD operations in business logic | ✓ | |
| Web API endpoints | ✓ | |
| Unit testable code | ✓ | |
| Plugin interception needed | ✓ | |
| Admin grid data providers | | ✓ |
| Bulk operations (1000+ records) | | ✓ |
| Complex SQL joins/aggregates | | ✓ |
| Internal module utilities | | ✓ (acceptable) |
| Third-party extension public API | ✓ (required) | |

---

## SearchCriteriaBuilder and Advanced Filtering

### The SearchCriteria Pattern

`getList()` methods accept a `SearchCriteriaInterface` that encapsulates:
- **Filters**: field conditions (equals, like, in, gt, etc.)
- **FilterGroups**: logical OR groups (filters within a group are AND)
- **SortOrders**: ascending/descending sort
- **Pagination**: page size and current page

```php
namespace Magento\Framework\Api;

interface SearchCriteriaInterface
{
    public function getFilterGroups();
    public function getSortOrders();
    public function getPageSize();
    public function getCurrentPage();
}
```

### Building Search Criteria

```php
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Api\SortOrderBuilder;

public function __construct(
    \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
    SearchCriteriaBuilder $searchCriteriaBuilder,
    SortOrderBuilder $sortOrderBuilder
) {
    $this->productRepository = $productRepository;
    $this->searchCriteriaBuilder = $searchCriteriaBuilder;
    $this->sortOrderBuilder = $sortOrderBuilder;
}

public function getActiveProductsByCategory(int $categoryId): array
{
    // Filters are AND within the same addFilters() call
    $this->searchCriteriaBuilder->addFilter('status', Status::STATUS_ENABLED);
    $this->searchCriteriaBuilder->addFilter('visibility', [
        Visibility::VISIBILITY_IN_CATALOG,
        Visibility::VISIBILITY_BOTH
    ], 'in');
    $this->searchCriteriaBuilder->addFilter('category_id', $categoryId);

    // Sort by name ascending
    $sortOrder = $this->sortOrderBuilder
        ->setField('name')
        ->setDirection(SortOrder::SORT_ASC)
        ->create();
    $this->searchCriteriaBuilder->setSortOrders([$sortOrder]);

    // Pagination
    $this->searchCriteriaBuilder->setPageSize(20);
    $this->searchCriteriaBuilder->setCurrentPage(1);

    $searchCriteria = $this->searchCriteriaBuilder->create();
    $searchResults = $this->productRepository->getList($searchCriteria);

    return $searchResults->getItems();
}
```

### OR Logic with Filter Groups

SearchCriteria uses **FilterGroups** for OR logic:
- Filters in the **same group** are OR'd
- **Different groups** are AND'd

```php
// Find products: (status = enabled) AND (price < 50 OR price > 1000)

// Group 1: status = enabled
$this->searchCriteriaBuilder->addFilter('status', Status::STATUS_ENABLED);

// Group 2: price < 50 OR price > 1000
$lowPriceFilter = $this->filterBuilder
    ->setField('price')
    ->setValue(50)
    ->setConditionType('lt')
    ->create();

$highPriceFilter = $this->filterBuilder
    ->setField('price')
    ->setValue(1000)
    ->setConditionType('gt')
    ->create();

$filterGroup = $this->filterGroupBuilder
    ->addFilter($lowPriceFilter)
    ->addFilter($highPriceFilter)
    ->create();

$this->searchCriteriaBuilder->setFilterGroups([$filterGroup]);

$searchCriteria = $this->searchCriteriaBuilder->create();
$searchResults = $this->productRepository->getList($searchCriteria);
```

### Condition Types

| Condition | SQL Equivalent | Example |
|-----------|----------------|---------|
| `eq` | `=` | `['field' => 'value']` |
| `neq` | `!=` | `['field' => 'value', 'condition_type' => 'neq']` |
| `like` | `LIKE` | `['field' => '%value%', 'condition_type' => 'like']` |
| `in` | `IN` | `['field' => [1, 2, 3], 'condition_type' => 'in']` |
| `nin` | `NOT IN` | `['field' => [1, 2, 3], 'condition_type' => 'nin']` |
| `gt` | `>` | `['field' => 100, 'condition_type' => 'gt']` |
| `lt` | `<` | `['field' => 100, 'condition_type' => 'lt']` |
| `gteq` | `>=` | `['field' => 100, 'condition_type' => 'gteq']` |
| `lteq` | `<=` | `['field' => 100, 'condition_type' => 'lteq']` |
| `null` | `IS NULL` | `['field' => null, 'condition_type' => 'null']` |
| `notnull` | `IS NOT NULL` | `['field' => null, 'condition_type' => 'notnull']` |

### Working with SearchResults

```php
$searchResults = $this->productRepository->getList($searchCriteria);

// Iterate items
foreach ($searchResults->getItems() as $product) {
    echo $product->getSku() . PHP_EOL;
}

// Total count (before pagination)
$totalCount = $searchResults->getTotalCount();

// Pagination info
$pageSize = $searchCriteria->getPageSize();
$currentPage = $searchCriteria->getCurrentPage();
$totalPages = ceil($totalCount / $pageSize);
```

---

## Extension Attributes vs Custom Attributes

Magento provides two mechanisms for extending entities without modifying core tables: **extension attributes** and **custom attributes** (EAV).

### Extension Attributes

**Purpose:** Add **structured, typed fields** to Data Interfaces for module integrations.

**Characteristics:**
- Defined in `extension_attributes.xml`
- Strongly typed (scalar, array, or another Data Interface)
- Populated via plugins on repositories
- Persisted in separate tables (custom logic required)
- Best for **module-to-module** contracts (e.g., inventory data on products)

#### Defining Extension Attributes

```xml
<!-- VendorName/ModuleName/etc/extension_attributes.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/extension_attributes.xsd">
    <extension_attributes for="Magento\Catalog\Api\Data\ProductInterface">
        <attribute code="warranty_info" type="VendorName\ModuleName\Api\Data\WarrantyInterface"/>
        <attribute code="supplier_id" type="int"/>
    </extension_attributes>
</config>
```

#### Populating Extension Attributes

```php
namespace VendorName\ModuleName\Plugin;

class LoadWarrantyExtensionAttribute
{
    public function __construct(
        private \VendorName\ModuleName\Model\WarrantyRepository $warrantyRepository,
        private \VendorName\ModuleName\Api\Data\WarrantyInterfaceFactory $warrantyFactory,
        private \Magento\Catalog\Api\Data\ProductExtensionFactory $extensionFactory
    ) {}

    public function afterGet(
        \Magento\Catalog\Api\ProductRepositoryInterface $subject,
        \Magento\Catalog\Api\Data\ProductInterface $product
    ): \Magento\Catalog\Api\Data\ProductInterface {
        $extensionAttributes = $product->getExtensionAttributes()
            ?? $this->extensionFactory->create();

        $warranty = $this->warrantyRepository->getByProductId($product->getId());
        $extensionAttributes->setWarrantyInfo($warranty);

        $product->setExtensionAttributes($extensionAttributes);
        return $product;
    }

    public function afterGetList(
        \Magento\Catalog\Api\ProductRepositoryInterface $subject,
        \Magento\Catalog\Api\Data\ProductSearchResultsInterface $searchResults
    ): \Magento\Catalog\Api\Data\ProductSearchResultsInterface {
        foreach ($searchResults->getItems() as $product) {
            $this->afterGet($subject, $product);
        }
        return $searchResults;
    }
}
```

#### Using Extension Attributes

```php
$product = $this->productRepository->get('TEST-SKU');
$warranty = $product->getExtensionAttributes()->getWarrantyInfo();
echo $warranty->getDurationMonths();
```

### Custom Attributes (EAV)

**Purpose:** Add **dynamic, user-defined fields** to EAV entities (products, customers, categories).

**Characteristics:**
- Defined via setup scripts or admin UI
- Loosely typed (stored as varchar, int, decimal, text)
- Automatically included in Data Interfaces under `custom_attributes`
- Persisted in EAV tables (`catalog_product_entity_varchar`, etc.)
- Best for **merchant-configurable** fields (product features, customer preferences)

#### Creating Custom Attributes

```php
// Setup/Patch/Data/AddProductManufacturerAttribute.php
namespace VendorName\ModuleName\Setup\Patch\Data;

use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddProductManufacturerAttribute implements DataPatchInterface
{
    public function __construct(
        private ModuleDataSetupInterface $moduleDataSetup,
        private EavSetupFactory $eavSetupFactory
    ) {}

    public function apply()
    {
        $eavSetup = $this->eavSetupFactory->create(['setup' => $this->moduleDataSetup]);

        $eavSetup->addAttribute(
            \Magento\Catalog\Model\Product::ENTITY,
            'manufacturer_country',
            [
                'type' => 'varchar',
                'label' => 'Manufacturer Country',
                'input' => 'select',
                'source' => \Magento\Eav\Model\Entity\Attribute\Source\Table::class,
                'required' => false,
                'sort_order' => 100,
                'global' => \Magento\Eav\Model\Entity\Attribute\ScopedAttributeInterface::SCOPE_GLOBAL,
                'used_in_product_listing' => true,
                'visible_on_front' => true,
                'option' => [
                    'values' => ['USA', 'Germany', 'China', 'Japan']
                ]
            ]
        );
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

#### Using Custom Attributes

```php
// Via Data Interface
$product = $this->productRepository->get('TEST-SKU');
$country = $product->getCustomAttribute('manufacturer_country');
echo $country ? $country->getValue() : 'N/A';

// Setting custom attributes
$product->setCustomAttribute('manufacturer_country', 'Germany');
$this->productRepository->save($product);

// Via Model (convenience methods)
$product = $this->productRepository->get('TEST-SKU');
echo $product->getData('manufacturer_country');
```

### When to Use Which?

| Use Case | Extension Attributes | Custom Attributes |
|----------|----------------------|-------------------|
| Merchant-configurable fields | | ✓ |
| Module integration data | ✓ | |
| Strongly typed (objects, arrays) | ✓ | |
| Searchable in admin grids | | ✓ (with config) |
| Exposed in GraphQL automatically | ✓ | ✓ |
| Requires custom persistence logic | ✓ (via plugins) | (automatic) |
| Visible in admin product form | Manual | ✓ (via attribute config) |

---

## API Stability and Backward Compatibility

### Magento's BC Policy

Magento follows **Semantic Versioning** for BC guarantees:

- **MAJOR** (2.x → 3.x): BC breaks allowed
- **MINOR** (2.4.7 → 2.4.9): No BC breaks in `@api`, new features added
- **PATCH** (2.4.7 → 2.4.8): No BC breaks, bug fixes only

### What's Protected by @api

| Element | BC Guarantee | Example |
|---------|--------------|---------|
| Interface method signature | ✓ | Cannot remove parameters, change return type |
| Interface method name | ✓ | Cannot rename or remove |
| Exception types thrown | ✓ | Cannot add new checked exceptions |
| Data Interface getters | ✓ | Cannot remove or rename |
| Data Interface setters | ✓ (fluid interface) | Must return `$this` |
| Constant values | ✓ | Cannot change meaning |

### What's NOT Protected

| Element | No BC Guarantee | Why |
|---------|-----------------|-----|
| Concrete classes | Can change | Internal implementation details |
| Private/protected methods | Can change | Not part of public API |
| Constructor signatures | Can change (caveat) | DI auto-wiring, but use caution |
| Non-@api interfaces | Can change | Internal contracts |
| Database schema (direct access) | Can change | Use repositories instead |

### Safe Changes

```php
// SAFE: Adding optional parameter with default
interface ProductRepositoryInterface
{
    // Before
    public function get($sku, $editMode = false, $storeId = null);

    // After - new optional parameter
    public function get($sku, $editMode = false, $storeId = null, $forceReload = false);
}

// SAFE: Adding new method (hypothetical example)
interface ProductRepositoryInterface
{
    public function getList(SearchCriteriaInterface $criteria);

    // New method added (does not break existing implementations)
    public function getByProductType(string $typeId);
}

// SAFE: Widening return type (PHP 7.4+)
interface ProductRepositoryInterface
{
    // Before
    public function get($sku): ProductInterface;

    // After - allows null (less restrictive)
    public function get($sku): ?ProductInterface;
}
```

### Breaking Changes (Forbidden)

```php
// BREAKING: Removing parameter
interface ProductRepositoryInterface
{
    // Before
    public function get($sku, $editMode = false, $storeId = null);

    // After - BREAKS EXISTING CALLS
    public function get($sku);
}

// BREAKING: Adding required parameter
interface ProductRepositoryInterface
{
    // Before
    public function get($sku);

    // After - BREAKS EXISTING CALLS
    public function get($sku, $storeId);
}

// BREAKING: Changing return type (narrowing)
interface ProductRepositoryInterface
{
    // Before
    public function getList(SearchCriteriaInterface $criteria): SearchResultsInterface;

    // After - BREAKS TYPE CHECKS
    public function getList(SearchCriteriaInterface $criteria): ProductSearchResultsInterface;
}
```

### Versioning Strategy

When BC breaks are unavoidable:

1. **Deprecate** the old method with `@deprecated` tag
2. Create **new method** with updated signature
3. Old method **delegates** to new method
4. Remove old method in **next major version**

```php
// Hypothetical example of versioning strategy:
interface ExampleRepositoryInterface
{
    /**
     * @deprecated Use saveV2() instead
     * @see saveV2()
     */
    public function save(ExampleInterface $entity);

    /**
     * @since 2.4.8
     */
    public function saveV2(ExampleInterface $entity, array $options = []);
}
```

> **Note:** This is an illustrative example. In practice, Magento rarely deprecates core repository methods and instead adds new optional parameters with defaults to maintain backward compatibility.

---

## Real-World Examples from Core Modules

### Example 1: CustomerRepository

**Location:** `Magento\Customer\Model\ResourceModel\CustomerRepository`

**Key Features:**
- Caching layer (registry pattern)
- Event dispatch (`customer_save_before`, `customer_save_after`)
- Validation (email uniqueness, required fields)
- Extension attribute population (customer groups, addresses)

```php
public function save(CustomerInterface $customer, $passwordHash = null)
{
    $this->validate($customer);

    // Trigger before-save event
    $this->eventManager->dispatch(
        'customer_save_before',
        ['customer' => $customer]
    );

    // Map Data Interface → Model
    $customerModel = $this->customerFactory->create();
    $this->dataObjectHelper->populateWithArray(
        $customerModel,
        $customer->__toArray(),
        CustomerInterface::class
    );

    // Persist
    $this->resourceModel->save($customerModel);

    // Clear cache
    $this->customerRegistry->remove($customerModel->getId());

    // Trigger after-save event
    $this->eventManager->dispatch(
        'customer_save_after',
        ['customer' => $customerModel]
    );

    // Map Model → Data Interface
    return $customerModel->getDataModel();
}
```

**Lessons:**
- Repositories orchestrate validation, events, caching—not just DB ops
- Always return updated entity (with new IDs, timestamps)
- Use registry/cache to avoid redundant DB hits

### Example 2: ProductRepository with Store Context

**Location:** `Magento\Catalog\Model\ProductRepository`

**Key Features:**
- Multi-store/multi-website support via `$storeId` parameter
- EAV attribute loading (only requested attributes)
- Extension attribute plugins (inventory, reviews, prices)
- Cache tags for FPC invalidation

```php
public function get($sku, $editMode = false, $storeId = null, $forceReload = false)
{
    $cacheKey = $this->getCacheKey([$editMode, $storeId]);

    if (!$forceReload && isset($this->instances[$sku][$cacheKey])) {
        return $this->instances[$sku][$cacheKey];
    }

    $product = $this->productFactory->create();

    if ($editMode) {
        $product->setData('_edit_mode', true);
    }

    if ($storeId !== null) {
        $product->setStoreId($storeId);
    }

    $this->resourceModel->load($product, $sku, 'sku');

    if (!$product->getId()) {
        throw new NoSuchEntityException(
            __('The product with SKU "%1" doesn\'t exist.', $sku)
        );
    }

    $this->instances[$sku][$cacheKey] = $product->getDataModel();

    return $this->instances[$sku][$cacheKey];
}
```

**Lessons:**
- `$editMode`, `$storeId`, `$forceReload` control caching and scope
- Store-level attributes (price, name) require explicit `$storeId`
- Cache keys must account for all context parameters

### Example 3: OrderRepository with Relations

**Location:** `Magento\Sales\Model\OrderRepository`

**Key Features:**
- Lazy-loads related entities (items, addresses, payments)
- Extension attributes for third-party data (shipping tracking, taxes)
- Search includes JOIN with grid table for performance

```php
public function getList(SearchCriteriaInterface $searchCriteria)
{
    $collection = $this->collectionFactory->create();

    // Join with sales_order_grid for pre-aggregated fields
    $collection->getSelect()->joinLeft(
        ['grid' => $this->resourceConnection->getTableName('sales_order_grid')],
        'main_table.entity_id = grid.entity_id',
        ['increment_id', 'customer_name']
    );

    // Apply SearchCriteria filters
    foreach ($searchCriteria->getFilterGroups() as $filterGroup) {
        $this->addFilterGroupToCollection($filterGroup, $collection);
    }

    // Apply sorting
    foreach ($searchCriteria->getSortOrders() as $sortOrder) {
        $collection->addOrder($sortOrder->getField(), $sortOrder->getDirection());
    }

    // Pagination
    $collection->setCurPage($searchCriteria->getCurrentPage());
    $collection->setPageSize($searchCriteria->getPageSize());

    // Load related items
    foreach ($collection as $order) {
        $order->getItems(); // Lazy-load items
        $order->getAddresses(); // Lazy-load addresses
    }

    // Build SearchResults
    $searchResults = $this->searchResultsFactory->create();
    $searchResults->setSearchCriteria($searchCriteria);
    $searchResults->setItems($collection->getItems());
    $searchResults->setTotalCount($collection->getSize());

    return $searchResults;
}
```

**Lessons:**
- `getList()` can optimize with JOINs to grid/flat tables
- Lazy-load related entities after collection iteration
- SearchResults must include original SearchCriteria for pagination

---

## Trade-Offs and Design Decisions

### Repository Overhead vs Flexibility

**Overhead:**
- Additional abstraction layer (Repository → ResourceModel → Collection → DB)
- Event dispatch, validation, and caching add milliseconds per call
- SearchCriteria less expressive than raw SQL

**Flexibility:**
- Plugin interception at single point
- Testable, mockable interfaces
- Automatic web API exposure
- Upgrade-safe, BC-guaranteed

**Verdict:** For public APIs and business logic, the overhead is justified. For internal batch jobs, direct ResourceModel access is acceptable.

### Data Interfaces vs Models

**Data Interfaces (DTOs):**
- Immutable (via setters returning new instances, conceptually)
- No behavior, just data
- Type-safe via interface contracts
- Serializable to JSON/XML

**Models:**
- Mutable (setters modify state)
- Business logic methods (calculateDiscount, canShip)
- Coupled to ResourceModels and Collections
- Not directly serializable

**Verdict:** Use Data Interfaces at API boundaries, Models internally. Map between them in repositories.

### SearchCriteria vs Direct Collections

**SearchCriteria:**
- Declarative, composable filters
- Pagination and sorting built-in
- Compatible with web APIs
- Limited to basic filters (no JOINs, subqueries)

**Collections:**
- Full SQL control (JOINs, GROUP BY, HAVING)
- Direct Zend_Db_Select manipulation
- Optimizable for specific queries
- Not portable to web APIs

**Verdict:** Default to SearchCriteria. Use Collections when you need SQL features (admin grids, reports).

---

## Antipatterns to Avoid

### 1. Bypassing Repositories in Business Logic

```php
// WRONG: Direct Model usage in service layer
public function __construct(
    \Magento\Catalog\Model\ProductFactory $productFactory
) {}

public function execute(int $productId): void
{
    $product = $this->productFactory->create()->load($productId);
    $product->setPrice(99.99);
    $product->save();
}
```

**Why Wrong:**
- No plugin interception
- No validation or events
- Breaks if Model signature changes
- Can't mock for testing

**Fix:** Use `ProductRepositoryInterface`.

### 2. Mixing Model and Data Interface Methods

```php
// WRONG: Calling Model methods on Data Interface
$product = $this->productRepository->get('TEST-SKU');
$product->load($productId); // load() doesn't exist on Data Interface!
$product->save(); // save() doesn't exist on Data Interface!
```

**Why Wrong:**
- Data Interfaces don't have `load()`, `save()`, `delete()` methods
- IDE autocomplete won't help
- Code breaks if implementation changes

**Fix:** Use repository methods (`save()`, `getById()`).

### 3. Ignoring SearchCriteria Limits

```php
// WRONG: Loading all products into memory
$searchCriteria = $this->searchCriteriaBuilder->create();
$searchResults = $this->productRepository->getList($searchCriteria);

foreach ($searchResults->getItems() as $product) {
    // Process 50,000 products - OOM!
}
```

**Why Wrong:**
- Default page size may be unlimited or very large
- Memory exhaustion on large catalogs

**Fix:** Use explicit pagination:

```php
$pageSize = 100;
$currentPage = 1;

do {
    $this->searchCriteriaBuilder->setPageSize($pageSize);
    $this->searchCriteriaBuilder->setCurrentPage($currentPage);
    $searchCriteria = $this->searchCriteriaBuilder->create();
    $searchResults = $this->productRepository->getList($searchCriteria);

    foreach ($searchResults->getItems() as $product) {
        // Process in batches
    }

    $currentPage++;
} while ($searchResults->getTotalCount() > ($currentPage - 1) * $pageSize);
```

### 4. Not Handling Extension Attributes in Plugins

```php
// WRONG: Plugin doesn't preserve extension attributes
public function afterSave(
    ProductRepositoryInterface $subject,
    ProductInterface $result
): ProductInterface {
    $newProduct = $this->productFactory->create();
    $newProduct->setSku($result->getSku());
    return $newProduct; // Lost all extension attributes!
}
```

**Why Wrong:**
- Other plugins' extension attributes are discarded
- GraphQL queries return incomplete data

**Fix:** Always return the original or clone:

```php
public function afterSave(
    ProductRepositoryInterface $subject,
    ProductInterface $result
): ProductInterface {
    // Modify extension attributes on existing object
    $extensionAttributes = $result->getExtensionAttributes();
    $extensionAttributes->setWarrantyInfo($this->loadWarranty($result->getId()));
    $result->setExtensionAttributes($extensionAttributes);

    return $result;
}
```

---

## Further Reading

### Official Documentation
- [Service Contracts](https://developer.adobe.com/commerce/php/development/components/service-contracts/) - Adobe Developer Portal
- [Backward Compatibility Policy](https://developer.adobe.com/commerce/contributor/guides/code-contributions/backward-compatibility-policy/) - Adobe Contributor Guide
- [Search Criteria](https://developer.adobe.com/commerce/php/development/components/searching-with-repositories/) - Adobe Developer Portal
- [Extension Attributes](https://developer.adobe.com/commerce/php/development/components/add-attributes/) - Adobe Developer Portal

### Core Module Examples
- `Magento\Catalog\Api\ProductRepositoryInterface` - Product CRUD
- `Magento\Customer\Api\CustomerRepositoryInterface` - Customer CRUD
- `Magento\Sales\Api\OrderRepositoryInterface` - Order CRUD with relations
- `Magento\Framework\Api\SearchCriteriaBuilder` - Query builder

### Books & Deep Dives
- *Magento 2 Developer's Guide* by Branko Ajzele (Chapter 4: Service Contracts)
- *Magento 2 Explained* by Stephen Burge (Chapter 7: Repositories)

---

## Conclusion

Service contracts and repositories are the **foundation of upgrade-safe, testable, extensible Magento 2 code**. By adhering to `@api` interfaces, you ensure your modules survive Magento upgrades and integrate seamlessly with the broader ecosystem.

**Key Takeaways:**
1. **Always prefer repositories over direct Model access** in business logic
2. **Use SearchCriteria for standard queries**, Collections for complex SQL
3. **Extension attributes for module integrations**, custom attributes for merchant config
4. **Understand BC policy** to avoid breaking changes
5. **Study core repositories** (Product, Customer, Order) for patterns and best practices

Mastering these patterns separates professional Magento developers from those fighting upgrade nightmares.

---

**Assumptions:**
- Magento Open Source / Adobe Commerce 2.4.7+
- PHP 8.2+
- Modules follow PSR-4 autoloading and Magento module structure

**Why This Approach:**
- **Upgrade Safety:** Service contracts guarantee BC across minor versions
- **Testability:** Interface-based DI enables mocking and unit tests
- **Extensibility:** Plugins on repositories provide centralized interception
- **Web API Ready:** `@api` interfaces map directly to REST/GraphQL
- **Standards:** Aligns with Magento architecture and community best practices

**Security Impact:**
- Repositories enforce validation (email uniqueness, required fields)
- Data Interfaces prevent SQL injection (no raw query construction)
- Extension attributes via plugins can add ACL checks, audit logging
- Service contracts expose consistent error handling (NoSuchEntityException)

**Performance Impact:**
- Repository overhead: ~1-5ms per call (events, validation, caching)
- SearchCriteria less efficient than optimized Collections for complex queries
- Extension attribute plugins can add N+1 query problems if not batched
- Caching in repositories (registry pattern) mitigates repeated `getById()` calls

**Backward Compatibility:**
- `@api` interfaces guaranteed BC within major version (2.x)
- New optional parameters safe to add
- New methods safe to add
- Removing methods, changing signatures, or adding required params = BC break
- Always check `@since` tags and CHANGELOG for deprecations

**Tests to Add:**
- Unit tests: Mock repositories, verify method calls and exceptions
- Integration tests: Test SearchCriteria filters, pagination, extension attributes
- API functional tests (MFTF): Validate web API endpoints using repositories
- Mutation tests: Ensure repository validation logic is covered (PHPUnit + Infection)

**Docs to Update:**
- README: Module dependencies, service contracts exposed
- CHANGELOG: New repository methods, BC notes
- GraphQL schema: Document extension attributes in schema.graphqls
- Admin user guide: Screenshots of custom attributes in product form

## Related Documentation

### Related Guides

- [GraphQL Resolver Patterns in Magento 2](graphql-resolver-patterns.md)
- [Plugin System Deep Dive: Mastering Magento 2 Interception](../tutorials/plugin-system-deep-dive.md)
- [ERP Integration Patterns for Magento 2](../how-to/erp-integration.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Customer Overview](../../modules/customer/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
- [Magento_Quote Overview](../../modules/quote/README.md)
