---
title: "Magento_Customer Known Issues"
module: "Magento_Customer"
doc_type: "known-issues"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Customer Known Issues

## Overview

This document catalogs known issues, bugs, limitations, and workarounds in the Magento_Customer module across different versions of Adobe Commerce and Magento Open Source. Understanding these issues helps developers anticipate problems, implement preventive measures, and apply appropriate workarounds.

**Target Versions:** Magento 2.4.0 through 2.4.8+ / Adobe Commerce equivalent

---

## Table of Contents

1. [Session Fixation Vulnerabilities](#session-fixation-vulnerabilities)
2. [Customer Grid Performance Issues](#customer-grid-performance-issues)
3. [Address Validation Inconsistencies](#address-validation-inconsistencies)
4. [Email Confirmation Loop](#email-confirmation-loop)
5. [Customer Account Lockout Issues](#customer-account-lockout-issues)
6. [Multi-Website Customer Sharing Problems](#multi-website-customer-sharing-problems)
7. [Password Reset Token Expiration](#password-reset-token-expiration)
8. [Customer Attribute Indexing Delays](#customer-attribute-indexing-delays)
9. [GraphQL Customer Mutation Limitations](#graphql-customer-mutation-limitations)
10. [VAT Validation Service Failures](#vat-validation-service-failures)
11. [Customer Session Timeout Issues](#customer-session-timeout-issues)
12. [REST API Customer Creation Race Conditions](#rest-api-customer-creation-race-conditions)

---

## Session Fixation Vulnerabilities

### Issue Description

**Versions Affected:** 2.4.0 - 2.4.3 (fixed in 2.4.4)

**Severity:** Critical (Security)

Prior to Magento 2.4.4, the customer session ID was not regenerated after successful authentication, making the application vulnerable to session fixation attacks. An attacker could set a known session ID before authentication and hijack the session after the victim logs in.

### Symptoms

- Session ID remains the same before and after login
- Session data accessible with pre-authentication session cookie
- Potential for session hijacking attacks

### Root Cause

`Magento\Customer\Model\Session::setCustomerDataAsLoggedIn()` did not call `regenerateId()` prior to 2.4.4.

### Impact

- **Security:** High - Session hijacking possible
- **Compliance:** Violates PCI DSS 6.5.10 (session management)
- **User Trust:** Potential account takeover

### Workaround (for versions < 2.4.4)

Create a plugin to force session regeneration after login:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Model;

use Magento\Customer\Model\Session;
use Magento\Customer\Api\Data\CustomerInterface;

class SessionSecurityExtend
{
    /**
     * Regenerate session ID after login to prevent fixation
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
        // Force session regeneration
        if (!$result->getSessionId() || $result->getSessionId() === $result->getOldSessionId()) {
            $result->regenerateId();
        }

        return $result;
    }
}
```

```xml
<!-- etc/frontend/di.xml -->
<type name="Magento\Customer\Model\Session">
    <plugin name="vendor_module_session_security"
            type="Vendor\Module\Plugin\Customer\Model\SessionSecurityExtend"
            sortOrder="10" />
</type>
```

### Official Fix

**Upgrade to 2.4.4+** where session regeneration is built-in.

**Reference:** Adobe Security Bulletin APSB21-64

---

## Customer Grid Performance Issues

### Issue Description

**Versions Affected:** All versions (performance degrades with customer count)

**Severity:** Medium (Performance)

The admin customer grid (`customer_grid_flat` table) becomes slow when the customer database exceeds 100,000+ records, particularly when filtering by custom attributes or performing full-text searches.

### Symptoms

- Admin customer grid load times > 10 seconds
- Timeout errors when applying filters
- Database CPU spikes when accessing customer admin pages
- Slow mass action operations

### Root Cause

- `customer_grid_flat` table not properly indexed for custom columns
- Full-text search scans entire table without optimization
- Join operations on large EAV attribute tables

### Impact

- **Performance:** Admin operations slow or timeout
- **User Experience:** Admin users frustrated
- **Scalability:** Limits merchant growth

### Workaround

#### 1. Optimize Customer Grid Index

```bash
# Reindex customer grid
bin/magento indexer:reindex customer_grid

# Schedule regular reindexing (crontab)
*/30 * * * * /path/to/magento/bin/magento indexer:reindex customer_grid
```

#### 2. Add Custom Indexes

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Schema;

use Magento\Framework\Setup\Patch\SchemaPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;

class AddCustomerGridIndexes implements SchemaPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup
    ) {}

    public function apply(): self
    {
        $connection = $this->moduleDataSetup->getConnection();
        $tableName = $this->moduleDataSetup->getTable('customer_grid_flat');

        // Add indexes for frequently filtered columns
        if (!$connection->getIndexList($tableName)['IDX_CUSTOMER_GRID_GROUP_CREATED'] ?? false) {
            $connection->addIndex(
                $tableName,
                'IDX_CUSTOMER_GRID_GROUP_CREATED',
                ['group_id', 'created_at']
            );
        }

        if (!$connection->getIndexList($tableName)['IDX_CUSTOMER_GRID_BILLING_COUNTRY'] ?? false) {
            $connection->addIndex(
                $tableName,
                'IDX_CUSTOMER_GRID_BILLING_COUNTRY',
                ['billing_country_id']
            );
        }

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

#### 3. Optimize Customer Grid Performance

> **Note:** There is no `customer/search/enable_elasticsearch` config path in core Magento.
> The customer admin grid uses a flat table (`customer_grid_flat`), not Elasticsearch.

```bash
# Reindex the customer grid flat table
bin/magento indexer:reindex customer_grid
bin/magento cache:flush
```

### Best Practices

- Archive inactive customers to separate table
- Use pagination in admin grid (limit to 20-50 records per page)
- Minimize custom attribute columns in grid
- Schedule grid reindexing during off-peak hours

---

## Address Validation Inconsistencies

### Issue Description

**Versions Affected:** All versions

**Severity:** Medium (Data Integrity)

Address validation rules are inconsistent between frontend, admin, API, and checkout contexts. Some validations (e.g., required telephone, street line count) are enforced in frontend forms but not in REST API or admin.

### Symptoms

- Customers can save addresses via API without telephone
- Admin can create addresses with > 4 street lines
- Checkout validates differently than account address book
- Addresses saved via GraphQL skip region validation

### Root Cause

Validation logic scattered across:
- Form validators (`Magento\Customer\Model\Address\Validator\*`)
- Controller validation
- UI component validation
- API lacks comprehensive validation layer

### Impact

- **Data Quality:** Inconsistent address data
- **Shipping Errors:** Invalid addresses cause carrier API failures
- **Tax Calculation:** Incorrect region causes wrong tax rates

### Workaround

Create a centralized address validation service:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\Data\AddressInterface;
use Magento\Directory\Model\ResourceModel\Region\CollectionFactory;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\App\Config\ScopeConfigInterface;

class AddressValidationService
{
    private const MAX_STREET_LINES = 4;

    public function __construct(
        private readonly CollectionFactory $regionCollectionFactory,
        private readonly ScopeConfigInterface $scopeConfig
    ) {}

    /**
     * Comprehensive address validation
     *
     * @param AddressInterface $address
     * @throws LocalizedException
     */
    public function validate(AddressInterface $address): void
    {
        $errors = [];

        // Validate required fields
        if (!$address->getFirstname()) {
            $errors[] = __('First name is required');
        }

        if (!$address->getLastname()) {
            $errors[] = __('Last name is required');
        }

        if (!$address->getStreet() || count($address->getStreet()) === 0) {
            $errors[] = __('Street address is required');
        }

        if (!$address->getCity()) {
            $errors[] = __('City is required');
        }

        if (!$address->getCountryId()) {
            $errors[] = __('Country is required');
        }

        if (!$address->getTelephone()) {
            $errors[] = __('Telephone is required');
        }

        // Validate street lines count
        if (is_array($address->getStreet()) && count($address->getStreet()) > self::MAX_STREET_LINES) {
            $errors[] = __('Address can have maximum %1 street lines', self::MAX_STREET_LINES);
        }

        // Validate region for countries that require it
        if ($this->isRegionRequired($address->getCountryId()) && !$address->getRegionId()) {
            $errors[] = __('State/Province is required');
        }

        // Validate region belongs to country
        if ($address->getRegionId()) {
            $this->validateRegionCountry($address->getRegionId(), $address->getCountryId(), $errors);
        }

        // Validate postal code format
        if ($address->getPostcode() && !$this->validatePostcode($address->getPostcode(), $address->getCountryId())) {
            $errors[] = __('Invalid postal code format');
        }

        if (!empty($errors)) {
            throw new LocalizedException(
                __('Address validation failed: %1', implode(', ', $errors))
            );
        }
    }

    private function isRegionRequired(string $countryId): bool
    {
        $regionsRequired = ['US', 'CA', 'AU'];
        return in_array($countryId, $regionsRequired, true);
    }

    private function validateRegionCountry(int $regionId, string $countryId, array &$errors): void
    {
        $regionCollection = $this->regionCollectionFactory->create();
        $regionCollection->addRegionIdFilter($regionId)
            ->addCountryFilter($countryId);

        if ($regionCollection->getSize() === 0) {
            $errors[] = __('Selected region does not belong to country');
        }
    }

    private function validatePostcode(string $postcode, string $countryId): bool
    {
        $patterns = [
            'US' => '/^\d{5}(-\d{4})?$/',
            'CA' => '/^[A-Z]\d[A-Z] \d[A-Z]\d$/',
            'GB' => '/^[A-Z]{1,2}\d{1,2}[A-Z]? \d[A-Z]{2}$/',
        ];

        if (!isset($patterns[$countryId])) {
            return true; // No validation pattern defined
        }

        return (bool)preg_match($patterns[$countryId], $postcode);
    }
}
```

Apply validation via plugin:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Api;

use Magento\Customer\Api\AddressRepositoryInterface;
use Magento\Customer\Api\Data\AddressInterface;
use Vendor\Module\Service\AddressValidationService;

class AddressRepositoryValidationExtend
{
    public function __construct(
        private readonly AddressValidationService $addressValidator
    ) {}

    public function beforeSave(
        AddressRepositoryInterface $subject,
        AddressInterface $address
    ): array {
        $this->addressValidator->validate($address);
        return [$address];
    }
}
```

---

## Email Confirmation Loop

### Issue Description

**Versions Affected:** 2.4.0 - 2.4.5 (intermittent)

**Severity:** Medium (User Experience)

Customers with email confirmation enabled sometimes get stuck in a loop where the confirmation email is never sent or the confirmation link is invalid. This particularly affects customers created via admin or API.

### Symptoms

- Customer account created but confirmation email not received
- Confirmation link shows "Invalid or expired confirmation token"
- Customer cannot log in (account pending confirmation)
- Resending confirmation generates new token but link still fails

### Root Cause

- Race condition between customer save and email queue
- Confirmation token overwritten by subsequent save operations
- Email template store scope mismatch
- Async email sending (if enabled) fails silently

### Impact

- **Customer Frustration:** Cannot access account
- **Support Burden:** Increased support tickets
- **Revenue Loss:** Customers abandon registration

### Workaround

#### 1. Manual Confirmation Activation

Admin can manually activate customer:

```sql
-- WARNING: Direct DB modification, use with caution
UPDATE customer_entity
SET confirmation = NULL
WHERE entity_id = <customer_id>;
```

#### 2. Programmatic Confirmation

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Model\CustomerRegistry;

class CustomerConfirmationService
{
    public function __construct(
        private readonly CustomerRepositoryInterface $customerRepository,
        private readonly CustomerRegistry $customerRegistry
    ) {}

    /**
     * Manually confirm customer account
     *
     * @param int $customerId
     */
    public function confirmCustomer(int $customerId): void
    {
        $customer = $this->customerRegistry->retrieve($customerId);

        // Clear confirmation token
        $customer->setConfirmation(null);
        $customer->save();

        // Refresh registry
        $this->customerRegistry->remove($customerId);
    }
}
```

#### 3. Disable Email Confirmation for API/Admin Created Customers

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Api;

use Magento\Customer\Api\AccountManagementInterface;
use Magento\Customer\Api\Data\CustomerInterface;

class DisableConfirmationForAdminExtend
{
    public function beforeCreateAccount(
        AccountManagementInterface $subject,
        CustomerInterface $customer,
        ?string $password = null,
        string $redirectUrl = ''
    ): array {
        // Mark customer as confirmed for admin-created accounts
        $customer->setConfirmation(null);

        return [$customer, $password, $redirectUrl];
    }
}
```

### Prevention

- Monitor email queue: `bin/magento queue:consumers:start async.operations.all`
- Test email templates in all store views
- Use synchronous email sending for critical emails

---

## Customer Account Lockout Issues

### Issue Description

**Versions Affected:** 2.4.0+

**Severity:** High (Security vs. UX trade-off)

Brute-force protection can inadvertently lock out legitimate customers, particularly in these scenarios:
- Multiple login attempts with password manager auto-fill
- Shared email used by multiple family members
- Bot traffic targeting legitimate customer emails

### Symptoms

- Customer locked out after 10 failed attempts
- Lockout persists even after correct password
- No admin interface to unlock customer
- Lock expiration not consistently enforced

### Root Cause

- `lock_expires` timestamp not always cleared properly
- No admin UI for manual unlock
- Lockout counter (`failures_num`) not reset on successful login in some edge cases

### Workaround

#### 1. Admin Unlock Script

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Console\Command;

use Magento\Customer\Model\ResourceModel\Customer as CustomerResource;
use Magento\Customer\Model\CustomerRegistry;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class UnlockCustomerCommand extends Command
{
    public function __construct(
        private readonly CustomerRegistry $customerRegistry,
        private readonly CustomerResource $customerResource,
        string $name = null
    ) {
        parent::__construct($name);
    }

    protected function configure(): void
    {
        $this->setName('customer:unlock')
            ->setDescription('Unlock customer account')
            ->addArgument('customer_id', InputArgument::REQUIRED, 'Customer ID');
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $customerId = (int)$input->getArgument('customer_id');

        try {
            $customer = $this->customerRegistry->retrieve($customerId);
            $customer->setFailuresNum(0);
            $customer->setFirstFailure(null);
            $customer->setLockExpires(null);

            $this->customerResource->save($customer);

            $output->writeln("<info>Customer {$customerId} unlocked successfully</info>");
            return Command::SUCCESS;
        } catch (\Exception $e) {
            $output->writeln("<error>Error: {$e->getMessage()}</error>");
            return Command::FAILURE;
        }
    }
}
```

#### 2. Auto-Unlock After Expiration

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Model;

use Magento\Customer\Model\AccountManagement;
use Magento\Framework\Stdlib\DateTime\DateTime;

class AccountManagementUnlockExtend
{
    public function __construct(
        private readonly DateTime $dateTime
    ) {}

    /**
     * Clear expired locks before authentication
     */
    public function beforeAuthenticate(
        AccountManagement $subject,
        string $username,
        string $password
    ): array {
        // Plugin would load customer and check lock_expires
        // If expired, clear lock fields

        return [$username, $password];
    }
}
```

### Configuration

Adjust lockout settings:

```bash
# Increase failure threshold
bin/magento config:set customer/password/lockout_failures 20

# Reduce lockout duration (seconds)
bin/magento config:set customer/password/lockout_threshold 300

# Clear config cache
bin/magento cache:flush config
```

---

## Multi-Website Customer Sharing Problems

### Issue Description

**Versions Affected:** All versions

**Severity:** Medium (Data Integrity)

When customer account sharing is set to "Global," issues arise:
- Customer updates on one website don't sync immediately to others
- Default addresses differ per website (unexpected behavior)
- Customer group changes on one website affect all websites
- Session data conflicts when customer browses multiple websites

### Symptoms

- Customer changes email on Website A, still sees old email on Website B
- Customer group differs between websites
- Cart merge issues when switching websites
- Address book shows different addresses per website

### Root Cause

- Cache invalidation not propagated across all websites
- Website-scoped session storage conflicts with global customer sharing
- Address website association not enforced

### Workaround

Ensure consistent cache invalidation:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Customer\Model\Customer;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\App\CacheInterface;

class InvalidateCustomerCacheObserver implements ObserverInterface
{
    public function __construct(
        private readonly CacheInterface $cache
    ) {}

    public function execute(Observer $observer): void
    {
        /** @var Customer $customer */
        $customer = $observer->getEvent()->getCustomer();

        // Invalidate customer-specific cache tags across all websites
        $this->cache->clean([
            'customer_' . $customer->getId(),
            'customer_section_' . $customer->getId()
        ]);
    }
}
```

### Best Practice

**Use per-website customer accounts** unless truly necessary:

```bash
bin/magento config:set customer/account_share/scope 1  # Per Website
```

---

## Password Reset Token Expiration

### Issue Description

**Versions Affected:** All versions

**Severity:** Low (User Experience)

Password reset tokens expire after configurable period (default 24 hours), but:
- No clear error message when token expires
- Multiple reset requests generate multiple tokens, creating confusion
- Token cleanup not automatic

### Workaround

Extend token validity:

```bash
# Extend to 48 hours
bin/magento config:set admin/security/password_reset_link_expiration_period 2
```

Custom error message:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Customer\Controller\Account;

use Magento\Customer\Controller\Account\CreatePassword;
use Magento\Framework\Controller\Result\Redirect;
use Magento\Framework\Message\ManagerInterface;

class CreatePasswordExtend
{
    public function __construct(
        private readonly ManagerInterface $messageManager
    ) {}

    public function afterExecute(
        CreatePassword $subject,
        $result
    ) {
        if ($result instanceof Redirect && strpos($result->getUrl(), 'noroute') !== false) {
            $this->messageManager->addErrorMessage(
                __('Your password reset link has expired. Please request a new one.')
            );
        }

        return $result;
    }
}
```

---

## Customer Attribute Indexing Delays

### Issue Description

**Versions Affected:** 2.4.0+

**Severity:** Medium (Performance)

Custom customer attributes added to admin grid don't appear immediately. The `customer_grid_flat` indexer runs on schedule (default: every 1 minute) causing delays.

### Workaround

Force immediate reindex:

```bash
bin/magento indexer:reindex customer_grid
```

Set indexer to "Update on Save" mode:

```bash
bin/magento indexer:set-mode realtime customer_grid
```

**Note:** Realtime mode impacts save performance with large customer bases.

---

## GraphQL Customer Mutation Limitations

### Issue Description

**Versions Affected:** 2.4.0 - 2.4.4

**Severity:** Medium (API Limitation)

GraphQL customer mutations have limitations:
- Cannot update custom attributes in `updateCustomer` mutation (2.4.0-2.4.2)
- Address mutations don't support VAT validation
- No bulk customer creation mutation
- Limited error messaging

### Workaround

Use REST API for advanced operations or upgrade to 2.4.5+ where custom attribute support improved.

---

## VAT Validation Service Failures

### Issue Description

**Versions Affected:** All versions (external dependency)

**Severity:** Low (External Dependency)

VAT validation via VIES (EU VAT validation service) fails intermittently:
- VIES service downtime
- Rate limiting by VIES
- Timeout errors (service slow)

### Workaround

Implement fallback validation:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\Data\AddressInterface;

class VatValidationService
{
    public function validateVat(AddressInterface $address): bool
    {
        $vatId = $address->getVatId();
        $countryId = $address->getCountryId();

        try {
            // Try VIES validation
            return $this->viesValidation($vatId, $countryId);
        } catch (\Exception $e) {
            // Fallback to format validation
            return $this->formatValidation($vatId, $countryId);
        }
    }

    private function formatValidation(string $vatId, string $countryId): bool
    {
        // Basic format checks for EU VAT numbers
        $patterns = [
            'GB' => '/^GB\d{9}$/',
            'DE' => '/^DE\d{9}$/',
            'FR' => '/^FR[A-Z0-9]{2}\d{9}$/',
        ];

        if (!isset($patterns[$countryId])) {
            return true; // Accept if no pattern
        }

        return (bool)preg_match($patterns[$countryId], $vatId);
    }
}
```

---

## Assumptions

- **Target Platform:** Adobe Commerce / Magento Open Source 2.4.0+
- **Environment:** Production deployments with standard configurations
- **Issue Tracking:** Adobe Commerce Support and GitHub issue tracker monitored

## Why This Document

Documenting known issues helps developers anticipate problems, implement preventive measures, and avoid time-consuming debugging. Many issues have workarounds that are not well-documented in official resources.

## Security Impact

- Session fixation (Critical - upgrade immediately if < 2.4.4)
- Account lockout (Balance security with UX)
- Password reset token handling (Moderate risk if tokens exposed)

## Performance Impact

- Customer grid indexing (High impact on large databases)
- Collection loading (Optimize queries and use pagination)
- Cache invalidation (Ensure proper cache clearing)

## Backward Compatibility

- Workarounds designed for backward compatibility
- Plugins preferred over rewrites
- Configuration changes reversible

## Tests to Add

- **Integration:** Test workarounds in realistic scenarios
- **Security:** Verify session regeneration after login
- **Performance:** Benchmark grid with 100k+ customers

## Docs to Update

- **KNOWN_ISSUES.md:** This document (update as new versions released)
- **CHANGELOG.md:** Note which issues fixed in which versions
- **README.md:** Link to known issues for developer awareness
