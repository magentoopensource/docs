---
title: "Magento_Sales Overview"
module: "Magento_Sales"
doc_type: "overview"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Sales Module

## Overview

The Magento_Sales module is the cornerstone of Adobe Commerce's order management system, providing comprehensive functionality for order lifecycle management, invoice generation, shipment processing, and credit memo handling. This module implements the complete sales workflow from quote conversion through order fulfillment and post-sales operations.

**Module Namespace:** `Magento\Sales`
**Area Coverage:** global, adminhtml, frontend, webapi_rest, webapi_soap, crontab
**Primary Dependencies:** Magento_Store, Magento_Customer, Magento_Catalog, Magento_Payment, Magento_Tax, Magento_Quote

## Core Responsibilities

### Order Management
The Sales module manages the complete order entity lifecycle:

- **Order Creation**: Converts quotes to orders with all associated data (items, addresses, payment, shipping)
- **Order State Machine**: Manages order states (new, processing, complete, closed, canceled, holded) and statuses (pending, pending_payment, processing, complete, closed, canceled, holded, payment_review, fraud)
- **Order Grid**: Provides admin interface for order search, filtering, and bulk operations
- **Order History**: Maintains comprehensive order history with comments, status changes, and notification tracking

### Invoice Management
Invoice generation and management capabilities:

- **Invoice Creation**: Generates invoices for full or partial order amounts
- **Invoice States**: Manages invoice lifecycle (open, paid, canceled)
- **Payment Capture**: Coordinates with payment gateways for fund capture
- **Invoice Numbering**: Maintains sequential invoice increment IDs per store

### Shipment Processing
Complete shipment lifecycle management:

- **Shipment Creation**: Generates shipments for full or partial order quantities
- **Tracking Information**: Stores carrier tracking numbers and provides customer notifications
- **Package Management**: Supports multi-package shipments with individual tracking
- **Shipment Comments**: Maintains shipment history and notes

### Credit Memo (Refund) Processing
Handles returns and refunds:

- **Credit Memo Creation**: Processes full or partial refunds with item-level control
- **Refund Methods**: Supports online refunds (payment gateway) and offline refunds
- **Inventory Restocking**: Coordinates with inventory module for stock restoration
- **Adjustment Fees**: Handles adjustment fees and shipping refunds

## Entity Hierarchy

```
Order (sales_order)
├── Order Items (sales_order_item)
│   └── Item Options (serialized configuration)
├── Order Addresses (sales_order_address)
│   └── Billing Address
│   └── Shipping Address
├── Order Payment (sales_order_payment)
│   └── Payment Transactions (sales_payment_transaction)
├── Order Status History (sales_order_status_history)
├── Invoices (sales_invoice)
│   ├── Invoice Items (sales_invoice_item)
│   └── Invoice Comments (sales_invoice_comment)
├── Shipments (sales_shipment)
│   ├── Shipment Items (sales_shipment_item)
│   ├── Shipment Tracks (sales_shipment_track)
│   └── Shipment Comments (sales_shipment_comment)
└── Credit Memos (sales_creditmemo)
    ├── Credit Memo Items (sales_creditmemo_item)
    └── Credit Memo Comments (sales_creditmemo_comment)
```

## Key Service Contracts

### Order Management Service Contracts

```php
<?php
namespace Magento\Sales\Api;

use Magento\Sales\Api\Data\OrderInterface;
use Magento\Framework\Exception\NoSuchEntityException;

/**
 * Order repository interface for CRUD operations
 */
interface OrderRepositoryInterface
{
    /**
     * Load order by entity ID
     *
     * @param int $id
     * @return OrderInterface
     * @throws NoSuchEntityException
     */
    public function get($id);

    /**
     * Find orders by search criteria
     *
     * @param \Magento\Framework\Api\SearchCriteriaInterface $searchCriteria
     * @return \Magento\Sales\Api\Data\OrderSearchResultInterface
     */
    public function getList(\Magento\Framework\Api\SearchCriteriaInterface $searchCriteria);

    /**
     * Save order
     *
     * @param OrderInterface $entity
     * @return OrderInterface
     */
    public function save(OrderInterface $entity);

    /**
     * Delete order
     *
     * @param OrderInterface $entity
     * @return bool
     */
    public function delete(OrderInterface $entity);
}
```

### Order Management Interface

```php
<?php
namespace Magento\Sales\Api;

/**
 * High-level order management operations
 */
interface OrderManagementInterface
{
    /**
     * Actual 2.4.7 signatures — no PHP return types, only @return docblocks
     */

    /** @return \Magento\Sales\Api\Data\OrderInterface */
    public function cancel($id);

    /** @return \Magento\Sales\Api\Data\OrderStatusHistorySearchResultInterface */
    public function getCommentsList($id);

    /** @return bool */
    public function addComment($id, \Magento\Sales\Api\Data\OrderStatusHistoryInterface $statusHistory);

    /** @return bool */
    public function notify($id);

    /** @return string */
    public function getStatus($id);

    /** @return bool */
    public function hold($id);

    /** @return bool */
    public function unHold($id);

    /** @return \Magento\Sales\Api\Data\OrderInterface */
    public function place(\Magento\Sales\Api\Data\OrderInterface $order);
}
```

### Invoice Service Contract

```php
<?php
namespace Magento\Sales\Api;

/**
 * Invoice management service
 */
interface InvoiceManagementInterface
{
    /**
     * Actual 2.4.7 signatures — no PHP return types
     */

    /** @return string */
    public function setCapture($id);

    /** @return \Magento\Sales\Api\Data\InvoiceCommentSearchResultInterface */
    public function getCommentsList($id);

    /** @return bool */
    public function notify($id);

    /** @return bool */
    public function setVoid($id);
}
```

### Shipment Management

```php
<?php
namespace Magento\Sales\Api;

/**
 * Shipment management service
 */
interface ShipmentManagementInterface
{
    /**
     * Actual 2.4.7 signatures — no PHP return types
     */

    /** @return string */
    public function getLabel($id);

    /** @return \Magento\Sales\Api\Data\ShipmentCommentSearchResultInterface */
    public function getCommentsList($id);

    /** @return bool */
    public function notify($id);
}
```

## Configuration Structure

### Module Configuration (etc/module.xml)

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Magento_Sales">
        <sequence>
            <module name="Magento_Rule"/>
            <module name="Magento_Catalog"/>
            <module name="Magento_Customer"/>
            <module name="Magento_Payment"/>
            <module name="Magento_SalesSequence"/>
        </sequence>
    </module>
</config>
```

### Admin Routes (etc/adminhtml/routes.xml)

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="admin">
        <route id="sales" frontName="sales">
            <module name="Magento_Sales"/>
        </route>
    </router>
</config>
```

### Web API Configuration (etc/webapi.xml)

```xml
<?xml version="1.0"?>
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Webapi:etc/webapi.xsd">

    <!-- Order Repository -->
    <route url="/V1/orders/:id" method="GET">
        <service class="Magento\Sales\Api\OrderRepositoryInterface" method="get"/>
        <resources>
            <resource ref="Magento_Sales::actions_view"/>
        </resources>
    </route>

    <route url="/V1/orders" method="GET">
        <service class="Magento\Sales\Api\OrderRepositoryInterface" method="getList"/>
        <resources>
            <resource ref="Magento_Sales::actions_view"/>
        </resources>
    </route>

    <!-- Order Management -->
    <route url="/V1/orders/:id/cancel" method="POST">
        <service class="Magento\Sales\Api\OrderManagementInterface" method="cancel"/>
        <resources>
            <resource ref="Magento_Sales::cancel"/>
        </resources>
    </route>

    <route url="/V1/orders/:id/hold" method="POST">
        <service class="Magento\Sales\Api\OrderManagementInterface" method="hold"/>
        <resources>
            <resource ref="Magento_Sales::hold"/>
        </resources>
    </route>

    <!-- Invoice Management -->
    <route url="/V1/invoices/:id/capture" method="POST">
        <service class="Magento\Sales\Api\InvoiceManagementInterface" method="setCapture"/>
        <resources>
            <resource ref="Magento_Sales::sales_invoice"/>
        </resources>
    </route>

    <!-- Shipment Management -->
    <route url="/V1/shipment/:id/label" method="GET">
        <service class="Magento\Sales\Api\ShipmentManagementInterface" method="getLabel"/>
        <resources>
            <resource ref="Magento_Sales::sales_shipment"/>
        </resources>
    </route>
</routes>
```

## Database Schema Overview

### Core Tables

**sales_order**: Main order entity with customer info, totals, state/status
- Primary columns: entity_id, increment_id, state, status, customer_id, grand_total, base_grand_total, store_id, created_at, updated_at

**sales_order_item**: Order line items with product information
- Primary columns: item_id, order_id, product_id, sku, name, qty_ordered, qty_invoiced, qty_shipped, qty_refunded, price, base_price, tax_amount, discount_amount

**sales_order_address**: Billing and shipping addresses
- Primary columns: entity_id, parent_id (order_id), address_type (billing/shipping), firstname, lastname, street, city, region, postcode, country_id

**sales_order_payment**: Payment information and transactions
- Primary columns: entity_id, parent_id (order_id), method, amount_ordered, amount_paid, base_amount_ordered, base_amount_paid

**sales_invoice**: Invoice records linked to orders
- Primary columns: entity_id, order_id, increment_id, state, grand_total, base_grand_total, created_at

**sales_shipment**: Shipment records with tracking
- Primary columns: entity_id, order_id, increment_id, total_qty, created_at

**sales_creditmemo**: Credit memo (refund) records
- Primary columns: entity_id, order_id, increment_id, state, grand_total, base_grand_total, adjustment_positive, adjustment_negative

**sales_order_status**: Order status definitions
- Primary columns: status, label

**sales_order_status_state**: Maps statuses to states
- Primary columns: status, state, is_default, visible_on_front

## Extension Points

### Plugins (Interceptors)

Common plugin targets for order customization:

```php
<?php
namespace Vendor\Module\Plugin\Sales\Model\Service;

use Magento\Sales\Api\Data\OrderInterface;
use Magento\Sales\Model\Service\OrderService;

/**
 * Plugin for order placement customization
 */
class OrderServiceExtend
{
    /**
     * Before order placement - validate custom business rules
     *
     * @param OrderService $subject
     * @param OrderInterface $order
     * @return array
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function beforePlace(
        OrderService $subject,
        OrderInterface $order
    ): array {
        // Custom validation logic
        if (!$this->validateCustomRules($order)) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Order does not meet custom business rules.')
            );
        }

        return [$order];
    }

    /**
     * After order placement - execute post-order logic
     *
     * @param OrderService $subject
     * @param OrderInterface $result
     * @return OrderInterface
     */
    public function afterPlace(
        OrderService $subject,
        OrderInterface $result
    ): OrderInterface {
        // Execute custom post-order logic
        $this->executeCustomPostOrderLogic($result);

        return $result;
    }

    private function validateCustomRules(OrderInterface $order): bool
    {
        // Implementation
        return true;
    }

    private function executeCustomPostOrderLogic(OrderInterface $order): void
    {
        // Implementation
    }
}
```

### Event Observers

Critical sales events for customization:

```php
<?php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Model\Order;

/**
 * Observer for sales_order_place_after event
 */
class OrderPlaceAfterObserver implements ObserverInterface
{
    /**
     * Execute after order is placed
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var Order $order */
        $order = $observer->getEvent()->getOrder();

        // Custom post-order processing
        $this->processOrderPlacement($order);
    }

    private function processOrderPlacement(Order $order): void
    {
        // Custom implementation
        // - Send to ERP system
        // - Trigger warehouse notification
        // - Update external inventory
        // - Create shipping labels
    }
}
```

Key sales events:
- `sales_order_place_before`: Before order is placed
- `sales_order_place_after`: After order is placed
- `sales_order_save_before`: Before order save
- `sales_order_save_after`: After order save
- `sales_order_invoice_save_after`: After invoice save
- `sales_order_shipment_save_after`: After shipment save
- `sales_order_creditmemo_save_after`: After credit memo save
- `sales_order_payment_place_start`: Before payment is placed
- `sales_order_payment_place_end`: After payment is placed

## Command-Line Tools

### Order Management Commands

> **Note:** Magento Open Source does not include CLI commands for viewing,
> invoicing, shipping, or canceling individual orders. These operations are
> performed through the Admin Panel (Sales > Orders) or via the REST/GraphQL API.

```bash
# Reindex order grid
bin/magento indexer:reindex sales_order_grid

# Example: View order via REST API
# curl -X GET https://your-store.test/rest/V1/orders/123 -H "Authorization: Bearer {token}"

# Example: Cancel order via REST API
# curl -X POST https://your-store.test/rest/V1/orders/123/cancel -H "Authorization: Bearer {token}"
```

## Performance Considerations

### Order Grid Indexing
The order grid uses a flat table (`sales_order_grid`) for performance:

```bash
# Reindex order grid
bin/magento indexer:reindex sales_order_grid

# Enable real-time grid updates
bin/magento config:set dev/grid/async_indexing 0
```

### Query Optimization
Use service contracts with search criteria for efficient queries:

```php
<?php
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Sales\Api\OrderRepositoryInterface;

class OrderService
{
    public function __construct(
        private OrderRepositoryInterface $orderRepository,
        private SearchCriteriaBuilder $searchCriteriaBuilder
    ) {}

    /**
     * Get orders efficiently with filters
     */
    public function getRecentOrders(int $customerId, int $limit = 10): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('customer_id', $customerId, 'eq')
            ->addFilter('created_at', date('Y-m-d', strtotime('-30 days')), 'gteq')
            ->setPageSize($limit)
            ->addSortOrder('created_at', 'DESC')
            ->create();

        return $this->orderRepository->getList($searchCriteria)->getItems();
    }
}
```

## Security Considerations

### ACL (Access Control Lists)
Always protect admin operations with ACL:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Acl/etc/acl.xsd">
    <acl>
        <resources>
            <resource id="Magento_Backend::admin">
                <resource id="Magento_Sales::sales" title="Sales" sortOrder="10">
                    <resource id="Magento_Sales::sales_operation" title="Operations">
                        <resource id="Magento_Sales::sales_order" title="Orders"/>
                        <resource id="Magento_Sales::create" title="Create"/>
                        <resource id="Magento_Sales::actions" title="Actions">
                            <resource id="Magento_Sales::cancel" title="Cancel"/>
                            <resource id="Magento_Sales::hold" title="Hold"/>
                            <resource id="Magento_Sales::unhold" title="Unhold"/>
                        </resource>
                    </resource>
                </resource>
            </resource>
        </resources>
    </acl>
</config>
```

### PII Data Handling
Orders contain sensitive customer data. Always:
- Use secure connections for API access
- Implement proper authentication and authorization
- Mask sensitive data in logs
- Follow GDPR/CCPA requirements for data export and deletion

## Module Dependencies

**Required Dependencies:**
- Magento_Store: Store context and multi-store support
- Magento_Customer: Customer entity and authentication
- Magento_Catalog: Product information for order items
- Magento_Quote: Quote to order conversion
- Magento_Payment: Payment processing
- Magento_Tax: Tax calculation for orders

**Optional Dependencies (via soft dependencies):**
- Magento_Shipping: Shipping methods and rates
- Magento_Inventory: Inventory management integration
- Magento_GiftMessage: Gift message functionality
- Magento_Bundle: Bundle product order items
- Magento_Downloadable: Downloadable product links

## Testing Strategies

### Unit Testing Example

```php
<?php
namespace Vendor\Module\Test\Unit\Model;

use PHPUnit\Framework\TestCase;
use Magento\Sales\Model\Order;
use Vendor\Module\Model\OrderValidator;

class OrderValidatorTest extends TestCase
{
    private OrderValidator $validator;

    protected function setUp(): void
    {
        $this->validator = new OrderValidator();
    }

    public function testValidateOrderAmount(): void
    {
        $order = $this->createMock(Order::class);
        $order->method('getGrandTotal')->willReturn(100.00);

        $result = $this->validator->validateMinimumAmount($order, 50.00);

        $this->assertTrue($result);
    }

    public function testInvalidOrderAmount(): void
    {
        $order = $this->createMock(Order::class);
        $order->method('getGrandTotal')->willReturn(25.00);

        $result = $this->validator->validateMinimumAmount($order, 50.00);

        $this->assertFalse($result);
    }
}
```

### Integration Testing

```php
<?php
namespace Vendor\Module\Test\Integration\Model;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;
use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Sales\Model\Order;

class OrderRepositoryTest extends TestCase
{
    private OrderRepositoryInterface $orderRepository;

    protected function setUp(): void
    {
        $this->orderRepository = Bootstrap::getObjectManager()
            ->get(OrderRepositoryInterface::class);
    }

    /**
     * @magentoDataFixture Magento/Sales/_files/order.php
     */
    public function testGetOrder(): void
    {
        $order = $this->orderRepository->get(1);

        $this->assertInstanceOf(Order::class, $order);
        $this->assertEquals(Order::STATE_PROCESSING, $order->getState());
    }
}
```

## Common Development Patterns

### Custom Order Status

```php
<?php
namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Sales\Model\Order\StatusFactory;
use Magento\Sales\Model\ResourceModel\Order\Status as StatusResource;

class AddCustomOrderStatus implements DataPatchInterface
{
    public function __construct(
        private StatusFactory $statusFactory,
        private StatusResource $statusResource
    ) {}

    public function apply()
    {
        $status = $this->statusFactory->create();
        $status->setData([
            'status' => 'custom_pending_approval',
            'label' => 'Pending Approval'
        ]);

        $this->statusResource->save($status);

        $status->assignState(
            \Magento\Sales\Model\Order::STATE_NEW,
            false,
            true
        );

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

## Resources and Documentation

- **Official DevDocs**: https://developer.adobe.com/commerce/php/module-reference/module-sales/
- **Service Contracts**: Review `Magento\Sales\Api` namespace interfaces
- **Database Schema**: Review `Setup/InstallSchema.php` and data patches
- **Admin UI**: Located in `view/adminhtml/` directory
- **REST API**: `/rest/V1/orders`, `/rest/V1/invoices`, `/rest/V1/shipments`

## Module File Structure

```
Magento/Sales/
├── Api/                          # Service contract interfaces
│   ├── Data/                     # Data transfer objects
│   ├── OrderRepositoryInterface.php
│   ├── InvoiceRepositoryInterface.php
│   ├── ShipmentRepositoryInterface.php
│   └── CreditmemoRepositoryInterface.php
├── Block/                        # View blocks
│   ├── Adminhtml/                # Admin blocks
│   └── Order/                    # Frontend order blocks
├── Console/                      # CLI commands
├── Controller/                   # HTTP controllers
│   ├── Adminhtml/                # Admin controllers
│   └── Order/                    # Frontend order controllers
├── Cron/                         # Cron jobs
├── etc/                          # Configuration files
│   ├── adminhtml/                # Admin-specific config
│   ├── frontend/                 # Frontend-specific config
│   ├── di.xml                    # Dependency injection
│   ├── module.xml                # Module declaration
│   ├── webapi.xml                # Web API routes
│   └── events.xml                # Event observers
├── Helper/                       # Helper classes
├── Model/                        # Business logic models
│   ├── Order/                    # Order-related models
│   ├── ResourceModel/            # Database layer
│   └── Service/                  # Service implementations
├── Observer/                     # Event observers
├── Plugin/                       # Plugins (interceptors)
├── Setup/                        # Installation/upgrade scripts
│   └── Patch/                    # Declarative patches
├── Test/                         # Unit and integration tests
├── Ui/                           # UI components (grids, forms)
└── view/                         # View layer
    ├── adminhtml/                # Admin templates/layouts
    │   ├── layout/
    │   ├── templates/
    │   └── ui_component/         # Admin grids
    └── frontend/                 # Frontend templates/layouts
        ├── layout/
        └── templates/
```

---

**Last Updated**: 2026-02-05
**Target Version**: Adobe Commerce 2.4.7+
**Maintenance**: This module is actively maintained by Adobe Commerce Core Team

## Related Guides

- [B2B Features Development](../../guides/explanations/b2b-features.md)
- [Indexer System Deep Dive: Understanding Magento 2's Data Indexing Architecture](../../guides/explanations/indexer-system.md)
- [Message Queue Architecture in Magento 2](../../guides/explanations/message-queue-architecture.md)
- [Service Contracts vs Repositories in Magento 2](../../guides/explanations/service-contracts-repositories.md)
- [Building Admin UI Components in Magento 2](../../guides/how-to/admin-ui-components.md)
- [Cron Jobs Implementation: Building Reliable Scheduled Tasks in Magento 2](../../guides/how-to/cron-jobs.md)
- [Email Templates Customization](../../guides/how-to/email-templates.md)
- [ERP Integration Patterns for Magento 2](../../guides/how-to/erp-integration.md)
- [Comprehensive Testing Strategies for Magento 2](../../guides/how-to/testing-strategies.md)
- [CLI Command Reference: Complete bin/magento Guide](../../guides/references/cli-command-reference.md)
- [Upgrade Guide: Adobe Commerce 2.4.6 to 2.4.7/2.4.8](../../guides/references/upgrade-guide-247-248.md)
- [Custom Payment Method Development: Building Secure Payment Integrations](../../guides/tutorials/custom-payment-method.md)
- [Custom Shipping Method Development: Building Flexible Shipping Solutions](../../guides/tutorials/custom-shipping-method.md)
- [Declarative Schema & Data Patches: Modern Database Management in Magento 2](../../guides/tutorials/declarative-schema-data-patches.md)
- [Plugin System Deep Dive: Mastering Magento 2 Interception](../../guides/tutorials/plugin-system-deep-dive.md)
