---
title: "Magento_Customer Execution Flows"
module: "Magento_Customer"
doc_type: "execution-flows"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Customer Execution Flows

## Overview

This document provides detailed execution flows for critical customer operations in the Magento_Customer module. Understanding these flows is essential for debugging customer-related issues, implementing custom authentication mechanisms, extending registration processes, and integrating third-party systems.

Each flow includes sequence diagrams in text format, code examples, event hooks, database interactions, and extension points for customization.

**Target Versions:** Magento 2.4.7+ / Adobe Commerce 2.4.7+ with PHP 8.2+

---

## Table of Contents

1. [Customer Registration Flow](#customer-registration-flow)
2. [Customer Login Flow](#customer-login-flow)
3. [Password Reset Flow](#password-reset-flow)
4. [Customer Account Update Flow](#customer-account-update-flow)
5. [Address Management Flow](#address-management-flow)
6. [Email Verification Flow](#email-verification-flow)
7. [Customer Session Management Flow](#customer-session-management-flow)
8. [Customer Group Assignment Flow](#customer-group-assignment-flow)
9. [VAT Validation Flow](#vat-validation-flow)
10. [Customer Deletion Flow](#customer-deletion-flow)

---

## Customer Registration Flow

### Flow Overview

Customer registration can occur through three primary paths:
1. **Frontend registration** - Customer self-registration via `/customer/account/create`
2. **Admin creation** - Admin user creates customer via backend
3. **API registration** - Third-party integration via REST/GraphQL

### Frontend Registration Sequence

```
[Customer Browser]
       |
       | POST /customer/account/createPost
       v
[Controller: Magento\Customer\Controller\Account\CreatePost]
       |
       | 1. Validate form key (CSRF protection)
       | 2. Extract customer data from request
       | 3. Validate required fields
       v
[AccountManagementInterface::createAccount()]
       |
       | 4. Check if email already exists
       | 5. Validate customer data (email format, password strength)
       v
[CustomerRepositoryInterface::save()]
       |
       | 6. Dispatch event: customer_save_before
       | 7. Hash password (Argon2id/SHA-256)
       | 8. Set default customer group
       | 9. Generate confirmation key (if required)
       v
[Database: INSERT customer_entity]
       |
       | 10. Save EAV attributes to customer_entity_*
       | 11. Update customer_grid_flat via indexer
       v
[Event: customer_register_success]
       |
       | 12. Send welcome email (if configured)
       | 13. Send confirmation email (if required)
       v
[Response: Redirect to success page or dashboard]
```

### Code Implementation

**Frontend Registration Controller**

```php
<?php
declare(strict_types=1);

namespace Magento\Customer\Controller\Account;

use Magento\Customer\Api\AccountManagementInterface;
use Magento\Customer\Api\Data\CustomerInterfaceFactory;
use Magento\Customer\Model\Session;
use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\App\RequestInterface;
use Magento\Framework\Controller\Result\Redirect;
use Magento\Framework\Controller\ResultFactory;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Message\ManagerInterface;
use Magento\Store\Model\StoreManagerInterface;

class CreatePost implements HttpPostActionInterface
{
    public function __construct(
        private readonly RequestInterface $request,
        private readonly AccountManagementInterface $accountManagement,
        private readonly CustomerInterfaceFactory $customerFactory,
        private readonly Session $customerSession,
        private readonly StoreManagerInterface $storeManager,
        private readonly ManagerInterface $messageManager,
        private readonly ResultFactory $resultFactory
    ) {}

    public function execute(): Redirect
    {
        $resultRedirect = $this->resultFactory->create(ResultFactory::TYPE_REDIRECT);

        // Validate form key (CSRF protection)
        if (!$this->request->isPost()) {
            $resultRedirect->setPath('*/*/create');
            return $resultRedirect;
        }

        try {
            // Extract customer data
            $customerData = $this->extractCustomerData();
            $password = $this->request->getParam('password');
            $passwordConfirmation = $this->request->getParam('password_confirmation');

            // Validate password match
            if ($password !== $passwordConfirmation) {
                throw new LocalizedException(__('Passwords do not match.'));
            }

            // Create customer
            $customer = $this->customerFactory->create();
            $customer->setEmail($customerData['email'])
                ->setFirstname($customerData['firstname'])
                ->setLastname($customerData['lastname'])
                ->setWebsiteId((int)$this->storeManager->getWebsite()->getId())
                ->setStoreId((int)$this->storeManager->getStore()->getId());

            // Save customer (triggers email confirmation if configured)
            $savedCustomer = $this->accountManagement->createAccount($customer, $password);

            // Auto-login if email confirmation not required
            if (!$savedCustomer->getConfirmation()) {
                $this->customerSession->setCustomerDataAsLoggedIn($savedCustomer);
                $this->customerSession->regenerateId();
            }

            $this->messageManager->addSuccessMessage(
                __('Thank you for registering with %1.', $this->storeManager->getStore()->getName())
            );

            $resultRedirect->setPath('customer/account/index');
            return $resultRedirect;

        } catch (LocalizedException $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
        } catch (\Exception $e) {
            $this->messageManager->addExceptionMessage(
                $e,
                __('We cannot create your account right now.')
            );
        }

        $resultRedirect->setPath('*/*/create');
        return $resultRedirect;
    }

    private function extractCustomerData(): array
    {
        return [
            'email' => $this->request->getParam('email'),
            'firstname' => $this->request->getParam('firstname'),
            'lastname' => $this->request->getParam('lastname'),
            'dob' => $this->request->getParam('dob'),
            'taxvat' => $this->request->getParam('taxvat'),
            'gender' => $this->request->getParam('gender'),
        ];
    }
}
```

**Custom Registration Observer**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Psr\Log\LoggerInterface;

class CustomerRegisterSuccessObserver implements ObserverInterface
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Execute after successful customer registration
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var CustomerInterface $customer */
        $customer = $observer->getEvent()->getCustomer();

        $this->logger->info('New customer registered', [
            'customer_id' => $customer->getId(),
            'email' => $customer->getEmail(),
            'website_id' => $customer->getWebsiteId(),
            'group_id' => $customer->getGroupId()
        ]);

        // Custom logic: assign to special group, send to CRM, etc.
    }
}
```

```xml
<!-- etc/frontend/events.xml -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="customer_register_success">
        <observer name="vendor_module_customer_register_success"
                  instance="Vendor\Module\Observer\CustomerRegisterSuccessObserver" />
    </event>
</config>
```

### Key Events in Registration Flow

| Event | Trigger Point | Use Case |
|-------|---------------|----------|
| `customer_save_before` | Before customer entity saved | Validate custom data, modify attributes |
| `customer_save_after` | After customer entity saved | Update external systems, log changes |
| `customer_register_success` | After successful registration | Send to CRM, assign rewards, analytics |

### Database Changes

1. **INSERT into `customer_entity`**
   - Generates `entity_id` (auto-increment)
   - Stores `password_hash` (Argon2id with salt)
   - Sets `created_at` / `updated_at` timestamps
   - Stores `confirmation` token if email verification required

2. **INSERT into `customer_entity_*` (EAV tables)**
   - Custom attributes stored in appropriate type tables
   - Example: `dob` → `customer_entity_datetime`

3. **INSERT into `customer_grid_flat`**
   - Indexer populates denormalized grid table
   - Triggered via message queue or cron

### Configuration Dependencies

- **Email Confirmation Required:** `customer/create_account/confirm` (0=No, 1=Yes)
- **Default Customer Group:** `customer/create_account/default_group` (default: 1 = General)
- **Email Templates:** `customer/create_account/email_template` and `customer/create_account/email_confirmation_template`
- **Password Requirements:** `customer/password/*` (minimum length, character classes)

---

## Customer Login Flow

### Flow Overview

Customer authentication validates credentials and establishes a session. The flow includes brute-force protection, account lockout, and optional two-factor authentication integration.

### Login Sequence

```
[Customer Browser]
       |
       | POST /customer/account/loginPost
       v
[Controller: Magento\Customer\Controller\Account\LoginPost]
       |
       | 1. Validate form key (CSRF)
       | 2. Extract username (email) and password
       v
[AccountManagementInterface::authenticate()]
       |
       | 3. Load customer by email and website ID
       | 4. Check account status (is_active, confirmation, lock_expires)
       v
[Customer::authenticate()]
       |
       | 5. Verify password hash (Argon2/SHA-256)
       | 6. Check failures_num for brute-force protection
       v
[SUCCESS PATH]
       |
       | 7. Reset failures_num to 0
       | 8. Clear lock_expires
       v
[Session::setCustomerDataAsLoggedIn()]
       |
       | 9. Load customer data into session
       | 10. Regenerate session ID (security)
       v
[Event: customer_login]
       |
       | 11. Update last login timestamp
       | 12. Merge guest cart with customer cart
       v
[Response: Redirect to account dashboard or referer]
```

**Failure Path:**

```
[FAILURE PATH]
       |
       | 7. Increment failures_num
       | 8. Set first_failure timestamp (if first attempt)
       | 9. Check if failures_num >= threshold (default: 10)
       v
[IF THRESHOLD EXCEEDED]
       |
       | 10. Set lock_expires = now + lockout_duration (default: 10 minutes)
       | 11. Throw LocalizedException with lockout message
       v
[Response: Error message, redirect to login page]
```

### Code Implementation

**Authentication Service**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\AccountManagementInterface;
use Magento\Customer\Api\Data\CustomerInterface;
use Magento\Customer\Model\Session;
use Magento\Framework\Exception\InvalidEmailOrPasswordException;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Exception\State\UserLockedException;
use Psr\Log\LoggerInterface;

class CustomerAuthenticationService
{
    public function __construct(
        private readonly AccountManagementInterface $accountManagement,
        private readonly Session $customerSession,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Authenticate customer and start session
     *
     * @param string $email
     * @param string $password
     * @return bool
     */
    public function login(string $email, string $password): bool
    {
        try {
            $customer = $this->accountManagement->authenticate($email, $password);
            $this->customerSession->setCustomerDataAsLoggedIn($customer);
            $this->customerSession->regenerateId();

            $this->logger->info('Customer logged in', [
                'customer_id' => $customer->getId(),
                'email' => $customer->getEmail()
            ]);

            return true;

        } catch (InvalidEmailOrPasswordException $e) {
            $this->logger->warning('Failed login attempt', [
                'email' => $email,
                'reason' => 'invalid_credentials'
            ]);
            return false;

        } catch (UserLockedException $e) {
            $this->logger->warning('Account locked', [
                'email' => $email,
                'reason' => 'brute_force_protection'
            ]);
            throw new LocalizedException(
                __('Your account is temporarily locked. Please try again later.')
            );

        } catch (\Exception $e) {
            $this->logger->error('Login error', [
                'email' => $email,
                'error' => $e->getMessage()
            ]);
            return false;
        }
    }
}
```

**Login Observer for Custom Logic**

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

        // Set custom attribute for last login time
        $customer->setData('last_login_at', $this->dateTime->gmtDate());
        $customer->getResource()->saveAttribute($customer, 'last_login_at');
    }
}
```

```xml
<!-- etc/frontend/events.xml -->
<event name="customer_login">
    <observer name="vendor_module_customer_login"
              instance="Vendor\Module\Observer\CustomerLoginObserver" />
</event>
```

### Key Events in Login Flow

| Event | Trigger Point | Use Case |
|-------|---------------|----------|
| `customer_login` | `Session::setCustomerAsLoggedIn()` (not AccountManagement) | Update login timestamp, sync with CRM |
| `customer_data_object_login` | `Session::setCustomerDataAsLoggedIn()` | Modify session data, add custom attributes |

### Database Changes

1. **UPDATE `customer_entity`**
   - Reset `failures_num = 0` on success
   - Clear `lock_expires = NULL` on success
   - Increment `failures_num` on failure
   - Set `first_failure` timestamp on first failure
   - Set `lock_expires` when threshold exceeded

2. **Session Storage** (Redis/Database)
   - Store customer_id, customer_group_id, customer_name
   - Session data encrypted and signed

### Brute-Force Protection Configuration

- **Max Failures:** `customer/password/lockout_failures` (default: 10)
- **Lockout Duration:** `customer/password/lockout_threshold` (default: 10 minutes)

---

## Password Reset Flow

### Flow Overview

Password reset uses a time-limited token sent via email to verify customer identity.

### Password Reset Sequence

```
[Customer Browser]
       |
       | POST /customer/account/forgotpasswordpost
       v
[AccountManagementInterface::initiatePasswordReset()]
       |
       | 1. Load customer by email
       | 2. Generate random token (64 chars)
       | 3. Set rp_token and rp_token_created_at
       v
[Database: UPDATE customer_entity]
       |
       | 4. Save token and timestamp
       v
[Email: Send password reset link]
       |
       | Link: /customer/account/createPassword?token=XXX
       v
[Customer clicks link]
       |
       | GET /customer/account/createPassword
       v
[Controller: Validate token]
       |
       | 5. Check token exists
       | 6. Check token not expired (2 hours default)
       v
[Customer submits new password]
       |
       | POST /customer/account/resetPasswordPost
       v
[AccountManagementInterface::resetPassword()]
       |
       | 7. Validate token again
       | 8. Hash new password
       | 9. Clear rp_token and rp_token_created_at
       | 10. Update password_hash
       v
[Database: UPDATE customer_entity]
       |
       | 11. Save new password
       |
       | 12. Send confirmation email
       v
[Response: Redirect to login page]
```

### Code Implementation

**Password Reset Service**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\AccountManagementInterface;
use Magento\Framework\Exception\LocalizedException;
use Magento\Store\Model\StoreManagerInterface;
use Psr\Log\LoggerInterface;

class PasswordResetService
{
    public function __construct(
        private readonly AccountManagementInterface $accountManagement,
        private readonly StoreManagerInterface $storeManager,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Initiate password reset for customer
     *
     * @param string $email
     * @return bool
     */
    public function initiateReset(string $email): bool
    {
        try {
            $websiteId = (int)$this->storeManager->getWebsite()->getId();
            $this->accountManagement->initiatePasswordReset(
                $email,
                AccountManagementInterface::EMAIL_RESET,
                $websiteId
            );

            $this->logger->info('Password reset initiated', [
                'email' => $email,
                'website_id' => $websiteId
            ]);

            return true;

        } catch (LocalizedException $e) {
            // Don't reveal if email exists (security)
            $this->logger->warning('Password reset request', [
                'email' => $email,
                'reason' => $e->getMessage()
            ]);
            return true; // Always return true to prevent email enumeration

        } catch (\Exception $e) {
            $this->logger->error('Password reset error', [
                'email' => $email,
                'error' => $e->getMessage()
            ]);
            return false;
        }
    }

    /**
     * Reset password with token
     *
     * @param string $email
     * @param string $resetToken
     * @param string $newPassword
     * @return bool
     */
    public function resetPassword(string $email, string $resetToken, string $newPassword): bool
    {
        try {
            $this->accountManagement->resetPassword($email, $resetToken, $newPassword);

            $this->logger->info('Password reset completed', [
                'email' => $email
            ]);

            return true;

        } catch (LocalizedException $e) {
            $this->logger->warning('Password reset failed', [
                'email' => $email,
                'reason' => $e->getMessage()
            ]);
            throw $e;
        }
    }
}
```

### Key Configuration

- **Token Expiration:** `customer/password/reset_link_expiration_period` (default: 2 hours)
- **Email Template:** `customer/password/forgot_email_template`
- **Reset Protection:** `customer/password/limit_password_reset_requests_method` (controls how password reset requests are limited)

---

## Address Management Flow

### Adding a New Address

```
[Customer Account Page]
       |
       | POST /customer/address/formPost
       v
[Controller: Validate form data]
       |
       | 1. Validate required fields (street, city, postcode, country)
       | 2. Validate region/state matches country
       | 3. Check address limit (if configured)
       v
[AddressRepositoryInterface::save()]
       |
       | 4. Create AddressInterface DTO
       | 5. Set customer_id
       | 6. Validate VAT ID (if provided and EU country)
       v
[Event: customer_address_save_before]
       |
       | 7. Custom validation or modification
       v
[Database: INSERT customer_address_entity]
       |
       | 8. Save EAV attributes
       v
[Event: customer_address_save_after]
       |
       | 9. Update external systems
       v
[Update default billing/shipping if selected]
       |
       | 10. UPDATE customer_entity.default_billing
       | 11. UPDATE customer_entity.default_shipping
       v
[Response: Redirect to address book]
```

### Code Implementation

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Api\AddressRepositoryInterface;
use Magento\Customer\Api\Data\AddressInterface;
use Magento\Customer\Api\Data\AddressInterfaceFactory;
use Magento\Customer\Api\Data\RegionInterfaceFactory;
use Magento\Framework\Exception\LocalizedException;

class AddressManagementService
{
    public function __construct(
        private readonly AddressRepositoryInterface $addressRepository,
        private readonly AddressInterfaceFactory $addressFactory,
        private readonly RegionInterfaceFactory $regionFactory
    ) {}

    /**
     * Create customer address
     *
     * @param int $customerId
     * @param array $addressData
     * @return AddressInterface
     * @throws LocalizedException
     */
    public function createAddress(int $customerId, array $addressData): AddressInterface
    {
        $address = $this->addressFactory->create();
        $address->setCustomerId($customerId)
            ->setFirstname($addressData['firstname'])
            ->setLastname($addressData['lastname'])
            ->setStreet($addressData['street']) // Array of street lines
            ->setCity($addressData['city'])
            ->setCountryId($addressData['country_id'])
            ->setPostcode($addressData['postcode'])
            ->setTelephone($addressData['telephone']);

        // Set region
        if (!empty($addressData['region_id'])) {
            $region = $this->regionFactory->create();
            $region->setRegionId((int)$addressData['region_id']);
            $address->setRegion($region);
        } elseif (!empty($addressData['region'])) {
            $region = $this->regionFactory->create();
            $region->setRegion($addressData['region']);
            $address->setRegion($region);
        }

        // Optional fields
        if (!empty($addressData['company'])) {
            $address->setCompany($addressData['company']);
        }

        if (!empty($addressData['vat_id'])) {
            $address->setVatId($addressData['vat_id']);
        }

        // Default flags
        $address->setIsDefaultBilling($addressData['default_billing'] ?? false);
        $address->setIsDefaultShipping($addressData['default_shipping'] ?? false);

        return $this->addressRepository->save($address);
    }
}
```

---

## Customer Session Management Flow

### Session Initialization

```
[HTTP Request]
       |
       | Frontend area detected
       v
[Session::start()]
       |
       | 1. Load session from storage (Redis/DB/Files)
       | 2. Validate session cookie
       v
[IF customer_id in session]
       |
       | 3. Load customer data from CustomerRegistry
       | 4. Validate customer is_active = 1
       | 5. Check confirmation status
       v
[Session data populated]
       |
       | - customer_id
       | - customer_group_id
       | - customer_name
       | - customer_email
       v
[Request continues with authenticated customer]
```

### Session Security

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

use Magento\Customer\Model\Session;

class SessionSecurityService
{
    public function __construct(
        private readonly Session $customerSession
    ) {}

    /**
     * Regenerate session ID after authentication
     * Prevents session fixation attacks
     */
    public function regenerateSession(): void
    {
        if ($this->customerSession->isLoggedIn()) {
            $this->customerSession->regenerateId();
        }
    }

    /**
     * Clear sensitive session data
     */
    public function clearSensitiveData(): void
    {
        $this->customerSession->unsData('credit_card_last4');
        $this->customerSession->unsData('temporary_password');
    }
}
```

---

## Assumptions

- **Target Platform:** Adobe Commerce / Magento Open Source 2.4.7+
- **PHP Version:** 8.2+
- **Session Storage:** Redis recommended for production
- **Email Delivery:** Configured SMTP or email service (SendGrid, AWS SES)

## Why This Approach

These execution flows follow Magento's event-driven architecture, allowing customization at multiple extension points without modifying core code. Service contracts ensure API stability, while observers and plugins provide safe customization hooks.

## Security Impact

- **CSRF:** All POST requests validate form keys
- **Session Fixation:** `regenerateId()` called after authentication
- **Brute-Force:** Account lockout after configurable failed attempts
- **Token Expiration:** Password reset tokens expire after 2 hours (configurable)
- **Email Enumeration:** Password reset always returns success (don't reveal if email exists)

## Performance Impact

- **Session Storage:** Use Redis for fast session access and horizontal scaling
- **Customer Registry:** In-memory cache prevents redundant database queries
- **Event Observers:** Keep observer logic lightweight; use message queues for heavy operations

## Backward Compatibility

- **Service Contracts:** Stable across 2.4.x versions
- **Events:** New events added, existing events never removed
- **Database Schema:** Additive changes only (new columns with defaults)

## Tests to Add

- **Unit:** Authentication logic, password validation, token generation
- **Integration:** Full registration flow, login with brute-force protection, address CRUD
- **MFTF:** End-to-end registration, login, password reset, address management

## Docs to Update

- **README.md:** Overview of module capabilities
- **EXECUTION_FLOWS.md:** This document
- **KNOWN_ISSUES.md:** Document session fixation mitigations, brute-force behavior
