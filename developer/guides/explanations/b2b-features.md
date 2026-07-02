---
title: "B2B Features Development"
description: "Master Adobe Commerce B2B architecture including company accounts, shared catalogs, requisition lists, quick order, quote negotiation, and B2B-specific permissions."
type: "explanation"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "60 minutes"
topics:
  - b2b
  - company accounts
  - shared catalogs
  - requisition lists
  - quick order
  - quote negotiation
  - b2b permissions
last_updated: "2026-02-07"
---

# B2B Features Development

## Overview

Adobe Commerce B2B extends the platform with enterprise features for business-to-business commerce. Understanding the B2B architecture is critical for implementing custom workflows, integrations, and business logic for corporate purchasing environments.

**What you'll learn:**
- Company account architecture and hierarchy management
- Shared catalog implementation and product visibility control
- Requisition list functionality and custom list types
- Quick order implementation and bulk ordering workflows
- Quote negotiation system and approval workflows
- B2B-specific permission models (roles, rules, ACL)
- Credit limit management and purchase order workflows
- Integration patterns for ERP and procurement systems

**Prerequisites:**
- Adobe Commerce 2.4.7+ with B2B module enabled
- Advanced understanding of Magento customer management
- Experience with catalog, pricing, and cart systems
- Knowledge of ACL, permissions, and authorization
- Familiarity with quote and order workflows

**Note:** B2B features are **Adobe Commerce only** and not available in Open Source.

---

## B2B Architecture Overview

### Core B2B Modules

```
Magento_Company           - Company account management
Magento_CompanyCredit     - Credit limit and payment on account
Magento_SharedCatalog     - Catalog visibility and custom pricing
Magento_RequisitionList   - Saved shopping lists
Magento_NegotiableQuote   - Quote negotiation workflow
Magento_QuickOrder        - Bulk SKU entry
Magento_PurchaseOrder     - Approval workflows
Magento_B2b               - Integration layer
```

### B2B Customer Hierarchy

```
Company (ABC Corporation)
  ├── Company Admin (john@abc.com)
  ├── Team Lead (sarah@abc.com)
  │   ├── Buyer 1 (mike@abc.com)
  │   └── Buyer 2 (lisa@abc.com)
  └── Finance Manager (david@abc.com)
```

### Key Concepts

- **Company Account**: Organization entity with admin, structure, and users
- **Shared Catalog**: Custom product catalog and pricing per company
- **Requisition List**: Reusable shopping lists for frequent purchases
- **Negotiable Quote**: Request-for-quote workflow with admin negotiation
- **Purchase Order**: Approval rules based on amount, requester, approvers
- **Company Credit**: Credit limit and payment-on-account capability

---

## Company Account Architecture

### Company Entity Structure

**Database Schema:**

```
company
  ├── company_id (PK)
  ├── company_name
  ├── legal_name
  ├── company_email
  ├── status (0=pending, 1=approved, 2=rejected, 3=blocked)
  ├── sales_representative_id
  ├── customer_group_id
  ├── super_user_id (company admin customer_id)
  └── ...

company_advanced_customer_entity
  ├── entity_id (PK)
  ├── customer_id (FK to customer_entity)
  ├── company_id (FK to company)
  ├── job_title
  ├── status (0=inactive, 1=active)
  └── telephone

company_structure
  ├── structure_id (PK)
  ├── parent_id (self-referencing)
  ├── entity_id (customer_id)
  ├── entity_type (0=customer, 1=team)
  ├── path (materialized path for hierarchy)
  └── level
```

### Creating a Company Programmatically

**File**: `Vendor/B2BExtension/Model/CompanyCreator.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\B2BExtension\Model;

use Magento\Company\Api\CompanyRepositoryInterface;
use Magento\Company\Api\Data\CompanyInterface;
use Magento\Company\Api\Data\CompanyInterfaceFactory;
use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Exception\LocalizedException;
use Psr\Log\LoggerInterface;

class CompanyCreator
{
    public function __construct(
        private readonly CompanyInterfaceFactory $companyFactory,
        private readonly CompanyRepositoryInterface $companyRepository,
        private readonly CustomerRepositoryInterface $customerRepository,
        private readonly LoggerInterface $logger
    ) {
    }

    /**
     * Create company with admin user
     *
     * @param array $companyData
     * @param CustomerInterface $adminCustomer
     * @return CompanyInterface
     * @throws LocalizedException
     */
    public function createCompany(array $companyData, CustomerInterface $adminCustomer): CompanyInterface
    {
        try {
            // Validate required fields
            $this->validateCompanyData($companyData);

            /** @var CompanyInterface $company */
            $company = $this->companyFactory->create();

            $company->setCompanyName($companyData['company_name'])
                ->setLegalName($companyData['legal_name'] ?? $companyData['company_name'])
                ->setCompanyEmail($companyData['company_email'])
                ->setStatus(CompanyInterface::STATUS_PENDING)
                ->setSuperUserId((int)$adminCustomer->getId())
                ->setCustomerGroupId((int)($companyData['customer_group_id'] ?? 1))
                ->setComment($companyData['comment'] ?? '');

            // Set address data
            if (isset($companyData['street'])) {
                $company->setStreet($companyData['street']);
            }
            if (isset($companyData['city'])) {
                $company->setCity($companyData['city']);
            }
            if (isset($companyData['country_id'])) {
                $company->setCountryId($companyData['country_id']);
            }
            if (isset($companyData['region_id'])) {
                $company->setRegionId($companyData['region_id']);
            }
            if (isset($companyData['postcode'])) {
                $company->setPostcode($companyData['postcode']);
            }
            if (isset($companyData['telephone'])) {
                $company->setTelephone($companyData['telephone']);
            }

            // Save company
            $savedCompany = $this->companyRepository->save($company);

            $this->logger->info('Company created successfully', [
                'company_id' => $savedCompany->getId(),
                'company_name' => $savedCompany->getCompanyName()
            ]);

            return $savedCompany;

        } catch (\Exception $e) {
            $this->logger->error('Company creation failed', [
                'exception' => $e->getMessage(),
                'company_data' => $companyData
            ]);
            throw new LocalizedException(__('Failed to create company: %1', $e->getMessage()));
        }
    }

    /**
     * Approve company
     */
    public function approveCompany(int $companyId): void
    {
        $company = $this->companyRepository->get($companyId);
        $company->setStatus(CompanyInterface::STATUS_APPROVED);
        $this->companyRepository->save($company);
    }

    /**
     * Add user to company
     */
    public function addCompanyUser(
        int $companyId,
        CustomerInterface $customer,
        ?int $parentId = null,
        string $jobTitle = ''
    ): void {
        $company = $this->companyRepository->get($companyId);

        // Set company association in customer extension attributes
        $extensionAttributes = $customer->getExtensionAttributes();
        $extensionAttributes->setCompanyAttributes([
            'company_id' => $companyId,
            'status' => 1, // Active
            'job_title' => $jobTitle,
        ]);

        $customer->setExtensionAttributes($extensionAttributes);
        $this->customerRepository->save($customer);
    }

    /**
     * Validate company data
     */
    private function validateCompanyData(array $data): void
    {
        $requiredFields = ['company_name', 'company_email'];

        foreach ($requiredFields as $field) {
            if (empty($data[$field])) {
                throw new LocalizedException(__('Required field missing: %1', $field));
            }
        }

        if (!filter_var($data['company_email'], FILTER_VALIDATE_EMAIL)) {
            throw new LocalizedException(__('Invalid company email format'));
        }
    }
}
```

### Company Hierarchy Management

**File**: `Vendor/B2BExtension/Model/CompanyStructure.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\B2BExtension\Model;

use Magento\Company\Api\Data\StructureInterface;
use Magento\Company\Model\Company\Structure as CompanyStructure;
use Magento\Framework\Exception\LocalizedException;

class CompanyStructureManager
{
    public function __construct(
        private readonly CompanyStructure $companyStructure
    ) {
    }

    /**
     * Create team under parent
     *
     * @param int $companyId
     * @param int $parentStructureId
     * @param string $teamName
     * @return StructureInterface
     */
    public function createTeam(int $companyId, int $parentStructureId, string $teamName): StructureInterface
    {
        $team = $this->companyStructure->addNode(
            0, // Team entity (not customer)
            $parentStructureId,
            $teamName
        );

        return $team;
    }

    /**
     * Move user to different team/parent
     */
    public function moveUser(int $customerId, int $newParentId): void
    {
        $userNode = $this->companyStructure->getStructureByCustomerId($customerId);

        if (!$userNode) {
            throw new LocalizedException(__('User structure not found'));
        }

        $this->companyStructure->moveNode($userNode->getId(), $newParentId);
    }

    /**
     * Get all users under a parent (recursive)
     *
     * @param int $parentStructureId
     * @return array
     */
    public function getAllChildUsers(int $parentStructureId): array
    {
        $children = $this->companyStructure->getAllowedChildrenIds($parentStructureId);

        $users = [];
        foreach ($children as $childId) {
            $node = $this->companyStructure->getStructureById($childId);

            if ($node->getEntityType() == StructureInterface::TYPE_CUSTOMER) {
                $users[] = $node->getEntityId(); // customer_id
            }

            // Recursively get children
            $users = array_merge($users, $this->getAllChildUsers($childId));
        }

        return array_unique($users);
    }

    /**
     * Get user's direct manager
     */
    public function getUserManager(int $customerId): ?int
    {
        $userNode = $this->companyStructure->getStructureByCustomerId($customerId);

        if (!$userNode || !$userNode->getParentId()) {
            return null;
        }

        $parentNode = $this->companyStructure->getStructureById($userNode->getParentId());

        // Find first customer-type parent
        while ($parentNode && $parentNode->getEntityType() != StructureInterface::TYPE_CUSTOMER) {
            if (!$parentNode->getParentId()) {
                return null;
            }
            $parentNode = $this->companyStructure->getStructureById($parentNode->getParentId());
        }

        return $parentNode ? $parentNode->getEntityId() : null;
    }
}
```

---

## Shared Catalog Implementation

### Shared Catalog Architecture

Shared catalogs control:
1. **Product Visibility**: Which products a company can see
2. **Custom Pricing**: Tier prices specific to company/catalog
3. **Category Access**: Category visibility restrictions

### Creating a Shared Catalog

**File**: `Vendor/B2BExtension/Model/SharedCatalogCreator.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\B2BExtension\Model;

use Magento\SharedCatalog\Api\Data\SharedCatalogInterface;
use Magento\SharedCatalog\Api\Data\SharedCatalogInterfaceFactory;
use Magento\SharedCatalog\Api\SharedCatalogRepositoryInterface;
use Magento\SharedCatalog\Model\SharedCatalogAssignment;
use Magento\Framework\Exception\LocalizedException;

class SharedCatalogCreator
{
    public function __construct(
        private readonly SharedCatalogInterfaceFactory $sharedCatalogFactory,
        private readonly SharedCatalogRepositoryInterface $sharedCatalogRepository,
        private readonly SharedCatalogAssignment $sharedCatalogAssignment,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {
    }

    /**
     * Create shared catalog for company
     *
     * @param string $name
     * @param int $customerGroupId
     * @param int $createdBy
     * @return SharedCatalogInterface
     */
    public function create(string $name, int $customerGroupId, int $createdBy): SharedCatalogInterface
    {
        try {
            /** @var SharedCatalogInterface $sharedCatalog */
            $sharedCatalog = $this->sharedCatalogFactory->create();

            $sharedCatalog->setName($name)
                ->setDescription('Shared catalog for ' . $name)
                ->setType(SharedCatalogInterface::TYPE_CUSTOM)
                ->setCreatedBy($createdBy)
                ->setCustomerGroupId($customerGroupId)
                ->setTaxClassId(3); // Default product tax class

            $savedCatalog = $this->sharedCatalogRepository->save($sharedCatalog);

            $this->logger->info('Shared catalog created', [
                'catalog_id' => $savedCatalog->getId(),
                'name' => $savedCatalog->getName()
            ]);

            return $savedCatalog;

        } catch (\Exception $e) {
            $this->logger->error('Shared catalog creation failed', [
                'exception' => $e->getMessage()
            ]);
            throw new LocalizedException(__('Failed to create shared catalog'));
        }
    }

    /**
     * Assign products to shared catalog
     *
     * @param int $sharedCatalogId
     * @param array $productSkus
     */
    public function assignProducts(int $sharedCatalogId, array $productSkus): void
    {
        $this->sharedCatalogAssignment->assignProductsForCatalog(
            $sharedCatalogId,
            $productSkus
        );
    }

    /**
     * Assign categories to shared catalog
     *
     * @param int $sharedCatalogId
     * @param array $categoryIds
     */
    public function assignCategories(int $sharedCatalogId, array $categoryIds): void
    {
        $this->sharedCatalogAssignment->assignCategoriesForCatalog(
            $sharedCatalogId,
            $categoryIds
        );
    }

    /**
     * Unassign products from shared catalog
     */
    public function unassignProducts(int $sharedCatalogId, array $productSkus): void
    {
        $this->sharedCatalogAssignment->unassignProductsForCatalog(
            $sharedCatalogId,
            $productSkus
        );
    }
}
```

### Custom Pricing for Shared Catalog

**File**: `Vendor/B2BExtension/Model/SharedCatalogPricing.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\B2BExtension\Model;

use Magento\SharedCatalog\Api\ProductItemManagementInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;

class SharedCatalogPricing
{
    public function __construct(
        private readonly ProductItemManagementInterface $productItemManagement,
        private readonly ProductRepositoryInterface $productRepository
    ) {
    }

    /**
     * Set custom price for product in shared catalog
     *
     * @param int $sharedCatalogId
     * @param string $sku
     * @param float $customPrice
     */
    public function setCustomPrice(int $sharedCatalogId, string $sku, float $customPrice): void
    {
        $this->productItemManagement->addItems(
            $sharedCatalogId,
            [
                [
                    'sku' => $sku,
                    'custom_price' => $customPrice
                ]
            ]
        );
    }

    /**
     * Set tier prices for shared catalog
     *
     * @param int $sharedCatalogId
     * @param string $sku
     * @param array $tierPrices [['qty' => 10, 'price' => 8.50], ...]
     */
    public function setTierPricing(int $sharedCatalogId, string $sku, array $tierPrices): void
    {
        $product = $this->productRepository->get($sku);

        $existingTierPrices = $product->getTierPrices() ?? [];

        // Add shared catalog tier prices
        foreach ($tierPrices as $tierPrice) {
            $existingTierPrices[] = [
                'website_id' => 0,
                'customer_group_id' => $this->getCustomerGroupId($sharedCatalogId),
                'qty' => $tierPrice['qty'],
                'value' => $tierPrice['price'],
                'percentage_value' => null,
            ];
        }

        $product->setTierPrices($existingTierPrices);
        $this->productRepository->save($product);
    }

    /**
     * Apply percentage discount to all products in catalog
     */
    public function applyGlobalDiscount(int $sharedCatalogId, float $discountPercent): void
    {
        // Implementation would iterate through assigned products
        // and apply percentage discount
    }

    private function getCustomerGroupId(int $sharedCatalogId): int
    {
        // Retrieve customer group ID from shared catalog
        return 0; // Simplified
    }
}
```

---

## Requisition Lists

### Requisition List Architecture

Requisition lists are persistent shopping lists that can be:
- Created by company users
- Shared within company hierarchy
- Used for recurring purchases
- Quickly added to cart

### Custom Requisition List Type

**File**: `Vendor/B2BExtension/Model/RequisitionList/CustomList.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\B2BExtension\Model\RequisitionList;

use Magento\RequisitionList\Api\Data\RequisitionListInterface;
use Magento\RequisitionList\Api\RequisitionListRepositoryInterface;
use Magento\RequisitionList\Model\RequisitionListItem\SaveHandler;
use Magento\Framework\Exception\LocalizedException;

class CustomListManager
{
    public function __construct(
        private readonly RequisitionListRepositoryInterface $requisitionListRepository,
        private readonly SaveHandler $itemSaveHandler,
        private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository
    ) {
    }

    /**
     * Create recurring order list
     *
     * @param int $customerId
     * @param string $name
     * @param string $description
     * @return RequisitionListInterface
     */
    public function createRecurringList(
        int $customerId,
        string $name,
        string $description = ''
    ): RequisitionListInterface {
        /** @var RequisitionListInterface $list */
        $list = $this->requisitionListRepository->create();

        $list->setCustomerId($customerId)
            ->setName($name)
            ->setDescription($description);

        return $this->requisitionListRepository->save($list);
    }

    /**
     * Add product to requisition list
     *
     * @param int $listId
     * @param string $sku
     * @param float $qty
     * @param array $options
     */
    public function addProduct(
        int $listId,
        string $sku,
        float $qty = 1.0,
        array $options = []
    ): void {
        $product = $this->productRepository->get($sku);

        $item = [
            'requisition_list_id' => $listId,
            'sku' => $sku,
            'qty' => $qty,
            'options' => $options,
        ];

        $this->itemSaveHandler->saveItem($item, $product);
    }

    /**
     * Add entire list to cart
     *
     * @param int $listId
     * @param \Magento\Quote\Api\CartRepositoryInterface $cartRepository
     * @param int $customerId
     */
    public function addListToCart(
        int $listId,
        \Magento\Quote\Api\CartRepositoryInterface $cartRepository,
        int $customerId
    ): void {
        $list = $this->requisitionListRepository->get($listId);

        if ($list->getCustomerId() != $customerId) {
            throw new LocalizedException(__('Unauthorized access to requisition list'));
        }

        $quote = $cartRepository->getActiveForCustomer($customerId);

        foreach ($list->getItems() as $item) {
            try {
                $product = $this->productRepository->get($item->getSku());

                $buyRequest = new \Magento\Framework\DataObject([
                    'qty' => $item->getQty(),
                    'options' => $item->getOptions(),
                ]);

                $quote->addProduct($product, $buyRequest);

            } catch (\Exception $e) {
                // Log error but continue with other items
                continue;
            }
        }

        $cartRepository->save($quote);
    }

    /**
     * Clone requisition list
     */
    public function cloneList(int $listId, int $targetCustomerId, string $newName): RequisitionListInterface
    {
        $sourceList = $this->requisitionListRepository->get($listId);

        $newList = $this->createRecurringList(
            $targetCustomerId,
            $newName,
            $sourceList->getDescription()
        );

        foreach ($sourceList->getItems() as $item) {
            $this->addProduct(
                (int)$newList->getId(),
                $item->getSku(),
                (float)$item->getQty(),
                $item->getOptions() ?? []
            );
        }

        return $newList;
    }
}
```

---

## Quick Order Implementation

### Quick Order Architecture

Quick Order allows bulk SKU entry with quantity for rapid ordering.

**File**: `Vendor/B2BExtension/Model/QuickOrder/Processor.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\B2BExtension\Model\QuickOrder;

use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Framework\Exception\NoSuchEntityException;

class Processor
{
    private array $errors = [];
    private array $successful = [];

    public function __construct(
        private readonly ProductRepositoryInterface $productRepository,
        private readonly CartRepositoryInterface $cartRepository
    ) {
    }

    /**
     * Process bulk SKU list
     *
     * @param array $items [['sku' => 'ABC', 'qty' => 5], ...]
     * @param int $customerId
     * @return array ['successful' => [...], 'errors' => [...]]
     */
    public function processBulkOrder(array $items, int $customerId): array
    {
        $this->errors = [];
        $this->successful = [];

        $quote = $this->cartRepository->getActiveForCustomer($customerId);

        foreach ($items as $item) {
            try {
                $this->processItem($item, $quote);
                $this->successful[] = $item['sku'];

            } catch (\Exception $e) {
                $this->errors[] = [
                    'sku' => $item['sku'],
                    'error' => $e->getMessage()
                ];
            }
        }

        if (!empty($this->successful)) {
            $this->cartRepository->save($quote);
        }

        return [
            'successful' => $this->successful,
            'errors' => $this->errors
        ];
    }

    /**
     * Process single item
     */
    private function processItem(array $item, $quote): void
    {
        if (empty($item['sku'])) {
            throw new \InvalidArgumentException('SKU is required');
        }

        $qty = $item['qty'] ?? 1;

        if ($qty <= 0) {
            throw new \InvalidArgumentException('Quantity must be greater than 0');
        }

        try {
            $product = $this->productRepository->get($item['sku']);

            if (!$product->isSalable()) {
                throw new \RuntimeException('Product is not available for purchase');
            }

            $buyRequest = new \Magento\Framework\DataObject(['qty' => $qty]);
            $quote->addProduct($product, $buyRequest);

        } catch (NoSuchEntityException $e) {
            throw new \RuntimeException('Product not found: ' . $item['sku']);
        }
    }

    /**
     * Parse CSV input for quick order
     *
     * @param string $csvContent
     * @return array
     */
    public function parseCsvInput(string $csvContent): array
    {
        $lines = explode("\n", trim($csvContent));
        $items = [];

        foreach ($lines as $line) {
            $parts = str_getcsv($line);

            if (count($parts) < 1 || empty(trim($parts[0]))) {
                continue;
            }

            $items[] = [
                'sku' => trim($parts[0]),
                'qty' => isset($parts[1]) ? (float)trim($parts[1]) : 1.0,
            ];
        }

        return $items;
    }

    /**
     * Validate SKUs before processing
     */
    public function validateSkus(array $skus): array
    {
        $validation = [
            'valid' => [],
            'invalid' => [],
        ];

        foreach ($skus as $sku) {
            try {
                $product = $this->productRepository->get($sku);
                $validation['valid'][] = [
                    'sku' => $sku,
                    'name' => $product->getName(),
                    'price' => $product->getFinalPrice(),
                ];
            } catch (NoSuchEntityException $e) {
                $validation['invalid'][] = $sku;
            }
        }

        return $validation;
    }
}
```

---

## Quote Negotiation Workflow

### Negotiable Quote Architecture

Flow:
1. Customer creates quote request
2. Admin/Sales rep reviews and modifies
3. Customer accepts/declines/negotiates
4. Approved quote converts to order

### Quote Management

**File**: `Vendor/B2BExtension/Model/Quote/NegotiableQuoteManager.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\B2BExtension\Model\Quote;

use Magento\NegotiableQuote\Api\NegotiableQuoteRepositoryInterface;
use Magento\NegotiableQuote\Api\Data\NegotiableQuoteInterface;
use Magento\Quote\Api\CartRepositoryInterface;
use Magento\Framework\Exception\LocalizedException;

class NegotiableQuoteManager
{
    public function __construct(
        private readonly NegotiableQuoteRepositoryInterface $negotiableQuoteRepository,
        private readonly CartRepositoryInterface $cartRepository,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {
    }

    /**
     * Create negotiable quote from cart
     *
     * @param int $quoteId
     * @param string $quoteName
     * @param string $comment
     * @return NegotiableQuoteInterface
     */
    public function createNegotiableQuote(
        int $quoteId,
        string $quoteName,
        string $comment = ''
    ): NegotiableQuoteInterface {
        $quote = $this->cartRepository->get($quoteId);

        if (!$quote->getItemsCount()) {
            throw new LocalizedException(__('Cannot create quote from empty cart'));
        }

        // Create negotiable quote
        $negotiableQuote = $quote->getExtensionAttributes()->getNegotiableQuote();

        if (!$negotiableQuote) {
            throw new LocalizedException(__('Quote is not negotiable'));
        }

        $negotiableQuote->setQuoteName($quoteName)
            ->setStatus(NegotiableQuoteInterface::STATUS_CREATED)
            ->setIsRegularQuote(true)
            ->setNegotiatedPriceValue(null);

        if ($comment) {
            $negotiableQuote->setCreatorId($quote->getCustomerId())
                ->setCreatorType(NegotiableQuoteInterface::CREATOR_TYPE_BUYER);
        }

        $this->negotiableQuoteRepository->save($negotiableQuote);

        $this->logger->info('Negotiable quote created', [
            'quote_id' => $quoteId,
            'quote_name' => $quoteName
        ]);

        return $negotiableQuote;
    }

    /**
     * Apply admin discount to negotiable quote
     *
     * @param int $quoteId
     * @param float $discountPercent
     * @param string $comment
     */
    public function applyAdminDiscount(
        int $quoteId,
        float $discountPercent,
        string $comment = ''
    ): void {
        $quote = $this->cartRepository->get($quoteId);
        $negotiableQuote = $quote->getExtensionAttributes()->getNegotiableQuote();

        if (!$negotiableQuote) {
            throw new LocalizedException(__('Quote is not negotiable'));
        }

        // Calculate new total
        $originalPrice = (float)$quote->getBaseGrandTotal();
        $discountAmount = $originalPrice * ($discountPercent / 100);
        $newPrice = $originalPrice - $discountAmount;

        $negotiableQuote->setNegotiatedPriceType(
            NegotiableQuoteInterface::NEGOTIATED_PRICE_TYPE_PERCENTAGE_DISCOUNT
        )
            ->setNegotiatedPriceValue($discountPercent)
            ->setStatus(NegotiableQuoteInterface::STATUS_SUBMITTED_BY_ADMIN);

        $this->negotiableQuoteRepository->save($negotiableQuote);

        $this->logger->info('Admin discount applied to quote', [
            'quote_id' => $quoteId,
            'discount_percent' => $discountPercent,
            'original_price' => $originalPrice,
            'new_price' => $newPrice
        ]);
    }

    /**
     * Customer accepts quote
     */
    public function acceptQuote(int $quoteId, int $customerId): void
    {
        $quote = $this->cartRepository->get($quoteId);

        if ($quote->getCustomerId() != $customerId) {
            throw new LocalizedException(__('Unauthorized access to quote'));
        }

        $negotiableQuote = $quote->getExtensionAttributes()->getNegotiableQuote();

        $negotiableQuote->setStatus(NegotiableQuoteInterface::STATUS_ORDERED);

        $this->negotiableQuoteRepository->save($negotiableQuote);
    }

    /**
     * Customer declines quote
     */
    public function declineQuote(int $quoteId, int $customerId, string $reason = ''): void
    {
        $quote = $this->cartRepository->get($quoteId);

        if ($quote->getCustomerId() != $customerId) {
            throw new LocalizedException(__('Unauthorized access to quote'));
        }

        $negotiableQuote = $quote->getExtensionAttributes()->getNegotiableQuote();

        $negotiableQuote->setStatus(NegotiableQuoteInterface::STATUS_DECLINED);

        $this->negotiableQuoteRepository->save($negotiableQuote);
    }

    /**
     * Get quote history/comments
     */
    public function getQuoteHistory(int $quoteId): array
    {
        // Retrieve comment history from negotiable_quote_history table
        // Simplified implementation
        return [];
    }
}
```

---

## Purchase Order Approval Workflow

### Purchase Order Rules

**File**: `Vendor/B2BExtension/Model/PurchaseOrder/ApprovalRule.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\B2BExtension\Model\PurchaseOrder;

use Magento\PurchaseOrder\Api\Data\PurchaseOrderInterface;
use Magento\Framework\Exception\LocalizedException;

class ApprovalRuleEngine
{
    /**
     * Check if purchase order requires approval
     *
     * @param PurchaseOrderInterface $purchaseOrder
     * @return bool
     */
    public function requiresApproval(PurchaseOrderInterface $purchaseOrder): bool
    {
        $rules = $this->getApprovalRules($purchaseOrder->getCompanyId());

        foreach ($rules as $rule) {
            if ($this->evaluateRule($rule, $purchaseOrder)) {
                return true;
            }
        }

        return false;
    }

    /**
     * Get approval rules for company
     */
    private function getApprovalRules(int $companyId): array
    {
        // Example rules
        return [
            [
                'type' => 'amount_threshold',
                'condition' => 'greater_than',
                'value' => 1000.00,
                'approvers' => [2, 5], // Customer IDs
            ],
            [
                'type' => 'requester_role',
                'condition' => 'equals',
                'value' => 'junior_buyer',
                'approvers' => [3], // Manager
            ],
        ];
    }

    /**
     * Evaluate single rule
     */
    private function evaluateRule(array $rule, PurchaseOrderInterface $purchaseOrder): bool
    {
        switch ($rule['type']) {
            case 'amount_threshold':
                return $this->evaluateAmountThreshold($rule, $purchaseOrder);

            case 'requester_role':
                return $this->evaluateRequesterRole($rule, $purchaseOrder);

            default:
                return false;
        }
    }

    /**
     * Check if amount exceeds threshold
     */
    private function evaluateAmountThreshold(array $rule, PurchaseOrderInterface $purchaseOrder): bool
    {
        $amount = (float)$purchaseOrder->getGrandTotal();
        $threshold = (float)$rule['value'];

        return match ($rule['condition']) {
            'greater_than' => $amount > $threshold,
            'less_than' => $amount < $threshold,
            'equals' => $amount == $threshold,
            default => false,
        };
    }

    /**
     * Check requester role
     */
    private function evaluateRequesterRole(array $rule, PurchaseOrderInterface $purchaseOrder): bool
    {
        // Get creator's role from company structure
        $creatorRole = $this->getCreatorRole($purchaseOrder->getCreatorId());

        return $creatorRole === $rule['value'];
    }

    /**
     * Get required approvers for purchase order
     */
    public function getRequiredApprovers(PurchaseOrderInterface $purchaseOrder): array
    {
        $approvers = [];
        $rules = $this->getApprovalRules($purchaseOrder->getCompanyId());

        foreach ($rules as $rule) {
            if ($this->evaluateRule($rule, $purchaseOrder)) {
                $approvers = array_merge($approvers, $rule['approvers']);
            }
        }

        return array_unique($approvers);
    }

    /**
     * Approve purchase order
     */
    public function approve(PurchaseOrderInterface $purchaseOrder, int $approverId): void
    {
        $requiredApprovers = $this->getRequiredApprovers($purchaseOrder);

        if (!in_array($approverId, $requiredApprovers)) {
            throw new LocalizedException(__('User not authorized to approve this purchase order'));
        }

        // Mark as approved by this approver
        // Check if all required approvers have approved
        // If yes, change status to approved
    }

    private function getCreatorRole(int $customerId): string
    {
        // Retrieve from company_advanced_customer_entity
        return 'junior_buyer'; // Simplified
    }
}
```

---

## Company Credit Management

### Credit Limit and Balance

**File**: `Vendor/B2BExtension/Model/CompanyCredit/CreditManager.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\B2BExtension\Model\CompanyCredit;

use Magento\CompanyCredit\Api\CreditLimitRepositoryInterface;
use Magento\CompanyCredit\Api\Data\CreditLimitInterface;
use Magento\Framework\Exception\LocalizedException;

class CreditManager
{
    public function __construct(
        private readonly CreditLimitRepositoryInterface $creditLimitRepository
    ) {
    }

    /**
     * Set credit limit for company
     *
     * @param int $companyId
     * @param float $creditLimit
     * @param string $currencyCode
     */
    public function setCreditLimit(int $companyId, float $creditLimit, string $currencyCode = 'USD'): void
    {
        $credit = $this->getCreditByCompanyId($companyId);

        $credit->setCreditLimit($creditLimit)
            ->setCurrencyCode($currencyCode);

        $this->creditLimitRepository->save($credit);
    }

    /**
     * Get available credit
     */
    public function getAvailableCredit(int $companyId): float
    {
        $credit = $this->getCreditByCompanyId($companyId);

        $creditLimit = (float)$credit->getCreditLimit();
        $balance = (float)$credit->getBalance();

        return $creditLimit - $balance;
    }

    /**
     * Check if company can purchase with credit
     */
    public function canPurchase(int $companyId, float $amount): bool
    {
        $availableCredit = $this->getAvailableCredit($companyId);

        return $availableCredit >= $amount;
    }

    /**
     * Allocate credit for purchase
     */
    public function allocateCredit(int $companyId, float $amount): void
    {
        if (!$this->canPurchase($companyId, $amount)) {
            throw new LocalizedException(__('Insufficient credit limit'));
        }

        $credit = $this->getCreditByCompanyId($companyId);

        $newBalance = (float)$credit->getBalance() + $amount;
        $credit->setBalance($newBalance);

        $this->creditLimitRepository->save($credit);
    }

    /**
     * Refund credit
     */
    public function refundCredit(int $companyId, float $amount): void
    {
        $credit = $this->getCreditByCompanyId($companyId);

        $newBalance = max(0, (float)$credit->getBalance() - $amount);
        $credit->setBalance($newBalance);

        $this->creditLimitRepository->save($credit);
    }

    /**
     * Get credit history
     */
    public function getCreditHistory(int $companyId): array
    {
        // Retrieve from company_credit_history table
        // Simplified
        return [];
    }

    private function getCreditByCompanyId(int $companyId): CreditLimitInterface
    {
        return $this->creditLimitRepository->get($companyId);
    }
}
```

---

## B2B Permissions and ACL

### Custom Company Roles

**File**: `Vendor/B2BExtension/Model/Authorization/RoleManager.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\B2BExtension\Model\Authorization;

use Magento\Company\Api\AclInterface;

class RoleManager
{
    public function __construct(
        private readonly AclInterface $acl
    ) {
    }

    /**
     * Check if user has permission
     *
     * @param int $customerId
     * @param string $resource
     * @return bool
     */
    public function hasPermission(int $customerId, string $resource): bool
    {
        return $this->acl->isAllowed($customerId, $resource);
    }

    /**
     * Define custom B2B resources
     */
    public function getB2BResources(): array
    {
        return [
            'Magento_Company::view',
            'Magento_Company::view_account',
            'Magento_Company::edit_account',
            'Magento_Company::view_address',
            'Magento_Company::edit_address',
            'Magento_Company::contacts',
            'Magento_Company::user_management',
            'Magento_Company::roles_view',
            'Magento_Company::roles_edit',
            'Magento_Sales::all',
            'Magento_Sales::place_order',
            'Magento_Sales::view_orders',
            'Magento_Sales::view_orders_sub',
            'Magento_NegotiableQuote::all',
            'Magento_NegotiableQuote::manage',
            'Magento_NegotiableQuote::checkout',
            'Magento_PurchaseOrder::all',
            'Magento_PurchaseOrder::view_purchase_orders',
            'Magento_PurchaseOrder::view_purchase_orders_for_subordinates',
            'Magento_PurchaseOrder::approve_purchase_orders',
        ];
    }
}
```

---

## Testing B2B Features

### Integration Test

**File**: `Vendor/B2BExtension/Test/Integration/Model/CompanyCreatorTest.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\B2BExtension\Test\Integration\Model;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

class CompanyCreatorTest extends TestCase
{
    private $objectManager;
    private $companyCreator;

    protected function setUp(): void
    {
        $this->objectManager = Bootstrap::getObjectManager();
        $this->companyCreator = $this->objectManager->get(
            \Vendor\B2BExtension\Model\CompanyCreator::class
        );
    }

    /**
     * @magentoDataFixture Magento/Customer/_files/customer.php
     */
    public function testCreateCompanySuccess(): void
    {
        $customerRepository = $this->objectManager->get(
            \Magento\Customer\Api\CustomerRepositoryInterface::class
        );

        $customer = $customerRepository->get('customer@example.com');

        $companyData = [
            'company_name' => 'Test Company',
            'legal_name' => 'Test Company LLC',
            'company_email' => 'company@test.com',
            'street' => '123 Main St',
            'city' => 'Los Angeles',
            'country_id' => 'US',
            'region_id' => 12,
            'postcode' => '90001',
            'telephone' => '555-1234',
        ];

        $company = $this->companyCreator->createCompany($companyData, $customer);

        $this->assertNotNull($company->getId());
        $this->assertEquals('Test Company', $company->getCompanyName());
        $this->assertEquals($customer->getId(), $company->getSuperUserId());
    }
}
```

---

## Summary

**Key Takeaways:**
- Company accounts provide hierarchical organization structure with admin and user roles
- Shared catalogs enable custom product visibility and pricing per company/customer group
- Requisition lists support recurring purchases and bulk ordering workflows
- Quick order facilitates rapid SKU-based ordering for experienced buyers
- Negotiable quotes support request-for-quote workflows with admin negotiation
- Purchase orders enforce approval rules based on amount, requester, or custom logic
- Company credit provides payment-on-account capability with credit limits
- B2B permissions system extends Magento ACL for company-specific resources

---

## Assumptions

- **Magento Version:** Adobe Commerce 2.4.7+ with B2B modules enabled
- **PHP Version:** 8.2+
- **License:** Valid Adobe Commerce license required for B2B features
- **Company Structure:** Hierarchical organization with defined roles
- **Integration:** May require ERP/CRM integration for credit, pricing sync

## Why This Approach

- **Service Contracts:** All B2B features use repository interfaces for API stability
- **Hierarchy Model:** Materialized path in company_structure enables efficient tree queries
- **Shared Catalog:** Catalog visibility and pricing decoupled from core catalog
- **Quote Workflow:** Stateful negotiation with history tracking
- **Approval Rules:** Configurable rule engine supports custom business logic
- **Credit Management:** Separate credit ledger with transaction history

## Security Impact

- **Company Isolation:** Strict validation of company ID on all operations
- **Authorization Checks:** ACL verification for company resources
- **Quote Access:** Users can only access quotes for their company
- **Approval Permissions:** Purchase order approvers validated against rules
- **Credit Limits:** Server-side validation prevents over-spending
- **PII Protection:** Company and user data subject to GDPR compliance

## Performance Impact

- **Company Structure Queries:** Materialized path enables efficient subtree queries
- **Shared Catalog Filtering:** Additional joins on product collections; use indexing
- **Quote Negotiation:** Quote locking prevents concurrent modifications
- **Credit Checks:** Real-time credit validation on checkout
- **FPC:** B2B pages often uncacheable due to user-specific content

## Backward Compatibility

- **B2B APIs:** Repository interfaces stable across Adobe Commerce versions
- **Company Structure:** Hierarchy model unchanged since B2B 1.0
- **Quote Schema:** Negotiable quote schema additions BC-compliant
- **Extension Attributes:** B2B uses extension attributes for core entity augmentation

## Tests to Add

**Unit Tests:**
```php
testCompanyCreation()
testSharedCatalogAssignment()
testQuoteNegotiation()
testApprovalRuleEvaluation()
testCreditAllocation()
```

**Integration Tests:**
```php
testCompanyHierarchyTraversal()
testSharedCatalogPricing()
testRequisitionListToCart()
testPurchaseOrderWorkflow()
```

**Functional Tests (MFTF):**
```xml
<test name="AdminCreateCompanyTest">
<test name="StorefrontNegotiableQuoteWorkflowTest">
<test name="StorefrontPurchaseOrderApprovalTest">
```

## Docs to Update

- **README.md:** B2B module installation, configuration, feature overview
- **docs/COMPANY.md:** Company structure diagram, hierarchy management
- **docs/QUOTES.md:** Quote negotiation workflow, state diagram
- **docs/APPROVALS.md:** Purchase order approval rules, configuration examples
- **Admin User Guide:** Screenshots of company admin, shared catalog setup, quote management UI

## Related Documentation

### Related Guides

- [Service Contracts vs Repositories in Magento 2](service-contracts-repositories.md)
- [Multi-Store Configuration Mastery](../how-to/multi-store-setup.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Customer Overview](../../modules/customer/README.md)
- [Magento_Quote Overview](../../modules/quote/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
