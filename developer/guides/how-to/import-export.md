---
title: "Import/Export System Deep Dive"
description: "Master Magento's import/export architecture for custom entity imports, scheduled imports, large file handling, validation strategies, and performance optimization."
type: "how-to"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "50 minutes"
topics:
  - import/export
  - custom entities
  - scheduled imports
  - data validation
  - performance optimization
  - csv processing
last_updated: "2026-02-07"
---

# Import/Export System Deep Dive

## Overview

Magento's import/export system provides a robust framework for bulk data operations. Understanding this architecture is critical for custom entity imports, third-party integrations, and large-scale catalog management.

**What you'll learn:**
- Import/export architecture and the adapter/entity/behavior pattern
- Creating custom entity importers with validation and error handling
- Scheduled imports with cron integration
- Large file handling strategies (chunking, streaming, memory optimization)
- Advanced validation techniques and error recovery
- Performance optimization for million-row imports
- Export customization and custom export entities

**Prerequisites:**
- Magento 2.4.7+ (Adobe Commerce or Open Source)
- Understanding of Magento module structure and dependency injection
- Experience with CSV file processing and data validation
- Knowledge of indexing and cache management
- Familiarity with cron system

---

## Import/Export Architecture

### Core Components

1. **Import Model**: `Magento\ImportExport\Model\Import` - Main import orchestrator
2. **Entity Adapters**: `Magento\ImportExport\Model\Import\Entity\AbstractEntity` - Per-entity logic
3. **Source Models**: `Magento\ImportExport\Model\Import\AbstractSource` - File reading abstraction
4. **Behavior Models**: Add, update, delete, or replace strategies
5. **Validation**: `Magento\ImportExport\Model\Import\ErrorProcessing\ProcessingErrorAggregator`
6. **Export Model**: `Magento\ImportExport\Model\Export` - Export orchestrator

### Import Flow

```
[CSV Upload] → [Source Model] → [Validation]
       ↓
[Entity Adapter] → [Behavior (Add/Update/Delete)]
       ↓
[Data Processing] → [Database Write] → [Indexing]
       ↓
[Error Aggregation] → [Report Generation]
```

### Entity Adapter Pattern

Each importable entity (products, customers, stock sources) has an adapter that:
- Validates row structure and data integrity
- Maps CSV columns to entity attributes
- Handles add/update/delete/replace behaviors
- Manages relationships (categories, links, options)

---

## Creating a Custom Entity Importer

### Use Case: Warranty Registration Import

We'll build an importer for warranty registrations that validates product SKU, customer email, purchase date, and warranty details.

### Step 1: Register Import Entity

**File**: `Vendor/WarrantyImport/etc/import.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_ImportExport:etc/import.xsd">

    <entity name="warranty_registration"
            label="Warranty Registrations"
            model="Vendor\WarrantyImport\Model\Import\WarrantyRegistration"
            behaviorModel="Vendor\WarrantyImport\Model\Import\WarrantyRegistration\Behavior"
            entityTypeCode="warranty_registration">
    </entity>
</config>
```

### Step 2: Create Entity Adapter

**File**: `Vendor/WarrantyImport/Model/Import/WarrantyRegistration.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\WarrantyImport\Model\Import;

use Magento\ImportExport\Model\Import;
use Magento\ImportExport\Model\Import\Entity\AbstractEntity;
use Magento\ImportExport\Model\Import\ErrorProcessing\ProcessingErrorAggregatorInterface;
use Magento\Framework\App\ResourceConnection;
use Vendor\WarrantyImport\Model\ResourceModel\Warranty as WarrantyResource;
use Psr\Log\LoggerInterface;

class WarrantyRegistration extends AbstractEntity
{
    public const ENTITY_CODE = 'warranty_registration';
    public const TABLE = 'warranty_registration';

    /**
     * Required columns
     */
    public const COL_EMAIL = 'customer_email';
    public const COL_SKU = 'product_sku';
    public const COL_PURCHASE_DATE = 'purchase_date';
    public const COL_WARRANTY_CODE = 'warranty_code';
    public const COL_SERIAL_NUMBER = 'serial_number';

    /**
     * Optional columns
     */
    public const COL_PURCHASE_PRICE = 'purchase_price';
    public const COL_RETAILER = 'retailer';
    public const COL_NOTES = 'notes';

    /**
     * Validation errors
     */
    private const ERROR_DUPLICATE_WARRANTY = 'duplicateWarranty';
    private const ERROR_INVALID_EMAIL = 'invalidEmail';
    private const ERROR_INVALID_SKU = 'invalidSku';
    private const ERROR_INVALID_DATE = 'invalidDate';
    private const ERROR_MISSING_REQUIRED = 'missingRequired';

    protected $validColumnNames = [
        self::COL_EMAIL,
        self::COL_SKU,
        self::COL_PURCHASE_DATE,
        self::COL_WARRANTY_CODE,
        self::COL_SERIAL_NUMBER,
        self::COL_PURCHASE_PRICE,
        self::COL_RETAILER,
        self::COL_NOTES,
    ];

    protected $needColumnCheck = true;
    protected $logInHistory = true;
    protected $_permanentAttributes = [self::COL_WARRANTY_CODE];

    private array $validatedRows = [];
    private array $invalidRows = [];
    private array $existingWarranties = [];

    public function __construct(
        \Magento\Framework\Json\Helper\Data $jsonHelper,
        \Magento\ImportExport\Helper\Data $importExportData,
        \Magento\ImportExport\Model\ResourceModel\Import\Data $importData,
        ResourceConnection $resource,
        \Magento\ImportExport\Model\ResourceModel\Helper $resourceHelper,
        ProcessingErrorAggregatorInterface $errorAggregator,
        private readonly WarrantyResource $warrantyResource,
        private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
        private readonly \Magento\Customer\Api\CustomerRepositoryInterface $customerRepository,
        private readonly LoggerInterface $logger
    ) {
        parent::__construct(
            $jsonHelper,
            $importExportData,
            $importData,
            $resource,
            $resourceHelper,
            $errorAggregator
        );
    }

    /**
     * Entity type code getter
     */
    public function getEntityTypeCode(): string
    {
        return self::ENTITY_CODE;
    }

    /**
     * Validate row data
     */
    public function validateRow(array $rowData, $rowNum): bool
    {
        // Check if row already validated
        if (isset($this->validatedRows[$rowNum])) {
            return !$this->getErrorAggregator()->isRowInvalid($rowNum);
        }

        $this->validatedRows[$rowNum] = true;

        // Validate required columns
        if (!$this->validateRequiredColumns($rowData, $rowNum)) {
            return false;
        }

        // Validate email format
        if (!$this->validateEmail($rowData, $rowNum)) {
            return false;
        }

        // Validate product SKU exists
        if (!$this->validateProductSku($rowData, $rowNum)) {
            return false;
        }

        // Validate date format
        if (!$this->validateDate($rowData, $rowNum)) {
            return false;
        }

        // Check for duplicate warranty codes
        if (!$this->validateUniqueWarranty($rowData, $rowNum)) {
            return false;
        }

        return !$this->getErrorAggregator()->isRowInvalid($rowNum);
    }

    /**
     * Validate required columns
     */
    private function validateRequiredColumns(array $rowData, int $rowNum): bool
    {
        $requiredColumns = [
            self::COL_EMAIL,
            self::COL_SKU,
            self::COL_PURCHASE_DATE,
            self::COL_WARRANTY_CODE,
        ];

        foreach ($requiredColumns as $column) {
            if (empty($rowData[$column])) {
                $this->addRowError(
                    self::ERROR_MISSING_REQUIRED,
                    $rowNum,
                    $column
                );
                return false;
            }
        }

        return true;
    }

    /**
     * Validate email format
     */
    private function validateEmail(array $rowData, int $rowNum): bool
    {
        $email = $rowData[self::COL_EMAIL];

        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            $this->addRowError(
                self::ERROR_INVALID_EMAIL,
                $rowNum,
                self::COL_EMAIL,
                null,
                null,
                sprintf('Invalid email format: %s', $email)
            );
            return false;
        }

        return true;
    }

    /**
     * Validate product SKU exists
     */
    private function validateProductSku(array $rowData, int $rowNum): bool
    {
        $sku = $rowData[self::COL_SKU];

        try {
            $this->productRepository->get($sku);
            return true;
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            $this->addRowError(
                self::ERROR_INVALID_SKU,
                $rowNum,
                self::COL_SKU,
                null,
                null,
                sprintf('Product SKU not found: %s', $sku)
            );
            return false;
        }
    }

    /**
     * Validate date format
     */
    private function validateDate(array $rowData, int $rowNum): bool
    {
        $dateString = $rowData[self::COL_PURCHASE_DATE];

        try {
            $date = new \DateTime($dateString);

            // Check if date is not in future
            $now = new \DateTime();
            if ($date > $now) {
                $this->addRowError(
                    self::ERROR_INVALID_DATE,
                    $rowNum,
                    self::COL_PURCHASE_DATE,
                    null,
                    null,
                    'Purchase date cannot be in the future'
                );
                return false;
            }

            return true;

        } catch (\Exception $e) {
            $this->addRowError(
                self::ERROR_INVALID_DATE,
                $rowNum,
                self::COL_PURCHASE_DATE,
                null,
                null,
                sprintf('Invalid date format: %s', $dateString)
            );
            return false;
        }
    }

    /**
     * Check for duplicate warranty codes
     */
    private function validateUniqueWarranty(array $rowData, int $rowNum): bool
    {
        $warrantyCode = $rowData[self::COL_WARRANTY_CODE];

        // Load existing warranties on first validation
        if (empty($this->existingWarranties)) {
            $this->existingWarranties = $this->loadExistingWarranties();
        }

        if (isset($this->existingWarranties[$warrantyCode])) {
            $this->addRowError(
                self::ERROR_DUPLICATE_WARRANTY,
                $rowNum,
                self::COL_WARRANTY_CODE,
                null,
                null,
                sprintf('Warranty code already exists: %s', $warrantyCode)
            );
            return false;
        }

        return true;
    }

    /**
     * Load existing warranty codes
     */
    private function loadExistingWarranties(): array
    {
        $connection = $this->_connection;
        $select = $connection->select()
            ->from($this->warrantyResource->getMainTable(), ['warranty_code'])
            ->where('warranty_code IS NOT NULL');

        $result = $connection->fetchCol($select);

        return array_fill_keys($result, true);
    }

    /**
     * Import data
     */
    protected function _importData(): bool
    {
        switch ($this->getBehavior()) {
            case Import::BEHAVIOR_DELETE:
                $this->deleteWarranties();
                break;
            case Import::BEHAVIOR_REPLACE:
                $this->replaceWarranties();
                break;
            case Import::BEHAVIOR_APPEND:
            default:
                $this->saveWarranties();
                break;
        }

        return true;
    }

    /**
     * Save warranty data
     */
    private function saveWarranties(): void
    {
        $warrantiesToInsert = [];
        $warrantiesToUpdate = [];

        while ($bunch = $this->_dataSourceModel->getNextBunch()) {
            foreach ($bunch as $rowNum => $rowData) {
                // Skip invalid rows
                if (!$this->validateRow($rowData, $rowNum)) {
                    continue;
                }

                // Prepare warranty data
                $warranty = $this->prepareWarrantyData($rowData);

                // Check if warranty exists (for update)
                if (isset($this->existingWarranties[$warranty['warranty_code']])) {
                    $warrantiesToUpdate[] = $warranty;
                } else {
                    $warrantiesToInsert[] = $warranty;
                }

                // Process in batches
                if (count($warrantiesToInsert) >= $this->getBatchSize()) {
                    $this->insertWarranties($warrantiesToInsert);
                    $warrantiesToInsert = [];
                }

                if (count($warrantiesToUpdate) >= $this->getBatchSize()) {
                    $this->updateWarranties($warrantiesToUpdate);
                    $warrantiesToUpdate = [];
                }
            }
        }

        // Process remaining
        if (!empty($warrantiesToInsert)) {
            $this->insertWarranties($warrantiesToInsert);
        }

        if (!empty($warrantiesToUpdate)) {
            $this->updateWarranties($warrantiesToUpdate);
        }
    }

    /**
     * Prepare warranty data for insertion
     */
    private function prepareWarrantyData(array $rowData): array
    {
        // Get customer ID from email
        $customerId = $this->getCustomerIdByEmail($rowData[self::COL_EMAIL]);

        // Get product ID from SKU
        $productId = $this->getProductIdBySku($rowData[self::COL_SKU]);

        return [
            'warranty_code' => $rowData[self::COL_WARRANTY_CODE],
            'customer_id' => $customerId,
            'customer_email' => $rowData[self::COL_EMAIL],
            'product_id' => $productId,
            'product_sku' => $rowData[self::COL_SKU],
            'serial_number' => $rowData[self::COL_SERIAL_NUMBER] ?? null,
            'purchase_date' => $rowData[self::COL_PURCHASE_DATE],
            'purchase_price' => $rowData[self::COL_PURCHASE_PRICE] ?? null,
            'retailer' => $rowData[self::COL_RETAILER] ?? null,
            'notes' => $rowData[self::COL_NOTES] ?? null,
            'created_at' => date('Y-m-d H:i:s'),
        ];
    }

    /**
     * Insert warranty records
     */
    private function insertWarranties(array $warranties): void
    {
        if (empty($warranties)) {
            return;
        }

        try {
            $this->_connection->insertMultiple(
                $this->warrantyResource->getMainTable(),
                $warranties
            );

            $this->countItemsCreated += count($warranties);

        } catch (\Exception $e) {
            $this->logger->error('Warranty import insert failed', [
                'exception' => $e->getMessage(),
                'count' => count($warranties)
            ]);
        }
    }

    /**
     * Update warranty records
     */
    private function updateWarranties(array $warranties): void
    {
        if (empty($warranties)) {
            return;
        }

        try {
            $this->_connection->insertOnDuplicate(
                $this->warrantyResource->getMainTable(),
                $warranties,
                array_keys(reset($warranties))
            );

            $this->countItemsUpdated += count($warranties);

        } catch (\Exception $e) {
            $this->logger->error('Warranty import update failed', [
                'exception' => $e->getMessage(),
                'count' => count($warranties)
            ]);
        }
    }

    /**
     * Delete warranties
     */
    private function deleteWarranties(): void
    {
        $warrantyCodesToDelete = [];

        while ($bunch = $this->_dataSourceModel->getNextBunch()) {
            foreach ($bunch as $rowNum => $rowData) {
                if (!$this->validateRow($rowData, $rowNum)) {
                    continue;
                }

                $warrantyCodesToDelete[] = $rowData[self::COL_WARRANTY_CODE];

                if (count($warrantyCodesToDelete) >= $this->getBatchSize()) {
                    $this->deleteWarrantyBatch($warrantyCodesToDelete);
                    $warrantyCodesToDelete = [];
                }
            }
        }

        if (!empty($warrantyCodesToDelete)) {
            $this->deleteWarrantyBatch($warrantyCodesToDelete);
        }
    }

    /**
     * Delete batch of warranties
     */
    private function deleteWarrantyBatch(array $warrantyCodes): void
    {
        try {
            $this->_connection->delete(
                $this->warrantyResource->getMainTable(),
                $this->_connection->quoteInto('warranty_code IN (?)', $warrantyCodes)
            );

            $this->countItemsDeleted += count($warrantyCodes);

        } catch (\Exception $e) {
            $this->logger->error('Warranty delete failed', [
                'exception' => $e->getMessage(),
                'codes' => $warrantyCodes
            ]);
        }
    }

    /**
     * Replace warranties (delete all, then insert)
     */
    private function replaceWarranties(): void
    {
        // Delete all existing warranties
        try {
            $this->_connection->delete($this->warrantyResource->getMainTable());
        } catch (\Exception $e) {
            $this->logger->error('Warranty replace (delete all) failed', [
                'exception' => $e->getMessage()
            ]);
        }

        // Insert new data
        $this->saveWarranties();
    }

    /**
     * Get customer ID by email
     */
    private function getCustomerIdByEmail(string $email): ?int
    {
        try {
            $customer = $this->customerRepository->get($email);
            return (int)$customer->getId();
        } catch (\Exception $e) {
            return null;
        }
    }

    /**
     * Get product ID by SKU
     */
    private function getProductIdBySku(string $sku): ?int
    {
        try {
            $product = $this->productRepository->get($sku);
            return (int)$product->getId();
        } catch (\Exception $e) {
            return null;
        }
    }

    /**
     * Get batch size for processing
     */
    private function getBatchSize(): int
    {
        return 500;
    }
}
```

---

## Scheduled Imports

### Cron-Based Import Processor

**File**: `Vendor/WarrantyImport/Model/ScheduledImport.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\WarrantyImport\Model;

use Magento\Framework\App\Filesystem\DirectoryList;
use Magento\Framework\Filesystem;
use Magento\ImportExport\Model\Import;
use Magento\ImportExport\Model\Import\Source\Csv;
use Psr\Log\LoggerInterface;

class ScheduledImport
{
    private const IMPORT_DIR = 'import/warranty';

    public function __construct(
        private readonly Filesystem $filesystem,
        private readonly Import $importModel,
        private readonly LoggerInterface $logger
    ) {
    }

    /**
     * Execute scheduled import
     */
    public function execute(): void
    {
        $varDirectory = $this->filesystem->getDirectoryRead(DirectoryList::VAR_DIR);
        $importPath = $varDirectory->getAbsolutePath(self::IMPORT_DIR);

        if (!is_dir($importPath)) {
            $this->logger->info('Import directory does not exist: ' . $importPath);
            return;
        }

        $files = glob($importPath . '/*.csv');

        foreach ($files as $filePath) {
            try {
                $this->processImportFile($filePath);

                // Move processed file to archive
                $this->archiveFile($filePath);

            } catch (\Exception $e) {
                $this->logger->error('Scheduled import failed', [
                    'file' => $filePath,
                    'exception' => $e->getMessage()
                ]);

                // Move failed file to error directory
                $this->moveToErrorDirectory($filePath);
            }
        }
    }

    /**
     * Process single import file
     */
    private function processImportFile(string $filePath): void
    {
        $source = new Csv($filePath);

        $this->importModel->setData([
            'entity' => 'warranty_registration',
            'behavior' => Import::BEHAVIOR_APPEND,
            'validation_strategy' => Import::VALIDATION_STRATEGY_STOP_ON_ERROR,
        ]);

        $validationResult = $this->importModel->validateSource($source);

        if (!$validationResult) {
            $errors = $this->importModel->getErrorAggregator()->getAllErrors();
            throw new \RuntimeException('Validation failed: ' . implode(', ', $errors));
        }

        $result = $this->importModel->importSource();

        if (!$result) {
            throw new \RuntimeException('Import failed');
        }

        $this->logger->info('Import completed successfully', [
            'file' => basename($filePath),
            'created' => $this->importModel->getCreatedItemsCount(),
            'updated' => $this->importModel->getUpdatedItemsCount(),
            'deleted' => $this->importModel->getDeletedItemsCount(),
        ]);
    }

    /**
     * Archive processed file
     */
    private function archiveFile(string $filePath): void
    {
        $varDirectory = $this->filesystem->getDirectoryWrite(DirectoryList::VAR_DIR);
        $archivePath = $varDirectory->getAbsolutePath(self::IMPORT_DIR . '/archive');

        if (!is_dir($archivePath)) {
            mkdir($archivePath, 0775, true);
        }

        $fileName = basename($filePath);
        $archiveFileName = date('YmdHis') . '_' . $fileName;

        rename($filePath, $archivePath . '/' . $archiveFileName);
    }

    /**
     * Move file to error directory
     */
    private function moveToErrorDirectory(string $filePath): void
    {
        $varDirectory = $this->filesystem->getDirectoryWrite(DirectoryList::VAR_DIR);
        $errorPath = $varDirectory->getAbsolutePath(self::IMPORT_DIR . '/error');

        if (!is_dir($errorPath)) {
            mkdir($errorPath, 0775, true);
        }

        $fileName = basename($filePath);
        $errorFileName = date('YmdHis') . '_' . $fileName;

        rename($filePath, $errorPath . '/' . $errorFileName);
    }
}
```

**Register cron job:**

**File**: `Vendor/WarrantyImport/etc/crontab.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Cron:etc/crontab.xsd">
    <group id="default">
        <job name="warranty_scheduled_import" instance="Vendor\WarrantyImport\Cron\ScheduledImport" method="execute">
            <schedule>*/15 * * * *</schedule>
        </job>
    </group>
</config>
```

---

## Large File Handling

### Memory-Efficient CSV Reading

**File**: `Vendor/WarrantyImport/Model/Import/Source/StreamingCsv.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\WarrantyImport\Model\Import\Source;

use Magento\ImportExport\Model\Import\AbstractSource;

class StreamingCsv extends AbstractSource
{
    private const BATCH_SIZE = 1000;

    private $fileHandle;
    private array $colNames;
    private int $currentRow = 0;
    private array $buffer = [];

    public function __construct(
        string $file,
        string $delimiter = ',',
        string $enclosure = '"'
    ) {
        $this->fileHandle = fopen($file, 'r');

        if ($this->fileHandle === false) {
            throw new \RuntimeException('Unable to open file: ' . $file);
        }

        // Read header
        $header = fgetcsv($this->fileHandle, 0, $delimiter, $enclosure);

        if ($header === false) {
            throw new \RuntimeException('Unable to read CSV header');
        }

        $this->colNames = $header;

        // Pre-load first batch
        $this->loadNextBatch($delimiter, $enclosure);

        parent::__construct($this->colNames);
    }

    /**
     * Load next batch of rows into buffer
     */
    private function loadNextBatch(string $delimiter, string $enclosure): void
    {
        $this->buffer = [];
        $count = 0;

        while ($count < self::BATCH_SIZE && !feof($this->fileHandle)) {
            $row = fgetcsv($this->fileHandle, 0, $delimiter, $enclosure);

            if ($row === false) {
                break;
            }

            // Skip empty rows
            if (empty(array_filter($row))) {
                continue;
            }

            $this->buffer[] = array_combine($this->colNames, $row);
            $count++;
        }
    }

    /**
     * Get next row
     */
    protected function _getNextRow()
    {
        if (empty($this->buffer)) {
            return false;
        }

        $row = array_shift($this->buffer);
        $this->currentRow++;

        // Reload buffer if empty
        if (empty($this->buffer) && !feof($this->fileHandle)) {
            $this->loadNextBatch(',', '"');
        }

        return $row;
    }

    /**
     * Rewind to start
     */
    public function rewind(): void
    {
        fseek($this->fileHandle, 0);
        fgetcsv($this->fileHandle); // Skip header
        $this->currentRow = 0;
        $this->loadNextBatch(',', '"');
        parent::rewind();
    }

    /**
     * Close file handle
     */
    public function __destruct()
    {
        if (is_resource($this->fileHandle)) {
            fclose($this->fileHandle);
        }
    }
}
```

---

## Performance Optimization

### Parallel Processing with Message Queue

**File**: `Vendor/WarrantyImport/Model/Import/Parallel/ChunkProcessor.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\WarrantyImport\Model\Import\Parallel;

use Magento\Framework\MessageQueue\PublisherInterface;

class ChunkProcessor
{
    private const CHUNK_SIZE = 5000;

    public function __construct(
        private readonly PublisherInterface $publisher
    ) {
    }

    /**
     * Split large import into chunks and publish to queue
     */
    public function processInParallel(string $filePath): void
    {
        $fileHandle = fopen($filePath, 'r');
        $header = fgetcsv($fileHandle);

        $chunk = [];
        $chunkNumber = 0;

        while (($row = fgetcsv($fileHandle)) !== false) {
            $chunk[] = $row;

            if (count($chunk) >= self::CHUNK_SIZE) {
                $this->publishChunk($header, $chunk, $chunkNumber++);
                $chunk = [];
            }
        }

        // Process remaining rows
        if (!empty($chunk)) {
            $this->publishChunk($header, $chunk, $chunkNumber);
        }

        fclose($fileHandle);
    }

    /**
     * Publish chunk to message queue
     */
    private function publishChunk(array $header, array $rows, int $chunkNumber): void
    {
        $chunkData = [
            'header' => $header,
            'rows' => $rows,
            'chunk_number' => $chunkNumber,
            'timestamp' => time(),
        ];

        $this->publisher->publish(
            'warranty.import.chunk',
            json_encode($chunkData)
        );
    }
}
```

**Queue Consumer:**

**File**: `Vendor/WarrantyImport/etc/queue_consumer.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework-message-queue:etc/consumer.xsd">
    <consumer name="warranty.import.chunk"
              queue="warranty.import.chunk"
              connection="amqp"
              handler="Vendor\WarrantyImport\Model\Import\Parallel\ChunkConsumer::process"
              maxMessages="10"/>
</config>
```

### Database Optimization

**Use batch inserts:**

```php
<?php
// Instead of single inserts
foreach ($warranties as $warranty) {
    $connection->insert($table, $warranty);
}

// Use batch insert
$connection->insertMultiple($table, $warranties);
```

**Disable indexers during import:**

```php
<?php
// Save indexer state
$indexerCollection = $objectManager->get(\Magento\Indexer\Model\Indexer\CollectionFactory::class)->create();
$indexerStates = [];

foreach ($indexerCollection as $indexer) {
    $indexerStates[$indexer->getId()] = $indexer->isScheduled();
    $indexer->setScheduled(true); // Set to scheduled mode
}

// Perform import
// ...

// Restore indexer state and reindex
foreach ($indexerCollection as $indexer) {
    $indexer->setScheduled($indexerStates[$indexer->getId()]);
    $indexer->reindexAll();
}
```

---

## Export Customization

### Custom Export Entity

**File**: `Vendor/WarrantyImport/Model/Export/Warranty.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\WarrantyImport\Model\Export;

use Magento\ImportExport\Model\Export\Entity\AbstractEntity;
use Magento\ImportExport\Model\Export\Factory as ExportFactory;
use Magento\Framework\App\ResourceConnection;

class Warranty extends AbstractEntity
{
    public const ENTITY_CODE = 'warranty_registration';

    protected $_permanentAttributes = ['warranty_id'];

    public function __construct(
        \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig,
        \Magento\Store\Model\StoreManagerInterface $storeManager,
        ExportFactory $collectionFactory,
        \Magento\ImportExport\Model\ResourceModel\CollectionByPagesIteratorFactory $resourceColFactory,
        private readonly ResourceConnection $resource
    ) {
        parent::__construct($scopeConfig, $storeManager, $collectionFactory, $resourceColFactory);
    }

    /**
     * Export process
     */
    public function export()
    {
        $connection = $this->resource->getConnection();
        $select = $connection->select()
            ->from($this->resource->getTableName('warranty_registration'));

        // Apply filters if set
        if ($filters = $this->_parameters['filters'] ?? []) {
            foreach ($filters as $field => $value) {
                $select->where("{$field} = ?", $value);
            }
        }

        $writer = $this->_getWriter();
        $writer->setHeaderCols($this->_getHeaderColumns());

        $result = $connection->fetchAll($select);

        foreach ($result as $row) {
            $writer->writeRow($this->_prepareExportRow($row));
        }

        return $writer->getContents();
    }

    /**
     * Get header columns
     */
    protected function _getHeaderColumns(): array
    {
        return [
            'warranty_id',
            'warranty_code',
            'customer_email',
            'product_sku',
            'serial_number',
            'purchase_date',
            'purchase_price',
            'retailer',
            'notes',
            'created_at',
        ];
    }

    /**
     * Prepare export row
     */
    private function _prepareExportRow(array $row): array
    {
        // Format dates
        if (isset($row['purchase_date'])) {
            $row['purchase_date'] = date('Y-m-d', strtotime($row['purchase_date']));
        }

        // Format price
        if (isset($row['purchase_price'])) {
            $row['purchase_price'] = number_format((float)$row['purchase_price'], 2, '.', '');
        }

        return $row;
    }

    /**
     * Entity type code
     */
    public function getEntityTypeCode(): string
    {
        return self::ENTITY_CODE;
    }
}
```

**Register export entity:**

**File**: `Vendor/WarrantyImport/etc/export.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_ImportExport:etc/export.xsd">
    <entity name="warranty_registration"
            label="Warranty Registrations"
            model="Vendor\WarrantyImport\Model\Export\Warranty"
            entityTypeCode="warranty_registration"/>
</config>
```

---

## Error Handling and Recovery

### Import with Error Threshold

```php
<?php
declare(strict_types=1);

namespace Vendor\WarrantyImport\Model\Import;

class ResilientImporter
{
    private const MAX_ERROR_PERCENTAGE = 5.0;

    public function __construct(
        private readonly \Magento\ImportExport\Model\Import $importModel,
        private readonly \Psr\Log\LoggerInterface $logger
    ) {
    }

    /**
     * Import with error threshold
     */
    public function import(string $filePath): array
    {
        $source = new \Magento\ImportExport\Model\Import\Source\Csv($filePath);

        $this->importModel->setData([
            'entity' => 'warranty_registration',
            'behavior' => \Magento\ImportExport\Model\Import::BEHAVIOR_APPEND,
            'validation_strategy' => \Magento\ImportExport\Model\Import::VALIDATION_STRATEGY_SKIP_ERRORS,
        ]);

        // Validate
        $validationResult = $this->importModel->validateSource($source);

        $errorAggregator = $this->importModel->getErrorAggregator();
        $totalRows = $errorAggregator->getErrorsCount() + $this->countValidRows($source);
        $errorPercentage = ($errorAggregator->getErrorsCount() / $totalRows) * 100;

        if ($errorPercentage > self::MAX_ERROR_PERCENTAGE) {
            throw new \RuntimeException(
                sprintf('Error rate %.2f%% exceeds threshold of %.2f%%', $errorPercentage, self::MAX_ERROR_PERCENTAGE)
            );
        }

        // Import valid rows
        $result = $this->importModel->importSource();

        return [
            'success' => $result,
            'created' => $this->importModel->getCreatedItemsCount(),
            'updated' => $this->importModel->getUpdatedItemsCount(),
            'errors' => $errorAggregator->getErrorsCount(),
            'error_percentage' => $errorPercentage,
        ];
    }

    private function countValidRows($source): int
    {
        $count = 0;
        foreach ($source as $row) {
            $count++;
        }
        $source->rewind();
        return $count;
    }
}
```

---

## Testing

### Unit Test

**File**: `Vendor/WarrantyImport/Test/Unit/Model/Import/WarrantyRegistrationTest.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\WarrantyImport\Test\Unit\Model\Import;

use PHPUnit\Framework\TestCase;
use Vendor\WarrantyImport\Model\Import\WarrantyRegistration;

class WarrantyRegistrationTest extends TestCase
{
    private WarrantyRegistration $importer;

    protected function setUp(): void
    {
        // Mock dependencies
        $this->importer = $this->createPartialMock(WarrantyRegistration::class, [
            'validateEmail',
            'validateProductSku'
        ]);
    }

    public function testValidateRowWithMissingEmail(): void
    {
        $rowData = [
            'product_sku' => 'TEST-SKU',
            'purchase_date' => '2025-01-01',
            'warranty_code' => 'WC12345',
        ];

        $result = $this->importer->validateRow($rowData, 1);

        $this->assertFalse($result);
    }
}
```

---

## Summary

**Key Takeaways:**
- Import entities extend AbstractEntity with validation, behavior, and data processing
- Source models handle file reading; use streaming for large files
- Validation uses ErrorAggregator for comprehensive error reporting
- Batch processing (insertMultiple) critical for performance
- Scheduled imports via cron enable automated data synchronization
- Message queues enable parallel processing for multi-million row imports
- Export entities mirror import structure for data extraction

---

## Assumptions

- **Magento Version:** 2.4.7+ (Adobe Commerce or Open Source)
- **PHP Version:** 8.2+ with sufficient memory_limit for large files
- **Database:** MySQL 8.0+ with optimized buffer pool
- **Message Queue:** RabbitMQ for parallel processing
- **File System:** Sufficient storage for archive/error directories

## Why This Approach

- **Entity Adapter Pattern:** Clean separation of concerns for validation, data transformation, persistence
- **Streaming Source:** Memory-efficient for files exceeding PHP memory limits
- **Batch Operations:** insertMultiple/insertOnDuplicate reduces query overhead
- **Error Aggregation:** Non-blocking validation enables partial imports
- **Queue-Based Parallelism:** Horizontal scaling for massive datasets

## Security Impact

- **File Upload Validation:** Restrict file types, sizes, and upload locations
- **Input Sanitization:** Validate all CSV data before database insertion
- **Authorization:** Restrict import/export to admin users with appropriate ACL
- **Path Traversal:** Use Magento filesystem abstraction, never raw file paths
- **SQL Injection:** Use parameterized queries and quoteInto()

## Performance Impact

- **Memory:** Streaming source reduces memory footprint from GB to MB
- **Database:** Batch inserts 50-100x faster than single-row inserts
- **Indexing:** Defer indexing until after import completes
- **FPC:** Import operations bypass FPC; minimal impact on frontend
- **Queue:** Parallel consumers scale horizontally for large imports

## Backward Compatibility

- **Import API:** Entity adapter interface stable since Magento 2.0
- **CSV Format:** Maintain backward compatibility with existing import files
- **Database Schema:** Use data patches for schema changes
- **Export Format:** Maintain column order for existing integrations

## Tests to Add

**Unit Tests:**
```php
testValidateEmailFormat()
testValidateDateRange()
testBatchInsertWarranties()
testErrorThresholdExceeded()
```

**Integration Tests:**
```php
testFullImportWorkflow()
testScheduledImportExecution()
testExportMatchesImport()
```

**Performance Tests:**
```bash
# Note: import:run is not a core Magento CLI command. Use the Admin Import UI
# (System > Data Transfer > Import) or a third-party CLI import tool.
# Example using a custom or third-party import command:
# time bin/magento custom:import:run --entity=warranty_registration --file=1m_rows.csv
```

## Docs to Update

- **README.md:** Installation, CSV format specification, usage examples
- **docs/IMPORT_FORMAT.md:** Column definitions, validation rules, examples
- **docs/PERFORMANCE.md:** Optimization guidelines, benchmarks
- **Admin User Guide:** Screenshots of import UI, error report interpretation

## Related Documentation

### Related Guides

- [Cron Jobs Implementation: Building Reliable Scheduled Tasks in Magento 2](cron-jobs.md)
- [ERP Integration Patterns for Magento 2](erp-integration.md)
- [Declarative Schema & Data Patches: Modern Database Management in Magento 2](../tutorials/declarative-schema-data-patches.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Customer Overview](../../modules/customer/README.md)
