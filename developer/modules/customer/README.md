---
title: "Magento_Customer Overview"
module: "Magento_Customer"
doc_type: "overview"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Customer Module

## Overview

The **Magento_Customer** module is one of the most critical core modules in Adobe Commerce and Magento Open Source, responsible for managing all customer-related functionality including registration, authentication, account management, address handling, customer groups, and customer attributes. This module implements a comprehensive EAV (Entity-Attribute-Value) system for flexible customer data storage and provides robust service contracts for integration with other modules.

The Customer module serves as the foundation for personalized shopping experiences, order history tracking, customer segmentation, and multi-address checkout functionality. It integrates deeply with Quote, Sales, Wishlist, Newsletter, and virtually every customer-facing feature in the platform.

**Module Location:** `vendor/magento/module-customer` (Composer) or `app/code/Magento/Customer` (legacy)

**Namespace:** `Magento\Customer`

**Target Versions:** Magento 2.4.7+ / Adobe Commerce 2.4.7+ with PHP 8.2+

---

## Key Features

### 1. Customer Account Management
- **Registration Flows:** Standard registration, admin-created accounts, social login integration points
- **Authentication:** Password-based login, token-based API authentication, two-factor authentication hooks
- **Profile Management:** Personal information, email changes with verification, password resets
- **Account Dashboard:** Centralized customer account interface with extensible sections

### 2. EAV Attribute System
- **Flexible Data Model:** Custom customer attributes with type validation (text, date, dropdown, multiselect)
- **Frontend/Backend Forms:** Automatic form generation from attribute metadata
- **Attribute Sets:** Logical grouping for account creation and admin forms
- **Validation Rules:** Built-in validators for email uniqueness, password strength, VAT numbers

### 3. Address Management
- **Multiple Addresses:** Customers can store unlimited addresses with default billing/shipping designation
- **Address Validation:** Street line limits, postal code formats, region/country dependencies
- **VAT Validation:** European VAT number validation with group assignment automation
- **Address Books:** Frontend and admin interfaces for CRUD operations

### 4. Customer Groups
- **Segmentation:** Group-based pricing, tax classes, and catalog visibility
- **Automatic Assignment:** VAT validation, custom plugin-based assignment logic
- **B2B Integration:** Foundation for company accounts and shared catalogs in Adobe Commerce B2B
- **Default Groups:** NOT LOGGED IN (0), General (1), Wholesale (2), Retailer (3)

### 5. Session Management
- **Frontend Sessions:** Customer login state persistence across requests
- **Admin Sessions:** Separate session handling for backend customer impersonation
- **Session Regeneration:** Security measures against session fixation attacks
- **Persistent Shopping Cart:** Cross-device cart persistence when enabled

### 6. Customer Grid & Admin Interface
- **Searchable Grid:** Advanced filtering by name, email, group, website, attributes
- **Inline Editing:** Quick updates to customer information without full page loads
- **Mass Actions:** Bulk operations for group assignment, newsletter subscription, account deletion
- **Custom Columns:** Extend grid with custom attribute columns via UI components

### 7. API & Service Contracts
- **REST API:** Full CRUD operations at `/V1/customers` and `/V1/customers/me`
- **GraphQL:** Query and mutation support for customer data and addresses
- **Service Contracts:** `CustomerRepositoryInterface`, `AddressRepositoryInterface`, `AccountManagementInterface`
- **Data Interfaces:** Strongly typed DTOs for customer and address entities

### 8. Email Communications
- **Welcome Emails:** Configurable templates for new account creation
- **Password Reset:** Secure token-based reset flow with expiration
- **Email Change Confirmation:** Verify new email addresses before updating
- **Admin Notifications:** Optional alerts for new registrations

### 9. Customer Attributes & Custom Fields
- **System Attributes:** firstname, lastname, email, dob, gender, taxvat
- **Custom Attributes:** Add unlimited attributes via admin or data patches
- **Form Rendering:** Automatic integration into registration, account edit, and checkout forms
- **API Exposure:** Custom attributes automatically available via REST/GraphQL

### 10. Integration Points
- **Quote Module:** Customer data injection into cart and checkout
- **Sales Module:** Order history, invoice/shipment tracking
- **Wishlist Module:** Persistent wishlists tied to customer accounts
- **Newsletter Module:** Subscription management from customer account
- **Review Module:** Customer-authored product reviews
- **Tax Module:** Customer group-based tax calculations

---

## Quick Start

### Installation & Setup

The Customer module is installed by default with Magento. To verify installation:

```bash
bin/magento module:status Magento_Customer
# Should show: "Module is enabled"

bin/magento setup:upgrade
bin/magento cache:flush
```

### Configuration Paths

Key configuration can be found at **Stores > Configuration > Customers**:

| Section | Path | Purpose |
|---------|------|---------|
| Customer Configuration | `customer/account_share/*` | Account sharing scope (global/per-website) |
| Customer Configuration | `customer/create_account/*` | Registration settings, email confirmation |
| Customer Configuration | `customer/password/*` | Password complexity rules |
| Customer Configuration | `customer/address/*` | Address line counts, validation rules |
| Customer Configuration | `customer/startup/*` | Redirect destinations after login |
| Online Customers | `customer/online_customers/*` | Session tracking settings |

### Basic Usage Examples

#### 1. Create a Customer Programmatically

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Api\Data\CustomerInterfaceFactory;
use Magento\Framework\Encryption\EncryptorInterface;
use Magento\Framework\Exception\LocalizedException;
use Magento\Store\Model\StoreManagerInterface;

class CustomerCreator
{
    public function __construct(
        private readonly CustomerRepositoryInterface $customerRepository,
        private readonly CustomerInterfaceFactory $customerFactory,
        private readonly EncryptorInterface $encryptor,
        private readonly StoreManagerInterface $storeManager
    ) {}

    /**
     * Create a new customer account
     *
     * @param string $email
     * @param string $firstname
     * @param string $lastname
     * @param string $password
     * @return \Magento\Customer\Api\Data\CustomerInterface
     * @throws LocalizedException
     */
    public function createCustomer(
        string $email,
        string $firstname,
        string $lastname,
        string $password
    ): \Magento\Customer\Api\Data\CustomerInterface {
        $customer = $this->customerFactory->create();
        $customer->setEmail($email)
            ->setFirstname($firstname)
            ->setLastname($lastname)
            ->setWebsiteId((int)$this->storeManager->getWebsite()->getId())
            ->setStoreId((int)$this->storeManager->getStore()->getId())
            ->setGroupId(1); // General customer group

        // Save customer (triggers email if configured)
        $savedCustomer = $this->customerRepository->save(
            $customer,
            $this->encryptor->getHash($password, true)
        );

        return $savedCustomer;
    }
}
```

#### 2. Retrieve Customer by Email

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Framework\Exception\NoSuchEntityException;

class CustomerRetriever
{
    public function __construct(
        private readonly CustomerRepositoryInterface $customerRepository
    ) {}

    /**
     * Get customer by email and website ID
     *
     * @param string $email
     * @param int $websiteId
     * @return \Magento\Customer\Api\Data\CustomerInterface|null
     */
    public function getCustomerByEmail(string $email, int $websiteId): ?\Magento\Customer\Api\Data\CustomerInterface
    {
        try {
            return $this->customerRepository->get($email, $websiteId);
        } catch (NoSuchEntityException) {
            return null;
        }
    }
}
```

#### 3. Add a Customer Address

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\AddressRepositoryInterface;
use Magento\Customer\Api\Data\AddressInterfaceFactory;
use Magento\Customer\Api\Data\RegionInterfaceFactory;

class AddressManager
{
    public function __construct(
        private readonly AddressRepositoryInterface $addressRepository,
        private readonly AddressInterfaceFactory $addressFactory,
        private readonly RegionInterfaceFactory $regionFactory
    ) {}

    /**
     * Add a new address to customer
     *
     * @param int $customerId
     * @param array $addressData
     * @return \Magento\Customer\Api\Data\AddressInterface
     */
    public function addAddress(int $customerId, array $addressData): \Magento\Customer\Api\Data\AddressInterface
    {
        $address = $this->addressFactory->create();
        $address->setCustomerId($customerId)
            ->setFirstname($addressData['firstname'])
            ->setLastname($addressData['lastname'])
            ->setStreet($addressData['street'])
            ->setCity($addressData['city'])
            ->setCountryId($addressData['country_id'])
            ->setPostcode($addressData['postcode'])
            ->setTelephone($addressData['telephone'])
            ->setIsDefaultBilling($addressData['default_billing'] ?? false)
            ->setIsDefaultShipping($addressData['default_shipping'] ?? false);

        if (!empty($addressData['region_id'])) {
            $region = $this->regionFactory->create();
            $region->setRegionId((int)$addressData['region_id']);
            $address->setRegion($region);
        }

        return $this->addressRepository->save($address);
    }
}
```

#### 4. Check if Customer is Logged In

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\ViewModel;

use Magento\Customer\Model\Session as CustomerSession;
use Magento\Framework\View\Element\Block\ArgumentInterface;

class CustomerInfo implements ArgumentInterface
{
    public function __construct(
        private readonly CustomerSession $customerSession
    ) {}

    /**
     * Check if customer is logged in
     *
     * @return bool
     */
    public function isLoggedIn(): bool
    {
        return $this->customerSession->isLoggedIn();
    }

    /**
     * Get current customer ID
     *
     * @return int|null
     */
    public function getCustomerId(): ?int
    {
        return $this->customerSession->getCustomerId();
    }

    /**
     * Get customer name
     *
     * @return string
     */
    public function getCustomerName(): string
    {
        $customer = $this->customerSession->getCustomer();
        return $customer->getName();
    }
}
```

#### 5. Create a Custom Customer Attribute

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Customer\Model\Customer;
use Magento\Customer\Setup\CustomerSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddLoyaltyNumberAttribute implements DataPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
        private readonly CustomerSetupFactory $customerSetupFactory
    ) {}

    public function apply(): self
    {
        $customerSetup = $this->customerSetupFactory->create(['setup' => $this->moduleDataSetup]);

        $customerSetup->addAttribute(
            Customer::ENTITY,
            'loyalty_number',
            [
                'type' => 'varchar',
                'label' => 'Loyalty Program Number',
                'input' => 'text',
                'required' => false,
                'visible' => true,
                'user_defined' => true,
                'position' => 100,
                'system' => false,
                'is_used_in_grid' => true,
                'is_visible_in_grid' => true,
                'is_filterable_in_grid' => true,
                'is_searchable_in_grid' => true,
            ]
        );

        // Add to forms
        $attribute = $customerSetup->getEavConfig()->getAttribute(Customer::ENTITY, 'loyalty_number');
        $attribute->setData('used_in_forms', [
            'adminhtml_customer',
            'customer_account_edit',
            'customer_account_create'
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

---

## Module Dependencies

The Customer module depends on:

- **Magento_Store** - Website/store scope for customer accounts
- **Magento_Eav** - Attribute system foundation
- **Magento_Directory** - Country/region data for addresses
- **Magento_Theme** - Frontend layout and templates
- **Magento_Backend** - Admin interface components
- **Magento_Authorization** - ACL for admin permissions

Modules that depend on Customer:

- **Magento_Quote** - Cart customer association
- **Magento_Sales** - Order customer linkage
- **Magento_Wishlist** - Wishlist ownership
- **Magento_Newsletter** - Subscription management
- **Magento_Review** - Customer reviews
- **Magento_Tax** - Customer group tax rates
- **Magento_Checkout** - Customer checkout flows
- **Magento_CustomerImportExport** - CSV import/export

---

## Directory Structure

```
Magento/Customer/
├── Api/                           # Service contracts
│   ├── AccountManagementInterface.php
│   ├── AddressRepositoryInterface.php
│   ├── CustomerRepositoryInterface.php
│   ├── GroupManagementInterface.php
│   └── Data/                      # Data interfaces (DTOs)
│       ├── CustomerInterface.php
│       └── AddressInterface.php
├── Block/                         # Frontend/admin blocks
│   ├── Account/
│   ├── Address/
│   └── Adminhtml/
├── Controller/                    # Frontend controllers
│   ├── Account/
│   ├── Address/
│   └── Adminhtml/                 # Admin controllers
├── Model/                         # Business logic
│   ├── Customer.php               # Customer entity model
│   ├── Address.php                # Address entity model
│   ├── AccountManagement.php      # Account operations
│   ├── CustomerRegistry.php       # Entity cache
│   ├── ResourceModel/             # Database operations
│   └── Session.php                # Customer session
├── Observer/                      # Event observers
├── Plugin/                        # Interceptors
├── Setup/                         # Installation/upgrade
│   ├── Patch/
│   └── InstallSchema.php
├── Ui/                            # Admin UI components
│   └── Component/
├── ViewModel/                     # Frontend view models
├── view/
│   ├── frontend/                  # Frontend templates/layouts
│   │   ├── layout/
│   │   ├── templates/
│   │   └── web/
│   └── adminhtml/                 # Admin templates/layouts
├── etc/
│   ├── module.xml                 # Module declaration
│   ├── di.xml                     # Dependency injection
│   ├── acl.xml                    # Admin permissions
│   ├── config.xml                 # Default configuration
│   ├── extension_attributes.xml   # Extension attributes
│   ├── webapi.xml                 # REST API routes
│   └── frontend/
│       ├── di.xml
│       └── routes.xml             # Frontend routes
└── Test/
    ├── Unit/
    ├── Integration/
    └── Mftf/                      # Functional tests
```

---

## Common Use Cases

### Use Case 1: Custom Registration Form Fields

Add custom fields to the registration form by creating a custom attribute and adding it to the `customer_account_create` form.

```xml
<!-- etc/extension_attributes.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/extension_attributes.xsd">
    <extension_attributes for="Magento\Customer\Api\Data\CustomerInterface">
        <attribute code="company_role" type="string" />
    </extension_attributes>
</config>
```

### Use Case 2: Customize Customer Groups

Create custom customer groups for segmentation, pricing rules, and catalog visibility.

```php
<?php
use Magento\Customer\Api\GroupRepositoryInterface;
use Magento\Customer\Api\Data\GroupInterfaceFactory;
use Magento\Tax\Model\ClassModel as TaxClass;

$group = $groupFactory->create();
$group->setCode('VIP')
    ->setTaxClassId(3) // Default retail tax class
    ->setTaxClassName('Retail Customer');

$groupRepository->save($group);
```

### Use Case 3: Implement Custom Authentication

Extend the authentication system for SSO or OAuth integrations using plugins on `AccountManagementInterface::authenticate()`.

### Use Case 4: Customer Data Import

Use `CustomerRepositoryInterface` for bulk imports instead of direct model manipulation to trigger proper validation and events.

### Use Case 5: Multi-Website Customer Sharing

Configure account sharing scope at **Stores > Configuration > Customer Configuration > Account Sharing Options**:
- **Global:** Customers shared across all websites
- **Per Website:** Separate customer bases per website

---

## Testing Approach

### Unit Tests
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Unit\Service;

use PHPUnit\Framework\TestCase;
use Vendor\Module\Service\CustomerCreator;

class CustomerCreatorTest extends TestCase
{
    public function testCreateCustomer(): void
    {
        $customerRepository = $this->createMock(\Magento\Customer\Api\CustomerRepositoryInterface::class);
        // ... mock setup and assertions
    }
}
```

### Integration Tests
```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Integration\Service;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

class CustomerCreatorTest extends TestCase
{
    /**
     * @magentoDataFixture Magento/Customer/_files/customer.php
     */
    public function testCustomerExists(): void
    {
        $objectManager = Bootstrap::getObjectManager();
        $repository = $objectManager->get(\Magento\Customer\Api\CustomerRepositoryInterface::class);
        $customer = $repository->get('customer@example.com');
        $this->assertNotNull($customer);
    }
}
```

---

## Security Considerations

1. **Password Handling:** Always use `EncryptorInterface::getHash()` for password storage
2. **Session Fixation:** Call `$session->regenerateId()` after authentication
3. **Email Validation:** Use built-in validators to prevent injection attacks
4. **CSRF Protection:** All forms must include form keys via `\Magento\Framework\Data\Form\FormKey`
5. **XSS Prevention:** Escape output with `$escaper->escapeHtml()` in templates
6. **API Authentication:** Use token-based auth for REST/GraphQL endpoints
7. **PII Handling:** Log customer data carefully; exclude PII from non-secure logs

---

## Performance Optimization

1. **Use CustomerRegistry:** Avoid loading same customer multiple times in a request
2. **Lazy Loading:** Don't load customer session in non-customer contexts
3. **Address Collections:** Use `AddressRepositoryInterface::getList()` with search criteria
4. **Customer Grid:** Use collection filtering instead of post-load filtering
5. **Session Storage:** Use Redis for session backend in production
6. **Full Page Cache:** Customer-specific content must use private content blocks
7. **Index Customer Grid:** Ensure `customer_grid_flat` index is up to date

---

## Additional Resources

- **DevDocs:** https://developer.adobe.com/commerce/php/module-reference/module-customer/
- **Service Contracts:** See `ARCHITECTURE.md` for detailed interface documentation
- **Execution Flows:** See `EXECUTION_FLOWS.md` for registration and authentication sequences
- **Integration Guide:** See `INTEGRATIONS.md` for connecting with other modules
- **Anti-Patterns:** See `ANTI_PATTERNS.md` for common mistakes and solutions

---

## Assumptions

- **Target Platform:** Adobe Commerce / Magento Open Source 2.4.7+
- **PHP Version:** 8.2+
- **Environment:** Production-ready code with proper error handling
- **Scope:** Single and multi-website deployments
- **Authentication:** Standard email/password authentication (extensible for OAuth/SSO)

## Why This Approach

The Customer module uses service contracts and EAV architecture to provide maximum flexibility while maintaining upgrade safety. Service contracts ensure API stability across versions, while the EAV system allows merchants to add custom fields without schema modifications. This design has proven robust across thousands of Magento installations.

## Security Impact

- **Authentication:** All password operations use secure hashing with salt
- **Authorization:** Admin operations protected by ACL (Resource: `Magento_Customer::manage`)
- **CSRF:** All forms include form key validation
- **XSS:** Output escaping enforced in templates
- **PII:** Customer data treated as sensitive; logging requires explicit exclusion of PII fields

## Performance Impact

- **FPC:** Customer-specific content uses private content mechanism (no cache pollution)
- **Database:** Customer grid indexed separately for admin performance
- **Sessions:** Redis backend recommended for horizontal scaling
- **CWV:** Customer JS widgets lazy-loaded; minimal blocking resources

## Backward Compatibility

- **API Stability:** Service contract interfaces guaranteed stable within 2.4.x
- **Database:** EAV schema additions backward compatible
- **Deprecations:** Direct model usage deprecated in favor of repositories (2.3+)
- **Migration:** No breaking changes expected in 2.4.7 to 2.4.8 upgrade

## Tests to Add

- **Unit:** Repository methods, validation logic, group assignment
- **Integration:** Customer CRUD via service contracts, address management, attribute handling
- **MFTF:** Registration flow, login flow, password reset, address CRUD
- **API:** REST/GraphQL contract tests for customer and address endpoints

## Docs to Update

- **README.md:** This file for module overview
- **ARCHITECTURE.md:** Service contracts and database schema
- **EXECUTION_FLOWS.md:** Registration and authentication flows
- **CHANGELOG.md:** Version-specific changes and deprecations
- **Admin Guide:** Screenshots for customer configuration pages (Stores > Configuration > Customers)

## Related Guides

- [B2B Features Development](../../guides/explanations/b2b-features.md)
- [EAV System Architecture: Understanding Entity-Attribute-Value in Magento 2](../../guides/explanations/eav-system.md)
- [GraphQL Resolver Patterns in Magento 2](../../guides/explanations/graphql-resolver-patterns.md)
- [Service Contracts vs Repositories in Magento 2](../../guides/explanations/service-contracts-repositories.md)
- [Building Admin UI Components in Magento 2](../../guides/how-to/admin-ui-components.md)
- [Email Templates Customization](../../guides/how-to/email-templates.md)
- [ERP Integration Patterns for Magento 2](../../guides/how-to/erp-integration.md)
- [Import/Export System Deep Dive](../../guides/how-to/import-export.md)
- [Multi-Store Configuration Mastery](../../guides/how-to/multi-store-setup.md)
- [Security Checklist for Custom Modules](../../guides/how-to/security-checklist.md)
- [CLI Command Reference: Complete bin/magento Guide](../../guides/references/cli-command-reference.md)
- [Upgrade Guide: Adobe Commerce 2.4.6 to 2.4.7/2.4.8](../../guides/references/upgrade-guide-247-248.md)
- [Declarative Schema & Data Patches: Modern Database Management in Magento 2](../../guides/tutorials/declarative-schema-data-patches.md)
- [Plugin System Deep Dive: Mastering Magento 2 Interception](../../guides/tutorials/plugin-system-deep-dive.md)
