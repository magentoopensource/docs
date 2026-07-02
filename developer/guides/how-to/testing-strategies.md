---
title: "Comprehensive Testing Strategies for Magento 2"
description: "Developer guide: Comprehensive Testing Strategies for Magento 2"
type: "how-to"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "30 minutes"
topics:
  - testing
  - phpunit
  - integration tests
  - mftf
  - code quality
  - ci/cd
last_updated: "2026-02-07"
---

# Comprehensive Testing Strategies for Magento 2

## Overview

Comprehensive testing is non-negotiable for production Magento applications. This guide provides battle-tested strategies for unit, integration, API functional, and MFTF testing, with patterns that ensure code quality, prevent regressions, and enable confident deployments.

**What you'll learn:**
- Unit testing with PHPUnit (mocking, ObjectManager usage)
- Integration tests with database fixtures
- API functional testing for REST/GraphQL endpoints
- MFTF (Magento Functional Testing Framework) for E2E scenarios
- Test coverage requirements and tooling
- CI/CD integration patterns

**Prerequisites:**
- PHP 8.2+ knowledge (typed properties, attributes)
- Magento module structure and DI
- Basic PHPUnit experience
- Understanding of service contracts

---

## Testing Pyramid for Magento

```
        /\
       /  \      E2E/MFTF (10%)
      /____\
     /      \    Integration (30%)
    /________\
   /          \  Unit Tests (60%)
  /__________\
```

**Distribution rationale:**
- **Unit tests (60%):** Fast, isolated, test business logic
- **Integration tests (30%):** Verify DI, DB interactions, service contracts
- **MFTF/E2E (10%):** Critical user journeys, smoke tests

---

## Unit Testing with PHPUnit

### Principles

1. **Isolated:** No database, no filesystem, no network
2. **Fast:** <100ms per test
3. **Deterministic:** Same input always produces same output
4. **Single responsibility:** One assertion per test (guideline)

### Test Structure

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Unit\Model;

use PHPUnit\Framework\TestCase;
use Vendor\Module\Model\OrderProcessor;
use Vendor\Module\Api\OrderValidatorInterface;
use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Sales\Api\Data\OrderInterface;
use Psr\Log\LoggerInterface;

class OrderProcessorTest extends TestCase
{
    private OrderRepositorInterface $orderRepositoryMock;
    private OrderValidatorInterface $validatorMock;
    private LoggerInterface $loggerMock;
    private OrderProcessor $orderProcessor;

    protected function setUp(): void
    {
        // Create mocks
        $this->orderRepositoryMock = $this->createMock(OrderRepositoryInterface::class);
        $this->validatorMock = $this->createMock(OrderValidatorInterface::class);
        $this->loggerMock = $this->createMock(LoggerInterface::class);

        // Instantiate SUT (System Under Test)
        $this->orderProcessor = new OrderProcessor(
            $this->orderRepositoryMock,
            $this->validatorMock,
            $this->loggerMock
        );
    }

    public function testProcessOrderSuccess(): void
    {
        // Arrange
        $orderId = 123;
        $orderMock = $this->createMock(OrderInterface::class);
        $orderMock->method('getId')->willReturn($orderId);
        $orderMock->method('getStatus')->willReturn('pending');

        $this->orderRepositoryMock->method('get')
            ->with($orderId)
            ->willReturn($orderMock);

        $this->validatorMock->method('validate')
            ->with($orderMock)
            ->willReturn(true);

        $this->orderRepositoryMock->expects($this->once())
            ->method('save')
            ->with($orderMock);

        // Act
        $result = $this->orderProcessor->process($orderId);

        // Assert
        $this->assertTrue($result);
    }

    public function testProcessOrderValidationFails(): void
    {
        // Arrange
        $orderId = 123;
        $orderMock = $this->createMock(OrderInterface::class);

        $this->orderRepositoryMock->method('get')
            ->with($orderId)
            ->willReturn($orderMock);

        $this->validatorMock->method('validate')
            ->with($orderMock)
            ->willReturn(false);

        $this->loggerMock->expects($this->once())
            ->method('warning')
            ->with(
                $this->stringContains('Order validation failed'),
                $this->arrayHasKey('order_id')
            );

        // Act
        $result = $this->orderProcessor->process($orderId);

        // Assert
        $this->assertFalse($result);
    }

    public function testProcessOrderThrowsExceptionOnRepositoryError(): void
    {
        // Arrange
        $orderId = 123;
        $exception = new \Magento\Framework\Exception\NoSuchEntityException(
            __('Order not found')
        );

        $this->orderRepositoryMock->method('get')
            ->with($orderId)
            ->willThrowException($exception);

        // Assert
        $this->expectException(\Magento\Framework\Exception\NoSuchEntityException::class);
        $this->expectExceptionMessage('Order not found');

        // Act
        $this->orderProcessor->process($orderId);
    }
}
```

### Mocking Strategies

**1. Simple mocks (no behavior):**

```php
$mock = $this->createMock(MyInterface::class);
```

**2. Stub methods (return values):**

```php
$mock = $this->createMock(OrderInterface::class);
$mock->method('getId')->willReturn(123);
$mock->method('getStatus')->willReturn('pending');
```

**3. Consecutive calls:**

```php
$mock->method('getStatus')
    ->willReturnOnConsecutiveCalls('pending', 'processing', 'complete');
```

**4. Callback logic:**

```php
$mock->method('validate')
    ->willReturnCallback(function (OrderInterface $order) {
        return $order->getGrandTotal() > 0;
    });
```

**5. Expectations (verify calls):**

```php
$mock->expects($this->once())
    ->method('save')
    ->with($this->equalTo($expectedOrder));
```

### Testing Exceptions

```php
public function testInvalidInputThrowsException(): void
{
    $this->expectException(\InvalidArgumentException::class);
    $this->expectExceptionMessage('Order ID must be positive');

    $this->orderProcessor->process(-1);
}
```

### Data Providers for Parametric Tests

```php
/**
 * @dataProvider orderStatusProvider
 */
public function testGetStatusLabel(string $status, string $expectedLabel): void
{
    $this->assertEquals($expectedLabel, $this->orderProcessor->getStatusLabel($status));
}

public function orderStatusProvider(): array
{
    return [
        'pending' => ['pending', 'Pending'],
        'processing' => ['processing', 'Processing'],
        'complete' => ['complete', 'Complete'],
        'canceled' => ['canceled', 'Canceled'],
    ];
}
```

### Avoid ObjectManager in Unit Tests

**BAD:**

```php
$objectManager = \Magento\TestFramework\Helper\Bootstrap::getObjectManager();
$orderProcessor = $objectManager->create(OrderProcessor::class);
```

**GOOD:**

```php
$orderProcessor = new OrderProcessor(
    $this->orderRepositoryMock,
    $this->validatorMock,
    $this->loggerMock
);
```

**Why?** Unit tests must be isolated. ObjectManager loads entire Magento framework, making tests slow and brittle.

---

## Integration Testing

### When to Use Integration Tests

Use integration tests when:
- Testing database interactions (repositories, resource models)
- Verifying DI configuration (`di.xml`)
- Testing service contracts with real implementations
- Validating events/observers
- Testing plugins (interceptors)

### Basic Integration Test Structure

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Integration\Model;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;
use Vendor\Module\Api\OrderProcessorInterface;
use Magento\Sales\Api\OrderRepositoryInterface;

class OrderProcessorTest extends TestCase
{
    private OrderProcessorInterface $orderProcessor;
    private OrderRepositoryInterface $orderRepository;

    protected function setUp(): void
    {
        $objectManager = Bootstrap::getObjectManager();
        $this->orderProcessor = $objectManager->get(OrderProcessorInterface::class);
        $this->orderRepository = $objectManager->get(OrderRepositoryInterface::class);
    }

    /**
     * @magentoDataFixture Magento/Sales/_files/order.php
     * @magentoDbIsolation enabled
     * @magentoAppIsolation enabled
     */
    public function testProcessOrderUpdatesStatus(): void
    {
        // Fixture creates order with ID 100000001
        $orderId = 100000001;

        $this->orderProcessor->process($orderId);

        $order = $this->orderRepository->get($orderId);
        $this->assertEquals('processing', $order->getStatus());
    }

    /**
     * @magentoDataFixture Vendor_Module::Test/Integration/_files/order_with_items.php
     * @magentoDbIsolation enabled
     */
    public function testProcessOrderWithItems(): void
    {
        $orderId = 100000002; // From fixture

        $result = $this->orderProcessor->process($orderId);

        $this->assertTrue($result);
        $order = $this->orderRepository->get($orderId);
        $this->assertCount(2, $order->getItems());
    }
}
```

### Creating Custom Fixtures

**File:** `Test/Integration/_files/order_with_items.php`

```php
<?php
declare(strict_types=1);

use Magento\TestFramework\Helper\Bootstrap;
use Magento\Sales\Model\Order;
use Magento\Sales\Model\Order\Item;
use Magento\Catalog\Model\Product;

$objectManager = Bootstrap::getObjectManager();

// Create product
$product = $objectManager->create(Product::class);
$product->setTypeId('simple')
    ->setId(1)
    ->setAttributeSetId(4)
    ->setName('Test Product')
    ->setSku('test-product-sku')
    ->setPrice(100.00)
    ->setVisibility(4)
    ->setStatus(1)
    ->setWebsiteIds([1]);
$product->save();

// Create order
$order = $objectManager->create(Order::class);
$order->setIncrementId('100000002')
    ->setStoreId(1)
    ->setState(Order::STATE_NEW)
    ->setStatus('pending')
    ->setCustomerIsGuest(false)
    ->setCustomerEmail('customer@example.com')
    ->setCustomerFirstname('John')
    ->setCustomerLastname('Doe')
    ->setGrandTotal(200.00)
    ->setBaseGrandTotal(200.00)
    ->setSubtotal(200.00)
    ->setBaseSubtotal(200.00);

// Add order items
$orderItem1 = $objectManager->create(Item::class);
$orderItem1->setProductId($product->getId())
    ->setSku($product->getSku())
    ->setName($product->getName())
    ->setPrice(100.00)
    ->setQtyOrdered(1);
$order->addItem($orderItem1);

$orderItem2 = $objectManager->create(Item::class);
$orderItem2->setProductId($product->getId())
    ->setSku($product->getSku())
    ->setName($product->getName())
    ->setPrice(100.00)
    ->setQtyOrdered(1);
$order->addItem($orderItem2);

$order->save();
```

**Rollback:** `Test/Integration/_files/order_with_items_rollback.php`

```php
<?php
declare(strict_types=1);

use Magento\TestFramework\Helper\Bootstrap;
use Magento\Sales\Model\Order;
use Magento\Framework\Registry;

$objectManager = Bootstrap::getObjectManager();
$registry = $objectManager->get(Registry::class);

// Prevent observer execution
$registry->unregister('isSecureArea');
$registry->register('isSecureArea', true);

// Delete order
$order = $objectManager->create(Order::class);
$order->loadByIncrementId('100000002');
if ($order->getId()) {
    $order->delete();
}

// Delete product
$product = $objectManager->create(\Magento\Catalog\Model\Product::class);
$product->load(1);
if ($product->getId()) {
    $product->delete();
}

$registry->unregister('isSecureArea');
```

### Testing Plugins (Interceptors)

**Plugin class:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin;

use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Sales\Api\Data\OrderInterface;

class OrderRepositoryExtend
{
    /**
     * Add custom attribute after loading order
     *
     * @param OrderRepositoryInterface $subject
     * @param OrderInterface $result
     * @return OrderInterface
     */
    public function afterGet(
        OrderRepositoryInterface $subject,
        OrderInterface $result
    ): OrderInterface {
        $extensionAttributes = $result->getExtensionAttributes();
        $extensionAttributes->setCustomAttribute('processed');
        $result->setExtensionAttributes($extensionAttributes);

        return $result;
    }
}
```

**Integration test:**

```php
/**
 * @magentoDataFixture Magento/Sales/_files/order.php
 */
public function testPluginAddsCustomAttribute(): void
{
    $order = $this->orderRepository->get(100000001);

    $extensionAttributes = $order->getExtensionAttributes();
    $this->assertEquals('processed', $extensionAttributes->getCustomAttribute());
}
```

### Testing Observers

**Observer:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Psr\Log\LoggerInterface;

class OrderPlaceAfter implements ObserverInterface
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {}

    public function execute(Observer $observer): void
    {
        $order = $observer->getEvent()->getOrder();
        $this->logger->info('Order placed', ['order_id' => $order->getId()]);
    }
}
```

**Integration test:**

```php
/**
 * @magentoDataFixture Magento/Sales/_files/order.php
 */
public function testObserverExecutesOnOrderPlace(): void
{
    $objectManager = Bootstrap::getObjectManager();
    $eventManager = $objectManager->get(\Magento\Framework\Event\ManagerInterface::class);
    $order = $this->orderRepository->get(100000001);

    // Dispatch event
    $eventManager->dispatch('sales_order_place_after', ['order' => $order]);

    // Verify log entry (requires custom log handler or DB logging)
    // For demonstration, test observer is registered
    $this->assertTrue(true);
}
```

---

## API Functional Testing

### REST API Tests

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Api;

use Magento\TestFramework\TestCase\WebapiAbstract;

class OrderManagementTest extends WebapiAbstract
{
    private const RESOURCE_PATH = '/V1/custom/orders';

    /**
     * @magentoApiDataFixture Magento/Sales/_files/order.php
     */
    public function testGetOrderViaApi(): void
    {
        $orderId = 100000001;

        $serviceInfo = [
            'rest' => [
                'resourcePath' => self::RESOURCE_PATH . '/' . $orderId,
                'httpMethod' => \Magento\Framework\Webapi\Rest\Request::HTTP_METHOD_GET,
            ],
        ];

        $response = $this->_webApiCall($serviceInfo);

        $this->assertArrayHasKey('entity_id', $response);
        $this->assertEquals($orderId, $response['entity_id']);
        $this->assertArrayHasKey('status', $response);
    }

    /**
     * @magentoApiDataFixture Magento/Customer/_files/customer.php
     */
    public function testCreateOrderViaApi(): void
    {
        $orderData = [
            'customer_email' => 'customer@example.com',
            'items' => [
                [
                    'sku' => 'simple-product',
                    'qty' => 1,
                    'price' => 100.00
                ]
            ],
            'billing_address' => [
                'firstname' => 'John',
                'lastname' => 'Doe',
                'street' => ['123 Main St'],
                'city' => 'Los Angeles',
                'region' => 'CA',
                'postcode' => '90001',
                'country_id' => 'US',
                'telephone' => '555-1234'
            ]
        ];

        $serviceInfo = [
            'rest' => [
                'resourcePath' => self::RESOURCE_PATH,
                'httpMethod' => \Magento\Framework\Webapi\Rest\Request::HTTP_METHOD_POST,
            ],
        ];

        $response = $this->_webApiCall($serviceInfo, $orderData);

        $this->assertArrayHasKey('entity_id', $response);
        $this->assertGreaterThan(0, $response['entity_id']);
    }

    public function testUnauthorizedAccessReturns401(): void
    {
        $this->expectException(\Exception::class);
        $this->expectExceptionCode(401);

        $serviceInfo = [
            'rest' => [
                'resourcePath' => self::RESOURCE_PATH . '/999',
                'httpMethod' => \Magento\Framework\Webapi\Rest\Request::HTTP_METHOD_GET,
                'token' => 'invalid_token'
            ],
        ];

        $this->_webApiCall($serviceInfo);
    }
}
```

### GraphQL API Tests

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\GraphQl;

use Magento\TestFramework\TestCase\GraphQlAbstract;

class OrderQueryTest extends GraphQlAbstract
{
    /**
     * @magentoApiDataFixture Magento/Sales/_files/order.php
     */
    public function testGetOrderViaGraphQl(): void
    {
        $orderId = 100000001;

        $query = <<<QUERY
{
  customOrder(id: $orderId) {
    id
    increment_id
    status
    grand_total
    items {
      sku
      name
      qty_ordered
    }
  }
}
QUERY;

        $response = $this->graphQlQuery($query);

        $this->assertArrayHasKey('customOrder', $response);
        $this->assertEquals($orderId, $response['customOrder']['id']);
        $this->assertIsArray($response['customOrder']['items']);
    }

    /**
     * @magentoApiDataFixture Magento/Customer/_files/customer.php
     */
    public function testCreateOrderMutation(): void
    {
        $mutation = <<<MUTATION
mutation {
  createCustomOrder(input: {
    customer_email: "customer@example.com"
    items: [
      {
        sku: "simple-product"
        qty: 1
      }
    ]
  }) {
    order {
      id
      increment_id
      status
    }
  }
}
MUTATION;

        $response = $this->graphQlMutation($mutation);

        $this->assertArrayHasKey('createCustomOrder', $response);
        $this->assertArrayHasKey('order', $response['createCustomOrder']);
        $this->assertGreaterThan(0, $response['createCustomOrder']['order']['id']);
    }
}
```

---

## MFTF (Magento Functional Testing Framework)

### When to Use MFTF

Use MFTF for:
- Critical user journeys (checkout, customer registration)
- Cross-module integration scenarios
- Admin panel workflows
- Smoke tests for deployment validation

**Not for:** Unit-level logic, API endpoints (use API functional tests)

### MFTF Test Structure

**File:** `Test/Mftf/Test/AdminProcessOrderTest.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<tests xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:noNamespaceSchemaLocation="urn:magento:mftf:Test/etc/testSchema.xsd">

    <test name="AdminProcessOrderTest">
        <annotations>
            <features value="Order Management"/>
            <stories value="Admin processes pending order"/>
            <title value="Admin can process order and update status"/>
            <description value="Verify admin can process order from pending to processing status"/>
            <severity value="CRITICAL"/>
            <testCaseId value="OM-123"/>
            <group value="order"/>
            <group value="admin"/>
        </annotations>

        <before>
            <!-- Create order fixture -->
            <createData entity="SimpleProduct" stepKey="createProduct"/>
            <createData entity="Simple_US_Customer" stepKey="createCustomer"/>
            <createData entity="CustomerCart" stepKey="createCustomerCart">
                <requiredEntity createDataKey="createCustomer"/>
            </createData>
            <createData entity="CustomerCartItem" stepKey="addCartItem">
                <requiredEntity createDataKey="createCustomerCart"/>
                <requiredEntity createDataKey="createProduct"/>
            </createData>
            <createData entity="CustomerAddressInformation" stepKey="addCustomerOrderAddress">
                <requiredEntity createDataKey="createCustomerCart"/>
            </createData>
            <updateData createDataKey="createCustomerCart" entity="CustomerOrderPaymentMethod" stepKey="sendCustomerPaymentInformation">
                <requiredEntity createDataKey="createCustomerCart"/>
            </updateData>

            <!-- Login as admin -->
            <actionGroup ref="AdminLoginActionGroup" stepKey="loginAsAdmin"/>
        </before>

        <after>
            <!-- Cleanup -->
            <deleteData createDataKey="createProduct" stepKey="deleteProduct"/>
            <deleteData createDataKey="createCustomer" stepKey="deleteCustomer"/>
            <actionGroup ref="AdminLogoutActionGroup" stepKey="logout"/>
        </after>

        <!-- Navigate to order -->
        <actionGroup ref="AdminOrdersPageOpenActionGroup" stepKey="goToOrdersPage"/>
        <actionGroup ref="AdminOrdersGridClearFiltersActionGroup" stepKey="clearFilters"/>
        <actionGroup ref="FilterOrderGridByIdActionGroup" stepKey="filterOrderById">
            <argument name="orderId" value="$createCustomerCart.return$"/>
        </actionGroup>
        <click selector="{{AdminOrdersGridSection.firstRow}}" stepKey="clickOrderRow"/>

        <!-- Process order -->
        <click selector="{{AdminOrderDetailsMainActionsSection.processOrder}}" stepKey="clickProcessOrder"/>
        <waitForPageLoad stepKey="waitForProcessing"/>

        <!-- Verify status updated -->
        <see selector="{{AdminOrderDetailsInformationSection.orderStatus}}" userInput="Processing" stepKey="seeProcessingStatus"/>
        <see selector="{{AdminMessagesSection.success}}" userInput="Order processed successfully" stepKey="seeSuccessMessage"/>
    </test>
</tests>
```

### Custom Action Groups

**File:** `Test/Mftf/ActionGroup/AdminProcessOrderActionGroup.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<actionGroups xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:noNamespaceSchemaLocation="urn:magento:mftf:Test/etc/actionGroupSchema.xsd">

    <actionGroup name="AdminProcessOrderActionGroup">
        <annotations>
            <description>Process order from order view page</description>
        </annotations>
        <arguments>
            <argument name="orderId" type="string"/>
        </arguments>

        <amOnPage url="{{AdminOrderPage.url(orderId)}}" stepKey="navigateToOrderPage"/>
        <waitForPageLoad stepKey="waitForOrderPage"/>
        <click selector="{{AdminOrderDetailsMainActionsSection.processOrder}}" stepKey="clickProcessButton"/>
        <waitForPageLoad stepKey="waitForProcessing"/>
        <see selector="{{AdminMessagesSection.success}}" userInput="Order processed successfully" stepKey="verifySuccess"/>
    </actionGroup>
</actionGroups>
```

### Running MFTF Tests

```bash
# Generate and run specific test
vendor/bin/mftf generate:tests
vendor/bin/mftf run:test AdminProcessOrderTest

# Run group
vendor/bin/mftf run:group order

# Run with specific browser
vendor/bin/mftf run:test AdminProcessOrderTest --browser chrome

# Parallel execution (requires Selenium Grid)
vendor/bin/mftf run:group order --parallel 4
```

---

## Test Coverage Requirements

### Coverage Targets

| Test Type | Minimum Coverage | Target Coverage |
|-----------|------------------|-----------------|
| Unit | 70% | 80%+ |
| Integration | 50% | 65%+ |
| MFTF | Critical paths | 100% of user journeys |

### Measuring Coverage

**PHPUnit with coverage:**

```bash
vendor/bin/phpunit -c dev/tests/unit/phpunit.xml.dist --coverage-html var/coverage/unit
```

**Integration test coverage:**

```bash
vendor/bin/phpunit -c dev/tests/integration/phpunit.xml.dist --coverage-html var/coverage/integration
```

**View coverage report:**

```bash
open var/coverage/unit/index.html
```

### Coverage Configuration (phpunit.xml)

```xml
<phpunit>
    <filter>
        <whitelist processUncoveredFilesFromWhitelist="true">
            <directory suffix=".php">../../../app/code/Vendor/Module</directory>
            <exclude>
                <directory>../../../app/code/Vendor/Module/Test</directory>
                <directory>../../../app/code/Vendor/Module/Setup</directory>
            </exclude>
        </whitelist>
    </filter>
</phpunit>
```

---

## CI/CD Integration

### GitHub Actions Workflow

**File:** `.github/workflows/tests.yml`

```yaml
name: Magento Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: bcmath, ctype, curl, dom, gd, intl, mbstring, pdo_mysql, simplexml, soap, xsl, zip
          coverage: xdebug

      - name: Install dependencies
        run: composer install --prefer-dist --no-progress

      - name: Run unit tests
        run: vendor/bin/phpunit -c dev/tests/unit/phpunit.xml.dist --coverage-clover coverage.xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml

  integration-tests:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: magento_integration_tests
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3

      elasticsearch:
        image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
        env:
          discovery.type: single-node
        ports:
          - 9200:9200

    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: bcmath, ctype, curl, dom, gd, intl, mbstring, pdo_mysql, simplexml, soap, xsl, zip

      - name: Install dependencies
        run: composer install --prefer-dist --no-progress

      - name: Setup Magento
        run: |
          php bin/magento setup:install \
            --db-host=127.0.0.1 \
            --db-name=magento_integration_tests \
            --db-user=root \
            --db-password=root \
            --backend-frontname=admin \
            --search-engine=elasticsearch7 \
            --elasticsearch-host=127.0.0.1 \
            --elasticsearch-port=9200

      - name: Run integration tests
        run: |
          cd dev/tests/integration
          ../../../vendor/bin/phpunit -c phpunit.xml.dist

  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'

      - name: Install dependencies
        run: composer install --prefer-dist --no-progress

      - name: Run PHPStan
        run: vendor/bin/phpstan analyse app/code/Vendor/Module --level 8

      - name: Run PHPCS
        run: vendor/bin/phpcs app/code/Vendor/Module --standard=Magento2
```

### GitLab CI Pipeline

**File:** `.gitlab-ci.yml`

```yaml
stages:
  - test
  - analyze

unit-tests:
  stage: test
  image: php:8.2-cli
  services:
    - mysql:8.0
  variables:
    MYSQL_ROOT_PASSWORD: root
    MYSQL_DATABASE: magento_tests
  script:
    - composer install
    - vendor/bin/phpunit -c dev/tests/unit/phpunit.xml.dist --coverage-text
  coverage: '/^\s*Lines:\s*\d+.\d+\%/'

integration-tests:
  stage: test
  image: php:8.2-cli
  services:
    - mysql:8.0
    - elasticsearch:7.17.0
  script:
    - composer install
    - bin/magento setup:install --db-host=mysql --db-name=magento_tests --db-user=root --db-password=root
    - cd dev/tests/integration && ../../../vendor/bin/phpunit

phpstan:
  stage: analyze
  image: php:8.2-cli
  script:
    - composer install
    - vendor/bin/phpstan analyse app/code/Vendor/Module --level 8
  allow_failure: false

phpcs:
  stage: analyze
  image: php:8.2-cli
  script:
    - composer install
    - vendor/bin/phpcs app/code/Vendor/Module --standard=Magento2
```

---

## Advanced Testing Patterns

### Testing Private Methods (via Reflection)

```php
public function testPrivateMethodLogic(): void
{
    $reflection = new \ReflectionClass(OrderProcessor::class);
    $method = $reflection->getMethod('calculateDiscount');
    $method->setAccessible(true);

    $orderProcessor = new OrderProcessor(/* dependencies */);
    $result = $method->invokeArgs($orderProcessor, [100.00, 0.15]);

    $this->assertEquals(15.00, $result);
}
```

**Note:** Prefer refactoring private methods to separate classes with public interfaces for better testability.

### Testing Event Dispatching

```php
public function testEventDispatched(): void
{
    $eventManagerMock = $this->createMock(\Magento\Framework\Event\ManagerInterface::class);
    $eventManagerMock->expects($this->once())
        ->method('dispatch')
        ->with(
            'vendor_module_order_processed',
            $this->callback(function ($data) {
                return isset($data['order']) && $data['order'] instanceof OrderInterface;
            })
        );

    $orderProcessor = new OrderProcessor($eventManagerMock, /* other dependencies */);
    $orderProcessor->process(123);
}
```

### Mutation Testing with Infection

```bash
composer require --dev infection/infection

vendor/bin/infection --threads=4 --min-msi=80
```

**Configuration:** `infection.json.dist`

```json
{
    "source": {
        "directories": [
            "app/code/Vendor/Module"
        ],
        "excludes": [
            "Test"
        ]
    },
    "logs": {
        "text": "var/infection.log"
    },
    "mutators": {
        "@default": true
    }
}
```

---

## Best Practices

### Test Naming Conventions

```php
// Pattern: test[MethodName][Scenario][ExpectedResult]
public function testProcessOrderWithValidDataReturnsTrue(): void
public function testProcessOrderWithInvalidOrderIdThrowsException(): void
public function testGetOrderStatusForPendingOrderReturnsPending(): void
```

### Arrange-Act-Assert Pattern

```php
public function testExample(): void
{
    // Arrange: Set up test data and mocks
    $orderId = 123;
    $this->orderRepositoryMock->method('get')->willReturn($orderMock);

    // Act: Execute the method under test
    $result = $this->orderProcessor->process($orderId);

    // Assert: Verify expected outcome
    $this->assertTrue($result);
}
```

### Test Isolation

```php
// BAD: Tests depend on execution order
public function testA(): void { self::$sharedState = 'value'; }
public function testB(): void { $this->assertEquals('value', self::$sharedState); }

// GOOD: Each test is independent
public function testA(): void { /* isolated */ }
public function testB(): void { /* isolated */ }
```

### Avoid Test Logic

```php
// BAD: Conditional logic in tests
if ($condition) {
    $this->assertTrue($result);
} else {
    $this->assertFalse($result);
}

// GOOD: Explicit assertions
$this->assertTrue($result);
```

---

## Assumptions

- **Magento version:** 2.4.7+ (PHPUnit 9.x, MFTF 3.x)
- **PHP version:** 8.2+ (typed properties, readonly)
- **Test frameworks:** PHPUnit 9.5+, MFTF 3.7+
- **CI/CD:** GitHub Actions or GitLab CI
- **Services:** MySQL 8.0, Elasticsearch 7.17

## Why This Approach

**Test pyramid distribution (60/30/10):**
- Unit tests are fast and catch logic errors early
- Integration tests verify framework interactions
- MFTF tests validate critical user paths without over-investment in slow E2E tests

**PHPUnit mocking over ObjectManager:**
- Unit tests run in <100ms vs 5-10s with ObjectManager
- True isolation prevents cascading failures
- Forces proper dependency injection design

**API functional tests for endpoints:**
- Tests service contracts as external consumers use them
- Validates serialization, authorization, error responses
- Catches breaking changes in API contracts

**MFTF for user journeys:**
- Cross-browser testing (Chrome, Firefox)
- Validates frontend-backend integration
- Smoke tests for deployment confidence

## Security Impact

- **Test data:** Use anonymized/synthetic PII in fixtures; never production data
- **API tests:** Validate authentication, authorization, rate limiting
- **MFTF:** Test CSRF token handling, XSS prevention in admin forms
- **Secrets:** Use environment variables for CI credentials; never commit to VCS
- **Test isolation:** Database rollback prevents data leakage between tests

## Performance Impact

- **Unit tests:** ~50ms per test; 1000 tests in <1 minute
- **Integration tests:** ~500ms per test; database overhead
- **MFTF:** ~30s per test; browser automation + page loads
- **CI runtime:** Unit (2min), Integration (10min), MFTF (30min)
- **Coverage reports:** Add 20-30% to test runtime (Xdebug overhead)

## Backward Compatibility

- **PHPUnit:** 9.x across Magento 2.4.x; migration to PHPUnit 10 in 2.5
- **MFTF:** 3.x stable; selector changes may break tests on Magento core updates
- **Test attributes:** PHP 8.2+ attributes preferred over annotations (future-proof)
- **Fixtures:** Backward compatible across 2.4.x if using entity builders
- **API tests:** Version endpoints (`/V1/`, `/V2/`) to maintain BC

## Tests to Add

**Unit tests:**
- All public methods in service classes
- Validation logic
- Exception handling paths
- Data transformation logic

**Integration tests:**
- Repository operations (CRUD)
- Plugin execution order
- Observer registration and execution
- DI configuration correctness

**API functional tests:**
- All REST/GraphQL endpoints
- Authentication/authorization flows
- Error responses (400, 401, 404, 500)
- Input validation

**MFTF tests:**
- Critical checkout flows
- Admin order management
- Customer account operations
- Payment gateway integration

## Documentation to Update

- **README:** Add test execution instructions (`composer test`, `composer test:unit`)
- **CONTRIBUTING:** Test requirements for PRs (70% unit coverage minimum)
- **CI/CD docs:** Pipeline configuration and troubleshooting
- **Test fixtures:** Document available fixtures and how to create custom ones
- **CHANGELOG:** Note breaking changes in test infrastructure
- **Screenshots:** CI badge, coverage report example

---

## Additional Resources

- [Magento DevDocs: Testing Guide](https://developer.adobe.com/commerce/testing/)
- [PHPUnit Documentation](https://phpunit.de/documentation.html)
- [MFTF Developer Guide](https://developer.adobe.com/commerce/testing/functional-testing-framework/)
- [Infection Mutation Testing](https://infection.github.io/)
- [GitHub Actions for PHP](https://github.com/shivammathur/setup-php)

## Related Documentation

### Related Guides

- [Docker Development Environment Setup for Magento 2](../tutorials/docker-development-environment.md)
- [CI/CD Deployment Pipelines for Magento 2](cicd-deployment.md)
- [Plugin System Deep Dive: Mastering Magento 2 Interception](../tutorials/plugin-system-deep-dive.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
- [Magento_Checkout Overview](../../modules/checkout/README.md)
