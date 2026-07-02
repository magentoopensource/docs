---
title: "ERP Integration Patterns for Magento 2"
description: "Developer guide: ERP Integration Patterns for Magento 2"
type: "how-to"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "35 minutes"
topics:
  - erp integration
  - webhooks
  - api design
  - data synchronization
  - idempotency
  - error handling
last_updated: "2026-02-07"
---

# ERP Integration Patterns for Magento 2

## Overview

ERP integrations are critical for enterprise Magento deployments. This guide provides production-proven patterns for bidirectional data synchronization, covering webhooks vs polling strategies, idempotency keys for reliability, bulk API operations, order/inventory sync patterns, and robust error handling.

**What you'll learn:**
- Webhook vs polling strategy trade-offs
- Idempotency keys for reliable operations
- Bulk API patterns for large datasets
- Order synchronization (Magento → ERP)
- Inventory synchronization (ERP → Magento)
- Error handling and retry logic
- Audit logging and data reconciliation

**Prerequisites:**
- Strong PHP knowledge (interfaces, async operations)
- REST/GraphQL API design principles
- Understanding of Magento service contracts
- Basic message queue knowledge
- Database transaction concepts

---

## Integration Architecture Overview

### Typical ERP Integration Flow

```
┌─────────────┐           ┌─────────────┐           ┌─────────────┐
│   Magento   │◄─────────►│  Middleware │◄─────────►│     ERP     │
│             │  Webhook  │   (Queue)   │   API     │  (SAP/etc)  │
└─────────────┘           └─────────────┘           └─────────────┘
      │                          │                          │
      ├─ Orders (outbound)       ├─ Transform/Map          ├─ Products
      ├─ Customers               ├─ Retry logic            ├─ Inventory
      └─ Inventory (inbound)     └─ Audit logs             └─ Prices
```

### Key Integration Points

1. **Order Export (Magento → ERP):** Order placed → webhook → transform → ERP order creation
2. **Inventory Import (ERP → Magento):** ERP stock update → API call → Magento stock update
3. **Product Sync (ERP → Magento):** ERP product data → bulk API → Magento catalog
4. **Price Sync (ERP → Magento):** ERP pricing → scheduled import → Magento price updates
5. **Customer Sync (Bidirectional):** Customer data consistency across systems

---

## Webhook vs Polling Strategies

### Webhook Strategy (Event-Driven)

**Architecture:**

```
[Magento Event] → [Observer] → [Webhook Publisher] → [HTTP POST] → [ERP Endpoint]
                                                                        ↓
                                                                  [200 OK / Retry]
```

**Advantages:**
- Real-time data propagation (<1s latency)
- No polling overhead on ERP
- Scales efficiently (push on change)
- Lower API rate limit consumption

**Disadvantages:**
- Requires public endpoint (firewall rules, security)
- Retry logic complexity (exponential backoff)
- Potential data loss if webhook fails permanently
- ERP must handle burst traffic (order spikes)

**Implementation:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Webhook;

use Magento\Framework\HTTP\Client\Curl;
use Magento\Framework\Serialize\Serializer\Json;
use Psr\Log\LoggerInterface;
use Vendor\Module\Api\Data\WebhookPayloadInterface;
use Vendor\Module\Model\Config;

class Publisher
{
    private const MAX_RETRY_ATTEMPTS = 3;
    private const RETRY_DELAY_MS = [1000, 2000, 4000]; // Exponential backoff

    public function __construct(
        private readonly Curl $curl,
        private readonly Json $json,
        private readonly Config $config,
        private readonly LoggerInterface $logger,
        private readonly AuditLogger $auditLogger
    ) {}

    /**
     * Publish webhook to ERP endpoint
     *
     * @param WebhookPayloadInterface $payload
     * @return bool
     * @throws \Exception
     */
    public function publish(WebhookPayloadInterface $payload): bool
    {
        $endpoint = $this->config->getWebhookEndpoint();
        $secret = $this->config->getWebhookSecret();

        if (!$endpoint) {
            throw new \RuntimeException('Webhook endpoint not configured');
        }

        $body = $this->json->serialize($payload->getData());
        $signature = $this->generateSignature($body, $secret);

        $headers = [
            'Content-Type' => 'application/json',
            'X-Magento-Signature' => $signature,
            'X-Magento-Event' => $payload->getEventType(),
            'X-Magento-Idempotency-Key' => $payload->getIdempotencyKey(),
            'X-Magento-Timestamp' => time()
        ];

        for ($attempt = 1; $attempt <= self::MAX_RETRY_ATTEMPTS; $attempt++) {
            try {
                $this->curl->setHeaders($headers);
                $this->curl->setTimeout(30);
                $this->curl->post($endpoint, $body);

                $statusCode = $this->curl->getStatus();
                $response = $this->curl->getBody();

                // Log attempt
                $this->auditLogger->log([
                    'event_type' => $payload->getEventType(),
                    'idempotency_key' => $payload->getIdempotencyKey(),
                    'attempt' => $attempt,
                    'status_code' => $statusCode,
                    'endpoint' => $endpoint
                ]);

                // Success: 2xx status codes
                if ($statusCode >= 200 && $statusCode < 300) {
                    $this->logger->info('Webhook published successfully', [
                        'event_type' => $payload->getEventType(),
                        'idempotency_key' => $payload->getIdempotencyKey(),
                        'status_code' => $statusCode
                    ]);
                    return true;
                }

                // Permanent failure: 4xx errors (don't retry)
                if ($statusCode >= 400 && $statusCode < 500) {
                    $this->logger->error('Webhook rejected by ERP (4xx)', [
                        'event_type' => $payload->getEventType(),
                        'status_code' => $statusCode,
                        'response' => $response
                    ]);
                    throw new \RuntimeException("Webhook rejected: HTTP $statusCode");
                }

                // Transient failure: 5xx errors (retry)
                if ($attempt < self::MAX_RETRY_ATTEMPTS) {
                    $delay = self::RETRY_DELAY_MS[$attempt - 1];
                    $this->logger->warning('Webhook failed, retrying', [
                        'attempt' => $attempt,
                        'status_code' => $statusCode,
                        'retry_delay_ms' => $delay
                    ]);
                    usleep($delay * 1000); // Convert to microseconds
                    continue;
                }

                // Max retries exceeded
                throw new \RuntimeException("Webhook failed after $attempt attempts: HTTP $statusCode");

            } catch (\Exception $e) {
                if ($attempt >= self::MAX_RETRY_ATTEMPTS) {
                    $this->logger->critical('Webhook failed permanently', [
                        'event_type' => $payload->getEventType(),
                        'error' => $e->getMessage()
                    ]);
                    throw $e;
                }
            }
        }

        return false;
    }

    /**
     * Generate HMAC signature for webhook payload
     *
     * @param string $payload
     * @param string $secret
     * @return string
     */
    private function generateSignature(string $payload, string $secret): string
    {
        return hash_hmac('sha256', $payload, $secret);
    }
}
```

**Observer to trigger webhook:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Magento\Sales\Api\Data\OrderInterface;
use Vendor\Module\Model\Webhook\Publisher;
use Vendor\Module\Model\Webhook\PayloadFactory;
use Vendor\Module\Model\Queue\WebhookQueuePublisher;

class OrderPlaceAfter implements ObserverInterface
{
    public function __construct(
        private readonly PayloadFactory $payloadFactory,
        private readonly WebhookQueuePublisher $queuePublisher
    ) {}

    /**
     * Publish order to ERP via webhook (async)
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        /** @var OrderInterface $order */
        $order = $observer->getEvent()->getOrder();

        // Create webhook payload
        $payload = $this->payloadFactory->create();
        $payload->setEventType('order.placed');
        $payload->setIdempotencyKey($this->generateIdempotencyKey($order));
        $payload->setData([
            'order_id' => $order->getEntityId(),
            'increment_id' => $order->getIncrementId(),
            'customer_email' => $order->getCustomerEmail(),
            'grand_total' => $order->getGrandTotal(),
            'status' => $order->getStatus(),
            'items' => $this->formatOrderItems($order),
            'billing_address' => $this->formatAddress($order->getBillingAddress()),
            'shipping_address' => $this->formatAddress($order->getShippingAddress())
        ]);

        // Publish to message queue for async processing
        $this->queuePublisher->publish($payload);
    }

    /**
     * Generate idempotency key for order
     *
     * @param OrderInterface $order
     * @return string
     */
    private function generateIdempotencyKey(OrderInterface $order): string
    {
        return sprintf('order_%s_%s', $order->getIncrementId(), $order->getCreatedAt());
    }

    /**
     * Format order items for webhook payload
     *
     * @param OrderInterface $order
     * @return array
     */
    private function formatOrderItems(OrderInterface $order): array
    {
        $items = [];
        foreach ($order->getItems() as $item) {
            $items[] = [
                'sku' => $item->getSku(),
                'name' => $item->getName(),
                'qty' => $item->getQtyOrdered(),
                'price' => $item->getPrice(),
                'row_total' => $item->getRowTotal()
            ];
        }
        return $items;
    }

    /**
     * Format address for webhook payload
     *
     * @param \Magento\Sales\Api\Data\OrderAddressInterface|null $address
     * @return array|null
     */
    private function formatAddress($address): ?array
    {
        if (!$address) {
            return null;
        }

        return [
            'firstname' => $address->getFirstname(),
            'lastname' => $address->getLastname(),
            'street' => $address->getStreet(),
            'city' => $address->getCity(),
            'region' => $address->getRegion(),
            'postcode' => $address->getPostcode(),
            'country_id' => $address->getCountryId(),
            'telephone' => $address->getTelephone()
        ];
    }
}
```

### Polling Strategy (Request-Response)

**Architecture:**

```
[Cron Job] → [Check Last Sync Time] → [Query Magento DB] → [Transform] → [POST to ERP]
                    ↓
            [Update Sync Timestamp]
```

**Advantages:**
- Simpler security model (outbound only)
- ERP controls request rate (no burst handling)
- Easy to pause/resume (disable cron)
- Explicit data boundaries (batch size control)

**Disadvantages:**
- Higher latency (depends on cron frequency)
- Resource overhead (polling empty results)
- API rate limit consumption
- Potential duplicate processing

**Implementation:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Cron;

use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Vendor\Module\Model\Erp\OrderExporter;
use Vendor\Module\Model\SyncStateManager;
use Psr\Log\LoggerInterface;

class ExportOrdersToErp
{
    private const BATCH_SIZE = 100;
    private const SYNC_STATE_KEY = 'order_export_last_sync';

    public function __construct(
        private readonly OrderRepositoryInterface $orderRepository,
        private readonly SearchCriteriaBuilder $searchCriteriaBuilder,
        private readonly OrderExporter $orderExporter,
        private readonly SyncStateManager $syncStateManager,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Export orders to ERP (runs every 5 minutes)
     *
     * @return void
     */
    public function execute(): void
    {
        $lastSyncTime = $this->syncStateManager->getLastSyncTime(self::SYNC_STATE_KEY);
        $this->logger->info('Starting order export to ERP', [
            'last_sync_time' => $lastSyncTime
        ]);

        try {
            // Query orders created/updated since last sync
            $searchCriteria = $this->searchCriteriaBuilder
                ->addFilter('created_at', $lastSyncTime, 'gt')
                ->addFilter('status', ['pending', 'processing'], 'in')
                ->setPageSize(self::BATCH_SIZE)
                ->create();

            $orderList = $this->orderRepository->getList($searchCriteria);
            $exportedCount = 0;

            foreach ($orderList->getItems() as $order) {
                try {
                    $this->orderExporter->export($order);
                    $exportedCount++;
                } catch (\Exception $e) {
                    $this->logger->error('Failed to export order', [
                        'order_id' => $order->getEntityId(),
                        'error' => $e->getMessage()
                    ]);
                    // Continue with next order (don't fail entire batch)
                }
            }

            // Update sync timestamp
            $this->syncStateManager->setLastSyncTime(self::SYNC_STATE_KEY, date('Y-m-d H:i:s'));

            $this->logger->info('Order export completed', [
                'exported_count' => $exportedCount,
                'total_orders' => $orderList->getTotalCount()
            ]);

        } catch (\Exception $e) {
            $this->logger->critical('Order export failed', [
                'error' => $e->getMessage()
            ]);
        }
    }
}
```

**Cron configuration (crontab.xml):**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/crontab.xsd">
    <group id="default">
        <job name="vendor_module_export_orders" instance="Vendor\Module\Cron\ExportOrdersToErp" method="execute">
            <schedule>*/5 * * * *</schedule> <!-- Every 5 minutes -->
        </job>
    </group>
</config>
```

### Strategy Comparison

| Criterion | Webhook | Polling |
|-----------|---------|---------|
| **Latency** | <1s (real-time) | 1-5 min (cron interval) |
| **Complexity** | High (retry, security) | Low (simple cron) |
| **Resource usage** | Low (event-driven) | Medium (periodic queries) |
| **Data loss risk** | Medium (retry failures) | Low (DB-backed) |
| **Scalability** | High (async queues) | Medium (cron concurrency) |
| **Best for** | Real-time integrations | Batch processing |

**Recommendation:** Use **webhooks** for time-sensitive data (orders, inventory alerts); use **polling** for bulk data (product catalog, historical data).

---

## Idempotency Keys for Reliability

### Why Idempotency Matters

**Problem:** Network failures, retries, and duplicate webhook deliveries can cause duplicate order creation in ERP.

**Solution:** Idempotency keys ensure repeated requests with the same key produce the same result without side effects.

### Implementation: Idempotency Key Generation

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Sales\Api\Data\OrderInterface;

class IdempotencyKeyGenerator
{
    /**
     * Generate idempotency key for order
     *
     * Format: order_{increment_id}_{created_at_timestamp}
     *
     * @param OrderInterface $order
     * @return string
     */
    public function generateForOrder(OrderInterface $order): string
    {
        $createdAt = strtotime($order->getCreatedAt());
        return sprintf('order_%s_%d', $order->getIncrementId(), $createdAt);
    }

    /**
     * Generate idempotency key for inventory update
     *
     * Format: inventory_{sku}_{timestamp}
     *
     * @param string $sku
     * @param int $timestamp
     * @return string
     */
    public function generateForInventory(string $sku, int $timestamp): string
    {
        return sprintf('inventory_%s_%d', $sku, $timestamp);
    }

    /**
     * Generate idempotency key for custom operation
     *
     * @param string $entityType
     * @param string $entityId
     * @param string $operation
     * @return string
     */
    public function generate(string $entityType, string $entityId, string $operation): string
    {
        return sprintf('%s_%s_%s_%d', $entityType, $entityId, $operation, time());
    }
}
```

### Idempotency Registry (Deduplication)

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\App\CacheInterface;

class IdempotencyRegistry
{
    private const CACHE_PREFIX = 'idempotency_';
    private const CACHE_LIFETIME = 86400; // 24 hours

    public function __construct(
        private readonly CacheInterface $cache
    ) {}

    /**
     * Check if idempotency key has been processed
     *
     * @param string $key
     * @return bool
     */
    public function exists(string $key): bool
    {
        return $this->cache->load(self::CACHE_PREFIX . $key) !== false;
    }

    /**
     * Register idempotency key as processed
     *
     * @param string $key
     * @param mixed $result
     * @return void
     */
    public function register(string $key, $result = null): void
    {
        $this->cache->save(
            json_encode(['result' => $result, 'processed_at' => time()]),
            self::CACHE_PREFIX . $key,
            [],
            self::CACHE_LIFETIME
        );
    }

    /**
     * Get cached result for idempotency key
     *
     * @param string $key
     * @return mixed|null
     */
    public function getResult(string $key)
    {
        $cached = $this->cache->load(self::CACHE_PREFIX . $key);
        if ($cached === false) {
            return null;
        }

        $data = json_decode($cached, true);
        return $data['result'] ?? null;
    }
}
```

### Using Idempotency in Webhook Handler

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Webhook;

use Vendor\Module\Model\IdempotencyRegistry;
use Vendor\Module\Model\Erp\OrderExporter;
use Psr\Log\LoggerInterface;

class OrderWebhookHandler
{
    public function __construct(
        private readonly IdempotencyRegistry $idempotencyRegistry,
        private readonly OrderExporter $orderExporter,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Handle order webhook with idempotency
     *
     * @param array $payload
     * @return array
     */
    public function handle(array $payload): array
    {
        $idempotencyKey = $payload['idempotency_key'] ?? null;

        if (!$idempotencyKey) {
            throw new \InvalidArgumentException('Missing idempotency key');
        }

        // Check if already processed
        if ($this->idempotencyRegistry->exists($idempotencyKey)) {
            $this->logger->info('Duplicate webhook detected, returning cached result', [
                'idempotency_key' => $idempotencyKey
            ]);

            return [
                'success' => true,
                'message' => 'Already processed (idempotent)',
                'result' => $this->idempotencyRegistry->getResult($idempotencyKey)
            ];
        }

        // Process webhook
        try {
            $result = $this->orderExporter->export($payload);

            // Register as processed
            $this->idempotencyRegistry->register($idempotencyKey, $result);

            return [
                'success' => true,
                'result' => $result
            ];
        } catch (\Exception $e) {
            $this->logger->error('Webhook processing failed', [
                'idempotency_key' => $idempotencyKey,
                'error' => $e->getMessage()
            ]);

            throw $e;
        }
    }
}
```

---

## Bulk API for Large Datasets

### When to Use Bulk API

- Product catalog imports (1000+ products)
- Price updates across catalog
- Inventory synchronization (multi-warehouse)
- Customer data migrations
- Historical order imports

### Bulk API Architecture

```
[ERP Data] → [Chunk into batches] → [Magento Bulk API] → [Async Queue] → [Process]
                                            ↓
                                    [Bulk Operation Status API]
```

### Implementation: Bulk Product Import

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Erp;

use Magento\AsynchronousOperations\Api\Data\OperationInterface;
use Magento\Framework\Bulk\BulkManagementInterface;
use Magento\Framework\DataObject\IdentityGeneratorInterface;
use Magento\Framework\Serialize\Serializer\Json;
use Magento\Authorization\Model\UserContextInterface;
use Psr\Log\LoggerInterface;

class BulkProductImporter
{
    private const BATCH_SIZE = 100;
    private const TOPIC_NAME = 'vendor.module.product.import';

    public function __construct(
        private readonly BulkManagementInterface $bulkManagement,
        private readonly IdentityGeneratorInterface $identityGenerator,
        private readonly Json $json,
        private readonly UserContextInterface $userContext,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Import products in bulk
     *
     * @param array $products
     * @return string Bulk UUID
     * @throws \Exception
     */
    public function import(array $products): string
    {
        $bulkUuid = $this->identityGenerator->generateId();
        $bulkDescription = sprintf('Import %d products from ERP', count($products));
        $userId = $this->userContext->getUserId();

        // Chunk products into batches
        $batches = array_chunk($products, self::BATCH_SIZE);
        $operations = [];

        foreach ($batches as $batchIndex => $batch) {
            $operations[] = [
                'data' => [
                    'bulk_uuid' => $bulkUuid,
                    'topic_name' => self::TOPIC_NAME,
                    'serialized_data' => $this->json->serialize([
                        'batch_index' => $batchIndex,
                        'products' => $batch
                    ]),
                    'status' => OperationInterface::STATUS_TYPE_OPEN,
                ]
            ];
        }

        // Schedule bulk operation
        $result = $this->bulkManagement->scheduleBulk(
            $bulkUuid,
            $operations,
            $bulkDescription,
            $userId
        );

        if (!$result) {
            throw new \RuntimeException('Failed to schedule bulk operation');
        }

        $this->logger->info('Bulk product import scheduled', [
            'bulk_uuid' => $bulkUuid,
            'product_count' => count($products),
            'batch_count' => count($batches)
        ]);

        return $bulkUuid;
    }
}
```

### Bulk Consumer Handler

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Queue;

use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Catalog\Api\Data\ProductInterfaceFactory;
use Magento\Framework\Exception\CouldNotSaveException;
use Psr\Log\LoggerInterface;

class ProductImportConsumer
{
    public function __construct(
        private readonly ProductRepositoryInterface $productRepository,
        private readonly ProductInterfaceFactory $productFactory,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Process product import batch
     *
     * @param string $serializedData
     * @return void
     */
    public function process(string $serializedData): void
    {
        $data = json_decode($serializedData, true);
        $batchIndex = $data['batch_index'];
        $products = $data['products'];

        $this->logger->info('Processing product import batch', [
            'batch_index' => $batchIndex,
            'product_count' => count($products)
        ]);

        $successCount = 0;
        $errorCount = 0;

        foreach ($products as $productData) {
            try {
                $this->importProduct($productData);
                $successCount++;
            } catch (\Exception $e) {
                $errorCount++;
                $this->logger->error('Failed to import product', [
                    'sku' => $productData['sku'] ?? 'unknown',
                    'error' => $e->getMessage()
                ]);
            }
        }

        $this->logger->info('Batch processing completed', [
            'batch_index' => $batchIndex,
            'success_count' => $successCount,
            'error_count' => $errorCount
        ]);
    }

    /**
     * Import single product
     *
     * @param array $productData
     * @return void
     * @throws CouldNotSaveException
     */
    private function importProduct(array $productData): void
    {
        $sku = $productData['sku'];

        try {
            // Try to load existing product
            $product = $this->productRepository->get($sku);
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            // Create new product
            $product = $this->productFactory->create();
            $product->setSku($sku);
        }

        // Update product data
        $product->setName($productData['name']);
        $product->setPrice($productData['price']);
        $product->setStatus($productData['status'] ?? 1);
        $product->setVisibility($productData['visibility'] ?? 4);
        $product->setTypeId($productData['type_id'] ?? 'simple');
        $product->setAttributeSetId($productData['attribute_set_id'] ?? 4);

        // Custom attributes
        if (isset($productData['custom_attributes'])) {
            foreach ($productData['custom_attributes'] as $code => $value) {
                $product->setCustomAttribute($code, $value);
            }
        }

        $this->productRepository->save($product);
    }
}
```

**Consumer configuration (communication.xml):**

```xml
<topic name="vendor.module.product.import" request="string">
    <handler name="productImport" type="Vendor\Module\Model\Queue\ProductImportConsumer" method="process"/>
</topic>
```

### Bulk Operation Status Check

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Erp;

use Magento\AsynchronousOperations\Api\BulkStatusInterface;

class BulkOperationStatusChecker
{
    public function __construct(
        private readonly BulkStatusInterface $bulkStatus
    ) {}

    /**
     * Get bulk operation status
     *
     * @param string $bulkUuid
     * @return array
     */
    public function getStatus(string $bulkUuid): array
    {
        $bulkDetails = $this->bulkStatus->getBulkDetailedStatus($bulkUuid);

        return [
            'bulk_uuid' => $bulkUuid,
            'status' => $bulkDetails->getStatus(),
            'operations_total' => $bulkDetails->getOperationsTotal(),
            'operations_successful' => $bulkDetails->getOperationsSuccessful(),
            'operations_failed' => $bulkDetails->getOperationsFailed(),
            'start_time' => $bulkDetails->getStartTime(),
            'user_id' => $bulkDetails->getUserId()
        ];
    }

    /**
     * Check if bulk operation is complete
     *
     * @param string $bulkUuid
     * @return bool
     */
    public function isComplete(string $bulkUuid): bool
    {
        $details = $this->bulkStatus->getBulkDetailedStatus($bulkUuid);
        $total = $details->getOperationsTotal();
        $completed = $details->getOperationsSuccessful() + $details->getOperationsFailed();

        return $total === $completed;
    }
}
```

---

## Order Sync Pattern (Magento → ERP)

### Complete Order Export Implementation

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Erp;

use Magento\Sales\Api\Data\OrderInterface;
use Magento\Framework\HTTP\Client\Curl;
use Magento\Framework\Serialize\Serializer\Json;
use Vendor\Module\Model\Config;
use Vendor\Module\Model\IdempotencyKeyGenerator;
use Psr\Log\LoggerInterface;

class OrderExporter
{
    public function __construct(
        private readonly Curl $curl,
        private readonly Json $json,
        private readonly Config $config,
        private readonly IdempotencyKeyGenerator $idempotencyKeyGenerator,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Export order to ERP
     *
     * @param OrderInterface $order
     * @return array
     * @throws \Exception
     */
    public function export(OrderInterface $order): array
    {
        $endpoint = $this->config->getErpOrderEndpoint();
        $apiKey = $this->config->getErpApiKey();

        $payload = $this->buildPayload($order);
        $idempotencyKey = $this->idempotencyKeyGenerator->generateForOrder($order);

        $this->logger->info('Exporting order to ERP', [
            'order_id' => $order->getEntityId(),
            'increment_id' => $order->getIncrementId(),
            'idempotency_key' => $idempotencyKey
        ]);

        $this->curl->setHeaders([
            'Content-Type' => 'application/json',
            'Authorization' => 'Bearer ' . $apiKey,
            'X-Idempotency-Key' => $idempotencyKey
        ]);

        $this->curl->post($endpoint, $this->json->serialize($payload));

        $statusCode = $this->curl->getStatus();
        $response = $this->curl->getBody();

        if ($statusCode !== 200 && $statusCode !== 201) {
            throw new \RuntimeException(
                sprintf('ERP order export failed: HTTP %d - %s', $statusCode, $response)
            );
        }

        $result = $this->json->unserialize($response);

        $this->logger->info('Order exported successfully', [
            'order_id' => $order->getEntityId(),
            'erp_order_id' => $result['erp_order_id'] ?? null
        ]);

        // Store ERP order ID in Magento order
        $order->setData('erp_order_id', $result['erp_order_id'] ?? null);
        $order->setData('erp_sync_status', 'synced');
        $order->setData('erp_sync_at', date('Y-m-d H:i:s'));

        return $result;
    }

    /**
     * Build ERP-compatible order payload
     *
     * @param OrderInterface $order
     * @return array
     */
    private function buildPayload(OrderInterface $order): array
    {
        return [
            'order_number' => $order->getIncrementId(),
            'customer' => [
                'email' => $order->getCustomerEmail(),
                'first_name' => $order->getCustomerFirstname(),
                'last_name' => $order->getCustomerLastname(),
                'customer_id' => $order->getCustomerId()
            ],
            'billing_address' => $this->formatAddress($order->getBillingAddress()),
            'shipping_address' => $this->formatAddress($order->getShippingAddress()),
            'items' => $this->formatItems($order),
            'totals' => [
                'subtotal' => $order->getSubtotal(),
                'shipping' => $order->getShippingAmount(),
                'tax' => $order->getTaxAmount(),
                'discount' => abs($order->getDiscountAmount()),
                'grand_total' => $order->getGrandTotal()
            ],
            'payment' => [
                'method' => $order->getPayment()->getMethod(),
                'amount' => $order->getGrandTotal(),
                'currency' => $order->getOrderCurrencyCode()
            ],
            'shipping' => [
                'method' => $order->getShippingMethod(),
                'description' => $order->getShippingDescription()
            ],
            'status' => $order->getStatus(),
            'created_at' => $order->getCreatedAt(),
            'metadata' => [
                'store_id' => $order->getStoreId(),
                'magento_order_id' => $order->getEntityId()
            ]
        ];
    }

    /**
     * Format address for ERP
     *
     * @param \Magento\Sales\Api\Data\OrderAddressInterface|null $address
     * @return array|null
     */
    private function formatAddress($address): ?array
    {
        if (!$address) {
            return null;
        }

        return [
            'first_name' => $address->getFirstname(),
            'last_name' => $address->getLastname(),
            'company' => $address->getCompany(),
            'street' => implode(', ', $address->getStreet()),
            'city' => $address->getCity(),
            'region' => $address->getRegion(),
            'postcode' => $address->getPostcode(),
            'country' => $address->getCountryId(),
            'telephone' => $address->getTelephone()
        ];
    }

    /**
     * Format order items for ERP
     *
     * @param OrderInterface $order
     * @return array
     */
    private function formatItems(OrderInterface $order): array
    {
        $items = [];

        foreach ($order->getItems() as $item) {
            // Skip child items (configurable product options)
            if ($item->getParentItemId()) {
                continue;
            }

            $items[] = [
                'sku' => $item->getSku(),
                'name' => $item->getName(),
                'qty' => $item->getQtyOrdered(),
                'price' => $item->getPrice(),
                'tax' => $item->getTaxAmount(),
                'discount' => $item->getDiscountAmount(),
                'row_total' => $item->getRowTotal(),
                'product_type' => $item->getProductType()
            ];
        }

        return $items;
    }
}
```

---

## Inventory Sync Pattern (ERP → Magento)

### Inventory Import Controller (REST API)

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Api;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\App\RequestInterface;
use Magento\Framework\Controller\Result\JsonFactory;
use Vendor\Module\Model\Erp\InventoryImporter;
use Vendor\Module\Model\ApiAuthenticator;
use Psr\Log\LoggerInterface;

class InventoryUpdate implements HttpPostActionInterface
{
    public function __construct(
        private readonly RequestInterface $request,
        private readonly JsonFactory $jsonFactory,
        private readonly InventoryImporter $inventoryImporter,
        private readonly ApiAuthenticator $authenticator,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Handle inventory update from ERP
     *
     * POST /rest/V1/erp/inventory/update
     * Body: {"items": [{"sku": "ABC123", "qty": 100, "warehouse": "US-EAST"}, ...]}
     *
     * @return \Magento\Framework\Controller\Result\Json
     */
    public function execute()
    {
        $result = $this->jsonFactory->create();

        // Authenticate request
        if (!$this->authenticator->authenticate($this->request)) {
            return $result->setHttpResponseCode(401)->setData([
                'success' => false,
                'message' => 'Unauthorized'
            ]);
        }

        try {
            $payload = json_decode($this->request->getContent(), true);

            if (!isset($payload['items']) || !is_array($payload['items'])) {
                return $result->setHttpResponseCode(400)->setData([
                    'success' => false,
                    'message' => 'Invalid payload: missing items array'
                ]);
            }

            $importResult = $this->inventoryImporter->import($payload['items']);

            return $result->setData([
                'success' => true,
                'result' => $importResult
            ]);

        } catch (\Exception $e) {
            $this->logger->error('Inventory import failed', [
                'error' => $e->getMessage()
            ]);

            return $result->setHttpResponseCode(500)->setData([
                'success' => false,
                'message' => 'Internal server error'
            ]);
        }
    }
}
```

### Inventory Importer (MSI-Compatible)

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Erp;

use Magento\InventoryApi\Api\SourceItemsSaveInterface;
use Magento\InventoryApi\Api\Data\SourceItemInterfaceFactory;
use Magento\Framework\Validation\ValidationException;
use Psr\Log\LoggerInterface;

class InventoryImporter
{
    public function __construct(
        private readonly SourceItemsSaveInterface $sourceItemsSave,
        private readonly SourceItemInterfaceFactory $sourceItemFactory,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Import inventory updates from ERP
     *
     * @param array $items
     * @return array
     * @throws ValidationException
     */
    public function import(array $items): array
    {
        $sourceItems = [];
        $successCount = 0;
        $errorCount = 0;
        $errors = [];

        foreach ($items as $item) {
            try {
                $this->validateItem($item);

                $sourceItem = $this->sourceItemFactory->create();
                $sourceItem->setSourceCode($item['warehouse'] ?? 'default');
                $sourceItem->setSku($item['sku']);
                $sourceItem->setQuantity($item['qty']);
                $sourceItem->setStatus($item['status'] ?? 1); // 1 = In Stock

                $sourceItems[] = $sourceItem;
                $successCount++;

            } catch (\Exception $e) {
                $errorCount++;
                $errors[] = [
                    'sku' => $item['sku'] ?? 'unknown',
                    'error' => $e->getMessage()
                ];

                $this->logger->error('Failed to prepare inventory item', [
                    'sku' => $item['sku'] ?? 'unknown',
                    'error' => $e->getMessage()
                ]);
            }
        }

        // Bulk save source items
        if (!empty($sourceItems)) {
            $this->sourceItemsSave->execute($sourceItems);
        }

        $this->logger->info('Inventory import completed', [
            'success_count' => $successCount,
            'error_count' => $errorCount
        ]);

        return [
            'success_count' => $successCount,
            'error_count' => $errorCount,
            'errors' => $errors
        ];
    }

    /**
     * Validate inventory item data
     *
     * @param array $item
     * @return void
     * @throws \InvalidArgumentException
     */
    private function validateItem(array $item): void
    {
        if (!isset($item['sku']) || empty($item['sku'])) {
            throw new \InvalidArgumentException('Missing required field: sku');
        }

        if (!isset($item['qty']) || !is_numeric($item['qty'])) {
            throw new \InvalidArgumentException('Invalid or missing qty field');
        }

        if ($item['qty'] < 0) {
            throw new \InvalidArgumentException('Quantity cannot be negative');
        }
    }
}
```

---

## Error Handling and Retry Logic

### Comprehensive Error Handler

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Vendor\Module\Model\Queue\RetryPublisher;
use Psr\Log\LoggerInterface;

class ErpErrorHandler
{
    private const TRANSIENT_HTTP_CODES = [408, 429, 500, 502, 503, 504];
    private const MAX_RETRY_ATTEMPTS = 3;

    public function __construct(
        private readonly RetryPublisher $retryPublisher,
        private readonly LoggerInterface $logger,
        private readonly ErrorNotifier $errorNotifier
    ) {}

    /**
     * Handle ERP integration error
     *
     * @param \Exception $exception
     * @param array $context
     * @return void
     */
    public function handle(\Exception $exception, array $context): void
    {
        $errorType = $this->classifyError($exception);
        $retryCount = $context['retry_count'] ?? 0;

        $this->logger->error('ERP integration error', [
            'error_type' => $errorType,
            'error_message' => $exception->getMessage(),
            'retry_count' => $retryCount,
            'context' => $context
        ]);

        switch ($errorType) {
            case 'transient':
                $this->handleTransientError($exception, $context, $retryCount);
                break;

            case 'validation':
                $this->handleValidationError($exception, $context);
                break;

            case 'authentication':
                $this->handleAuthenticationError($exception, $context);
                break;

            case 'permanent':
            default:
                $this->handlePermanentError($exception, $context);
                break;
        }
    }

    /**
     * Classify error type
     *
     * @param \Exception $exception
     * @return string
     */
    private function classifyError(\Exception $exception): string
    {
        $message = $exception->getMessage();
        $code = $exception->getCode();

        // HTTP status code based classification
        if (in_array($code, self::TRANSIENT_HTTP_CODES, true)) {
            return 'transient';
        }

        // Authentication errors
        if ($code === 401 || $code === 403) {
            return 'authentication';
        }

        // Validation errors
        if ($code === 400 || str_contains($message, 'validation')) {
            return 'validation';
        }

        // Permanent errors
        if ($code === 404 || $code === 405) {
            return 'permanent';
        }

        return 'unknown';
    }

    /**
     * Handle transient error (retry)
     *
     * @param \Exception $exception
     * @param array $context
     * @param int $retryCount
     * @return void
     */
    private function handleTransientError(\Exception $exception, array $context, int $retryCount): void
    {
        if ($retryCount >= self::MAX_RETRY_ATTEMPTS) {
            $this->logger->critical('Max retry attempts exceeded', $context);
            $this->errorNotifier->notifyDevOps('ERP integration failed after retries', $context);
            return;
        }

        $this->logger->info('Scheduling retry for transient error', [
            'retry_count' => $retryCount + 1,
            'context' => $context
        ]);

        $this->retryPublisher->publishWithDelay($context, $retryCount + 1);
    }

    /**
     * Handle validation error (log and alert)
     *
     * @param \Exception $exception
     * @param array $context
     * @return void
     */
    private function handleValidationError(\Exception $exception, array $context): void
    {
        $this->logger->warning('Validation error in ERP integration', [
            'error' => $exception->getMessage(),
            'context' => $context
        ]);

        // Alert business team (invalid data from Magento)
        $this->errorNotifier->notifyBusinessTeam('Invalid data sent to ERP', $context);
    }

    /**
     * Handle authentication error (alert DevOps)
     *
     * @param \Exception $exception
     * @param array $context
     * @return void
     */
    private function handleAuthenticationError(\Exception $exception, array $context): void
    {
        $this->logger->critical('Authentication failure with ERP', [
            'error' => $exception->getMessage(),
            'context' => $context
        ]);

        $this->errorNotifier->notifyDevOps('ERP API authentication failed', $context);
    }

    /**
     * Handle permanent error (log and skip)
     *
     * @param \Exception $exception
     * @param array $context
     * @return void
     */
    private function handlePermanentError(\Exception $exception, array $context): void
    {
        $this->logger->error('Permanent error in ERP integration', [
            'error' => $exception->getMessage(),
            'context' => $context
        ]);

        // Move to dead letter queue for manual review
        $this->errorNotifier->notifyDevOps('Permanent ERP integration error', $context);
    }
}
```

---

## Audit Logging

### Audit Logger for ERP Operations

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Magento\Framework\App\ResourceConnection;

class AuditLogger
{
    private const TABLE_NAME = 'vendor_module_erp_audit_log';

    public function __construct(
        private readonly ResourceConnection $resourceConnection
    ) {}

    /**
     * Log ERP operation
     *
     * @param array $data
     * @return void
     */
    public function log(array $data): void
    {
        $connection = $this->resourceConnection->getConnection();
        $table = $this->resourceConnection->getTableName(self::TABLE_NAME);

        $logEntry = [
            'operation_type' => $data['operation_type'] ?? 'unknown',
            'entity_type' => $data['entity_type'] ?? null,
            'entity_id' => $data['entity_id'] ?? null,
            'idempotency_key' => $data['idempotency_key'] ?? null,
            'request_payload' => json_encode($data['request'] ?? []),
            'response_payload' => json_encode($data['response'] ?? []),
            'status_code' => $data['status_code'] ?? null,
            'error_message' => $data['error'] ?? null,
            'duration_ms' => $data['duration_ms'] ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $connection->insert($table, $logEntry);
    }
}
```

**Database schema (db_schema.xml):**

```xml
<table name="vendor_module_erp_audit_log" resource="default" engine="innodb" comment="ERP Audit Log">
    <column xsi:type="int" name="log_id" unsigned="true" nullable="false" identity="true" comment="Log ID"/>
    <column xsi:type="varchar" name="operation_type" nullable="false" length="50" comment="Operation Type"/>
    <column xsi:type="varchar" name="entity_type" nullable="true" length="50" comment="Entity Type"/>
    <column xsi:type="varchar" name="entity_id" nullable="true" length="255" comment="Entity ID"/>
    <column xsi:type="varchar" name="idempotency_key" nullable="true" length="255" comment="Idempotency Key"/>
    <column xsi:type="text" name="request_payload" nullable="true" comment="Request Payload"/>
    <column xsi:type="text" name="response_payload" nullable="true" comment="Response Payload"/>
    <column xsi:type="int" name="status_code" nullable="true" comment="HTTP Status Code"/>
    <column xsi:type="text" name="error_message" nullable="true" comment="Error Message"/>
    <column xsi:type="int" name="duration_ms" nullable="true" comment="Duration (ms)"/>
    <column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP" comment="Created At"/>
    <constraint xsi:type="primary" referenceId="PRIMARY">
        <column name="log_id"/>
    </constraint>
    <index referenceId="VENDOR_MODULE_ERP_AUDIT_LOG_IDEMPOTENCY_KEY" indexType="btree">
        <column name="idempotency_key"/>
    </index>
    <index referenceId="VENDOR_MODULE_ERP_AUDIT_LOG_CREATED_AT" indexType="btree">
        <column name="created_at"/>
    </index>
</table>
```

---

## Assumptions

- **Magento version:** 2.4.7+ (MSI, Bulk API, Message Queues)
- **PHP version:** 8.2+ (typed properties, readonly)
- **ERP systems:** SAP, Oracle, NetSuite, Dynamics 365, or custom
- **Authentication:** Bearer tokens or HMAC signatures
- **Data volume:** 100-10,000 orders/day, 1,000-100,000 products
- **Network:** Reliable connectivity between Magento and ERP (consider VPN/private network for security)

## Why This Approach

**Webhooks for real-time sync:**
- Eliminates polling latency
- Reduces API rate limit consumption
- Event-driven architecture scales better
- Supports time-sensitive operations (order fulfillment)

**Idempotency keys:**
- Prevents duplicate order creation in ERP
- Enables safe retries without side effects
- Industry-standard pattern (Stripe, PayPal, Square use this)
- Simplifies error recovery

**Bulk API for large datasets:**
- 100x faster than individual API calls
- Async processing prevents timeouts
- Scalable to millions of records
- Built-in status tracking and error reporting

**Retry with exponential backoff:**
- Handles transient network failures gracefully
- Respects rate limits (delays increase with attempts)
- Prevents cascade failures
- Standard pattern in distributed systems

**Audit logging:**
- Forensic analysis of integration issues
- Compliance requirements (SOX, GDPR)
- Data reconciliation between systems
- Performance monitoring (duration_ms)

## Security Impact

- **Authentication:** Use API keys/tokens stored in `app/etc/env.php`; rotate regularly (90 days)
- **Authorization:** Validate ERP requests have proper permissions; implement IP whitelisting
- **HMAC signatures:** Verify webhook authenticity; prevents replay attacks
- **Idempotency keys:** Include timestamp to prevent replay beyond 24 hours
- **PII handling:** Log only entity IDs, not customer names/emails; encrypt audit logs at rest
- **Secrets management:** Use environment variables or vault services (AWS Secrets Manager, HashiCorp Vault)
- **HTTPS only:** Enforce TLS 1.2+ for all ERP communication; validate certificates

## Performance Impact

- **Webhook latency:** <1s for order export (async queue)
- **Polling overhead:** 5-10 min latency (cron frequency)
- **Bulk API:** 100 products/second (vs 1-2 with individual API calls)
- **Database load:** Audit logging adds 1-2ms per operation; index on idempotency_key critical
- **Cache impact:** Inventory updates invalidate FPC for affected products; use cache tags for selective invalidation
- **Message queue:** RabbitMQ recommended for high volume (10,000+ messages/day)

## Backward Compatibility

- **API versioning:** Use `/V1/`, `/V2/` endpoints; maintain both during deprecation period
- **Payload format:** Never remove fields; add new fields as optional
- **Idempotency keys:** Format is stable across versions; safe to cache for 24 hours
- **Bulk API:** Available in Magento 2.2+; schema stable through 2.4.x
- **MSI inventory:** Magento 2.3+ (replaces legacy CatalogInventory); migration required for 2.2 → 2.3+

## Tests to Add

**Unit tests:**
- Idempotency key generation consistency
- Error classification logic (transient vs permanent)
- Payload transformation correctness
- Signature generation/verification

**Integration tests:**
- Webhook delivery with retry logic
- Bulk operation scheduling and execution
- Inventory update with MSI
- Audit log persistence

**API functional tests:**
- POST /rest/V1/erp/inventory/update (success, validation errors, auth failure)
- Bulk operation status endpoint
- Idempotent request handling (duplicate detection)

**End-to-end tests:**
- Order placed → webhook sent → ERP receives → confirmation
- ERP inventory update → API call → Magento stock updated
- Network failure → retry → eventual success

## Documentation to Update

- **README:** ERP integration setup, required configuration, webhook endpoints
- **API documentation:** OpenAPI/Swagger spec for inventory/order endpoints
- **Runbook:** Error handling procedures, manual retry steps, rollback process
- **Architecture diagram:** Data flow between Magento, middleware, ERP
- **Configuration guide:** How to set webhook URLs, API keys, retry policies
- **Troubleshooting:** Common errors (401, 429, 500) and resolutions

---

## Additional Resources

- [Magento DevDocs: Bulk Operations](https://developer.adobe.com/commerce/php/development/components/message-queues/bulk-operations/)
- [Magento DevDocs: MSI (Multi-Source Inventory)](https://experienceleague.adobe.com/docs/commerce-admin/inventory/introduction.html)
- [Stripe Idempotency Guide](https://stripe.com/docs/api/idempotent_requests)
- [Webhook Best Practices](https://webhooks.fyi/)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)

## Related Documentation

### Related Guides

- [Service Contracts vs Repositories in Magento 2](../explanations/service-contracts-repositories.md)
- [Message Queue Architecture in Magento 2](../explanations/message-queue-architecture.md)
- [Cron Jobs Implementation: Building Reliable Scheduled Tasks in Magento 2](cron-jobs.md)
- [Import/Export System Deep Dive](import-export.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
- [Magento_Customer Overview](../../modules/customer/README.md)
