---
title: "Message Queue Architecture in Magento 2"
description: "Developer guide: Message Queue Architecture in Magento 2"
type: "explanation"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "25 minutes"
topics:
  - message queues
  - rabbitmq
  - async operations
  - performance
  - system architecture
last_updated: "2026-02-07"
---

# Message Queue Architecture in Magento 2

## Overview

Message queue architecture is critical for building scalable Magento applications that handle high-volume operations without blocking user interactions. This guide explores Magento's message queue framework, comparing RabbitMQ and MySQL queue implementations, and provides production-ready patterns for async operations.

**What you'll learn:**
- Architecture of Magento's message queue system
- RabbitMQ vs MySQL queue trade-offs
- Implementing publishers and consumers
- Retry logic and dead letter queue patterns
- Performance tuning and monitoring
- Production debugging strategies

**Prerequisites:**
- Strong PHP knowledge (interfaces, DI, typed properties)
- Understanding of Magento module structure
- Familiarity with service contracts
- Basic RabbitMQ concepts (exchanges, queues, bindings)

---

## Message Queue Architecture Overview

### Core Components

Magento's message queue framework consists of four primary components:

1. **Publisher** – Publishes messages to a topic
2. **Topic** – Named channel that routes messages
3. **Consumer** – Processes messages from a queue
4. **Queue** – Stores messages until processed

**Data Flow:**
```
[Service Contract] → [Publisher] → [Topic] → [Exchange] → [Queue] → [Consumer] → [Handler]
```

### Framework Integration

The message queue framework integrates with:
- `Magento\Framework\MessageQueue\PublisherInterface`
- `Magento\Framework\MessageQueue\ConsumerInterface`
- `Magento\Framework\MessageQueue\EnvelopeInterface`
- `Magento\Framework\Bulk\OperationInterface` (for bulk operations)

---

## RabbitMQ vs MySQL Queue: Architecture Comparison

### RabbitMQ Architecture

**Strengths:**
- Native async processing with AMQP protocol
- High throughput (10,000+ messages/second)
- Advanced routing (topic exchanges, fanout, direct)
- TTL, DLX (dead letter exchange), priority queues
- Horizontal scaling via clustering
- Management UI for monitoring

**Trade-offs:**
- Additional infrastructure dependency
- Memory-based (configure persistence)
- Requires operational expertise
- Network latency considerations

**Recommended for:**
- High-volume operations (bulk imports, exports)
- Real-time integrations (ERP sync, inventory updates)
- Multi-instance deployments
- Production environments with DevOps support

### MySQL Queue Architecture

**Strengths:**
- Zero additional infrastructure
- ACID guarantees (transactional integrity)
- Familiar tooling (SQL queries for debugging)
- Simpler deployment

**Trade-offs:**
- Lower throughput (~100-500 messages/second)
- Table locking under high concurrency
- Limited routing capabilities
- Database I/O overhead

**Recommended for:**
- Development/staging environments
- Low-volume async operations
- Single-instance deployments
- When operational simplicity is paramount

### Performance Comparison

| Metric | RabbitMQ | MySQL Queue |
|--------|----------|-------------|
| Throughput | 10,000+ msg/s | 100-500 msg/s |
| Latency (p99) | <50ms | 200-500ms |
| Concurrent consumers | 50+ | 5-10 |
| Message ordering | Per-queue | Global |
| Persistence | Configurable | Always (DB) |
| Memory footprint | 100-500MB | Negligible |

---

## Implementing Message Queue Operations

### Step 1: Define Message DTO (Data Transfer Object)

Create a typed data interface for your message payload:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Api\Data;

/**
 * Order sync message DTO
 *
 * @api
 */
interface OrderSyncMessageInterface
{
    /**
     * Get order ID
     *
     * @return int
     */
    public function getOrderId(): int;

    /**
     * Set order ID
     *
     * @param int $orderId
     * @return $this
     */
    public function setOrderId(int $orderId): self;

    /**
     * Get sync operation type
     *
     * @return string
     */
    public function getOperation(): string;

    /**
     * Set sync operation type
     *
     * @param string $operation
     * @return $this
     */
    public function setOperation(string $operation): self;

    /**
     * Get retry attempt count
     *
     * @return int
     */
    public function getRetryCount(): int;

    /**
     * Set retry attempt count
     *
     * @param int $count
     * @return $this
     */
    public function setRetryCount(int $count): self;
}
```

**Implementation:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Data;

use Vendor\Module\Api\Data\OrderSyncMessageInterface;
use Magento\Framework\DataObject;

class OrderSyncMessage extends DataObject implements OrderSyncMessageInterface
{
    private const KEY_ORDER_ID = 'order_id';
    private const KEY_OPERATION = 'operation';
    private const KEY_RETRY_COUNT = 'retry_count';

    public function getOrderId(): int
    {
        return (int) $this->getData(self::KEY_ORDER_ID);
    }

    public function setOrderId(int $orderId): OrderSyncMessageInterface
    {
        return $this->setData(self::KEY_ORDER_ID, $orderId);
    }

    public function getOperation(): string
    {
        return (string) $this->getData(self::KEY_OPERATION);
    }

    public function setOperation(string $operation): OrderSyncMessageInterface
    {
        return $this->setData(self::KEY_OPERATION, $operation);
    }

    public function getRetryCount(): int
    {
        return (int) $this->getData(self::KEY_RETRY_COUNT);
    }

    public function setRetryCount(int $count): OrderSyncMessageInterface
    {
        return $this->setData(self::KEY_RETRY_COUNT, $count);
    }
}
```

### Step 2: Configure Queue Topology (communication.xml)

Define topics, queues, exchanges, and bindings:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Communication/etc/communication.xsd">

    <!-- Define Topic -->
    <topic name="vendor.module.order.sync" request="Vendor\Module\Api\Data\OrderSyncMessageInterface">
        <handler name="syncToErp" type="Vendor\Module\Model\Queue\OrderSyncHandler" method="execute"/>
    </topic>

    <!-- Define Publisher (optional, uses default if omitted) -->
    <publisher topic="vendor.module.order.sync">
        <connection name="amqp" exchange="magento"/>
    </publisher>
</config>
```

**queue_topology.xml** (RabbitMQ-specific):

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework-message-queue:etc/topology.xsd">

    <exchange name="magento" type="topic" connection="amqp">
        <binding id="orderSyncBinding"
                 topic="vendor.module.order.sync"
                 destinationType="queue"
                 destination="vendor.module.order.sync.queue"/>
    </exchange>
</config>
```

**queue_consumer.xml**:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework-message-queue:etc/consumer.xsd">

    <consumer name="vendor.module.order.sync.consumer"
              queue="vendor.module.order.sync.queue"
              connection="amqp"
              consumerInstance="Magento\Framework\MessageQueue\Consumer"
              handler="Vendor\Module\Model\Queue\OrderSyncHandler::execute"
              maxMessages="1000"/>
</config>
```

### Step 3: Implement Publisher

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model;

use Vendor\Module\Api\Data\OrderSyncMessageInterface;
use Vendor\Module\Api\Data\OrderSyncMessageInterfaceFactory;
use Magento\Framework\MessageQueue\PublisherInterface;
use Psr\Log\LoggerInterface;

class OrderSyncPublisher
{
    private const TOPIC_NAME = 'vendor.module.order.sync';

    public function __construct(
        private readonly PublisherInterface $publisher,
        private readonly OrderSyncMessageInterfaceFactory $messageFactory,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Publish order sync message to queue
     *
     * @param int $orderId
     * @param string $operation
     * @return void
     * @throws \Exception
     */
    public function publish(int $orderId, string $operation): void
    {
        try {
            $message = $this->messageFactory->create();
            $message->setOrderId($orderId);
            $message->setOperation($operation);
            $message->setRetryCount(0);

            $this->publisher->publish(self::TOPIC_NAME, $message);

            $this->logger->info('Order sync message published', [
                'order_id' => $orderId,
                'operation' => $operation
            ]);
        } catch (\Exception $e) {
            $this->logger->error('Failed to publish order sync message', [
                'order_id' => $orderId,
                'error' => $e->getMessage()
            ]);
            throw $e;
        }
    }
}
```

### Step 4: Implement Consumer Handler

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Queue;

use Vendor\Module\Api\Data\OrderSyncMessageInterface;
use Vendor\Module\Model\Erp\OrderSyncService;
use Magento\Framework\Exception\LocalizedException;
use Psr\Log\LoggerInterface;

class OrderSyncHandler
{
    private const MAX_RETRY_ATTEMPTS = 3;

    public function __construct(
        private readonly OrderSyncService $syncService,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Process order sync message
     *
     * @param OrderSyncMessageInterface $message
     * @return void
     * @throws \Magento\Framework\Exception\LocalizedException
     */
    public function execute(OrderSyncMessageInterface $message): void
    {
        $orderId = $message->getOrderId();
        $operation = $message->getOperation();
        $retryCount = $message->getRetryCount();

        $this->logger->info('Processing order sync', [
            'order_id' => $orderId,
            'operation' => $operation,
            'retry_count' => $retryCount
        ]);

        try {
            $this->syncService->syncOrder($orderId, $operation);

            $this->logger->info('Order sync completed', [
                'order_id' => $orderId
            ]);
        } catch (LocalizedException $e) {
            $this->handleRetry($message, $e);
        } catch (\Exception $e) {
            $this->logger->critical('Order sync failed with unexpected error', [
                'order_id' => $orderId,
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString()
            ]);
            throw $e; // Reject message to DLX
        }
    }

    /**
     * Handle retry logic for transient failures
     *
     * @param OrderSyncMessageInterface $message
     * @param \Exception $exception
     * @return void
     * @throws \Exception
     */
    private function handleRetry(OrderSyncMessageInterface $message, \Exception $exception): void
    {
        $retryCount = $message->getRetryCount();

        if ($retryCount < self::MAX_RETRY_ATTEMPTS) {
            $this->logger->warning('Order sync failed, will retry', [
                'order_id' => $message->getOrderId(),
                'retry_count' => $retryCount + 1,
                'error' => $exception->getMessage()
            ]);

            // Re-publish with incremented retry count
            $message->setRetryCount($retryCount + 1);
            // Implementation would use PublisherInterface to republish
            throw $exception; // Temporary: reject to trigger retry
        } else {
            $this->logger->error('Order sync failed after max retries', [
                'order_id' => $message->getOrderId(),
                'retry_count' => $retryCount,
                'error' => $exception->getMessage()
            ]);
            throw $exception; // Send to DLX
        }
    }
}
```

---

## Advanced Patterns: Retry Logic and Dead Letter Queues

### Retry Pattern with Exponential Backoff

For transient failures (network timeouts, rate limits), implement exponential backoff:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Queue;

use Magento\Framework\MessageQueue\PublisherInterface;

class RetryPublisher
{
    private const TOPIC_RETRY = 'vendor.module.order.sync.retry';
    private const BASE_DELAY = 60; // seconds

    public function __construct(
        private readonly PublisherInterface $publisher
    ) {}

    /**
     * Republish message with delay
     *
     * @param mixed $message
     * @param int $retryCount
     * @return void
     */
    public function publishWithDelay($message, int $retryCount): void
    {
        // Calculate exponential backoff: 60s, 120s, 240s
        $delay = self::BASE_DELAY * (2 ** $retryCount);

        // Set message expiration for delayed processing
        $message->setRetryCount($retryCount + 1);
        $message->setScheduledAt(time() + $delay);

        $this->publisher->publish(self::TOPIC_RETRY, $message);
    }
}
```

### Dead Letter Queue Configuration (RabbitMQ)

Configure DLX in `queue_topology.xml`:

```xml
<exchange name="magento.dlx" type="fanout" connection="amqp">
    <binding id="dlxBinding"
             destinationType="queue"
             destination="vendor.module.order.sync.dlq"/>
</exchange>

<queue name="vendor.module.order.sync.queue" connection="amqp">
    <arguments>
        <argument name="x-dead-letter-exchange" xsi:type="string">magento.dlx</argument>
        <argument name="x-message-ttl" xsi:type="number">3600000</argument> <!-- 1 hour -->
        <argument name="x-max-retries" xsi:type="number">3</argument>
    </arguments>
</queue>
```

### DLQ Monitoring and Recovery

Create admin command to inspect DLQ:

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Console\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use PhpAmqpLib\Connection\AMQPStreamConnection;

class InspectDlqCommand extends Command
{
    protected function configure(): void
    {
        $this->setName('queue:dlq:inspect')
            ->setDescription('Inspect dead letter queue messages');
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        // Implementation connects to RabbitMQ management API
        // Lists messages in DLQ with metadata
        // Provides options to requeue or purge

        return Command::SUCCESS;
    }
}
```

---

## Performance Tuning

### Consumer Configuration

**Optimize `maxMessages` parameter:**

```xml
<!-- Process 1000 messages before restarting consumer -->
<consumer name="vendor.module.order.sync.consumer"
          maxMessages="1000"/>
```

**Why 1000?** Prevents memory leaks from long-running processes while maintaining throughput.

### Parallel Consumers

Run multiple consumer instances:

```bash
# Start 5 parallel consumers
for i in {1..5}; do
    bin/magento queue:consumers:start vendor.module.order.sync.consumer --max-messages=1000 &
done
```

**Supervisor configuration** (production):

```ini
[program:magento_order_sync_consumer]
command=/var/www/html/bin/magento queue:consumers:start vendor.module.order.sync.consumer --max-messages=1000
process_name=%(program_name)s_%(process_num)02d
numprocs=5
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/log/magento/consumer_order_sync.log
```

### RabbitMQ Tuning

**Connection pooling** in `env.php`:

```php
'queue' => [
    'amqp' => [
        'host' => 'localhost',
        'port' => '5672',
        'user' => 'guest',
        'password' => 'guest',
        'virtualhost' => '/',
        'ssl' => false,
        'pool_size' => 10, // Connection pool
        'heartbeat' => 60,
        'read_timeout' => 30,
        'write_timeout' => 30
    ]
]
```

**Prefetch count** (message batching):

```xml
<consumer name="vendor.module.order.sync.consumer"
          connection="amqp"
          maxMessages="1000">
    <arguments>
        <argument name="prefetch_count" xsi:type="number">100</argument>
    </arguments>
</consumer>
```

---

## Monitoring and Debugging

### RabbitMQ Management UI

Access at `http://localhost:15672` (default credentials: guest/guest)

**Key metrics:**
- Queue depth (unacked messages)
- Consumer utilization (%)
- Message rates (publish/consume per second)
- Memory usage

### CLI Monitoring

```bash
# Check queue status
bin/magento queue:consumers:list

# View RabbitMQ queue details
rabbitmqctl list_queues name messages consumers

# Consumer process status
ps aux | grep "queue:consumers:start"
```

### Custom Monitoring Service

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Monitoring;

use PhpAmqpLib\Connection\AMQPStreamConnection;

class QueueMonitor
{
    public function __construct(
        private readonly AMQPStreamConnection $connection
    ) {}

    /**
     * Get queue metrics
     *
     * @param string $queueName
     * @return array
     */
    public function getQueueMetrics(string $queueName): array
    {
        $channel = $this->connection->channel();
        [, $messageCount, $consumerCount] = $channel->queue_declare($queueName, true);

        return [
            'queue' => $queueName,
            'messages' => $messageCount,
            'consumers' => $consumerCount,
            'avg_processing_time' => $this->getAvgProcessingTime($queueName),
            'error_rate' => $this->getErrorRate($queueName)
        ];
    }

    private function getAvgProcessingTime(string $queueName): float
    {
        // Query logs or metrics database
        return 0.0;
    }

    private function getErrorRate(string $queueName): float
    {
        // Calculate from exception logs
        return 0.0;
    }
}
```

### Debugging Failed Messages

**Enable verbose logging:**

```php
<?php
// In consumer handler
$this->logger->debug('Message payload', [
    'message' => json_encode($message->getData()),
    'memory_usage' => memory_get_usage(true),
    'peak_memory' => memory_get_peak_usage(true)
]);
```

**Replay messages from DLQ:**

> **Note:** `queue:dlq:replay` is not a core Magento CLI command. DLQ replay
> must be implemented as a custom command or handled through your message broker
> (e.g., RabbitMQ Management UI or `rabbitmqadmin`).

```bash
# Example using RabbitMQ CLI to move messages from DLQ back to the original queue:
rabbitmqadmin get queue=vendor.module.order.sync.dlq count=10 --format=json
```

---

## Security Considerations

### Message Validation

Always validate message payloads:

```php
private function validateMessage(OrderSyncMessageInterface $message): void
{
    if ($message->getOrderId() <= 0) {
        throw new \InvalidArgumentException('Invalid order ID');
    }

    $allowedOperations = ['create', 'update', 'cancel'];
    if (!in_array($message->getOperation(), $allowedOperations, true)) {
        throw new \InvalidArgumentException('Invalid operation type');
    }
}
```

### Authorization in Consumers

Verify permissions before processing:

```php
public function execute(OrderSyncMessageInterface $message): void
{
    // Check if order exists and is accessible
    try {
        $order = $this->orderRepository->get($message->getOrderId());
    } catch (NoSuchEntityException $e) {
        $this->logger->warning('Order not found', ['order_id' => $message->getOrderId()]);
        return; // Skip processing
    }

    // Proceed with sync
}
```

### Sensitive Data Handling

**Never log full order data:**

```php
// BAD
$this->logger->info('Processing order', ['order' => $order->getData()]);

// GOOD
$this->logger->info('Processing order', [
    'order_id' => $order->getId(),
    'status' => $order->getStatus(),
    'store_id' => $order->getStoreId()
]);
```

### RabbitMQ Access Control

Configure vhosts and user permissions:

```bash
rabbitmqctl add_vhost magento_prod
rabbitmqctl add_user magento_user secure_password
rabbitmqctl set_permissions -p magento_prod magento_user ".*" ".*" ".*"
```

---

## Testing Strategies

### Unit Test: Publisher

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Unit\Model;

use PHPUnit\Framework\TestCase;
use Vendor\Module\Model\OrderSyncPublisher;
use Magento\Framework\MessageQueue\PublisherInterface;

class OrderSyncPublisherTest extends TestCase
{
    private PublisherInterface $publisherMock;
    private OrderSyncPublisher $publisher;

    protected function setUp(): void
    {
        $this->publisherMock = $this->createMock(PublisherInterface::class);
        $messageFactory = $this->createMock(OrderSyncMessageInterfaceFactory::class);
        $logger = $this->createMock(LoggerInterface::class);

        $this->publisher = new OrderSyncPublisher(
            $this->publisherMock,
            $messageFactory,
            $logger
        );
    }

    public function testPublishSuccess(): void
    {
        $this->publisherMock->expects($this->once())
            ->method('publish')
            ->with('vendor.module.order.sync', $this->anything());

        $this->publisher->publish(123, 'create');
    }
}
```

### Integration Test: Consumer

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Integration\Model\Queue;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

class OrderSyncHandlerTest extends TestCase
{
    /**
     * @magentoDataFixture Magento/Sales/_files/order.php
     */
    public function testExecuteProcessesOrder(): void
    {
        $objectManager = Bootstrap::getObjectManager();
        $handler = $objectManager->create(OrderSyncHandler::class);
        $messageFactory = $objectManager->create(OrderSyncMessageInterfaceFactory::class);

        $message = $messageFactory->create();
        $message->setOrderId(100000001);
        $message->setOperation('create');

        $handler->execute($message);

        // Assert ERP sync occurred
        // Verify order status updated
    }
}
```

---

## Production Deployment Checklist

### Pre-Deployment

- [ ] RabbitMQ cluster provisioned (3+ nodes for HA)
- [ ] Connection pooling configured in `env.php`
- [ ] Supervisor/systemd consumer services configured
- [ ] Dead letter queue exchange and queue created
- [ ] Monitoring alerts configured (queue depth > 1000)
- [ ] Log aggregation enabled (ELK, Splunk)

### Post-Deployment

- [ ] Verify consumers are running (`ps aux | grep queue:consumers`)
- [ ] Check RabbitMQ management UI for message flow
- [ ] Publish test message and verify processing
- [ ] Validate DLQ is empty
- [ ] Review consumer error logs
- [ ] Load test with expected message volume

### Rollback Plan

If consumers fail:
1. Stop all consumer processes
2. Drain queue manually or via CLI
3. Revert to previous codebase version
4. Restart consumers
5. Replay failed messages from backup

---

## Assumptions

- **Magento version:** 2.4.7+ (uses PHP 8.2+ typed properties)
- **RabbitMQ version:** 3.12+ (for latest features)
- **PHP version:** 8.2+ (readonly properties, enums)
- **Environment:** Production assumes Supervisor/systemd for process management
- **Modules:** `Magento_MessageQueue`, `Magento_AmqpBase` enabled

## Why This Approach

**RabbitMQ over MySQL queue:**
- 100x throughput improvement for high-volume operations
- Native async processing without DB overhead
- Advanced routing and retry capabilities
- Industry-standard AMQP protocol

**Typed DTOs:**
- Compile-time safety with PHP 8.2+ types
- Serialization predictability
- IDE autocomplete and static analysis

**Retry with exponential backoff:**
- Handles transient failures gracefully
- Prevents cascade failures
- Respects rate limits on external APIs

**Dead letter queues:**
- Prevents infinite retry loops
- Enables manual inspection of failures
- Supports forensic analysis

## Security Impact

- **Message validation:** Prevents malicious payloads from executing arbitrary operations
- **Authorization checks:** Ensures consumers respect ACL and entity permissions
- **Sensitive data:** PII logging restricted; use order IDs only
- **RabbitMQ ACL:** Vhost isolation prevents cross-tenant access
- **Secrets:** RabbitMQ credentials stored in `env.php` (excluded from VCS)

## Performance Impact

- **Throughput:** 10,000+ messages/second with RabbitMQ
- **Latency:** <50ms p99 with prefetch tuning
- **Memory:** 100-500MB per RabbitMQ node
- **Database:** Eliminates DB writes for queue operations (RabbitMQ mode)
- **FPC:** No impact (async operations don't affect page cache)
- **Redis:** Can use Redis as alternative queue backend (experimental)

## Backward Compatibility

- **Configuration:** `communication.xml`, `queue_topology.xml`, `queue_consumer.xml` are backward compatible across 2.4.x
- **API changes:** Message DTOs must implement versioned interfaces for BC
- **Deprecations:** Avoid `Magento\Framework\MessageQueue\ConfigInterface` (deprecated in 2.4.6+)
- **Migration:** MySQL queue to RabbitMQ requires no code changes, only config

## Tests to Add

**Unit tests:**
- Publisher publishes to correct topic
- Consumer handler processes valid messages
- Retry logic increments count correctly
- Validation rejects invalid payloads

**Integration tests:**
- End-to-end message flow (publish → consume)
- Database state changes after processing
- Error handling (exceptions, retries)
- Message deserialization

**Functional tests:**
- RabbitMQ connection pooling under load
- Consumer process stability (memory leaks)
- DLQ message routing
- Performance benchmarks (messages/second)

**Mutation tests:**
- Remove retry logic → verify failures cascade
- Skip validation → verify malformed messages rejected
- Disable DLX → verify queue blocks on errors

## Documentation to Update

- **README:** Add RabbitMQ setup instructions and Supervisor config examples
- **CHANGELOG:** Document new queue topics and consumer commands
- **Admin guide:** How to monitor queue health via RabbitMQ UI
- **Runbook:** Consumer restart procedures, DLQ replay process
- **Screenshots:** RabbitMQ management UI (queues, exchanges, bindings)
- **Architecture diagram:** Message flow with exchanges and bindings

---

## Additional Resources

- [Magento DevDocs: Message Queues](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/message-queues/message-queue-framework.html)
- [RabbitMQ Best Practices](https://www.rabbitmq.com/best-practices.html)
- [AMQP 0-9-1 Protocol Reference](https://www.rabbitmq.com/amqp-0-9-1-reference.html)
- [Supervisor Configuration](http://supervisord.org/configuration.html)

## Related Documentation

### Related Guides

- [Cron Jobs Implementation: Building Reliable Scheduled Tasks in Magento 2](../how-to/cron-jobs.md)
- [ERP Integration Patterns for Magento 2](../how-to/erp-integration.md)
- [Indexer System Deep Dive: Understanding Magento 2's Data Indexing Architecture](indexer-system.md)

### Related Module Documentation

- [Magento_Sales Overview](../../modules/sales/README.md)
- [Magento_Catalog Overview](../../modules/catalog/README.md)
