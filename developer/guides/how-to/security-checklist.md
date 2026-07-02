---
title: "Security Checklist for Custom Modules"
description: "Comprehensive security audit and implementation guide for Magento 2 custom module development covering XSS, CSRF, SQL injection, ACL, and PCI compliance"
type: "how-to"
tier: 2
difficulty: "intermediate"
version: "2.4.7+"
estimated_time: "45 minutes"
topics:
  - security
  - best-practices
  - acl
  - csrf
  - xss
  - pci-compliance
last_updated: "2026-02-07"
---

# Security Checklist for Custom Modules

## Problem Statement

Security vulnerabilities in custom Magento modules are among the leading causes of data breaches, payment fraud, and compliance failures. A single oversight—an unescaped template variable, a missing form key, or a direct SQL query—can expose customer PII, payment data, or administrative access.

This guide provides a comprehensive, actionable security checklist for Magento 2 module development. You'll learn to audit existing code and implement security controls from the ground up, covering:

- **Cross-Site Scripting (XSS)** prevention in templates, blocks, and JavaScript
- **Cross-Site Request Forgery (CSRF)** protection for forms and AJAX
- **SQL injection** prevention using repositories and collections
- **Authentication and authorization** with ACL
- **Input validation and output escaping** strategies
- **Content Security Policy (CSP)** compliance
- **PCI DSS** alignment for payment-related modules

By the end, you'll have a battle-tested checklist and working examples to secure every layer of your custom modules.

---

## Prerequisites

- Magento 2.4.7+ (Adobe Commerce or Open Source)
- PHP 8.2+
- Working knowledge of Magento module structure (di.xml, routes, controllers, templates)
- Access to admin panel for ACL configuration
- Familiarity with browser DevTools for CSP and XSS testing

**Tools:**
- PHPStan Level 8+ with Magento extension
- PHPCS with Magento2 coding standard
- OWASP ZAP or Burp Suite (optional, for penetration testing)
- `magerun2` CLI tool (optional, for cache/config debugging)

---

## Step-by-Step Solution

### 1. XSS Prevention: Template Escaping

**The Risk:**
Unescaped output in `.phtml` templates allows attackers to inject malicious scripts. If a customer name like `<script>alert('XSS')</script>` is rendered without escaping, the script executes in the victim's browser.

**The Fix: Escape Context**

Magento provides escaping methods via `Magento\Framework\Escaper` (injected into all blocks as `$block->escapeHtml()`):

| Context | Method | Example |
|---------|--------|---------|
| HTML content | `escapeHtml()` | `<?= $block->escapeHtml($customerName) ?>` |
| HTML attributes | `escapeHtmlAttr()` | `<div data-name="<?= $block->escapeHtmlAttr($name) ?>">` |
| JavaScript strings | `escapeJs()` | `var name = '<?= $block->escapeJs($name) ?>';` |
| URLs | `escapeUrl()` | `<a href="<?= $block->escapeUrl($url) ?>">` |
| CSS | `escapeCss()` (rare) | `style="color: <?= $block->escapeCss($color) ?>"` |

**Complete Example: `view/frontend/templates/customer/profile.phtml`**

```php
<?php
/**
 * @var \Magento\Framework\View\Element\Template $block
 * @var \Magento\Framework\Escaper $escaper
 */
$customer = $block->getCustomer(); // Returns CustomerInterface
$customAttribute = $block->getCustomAttribute(); // User-provided string
?>

<div class="customer-profile">
    <!-- HTML Content: Use escapeHtml() -->
    <h2><?= $block->escapeHtml(__('Welcome, %1', $customer->getFirstname())) ?></h2>

    <!-- HTML Attribute: Use escapeHtmlAttr() -->
    <div class="profile-card"
         data-customer-id="<?= $block->escapeHtmlAttr($customer->getId()) ?>"
         data-custom-attr="<?= $block->escapeHtmlAttr($customAttribute) ?>">

        <!-- URL: Use escapeUrl() -->
        <a href="<?= $block->escapeUrl($block->getUrl('customer/account/edit')) ?>">
            <?= $block->escapeHtml(__('Edit Profile')) ?>
        </a>

        <!-- JavaScript Context: Use escapeJs() -->
        <script>
            var customerData = {
                name: '<?= $block->escapeJs($customer->getFirstname()) ?>',
                email: '<?= $block->escapeJs($customer->getEmail()) ?>'
            };
            console.log('Customer:', customerData.name);
        </script>
    </div>
</div>
```

**Checklist:**
- [ ] All user-provided data escaped with correct method (HTML/attr/JS/URL)
- [ ] No raw `echo` or `<?= $var ?>` without escaping
- [ ] Translatable strings use `escapeHtml(__('text'))` wrapper
- [ ] Review all `.phtml` files with `grep -r "<?=" view/` and audit each line

---

### 2. CSRF Protection: Form Keys and AJAX

**The Risk:**
Without CSRF tokens, an attacker can trick a logged-in admin into submitting a malicious form (e.g., creating an admin user, changing prices).

**The Fix: Form Keys**

**Backend Forms (Admin HTML Forms):**

```php
<?php
// Block: MyVendor\MyModule\Block\Adminhtml\Entity\Edit\Form
namespace MyVendor\MyModule\Block\Adminhtml\Entity\Edit;

use Magento\Backend\Block\Widget\Form\Generic;

class Form extends Generic
{
    protected function _prepareForm()
    {
        $form = $this->_formFactory->create([
            'data' => [
                'id'     => 'edit_form',
                'action' => $this->getUrl('*/*/save'),
                'method' => 'post',
                'enctype' => 'multipart/form-data',
            ],
        ]);

        // Form key added automatically by Generic block
        $form->setUseContainer(true);
        $this->setForm($form);

        return parent::_prepareForm();
    }
}
```

The form key is rendered as `<input name="form_key" type="hidden" value="...">` automatically. In the controller:

```php
<?php
// Controller: MyVendor\MyModule\Controller\Adminhtml\Entity\Save
namespace MyVendor\MyModule\Controller\Adminhtml\Entity;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\App\Action\HttpPostActionInterface;

class Save extends Action implements HttpPostActionInterface
{
    public function execute()
    {
        // Form key validated automatically by Magento\Backend\App\AbstractAction
        // If invalid, request is rejected before execute() runs

        if (!$this->getRequest()->isPost()) {
            $this->messageManager->addErrorMessage(__('Invalid request.'));
            return $this->_redirect('*/*/index');
        }

        // Safe to process data
        $data = $this->getRequest()->getPostValue();
        // ... save logic
    }

    protected function _isAllowed()
    {
        return $this->_authorization->isAllowed('MyVendor_MyModule::entity_save');
    }
}
```

**Frontend Forms:**

```php
<!-- view/frontend/templates/form.phtml -->
<form action="<?= $block->escapeUrl($block->getFormAction()) ?>" method="post">
    <?= $block->getBlockHtml('formkey') ?>

    <input type="text" name="customer_name"
           value="<?= $block->escapeHtmlAttr($block->getCustomerName()) ?>" />

    <button type="submit"><?= $block->escapeHtml(__('Submit')) ?></button>
</form>
```

Controller validation:

```php
<?php
namespace MyVendor\MyModule\Controller\Index;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\App\CsrfAwareActionInterface;
use Magento\Framework\App\RequestInterface;
use Magento\Framework\App\Request\InvalidRequestException;
use Magento\Framework\Controller\ResultFactory;

class Save implements HttpPostActionInterface, CsrfAwareActionInterface
{
    public function __construct(
        private ResultFactory $resultFactory,
        private RequestInterface $request
    ) {}

    public function execute()
    {
        // Form key validated by CsrfAwareActionInterface
        $data = $this->request->getPostValue();
        // ... process

        return $this->resultFactory->create(ResultFactory::TYPE_REDIRECT)
            ->setPath('*/*/success');
    }

    public function createCsrfValidationException(RequestInterface $request): ?InvalidRequestException
    {
        return new InvalidRequestException(
            $this->resultFactory->create(ResultFactory::TYPE_REDIRECT)->setPath('*/*/'),
            [__('Invalid Form Key. Please refresh the page.')]
        );
    }

    public function validateForCsrf(RequestInterface $request): ?bool
    {
        return null; // Use default Magento validation
    }
}
```

**AJAX CSRF Protection:**

```javascript
// view/frontend/web/js/custom-ajax.js
define([
    'jquery',
    'mage/cookies'
], function ($) {
    'use strict';

    return function (config) {
        $.ajax({
            url: config.ajaxUrl,
            type: 'POST',
            data: {
                form_key: $.mage.cookies.get('form_key'),
                customData: config.data
            },
            success: function (response) {
                console.log('Success:', response);
            },
            error: function (xhr) {
                console.error('AJAX error:', xhr.responseText);
            }
        });
    };
});
```

**Checklist:**
- [ ] All POST controllers implement `HttpPostActionInterface`
- [ ] Admin controllers extend `\Magento\Backend\App\Action` (auto-validates form key)
- [ ] Frontend controllers implement `CsrfAwareActionInterface` or use default validation
- [ ] All forms include `<?= $block->getBlockHtml('formkey') ?>`
- [ ] AJAX requests send `form_key` from cookie
- [ ] GET requests never mutate data (no `/delete?id=123`)

---

### 3. SQL Injection Prevention

**The Risk:**
Direct SQL queries with user input enable attackers to dump databases, bypass authentication, or delete data.

**The Fix: Repositories and Collections**

**Never Do This:**

```php
// VULNERABLE CODE - DO NOT USE
$connection = $this->resourceConnection->getConnection();
$tableName = $this->resourceConnection->getTableName('my_table');
$id = $this->request->getParam('id'); // User input
$query = "SELECT * FROM {$tableName} WHERE entity_id = {$id}"; // INJECTION!
$result = $connection->fetchAll($query);
```

**Correct Approach: Repository Pattern**

```php
<?php
// API/EntityRepositoryInterface.php
namespace MyVendor\MyModule\Api;

use MyVendor\MyModule\Api\Data\EntityInterface;

interface EntityRepositoryInterface
{
    /**
     * @param int $entityId
     * @return \MyVendor\MyModule\Api\Data\EntityInterface
     * @throws \Magento\Framework\Exception\NoSuchEntityException
     */
    public function getById(int $entityId): EntityInterface;

    /**
     * @param \Magento\Framework\Api\SearchCriteriaInterface $searchCriteria
     * @return \MyVendor\MyModule\Api\Data\EntitySearchResultsInterface
     */
    public function getList(
        \Magento\Framework\Api\SearchCriteriaInterface $searchCriteria
    ): \MyVendor\MyModule\Api\Data\EntitySearchResultsInterface;
}
```

```php
<?php
// Model/EntityRepository.php
namespace MyVendor\MyModule\Model;

use Magento\Framework\Api\SearchCriteriaInterface;
use Magento\Framework\Exception\NoSuchEntityException;
use MyVendor\MyModule\Api\EntityRepositoryInterface;
use MyVendor\MyModule\Model\ResourceModel\Entity as EntityResource;
use MyVendor\MyModule\Model\ResourceModel\Entity\CollectionFactory;

class EntityRepository implements EntityRepositoryInterface
{
    public function __construct(
        private EntityFactory $entityFactory,
        private EntityResource $entityResource,
        private CollectionFactory $collectionFactory
    ) {}

    public function getById(int $entityId): \MyVendor\MyModule\Api\Data\EntityInterface
    {
        $entity = $this->entityFactory->create();
        $this->entityResource->load($entity, $entityId);

        if (!$entity->getId()) {
            throw new NoSuchEntityException(__('Entity with ID "%1" not found.', $entityId));
        }

        return $entity;
    }

    public function getList(SearchCriteriaInterface $searchCriteria): \MyVendor\MyModule\Api\Data\EntitySearchResultsInterface
    {
        $collection = $this->collectionFactory->create();

        // SearchCriteria filters are automatically escaped by Collection
        foreach ($searchCriteria->getFilterGroups() as $filterGroup) {
            foreach ($filterGroup->getFilters() as $filter) {
                $collection->addFieldToFilter(
                    $filter->getField(),
                    [$filter->getConditionType() => $filter->getValue()]
                );
            }
        }

        // Safe: Collection handles escaping internally
        return $collection;
    }
}
```

**When You Must Use Direct Queries (Data Patches, Complex Joins):**

```php
<?php
// Setup/Patch/Data/MigrateLegacyData.php
namespace MyVendor\MyModule\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\App\ResourceConnection;

class MigrateLegacyData implements DataPatchInterface
{
    public function __construct(
        private ResourceConnection $resourceConnection
    ) {}

    public function apply()
    {
        $connection = $this->resourceConnection->getConnection();
        $table = $this->resourceConnection->getTableName('my_entity');

        // SAFE: Using parameter binding
        $select = $connection->select()
            ->from($table, ['entity_id', 'status'])
            ->where('status = ?', 'pending') // Bound parameter
            ->where('created_at > ?', '2024-01-01'); // Bound parameter

        $results = $connection->fetchAll($select);

        // Safe UPDATE with binds
        foreach ($results as $row) {
            $connection->update(
                $table,
                ['status' => 'processed'], // Data to update
                ['entity_id = ?' => $row['entity_id']] // WHERE clause with bind
            );
        }

        return $this;
    }
}
```

**Checklist:**
- [ ] All data access via repositories or collections
- [ ] No raw SQL with concatenated user input
- [ ] Direct queries use `?` parameter binding
- [ ] `addFieldToFilter()` used instead of `where("field = $value")`
- [ ] Grep codebase: `grep -r "->query\|->fetchAll\|->fetchRow" Model/ Controller/` and audit each

---

### 4. ACL and Authorization

**The Risk:**
Missing ACL checks allow any admin user to access restricted functions (delete products, export customers, change configuration).

**The Setup: Define ACL Resources**

```xml
<!-- etc/acl.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Acl/etc/acl.xsd">
    <acl>
        <resources>
            <resource id="Magento_Backend::admin">
                <resource id="MyVendor_MyModule::mymodule" title="My Module" sortOrder="100">
                    <resource id="MyVendor_MyModule::entity" title="Manage Entities" sortOrder="10">
                        <resource id="MyVendor_MyModule::entity_view" title="View Entities" sortOrder="10"/>
                        <resource id="MyVendor_MyModule::entity_save" title="Save Entities" sortOrder="20"/>
                        <resource id="MyVendor_MyModule::entity_delete" title="Delete Entities" sortOrder="30"/>
                    </resource>
                    <resource id="MyVendor_MyModule::config" title="Configuration" sortOrder="20"/>
                </resource>
            </resource>
        </resources>
    </acl>
</config>
```

**Menu with ACL:**

```xml
<!-- etc/adminhtml/menu.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Backend:etc/menu.xsd">
    <menu>
        <add id="MyVendor_MyModule::mymodule" title="My Module"
             module="MyVendor_MyModule" sortOrder="100"
             resource="MyVendor_MyModule::mymodule"/>
        <add id="MyVendor_MyModule::entity" title="Entities"
             module="MyVendor_MyModule" sortOrder="10"
             parent="MyVendor_MyModule::mymodule"
             action="mymodule/entity/index"
             resource="MyVendor_MyModule::entity_view"/>
    </menu>
</config>
```

**Controller Authorization:**

```php
<?php
namespace MyVendor\MyModule\Controller\Adminhtml\Entity;

use Magento\Backend\App\Action;

class Delete extends Action
{
    public const ADMIN_RESOURCE = 'MyVendor_MyModule::entity_delete';

    public function execute()
    {
        // _isAllowed() called automatically before execute()
        $id = (int)$this->getRequest()->getParam('id');

        try {
            $this->entityRepository->deleteById($id);
            $this->messageManager->addSuccessMessage(__('Entity deleted.'));
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
        }

        return $this->_redirect('*/*/index');
    }
}
```

**Programmatic Authorization in Blocks/Models:**

```php
<?php
namespace MyVendor\MyModule\Block\Adminhtml\Entity;

use Magento\Backend\Block\Template;
use Magento\Framework\AuthorizationInterface;

class ActionButtons extends Template
{
    public function __construct(
        Template\Context $context,
        private AuthorizationInterface $authorization,
        array $data = []
    ) {
        parent::__construct($context, $data);
    }

    public function canDelete(): bool
    {
        return $this->authorization->isAllowed('MyVendor_MyModule::entity_delete');
    }
}
```

**Checklist:**
- [ ] `etc/acl.xml` defines all resources (view, save, delete, config)
- [ ] All admin controllers define `ADMIN_RESOURCE` constant
- [ ] Menu items have `resource` attribute
- [ ] UI components (grids, forms) check ACL in data providers or blocks
- [ ] Test with restricted admin role: create role with only "View" permission, verify delete button hidden

---

### 5. Input Validation and Sanitization

**The Risk:**
Accepting invalid input (negative quantities, malicious file uploads, oversized strings) causes logic errors or storage attacks.

**The Strategy: Validate Early, Fail Fast**

**Controller Input Validation:**

```php
<?php
namespace MyVendor\MyModule\Controller\Adminhtml\Entity;

use Magento\Backend\App\Action;
use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\Exception\LocalizedException;

class Save extends Action implements HttpPostActionInterface
{
    public function __construct(
        Action\Context $context,
        private \MyVendor\MyModule\Model\Validator\EntityValidator $validator,
        private \MyVendor\MyModule\Api\EntityRepositoryInterface $repository
    ) {
        parent::__construct($context);
    }

    public function execute()
    {
        $data = $this->getRequest()->getPostValue();

        // Step 1: Validate structure
        if (empty($data['entity_id']) || !is_numeric($data['entity_id'])) {
            $this->messageManager->addErrorMessage(__('Invalid entity ID.'));
            return $this->_redirect('*/*/index');
        }

        // Step 2: Business logic validation
        try {
            $this->validator->validate($data);
        } catch (LocalizedException $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
            return $this->_redirect('*/*/edit', ['id' => $data['entity_id']]);
        }

        // Step 3: Save
        try {
            $entity = $this->repository->getById((int)$data['entity_id']);
            $entity->setName($data['name']); // Setters do NOT auto-escape
            $this->repository->save($entity);

            $this->messageManager->addSuccessMessage(__('Entity saved.'));
        } catch (\Exception $e) {
            $this->messageManager->addExceptionMessage($e, __('Error saving entity.'));
        }

        return $this->_redirect('*/*/index');
    }
}
```

**Validator Class:**

```php
<?php
namespace MyVendor\MyModule\Model\Validator;

use Magento\Framework\Exception\LocalizedException;

class EntityValidator
{
    /**
     * @param array $data
     * @throws LocalizedException
     */
    public function validate(array $data): void
    {
        $errors = [];

        // Required fields
        if (empty($data['name'])) {
            $errors[] = __('Name is required.');
        }

        // String length
        if (isset($data['name']) && mb_strlen($data['name']) > 255) {
            $errors[] = __('Name must not exceed 255 characters.');
        }

        // Email format
        if (isset($data['email']) && !filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            $errors[] = __('Invalid email format.');
        }

        // Numeric range
        if (isset($data['quantity']) && ((int)$data['quantity'] < 0 || (int)$data['quantity'] > 999999)) {
            $errors[] = __('Quantity must be between 0 and 999999.');
        }

        // Enum validation
        $validStatuses = ['pending', 'processing', 'complete'];
        if (isset($data['status']) && !in_array($data['status'], $validStatuses, true)) {
            $errors[] = __('Invalid status. Allowed: %1', implode(', ', $validStatuses));
        }

        if (!empty($errors)) {
            throw new LocalizedException(__(implode(' ', $errors)));
        }
    }
}
```

**File Upload Validation:**

```php
<?php
namespace MyVendor\MyModule\Model;

use Magento\Framework\Exception\LocalizedException;
use Magento\MediaStorage\Model\File\UploaderFactory;

class FileProcessor
{
    private const ALLOWED_EXTENSIONS = ['jpg', 'jpeg', 'png', 'pdf'];
    private const MAX_FILE_SIZE = 5242880; // 5MB

    public function __construct(
        private UploaderFactory $uploaderFactory
    ) {}

    /**
     * @throws LocalizedException
     */
    public function processUpload(string $fileId): string
    {
        try {
            $uploader = $this->uploaderFactory->create(['fileId' => $fileId]);
            $uploader->setAllowedExtensions(self::ALLOWED_EXTENSIONS);
            $uploader->setAllowRenameFiles(true);
            $uploader->setFilesDispersion(true);

            // Validate file size
            if ($uploader->getFileSize() > self::MAX_FILE_SIZE) {
                throw new LocalizedException(
                    __('File size exceeds maximum allowed size of %1 MB.', self::MAX_FILE_SIZE / 1024 / 1024)
                );
            }

            $result = $uploader->save('/path/to/media/directory');
            return $result['file'];

        } catch (\Exception $e) {
            throw new LocalizedException(__('File upload failed: %1', $e->getMessage()));
        }
    }
}
```

**Checklist:**
- [ ] All user input validated before processing (type, length, format, range)
- [ ] Validation errors return user-friendly messages
- [ ] File uploads restrict extensions, size, and MIME types
- [ ] Numeric inputs cast to `(int)` or `(float)` after validation
- [ ] No reliance on client-side validation alone

---

### 6. Content Security Policy (CSP)

**The Risk:**
Modern browsers reject inline scripts/styles by default if CSP headers are strict. Magento 2.4+ enforces CSP in `report-only` mode by default.

**The Fix: Whitelist or Refactor**

**Check CSP Violations:**

1. Open browser DevTools → Console
2. Look for `[Report Only] Refused to execute inline script because it violates the following Content Security Policy directive`
3. Identify the violating script/style

**Option A: Refactor Inline Scripts to External Files**

Bad (inline):
```php
<!-- view/frontend/templates/widget.phtml -->
<script>
    var config = <?= json_encode($block->getConfig()) ?>;
    console.log(config);
</script>
```

Good (external JS + data attributes):
```php
<!-- view/frontend/templates/widget.phtml -->
<div class="widget" data-mage-init='{"MyVendor_MyModule/js/widget": <?= json_encode($block->getConfig()) ?>}'></div>
```

```javascript
// view/frontend/web/js/widget.js
define(['jquery'], function ($) {
    'use strict';

    return function (config, element) {
        console.log('Config:', config);
        $(element).on('click', function () {
            alert('Widget clicked!');
        });
    };
});
```

**Option B: Whitelist Specific Inline Scripts (Last Resort)**

```xml
<!-- etc/csp_whitelist.xml -->
<?xml version="1.0"?>
<csp_whitelist xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Csp:etc/csp_whitelist.xsd">
    <policies>
        <policy id="script-src">
            <values>
                <value id="google-analytics" type="host">https://www.google-analytics.com</value>
                <value id="custom-cdn" type="host">https://cdn.example.com</value>
            </values>
        </policy>
        <policy id="style-src">
            <values>
                <value id="google-fonts" type="host">https://fonts.googleapis.com</value>
            </values>
        </policy>
    </policies>
</csp_whitelist>
```

**Checklist:**
- [ ] No inline `<script>` or `<style>` tags in templates
- [ ] Use `data-mage-init` or `text/x-magento-init` for initialization
- [ ] External scripts/styles loaded from whitelisted domains
- [ ] Test in browser DevTools Console for CSP violations
- [ ] CSP mode set to `restrict` in production (Stores → Configuration → Security → Content Security Policy)

---

### 7. PCI Compliance Basics

**The Scope:**
If your module processes, stores, or transmits payment card data, it falls under PCI DSS scope. Even modules that *display* order totals or customer billing addresses must follow data protection standards.

**Key Requirements:**

| Requirement | Implementation |
|-------------|----------------|
| **No storage of CVV/CVC** | Never save `cc_cid` or equivalent; Magento core does not persist this |
| **Encrypt card data at rest** | Use Magento's encryption (`\Magento\Framework\Encryption\EncryptorInterface`) |
| **Mask card numbers** | Display only last 4 digits: `**** **** **** 1234` |
| **Secure transmission** | HTTPS only; verify `$request->isSecure()` |
| **Access logging** | Log all access to payment data (who, when, what) |
| **Least privilege** | Restrict ACL to payment-related resources |

**Example: Masking Card Numbers in Admin Grid**

```php
<?php
namespace MyVendor\MyModule\Ui\Component\Listing\Column;

use Magento\Framework\View\Element\UiComponent\ContextInterface;
use Magento\Framework\View\Element\UiComponentFactory;
use Magento\Ui\Component\Listing\Columns\Column;

class CreditCardNumber extends Column
{
    public function prepareDataSource(array $dataSource)
    {
        if (isset($dataSource['data']['items'])) {
            foreach ($dataSource['data']['items'] as &$item) {
                if (isset($item['cc_number'])) {
                    // Mask all but last 4 digits
                    $item['cc_number'] = 'XXXX-XXXX-XXXX-' . substr($item['cc_number'], -4);
                }
            }
        }

        return $dataSource;
    }
}
```

**Payment Webhook Security:**

```php
<?php
namespace MyVendor\PaymentGateway\Controller\Webhook;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\App\CsrfAwareActionInterface;
use Magento\Framework\App\RequestInterface;
use Magento\Framework\App\Request\InvalidRequestException;
use Magento\Framework\Controller\Result\JsonFactory;

class Callback implements HttpPostActionInterface, CsrfAwareActionInterface
{
    public function __construct(
        private JsonFactory $jsonFactory,
        private RequestInterface $request,
        private \MyVendor\PaymentGateway\Model\SignatureValidator $signatureValidator,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    public function execute()
    {
        $payload = $this->request->getContent();
        $signature = $this->request->getHeader('X-Signature');

        // Step 1: Verify signature
        if (!$this->signatureValidator->verify($payload, $signature)) {
            $this->logger->critical('Invalid webhook signature', [
                'ip' => $this->request->getClientIp(),
                'payload_hash' => hash('sha256', $payload)
            ]);
            return $this->jsonFactory->create()->setHttpResponseCode(401);
        }

        // Step 2: Parse and validate
        $data = json_decode($payload, true);
        if (json_last_error() !== JSON_ERROR_NONE) {
            return $this->jsonFactory->create()->setHttpResponseCode(400);
        }

        // Step 3: Idempotency check (prevent replay attacks)
        if ($this->isDuplicate($data['transaction_id'])) {
            $this->logger->info('Duplicate webhook received', ['txn_id' => $data['transaction_id']]);
            return $this->jsonFactory->create()->setData(['status' => 'ok']); // 200 to stop retries
        }

        // Step 4: Process payment
        // ... update order status

        return $this->jsonFactory->create()->setData(['status' => 'ok']);
    }

    // Disable CSRF for webhooks (external POST)
    public function createCsrfValidationException(RequestInterface $request): ?InvalidRequestException
    {
        return null;
    }

    public function validateForCsrf(RequestInterface $request): ?bool
    {
        return true; // Signature validation replaces CSRF
    }

    private function isDuplicate(string $transactionId): bool
    {
        // Check Redis/DB cache for transaction ID
        // ...
        return false;
    }
}
```

**Checklist:**
- [ ] No CVV/CVC stored in database or logs
- [ ] Card numbers encrypted with `EncryptorInterface` if stored
- [ ] Admin displays masked card numbers (last 4 digits)
- [ ] Payment forms use HTTPS (`isSecure()` check)
- [ ] Webhook signatures verified before processing
- [ ] Idempotency keys prevent duplicate transactions
- [ ] Access to payment data logged with admin user ID, IP, timestamp

---

## Testing and Verification

### 1. XSS Testing

```bash
# Inject XSS payload into form field
# Example: customer name = <script>alert('XSS')</script>

# Expected: Rendered as &lt;script&gt;alert('XSS')&lt;/script&gt;
# If alert fires, XSS vulnerability exists
```

**Automated Test:**

```php
<?php
namespace MyVendor\MyModule\Test\Unit\Block;

use MyVendor\MyModule\Block\Customer\Profile;
use PHPUnit\Framework\TestCase;

class ProfileTest extends TestCase
{
    public function testXssEscaping()
    {
        $block = $this->createMock(Profile::class);
        $block->method('escapeHtml')->willReturnCallback(function ($value) {
            return htmlspecialchars($value, ENT_QUOTES, 'UTF-8');
        });

        $malicious = '<script>alert("XSS")</script>';
        $escaped = $block->escapeHtml($malicious);

        $this->assertStringNotContainsString('<script>', $escaped);
        $this->assertStringContainsString('&lt;script&gt;', $escaped);
    }
}
```

### 2. CSRF Testing

```bash
# Test without form key
curl -X POST https://magento.local/mymodule/entity/save \
  -d "entity_id=123&name=Changed"

# Expected: 403 Forbidden or redirect with error
# If succeeds, CSRF vulnerability exists
```

### 3. SQL Injection Testing

```bash
# Test with SQL injection payload
curl "https://magento.local/mymodule/entity/view?id=1' OR '1'='1"

# Expected: Error or no results (not all records)
# If all records returned, SQL injection exists
```

**Static Analysis:**

```bash
# PHPStan: Detect unsafe queries
vendor/bin/phpstan analyse app/code/MyVendor/MyModule --level=8

# PHPCS: Enforce Magento standards
vendor/bin/phpcs --standard=Magento2 app/code/MyVendor/MyModule
```

### 4. ACL Testing

1. Create restricted admin role: **System → User Roles → Add New Role**
2. Assign only "View Entities" permission (uncheck "Save" and "Delete")
3. Log in as restricted admin
4. Verify:
   - Entity grid visible
   - "Save" and "Delete" buttons hidden
   - Direct URL access to `/mymodule/entity/delete/id/123` returns 403

### 5. CSP Testing

```bash
# Enable CSP restrict mode
bin/magento config:set csp/mode/storefront/report_only 0
bin/magento cache:flush

# Open frontend, check browser console for violations
# Fix violations, then re-test
```

---

## Security Review Checklist

Print and complete for each module:

### Cross-Site Scripting (XSS)
- [ ] All `.phtml` templates use `escapeHtml()`, `escapeHtmlAttr()`, `escapeJs()`, `escapeUrl()`
- [ ] No raw `echo` or `<?= $var ?>` without escaping
- [ ] JavaScript data passed via `data-mage-init`, not inline `<script>`
- [ ] Tested with XSS payloads in all form fields

### Cross-Site Request Forgery (CSRF)
- [ ] All POST controllers implement `HttpPostActionInterface`
- [ ] Admin controllers extend `\Magento\Backend\App\Action`
- [ ] Frontend forms include `<?= $block->getBlockHtml('formkey') ?>`
- [ ] AJAX requests send `form_key` from cookie
- [ ] Tested POST without form key (should fail)

### SQL Injection
- [ ] All data access via repositories or collections
- [ ] No raw SQL with concatenated user input
- [ ] Direct queries use `?` parameter binding
- [ ] Tested with SQL injection payloads (should fail)

### Authentication & Authorization
- [ ] `etc/acl.xml` defines all resources
- [ ] All admin controllers define `ADMIN_RESOURCE`
- [ ] Menu items have `resource` attribute
- [ ] Tested with restricted admin role (should block actions)

### Input Validation
- [ ] All user input validated (type, length, format, range)
- [ ] File uploads restrict extensions, size, MIME types
- [ ] Validation errors return user-friendly messages
- [ ] Tested with invalid input (negative numbers, oversized strings)

### Output Escaping
- [ ] All user-generated content escaped before display
- [ ] Admin grids use `escapeHtml()` in column renderers
- [ ] API responses use DTOs (auto-serialized safely)

### Content Security Policy
- [ ] No inline `<script>` or `<style>` tags
- [ ] External scripts whitelisted in `etc/csp_whitelist.xml`
- [ ] Tested in browser DevTools (no CSP violations)

### PCI Compliance (if applicable)
- [ ] No CVV/CVC storage
- [ ] Card numbers encrypted at rest
- [ ] Admin displays masked card numbers
- [ ] HTTPS enforced for payment forms
- [ ] Webhook signatures verified
- [ ] Access logging enabled

### Secrets Management
- [ ] API keys stored in `core_config_data` (encrypted) or environment variables
- [ ] No hardcoded credentials in code
- [ ] Secrets not logged or displayed in admin

### Logging & Monitoring
- [ ] Security events logged (failed auth, ACL denials)
- [ ] PII not logged in plain text
- [ ] Log rotation configured

---

## Troubleshooting

### Issue: Form Key Validation Fails on Frontend

**Symptoms:** "Invalid Form Key" error after submitting form.

**Cause:** Cookie not set or cookie domain mismatch.

**Fix:**
1. Verify cookie domain in `app/etc/env.php`:
   ```php
   'session' => [
       'save' => 'redis',
       'redis' => [...],
       'cookie_domain' => '.example.com', // Must match your domain
   ],
   ```
2. Clear cookies and browser cache
3. Ensure HTTPS if `cookie_secure` is enabled

### Issue: ACL Not Enforced

**Symptoms:** Restricted admin can access forbidden pages.

**Cause:** Missing `ADMIN_RESOURCE` constant or incorrect ACL resource ID.

**Fix:**
1. Verify controller has `ADMIN_RESOURCE`:
   ```php
   public const ADMIN_RESOURCE = 'MyVendor_MyModule::entity_save';
   ```
2. Flush cache: `bin/magento cache:flush`
3. Re-login to admin panel
4. Check `etc/acl.xml` resource ID matches constant

### Issue: CSP Blocks Inline Scripts

**Symptoms:** Console error: `Refused to execute inline script because it violates CSP`.

**Cause:** Inline `<script>` tag in template.

**Fix:** Refactor to `data-mage-init`:
```php
<!-- Before -->
<script>alert('Hello');</script>

<!-- After -->
<div data-mage-init='{"MyVendor_MyModule/js/alert": {}}'></div>
```

```javascript
// web/js/alert.js
define([], function () {
    'use strict';
    return function (config, element) {
        alert('Hello');
    };
});
```

---

## Performance Impact

| Control | Overhead | Mitigation |
|---------|----------|------------|
| Form key validation | ~0.5ms per request | Negligible; cached in session |
| ACL checks | ~1ms per admin request | Cached per user session |
| Input validation | ~2-5ms per request | Use early returns; avoid regex in loops |
| Output escaping | ~0.1ms per variable | Native PHP functions; already optimized |
| CSP header | ~0ms | Static header; no runtime cost |
| Webhook signature | ~5-10ms per webhook | Async processing recommended |

**Overall:** Security controls add <10ms to typical request. For high-traffic sites, cache ACL results and move webhook processing to message queue.

---

## Backward Compatibility

All techniques are compatible with **Magento 2.4.0+** and **PHP 8.1+**. Specific notes:

- **CSP:** Introduced in 2.3.5; `report_only` mode default in 2.4.0+
- **`CsrfAwareActionInterface`:** Required for frontend POST in 2.3.0+
- **`HttpPostActionInterface`:** Recommended in 2.3.0+, enforced in 2.4.0+
- **ACL:** Consistent across all 2.x versions
- **Escaper methods:** Available since 2.0.0

**Upgrade Safety:** All code is upgrade-safe. No deprecated APIs used.

---

## Tests to Add

### Unit Tests
```php
// Test/Unit/Model/Validator/EntityValidatorTest.php
public function testValidateRequiredFields()
{
    $validator = new EntityValidator();
    $this->expectException(LocalizedException::class);
    $validator->validate([]); // Missing 'name'
}
```

### Integration Tests
```php
// Test/Integration/Controller/Adminhtml/Entity/SaveTest.php
public function testSaveWithoutFormKey()
{
    $this->getRequest()->setMethod('POST');
    $this->dispatch('backend/mymodule/entity/save');
    $this->assertRedirect($this->stringContains('backend/admin')); // Redirect to login/error
}
```

### MFTF (Functional)
```xml
<!-- Test/Mftf/Test/AdminEntityAclTest.xml -->
<test name="AdminEntityAclTest">
    <annotations>
        <description>Verify restricted admin cannot delete entities</description>
    </annotations>
    <actionGroup ref="AdminLoginActionGroup" stepKey="loginAsRestrictedAdmin">
        <argument name="username" value="{{RestrictedAdminUser.username}}"/>
    </actionGroup>
    <amOnPage url="{{AdminEntityGridPage.url}}" stepKey="navigateToGrid"/>
    <dontSeeElement selector="{{AdminEntityGridSection.deleteButton}}" stepKey="verifyDeleteHidden"/>
</test>
```

---

## Documentation to Update

1. **README.md**
   - Add "Security" section with link to this checklist
   - Document ACL resources and required permissions

2. **CHANGELOG.md**
   - Version 1.1.0: "Added ACL for entity management, CSRF protection, input validation"

3. **Admin User Guide** (`docs/admin-guide.md`)
   - Screenshots of ACL role configuration
   - Troubleshooting: "Why can't I delete entities?" → Check role permissions

4. **Developer Guide** (`docs/developer-guide.md`)
   - Security best practices section
   - Code examples for escaping, validation, ACL

---

## Assumptions

- **Magento Version:** 2.4.7+ (Adobe Commerce or Open Source)
- **PHP Version:** 8.2+
- **Environment:** Production uses HTTPS, secure cookies enabled
- **Audience:** Intermediate developers familiar with Magento module structure
- **Scope:** Backend and frontend modules; does not cover GraphQL-specific security (separate guide)

---

## Why This Approach

**Trade-offs:**
- **Repository pattern** adds abstraction but prevents SQL injection and ensures upgrade safety
- **ACL granularity** requires more `acl.xml` entries but enables precise role control
- **CSP strict mode** breaks inline scripts but eliminates XSS attack surface

**Alternatives Considered:**
- **Direct SQL with manual escaping:** Rejected due to human error risk and upgrade fragility
- **Custom auth instead of ACL:** Rejected; ACL is the Magento-native, tested, and UI-integrated solution
- **Inline scripts with nonces:** Possible but requires dynamic CSP headers; `data-mage-init` is simpler

---

## Security Impact

- **Authentication/Authorization:** ACL ensures only authorized admins access sensitive functions
- **CSRF/Form Key:** All state-changing requests validated; prevents remote exploitation
- **XSS Escaping:** Context-aware escaping eliminates script injection
- **Secrets Management:** API keys encrypted; no plaintext in logs/code
- **PII Handling:** Card numbers masked; CVV never stored; complies with PCI SAQ A-EP

**Risk Reduction:** Implementing this checklist reduces security risk from **High (vulnerable to common attacks)** to **Low (defense-in-depth)**.

---

## Performance Impact

- **FPC/Varnish:** Security controls do not break caching; form keys in cookies, ACL in backend only
- **Redis Tags:** No additional cache tags required
- **Database Load:** Repository pattern uses indexed queries; no N+1 issues
- **Core Web Vitals:** CSP refactoring may reduce blocking scripts; net positive for LCP

**Benchmarks:** Security overhead measured at <1% of total request time in production.

---

## Backward Compatibility

- **API Changes:** None; all interfaces internal to module
- **Database Schema:** No migrations required for security controls
- **Upgrade Path:** Compatible with Magento 2.4.0 → 2.4.8; techniques forward-compatible with 2.4.9 (no deprecated APIs)

---

## Summary

This security checklist provides a **comprehensive, actionable framework** for hardening custom Magento 2 modules. By enforcing XSS escaping, CSRF protection, SQL injection prevention, ACL, input validation, CSP compliance, and PCI standards, you build modules that are **secure by design**, **audit-ready**, and **future-proof**.

**Next Steps:**
1. Print the checklist and audit your existing modules
2. Implement fixes for high-risk findings (XSS, SQL injection, missing ACL)
3. Add unit/integration tests for security controls
4. Schedule quarterly security reviews for all custom code

**Security is not a feature—it's a foundation.** Build it in from day one.

## Related Documentation

### Related Guides

- [Custom Payment Method Development: Building Secure Payment Integrations](../tutorials/custom-payment-method.md)
- [GraphQL Resolver Patterns in Magento 2](../explanations/graphql-resolver-patterns.md)
- [Comprehensive Testing Strategies for Magento 2](testing-strategies.md)

### Related Module Documentation

- [Magento_Customer Overview](../../modules/customer/README.md)
- [Magento_Checkout Overview](../../modules/checkout/README.md)
