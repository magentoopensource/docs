---
title: "GraphQL Resolver Patterns in Magento 2"
description: "Developer guide: GraphQL Resolver Patterns in Magento 2"
type: "explanation"
tier: 1
difficulty: "intermediate"
version: "2.4.7+"
estimated_time: "20 minutes"
topics:
  - graphql
  - resolvers
  - api-design
  - performance
  - authorization
last_updated: "2026-02-07"
---

# GraphQL Resolver Patterns in Magento 2

## Introduction

GraphQL has emerged as Magento's **preferred API layer** for headless commerce, Progressive Web Apps (PWA Studio), and mobile applications. Unlike REST, which requires multiple round-trips for related data, GraphQL enables clients to fetch exactly the data they need in a single request—no more over-fetching or under-fetching.

Magento's GraphQL implementation is built on **webonyx/graphql-php** with custom resolver patterns, performance optimizations, and integration with existing service contracts. Understanding these patterns is critical for building fast, secure, and maintainable GraphQL endpoints.

### What You'll Learn

- Magento's GraphQL architecture and request lifecycle
- schema.graphqls syntax and type system
- Query resolvers vs Mutation resolvers
- DataProvider pattern for separation of concerns
- Batch resolvers to eliminate N+1 query problems
- Custom types, interfaces, and unions
- Authorization and ACL integration
- Caching strategies for GraphQL responses
- Error handling and validation
- Real-world examples: custom product attributes, customer mutations

---

## Magento GraphQL Architecture Overview

### Request Lifecycle

```
Client Request (POST /graphql)
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│             GraphQL Entry Point                         │
│          (Magento\GraphQl\Controller\GraphQl)           │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              Schema Provider                            │
│  - Aggregates schema.graphqls from all modules          │
│  - Builds type registry                                 │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│             Query Parser & Validator                    │
│  - Parses query syntax                                  │
│  - Validates against schema                             │
│  - Checks query depth/complexity                        │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│            Resolver Execution                           │
│  - Invokes field resolvers (Query, Mutation, Type)      │
│  - Passes context (store, customer, authorization)      │
│  - Handles batching and lazy loading                    │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│           Response Formatter                            │
│  - Serializes resolved data to JSON                     │
│  - Includes errors array if validation/exceptions       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
                JSON Response
```

### Core Components

| Component | Purpose | Example |
|-----------|---------|---------|
| **schema.graphqls** | Define types, queries, mutations | `type Product`, `type Query` |
| **Resolver** | Fetch data for a field | `ProductResolver`, `CustomerResolver` |
| **DataProvider** | Business logic layer (delegates to services) | `ProductDataProvider` |
| **Type Resolver** | Resolve interfaces/unions to concrete types | `ProductTypeResolver` |
| **Authorization** | Check ACL/permissions | `@doc(description: "requires customer token")` |
| **Cache** | Cache resolved data per query | `CacheKeyCalculator`, `@cache` directive |

---

## schema.graphqls Syntax

### Basic Structure

Every Magento module can contribute a `etc/schema.graphqls` file. Magento merges all schemas into a single type registry.

```graphql
# VendorName/ModuleName/etc/schema.graphqls

type Query {
    """
    Fetch a product by SKU
    @doc(description: "Returns product details including custom attributes")
    """
    product(
        sku: String! @doc(description: "Product SKU")
    ): Product @resolver(class: "VendorName\\ModuleName\\Model\\Resolver\\Product") @cache(cacheIdentity: "VendorName\\ModuleName\\Model\\Resolver\\Product\\Identity")
}

type Product {
    id: Int
    sku: String
    name: String
    price: Float
    warranty_info: WarrantyInfo @resolver(class: "VendorName\\ModuleName\\Model\\Resolver\\Product\\WarrantyInfo")
}

type WarrantyInfo {
    duration_months: Int
    coverage_type: String
    terms_url: String
}

type Mutation {
    """
    Update product warranty information
    @doc(description: "Requires admin authorization")
    """
    updateProductWarranty(
        input: UpdateProductWarrantyInput!
    ): UpdateProductWarrantyOutput @resolver(class: "VendorName\\ModuleName\\Model\\Resolver\\UpdateProductWarranty")
}

input UpdateProductWarrantyInput {
    sku: String!
    duration_months: Int!
    coverage_type: String!
}

type UpdateProductWarrantyOutput {
    product: Product
}
```

### Type System

#### Scalar Types

| Type | Description | Example |
|------|-------------|---------|
| `Int` | Signed 32-bit integer | `42`, `-10` |
| `Float` | Double-precision floating point | `99.99`, `3.14` |
| `String` | UTF-8 character sequence | `"Hello"`, `"SKU123"` |
| `Boolean` | true/false | `true`, `false` |
| `ID` | Unique identifier (serialized as String) | `"123"`, `"ABC"` |

#### Object Types

```graphql
type Product {
    id: Int!              # Non-nullable
    sku: String!
    name: String
    price: Float
    categories: [Category]  # Nullable list of Category objects
    related_products: [Product!]!  # Non-nullable list of non-nullable Products
}
```

#### Input Types

Used for mutation arguments (cannot have resolvers):

```graphql
input CreateProductInput {
    sku: String!
    name: String!
    price: Float!
    attribute_set_id: Int!
    type_id: String! = "simple"  # Default value
}
```

#### Interfaces

```graphql
interface ProductInterface {
    id: Int!
    sku: String!
    name: String!
}

type SimpleProduct implements ProductInterface {
    id: Int!
    sku: String!
    name: String!
    weight: Float
}

type ConfigurableProduct implements ProductInterface {
    id: Int!
    sku: String!
    name: String!
    configurable_options: [ConfigurableOption!]!
}

type Query {
    productBySku(sku: String!): ProductInterface @resolver(...)
}
```

**Type Resolver Required:** When returning an interface, Magento needs to know the concrete type:

```php
// Model/Resolver/Product/ProductTypeResolver.php
class ProductTypeResolver implements TypeResolverInterface
{
    public function resolveType(array $data): string
    {
        if (isset($data['type_id'])) {
            return match ($data['type_id']) {
                'simple' => 'SimpleProduct',
                'configurable' => 'ConfigurableProduct',
                'bundle' => 'BundleProduct',
                default => 'SimpleProduct',
            };
        }
        return 'SimpleProduct';
    }
}
```

#### Unions

```graphql
union SearchResult = Product | Category | CmsPage

type Query {
    search(query: String!): [SearchResult!]! @resolver(...)
}
```

**Type Resolver:**

```php
class SearchResultTypeResolver implements TypeResolverInterface
{
    public function resolveType(array $data): string
    {
        return match ($data['entity_type']) {
            'product' => 'Product',
            'category' => 'Category',
            'cms_page' => 'CmsPage',
            default => throw new \RuntimeException('Unknown entity type'),
        };
    }
}
```

#### Enums

```graphql
enum ProductStatus {
    ENABLED
    DISABLED
}

type Product {
    status: ProductStatus
}
```

### Directives

| Directive | Purpose | Example |
|-----------|---------|---------|
| `@resolver` | Specify resolver class for field | `@resolver(class: "VendorName\\Module\\Model\\Resolver\\Product")` |
| `@doc` | Add description (appears in introspection) | `@doc(description: "Product SKU")` |
| `@cache` | Define cache identity for FPC | `@cache(cacheIdentity: "VendorName\\Module\\Model\\CacheIdentity")` |
| `@deprecated` | Mark field as deprecated | `@deprecated(reason: "Use `newField` instead")` |

---

## Query Resolvers

### Purpose

Query resolvers fetch data for read operations. They implement `Magento\Framework\GraphQl\Query\ResolverInterface`.

### Interface Contract

```php
namespace Magento\Framework\GraphQl\Query;

interface ResolverInterface
{
    /**
     * @param Field $field
     * @param ContextInterface $context
     * @param ResolveInfo $info
     * @param array|null $value
     * @param array|null $args
     * @return mixed|Value
     */
    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    );
}
```

### Parameters Explained

| Parameter | Type | Purpose |
|-----------|------|---------|
| `$field` | `Field` | Current field being resolved (name, alias) |
| `$context` | `ContextInterface` | Request context (customer, store, authorization) |
| `$info` | `ResolveInfo` | Query AST, parent type, selection set |
| `$value` | `array\|null` | Parent object's resolved value (for nested fields) |
| `$args` | `array\|null` | Arguments passed to this field |

### Example: Simple Product Resolver

```php
namespace VendorName\ModuleName\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\GraphQl\Exception\GraphQlInputException;
use Magento\Framework\GraphQl\Exception\GraphQlNoSuchEntityException;

class Product implements ResolverInterface
{
    public function __construct(
        private \Magento\Catalog\Api\ProductRepositoryInterface $productRepository
    ) {}

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        if (!isset($args['sku']) || empty($args['sku'])) {
            throw new GraphQlInputException(__('SKU is required'));
        }

        $storeId = (int) $context->getExtensionAttributes()->getStore()->getId();

        try {
            $product = $this->productRepository->get($args['sku'], false, $storeId);
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            throw new GraphQlNoSuchEntityException(
                __('Product with SKU "%1" does not exist', $args['sku'])
            );
        }

        return [
            'id' => $product->getId(),
            'sku' => $product->getSku(),
            'name' => $product->getName(),
            'price' => $product->getPrice(),
            'model' => $product, // Pass model for nested resolvers
        ];
    }
}
```

**Key Points:**
- Validate arguments; throw `GraphQlInputException` for invalid input
- Use service contracts (`ProductRepositoryInterface`), not Models directly
- Respect store context from `$context->getExtensionAttributes()->getStore()`
- Return associative array with snake_case keys matching schema fields
- Include `model` key for nested resolvers to avoid re-fetching

---

## Mutation Resolvers

### Purpose

Mutation resolvers handle write operations (create, update, delete). They follow the same `ResolverInterface` but typically return output types with success indicators.

### Example: Update Product Warranty

**Schema:**

```graphql
type Mutation {
    updateProductWarranty(
        input: UpdateProductWarrantyInput!
    ): UpdateProductWarrantyOutput @resolver(class: "VendorName\\ModuleName\\Model\\Resolver\\UpdateProductWarranty")
}

input UpdateProductWarrantyInput {
    sku: String!
    duration_months: Int!
    coverage_type: String!
    terms_url: String
}

type UpdateProductWarrantyOutput {
    product: Product
    success: Boolean!
    message: String
}
```

**Resolver:**

```php
namespace VendorName\ModuleName\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\GraphQl\Exception\GraphQlAuthorizationException;
use Magento\Framework\GraphQl\Exception\GraphQlInputException;

class UpdateProductWarranty implements ResolverInterface
{
    public function __construct(
        private \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
        private \VendorName\ModuleName\Api\WarrantyManagementInterface $warrantyManagement,
        private \Magento\Framework\AuthorizationInterface $authorization
    ) {}

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        // Authorization check
        if (!$this->authorization->isAllowed('Magento_Catalog::products')) {
            throw new GraphQlAuthorizationException(
                __('You do not have permission to update product warranty')
            );
        }

        // Validate input
        if (!isset($args['input']['sku']) || empty($args['input']['sku'])) {
            throw new GraphQlInputException(__('SKU is required'));
        }

        if ($args['input']['duration_months'] < 1) {
            throw new GraphQlInputException(__('Warranty duration must be at least 1 month'));
        }

        $input = $args['input'];
        $storeId = (int) $context->getExtensionAttributes()->getStore()->getId();

        try {
            // Fetch product
            $product = $this->productRepository->get($input['sku'], false, $storeId);

            // Update warranty via service contract
            $this->warrantyManagement->update(
                $product->getId(),
                $input['duration_months'],
                $input['coverage_type'],
                $input['terms_url'] ?? ''
            );

            // Reload product to reflect changes
            $product = $this->productRepository->get($input['sku'], false, $storeId, true);

            return [
                'product' => [
                    'id' => $product->getId(),
                    'sku' => $product->getSku(),
                    'name' => $product->getName(),
                    'model' => $product,
                ],
                'success' => true,
                'message' => 'Warranty updated successfully',
            ];
        } catch (\Exception $e) {
            return [
                'product' => null,
                'success' => false,
                'message' => $e->getMessage(),
            ];
        }
    }
}
```

**Best Practices:**
- Check authorization FIRST (ACL, customer token, admin role)
- Validate all input fields; throw `GraphQlInputException` for bad data
- Delegate business logic to service contracts (testable, reusable)
- Return structured output with `success` boolean and `message`
- Reload entity after mutation to reflect DB changes (auto-increments, timestamps)

---

## DataProvider Pattern

### Problem

Resolvers should be **thin orchestrators**, not business logic containers. Mixing data fetching, validation, and transformation in resolvers leads to:
- Untestable code (hard to mock GraphQL-specific objects)
- Code duplication across Query and Mutation resolvers
- Difficult to reuse logic in REST APIs or CLI commands

### Solution: DataProvider

Extract data-fetching logic into **DataProvider** classes that accept simple types and return domain objects.

**Architecture:**

```
Resolver (GraphQL-specific)
    │
    ├─ Validate arguments
    ├─ Check authorization
    └─ Delegate to DataProvider
               │
               ├─ Use service contracts (Repository, Management)
               ├─ Transform data
               └─ Return domain objects
```

### Example: ProductDataProvider

```php
namespace VendorName\ModuleName\Model\Resolver\Product;

class DataProvider
{
    public function __construct(
        private \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
        private \Magento\Catalog\Model\ResourceModel\Product\CollectionFactory $collectionFactory
    ) {}

    /**
     * Get product by SKU
     *
     * @param string $sku
     * @param int $storeId
     * @return array
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function getProductBySku(string $sku, int $storeId): array
    {
        $product = $this->productRepository->get($sku, false, $storeId);

        return $this->formatProduct($product);
    }

    /**
     * Get products by filter
     *
     * @param array $filters ['status' => 1, 'visibility' => [2, 3, 4]]
     * @param int $storeId
     * @param int $pageSize
     * @param int $currentPage
     * @return array
     */
    public function getProducts(
        array $filters,
        int $storeId,
        int $pageSize = 20,
        int $currentPage = 1
    ): array {
        $collection = $this->collectionFactory->create();
        $collection->setStoreId($storeId);
        $collection->addAttributeToSelect(['name', 'sku', 'price', 'status']);

        foreach ($filters as $field => $value) {
            $collection->addFieldToFilter($field, $value);
        }

        $collection->setPageSize($pageSize);
        $collection->setCurPage($currentPage);

        $items = [];
        foreach ($collection as $product) {
            $items[] = $this->formatProduct($product);
        }

        return [
            'items' => $items,
            'total_count' => $collection->getSize(),
            'page_info' => [
                'page_size' => $pageSize,
                'current_page' => $currentPage,
                'total_pages' => ceil($collection->getSize() / $pageSize),
            ],
        ];
    }

    /**
     * Format product for GraphQL output
     *
     * @param \Magento\Catalog\Api\Data\ProductInterface $product
     * @return array
     */
    private function formatProduct(\Magento\Catalog\Api\Data\ProductInterface $product): array
    {
        return [
            'id' => $product->getId(),
            'sku' => $product->getSku(),
            'name' => $product->getName(),
            'price' => $product->getPrice(),
            'status' => $product->getStatus(),
            'model' => $product, // For nested resolvers
        ];
    }
}
```

**Resolver (Thin Wrapper):**

```php
namespace VendorName\ModuleName\Model\Resolver;

class Products implements ResolverInterface
{
    public function __construct(
        private \VendorName\ModuleName\Model\Resolver\Product\DataProvider $dataProvider
    ) {}

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $storeId = (int) $context->getExtensionAttributes()->getStore()->getId();

        $filters = $args['filter'] ?? [];
        $pageSize = $args['pageSize'] ?? 20;
        $currentPage = $args['currentPage'] ?? 1;

        return $this->dataProvider->getProducts($filters, $storeId, $pageSize, $currentPage);
    }
}
```

**Benefits:**
- DataProvider is testable with simple mocks (no GraphQL objects)
- Reusable in REST API controllers, CLI commands
- Single responsibility: Resolver = GraphQL contract, DataProvider = business logic

---

## Batch Resolvers for Performance

### The N+1 Query Problem

Consider this query:

```graphql
query {
  products(filter: {status: {eq: "1"}}) {
    items {
      sku
      name
      warranty_info {
        duration_months
        coverage_type
      }
    }
  }
}
```

**Naive Implementation:**

```php
// Product resolver returns 100 products
// WarrantyInfo resolver runs ONCE PER PRODUCT:
class WarrantyInfo implements ResolverInterface
{
    public function resolve(/* ... */)
    {
        $productId = $value['id'];
        // SELECT * FROM warranty WHERE product_id = ? -- 100 QUERIES!
        $warranty = $this->warrantyRepository->getByProductId($productId);

        return [
            'duration_months' => $warranty->getDurationMonths(),
            'coverage_type' => $warranty->getCoverageType(),
        ];
    }
}
```

**Result:** 1 query for products + 100 queries for warranties = **101 total queries** (N+1 problem).

### Solution: Batch Resolver

Magento's `webonyx/graphql-php` doesn't natively support DataLoader-style batching (as in GraphQL.js). The pattern is to:

1. **Collect product IDs** during product resolution
2. **Batch-fetch warranties** in a single query
3. **Cache results** in resolver or DataProvider

**Optimized WarrantyInfo Resolver:**

```php
namespace VendorName\ModuleName\Model\Resolver\Product;

class WarrantyInfo implements ResolverInterface
{
    private array $warrantyCache = [];

    public function __construct(
        private \VendorName\ModuleName\Model\ResourceModel\Warranty\CollectionFactory $warrantyCollectionFactory
    ) {}

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $productId = $value['id'] ?? null;

        if ($productId === null) {
            return null;
        }

        // Lazy-load warranty cache on first access
        if (empty($this->warrantyCache)) {
            $this->loadWarrantyBatch($info);
        }

        $warranty = $this->warrantyCache[$productId] ?? null;

        if ($warranty === null) {
            return null;
        }

        return [
            'duration_months' => $warranty->getDurationMonths(),
            'coverage_type' => $warranty->getCoverageType(),
            'terms_url' => $warranty->getTermsUrl(),
        ];
    }

    /**
     * Batch-load warranties for all products in current query
     */
    private function loadWarrantyBatch(ResolveInfo $info): void
    {
        // Extract all product IDs from parent query
        // This is a simplification; real implementation inspects $info->path
        // or collects IDs from parent resolver
        $productIds = $this->extractProductIds($info);

        if (empty($productIds)) {
            return;
        }

        // Single query for all warranties
        $collection = $this->warrantyCollectionFactory->create();
        $collection->addFieldToFilter('product_id', ['in' => $productIds]);

        foreach ($collection as $warranty) {
            $this->warrantyCache[$warranty->getProductId()] = $warranty;
        }
    }

    /**
     * Extract product IDs from ResolveInfo (simplified)
     */
    private function extractProductIds(ResolveInfo $info): array
    {
        // In practice, this requires traversing $info->fieldNodes
        // or receiving IDs from parent resolver via $value['product_ids']
        // For demonstration, assume parent passes this in context
        return $info->variableValues['product_ids'] ?? [];
    }
}
```

**Better Approach: Parent Passes IDs**

```php
// Products resolver
class Products implements ResolverInterface
{
    public function resolve(/* ... */)
    {
        $products = $this->dataProvider->getProducts($filters, $storeId, $pageSize, $currentPage);

        $productIds = array_map(fn($p) => $p['id'], $products['items']);

        // Pre-load warranties in DataProvider
        $this->dataProvider->preloadWarranties($productIds);

        return $products;
    }
}

// DataProvider
class DataProvider
{
    private array $warrantyCache = [];

    public function preloadWarranties(array $productIds): void
    {
        $collection = $this->warrantyCollectionFactory->create();
        $collection->addFieldToFilter('product_id', ['in' => $productIds]);

        foreach ($collection as $warranty) {
            $this->warrantyCache[$warranty->getProductId()] = $warranty;
        }
    }

    public function getWarrantyByProductId(int $productId): ?array
    {
        $warranty = $this->warrantyCache[$productId] ?? null;

        if ($warranty === null) {
            return null;
        }

        return [
            'duration_months' => $warranty->getDurationMonths(),
            'coverage_type' => $warranty->getCoverageType(),
        ];
    }
}

// WarrantyInfo resolver
class WarrantyInfo implements ResolverInterface
{
    public function __construct(
        private DataProvider $dataProvider
    ) {}

    public function resolve(/* ... */)
    {
        $productId = $value['id'] ?? null;
        return $this->dataProvider->getWarrantyByProductId($productId);
    }
}
```

**Result:** 1 query for products + 1 query for warranties = **2 total queries** (optimal).

---

## Custom Types and Interfaces

### When to Create Custom Types

- **Complex nested data:** Product options, customer addresses, order items
- **Shared fields across entities:** Use interfaces for common fields (e.g., `EntityInterface` with `id`, `created_at`)
- **Polymorphic responses:** Use unions for search results (Product | Category | Page)

### Example: Custom Address Type

```graphql
interface AddressInterface {
    street: [String!]!
    city: String!
    region: String
    postcode: String
    country_code: String!
}

type CustomerAddress implements AddressInterface {
    id: Int!
    street: [String!]!
    city: String!
    region: String
    postcode: String
    country_code: String!
    default_shipping: Boolean
    default_billing: Boolean
}

type OrderAddress implements AddressInterface {
    street: [String!]!
    city: String!
    region: String
    postcode: String
    country_code: String!
    telephone: String
}

type Customer {
    addresses: [CustomerAddress!]!
}

type Order {
    shipping_address: OrderAddress
    billing_address: OrderAddress
}
```

**Type Resolver:**

```php
class AddressTypeResolver implements TypeResolverInterface
{
    public function resolveType(array $data): string
    {
        if (isset($data['default_shipping'])) {
            return 'CustomerAddress';
        }

        if (isset($data['telephone'])) {
            return 'OrderAddress';
        }

        return 'CustomerAddress'; // Default
    }
}
```

---

## Authorization in Resolvers

### Customer Token Authorization

```php
class MyAccountResolver implements ResolverInterface
{
    public function resolve(/* ... */)
    {
        $currentUserId = $context->getUserId();

        if ($currentUserId === null || $currentUserId === 0) {
            throw new GraphQlAuthorizationException(
                __('The current customer is not authorized.')
            );
        }

        // Customer-specific logic
        $customer = $this->customerRepository->getById($currentUserId);

        return [
            'email' => $customer->getEmail(),
            'firstname' => $customer->getFirstname(),
        ];
    }
}
```

**Request:**

```http
POST /graphql
Authorization: Bearer <customer_token>
Content-Type: application/json

{
  "query": "{ myAccount { email firstname } }"
}
```

### Admin ACL Authorization

```php
class UpdateProductResolver implements ResolverInterface
{
    public function __construct(
        private \Magento\Framework\AuthorizationInterface $authorization
    ) {}

    public function resolve(/* ... */)
    {
        if (!$this->authorization->isAllowed('Magento_Catalog::products')) {
            throw new GraphQlAuthorizationException(
                __('You do not have permission to update products.')
            );
        }

        // Admin-only mutation logic
    }
}
```

**ACL Definition:**

```xml
<!-- etc/acl.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Acl/etc/acl.xsd">
    <acl>
        <resources>
            <resource id="Magento_Backend::admin">
                <resource id="Magento_Catalog::catalog">
                    <resource id="Magento_Catalog::products" title="Products" sortOrder="10"/>
                </resource>
            </resource>
        </resources>
    </acl>
</config>
```

### Guest vs Authenticated

```php
$currentUserId = $context->getUserId();

if ($currentUserId !== null && $currentUserId > 0) {
    // Customer authenticated
} else {
    // Guest user
}
```

---

## Caching GraphQL Responses

### Full Page Cache (FPC) Integration

Magento can cache GraphQL responses in Varnish/FPC if the query is:
1. **Publicly cacheable** (no customer-specific data)
2. **Annotated with `@cache` directive**

### Cache Identity

```graphql
type Query {
    products(filter: ProductFilterInput): ProductSearchResults
        @resolver(class: "Magento\\CatalogGraphQl\\Model\\Resolver\\Products")
        @cache(cacheIdentity: "Magento\\CatalogGraphQl\\Model\\Resolver\\Products\\Identity")
}
```

**Cache Identity Class:**

```php
namespace Magento\CatalogGraphQl\Model\Resolver\Products;

use Magento\Framework\GraphQl\Query\Resolver\IdentityInterface;

class Identity implements IdentityInterface
{
    private const CACHE_TAG = 'cat_p';

    /**
     * @param array $resolvedData
     * @return string[]
     */
    public function getIdentities(array $resolvedData): array
    {
        $ids = [];

        foreach ($resolvedData['items'] ?? [] as $item) {
            $ids[] = sprintf('%s_%s', self::CACHE_TAG, $item['id']);
        }

        return $ids;
    }
}
```

**Cache Tags:** When a product is updated, Magento invalidates `cat_p_123` tag, purging cached queries that included that product.

### Cache Headers

For publicly cacheable queries, Magento sets:

```http
X-Magento-Tags: cat_p_123,cat_p_456
X-Magento-Cache-Control: max-age=3600
```

Varnish uses these headers to cache and invalidate responses.

### Disabling Cache for Customer-Specific Data

```php
class MyAccountResolver implements ResolverInterface
{
    public function resolve(/* ... */)
    {
        // Disable FPC for this query
        $context->getExtensionAttributes()->getStore()->setCurrentCurrencyCode('USD');

        // Customer-specific data
        return ['email' => $customer->getEmail()];
    }
}
```

Or use `@cache(cacheable: false)` in schema:

```graphql
type Query {
    myAccount: Customer
        @resolver(class: "...")
        @cache(cacheable: false)
}
```

---

## Error Handling Best Practices

### Exception Types

| Exception | HTTP Status | Use Case |
|-----------|-------------|----------|
| `GraphQlInputException` | 400 | Invalid arguments, validation errors |
| `GraphQlNoSuchEntityException` | 404 | Entity not found |
| `GraphQlAuthorizationException` | 403 | Insufficient permissions |
| `GraphQlAuthenticationException` | 401 | Missing/invalid token |
| `LocalizedException` | 500 | Generic business logic errors |

### Example: Comprehensive Error Handling

```php
class CreateCustomerResolver implements ResolverInterface
{
    public function resolve(/* ... */)
    {
        // Input validation
        if (empty($args['input']['email'])) {
            throw new GraphQlInputException(__('Email is required'));
        }

        if (!filter_var($args['input']['email'], FILTER_VALIDATE_EMAIL)) {
            throw new GraphQlInputException(__('Invalid email format'));
        }

        try {
            $customer = $this->customerDataFactory->create();
            $customer->setEmail($args['input']['email']);
            $customer->setFirstname($args['input']['firstname']);
            $customer->setLastname($args['input']['lastname']);

            $savedCustomer = $this->customerRepository->save($customer, $args['input']['password']);

            return [
                'customer' => $this->formatCustomer($savedCustomer),
                'success' => true,
            ];
        } catch (\Magento\Framework\Exception\AlreadyExistsException $e) {
            throw new GraphQlInputException(
                __('A customer with email "%1" already exists', $args['input']['email'])
            );
        } catch (\Magento\Framework\Exception\LocalizedException $e) {
            throw new GraphQlInputException(__($e->getMessage()));
        } catch (\Exception $e) {
            // Log unexpected errors
            $this->logger->critical($e);
            throw new LocalizedException(__('Unable to create customer'));
        }
    }
}
```

### Response Format

**Success:**

```json
{
  "data": {
    "createCustomer": {
      "customer": {
        "id": 123,
        "email": "john@example.com"
      },
      "success": true
    }
  }
}
```

**Error:**

```json
{
  "data": {
    "createCustomer": null
  },
  "errors": [
    {
      "message": "A customer with email \"john@example.com\" already exists",
      "extensions": {
        "category": "graphql-input"
      },
      "locations": [{"line": 2, "column": 3}],
      "path": ["createCustomer"]
    }
  ]
}
```

---

## Real-World Examples

### Example 1: Custom Product Attribute in GraphQL

**Requirement:** Expose `manufacturer_country` custom attribute in GraphQL.

**Step 1: Extend Schema**

```graphql
# VendorName/ModuleName/etc/schema.graphqls
interface ProductInterface {
    manufacturer_country: String @doc(description: "Country of manufacture")
        @resolver(class: "VendorName\\ModuleName\\Model\\Resolver\\Product\\ManufacturerCountry")
}
```

**Step 2: Resolver**

```php
namespace VendorName\ModuleName\Model\Resolver\Product;

class ManufacturerCountry implements ResolverInterface
{
    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $product = $value['model'] ?? null;

        if (!$product instanceof \Magento\Catalog\Api\Data\ProductInterface) {
            return null;
        }

        $customAttribute = $product->getCustomAttribute('manufacturer_country');

        return $customAttribute ? $customAttribute->getValue() : null;
    }
}
```

**Query:**

```graphql
query {
  products(filter: {sku: {eq: "TEST-SKU"}}) {
    items {
      sku
      name
      manufacturer_country
    }
  }
}
```

**Response:**

```json
{
  "data": {
    "products": {
      "items": [
        {
          "sku": "TEST-SKU",
          "name": "Test Product",
          "manufacturer_country": "Germany"
        }
      ]
    }
  }
}
```

### Example 2: Customer Address Mutation

**Requirement:** Allow customers to add/update addresses via GraphQL.

**Schema:**

```graphql
type Mutation {
    createCustomerAddress(
        input: CreateCustomerAddressInput!
    ): CreateCustomerAddressOutput
        @resolver(class: "VendorName\\ModuleName\\Model\\Resolver\\CreateCustomerAddress")
}

input CreateCustomerAddressInput {
    street: [String!]!
    city: String!
    region: String
    postcode: String!
    country_code: String!
    telephone: String
    default_shipping: Boolean
    default_billing: Boolean
}

type CreateCustomerAddressOutput {
    address: CustomerAddress
    success: Boolean!
    message: String
}
```

**Resolver:**

```php
namespace VendorName\ModuleName\Model\Resolver;

class CreateCustomerAddress implements ResolverInterface
{
    public function __construct(
        private \Magento\Customer\Api\AddressRepositoryInterface $addressRepository,
        private \Magento\Customer\Api\Data\AddressInterfaceFactory $addressFactory
    ) {}

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $currentUserId = $context->getUserId();

        if ($currentUserId === null || $currentUserId === 0) {
            throw new GraphQlAuthorizationException(
                __('The current customer is not authorized.')
            );
        }

        $input = $args['input'];

        $address = $this->addressFactory->create();
        $address->setCustomerId($currentUserId);
        $address->setStreet($input['street']);
        $address->setCity($input['city']);
        $address->setRegion($input['region'] ?? '');
        $address->setPostcode($input['postcode']);
        $address->setCountryId($input['country_code']);
        $address->setTelephone($input['telephone'] ?? '');
        $address->setIsDefaultShipping($input['default_shipping'] ?? false);
        $address->setIsDefaultBilling($input['default_billing'] ?? false);

        try {
            $savedAddress = $this->addressRepository->save($address);

            return [
                'address' => [
                    'id' => $savedAddress->getId(),
                    'street' => $savedAddress->getStreet(),
                    'city' => $savedAddress->getCity(),
                ],
                'success' => true,
                'message' => 'Address created successfully',
            ];
        } catch (\Exception $e) {
            return [
                'address' => null,
                'success' => false,
                'message' => $e->getMessage(),
            ];
        }
    }
}
```

**Request:**

```graphql
mutation {
  createCustomerAddress(input: {
    street: ["123 Main St", "Apt 4B"]
    city: "New York"
    region: "NY"
    postcode: "10001"
    country_code: "US"
    telephone: "555-1234"
    default_shipping: true
  }) {
    address {
      id
      street
      city
    }
    success
    message
  }
}
```

---

## Trade-Offs and Design Decisions

### GraphQL vs REST

| Aspect | GraphQL | REST |
|--------|---------|------|
| **Over-fetching** | Client requests only needed fields | Returns full entity |
| **Under-fetching** | Single query for nested data | Multiple endpoints (N+1 HTTP calls) |
| **Caching** | Complex (requires cache identity) | Simple (URL-based) |
| **Tooling** | Strong typing, introspection, playground | OpenAPI/Swagger for docs |
| **Learning curve** | Steeper (schema, resolvers, batching) | Familiar (HTTP verbs, endpoints) |
| **Versioning** | Schema evolution (deprecate fields) | URL versioning (/v1/, /v2/) |

**Verdict:** GraphQL excels for frontend-heavy apps (PWA Studio, mobile), REST for simpler integrations.

### Thin Resolvers vs Fat Resolvers

**Thin Resolvers (Recommended):**
- Resolver validates arguments, checks auth
- Delegates to DataProvider/Service contracts
- Testable, reusable business logic

**Fat Resolvers:**
- Business logic embedded in resolver
- Hard to test (GraphQL objects)
- Cannot reuse in REST, CLI

**Verdict:** Always use thin resolvers + DataProvider pattern.

### N+1 Queries: Acceptable vs Unacceptable

**Acceptable:**
- Nested fields rarely requested (e.g., `product.reviews` not in 90% of queries)
- Parent resolver triggers batch pre-load when nested field detected

**Unacceptable:**
- Common fields (e.g., `product.price`) causing 100+ queries per request
- No batching or caching

**Solution:** Monitor query performance with New Relic/Blackfire, batch-load common fields.

---

## Antipatterns to Avoid

### 1. Direct Model Access in Resolvers

```php
// WRONG
class ProductResolver implements ResolverInterface
{
    public function resolve(/* ... */)
    {
        $product = $this->productFactory->create()->load($args['id']);
        return $product->getData();
    }
}
```

**Why Wrong:** No service contract, no store context, no caching.

**Fix:** Use `ProductRepositoryInterface`.

### 2. Ignoring Store Context

```php
// WRONG
public function resolve(/* ... */)
{
    $product = $this->productRepository->get($args['sku']); // Default store!
}
```

**Why Wrong:** Multi-store setups return wrong prices/names.

**Fix:**

```php
$storeId = (int) $context->getExtensionAttributes()->getStore()->getId();
$product = $this->productRepository->get($args['sku'], false, $storeId);
```

### 3. Returning Raw Models

```php
// WRONG
public function resolve(/* ... */)
{
    return $this->productRepository->get($args['sku']); // Magento\Catalog\Model\Product
}
```

**Why Wrong:** Exposes internal Model methods, breaks schema contract (expects plain array).

**Fix:**

```php
return [
    'id' => $product->getId(),
    'sku' => $product->getSku(),
    'model' => $product, // For nested resolvers
];
```

### 4. No Pagination on Lists

```graphql
# WRONG
type Query {
    products: [Product!]! # Returns ALL products!
}
```

**Why Wrong:** OOM on large catalogs.

**Fix:**

```graphql
type Query {
    products(
        pageSize: Int = 20
        currentPage: Int = 1
    ): ProductSearchResults!
}

type ProductSearchResults {
    items: [Product!]!
    total_count: Int!
    page_info: SearchResultPageInfo!
}
```

### 5. Swallowing Exceptions

```php
// WRONG
try {
    $product = $this->productRepository->get($args['sku']);
} catch (\Exception $e) {
    return null; // Silent failure!
}
```

**Why Wrong:** Client has no idea why query failed.

**Fix:**

```php
try {
    $product = $this->productRepository->get($args['sku']);
} catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
    throw new GraphQlNoSuchEntityException(__('Product not found'));
}
```

---

## Further Reading

### Official Documentation
- [GraphQL Developer Guide](https://developer.adobe.com/commerce/webapi/graphql/) - Adobe Commerce GraphQL docs
- [GraphQL Schema Reference](https://developer.adobe.com/commerce/webapi/graphql/schema/) - Complete schema reference
- [GraphQL Authorization](https://developer.adobe.com/commerce/webapi/graphql/usage/authorization-tokens/) - Customer/admin tokens

### Core Module Examples
- `Magento\CatalogGraphQl` - Product queries, filters, caching
- `Magento\CustomerGraphQl` - Customer mutations, address management
- `Magento\SalesGraphQl` - Order queries, order history
- `Magento\GraphQl` - Base framework classes

### Tools
- [GraphQL Playground](https://github.com/graphql/graphql-playground) - Interactive query builder
- [Altair GraphQL Client](https://altair.sirmuel.design/) - Standalone client with docs explorer
- [Magento 2 GraphQL Schema Stitching](https://devdocs.magento.com/guides/v2.4/graphql/develop/create-graphqls-file.html)

### Performance
- [GraphQL Query Complexity Analysis](https://webonyx.github.io/graphql-php/security/#query-complexity-analysis) - Prevent DoS
- [DataLoader Pattern (JS)](https://github.com/graphql/dataloader) - Batching concept (manual in Magento)

---

## Conclusion

Magento's GraphQL implementation provides a **powerful, flexible API layer** for modern commerce experiences. By mastering resolver patterns, DataProviders, batching, and caching strategies, you can build APIs that are:

- **Fast:** Batch resolvers eliminate N+1 queries
- **Secure:** Authorization checks at resolver level
- **Maintainable:** Thin resolvers + service contracts
- **Cacheable:** FPC integration for public queries

**Key Takeaways:**
1. **Use thin resolvers + DataProvider** for testable, reusable logic
2. **Batch-load nested fields** to avoid N+1 queries
3. **Respect store context** for multi-store setups
4. **Implement proper authorization** (customer token, admin ACL)
5. **Leverage caching** for public queries
6. **Return structured errors** with appropriate exception types
7. **Study core modules** (CatalogGraphQl, CustomerGraphQl) for patterns

GraphQL is Magento's future API strategy—invest in learning it well.

---

**Assumptions:**
- Magento Open Source / Adobe Commerce 2.4.7+
- PHP 8.2+
- webonyx/graphql-php 14.x (bundled with Magento)
- Modules follow PSR-4 autoloading and Magento module structure

**Why This Approach:**
- **Performance:** Batching prevents N+1 queries; FPC caching reduces server load
- **Maintainability:** DataProvider pattern separates GraphQL concerns from business logic
- **Extensibility:** Plugins on service contracts allow third-party modifications
- **Security:** Authorization checks at resolver level; input validation prevents injection
- **Standards:** Follows GraphQL spec and Magento best practices

**Security Impact:**
- **Authorization:** Resolver checks customer token, admin ACL before mutations
- **Input validation:** Throw `GraphQlInputException` for malformed data; prevents SQL injection, XSS
- **Rate limiting:** Use Magento's built-in rate limiter or Varnish rules
- **Query depth/complexity limits:** Prevent DoS via deeply nested queries (configure in `GraphQl\Config`)
- **CSRF:** GraphQL uses token-based auth, no form keys needed (stateless)

**Performance Impact:**
- **Batching:** Reduces 100+ queries to 2-3 queries per request
- **FPC caching:** Public queries cached in Varnish (3600s TTL typical)
- **Lazy loading:** Only resolve requested fields (no over-fetching)
- **N+1 monitoring:** Use Blackfire/New Relic to detect unbatched resolvers
- **Query complexity limits:** Reject queries with complexity > 300 (configurable)

**Backward Compatibility:**
- **Schema evolution:** Add new fields (BC-safe), deprecate old fields (BC-safe), never remove fields (BC-break)
- **Resolver signatures:** Don't change `ResolverInterface::resolve()` parameters
- **Type changes:** Widening return types safe (nullable to non-nullable = BC-break)
- **Input types:** Adding optional fields safe, adding required fields = BC-break
- **Versioning:** Use `@deprecated` directive, maintain old fields for 2-3 minor versions

**Tests to Add:**
- **Unit tests:** Mock DataProviders, verify resolver logic
- **Integration tests (GraphQlTester):** Test full query execution, context, authorization
- **API functional tests:** Verify requests/responses match schema
- **Performance tests (Blackfire):** Assert query count < threshold, response time < 500ms
- **Security tests:** Verify unauthorized access throws `GraphQlAuthorizationException`

**Docs to Update:**
- **README:** GraphQL endpoints, authentication, example queries
- **CHANGELOG:** New queries, mutations, breaking changes
- **schema.graphqls:** Inline `@doc` descriptions for introspection
- **Admin user guide:** Screenshots of using GraphQL in Postman/Altair
- **API reference:** Generate from schema with graphql-markdown or Spectaql

## Related Documentation

### Related Guides

- [Service Contracts vs Repositories in Magento 2](service-contracts-repositories.md)
- [Full Page Cache Strategy for High-Performance Magento](../how-to/full-page-cache-strategy.md)
- [Security Checklist for Custom Modules](../how-to/security-checklist.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Customer Overview](../../modules/customer/README.md)
- [Magento_Checkout Overview](../../modules/checkout/README.md)
