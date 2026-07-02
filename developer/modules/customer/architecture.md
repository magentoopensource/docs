---
title: "Magento_Customer Architecture"
module: "Magento_Customer"
doc_type: "architecture"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Customer Architecture

## Overview

The Magento_Customer module implements a sophisticated multi-layer architecture combining Entity-Attribute-Value (EAV) flexibility with strongly-typed service contracts. This design provides extensibility for custom customer fields while maintaining API stability and type safety. The architecture separates concerns across service layer (API contracts), domain layer (models and business logic), persistence layer (resource models and repositories), and presentation layer (blocks, view models, UI components).

This document provides an in-depth examination of the module's architectural components, database schema, service contracts, and design patterns.

**Target Versions:** Magento 2.4.7+ / Adobe Commerce 2.4.7+ with PHP 8.2+

---

## Architectural Layers

### 1. Service Contract Layer (API)

The service contract layer defines stable interfaces for external consumption (other modules, REST/GraphQL APIs, third-party integrations). These contracts are guaranteed to remain backward compatible within major versions.

#### Core Service Contracts

**CustomerRepositoryInterface**
- **Purpose:** CRUD operations for customer entities
- **Location:** `Magento\Customer\Api\CustomerRepositoryInterface`
- **Key Methods:**
  - `save(CustomerInterface $customer, string $passwordHash = null): CustomerInterface`
  - `get(string $email, int $websiteId = null): CustomerInterface`
  - `getById(int $customerId): CustomerInterface`
  - `getList(SearchCriteriaInterface $searchCriteria): CustomerSearchResultsInterface`
  - `delete(CustomerInterface $customer): bool`
  - `deleteById(int $customerId): bool`

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Api\SortOrderBuilder;

class CustomerSearchService
{
    public function __construct(
        private readonly CustomerRepositoryInterface $customerRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder,
        private readonly SortOrderBuilder $sortOrderBuilder
    ) {}

    /**
     * Find customers by group ID with pagination
     *
     * @param int $groupId
     * @param int $pageSize
     * @param int $currentPage
     * @return \Magento\Customer\Api\Data\CustomerInterface[]
     */
    public function findByGroupId(int $groupId, int $pageSize = 20, int $currentPage = 1): array
    {
        $sortOrder = $this->sortOrderBuilder
            ->setField('created_at')
            ->setDirection('DESC')
            ->create();

        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('group_id', $groupId)
            ->setPageSize($pageSize)
            ->setCurrentPage($currentPage)
            ->addSortOrder($sortOrder)
            ->create();

        $searchResults = $this->customerRepository->getList($searchCriteria);
        return $searchResults->getItems();
    }
}
```

**AddressRepositoryInterface**
- **Purpose:** Manage customer addresses
- **Location:** `Magento\Customer\Api\AddressRepositoryInterface`
- **Key Methods:**
  - `save(AddressInterface $address): AddressInterface`
  - `getById(int $addressId): AddressInterface`
  - `getList(SearchCriteriaInterface $searchCriteria): AddressSearchResultsInterface`
  - `delete(AddressInterface $address): bool`
  - `deleteById(int $addressId): bool`

**AccountManagementInterface**
- **Purpose:** High-level account operations (authentication, password management, validation)
- **Location:** `Magento\Customer\Api\AccountManagementInterface`
- **Key Methods:**
  - `authenticate(string $username, string $password): CustomerInterface`
  - `createAccount(CustomerInterface $customer, string $password = null, string $redirectUrl = ''): CustomerInterface`
  - `changePassword(string $email, string $currentPassword, string $newPassword): bool`
  - `initiatePasswordReset(string $email, string $template, int $websiteId = null): bool`
  - `resetPassword(string $email, string $resetToken, string $newPassword): bool`
  - `validate(CustomerInterface $customer): ValidationResultsInterface`
  - `isEmailAvailable(string $customerEmail, int $websiteId = null): bool`

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\AccountManagementInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Exception\InvalidEmailOrPasswordException;
use Psr\Log\LoggerInterface;

class CustomerAuthenticationService
{
    public function __construct(
        private readonly AccountManagementInterface $accountManagement,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Authenticate customer and return customer data
     *
     * @param string $email
     * @param string $password
     * @return CustomerInterface|null
     */
    public function authenticate(string $email, string $password): ?CustomerInterface
    {
        try {
            return $this->accountManagement->authenticate($email, $password);
        } catch (InvalidEmailOrPasswordException $e) {
            $this->logger->info('Failed authentication attempt', [
                'email' => $email,
                'reason' => 'invalid_credentials'
            ]);
            return null;
        }
    }

    /**
     * Check if email is available for registration
     *
     * @param string $email
     * @param int $websiteId
     * @return bool
     */
    public function isEmailAvailable(string $email, int $websiteId): bool
    {
        try {
            return $this->accountManagement->isEmailAvailable($email, $websiteId);
        } catch (\Exception $e) {
            $this->logger->error('Email availability check failed', [
                'email' => $email,
                'error' => $e->getMessage()
            ]);
            return false;
        }
    }
}
```

**GroupRepositoryInterface**
- **Purpose:** Customer group management
- **Location:** `Magento\Customer\Api\GroupRepositoryInterface`
- **Key Methods:**
  - `save(GroupInterface $group): GroupInterface`
  - `getById(int $id): GroupInterface`
  - `getList(SearchCriteriaInterface $searchCriteria): GroupSearchResultsInterface`
  - `delete(GroupInterface $group): bool`
  - `deleteById(int $id): bool`

---

### 2. Data Interface Layer (DTOs)

Data Transfer Objects provide strongly-typed structures for customer data across layers.

**CustomerInterface**
- **Location:** `Magento\Customer\Api\Data\CustomerInterface`
- **Key Properties:**
  - `id` (int) - Customer entity ID
  - `email` (string) - Primary email address (unique per website)
  - `firstname` (string) - First name
  - `lastname` (string) - Last name
  - `middlename` (string, nullable) - Middle name or initial
  - `prefix` (string, nullable) - Name prefix (Mr., Mrs., Dr.)
  - `suffix` (string, nullable) - Name suffix (Jr., Sr., III)
  - `dob` (string, nullable) - Date of birth (YYYY-MM-DD)
  - `taxvat` (string, nullable) - Tax/VAT number
  - `gender` (int, nullable) - Gender (1=Male, 2=Female, 3=Not Specified)
  - `store_id` (int) - Store ID where customer was created
  - `website_id` (int) - Website ID for customer account
  - `addresses` (AddressInterface[]) - Customer addresses
  - `group_id` (int) - Customer group ID
  - `default_billing` (int, nullable) - Default billing address ID
  - `default_shipping` (int, nullable) - Default shipping address ID
  - `created_at` (string) - Account creation timestamp
  - `updated_at` (string) - Last update timestamp
  - `created_in` (string, nullable) - Store view name where created
  - `disable_auto_group_change` (int) - Prevent automatic group changes
  - `extension_attributes` (ExtensionAttributesInterface) - Custom extensions
  - `custom_attributes` (AttributeInterface[]) - EAV custom attributes

**AddressInterface**
- **Location:** `Magento\Customer\Api\Data\AddressInterface`
- **Key Properties:**
  - `id` (int) - Address entity ID
  - `customer_id` (int) - Parent customer ID
  - `region` (RegionInterface) - Region/state object
  - `region_id` (int) - Region ID from directory
  - `country_id` (string) - Two-letter country code (ISO 3166-1 alpha-2)
  - `street` (string[]) - Street address lines (array)
  - `company` (string, nullable) - Company name
  - `telephone` (string) - Phone number
  - `fax` (string, nullable) - Fax number
  - `postcode` (string) - Postal/ZIP code
  - `city` (string) - City name
  - `firstname` (string) - First name
  - `lastname` (string) - Last name
  - `middlename` (string, nullable) - Middle name
  - `prefix` (string, nullable) - Name prefix
  - `suffix` (string, nullable) - Name suffix
  - `vat_id` (string, nullable) - VAT identification number
  - `default_shipping` (bool) - Is default shipping address
  - `default_billing` (bool) - Is default billing address
  - `custom_attributes` (AttributeInterface[]) - EAV custom attributes

**Extension Attributes**

Extension attributes allow third-party modules to add fields to data interfaces without modifying core code.

```xml
<!-- etc/extension_attributes.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/extension_attributes.xsd">
    <extension_attributes for="Magento\Customer\Api\Data\CustomerInterface">
        <attribute code="loyalty_points" type="int" />
        <attribute code="marketing_consent" type="bool" />
        <attribute code="preferred_store_id" type="int" />
    </extension_attributes>

    <extension_attributes for="Magento\Customer\Api\Data\AddressInterface">
        <attribute code="delivery_instructions" type="string" />
        <attribute code="address_notes" type="string" />
    </extension_attributes>
</config>
```

Access extension attributes:

```php
<?php
$customer = $customerRepository->getById($customerId);
$extensionAttributes = $customer->getExtensionAttributes();

if ($extensionAttributes) {
    $loyaltyPoints = $extensionAttributes->getLoyaltyPoints();
    $marketingConsent = $extensionAttributes->getMarketingConsent();
}
```

---

### 3. Domain Model Layer

Domain models implement business logic and data manipulation. These are concrete implementations of the service contracts.

**Customer Model**
- **Location:** `Magento\Customer\Model\Customer`
- **Purpose:** EAV entity representing a customer
- **Key Methods:**
  - `authenticate(string $password): bool` - Verify password
  - `setPassword(string $password): self` - Set new password (pre-hashed)
  - `changePassword(string $newPassword): self` - Change password with validation
  - `getName(): string` - Get full name
  - `getDataModel(): CustomerInterface` - Convert to DTO
  - `updateData(CustomerInterface $customer): self` - Update from DTO

**Address Model**
- **Location:** `Magento\Customer\Model\Address`
- **Purpose:** Customer address entity
- **Key Methods:**
  - `validate(): bool|array` - Validate address data
  - `getDataModel(): AddressInterface` - Convert to DTO
  - `updateData(AddressInterface $address): self` - Update from DTO

**CustomerRegistry**
- **Location:** `Magento\Customer\Model\CustomerRegistry`
- **Purpose:** In-memory cache for customer entities (prevents redundant database queries)
- **Pattern:** Registry pattern with identity map

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Model\CustomerRegistry;
use Magento\Customer\Model\Customer;

class CustomerCacheExample
{
    public function __construct(
        private readonly CustomerRegistry $customerRegistry
    ) {}

    /**
     * Retrieve customer from registry (cached) or load from database
     *
     * @param int $customerId
     * @return Customer
     */
    public function getCustomer(int $customerId): Customer
    {
        // Registry checks cache first, loads from DB if not cached
        return $this->customerRegistry->retrieve($customerId);
    }

    /**
     * Remove customer from registry cache
     *
     * @param int $customerId
     */
    public function removeFromCache(int $customerId): void
    {
        $this->customerRegistry->remove($customerId);
    }
}
```

**Session Model**
- **Location:** `Magento\Customer\Model\Session`
- **Purpose:** Manage customer session state
- **Key Methods:**
  - `loginById(int $customerId): bool` - Log in customer by ID
  - `logout(): self` - End customer session
  - `isLoggedIn(): bool` - Check authentication state
  - `getCustomerId(): int|null` - Get current customer ID
  - `getCustomer(): Customer` - Get current customer model
  - `regenerateId(): self` - Prevent session fixation

**IMPORTANT:** Never use Session in service contracts or API contexts. Session is frontend-only.

---

### 4. Persistence Layer

**CustomerRepository**
- **Location:** `Magento\Customer\Model\ResourceModel\CustomerRepository`
- **Purpose:** Implementation of CustomerRepositoryInterface
- **Responsibilities:**
  - Orchestrate model hydration and persistence
  - Trigger events (`customer_save_before`, `customer_save_after`)
  - Validate data before save
  - Handle search criteria and filtering

**Customer Resource Model**
- **Location:** `Magento\Customer\Model\ResourceModel\Customer`
- **Purpose:** Direct database operations for customer entity
- **Responsibilities:**
  - Execute SQL queries for CRUD operations
  - Handle EAV attribute loading/saving
  - Manage customer-website relationships

**Customer Collection**
- **Location:** `Magento\Customer\Model\ResourceModel\Customer\Collection`
- **Purpose:** Load multiple customer entities with filtering
- **Key Methods:**
  - `addFieldToFilter()` - Add WHERE conditions
  - `addAttributeToSelect()` - Load specific EAV attributes
  - `setPageSize()` / `setCurPage()` - Pagination
  - `addNameToSelect()` - Optimized name loading
  - `joinAttribute()` - Join additional EAV attributes

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Customer\Model\ResourceModel\Customer\CollectionFactory;

class CustomerCollectionExample
{
    public function __construct(
        private readonly CollectionFactory $customerCollectionFactory
    ) {}

    /**
     * Get customers created in the last 30 days
     *
     * @return \Magento\Customer\Model\ResourceModel\Customer\Collection
     */
    public function getRecentCustomers(): \Magento\Customer\Model\ResourceModel\Customer\Collection
    {
        $collection = $this->customerCollectionFactory->create();
        $collection->addAttributeToSelect('*')
            ->addFieldToFilter('created_at', [
                'from' => date('Y-m-d', strtotime('-30 days')),
                'to' => date('Y-m-d'),
                'datetime' => true
            ])
            ->setOrder('created_at', 'DESC')
            ->setPageSize(100);

        return $collection;
    }
}
```

---

## Database Schema

### Core Tables

#### customer_entity
Primary customer entity table.

```sql
CREATE TABLE `customer_entity` (
  `entity_id` int unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `website_id` smallint unsigned DEFAULT NULL,
  `email` varchar(255) DEFAULT NULL,
  `group_id` smallint unsigned NOT NULL DEFAULT 0,
  `increment_id` varchar(50) DEFAULT NULL,
  `store_id` smallint unsigned DEFAULT 0,
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `is_active` smallint unsigned NOT NULL DEFAULT 1,
  `disable_auto_group_change` smallint unsigned NOT NULL DEFAULT 0,
  `created_in` varchar(255) DEFAULT NULL,
  `prefix` varchar(40) DEFAULT NULL,
  `firstname` varchar(255) DEFAULT NULL,
  `middlename` varchar(255) DEFAULT NULL,
  `lastname` varchar(255) DEFAULT NULL,
  `suffix` varchar(40) DEFAULT NULL,
  `dob` date DEFAULT NULL,
  `password_hash` varchar(128) DEFAULT NULL,
  `rp_token` varchar(128) DEFAULT NULL,
  `rp_token_created_at` datetime DEFAULT NULL,
  `default_billing` int unsigned DEFAULT NULL,
  `default_shipping` int unsigned DEFAULT NULL,
  `taxvat` varchar(50) DEFAULT NULL,
  `confirmation` varchar(64) DEFAULT NULL,
  `gender` smallint unsigned DEFAULT NULL,
  `failures_num` smallint unsigned DEFAULT 0,
  `first_failure` timestamp NULL DEFAULT NULL,
  `lock_expires` timestamp NULL DEFAULT NULL,

  KEY `CUSTOMER_ENTITY_STORE_ID` (`store_id`),
  KEY `CUSTOMER_ENTITY_WEBSITE_ID` (`website_id`),
  KEY `CUSTOMER_ENTITY_FIRSTNAME` (`firstname`),
  KEY `CUSTOMER_ENTITY_LASTNAME` (`lastname`),
  KEY `CUSTOMER_ENTITY_EMAIL` (`email`),
  UNIQUE KEY `CUSTOMER_ENTITY_EMAIL_WEBSITE_ID` (`email`,`website_id`),

  CONSTRAINT `CUSTOMER_ENTITY_STORE_ID_STORE_STORE_ID`
    FOREIGN KEY (`store_id`) REFERENCES `store` (`store_id`) ON DELETE SET NULL,
  CONSTRAINT `CUSTOMER_ENTITY_WEBSITE_ID_STORE_WEBSITE_WEBSITE_ID`
    FOREIGN KEY (`website_id`) REFERENCES `store_website` (`website_id`) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**Key Indexes:**
- **UNIQUE(email, website_id):** Enforces email uniqueness per website
- **INDEX(website_id, store_id):** Fast filtering by scope
- **INDEX(firstname, lastname):** Name searches in admin grid
- **INDEX(email):** Login lookups

**Security Fields:**
- `password_hash` - Argon2 or SHA-256 hashed password
- `rp_token` / `rp_token_created_at` - Password reset token with expiration
- `failures_num` / `first_failure` / `lock_expires` - Account lockout protection

#### customer_address_entity
Customer addresses with support for multiple addresses per customer.

```sql
CREATE TABLE `customer_address_entity` (
  `entity_id` int unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `increment_id` varchar(50) DEFAULT NULL,
  `parent_id` int unsigned DEFAULT NULL,
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `is_active` smallint unsigned NOT NULL DEFAULT 1,
  `city` varchar(255) DEFAULT NULL,
  `company` varchar(255) DEFAULT NULL,
  `country_id` varchar(255) NOT NULL,
  `fax` varchar(255) DEFAULT NULL,
  `firstname` varchar(255) DEFAULT NULL,
  `lastname` varchar(255) DEFAULT NULL,
  `middlename` varchar(255) DEFAULT NULL,
  `postcode` varchar(255) DEFAULT NULL,
  `prefix` varchar(40) DEFAULT NULL,
  `region` varchar(255) DEFAULT NULL,
  `region_id` int unsigned DEFAULT NULL,
  `street` text,
  `suffix` varchar(40) DEFAULT NULL,
  `telephone` varchar(255) DEFAULT NULL,
  `vat_id` varchar(255) DEFAULT NULL,
  `vat_is_valid` int unsigned DEFAULT NULL,
  `vat_request_date` varchar(255) DEFAULT NULL,
  `vat_request_id` varchar(255) DEFAULT NULL,
  `vat_request_success` int unsigned DEFAULT NULL,

  KEY `CUSTOMER_ADDRESS_ENTITY_PARENT_ID` (`parent_id`),

  CONSTRAINT `CUSTOMER_ADDRESS_ENTITY_PARENT_ID_CUSTOMER_ENTITY_ENTITY_ID`
    FOREIGN KEY (`parent_id`) REFERENCES `customer_entity` (`entity_id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**Key Relationships:**
- `parent_id` links to `customer_entity.entity_id`
- Cascade delete: removing customer deletes all addresses

#### customer_group
Defines customer segments for pricing and tax rules.

```sql
CREATE TABLE `customer_group` (
  `customer_group_id` int unsigned NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `customer_group_code` varchar(32) NOT NULL,
  `tax_class_id` int unsigned NOT NULL DEFAULT 0,

  UNIQUE KEY `CUSTOMER_GROUP_CUSTOMER_GROUP_CODE` (`customer_group_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**Default Groups:**
```sql
INSERT INTO customer_group VALUES
(0, 'NOT LOGGED IN', 3),
(1, 'General', 3),
(2, 'Wholesale', 3),
(3, 'Retailer', 3);
```

#### customer_grid_flat
Denormalized table for fast admin grid performance.

```sql
CREATE TABLE `customer_grid_flat` (
  `entity_id` int unsigned NOT NULL,
  `name` text,
  `email` varchar(255) DEFAULT NULL,
  `group_id` int unsigned DEFAULT NULL,
  `created_at` timestamp NULL DEFAULT NULL,
  `website_id` int unsigned DEFAULT NULL,
  `confirmation` varchar(64) DEFAULT NULL,
  `created_in` text,
  `dob` date DEFAULT NULL,
  `gender` int unsigned DEFAULT NULL,
  `taxvat` varchar(50) DEFAULT NULL,
  `lock_expires` timestamp NULL DEFAULT NULL,
  `shipping_full` text,
  `billing_full` text,
  `billing_firstname` varchar(255) DEFAULT NULL,
  `billing_lastname` varchar(255) DEFAULT NULL,
  `billing_telephone` varchar(255) DEFAULT NULL,
  `billing_postcode` varchar(255) DEFAULT NULL,
  `billing_country_id` varchar(255) DEFAULT NULL,
  `billing_region` varchar(255) DEFAULT NULL,
  `billing_region_id` int unsigned DEFAULT NULL,
  `billing_street` varchar(255) DEFAULT NULL,
  `billing_city` varchar(255) DEFAULT NULL,
  `billing_fax` varchar(255) DEFAULT NULL,
  `billing_vat_id` varchar(255) DEFAULT NULL,
  `billing_company` varchar(255) DEFAULT NULL,

  PRIMARY KEY (`entity_id`),
  FULLTEXT KEY `CUSTOMER_GRID_FLAT_NAME_EMAIL` (`name`,`email`),
  KEY `CUSTOMER_GRID_FLAT_GROUP_ID` (`group_id`),
  KEY `CUSTOMER_GRID_FLAT_CREATED_AT` (`created_at`),
  KEY `CUSTOMER_GRID_FLAT_WEBSITE_ID` (`website_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**Purpose:** Optimized for admin customer grid queries. Updated via indexer:
```bash
bin/magento indexer:reindex customer_grid
```

### EAV Attribute Tables

Customer and address entities use EAV for custom attributes.

#### customer_eav_attribute
Stores customer attribute metadata.

```sql
CREATE TABLE `customer_eav_attribute` (
  `attribute_id` smallint unsigned NOT NULL,
  `is_visible` smallint unsigned NOT NULL DEFAULT 1,
  `input_filter` varchar(255) DEFAULT NULL,
  `multiline_count` smallint unsigned NOT NULL DEFAULT 1,
  `validate_rules` text,
  `is_system` smallint unsigned NOT NULL DEFAULT 0,
  `sort_order` int unsigned NOT NULL DEFAULT 0,
  `data_model` varchar(255) DEFAULT NULL,
  `is_used_in_grid` smallint unsigned NOT NULL DEFAULT 0,
  `is_visible_in_grid` smallint unsigned NOT NULL DEFAULT 0,
  `is_filterable_in_grid` smallint unsigned NOT NULL DEFAULT 0,
  `is_searchable_in_grid` smallint unsigned NOT NULL DEFAULT 0,

  PRIMARY KEY (`attribute_id`),
  CONSTRAINT `CUSTOMER_EAV_ATTRIBUTE_ATTRIBUTE_ID_EAV_ATTRIBUTE_ATTRIBUTE_ID`
    FOREIGN KEY (`attribute_id`) REFERENCES `eav_attribute` (`attribute_id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### customer_entity_*
Value tables for different attribute data types:
- `customer_entity_datetime` - Date/time attributes
- `customer_entity_decimal` - Numeric attributes
- `customer_entity_int` - Integer attributes (including dropdowns)
- `customer_entity_text` - Short text attributes
- `customer_entity_varchar` - Variable-length text attributes

Example structure (all follow same pattern):

```sql
CREATE TABLE `customer_entity_varchar` (
  `value_id` int NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `attribute_id` smallint unsigned NOT NULL,
  `entity_id` int unsigned NOT NULL,
  `value` varchar(255) DEFAULT NULL,

  UNIQUE KEY `CUSTOMER_ENTITY_VARCHAR_ENTITY_ID_ATTRIBUTE_ID` (`entity_id`,`attribute_id`),
  KEY `CUSTOMER_ENTITY_VARCHAR_ATTRIBUTE_ID` (`attribute_id`),
  KEY `CUSTOMER_ENTITY_VARCHAR_ENTITY_ID_ATTRIBUTE_ID_VALUE` (`entity_id`,`attribute_id`,`value`),

  CONSTRAINT `CUSTOMER_ENTITY_VARCHAR_ENTITY_ID_CUSTOMER_ENTITY_ENTITY_ID`
    FOREIGN KEY (`entity_id`) REFERENCES `customer_entity` (`entity_id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## EAV Attribute System

### Adding Custom Customer Attributes

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Customer\Model\Customer;
use Magento\Customer\Setup\CustomerSetupFactory;
use Magento\Eav\Model\Entity\Attribute\SetFactory as AttributeSetFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddCustomerSubscriptionTier implements DataPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
        private readonly CustomerSetupFactory $customerSetupFactory,
        private readonly AttributeSetFactory $attributeSetFactory
    ) {}

    public function apply(): self
    {
        $customerSetup = $this->customerSetupFactory->create(['setup' => $this->moduleDataSetup]);

        $customerSetup->addAttribute(
            Customer::ENTITY,
            'subscription_tier',
            [
                'type' => 'varchar',
                'label' => 'Subscription Tier',
                'input' => 'select',
                'source' => \Vendor\Module\Model\Customer\Attribute\Source\SubscriptionTier::class,
                'required' => false,
                'visible' => true,
                'user_defined' => true,
                'sort_order' => 200,
                'position' => 200,
                'system' => false,
                'is_used_in_grid' => true,
                'is_visible_in_grid' => true,
                'is_filterable_in_grid' => true,
                'is_searchable_in_grid' => true,
            ]
        );

        // Add attribute to default attribute set
        $customerEntity = $customerSetup->getEavConfig()->getEntityType(Customer::ENTITY);
        $attributeSetId = $customerEntity->getDefaultAttributeSetId();
        $attributeSet = $this->attributeSetFactory->create();
        $attributeGroupId = $attributeSet->getDefaultGroupId($attributeSetId);

        $attribute = $customerSetup->getEavConfig()->getAttribute(Customer::ENTITY, 'subscription_tier');
        $attribute->addData([
            'attribute_set_id' => $attributeSetId,
            'attribute_group_id' => $attributeGroupId,
            'used_in_forms' => [
                'adminhtml_customer',
                'customer_account_edit',
                'customer_account_create'
            ],
        ]);
        $attribute->save();

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

### Attribute Source Model

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Customer\Attribute\Source;

use Magento\Eav\Model\Entity\Attribute\Source\AbstractSource;

class SubscriptionTier extends AbstractSource
{
    public const TIER_FREE = 'free';
    public const TIER_BASIC = 'basic';
    public const TIER_PREMIUM = 'premium';
    public const TIER_ENTERPRISE = 'enterprise';

    /**
     * Get all options for dropdown
     *
     * @return array
     */
    public function getAllOptions(): array
    {
        if ($this->_options === null) {
            $this->_options = [
                ['label' => __('Free'), 'value' => self::TIER_FREE],
                ['label' => __('Basic'), 'value' => self::TIER_BASIC],
                ['label' => __('Premium'), 'value' => self::TIER_PREMIUM],
                ['label' => __('Enterprise'), 'value' => self::TIER_ENTERPRISE],
            ];
        }
        return $this->_options;
    }
}
```

---

## Design Patterns

### 1. Repository Pattern
Abstracts data access logic behind service contract interfaces.

**Benefits:**
- Decouples business logic from persistence mechanism
- Enables switching storage backends without API changes
- Facilitates testing with mock repositories

### 2. Data Transfer Object (DTO) Pattern
Uses data interfaces (CustomerInterface, AddressInterface) for type-safe data transfer.

**Benefits:**
- Strong typing prevents runtime errors
- IDE autocomplete and static analysis support
- Clear contracts for API consumers

### 3. Registry Pattern
CustomerRegistry maintains identity map of loaded entities.

**Benefits:**
- Prevents redundant database queries
- Ensures single instance per entity ID
- Maintains consistency within request scope

### 4. EAV Pattern
Entity-Attribute-Value for extensible customer fields.

**Trade-offs:**
- **Pro:** Add fields without schema migrations
- **Pro:** Flexible data model for multi-tenant scenarios
- **Con:** Complex queries (multiple joins)
- **Con:** Harder to index and optimize

### 5. Observer Pattern
Event-driven extensions via `events.xml`.

**Key Events:**
- `customer_register_success` - After registration
- `customer_login` - After successful login
- `customer_logout` - After logout
- `customer_save_before` / `customer_save_after` - Around customer save
- `customer_address_save_before` / `customer_address_save_after` - Around address save
- `customer_delete_before` / `customer_delete_after` - Around customer deletion

### 6. Plugin Pattern (Interception)
Modify behavior of public methods without inheritance.

**Example:** Add logging to customer authentication:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Model;

use Magento\Customer\Api\AccountManagementInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Psr\Log\LoggerInterface;

class AccountManagementExtend
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Log successful authentication
     *
     * @param AccountManagementInterface $subject
     * @param CustomerInterface $result
     * @return CustomerInterface
     */
    public function afterAuthenticate(
        AccountManagementInterface $subject,
        CustomerInterface $result
    ): CustomerInterface {
        $this->logger->info('Customer authenticated', [
            'customer_id' => $result->getId(),
            'email' => $result->getEmail()
        ]);

        return $result;
    }
}
```

```xml
<!-- etc/di.xml -->
<type name="Magento\Customer\Api\AccountManagementInterface">
    <plugin name="vendor_module_account_management_logger"
            type="Vendor\Module\Plugin\Customer\Model\AccountManagementExtend" />
</type>
```

---

## Configuration Schema

Key configuration paths in `core_config_data`:

| Path | Type | Purpose |
|------|------|---------|
| `customer/account_share/scope` | int | 0=Global, 1=Per Website |
| `customer/create_account/confirm` | bool | Email confirmation required |
| `customer/create_account/email_domain` | string | Allowed email domains |
| `customer/create_account/default_group` | int | Default group for new customers |
| `customer/password/required_character_classes_number` | int | Password complexity (0-4) |
| `customer/password/minimum_password_length` | int | Min password length |
| `customer/address/street_lines` | int | Number of street lines (1-4) |
| `customer/startup/redirect_dashboard` | bool | Redirect to dashboard after login |
| `customer/online_customers/section_data_lifetime` | int | Private content cache lifetime (seconds) |

---

## Assumptions

- **Target Platform:** Adobe Commerce / Magento Open Source 2.4.7+
- **PHP Version:** 8.2+
- **Database:** MySQL 8.0+ or MariaDB 10.6+ with InnoDB storage engine
- **Scope:** Single and multi-website deployments with account sharing configured appropriately
- **Authentication:** Standard email/password (extensible for OAuth/SAML)

## Why This Approach

The Customer module's architecture balances flexibility (EAV), type safety (service contracts), and performance (indexed grid, registry caching). Service contracts ensure API stability for integrations, while EAV allows merchants to extend customer data without code modifications. The repository pattern abstracts persistence, enabling future storage backend changes (e.g., NoSQL for specific use cases) without breaking consuming code.

## Security Impact

- **Authorization:** All admin operations check ACL (`Magento_Customer::manage`, `Magento_Customer::customer`)
- **Password Storage:** Argon2id (PHP 8.2+) or SHA-256 with salt
- **Session Security:** `regenerateId()` called after authentication to prevent fixation
- **CSRF:** All forms include form key validation
- **PII Protection:** Customer data logged only when explicitly required; use data scrubbers for logs

## Performance Impact

- **FPC:** Customer-specific content uses private content blocks (`data-bind="scope: 'customer'"`)
- **Database:** Customer grid indexed separately; EAV queries optimized with proper indexing
- **Caching:** CustomerRegistry prevents redundant loads; use `customer_grid_flat` for admin queries
- **Sessions:** Redis backend recommended for horizontal scaling

## Backward Compatibility

- **Service Contracts:** Interfaces stable within 2.4.x; new methods added to end of interface
- **Database Schema:** EAV additions backward compatible; new columns added with defaults
- **Deprecations:** Direct model usage (e.g., `Customer::load()`) deprecated since 2.3; use repositories
- **API Versioning:** REST API versioned (`/V1/`); breaking changes require new version

## Tests to Add

- **Unit:** Repository methods, attribute source models, validation logic
- **Integration:** CRUD via service contracts, EAV attribute loading, address operations
- **API:** REST/GraphQL contract tests (customer CRUD, authentication, address management)
- **MFTF:** Customer registration, login, profile edit, address CRUD

## Docs to Update

- **README.md:** High-level module overview
- **ARCHITECTURE.md:** This document
- **EXECUTION_FLOWS.md:** Registration, authentication, address management flows
- **CHANGELOG.md:** Version-specific schema and API changes
