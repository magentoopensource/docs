---
title: "Cron Jobs Implementation: Building Reliable Scheduled Tasks in Magento 2"
description: "Complete guide to implementing cron jobs in Magento 2: crontab.xml configuration, cron group management, long-running job strategies, logging and monitoring, error handling, testing, and production deployment"
type: "how-to"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "75 minutes"
topics:
  - cron
  - scheduled-tasks
  - background-jobs
  - async-processing
  - monitoring
  - error-handling
  - testing
  - performance
last_updated: "2026-02-07"
---

# Cron Jobs Implementation: Building Reliable Scheduled Tasks in Magento 2

## Learning Objectives

By completing this guide, you will:

- Understand Magento 2's cron architecture and execution flow
- Configure cron jobs via `crontab.xml` with proper scheduling
- Implement cron job classes with robust error handling
- Manage cron groups for resource isolation and scaling
- Handle long-running jobs with proper timeout and chunking strategies
- Implement comprehensive logging and monitoring
- Test cron jobs in development and staging environments
- Deploy and monitor cron jobs in production
- Debug common cron issues (missed runs, overlapping executions, failures)

## Introduction

Magento 2's cron system enables scheduled background tasks for indexing, cache cleaning, email sending, order processing, and custom business logic. Unlike server-level cron jobs, Magento's cron framework provides scheduling, error handling, locking, and monitoring within the application.

### Magento Cron vs System Cron

**System cron (Linux crontab):**
- Schedules Magento's cron runner
- Runs: `bin/magento cron:run`
- Frequency: Every minute (typical)

**Magento cron (application-level):**
- Schedules individual jobs
- Managed in `crontab.xml`
- Executed by cron runner

**Architecture:**
```
System Cron (every 1 minute)
  ↓
bin/magento cron:run
  ↓
Cron Schedule Table (cron_schedule)
  ↓
Job Execution (per schedule entry)
  ↓
Job Class::execute()
```

### Cron Groups

Magento organizes cron jobs into **groups** for resource management:

| Group | Purpose | Jobs | Frequency |
|-------|---------|------|-----------|
| **default** | General tasks | Cache cleaning, sitemap generation | Every 1 minute |
| **index** | Indexer updates | Mview changelog processing | Every 1 minute |
| **consumers** | Message queue consumers | Async operations, bulk API | Continuous |
| **catalog_event** | Catalog operations | Scheduled product updates | Every 1 minute |

**Benefits of groups:**
- Isolate resource-intensive jobs
- Run groups on different servers
- Control concurrency per group

## Cron Job Configuration

### Basic Cron Job

**etc/crontab.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/crontab.xsd">
    <group id="default">
        <job name="vendor_module_cleanup_temp_data" instance="Vendor\Module\Cron\CleanupTempData" method="execute">
            <schedule>0 2 * * *</schedule><!-- Daily at 2:00 AM -->
        </job>
    </group>
</config>
```

**Configuration elements:**
- `group id`: Cron group (`default`, `index`, custom)
- `job name`: Unique identifier (module_name format)
- `instance`: Cron job class (must implement `execute()` method)
- `method`: Method to execute (default: `execute`)
- `schedule`: Cron expression (minute, hour, day, month, weekday)

### Cron Expression Syntax

Magento uses standard cron expression format:

```
* * * * *
│ │ │ │ │
│ │ │ │ └─ Day of week (0-6, Sunday=0)
│ │ │ └─── Month (1-12)
│ │ └───── Day of month (1-31)
│ └─────── Hour (0-23)
└───────── Minute (0-59)
```

**Common patterns:**

```xml
<!-- Every 5 minutes -->
<schedule>*/5 * * * *</schedule>

<!-- Every hour at minute 15 -->
<schedule>15 * * * *</schedule>

<!-- Daily at 3:30 AM -->
<schedule>30 3 * * *</schedule>

<!-- Mondays at 9:00 AM -->
<schedule>0 9 * * 1</schedule>

<!-- First day of month at midnight -->
<schedule>0 0 1 * *</schedule>

<!-- Every 15 minutes during business hours (9 AM - 5 PM) -->
<schedule>*/15 9-17 * * 1-5</schedule>
```

### Configuration-Based Schedule

For admin-configurable schedules, use config path instead of hardcoded expression:

**etc/crontab.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/crontab.xsd">
    <group id="default">
        <job name="vendor_module_report_generation" instance="Vendor\Module\Cron\GenerateReport" method="execute">
            <config_path>vendor_module/reports/cron_schedule</config_path>
        </job>
    </group>
</config>
```

**etc/config.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Store:etc/config.xsd">
    <default>
        <vendor_module>
            <reports>
                <cron_schedule>0 1 * * *</cron_schedule><!-- Daily at 1:00 AM -->
            </reports>
        </vendor_module>
    </default>
</config>
```

**etc/adminhtml/system.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system_file.xsd">
    <system>
        <section id="vendor_module">
            <group id="reports">
                <field id="cron_schedule" translate="label" type="text" sortOrder="10" showInDefault="1" showInWebsite="0" showInStore="0">
                    <label>Report Generation Schedule</label>
                    <comment>Cron expression (e.g., "0 1 * * *" for daily at 1 AM)</comment>
                    <!-- Custom backend model for cron expression validation -->
                    <backend_model>Vendor\Module\Model\Config\Backend\CronSchedule</backend_model>
                </field>
            </group>
        </section>
    </system>
</config>
```

**Benefits:**
- Merchants can adjust schedule without code changes
- Backend model validates cron expression syntax
- Schedule changes take effect on next cron run

## Implementing Cron Job Classes

### Basic Cron Job

**Cron/CleanupTempData.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Cron;

use Psr\Log\LoggerInterface;
use Vendor\Module\Model\TempDataRepository;

/**
 * Cleanup temporary data older than 30 days
 */
class CleanupTempData
{
    private const RETENTION_DAYS = 30;

    public function __construct(
        private readonly TempDataRepository $tempDataRepository,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Execute cron job
     *
     * @return void
     */
    public function execute(): void
    {
        $this->logger->info('Starting temporary data cleanup');

        try {
            $deletedCount = $this->tempDataRepository->deleteOlderThan(self::RETENTION_DAYS);

            $this->logger->info('Temporary data cleanup completed', [
                'deleted_records' => $deletedCount,
                'retention_days' => self::RETENTION_DAYS,
            ]);

        } catch (\Exception $e) {
            $this->logger->error('Temporary data cleanup failed', [
                'exception' => $e->getMessage(),
                'trace' => $e->getTraceAsString(),
            ]);

            // Re-throw to mark cron job as failed
            throw $e;
        }
    }
}
```

**Key patterns:**
- **Return type:** `void` (exceptions indicate failure)
- **Dependency injection:** All dependencies via constructor
- **Logging:** Info-level for normal execution, error-level for failures
- **Error handling:** Try-catch with re-throw (ensures job marked as failed)

### Cron Job with Locking

Prevent overlapping executions using locks:

**Cron/GenerateReport.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Cron;

use Magento\Framework\Lock\LockManagerInterface;
use Psr\Log\LoggerInterface;
use Vendor\Module\Service\ReportGenerator;

/**
 * Generate daily sales report
 */
class GenerateReport
{
    private const LOCK_NAME = 'vendor_module_generate_report';
    private const LOCK_TIMEOUT = 3600; // 1 hour

    public function __construct(
        private readonly LockManagerInterface $lockManager,
        private readonly ReportGenerator $reportGenerator,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Execute cron job
     *
     * @return void
     */
    public function execute(): void
    {
        // Attempt to acquire lock
        if (!$this->lockManager->lock(self::LOCK_NAME, self::LOCK_TIMEOUT)) {
            $this->logger->warning('Report generation already running, skipping execution');
            return;
        }

        try {
            $this->logger->info('Starting report generation');

            $report = $this->reportGenerator->generate();

            $this->logger->info('Report generation completed', [
                'report_id' => $report->getId(),
                'records_processed' => $report->getRecordCount(),
            ]);

        } catch (\Exception $e) {
            $this->logger->error('Report generation failed', [
                'exception' => $e->getMessage(),
            ]);

            throw $e;

        } finally {
            // Always release lock
            $this->lockManager->unlock(self::LOCK_NAME);
        }
    }
}
```

**Locking strategies:**

**1. Skip execution if locked (shown above):**
- Use case: Job can be delayed without issue
- Behavior: Skip if already running

**2. Wait for lock:**
```php
// Wait up to 30 seconds for lock
if (!$this->lockManager->lock(self::LOCK_NAME, self::LOCK_TIMEOUT, 30)) {
    throw new \RuntimeException('Could not acquire lock within 30 seconds');
}
```

**3. Force unlock (dangerous):**
```php
// Only use if previous execution crashed without releasing lock
$this->lockManager->unlock(self::LOCK_NAME);
$this->lockManager->lock(self::LOCK_NAME, self::LOCK_TIMEOUT);
```

### Chunked Processing for Large Datasets

**Cron/ProcessPendingOrders.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Cron;

use Magento\Framework\App\ResourceConnection;
use Psr\Log\LoggerInterface;
use Vendor\Module\Model\OrderProcessor;

/**
 * Process pending orders in batches
 */
class ProcessPendingOrders
{
    private const BATCH_SIZE = 100;
    private const MAX_EXECUTION_TIME = 300; // 5 minutes

    public function __construct(
        private readonly ResourceConnection $resourceConnection,
        private readonly OrderProcessor $orderProcessor,
        private readonly LoggerInterface $logger
    ) {}

    /**
     * Execute cron job
     *
     * @return void
     */
    public function execute(): void
    {
        $startTime = time();
        $totalProcessed = 0;

        $this->logger->info('Starting pending order processing');

        try {
            while (true) {
                // Check execution time limit
                if (time() - $startTime >= self::MAX_EXECUTION_TIME) {
                    $this->logger->info('Execution time limit reached, stopping processing', [
                        'total_processed' => $totalProcessed,
                        'execution_time' => time() - $startTime,
                    ]);
                    break;
                }

                // Fetch batch of pending orders
                $orders = $this->fetchPendingOrders(self::BATCH_SIZE);

                if (empty($orders)) {
                    $this->logger->info('No more pending orders to process');
                    break;
                }

                // Process batch
                $batchProcessed = $this->processBatch($orders);
                $totalProcessed += $batchProcessed;

                $this->logger->debug('Processed batch of orders', [
                    'batch_size' => count($orders),
                    'processed' => $batchProcessed,
                    'total_processed' => $totalProcessed,
                ]);
            }

            $this->logger->info('Pending order processing completed', [
                'total_processed' => $totalProcessed,
                'execution_time' => time() - $startTime,
            ]);

        } catch (\Exception $e) {
            $this->logger->error('Pending order processing failed', [
                'exception' => $e->getMessage(),
                'total_processed' => $totalProcessed,
            ]);

            throw $e;
        }
    }

    /**
     * Fetch batch of pending orders
     *
     * @param int $limit
     * @return array
     */
    private function fetchPendingOrders(int $limit): array
    {
        $connection = $this->resourceConnection->getConnection();

        $select = $connection->select()
            ->from('sales_order', ['entity_id'])
            ->where('status = ?', 'pending_processing')
            ->order('created_at ASC')
            ->limit($limit);

        return $connection->fetchCol($select);
    }

    /**
     * Process batch of orders
     *
     * @param array $orderIds
     * @return int Number of successfully processed orders
     */
    private function processBatch(array $orderIds): int
    {
        $processed = 0;

        foreach ($orderIds as $orderId) {
            try {
                $this->orderProcessor->process($orderId);
                $processed++;
            } catch (\Exception $e) {
                $this->logger->error('Failed to process order', [
                    'order_id' => $orderId,
                    'exception' => $e->getMessage(),
                ]);
                // Continue processing other orders
            }
        }

        return $processed;
    }
}
```

**Chunking strategies:**

**Time-based:**
- Stop after fixed duration (5 minutes)
- Ensures cron completes before next run
- Remaining work handled by next execution

**Count-based:**
- Process fixed number of items (e.g., 1000)
- Predictable execution time
- May leave work incomplete if limit reached

**Hybrid:**
- Process until time limit OR item limit reached
- Most flexible approach

### Long-Running Jobs with Status Tracking

For jobs requiring >1 hour, use status table:

**etc/db_schema.xml**

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    <table name="vendor_module_job_status" resource="default" engine="innodb" comment="Cron Job Status Tracking">
        <column xsi:type="int" name="id" unsigned="true" nullable="false" identity="true" comment="ID"/>
        <column xsi:type="varchar" name="job_code" length="255" nullable="false" comment="Job Code"/>
        <column xsi:type="varchar" name="status" length="50" nullable="false" default="pending" comment="Status"/>
        <column xsi:type="int" name="total_items" unsigned="true" nullable="false" default="0" comment="Total Items"/>
        <column xsi:type="int" name="processed_items" unsigned="true" nullable="false" default="0" comment="Processed Items"/>
        <column xsi:type="text" name="error_message" nullable="true" comment="Error Message"/>
        <column xsi:type="timestamp" name="started_at" nullable="true" comment="Started At"/>
        <column xsi:type="timestamp" name="completed_at" nullable="true" comment="Completed At"/>
        <column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP" comment="Created At"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="id"/>
        </constraint>

        <index referenceId="IDX_JOB_CODE_STATUS" indexType="btree">
            <column name="job_code"/>
            <column name="status"/>
        </index>
    </table>
</schema>
```

**Cron/LongRunningImport.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Cron;

use Magento\Framework\App\ResourceConnection;
use Psr\Log\LoggerInterface;
use Vendor\Module\Model\ImportProcessor;

class LongRunningImport
{
    private const JOB_CODE = 'long_running_import';
    private const BATCH_SIZE = 100;
    private const MAX_EXECUTION_TIME = 840; // 14 minutes (leave buffer before next cron run)

    public function __construct(
        private readonly ResourceConnection $resourceConnection,
        private readonly ImportProcessor $importProcessor,
        private readonly LoggerInterface $logger
    ) {}

    public function execute(): void
    {
        $connection = $this->resourceConnection->getConnection();

        // Find or create job status record
        $jobStatus = $this->getOrCreateJobStatus();

        if ($jobStatus['status'] === 'completed') {
            $this->logger->info('Import already completed, creating new job');
            $jobStatus = $this->createNewJobStatus();
        }

        if ($jobStatus['status'] === 'pending') {
            $this->updateJobStatus($jobStatus['id'], 'running', ['started_at' => date('Y-m-d H:i:s')]);
        }

        $startTime = time();

        try {
            while (true) {
                if (time() - $startTime >= self::MAX_EXECUTION_TIME) {
                    $this->logger->info('Time limit reached, pausing job', [
                        'job_id' => $jobStatus['id'],
                        'processed' => $jobStatus['processed_items'],
                    ]);
                    break;
                }

                $batch = $this->fetchNextBatch($jobStatus['processed_items'], self::BATCH_SIZE);

                if (empty($batch)) {
                    // All items processed
                    $this->updateJobStatus($jobStatus['id'], 'completed', [
                        'completed_at' => date('Y-m-d H:i:s'),
                    ]);

                    $this->logger->info('Import completed', [
                        'total_processed' => $jobStatus['processed_items'],
                    ]);
                    break;
                }

                $this->importProcessor->processBatch($batch);

                // Update progress
                $jobStatus['processed_items'] += count($batch);
                $this->updateJobStatus($jobStatus['id'], 'running', [
                    'processed_items' => $jobStatus['processed_items'],
                ]);
            }

        } catch (\Exception $e) {
            $this->updateJobStatus($jobStatus['id'], 'failed', [
                'error_message' => $e->getMessage(),
            ]);

            $this->logger->error('Import failed', [
                'job_id' => $jobStatus['id'],
                'exception' => $e->getMessage(),
            ]);

            throw $e;
        }
    }

    private function getOrCreateJobStatus(): array
    {
        $connection = $this->resourceConnection->getConnection();

        $select = $connection->select()
            ->from('vendor_module_job_status')
            ->where('job_code = ?', self::JOB_CODE)
            ->order('id DESC')
            ->limit(1);

        $status = $connection->fetchRow($select);

        if ($status) {
            return $status;
        }

        return $this->createNewJobStatus();
    }

    private function createNewJobStatus(): array
    {
        $connection = $this->resourceConnection->getConnection();

        $totalItems = $this->importProcessor->getTotalItemCount();

        $data = [
            'job_code' => self::JOB_CODE,
            'status' => 'pending',
            'total_items' => $totalItems,
            'processed_items' => 0,
        ];

        $connection->insert('vendor_module_job_status', $data);

        $data['id'] = $connection->lastInsertId();

        return $data;
    }

    private function updateJobStatus(int $id, string $status, array $additionalData = []): void
    {
        $connection = $this->resourceConnection->getConnection();

        $data = array_merge(['status' => $status], $additionalData);

        $connection->update(
            'vendor_module_job_status',
            $data,
            ['id = ?' => $id]
        );
    }

    private function fetchNextBatch(int $offset, int $limit): array
    {
        return $this->importProcessor->fetchBatch($offset, $limit);
    }
}
```

**Benefits:**
- Job survives multiple cron runs
- Progress tracked in database
- Admin can monitor via UI
- Failed jobs can be retried from last position

## Cron Group Management

### Creating Custom Cron Group

**etc/cron_groups.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/cron_groups.xsd">
    <group id="vendor_heavy_processing">
        <schedule_generate_every>15</schedule_generate_every>
        <schedule_ahead_for>60</schedule_ahead_for>
        <schedule_lifetime>120</schedule_lifetime>
        <history_cleanup_every>10</history_cleanup_every>
        <history_success_lifetime>1440</history_success_lifetime>
        <history_failure_lifetime>10080</history_failure_lifetime>
        <use_separate_process>1</use_separate_process>
    </group>
</config>
```

**Configuration options:**

- `schedule_generate_every`: Generate schedules every N minutes (default: 1)
- `schedule_ahead_for`: Generate schedules N minutes ahead (default: 4)
- `schedule_lifetime`: Maximum minutes a schedule can be pending (default: 2)
- `history_cleanup_every`: Cleanup history every N minutes (default: 10)
- `history_success_lifetime`: Keep successful jobs N minutes (default: 10080 = 7 days)
- `history_failure_lifetime`: Keep failed jobs N minutes (default: 10080 = 7 days)
- `use_separate_process`: Run group in separate process (0 or 1)

### Assigning Jobs to Custom Group

**etc/crontab.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/crontab.xsd">
    <group id="vendor_heavy_processing">
        <job name="vendor_module_data_aggregation" instance="Vendor\Module\Cron\DataAggregation" method="execute">
            <schedule>0 */6 * * *</schedule><!-- Every 6 hours -->
        </job>
        <job name="vendor_module_report_generation" instance="Vendor\Module\Cron\ReportGeneration" method="execute">
            <schedule>0 3 * * *</schedule><!-- Daily at 3 AM -->
        </job>
    </group>
</config>
```

### Running Specific Cron Group

```bash
# Run only specific group
bin/magento cron:run --group="vendor_heavy_processing"

# Run all groups
bin/magento cron:run

# Run in background
bin/magento cron:run --group="vendor_heavy_processing" > /dev/null 2>&1 &
```

**Production setup (separate server for heavy jobs):**

```bash
# Server 1: Run default and index groups
* * * * * /usr/bin/php /var/www/magento/bin/magento cron:run --group=default
* * * * * /usr/bin/php /var/www/magento/bin/magento cron:run --group=index

# Server 2: Run heavy processing group
* * * * * /usr/bin/php /var/www/magento/bin/magento cron:run --group=vendor_heavy_processing
```

## Error Handling and Logging

### Comprehensive Error Handling

**Cron/RobustExample.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Cron;

use Magento\Framework\Exception\LocalizedException;
use Psr\Log\LoggerInterface;
use Vendor\Module\Model\DataProcessor;

class RobustExample
{
    public function __construct(
        private readonly DataProcessor $dataProcessor,
        private readonly LoggerInterface $logger
    ) {}

    public function execute(): void
    {
        $context = ['job' => 'robust_example'];

        $this->logger->info('Job started', $context);

        try {
            $result = $this->dataProcessor->process();

            if ($result->hasErrors()) {
                $this->handlePartialFailure($result, $context);
                return;
            }

            $this->logger->info('Job completed successfully', array_merge($context, [
                'records_processed' => $result->getProcessedCount(),
            ]));

        } catch (LocalizedException $e) {
            // Business logic exception (expected, recoverable)
            $this->logger->warning('Job failed with expected error', array_merge($context, [
                'error' => $e->getMessage(),
            ]));

            // Don't re-throw: Job completes with warning, won't retry

        } catch (\InvalidArgumentException $e) {
            // Configuration/input error (permanent)
            $this->logger->error('Job failed due to configuration error', array_merge($context, [
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString(),
            ]));

            // Don't re-throw: Fix configuration, retrying won't help

        } catch (\Exception $e) {
            // Unexpected exception (infrastructure, database, etc.)
            $this->logger->critical('Job failed with unexpected error', array_merge($context, [
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString(),
            ]));

            // Re-throw: Mark job as failed, enable retry mechanism
            throw $e;
        }
    }

    private function handlePartialFailure($result, array $context): void
    {
        $this->logger->warning('Job completed with partial failures', array_merge($context, [
            'records_processed' => $result->getProcessedCount(),
            'records_failed' => $result->getFailedCount(),
            'errors' => $result->getErrors(),
        ]));
    }
}
```

**Error handling strategy:**

| Exception Type | Log Level | Re-throw? | Reason |
|----------------|-----------|-----------|--------|
| Expected business errors | WARNING | No | Normal operation, no retry needed |
| Configuration errors | ERROR | No | Manual fix required |
| Infrastructure errors | CRITICAL | Yes | Temporary, may succeed on retry |

### Custom Logger for Cron Jobs

**etc/di.xml**

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">

    <!-- Custom logger for cron jobs -->
    <virtualType name="VendorCronLogger" type="Magento\Framework\Logger\Monolog">
        <arguments>
            <argument name="name" xsi:type="string">vendorCron</argument>
            <argument name="handlers" xsi:type="array">
                <item name="system" xsi:type="object">VendorCronLogHandler</item>
            </argument>
        </arguments>
    </virtualType>

    <virtualType name="VendorCronLogHandler" type="Magento\Framework\Logger\Handler\Base">
        <arguments>
            <argument name="fileName" xsi:type="string">/var/log/vendor_cron.log</argument>
        </arguments>
    </virtualType>

    <!-- Inject custom logger into cron job -->
    <type name="Vendor\Module\Cron\CleanupTempData">
        <arguments>
            <argument name="logger" xsi:type="object">VendorCronLogger</argument>
        </arguments>
    </type>

</config>
```

**Benefits:**
- Separate log file for cron jobs
- Easier debugging and monitoring
- Different retention policies per log

## Testing Cron Jobs

### Manual Testing

**1. Generate schedule entries:**
```bash
# Generate schedules for all groups
bin/magento cron:run --group=default

# Check schedules created
mysql> SELECT job_code, status, scheduled_at, executed_at FROM cron_schedule WHERE job_code = 'vendor_module_cleanup_temp_data' ORDER BY scheduled_at DESC LIMIT 10;
```

**2. Run specific job:**
```bash
# Trigger all pending jobs
bin/magento cron:run

# Or run specific group
bin/magento cron:run --group=default
```

**3. Check execution:**
```bash
# View job status
mysql> SELECT job_code, status, messages, executed_at FROM cron_schedule WHERE job_code = 'vendor_module_cleanup_temp_data' ORDER BY schedule_id DESC LIMIT 1;

# View logs
tail -f var/log/system.log | grep vendor_module_cleanup
```

### Unit Testing

**Test/Unit/Cron/CleanupTempDataTest.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Unit\Cron;

use PHPUnit\Framework\TestCase;
use Psr\Log\LoggerInterface;
use Vendor\Module\Cron\CleanupTempData;
use Vendor\Module\Model\TempDataRepository;

class CleanupTempDataTest extends TestCase
{
    private CleanupTempData $cronJob;
    private TempDataRepository $tempDataRepository;
    private LoggerInterface $logger;

    protected function setUp(): void
    {
        $this->tempDataRepository = $this->createMock(TempDataRepository::class);
        $this->logger = $this->createMock(LoggerInterface::class);

        $this->cronJob = new CleanupTempData(
            $this->tempDataRepository,
            $this->logger
        );
    }

    public function testExecuteSuccessfully(): void
    {
        $this->tempDataRepository
            ->expects($this->once())
            ->method('deleteOlderThan')
            ->with(30)
            ->willReturn(150);

        $this->logger
            ->expects($this->exactly(2))
            ->method('info');

        $this->cronJob->execute();
    }

    public function testExecuteHandlesException(): void
    {
        $this->tempDataRepository
            ->expects($this->once())
            ->method('deleteOlderThan')
            ->willThrowException(new \RuntimeException('Database error'));

        $this->logger
            ->expects($this->once())
            ->method('error');

        $this->expectException(\RuntimeException::class);

        $this->cronJob->execute();
    }
}
```

### Integration Testing

**Test/Integration/Cron/CleanupTempDataTest.php**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Test\Integration\Cron;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;
use Vendor\Module\Cron\CleanupTempData;

class CleanupTempDataTest extends TestCase
{
    private CleanupTempData $cronJob;

    protected function setUp(): void
    {
        $objectManager = Bootstrap::getObjectManager();
        $this->cronJob = $objectManager->get(CleanupTempData::class);
    }

    public function testExecuteDeletesOldRecords(): void
    {
        // Create test data
        $this->createTestData();

        // Execute cron
        $this->cronJob->execute();

        // Assert old records deleted
        $this->assertOldRecordsDeleted();
    }

    private function createTestData(): void
    {
        // Insert test records with old timestamps
    }

    private function assertOldRecordsDeleted(): void
    {
        // Verify cleanup occurred
    }
}
```

### Testing Cron Schedule Generation

```php
public function testCronScheduleIsGenerated(): void
{
    $objectManager = Bootstrap::getObjectManager();

    /** @var \Magento\Cron\Model\Schedule $schedule */
    $schedule = $objectManager->create(\Magento\Cron\Model\Schedule::class);

    // Generate schedules
    $schedule->generate();

    // Check schedule exists
    $scheduleCollection = $objectManager->create(\Magento\Cron\Model\ResourceModel\Schedule\Collection::class);
    $scheduleCollection->addFieldToFilter('job_code', 'vendor_module_cleanup_temp_data');

    $this->assertGreaterThan(0, $scheduleCollection->getSize());
}
```

## Production Deployment

### Server Crontab Setup

**Option 1: All groups in one cron entry:**
```bash
# Run every minute
* * * * * /usr/bin/php /var/www/magento/bin/magento cron:run 2>&1 | grep -v "Ran jobs by schedule" >> /var/www/magento/var/log/magento.cron.log
```

**Option 2: Separate entries per group:**
```bash
# Default group
* * * * * /usr/bin/php /var/www/magento/bin/magento cron:run --group=default >> /var/www/magento/var/log/cron_default.log 2>&1

# Index group
* * * * * /usr/bin/php /var/www/magento/bin/magento cron:run --group=index >> /var/www/magento/var/log/cron_index.log 2>&1

# Consumers group (for message queues)
* * * * * /usr/bin/php /var/www/magento/bin/magento cron:run --group=consumers >> /var/www/magento/var/log/cron_consumers.log 2>&1
```

**Option 3: Supervisor (recommended for production):**

**/etc/supervisor/conf.d/magento-cron.conf**

```ini
[program:magento-cron]
command=/usr/bin/php /var/www/magento/bin/magento cron:run
autostart=true
autorestart=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/magento/var/log/cron.log
```

### Monitoring Cron Health

**Check cron is running:**
```bash
# Last execution time
mysql> SELECT MAX(executed_at) FROM cron_schedule WHERE status = 'success';

# Pending jobs
mysql> SELECT job_code, COUNT(*) FROM cron_schedule WHERE status = 'pending' GROUP BY job_code;

# Failed jobs (last hour)
mysql> SELECT job_code, messages FROM cron_schedule WHERE status = 'error' AND executed_at > DATE_SUB(NOW(), INTERVAL 1 HOUR);
```

**Monitoring script:**

**bin/check-cron-health.php**

```php
#!/usr/bin/env php
<?php

require __DIR__ . '/../app/bootstrap.php';

$bootstrap = \Magento\Framework\App\Bootstrap::create(BP, $_SERVER);
$objectManager = $bootstrap->getObjectManager();

$resource = $objectManager->get(\Magento\Framework\App\ResourceConnection::class);
$connection = $resource->getConnection();

// Check last successful execution
$lastSuccess = $connection->fetchOne(
    "SELECT MAX(executed_at) FROM cron_schedule WHERE status = 'success'"
);

if (!$lastSuccess || strtotime($lastSuccess) < time() - 300) {
    echo "CRITICAL: No successful cron jobs in last 5 minutes\n";
    exit(2);
}

// Check for stuck jobs
$stuck = $connection->fetchOne(
    "SELECT COUNT(*) FROM cron_schedule WHERE status = 'running' AND executed_at < DATE_SUB(NOW(), INTERVAL 1 HOUR)"
);

if ($stuck > 0) {
    echo "WARNING: $stuck jobs stuck in 'running' state\n";
    exit(1);
}

// Check for high failure rate
$recentTotal = $connection->fetchOne(
    "SELECT COUNT(*) FROM cron_schedule WHERE executed_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)"
);

$recentFailed = $connection->fetchOne(
    "SELECT COUNT(*) FROM cron_schedule WHERE status = 'error' AND executed_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)"
);

if ($recentTotal > 0 && ($recentFailed / $recentTotal) > 0.1) {
    echo "WARNING: High failure rate: $recentFailed / $recentTotal in last hour\n";
    exit(1);
}

echo "OK: Cron system healthy\n";
exit(0);
```

**Integrate with monitoring system:**
```bash
# Nagios, Icinga, etc.
*/5 * * * * /var/www/magento/bin/check-cron-health.php
```

---

## Assumptions

- **Target versions**: Magento 2.4.7+, PHP 8.2+
- **Deployment**: Linux server with cron daemon installed
- **Infrastructure**: Adequate resources for concurrent cron execution
- **Monitoring**: Logging and alerting infrastructure in place

## Why This Approach

**crontab.xml**: Centralized, version-controlled cron configuration

**Cron groups**: Resource isolation, enables scaling across servers

**Locking**: Prevents race conditions and duplicate processing

**Chunked processing**: Memory-efficient, respects execution time limits

**Status tracking**: Enables long-running jobs spanning multiple executions

**Custom loggers**: Separate cron logs for easier debugging

## Security Impact

**Authorization:** Cron jobs run as PHP user, inherit filesystem permissions

**CLI access:** Cron execution requires PHP CLI access (secure server access)

**Sensitive data:** Avoid logging credentials, PII, or payment data

**Resource limits:** PHP memory_limit and max_execution_time apply to cron jobs

## Performance Impact

**System resources:**
- Each cron group = separate PHP process
- Peak load: 3-5 groups × 1-2 minutes execution = high CPU/memory spikes
- Stagger heavy jobs (run at different times)

**Database impact:**
- `cron_schedule` table grows continuously (cleanup via history settings)
- Concurrent cron jobs may lock database tables
- Use row-level locking (InnoDB) to minimize contention

**Scalability:**
- Single server: 10-20 concurrent jobs (depends on resources)
- Multi-server: Run groups on separate servers
- Message queues: Offload heavy processing to consumers

## Backward Compatibility

**API stability:**
- Cron configuration XML schema stable across Magento 2.4.x
- `cron:run` command stable

**Database schema:**
- `cron_schedule` table structure stable
- Job status tracking via custom tables (safe)

**Upgrade path:**
- Magento 2.4.7 → 2.4.8: No breaking changes expected
- Magento 2.4.x → 2.4.9: Monitor deprecation notices

## Tests to Add

**Unit tests:**
- Cron job execute method logic
- Error handling paths
- Locking behavior (mock LockManager)

**Integration tests:**
- Cron schedule generation
- Job execution with real dependencies
- Cleanup of old schedules

**Functional tests:**
- End-to-end job execution
- Long-running job status tracking
- Failure recovery

## Documentation to Update

**Developer documentation:**
- `README.md`: Cron architecture overview
- `CRON.md`: Custom cron job development guide
- `MONITORING.md`: Cron health monitoring setup

**Operations documentation:**
- Server crontab configuration
- Supervisor setup for production
- Troubleshooting stuck jobs
- Log rotation and retention policies

## Related Documentation

### Related Guides

- [Message Queue Architecture in Magento 2](../explanations/message-queue-architecture.md)
- [Indexer System Deep Dive: Understanding Magento 2's Data Indexing Architecture](../explanations/indexer-system.md)
- [ERP Integration Patterns for Magento 2](erp-integration.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
