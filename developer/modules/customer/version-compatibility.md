---
title: "Magento_Customer Version Compatibility"
module: "Magento_Customer"
doc_type: "version-compatibility"
version: "2.4.7+"
last_updated: "2026-02-09"
---

# Magento_Customer Version Compatibility

## Overview

This document tracks verified API surface, deprecations, and compatibility information for the Magento_Customer module in version 2.4.7.

> **Note:** This document only contains facts verified against the Magento 2.4.7 source code. Version-to-version changelog entries that could not be verified against source have been removed.

**Module Version (2.4.7):** 103.0.7
**PHP Requirement:** `~8.1.0||~8.2.0||~8.3.0`

---

## Key Facts

### Interface Return Types

Magento 2.4.7 Customer API interfaces use **docblock `@return` annotations only** — they do NOT have PHP native return type declarations:

```php
// Actual CustomerRepositoryInterface in 2.4.7
// NO PHP return type declarations — only @return docblocks
public function save(\Magento\Customer\Api\Data\CustomerInterface $customer, $passwordHash = null);
public function get($email, $websiteId = null);
public function getById($customerId);
public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria);
public function delete(\Magento\Customer\Api\Data\CustomerInterface $customer);
public function deleteById($customerId);
```

Similarly, `AccountManagementInterface` has NO PHP return type declarations on any of its 19 methods.

### Database Schema: `session_cutoff` Column

The `customer_entity` table in 2.4.7 includes:

```xml
<column xsi:type="timestamp" name="session_cutoff" nullable="true" on_update="false"
        comment="Session Cutoff Time"/>
```

**Note:** The columns `last_password_change` and `mfa_enabled` do **NOT exist** in the 2.4.7 `customer_entity` schema.

### Customer Grid

The customer admin grid uses a **flat table** (`customer_grid_flat`), NOT Elasticsearch. There is no `customer/search/enable_elasticsearch` configuration path in the module.

---

## Composer Dependencies (2.4.7)

| Dependency | Version Constraint |
|---|---|
| magento/framework | 103.0.* |
| magento/module-authorization | 100.4.* |
| magento/module-backend | 102.0.* |
| magento/module-catalog | 104.0.* |
| magento/module-checkout | 100.4.* |
| magento/module-config | 101.2.* |
| magento/module-directory | 100.4.* |
| magento/module-eav | 102.1.* |
| magento/module-integration | 100.4.* |
| magento/module-media-storage | 100.4.* |
| magento/module-newsletter | 100.4.* |
| magento/module-page-cache | 100.4.* |
| magento/module-quote | 101.2.* |
| magento/module-sales | 103.0.* |
| magento/module-store | 101.1.* |
| magento/module-tax | 100.4.* |
| magento/module-theme | 101.1.* |
| magento/module-ui | 101.2.* |
| magento/module-wishlist | 101.2.* |

---

## API Interfaces

### Management Interfaces (Api/)

- `AccountManagementInterface`
- `AddressMetadataInterface`
- `AddressMetadataManagementInterface`
- `AddressRepositoryInterface`
- `CustomerMetadataInterface`
- `CustomerMetadataManagementInterface`
- `CustomerRepositoryInterface`
- `CustomerManagementInterface`
- `CustomerNameGenerationInterface`
- `CustomerGroupConfigInterface`
- `GroupManagementInterface`
- `GroupRepositoryInterface`
- `GroupExcludedWebsiteRepositoryInterface`
- `MetadataInterface`
- `MetadataManagementInterface`
- `SessionCleanerInterface`
- `AccountDelegationInterface`

### Data Interfaces (Api/Data/)

- `CustomerInterface`
- `AddressInterface`
- `GroupInterface`
- `RegionInterface`
- `AttributeMetadataInterface`
- `OptionInterface`
- `ValidationResultsInterface`
- `ValidationRuleInterface`
- `CustomerSearchResultsInterface`
- `AddressSearchResultsInterface`
- `GroupSearchResultsInterface`
- `GroupExcludedWebsiteInterface`

---

## Verified Deprecations (from `@deprecated` annotations in source)

### Deprecated Classes

| Class | Deprecated Since | Replacement |
|---|---|---|
| `Model\Observer\Grid` | 100.1.0 | — |
| `Model\Customer\DataProvider` | 102.0.1 | `DataProviderWithDefaultAddresses` |
| `Model\Customer\Attribute\Backend\Password` | 101.0.0 | — |
| `Model\ForgotPasswordToken\GetCustomerByToken` | (no version) | "Rp Tokens cannot be looked up directly" |

### Deprecated AccountManagement Constants

All email-related XML path constants in `AccountManagement` are deprecated with the note "Get rid of Helpers in Password Security Management":

- `XML_PATH_REGISTER_EMAIL_TEMPLATE`
- `XML_PATH_REGISTER_NO_PASSWORD_EMAIL_TEMPLATE`
- `XML_PATH_REGISTER_EMAIL_IDENTITY`
- `XML_PATH_REMIND_EMAIL_TEMPLATE`
- `XML_PATH_FORGOT_EMAIL_TEMPLATE`
- `XML_PATH_FORGOT_EMAIL_IDENTITY`
- `XML_PATH_IS_CONFIRM`
- `XML_PATH_CONFIRM_EMAIL_TEMPLATE`
- `XML_PATH_CONFIRMED_EMAIL_TEMPLATE`

See `EmailNotification` class for replacements.

### Deprecated Methods

| Class | Method | Deprecated Since |
|---|---|---|
| `Model\Account\Redirect` | `getCookieManager()` | 100.0.10 |
| `Model\Account\Redirect` | `setCookieManager()` | 100.0.10 |
| `Model\Account\Redirect` | `getRedirectRoute()` | 100.0.10 |
| `Model\Account\Redirect` | `saveRedirectRoute()` | 100.0.10 |
| `Model\Account\Redirect` | `clearRedirectRoute()` | 100.0.10 |
| `Controller\Account\Logout` | `getCookieManager()` | 100.1.0 |
| `Controller\Account\Logout` | `getCookieMetadataFactory()` | 100.1.0 |
| `Controller\Ajax\Login` | `getAccountRedirect()` | 100.0.10 |
| `Controller\Ajax\Login` | `setScopeConfig()` | 100.0.10 |

---

## `@since` Version Tags

Most API interfaces are tagged `@since 100.0.2`. Notable later additions:

| Class/Interface | @since |
|---|---|
| `Model\Address\ValidatorInterface` | 102.0.0 |
| `Model\EmailNotificationInterface` | 100.1.0 |
| `Model\AuthenticationInterface` | 100.1.0 |
| `Model\Group\RetrieverInterface` | 101.0.0 |
| `Block\Address\Grid` | 102.0.1 |
| `CustomerData\SectionPool::getSectionsData()` | 102.0.4 |

---

## Upgrade Best Practices

### Use Service Contracts

```php
// Recommended
$customer = $this->customerRepository->getById($customerId);

// Deprecated pattern (still works)
$customer = $this->customerFactory->create()->load($customerId);
```

### Session Handling

```php
// Recommended — returns CustomerInterface
$customerData = $this->customerSession->getCustomerData();

// Older pattern — returns Customer model
$customerModel = $this->customerSession->getCustomer();
```

---

**Disclaimer:** This document covers the Magento_Customer module as found in `magento/product-community-edition:2.4.7`. Version-to-version change details could not be verified from source alone and have been removed. For version-specific changelogs, consult the official Adobe Commerce release notes.
