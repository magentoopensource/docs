---
title: "Magento_Customer Plugins & Observers"
module: "Magento_Customer"
doc_type: "plugins-and-observers"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Customer Plugins and Observers

## Overview

The Magento_Customer module exposes numerous extension points through **plugins** (interceptors) and **observers** (event handlers). These mechanisms allow developers to customize customer behavior without modifying core code, ensuring upgrade safety and maintainability.

This document catalogs all major plugin opportunities and observable events within the Customer module, provides real-world implementation examples, and explains best practices for extending customer functionality.

**Target Versions:** Magento 2.4.7+ / Adobe Commerce 2.4.7+ with PHP 8.2+

---

## Table of Contents

1. [Plugins vs Observers: When to Use Each](#plugins-vs-observers-when-to-use-each)
2. [Customer Authentication Plugins](#customer-authentication-plugins)
3. [Customer Save Plugins](#customer-save-plugins)
4. [Address Management Plugins](#address-management-plugins)
5. [Customer Repository Plugins](#customer-repository-plugins)
6. [Customer Session Plugins](#customer-session-plugins)
7. [Core Customer Events](#core-customer-events)
8. [Address Events](#address-events)
9. [Group Assignment Events](#group-assignment-events)
10. [Real-World Use Cases](#real-world-use-cases)

---

## Plugins vs Observers: When to Use Each

### Plugins (Interceptors)

**Use plugins when you need to:**
- Modify method arguments (before plugin)
- Change return values (after plugin)
- Wrap method execution (around plugin)
- Intercept service contract methods for API consistency
- Implement cross-cutting concerns (logging, caching, validation)

**Plugin Types:**
- **Before:** Execute before target method, can modify arguments
- **After:** Execute after target method, can modify return value
- **Around:** Wrap target method, full control over execution

**Example Use Cases:**
- Add custom validation before customer save
- Modify customer data before returning from API
- Log authentication attempts
- Integrate with external CRM systems

### Observers (Event Handlers)

**Use observers when you need to:**
- React to events without modifying core flow
- Execute side effects (send emails, update external systems)
- Implement loosely coupled extensions
- Listen to events that don't have direct method interception points

**Observer Characteristics:**
- Cannot modify method arguments or return values
- Multiple observers can listen to same event
- Execution order controllable via `sortOrder` attribute
- Fire-and-forget pattern (don't throw exceptions unless critical)

**Example Use Cases:**
- Send customer data to marketing automation platform
- Update loyalty points after registration
- Sync customer addresses with ERP system
- Log customer login events

---

## Customer Authentication Plugins

### AccountManagementInterface::authenticate()

**Plugin Point:** `Magento\Customer\Api\AccountManagementInterface::authenticate`

**Purpose:** Intercept customer authentication to add custom validation, logging, or integration with external auth systems.

#### Before Plugin: Add Pre-Authentication Validation

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Model;

use Magento\Customer\Api\AccountManagementInterface;
use Magento\Framework\Exception\AuthenticationException;
use Psr\Log\LoggerInterface;

class AccountManagementAuthenticationValidatorExtend
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Validate customer before authentication
     *
     * @param AccountManagementInterface $subject
     * @param string $username
     * @param string $password
     * @return array
     * @throws AuthenticationException
     */
    public function beforeAuthenticate(
        AccountManagementInterface $subject,
        string $username,
        string $password
    ): array {
        // Custom validation: check if email is from blocked domain
        $blockedDomains = ['disposable-email.com', 'tempmail.com'];
        $emailDomain = substr(strrchr($username, '@'), 1);

        if (in_array($emailDomain, $blockedDomains, true)) {
            $this->logger->warning('Authentication attempt from blocked domain', [
                'email' => $username,
                'domain' => $emailDomain
            ]);

            throw new AuthenticationException(
                __('Authentication is not available for this email domain.')
            );
        }

        // Return original arguments (required for before plugins)
        return [$username, $password];
    }
}
```

#### After Plugin: Log Successful Authentication

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Model;

use Magento\Customer\Api\AccountManagementInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Psr\Log\LoggerInterface;

class AccountManagementAuthenticationLoggerExtend
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
        $this->logger->info('Customer authenticated successfully', [
            'customer_id' => $result->getId(),
            'email' => $result->getEmail(),
            'group_id' => $result->getGroupId(),
            'timestamp' => date('Y-m-d H:i:s')
        ]);

        return $result;
    }
}
```

#### Around Plugin: Implement SSO Integration

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Model;

use Magento\Customer\Api\AccountManagementInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Exception\AuthenticationException;
use Vendor\Module\Service\SsoAuthenticationService;

class AccountManagementSsoExtend
{
    public function __construct(
        private readonly SsoAuthenticationService $ssoAuth
    ) {}

    /**
     * Wrap authentication to check SSO first
     *
     * @param AccountManagementInterface $subject
     * @param callable $proceed
     * @param string $username
     * @param string $password
     * @return CustomerInterface
     * @throws AuthenticationException
     */
    public function aroundAuthenticate(
        AccountManagementInterface $subject,
        callable $proceed,
        string $username,
        string $password
    ): CustomerInterface {
        // Check if SSO token provided instead of password
        if ($this->ssoAuth->isSsoToken($password)) {
            return $this->ssoAuth->authenticateWithToken($username, $password);
        }

        // Fall back to standard authentication
        return $proceed($username, $password);
    }
}
```

**DI Configuration:**

```xml
<!-- etc/di.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Customer\Api\AccountManagementInterface">
        <plugin name="vendor_module_auth_validator"
                type="Vendor\Module\Plugin\Customer\Model\AccountManagementAuthenticationValidatorExtend"
                sortOrder="10" />
        <plugin name="vendor_module_auth_logger"
                type="Vendor\Module\Plugin\Customer\Model\AccountManagementAuthenticationLoggerExtend"
                sortOrder="20" />
        <plugin name="vendor_module_sso_auth"
                type="Vendor\Module\Plugin\Customer\Model\AccountManagementSsoExtend"
                sortOrder="5"
                disabled="false" />
    </type>
</config>
```

---

## Customer Save Plugins

### CustomerRepositoryInterface::save()

**Plugin Point:** `Magento\Customer\Api\CustomerRepositoryInterface::save`

**Purpose:** Modify customer data before/after save, trigger external integrations, add custom validation.

#### Before Plugin: Validate Custom Attribute

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Api;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Exception\LocalizedException;

class CustomerRepositoryValidatorExtend
{
    /**
     * Validate custom attribute before save
     *
     * @param CustomerRepositoryInterface $subject
     * @param CustomerInterface $customer
     * @param string|null $passwordHash
     * @return array
     * @throws LocalizedException
     */
    public function beforeSave(
        CustomerRepositoryInterface $subject,
        CustomerInterface $customer,
        ?string $passwordHash = null
    ): array {
        // Validate loyalty number format (if provided)
        $customAttributes = $customer->getCustomAttributes();
        if ($customAttributes) {
            foreach ($customAttributes as $attribute) {
                if ($attribute->getAttributeCode() === 'loyalty_number') {
                    $loyaltyNumber = $attribute->getValue();
                    if (!preg_match('/^LOY-\d{8}$/', $loyaltyNumber)) {
                        throw new LocalizedException(
                            __('Loyalty number must be in format LOY-XXXXXXXX')
                        );
                    }
                }
            }
        }

        return [$customer, $passwordHash];
    }
}
```

#### After Plugin: Sync with External CRM

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Api;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Psr\Log\LoggerInterface;
use Vendor\Module\Service\CrmSyncService;

class CustomerRepositoryCrmSyncExtend
{
    public function __construct(
        private readonly CrmSyncService $crmSync,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Sync customer to CRM after save
     *
     * @param CustomerRepositoryInterface $subject
     * @param CustomerInterface $result
     * @return CustomerInterface
     */
    public function afterSave(
        CustomerRepositoryInterface $subject,
        CustomerInterface $result
    ): CustomerInterface {
        try {
            $this->crmSync->syncCustomer($result);
        } catch (\Exception $e) {
            // Don't fail save if CRM sync fails, just log
            $this->logger->error('CRM sync failed', [
                'customer_id' => $result->getId(),
                'error' => $e->getMessage()
            ]);
        }

        return $result;
    }
}
```

---

## Address Management Plugins

### AddressRepositoryInterface::save()

**Plugin Point:** `Magento\Customer\Api\AddressRepositoryInterface::save`

#### Before Plugin: Validate Address with External Service

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Api;

use Magento\Customer\Api\AddressRepositoryInterface;
use Magento\Customer\Api\Data\AddressInterface;
use Magento\Framework\Exception\LocalizedException;
use Vendor\Module\Service\AddressValidationService;

class AddressRepositoryValidationExtend
{
    public function __construct(
        private readonly AddressValidationService $addressValidator
    ) {}

    /**
     * Validate address with external service
     *
     * @param AddressRepositoryInterface $subject
     * @param AddressInterface $address
     * @return array
     * @throws LocalizedException
     */
    public function beforeSave(
        AddressRepositoryInterface $subject,
        AddressInterface $address
    ): array {
        // Validate with USPS/Google Maps/etc.
        $isValid = $this->addressValidator->validate(
            $address->getStreet(),
            $address->getCity(),
            $address->getRegionId(),
            $address->getPostcode(),
            $address->getCountryId()
        );

        if (!$isValid) {
            throw new LocalizedException(
                __('Address validation failed. Please verify your address details.')
            );
        }

        return [$address];
    }
}
```

#### After Plugin: Geocode Address

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Api;

use Magento\Customer\Api\AddressRepositoryInterface;
use Magento\Customer\Api\Data\AddressInterface;
use Vendor\Module\Service\GeocodingService;

class AddressRepositoryGeocodingExtend
{
    public function __construct(
        private readonly GeocodingService $geocoding
    ) {}

    /**
     * Add latitude/longitude to address after save
     *
     * @param AddressRepositoryInterface $subject
     * @param AddressInterface $result
     * @return AddressInterface
     */
    public function afterSave(
        AddressRepositoryInterface $subject,
        AddressInterface $result
    ): AddressInterface {
        // Get coordinates
        $coordinates = $this->geocoding->geocode(
            $result->getStreet(),
            $result->getCity(),
            $result->getPostcode(),
            $result->getCountryId()
        );

        if ($coordinates) {
            // Store as extension attributes
            $extensionAttributes = $result->getExtensionAttributes();
            $extensionAttributes->setLatitude($coordinates['lat']);
            $extensionAttributes->setLongitude($coordinates['lng']);
            $result->setExtensionAttributes($extensionAttributes);

            // Save coordinates (in real implementation, save to DB)
        }

        return $result;
    }
}
```

---

## Customer Repository Plugins

### CustomerRepositoryInterface::getById()

**Plugin Point:** `Magento\Customer\Api\CustomerRepositoryInterface::getById`

#### After Plugin: Load Additional Data

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Api;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Vendor\Module\Service\LoyaltyPointsService;

class CustomerRepositoryLoyaltyExtend
{
    public function __construct(
        private readonly LoyaltyPointsService $loyaltyPoints
    ) {}

    /**
     * Load loyalty points into extension attributes
     *
     * @param CustomerRepositoryInterface $subject
     * @param CustomerInterface $result
     * @return CustomerInterface
     */
    public function afterGetById(
        CustomerRepositoryInterface $subject,
        CustomerInterface $result
    ): CustomerInterface {
        $extensionAttributes = $result->getExtensionAttributes();
        $points = $this->loyaltyPoints->getPointsForCustomer((int)$result->getId());
        $extensionAttributes->setLoyaltyPoints($points);
        $result->setExtensionAttributes($extensionAttributes);

        return $result;
    }
}
```

---

## Customer Session Plugins

### Session::setCustomerDataAsLoggedIn()

**Plugin Point:** `Magento\Customer\Model\Session::setCustomerDataAsLoggedIn`

#### After Plugin: Load Additional Session Data

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Model;

use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Customer\Model\Session;
use Vendor\Module\Service\CustomerPreferencesService;

class SessionExtend
{
    public function __construct(
        private readonly CustomerPreferencesService $preferences
    ) {}

    /**
     * Load customer preferences into session
     *
     * @param Session $subject
     * @param Session $result
     * @param CustomerInterface $customer
     * @return Session
     */
    public function afterSetCustomerDataAsLoggedIn(
        Session $subject,
        Session $result,
        CustomerInterface $customer
    ): Session {
        // Load customer preferences
        $preferences = $this->preferences->getPreferences((int)$customer->getId());
        $result->setData('customer_preferences', $preferences);

        return $result;
    }
}
```

---

## Core Customer Events

### customer_register_success

**Dispatched:** After successful customer registration
**Location:** `Magento\Customer\Controller\Account\CreatePost::execute()`

**Event Data:**
- `customer` - CustomerInterface object
- `account_controller` - Controller instance

**Use Cases:**
- Send welcome email with discount code
- Assign initial loyalty points
- Create customer record in external system
- Trigger marketing automation workflow

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Vendor\Module\Service\WelcomeBonusService;

class CustomerRegisterSuccessObserver implements ObserverInterface
{
    public function __construct(
        private readonly WelcomeBonusService $welcomeBonus
    ) {}

    /**
     * Award welcome bonus after registration
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var CustomerInterface $customer */
        $customer = $observer->getEvent()->getCustomer();

        // Award 100 loyalty points for new registration
        $this->welcomeBonus->awardPoints((int)$customer->getId(), 100);
    }
}
```

### customer_login

**Dispatched:** After successful customer login
**Location:** `Magento\Customer\Model\Session::setCustomerAsLoggedIn()` and `Session::setCustomerDataAsLoggedIn()`

**Event Data:**
- `customer` - Customer model object

**Use Cases:**
- Update last login timestamp
- Log login attempts for security audit
- Sync login status with external systems
- Load customer-specific cache

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Customer\Model\Customer;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\Stdlib\DateTime\DateTime;

class CustomerLoginObserver implements ObserverInterface
{
    public function __construct(
        private readonly DateTime $dateTime
    ) {}

    /**
     * Update last login timestamp
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Customer $customer */
        $customer = $observer->getEvent()->getCustomer();

        // Update custom attribute
        $customer->setData('last_login_at', $this->dateTime->gmtDate());
        $customer->getResource()->saveAttribute($customer, 'last_login_at');
    }
}
```

### customer_logout

**Dispatched:** After customer logout
**Location:** `Magento\Customer\Model\Session::logout()`

**Event Data:**
- `customer` - Customer model object

**Use Cases:**
- Clear customer-specific cache
- Log logout event
- Update session analytics

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Psr\Log\LoggerInterface;

class CustomerLogoutObserver implements ObserverInterface
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Log customer logout
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        $customer = $observer->getEvent()->getCustomer();

        $this->logger->info('Customer logged out', [
            'customer_id' => $customer->getId(),
            'email' => $customer->getEmail()
        ]);
    }
}
```

### customer_save_before

**Dispatched:** Before customer entity is saved
**Location:** `Magento\Framework\Model\AbstractModel::beforeSave()` (dispatched by framework, not ResourceModel)

**Event Data:**
- `customer` - Customer model object
- `data_object` - Customer data object

**Use Cases:**
- Modify customer data before save
- Add custom validation
- Generate computed fields

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Customer\Model\Customer;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

class CustomerSaveBeforeObserver implements ObserverInterface
{
    /**
     * Generate customer display name
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Customer $customer */
        $customer = $observer->getEvent()->getCustomer();

        // Generate full display name
        $displayName = trim(sprintf(
            '%s %s %s %s',
            $customer->getPrefix() ?? '',
            $customer->getFirstname(),
            $customer->getMiddlename() ?? '',
            $customer->getLastname()
        ));

        $customer->setData('display_name', $displayName);
    }
}
```

### customer_save_after

**Dispatched:** After customer entity is saved
**Location:** `Magento\Framework\Model\AbstractModel::afterSave()` (dispatched by framework, not ResourceModel)

**Event Data:**
- `customer` - Customer model object
- `data_object` - Customer data object

**Use Cases:**
- Sync to external systems
- Invalidate cache
- Update related entities

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Customer\Model\Customer;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Vendor\Module\Service\CustomerIndexService;

class CustomerSaveAfterObserver implements ObserverInterface
{
    public function __construct(
        private readonly CustomerIndexService $indexer
    ) {}

    /**
     * Reindex customer data
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Customer $customer */
        $customer = $observer->getEvent()->getCustomer();

        // Queue reindex
        $this->indexer->scheduleReindex((int)$customer->getId());
    }
}
```

### customer_delete_before

**Dispatched:** Before customer is deleted
**Location:** `Magento\Framework\Model\AbstractModel::beforeDelete()` (dispatched by framework, not ResourceModel)

**Event Data:**
- `customer` - Customer model object

**Use Cases:**
- Archive customer data
- Remove related data
- GDPR compliance logging

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Customer\Model\Customer;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Vendor\Module\Service\GdprArchiveService;

class CustomerDeleteBeforeObserver implements ObserverInterface
{
    public function __construct(
        private readonly GdprArchiveService $archive
    ) {}

    /**
     * Archive customer data before deletion
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Customer $customer */
        $customer = $observer->getEvent()->getCustomer();

        // Archive for GDPR compliance
        $this->archive->archiveCustomerData($customer);
    }
}
```

---

## Address Events

### customer_address_save_before / customer_address_save_after

**Use Cases:**
- Validate address with external service
- Geocode address coordinates
- Update address book in ERP

### customer_address_delete_before / customer_address_delete_after

**Use Cases:**
- Clean up related data
- Update external systems

---

## Group Assignment Events

### customer_group_save_after

**Dispatched:** After customer group is saved

**Use Case:** Update pricing rules, invalidate cache

---

## Real-World Use Cases

### Use Case 1: Two-Factor Authentication Integration

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Model;

use Magento\Customer\Api\AccountManagementInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Exception\AuthenticationException;
use Vendor\Module\Service\TwoFactorAuthService;

class AccountManagementTwoFactorExtend
{
    public function __construct(
        private readonly TwoFactorAuthService $twoFactorAuth
    ) {}

    /**
     * Require 2FA after password authentication
     *
     * @param AccountManagementInterface $subject
     * @param CustomerInterface $result
     * @return CustomerInterface
     * @throws AuthenticationException
     */
    public function afterAuthenticate(
        AccountManagementInterface $subject,
        CustomerInterface $result
    ): CustomerInterface {
        // Check if customer has 2FA enabled
        if ($this->twoFactorAuth->isEnabledForCustomer((int)$result->getId())) {
            // Store pending authentication state
            $this->twoFactorAuth->setPendingAuthentication((int)$result->getId());

            throw new AuthenticationException(
                __('Two-factor authentication required. Please enter your code.')
            );
        }

        return $result;
    }
}
```

### Use Case 2: Customer Tier Auto-Assignment

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Customer\Model\Customer;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;

class CustomerTierAssignmentObserver implements ObserverInterface
{
    public function __construct(
        private readonly OrderRepositoryInterface $orderRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder
    ) {}

    /**
     * Auto-assign customer tier based on order history
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Customer $customer */
        $customer = $observer->getEvent()->getCustomer();

        // Calculate total order value
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('customer_id', $customer->getId())
            ->create();

        $orders = $this->orderRepository->getList($searchCriteria);
        $totalSpent = 0;

        foreach ($orders->getItems() as $order) {
            $totalSpent += (float)$order->getGrandTotal();
        }

        // Assign tier based on spend
        if ($totalSpent >= 10000) {
            $customer->setGroupId(4); // VIP group
        } elseif ($totalSpent >= 5000) {
            $customer->setGroupId(3); // Premium group
        } elseif ($totalSpent >= 1000) {
            $customer->setGroupId(2); // Regular group
        }

        $customer->save();
    }
}
```

---

## Observer Registration

```xml
<!-- etc/events.xml (global) -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="customer_save_after">
        <observer name="vendor_module_customer_save_after"
                  instance="Vendor\Module\Observer\CustomerSaveAfterObserver"
                  shared="false" />
    </event>
</config>

<!-- etc/frontend/events.xml (frontend only) -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="customer_login">
        <observer name="vendor_module_customer_login"
                  instance="Vendor\Module\Observer\CustomerLoginObserver" />
    </event>
    <event name="customer_logout">
        <observer name="vendor_module_customer_logout"
                  instance="Vendor\Module\Observer\CustomerLogoutObserver" />
    </event>
</config>

<!-- etc/adminhtml/events.xml (admin only) -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="customer_save_before">
        <observer name="vendor_module_admin_customer_save_before"
                  instance="Vendor\Module\Observer\AdminCustomerSaveBeforeObserver" />
    </event>
</config>
```

---

## Best Practices

### Plugin Best Practices

1. **Use Specific Sort Order:** Define explicit `sortOrder` to control execution sequence
2. **Prefer After Over Around:** Use `around` only when absolutely necessary (performance impact)
3. **Don't Modify Arguments in After:** After plugins should only modify return values
4. **Return Original Value:** Before plugins must return arguments array
5. **Handle Exceptions:** Catch and log exceptions to prevent breaking core functionality

### Observer Best Practices

1. **Keep Logic Lightweight:** Heavy operations should use message queues
2. **Don't Throw Exceptions:** Observers shouldn't break core flow (log errors instead)
3. **Use Appropriate Area:** Register in correct `events.xml` (global/frontend/adminhtml)
4. **Avoid Direct Model Save:** Use repositories to trigger proper events and validation
5. **Set shared="false":** For observers that maintain state, prevent singleton issues

---

## Assumptions

- **Target Platform:** Adobe Commerce / Magento Open Source 2.4.7+
- **PHP Version:** 8.2+
- **Plugin Scope:** Plugins apply to all areas unless specified in area-specific `di.xml`
- **Observer Scope:** Observers registered in appropriate area configuration

## Why This Approach

Plugins and observers provide upgrade-safe extension points that don't require core modifications. Service contract plugins ensure API consistency across REST, GraphQL, and PHP integrations, while observers enable loosely coupled reactions to business events.

## Security Impact

- **Authorization:** Plugins and observers have full access; implement proper validation
- **Data Exposure:** Be cautious logging customer PII in plugins/observers
- **Exception Handling:** Plugins can block execution; observers should fail gracefully

## Performance Impact

- **Around Plugins:** Highest overhead; use sparingly
- **Observer Chains:** Multiple observers on same event execute sequentially
- **Heavy Operations:** Use message queues for time-consuming tasks

## Backward Compatibility

- **Plugin Stability:** Plugin on service contracts safe across minor versions
- **Event Stability:** Events never removed, only added
- **Observer Registration:** Configuration format stable

## Tests to Add

- **Unit:** Test observer and plugin logic in isolation with mocks
- **Integration:** Test observer/plugin execution in real Magento context
- **MFTF:** Test end-to-end flows that trigger plugins/observers

## Docs to Update

- **README.md:** List available extension points
- **PLUGINS_AND_OBSERVERS.md:** This document
- **CHANGELOG.md:** Document new events added in version updates
