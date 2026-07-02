---
title: "Magento_Customer Anti-Patterns"
module: "Magento_Customer"
doc_type: "anti-patterns"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Customer Anti-Patterns

## Overview

This document catalogs common anti-patterns, mistakes, and bad practices when working with the Magento_Customer module. Understanding these anti-patterns helps developers write upgrade-safe, performant, and secure customer-related code.

Each anti-pattern includes an explanation of why it's problematic, real-world consequences, and the correct approach with code examples.

**Target Versions:** Magento 2.4.7+ / Adobe Commerce 2.4.7+ with PHP 8.2+

---

## Table of Contents

1. [Direct Model Usage Instead of Service Contracts](#direct-model-usage-instead-of-service-contracts)
2. [Session Misuse in Service Contracts](#session-misuse-in-service-contracts)
3. [Password Handling Mistakes](#password-handling-mistakes)
4. [Bypassing Customer Validation](#bypassing-customer-validation)
5. [Improper Exception Handling](#improper-exception-handling)
6. [Collection Performance Anti-Patterns](#collection-performance-anti-patterns)
7. [Session Data Pollution](#session-data-pollution)
8. [Ignoring Customer Registry](#ignoring-customer-registry)
9. [Insecure PII Logging](#insecure-pii-logging)
10. [Email Enumeration Vulnerabilities](#email-enumeration-vulnerabilities)
11. [Hardcoded Customer Group IDs](#hardcoded-customer-group-ids)
12. [Improper Address Validation](#improper-address-validation)

---

## Direct Model Usage Instead of Service Contracts

### Anti-Pattern

Using `Magento\Customer\Model\Customer` and `Magento\Customer\Model\CustomerFactory` directly instead of service contracts.

### Why It's Wrong

- **Breaks API Stability:** Direct model access bypasses service contract guarantees
- **Skips Validation:** Service contracts enforce business rules and validation
- **Missing Events:** Direct model save doesn't dispatch all necessary events
- **Plugin Bypassing:** Plugins on repositories won't execute
- **Upgrade Risk:** Model implementations may change; service contracts won't

### Bad Example

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Customer\Model\CustomerFactory;
use Magento\Customer\Model\ResourceModel\Customer as CustomerResource;

class CustomerService
{
    public function __construct(
        private readonly CustomerFactory $customerFactory,
        private readonly CustomerResource $customerResource
    ) {}

    /**
     * ANTI-PATTERN: Direct model usage
     */
    public function createCustomer(array $data): void
    {
        $customer = $this->customerFactory->create();
        $customer->setEmail($data['email']);
        $customer->setFirstname($data['firstname']);
        $customer->setLastname($data['lastname']);
        $customer->setWebsiteId($data['website_id']);

        // PROBLEM: Bypasses service contract validation and events
        $this->customerResource->save($customer);
    }

    /**
     * ANTI-PATTERN: Direct model load
     */
    public function getCustomer(int $customerId)
    {
        $customer = $this->customerFactory->create();
        // PROBLEM: Bypasses CustomerRegistry cache
        $this->customerResource->load($customer, $customerId);
        return $customer;
    }
}
```

### Correct Approach

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Api\Data\CustomerInterfaceFactory;
use Magento\Framework\Encryption\EncryptorInterface;
use Magento\Framework\Exception\LocalizedException;

class CustomerService
{
    public function __construct(
        private readonly CustomerRepositoryInterface $customerRepository,
        private readonly CustomerInterfaceFactory $customerFactory,
        private readonly EncryptorInterface $encryptor
    ) {}

    /**
     * CORRECT: Use service contract
     */
    public function createCustomer(array $data, string $password): \Magento\Customer\Api\Data\CustomerInterface
    {
        $customer = $this->customerFactory->create();
        $customer->setEmail($data['email'])
            ->setFirstname($data['firstname'])
            ->setLastname($data['lastname'])
            ->setWebsiteId((int)$data['website_id']);

        // Triggers validation, events, and plugins
        return $this->customerRepository->save(
            $customer,
            $this->encryptor->getHash($password, true)
        );
    }

    /**
     * CORRECT: Use repository
     */
    public function getCustomer(int $customerId): \Magento\Customer\Api\Data\CustomerInterface
    {
        // Uses CustomerRegistry cache, triggers plugins
        return $this->customerRepository->getById($customerId);
    }
}
```

### Consequences of Anti-Pattern

- Skipped validation allows invalid data in database
- Missing events prevent integrations from working
- Performance degradation from bypassing registry cache
- Upgrade failures when model internals change

---

## Session Misuse in Service Contracts

### Anti-Pattern

Injecting and using `Magento\Customer\Model\Session` in service contracts, repositories, or API classes.

### Why It's Wrong

- **Breaks API Contracts:** REST/GraphQL APIs don't have sessions
- **Area Coupling:** Service contracts must work in all areas (cron, API, CLI)
- **Testability:** Session dependency makes unit testing difficult
- **State Management:** Creates hidden state dependencies

### Bad Example

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Customer\Model\Session;
use Magento\Customer\Api\CustomerRepositoryInterface;

class OrderService
{
    public function __construct(
        private readonly Session $customerSession,
        private readonly CustomerRepositoryInterface $customerRepository
    ) {}

    /**
     * ANTI-PATTERN: Session in service contract
     */
    public function createOrder(array $orderData): void
    {
        // PROBLEM: Breaks in API/cron contexts
        if (!$this->customerSession->isLoggedIn()) {
            throw new \Exception('Customer not logged in');
        }

        $customerId = $this->customerSession->getCustomerId();
        $customer = $this->customerRepository->getById($customerId);

        // Create order...
    }
}
```

### Correct Approach

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Framework\Exception\NoSuchEntityException;

class OrderService
{
    public function __construct(
        private readonly CustomerRepositoryInterface $customerRepository
    ) {}

    /**
     * CORRECT: Accept customer ID as parameter
     */
    public function createOrder(int $customerId, array $orderData): void
    {
        try {
            $customer = $this->customerRepository->getById($customerId);
        } catch (NoSuchEntityException) {
            throw new \InvalidArgumentException('Customer not found');
        }

        // Create order...
    }
}
```

**Controller Layer (where session is appropriate):**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Order;

use Magento\Customer\Model\Session;
use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\Controller\ResultFactory;
use Vendor\Module\Service\OrderService;

class Create implements HttpPostActionInterface
{
    public function __construct(
        private readonly Session $customerSession,
        private readonly OrderService $orderService,
        private readonly ResultFactory $resultFactory
    ) {}

    public function execute()
    {
        // Session usage appropriate in controller
        if (!$this->customerSession->isLoggedIn()) {
            return $this->resultFactory->create(ResultFactory::TYPE_REDIRECT)
                ->setPath('customer/account/login');
        }

        $customerId = (int)$this->customerSession->getCustomerId();
        $this->orderService->createOrder($customerId, $orderData);

        // Return response...
    }
}
```

---

## Password Handling Mistakes

### Anti-Pattern

Storing, logging, or transmitting plaintext passwords; using weak hashing; storing passwords in custom attributes.

### Why It's Wrong

- **Security Breach:** Plaintext passwords expose customer accounts
- **Compliance Violation:** Violates PCI DSS, GDPR, and other regulations
- **Credential Stuffing:** Attackers can use exposed passwords on other sites
- **Audit Failures:** Security audits will flag password mishandling

### Bad Examples

```php
<?php
// ANTI-PATTERN 1: Logging plaintext password
$logger->info('Customer registered', [
    'email' => $email,
    'password' => $password  // NEVER DO THIS
]);

// ANTI-PATTERN 2: Storing plaintext password
$customer->setData('temp_password', $password);  // NEVER DO THIS

// ANTI-PATTERN 3: Using weak hashing
$passwordHash = md5($password);  // NEVER DO THIS

// ANTI-PATTERN 4: Custom password storage
$customer->setCustomAttribute('backup_password', $password);  // NEVER DO THIS

// ANTI-PATTERN 5: Passing unhashed password to model
$customer->setPassword($password);  // Wrong - needs hash
$customer->save();
```

### Correct Approach

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Encryption\EncryptorInterface;

class CustomerPasswordService
{
    public function __construct(
        private readonly CustomerRepositoryInterface $customerRepository,
        private readonly EncryptorInterface $encryptor
    ) {}

    /**
     * CORRECT: Hash password before storage
     */
    public function createCustomerWithPassword(
        CustomerInterface $customer,
        string $password
    ): CustomerInterface {
        // Hash using Argon2id (PHP 8.2+) or SHA-256 with salt
        $passwordHash = $this->encryptor->getHash($password, true);

        // Repository handles secure storage
        return $this->customerRepository->save($customer, $passwordHash);
    }

    /**
     * CORRECT: Never log passwords
     */
    public function logCustomerCreation(CustomerInterface $customer): void
    {
        $this->logger->info('Customer created', [
            'customer_id' => $customer->getId(),
            'email' => $customer->getEmail()
            // Password never logged
        ]);
    }
}
```

**Password Change (Correct):**

```php
<?php
declare(strict_types=1);

use Magento\Customer\Api\AccountManagementInterface;

class PasswordChangeService
{
    public function __construct(
        private readonly AccountManagementInterface $accountManagement
    ) {}

    public function changePassword(
        string $email,
        string $currentPassword,
        string $newPassword
    ): bool {
        // Service contract handles hashing internally
        return $this->accountManagement->changePassword(
            $email,
            $currentPassword,
            $newPassword
        );
    }
}
```

---

## Bypassing Customer Validation

### Anti-Pattern

Skipping validation by directly saving models or using resource models.

### Bad Example

```php
<?php
// ANTI-PATTERN: Skip email validation
$customer->setEmail('invalid-email');
$customerResource->save($customer);  // Saves invalid email

// ANTI-PATTERN: Skip password requirements
$customer->setPasswordHash('weak');  // No strength validation
$customerResource->save($customer);
```

### Correct Approach

```php
<?php
declare(strict_types=1);

use Magento\Customer\Api\AccountManagementInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Exception\InputException;

class CustomerValidator
{
    public function __construct(
        private readonly AccountManagementInterface $accountManagement
    ) {}

    /**
     * CORRECT: Use service contract validation
     */
    public function validateAndCreate(
        CustomerInterface $customer,
        string $password
    ): CustomerInterface {
        // Validates email format, uniqueness, password strength
        $validationResults = $this->accountManagement->validate($customer);

        if (!$validationResults->isValid()) {
            $messages = [];
            foreach ($validationResults->getMessages() as $message) {
                $messages[] = $message->getMessage();
            }
            throw new InputException(__('Validation failed: %1', implode(', ', $messages)));
        }

        return $this->accountManagement->createAccount($customer, $password);
    }
}
```

---

## Improper Exception Handling

### Anti-Pattern

Catching all exceptions, swallowing exceptions, or not handling specific customer exceptions.

### Bad Example

```php
<?php
// ANTI-PATTERN 1: Catch all exceptions
try {
    $customer = $customerRepository->get($email);
} catch (\Exception $e) {
    // PROBLEM: Catches too broadly
    return null;
}

// ANTI-PATTERN 2: Swallow exceptions
try {
    $customerRepository->save($customer);
} catch (\Exception $e) {
    // PROBLEM: Silent failure
}

// ANTI-PATTERN 3: Generic error messages
try {
    $accountManagement->authenticate($email, $password);
} catch (\Exception $e) {
    throw new \Exception('Login failed');  // PROBLEM: Loses specific error
}
```

### Correct Approach

```php
<?php
declare(strict_types=1);

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Customer\Api\AccountManagementInterface;
use Magento\Framework\Exception\NoSuchEntityException;
use Magento\Framework\Exception\InvalidEmailOrPasswordException;
use Magento\Framework\Exception\State\UserLockedException;
use Psr\Log\LoggerInterface;

class CustomerAuthService
{
    public function __construct(
        private readonly CustomerRepositoryInterface $customerRepository,
        private readonly AccountManagementInterface $accountManagement,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * CORRECT: Handle specific exceptions
     */
    public function getCustomerByEmail(string $email, int $websiteId): ?\Magento\Customer\Api\Data\CustomerInterface
    {
        try {
            return $this->customerRepository->get($email, $websiteId);
        } catch (NoSuchEntityException) {
            // Expected exception for non-existent customer
            return null;
        } catch (\Exception $e) {
            // Unexpected exceptions still logged
            $this->logger->error('Error loading customer', [
                'email' => $email,
                'error' => $e->getMessage()
            ]);
            throw $e;
        }
    }

    /**
     * CORRECT: Differentiate authentication failures
     */
    public function authenticate(string $email, string $password): ?\Magento\Customer\Api\Data\CustomerInterface
    {
        try {
            return $this->accountManagement->authenticate($email, $password);
        } catch (InvalidEmailOrPasswordException) {
            $this->logger->info('Failed login attempt', ['email' => $email]);
            return null;
        } catch (UserLockedException $e) {
            $this->logger->warning('Account locked', ['email' => $email]);
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Account temporarily locked. Please try again later.')
            );
        }
    }
}
```

---

## Collection Performance Anti-Patterns

### Anti-Pattern

Loading entire customer collections, not using pagination, loading unnecessary attributes, post-collection filtering.

### Bad Examples

```php
<?php
// ANTI-PATTERN 1: Load all customers
$collection = $customerCollectionFactory->create();
$allCustomers = $collection->getItems();  // PROBLEM: Can load millions of records

// ANTI-PATTERN 2: Load unnecessary attributes
$collection->addAttributeToSelect('*');  // PROBLEM: Loads all EAV attributes

// ANTI-PATTERN 3: Post-collection filtering
$collection = $customerCollectionFactory->create();
$collection->load();
foreach ($collection as $customer) {
    if ($customer->getEmail() === $searchEmail) {  // PROBLEM: Filtering in PHP, not SQL
        return $customer;
    }
}

// ANTI-PATTERN 4: No pagination
$collection = $customerCollectionFactory->create();
$collection->addFieldToFilter('group_id', 2);
// PROBLEM: No setPageSize() - loads all matching records
```

### Correct Approach

```php
<?php
declare(strict_types=1);

use Magento\Customer\Api\CustomerRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Api\SortOrderBuilder;
use Magento\Customer\Model\ResourceModel\Customer\CollectionFactory;

class CustomerSearchService
{
    public function __construct(
        private readonly CustomerRepositoryInterface $customerRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder,
        private readonly SortOrderBuilder $sortOrderBuilder,
        private readonly CollectionFactory $customerCollectionFactory
    ) {}

    /**
     * CORRECT: Use repository with pagination
     */
    public function findCustomersByGroup(int $groupId, int $pageSize = 20, int $page = 1): array
    {
        $sortOrder = $this->sortOrderBuilder
            ->setField('created_at')
            ->setDirection('DESC')
            ->create();

        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('group_id', $groupId)
            ->setPageSize($pageSize)
            ->setCurrentPage($page)
            ->addSortOrder($sortOrder)
            ->create();

        $searchResults = $this->customerRepository->getList($searchCriteria);

        return [
            'items' => $searchResults->getItems(),
            'total_count' => $searchResults->getTotalCount()
        ];
    }

    /**
     * CORRECT: Select only needed attributes
     */
    public function getCustomerNamesAndEmails(): array
    {
        $collection = $this->customerCollectionFactory->create();
        $collection->addAttributeToSelect(['firstname', 'lastname', 'email'])
            ->setPageSize(100)
            ->setCurPage(1);

        $results = [];
        foreach ($collection as $customer) {
            $results[] = [
                'name' => $customer->getName(),
                'email' => $customer->getEmail()
            ];
        }

        return $results;
    }

    /**
     * CORRECT: Use SQL filtering
     */
    public function findByEmail(string $email): ?\Magento\Customer\Model\Customer
    {
        $collection = $this->customerCollectionFactory->create();
        $collection->addFieldToFilter('email', $email)
            ->setPageSize(1);

        return $collection->getFirstItem()->getId() ? $collection->getFirstItem() : null;
    }
}
```

---

## Session Data Pollution

### Anti-Pattern

Storing large objects, sensitive data, or temporary data in customer session.

### Bad Examples

```php
<?php
// ANTI-PATTERN 1: Store large objects
$customerSession->setData('product_catalog', $largeArray);  // PROBLEM: Session bloat

// ANTI-PATTERN 2: Store sensitive data
$customerSession->setData('credit_card_number', $ccNumber);  // PROBLEM: Security risk

// ANTI-PATTERN 3: Use as cache
$customerSession->setData('api_response_cache', $apiData);  // PROBLEM: Wrong tool

// ANTI-PATTERN 4: Never clean up
$customerSession->setData('temp_checkout_data', $data);
// PROBLEM: No cleanup after use
```

### Correct Approach

```php
<?php
declare(strict_types=1);

use Magento\Customer\Model\Session;
use Magento\Framework\Session\SessionManagerInterface;

class SessionManagementService
{
    public function __construct(
        private readonly Session $customerSession,
        private readonly SessionManagerInterface $sessionManager
    ) {}

    /**
     * CORRECT: Store only essential data
     */
    public function storeCheckoutStep(string $step): void
    {
        // Store minimal data
        $this->customerSession->setData('checkout_step', $step);
    }

    /**
     * CORRECT: Clean up after use
     */
    public function clearCheckoutData(): void
    {
        $this->customerSession->unsData('checkout_step');
        $this->customerSession->unsData('shipping_method');
        $this->customerSession->unsData('payment_method');
    }

    /**
     * CORRECT: Never store sensitive data in session
     * Use secure payment vault instead
     */
    public function storePaymentMethod(string $methodCode): void
    {
        // Only store method code, not card details
        $this->customerSession->setData('payment_method_code', $methodCode);
    }
}
```

---

## Ignoring Customer Registry

### Anti-Pattern

Loading the same customer multiple times in a request instead of using `CustomerRegistry`.

### Bad Example

```php
<?php
// ANTI-PATTERN: Multiple loads
$customer1 = $customerFactory->create()->load($customerId);  // DB query
$customer2 = $customerFactory->create()->load($customerId);  // Redundant DB query
$customer3 = $customerFactory->create()->load($customerId);  // Another redundant query
```

### Correct Approach

```php
<?php
declare(strict_types=1);

use Magento\Customer\Model\CustomerRegistry;
use Magento\Customer\Api\CustomerRepositoryInterface;

class CustomerDataService
{
    public function __construct(
        private readonly CustomerRegistry $customerRegistry,
        private readonly CustomerRepositoryInterface $customerRepository
    ) {}

    /**
     * CORRECT: Registry caches customer
     */
    public function processCustomer(int $customerId): void
    {
        // First call: loads from DB and caches
        $customer1 = $this->customerRegistry->retrieve($customerId);

        // Subsequent calls: returns cached instance
        $customer2 = $this->customerRegistry->retrieve($customerId);
        $customer3 = $this->customerRegistry->retrieve($customerId);

        // All three are same instance, only one DB query
    }

    /**
     * CORRECT: Repository uses registry internally
     */
    public function getCustomerData(int $customerId): array
    {
        // Repository uses registry cache
        $customer = $this->customerRepository->getById($customerId);
        return [
            'name' => $customer->getFirstname() . ' ' . $customer->getLastname(),
            'email' => $customer->getEmail()
        ];
    }
}
```

---

## Insecure PII Logging

### Anti-Pattern

Logging personally identifiable information (PII) in plaintext logs.

### Bad Examples

```php
<?php
// ANTI-PATTERN: Log customer PII
$logger->info('Customer updated', [
    'customer_id' => $customer->getId(),
    'email' => $customer->getEmail(),  // PII
    'name' => $customer->getName(),    // PII
    'dob' => $customer->getDob(),      // PII
    'address' => $customer->getDefaultBillingAddress()  // PII
]);

// ANTI-PATTERN: Debug output with PII
var_dump($customer->getData());  // Contains all PII
```

### Correct Approach

```php
<?php
declare(strict_types=1);

use Psr\Log\LoggerInterface;
use Magento\Customer\Api\Data\CustomerInterface;

class SecureCustomerLogger
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {}

    /**
     * CORRECT: Log without PII
     */
    public function logCustomerUpdate(CustomerInterface $customer): void
    {
        $this->logger->info('Customer updated', [
            'customer_id' => $customer->getId(),
            'website_id' => $customer->getWebsiteId(),
            'group_id' => $customer->getGroupId()
            // No email, name, DOB, or address
        ]);
    }

    /**
     * CORRECT: Hash PII if logging necessary
     */
    public function logAuthenticationAttempt(string $email): void
    {
        $this->logger->info('Authentication attempt', [
            'email_hash' => hash('sha256', strtolower($email))
        ]);
    }

    /**
     * CORRECT: Scrub sensitive data from exceptions
     */
    public function logException(\Exception $e, array $context = []): void
    {
        $sanitizedContext = $this->scrubPii($context);
        $this->logger->error($e->getMessage(), $sanitizedContext);
    }

    private function scrubPii(array $data): array
    {
        $piiFields = ['email', 'firstname', 'lastname', 'dob', 'taxvat', 'telephone'];

        foreach ($piiFields as $field) {
            if (isset($data[$field])) {
                $data[$field] = '[REDACTED]';
            }
        }

        return $data;
    }
}
```

---

## Email Enumeration Vulnerabilities

### Anti-Pattern

Revealing whether an email exists in the system through different error messages or response times.

### Bad Example

```php
<?php
// ANTI-PATTERN: Reveals if email exists
public function resetPassword(string $email): array
{
    try {
        $customer = $customerRepository->get($email);
    } catch (NoSuchEntityException) {
        return ['success' => false, 'message' => 'Email not found'];  // PROBLEM: Enumeration
    }

    // Send reset email
    return ['success' => true, 'message' => 'Reset email sent'];
}
```

### Correct Approach

```php
<?php
declare(strict_types=1);

use Magento\Customer\Api\AccountManagementInterface;
use Magento\Framework\Exception\NoSuchEntityException;
use Psr\Log\LoggerInterface;

class PasswordResetService
{
    public function __construct(
        private readonly AccountManagementInterface $accountManagement,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * CORRECT: Same response regardless of email existence
     */
    public function initiatePasswordReset(string $email, int $websiteId): array
    {
        try {
            $this->accountManagement->initiatePasswordReset(
                $email,
                AccountManagementInterface::EMAIL_RESET,
                $websiteId
            );
        } catch (NoSuchEntityException) {
            // Log but don't reveal
            $this->logger->info('Password reset for non-existent email', [
                'email_hash' => hash('sha256', strtolower($email))
            ]);
        }

        // Always return same message (prevents enumeration)
        return [
            'success' => true,
            'message' => 'If the email exists, you will receive a password reset link.'
        ];
    }
}
```

---

## Hardcoded Customer Group IDs

### Anti-Pattern

Using hardcoded customer group IDs instead of configuration or database lookups.

### Bad Example

```php
<?php
// ANTI-PATTERN: Hardcoded group IDs
if ($customer->getGroupId() === 2) {  // PROBLEM: What if group 2 doesn't exist?
    $discount = 0.10;
}

// ANTI-PATTERN: Hardcoded group assignment
$customer->setGroupId(3);  // PROBLEM: Group 3 might not be "Wholesale"
```

### Correct Approach

```php
<?php
declare(strict_types=1);

use Magento\Customer\Api\GroupRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;

class CustomerGroupService
{
    public function __construct(
        private readonly GroupRepositoryInterface $groupRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder
    ) {}

    /**
     * CORRECT: Look up group by code
     */
    public function getGroupIdByCode(string $groupCode): ?int
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('customer_group_code', $groupCode)
            ->setPageSize(1)
            ->create();

        $groups = $this->groupRepository->getList($searchCriteria)->getItems();

        if (empty($groups)) {
            return null;
        }

        $group = reset($groups);
        return (int)$group->getId();
    }

    /**
     * CORRECT: Use configuration for discount logic
     */
    public function getDiscountForCustomer(int $customerId): float
    {
        $customer = $this->customerRepository->getById($customerId);
        $groupId = (int)$customer->getGroupId();

        // Use catalog price rules or configuration instead of hardcoding
        return $this->scopeConfig->getValue(
            'customer_discounts/group_' . $groupId . '/discount_percent',
            ScopeInterface::SCOPE_STORE
        ) ?? 0.0;
    }
}
```

---

## Improper Address Validation

### Anti-Pattern

Not validating address data before save, allowing invalid country/region combinations.

### Bad Example

```php
<?php
// ANTI-PATTERN: No validation
$address->setCountryId('US');
$address->setRegionId(999);  // PROBLEM: Invalid region for US
$addressRepository->save($address);

// ANTI-PATTERN: No street line validation
$address->setStreet(['Line 1', 'Line 2', 'Line 3', 'Line 4', 'Line 5']);  // PROBLEM: Too many lines
```

### Correct Approach

```php
<?php
declare(strict_types=1);

use Magento\Customer\Api\AddressRepositoryInterface;
use Magento\Customer\Api\Data\AddressInterface;
use Magento\Directory\Model\ResourceModel\Region\CollectionFactory as RegionCollectionFactory;
use Magento\Framework\Exception\LocalizedException;

class AddressValidationService
{
    public function __construct(
        private readonly AddressRepositoryInterface $addressRepository,
        private readonly RegionCollectionFactory $regionCollectionFactory
    ) {}

    /**
     * CORRECT: Validate before save
     */
    public function saveAddress(AddressInterface $address): AddressInterface
    {
        $this->validateAddress($address);
        return $this->addressRepository->save($address);
    }

    private function validateAddress(AddressInterface $address): void
    {
        // Validate required fields
        if (!$address->getCountryId()) {
            throw new LocalizedException(__('Country is required'));
        }

        // Validate region for country
        if ($address->getRegionId()) {
            $this->validateRegion(
                (int)$address->getRegionId(),
                $address->getCountryId()
            );
        }

        // Validate street lines
        $street = $address->getStreet();
        if (is_array($street) && count($street) > 4) {
            throw new LocalizedException(
                __('Address can have maximum 4 street lines')
            );
        }
    }

    private function validateRegion(int $regionId, string $countryId): void
    {
        $regionCollection = $this->regionCollectionFactory->create();
        $regionCollection->addRegionIdFilter($regionId)
            ->addCountryFilter($countryId);

        if ($regionCollection->getSize() === 0) {
            throw new LocalizedException(
                __('Invalid region for selected country')
            );
        }
    }
}
```

---

## Summary of Anti-Patterns

| Anti-Pattern | Impact | Correct Approach |
|--------------|--------|------------------|
| Direct model usage | Bypasses validation, events, plugins | Use service contracts |
| Session in service contracts | Breaks API contexts | Pass customer ID as parameter |
| Plaintext passwords | Security breach | Use EncryptorInterface |
| Skip validation | Data integrity issues | Use AccountManagementInterface |
| Generic exception handling | Lost error context | Catch specific exceptions |
| Load entire collections | Performance degradation | Use pagination and filtering |
| Session data pollution | Memory bloat | Store minimal data, clean up |
| Ignore registry | Redundant DB queries | Use CustomerRegistry |
| Log PII | Privacy violations | Hash or redact PII |
| Email enumeration | Security vulnerability | Same response for all emails |
| Hardcoded group IDs | Fragile code | Look up by code |
| No address validation | Invalid data | Validate before save |

---

## Assumptions

- **Target Platform:** Adobe Commerce / Magento Open Source 2.4.7+
- **PHP Version:** 8.2+
- **Security Standards:** PCI DSS, GDPR compliance required
- **Performance Requirements:** Production-level optimization

## Why This Approach

Following these correct patterns ensures upgrade safety, security, performance, and compliance. Service contracts provide API stability, proper validation prevents data corruption, and secure practices protect customer data.

## Security Impact

- **Password Security:** Never store plaintext; use Argon2id hashing
- **PII Protection:** Redact sensitive data from logs and debug output
- **Email Enumeration:** Prevent attackers from discovering valid accounts
- **Session Security:** Minimize session data; regenerate after authentication

## Performance Impact

- **Repository Pattern:** Use registry caching to avoid redundant queries
- **Collection Optimization:** Paginate, filter in SQL, load only needed attributes
- **Session Minimization:** Reduce session storage size for faster serialization

## Backward Compatibility

- **Service Contracts:** Stable across minor versions
- **Deprecation:** Direct model usage deprecated since 2.3
- **Migration Path:** Gradually refactor to repositories

## Tests to Add

- **Unit:** Test validation logic, exception handling
- **Integration:** Test service contracts with various inputs
- **Security:** Test password hashing, PII logging prevention

## Docs to Update

- **ANTI_PATTERNS.md:** This document
- **README.md:** Link to anti-patterns
- **Code review checklist:** Include anti-pattern checks
