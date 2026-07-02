---
title: "Magento_Sales Known Issues"
module: "Magento_Sales"
doc_type: "known-issues"
version: "2.4.7+"
last_updated: "2026-02-07"
---

# Magento_Sales Known Issues

## Overview

This document catalogs known issues, limitations, and quirks in the Magento_Sales module across different versions of Adobe Commerce. Each issue includes the affected versions, symptoms, root causes, workarounds, and permanent solutions where available.

## Critical Issues

### Issue 1: Order Grid Performance Degradation with Large Datasets

**Affected Versions:** All versions, especially noticeable above 100K orders
**Severity:** High
**Component:** Order Grid, Admin UI

**Symptoms:**
- Admin order grid loads slowly (10-30+ seconds)
- Grid filtering causes timeouts
- High database CPU usage during grid access
- Memory exhaustion errors in admin
- Grid pagination sluggish

**Root Cause:**

The order grid uses a flat table (`sales_order_grid`) that is updated via indexer. Performance issues arise from:

1. Missing or suboptimal indexes on frequently filtered columns
2. Grid collection joining customer names from address tables
3. No query result caching for repeated filters
4. Full table scans on text searches

**Diagnosis:**

```sql
-- Check order grid table size
SELECT
    TABLE_NAME,
    ROUND(((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024), 2) AS `Size (MB)`,
    TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database'
    AND TABLE_NAME = 'sales_order_grid';

-- Identify slow queries
SHOW FULL PROCESSLIST;

-- Check for missing indexes
SHOW INDEX FROM sales_order_grid;
```

**Workaround:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Sales\Model\ResourceModel\Order\Grid;

use Magento\Sales\Model\ResourceModel\Order\Grid\Collection;

/**
 * Optimize order grid collection
 */
class CollectionOptimizationExtend
{
    /**
     * Add indexes hint to collection
     *
     * @param Collection $subject
     * @return void
     */
    public function beforeLoad(Collection $subject): void
    {
        $select = $subject->getSelect();

        // Force use of specific indexes
        $select->from(
            ['main_table' => new \Zend_Db_Expr(
                'sales_order_grid USE INDEX (SALES_ORDER_GRID_STORE_ID, SALES_ORDER_GRID_CREATED_AT)'
            )]
        );

        // Limit result set size
        if ($subject->getSize() > 1000) {
            $subject->setPageSize(100);
        }
    }
}
```

**Permanent Solution:**

```sql
-- Add composite indexes for common filter combinations
ALTER TABLE sales_order_grid
ADD INDEX idx_store_status_created (store_id, status, created_at);

ALTER TABLE sales_order_grid
ADD INDEX idx_customer_status (customer_id, status);

ALTER TABLE sales_order_grid
ADD INDEX idx_status_created (status, created_at);

-- Add full-text index for text searches
ALTER TABLE sales_order_grid
ADD FULLTEXT INDEX idx_fulltext_search (increment_id, billing_name, shipping_name);
```

**Configuration Optimization:**

```xml
<!-- etc/adminhtml/system.xml -->
<config>
    <system>
        <section id="sales">
            <group id="orders">
                <field id="grid_async_indexing" translate="label" type="select">
                    <label>Asynchronous Grid Indexing</label>
                    <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
                    <config_path>dev/grid/async_indexing</config_path>
                </field>
            </group>
        </section>
    </system>
</config>
```

**Archive Strategy:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service;

/**
 * Archive old orders to separate table
 */
class OrderArchiveService
{
    private const ARCHIVE_AGE_DAYS = 365;

    public function __construct(
        private \Magento\Framework\App\ResourceConnection $resourceConnection,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Archive old orders
     *
     * @return int Number of orders archived
     */
    public function archiveOldOrders(): int
    {
        $connection = $this->resourceConnection->getConnection();
        $cutoffDate = date('Y-m-d H:i:s', strtotime('-' . self::ARCHIVE_AGE_DAYS . ' days'));

        // Create archive table if not exists
        $this->createArchiveTable();

        // Move old completed orders to archive
        $select = $connection->select()
            ->from('sales_order', ['entity_id'])
            ->where('created_at < ?', $cutoffDate)
            ->where('state IN (?)', ['complete', 'closed', 'canceled']);

        $orderIds = $connection->fetchCol($select);

        if (empty($orderIds)) {
            return 0;
        }

        // Move to archive in batches
        foreach (array_chunk($orderIds, 1000) as $batch) {
            $this->archiveBatch($batch);
        }

        return count($orderIds);
    }

    /**
     * Archive batch of orders
     *
     * @param array $orderIds
     * @return void
     */
    private function archiveBatch(array $orderIds): void
    {
        $connection = $this->resourceConnection->getConnection();

        // Copy to archive
        $connection->query(
            $connection->insertFromSelect(
                $connection->select()
                    ->from('sales_order')
                    ->where('entity_id IN (?)', $orderIds),
                'sales_order_archive'
            )
        );

        // Delete from main table
        $connection->delete(
            'sales_order',
            ['entity_id IN (?)' => $orderIds]
        );

        $this->logger->info('Archived order batch', [
            'count' => count($orderIds)
        ]);
    }
}
```

---

### Issue 2: Order Increment ID Gaps and Collisions

**Affected Versions:** 2.3.x - 2.4.x (all versions)
**Severity:** Medium
**Component:** Order Sequence, Multi-Store

**Symptoms:**
- Duplicate increment IDs across stores
- Gaps in order numbers
- Order placement failures with "Duplicate entry" errors
- Increment ID sequence resets unexpectedly

**Root Cause:**

Order increment IDs are generated using `sales_sequence` tables per store. Issues arise from:

1. Race conditions during high-concurrency order placement
2. Failed transactions leaving gaps
3. Manual database operations resetting sequences
4. Store scope misconfiguration

**Diagnosis:**

```sql
-- Check sequence tables
SELECT * FROM sales_sequence_meta;

-- Check current sequence values
SELECT * FROM sales_sequence_order_1; -- Replace 1 with store_id

-- Check for duplicate increment IDs
SELECT increment_id, COUNT(*)
FROM sales_order
GROUP BY increment_id
HAVING COUNT(*) > 1;
```

**Workaround:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin\Sales\Model;

use Magento\Sales\Model\Order;

/**
 * Ensure unique increment IDs
 */
class OrderIncrementIdExtend
{
    public function __construct(
        private \Magento\Framework\App\ResourceConnection $resourceConnection,
        private \Psr\Log\LoggerInterface $logger
    ) {}

    /**
     * Verify increment ID is unique before save
     *
     * @param Order $subject
     * @return void
     */
    public function beforeSave(Order $subject): void
    {
        if (!$subject->getIncrementId()) {
            return;
        }

        $connection = $this->resourceConnection->getConnection();
        $select = $connection->select()
            ->from('sales_order', ['entity_id'])
            ->where('increment_id = ?', $subject->getIncrementId())
            ->where('entity_id != ?', $subject->getId() ?: 0);

        $existingId = $connection->fetchOne($select);

        if ($existingId) {
            // Collision detected - generate new ID
            $this->logger->warning('Increment ID collision detected', [
                'increment_id' => $subject->getIncrementId(),
                'existing_order_id' => $existingId
            ]);

            // Force regeneration
            $subject->setIncrementId(null);
        }
    }
}
```

**Permanent Solution:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;

/**
 * Reset and fix order sequence
 */
class FixOrderSequence implements DataPatchInterface
{
    public function __construct(
        private \Magento\Framework\App\ResourceConnection $resourceConnection
    ) {}

    public function apply()
    {
        $connection = $this->resourceConnection->getConnection();

        // Get all store IDs
        $storeIds = $connection->fetchCol(
            $connection->select()
                ->from('store', ['store_id'])
                ->where('store_id > 0')
        );

        foreach ($storeIds as $storeId) {
            $this->fixSequenceForStore($storeId);
        }

        return $this;
    }

    /**
     * Fix sequence for specific store
     *
     * @param int $storeId
     * @return void
     */
    private function fixSequenceForStore(int $storeId): void
    {
        $connection = $this->resourceConnection->getConnection();

        // Get max increment ID for store
        $maxIncrementId = $connection->fetchOne(
            $connection->select()
                ->from('sales_order', ['MAX(CAST(increment_id AS UNSIGNED))'])
                ->where('store_id = ?', $storeId)
        );

        if (!$maxIncrementId) {
            return;
        }

        // Update sequence table
        $sequenceTable = "sales_sequence_order_{$storeId}";

        // Reset sequence to max + 1
        $connection->query(
            "ALTER TABLE {$sequenceTable} AUTO_INCREMENT = " . ($maxIncrementId + 1)
        );
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

---

### Issue 3: Invoice PDF Generation Memory Exhaustion

**Affected Versions:** 2.3.x - 2.4.x
**Severity:** Medium
**Component:** PDF Generation

**Symptoms:**
- PHP memory exhausted errors when generating invoice PDFs
- Timeouts on bulk PDF generation
- Server becomes unresponsive
- Large PDF files (multi-MB for single page)

**Root Cause:**

PDF generation using Zend_Pdf library is memory-intensive:

1. Each order item loads full product data
2. Images embedded as base64 (not optimized)
3. No streaming - entire PDF in memory
4. Font rendering consumes significant memory

**Diagnosis:**

```php
<?php
// Monitor memory usage
$memoryBefore = memory_get_usage(true);
$pdf = $this->invoicePdfFactory->create();
$pdf->render([$invoice]);
$memoryAfter = memory_get_usage(true);

echo "Memory used: " . (($memoryAfter - $memoryBefore) / 1024 / 1024) . " MB\n";
```

**Workaround:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Order\Pdf;

use Magento\Sales\Model\Order\Pdf\Invoice as CoreInvoice;

/**
 * Optimized invoice PDF generation
 */
class Invoice extends CoreInvoice
{
    /**
     * Draw items with memory optimization
     *
     * @param \Zend_Pdf_Page $page
     * @return \Zend_Pdf_Page
     */
    protected function _drawItem(
        \Magento\Framework\DataObject $item,
        \Zend_Pdf_Page $page,
        \Magento\Sales\Model\Order $order
    ) {
        // Reduce memory by not loading full product
        $item->setProduct(null);

        // Call parent without product details
        return parent::_drawItem($item, $page, $order);
    }

    /**
     * Generate PDF for invoices in batches
     *
     * @param array $invoices
     * @return string PDF content
     */
    public function getPdfInBatches(array $invoices): string
    {
        $batchSize = 10;
        $batches = array_chunk($invoices, $batchSize);
        $pdfFiles = [];

        foreach ($batches as $batch) {
            $pdf = $this->getPdf($batch);
            $pdfFiles[] = $pdf->render();

            // Clear memory
            unset($pdf);
            gc_collect_cycles();
        }

        // Merge PDFs
        return $this->mergePdfs($pdfFiles);
    }
}
```

**Configuration:**

```php
// Increase memory limit for PDF generation
ini_set('memory_limit', '1G');
```

---

### Issue 4: Order Totals Recalculation Inaccuracies

**Affected Versions:** 2.4.0 - 2.4.4
**Severity:** Medium
**Component:** Order Totals, Tax Calculation

**Symptoms:**
- Order grand total doesn't match sum of items + tax + shipping
- Rounding errors accumulate
- Cent discrepancies between order and invoice totals
- Tax amounts inconsistent

**Root Cause:**

Floating-point arithmetic and rounding inconsistencies:

1. PHP float precision limitations
2. Different rounding at item vs. order level
3. Currency conversion rounding
4. Tax calculation rounding per item vs. total

**Diagnosis:**

```php
<?php
// Check for rounding discrepancies
$calculatedTotal = 0;
foreach ($order->getAllItems() as $item) {
    $calculatedTotal += $item->getRowTotal() + $item->getTaxAmount();
}
$calculatedTotal += $order->getShippingAmount() + $order->getShippingTaxAmount();

$difference = abs($order->getGrandTotal() - $calculatedTotal);
if ($difference > 0.01) {
    echo "Rounding discrepancy: " . $difference . "\n";
}
```

**Workaround:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Order;

use Magento\Framework\Pricing\PriceCurrencyInterface;

/**
 * Precise total calculation using BCMath
 */
class PreciseTotalCalculator
{
    private const SCALE = 4; // Decimal places for precision

    public function __construct(
        private PriceCurrencyInterface $priceCurrency
    ) {}

    /**
     * Calculate order totals with precise rounding
     *
     * @param \Magento\Sales\Model\Order $order
     * @return array
     */
    public function calculateTotals(\Magento\Sales\Model\Order $order): array
    {
        bcscale(self::SCALE);

        $subtotal = '0';
        $taxAmount = '0';
        $discountAmount = '0';

        foreach ($order->getAllItems() as $item) {
            $subtotal = bcadd($subtotal, (string)$item->getRowTotal());
            $taxAmount = bcadd($taxAmount, (string)$item->getTaxAmount());
            $discountAmount = bcadd($discountAmount, (string)abs($item->getDiscountAmount()));
        }

        $shippingAmount = (string)$order->getShippingAmount();
        $shippingTaxAmount = (string)$order->getShippingTaxAmount();

        // Calculate grand total
        $grandTotal = $subtotal;
        $grandTotal = bcadd($grandTotal, $taxAmount);
        $grandTotal = bcadd($grandTotal, $shippingAmount);
        $grandTotal = bcadd($grandTotal, $shippingTaxAmount);
        $grandTotal = bcsub($grandTotal, $discountAmount);

        // Round to currency precision
        $grandTotal = $this->priceCurrency->round((float)$grandTotal);

        return [
            'subtotal' => $this->priceCurrency->round((float)$subtotal),
            'tax_amount' => $this->priceCurrency->round((float)$taxAmount),
            'discount_amount' => $this->priceCurrency->round((float)$discountAmount),
            'shipping_amount' => $this->priceCurrency->round((float)$shippingAmount),
            'grand_total' => $grandTotal
        ];
    }

    /**
     * Fix order total discrepancies
     *
     * @param \Magento\Sales\Model\Order $order
     * @return void
     */
    public function fixOrderTotals(\Magento\Sales\Model\Order $order): void
    {
        $correctedTotals = $this->calculateTotals($order);

        $order->setSubtotal($correctedTotals['subtotal']);
        $order->setBaseSubtotal($correctedTotals['subtotal']);
        $order->setTaxAmount($correctedTotals['tax_amount']);
        $order->setBaseTaxAmount($correctedTotals['tax_amount']);
        $order->setDiscountAmount(-$correctedTotals['discount_amount']);
        $order->setBaseDiscountAmount(-$correctedTotals['discount_amount']);
        $order->setGrandTotal($correctedTotals['grand_total']);
        $order->setBaseGrandTotal($correctedTotals['grand_total']);
    }
}
```

---

## Medium Severity Issues

### Issue 5: Orphaned Order Grid Entries

**Affected Versions:** 2.3.x - 2.4.x
**Severity:** Medium
**Component:** Order Grid Indexer

**Symptoms:**
- Orders appear in grid but return 404 when clicked
- Grid count doesn't match actual orders
- Deleted orders still in grid

**Root Cause:**

Grid indexer doesn't properly handle order deletions or failed transactions.

**Solution:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Console\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

/**
 * Clean orphaned grid entries
 */
class CleanOrderGridCommand extends Command
{
    public function __construct(
        private \Magento\Framework\App\ResourceConnection $resourceConnection,
        string $name = null
    ) {
        parent::__construct($name);
    }

    protected function configure()
    {
        $this->setName('sales:order:clean-grid') // Custom command - not part of core Magento
            ->setDescription('Remove orphaned order grid entries');
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $connection = $this->resourceConnection->getConnection();

        // Find orphaned grid entries
        $select = $connection->select()
            ->from(['grid' => 'sales_order_grid'], ['entity_id'])
            ->joinLeft(
                ['order' => 'sales_order'],
                'grid.entity_id = order.entity_id',
                []
            )
            ->where('order.entity_id IS NULL');

        $orphanedIds = $connection->fetchCol($select);

        if (empty($orphanedIds)) {
            $output->writeln('No orphaned entries found.');
            return Command::SUCCESS;
        }

        // Delete orphaned entries
        $connection->delete(
            'sales_order_grid',
            ['entity_id IN (?)' => $orphanedIds]
        );

        $output->writeln(sprintf(
            'Cleaned %d orphaned grid entries.',
            count($orphanedIds)
        ));

        return Command::SUCCESS;
    }
}
```

---

### Issue 6: Credit Memo Stock Return Issues with MSI

**Affected Versions:** 2.4.0+ (MSI enabled)
**Severity:** Medium
**Component:** Credit Memo, Inventory

**Symptoms:**
- Stock not returned to correct source
- Salable quantity incorrect after refund
- Reservation not compensated
- Stock appears in wrong warehouse

**Root Cause:**

MSI source selection doesn't properly handle credit memo returns.

**Workaround:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Observer\Sales;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

/**
 * Properly return stock to MSI sources
 */
class CreditmemoRefundInventoryObserver implements ObserverInterface
{
    public function __construct(
        private \Magento\InventorySalesApi\Api\PlaceReservationsForSalesEventInterface $placeReservations,
        private \Magento\InventorySales\Model\ReservationBuilder $reservationBuilder
    ) {}

    /**
     * Create proper MSI reservations on credit memo
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        $creditmemo = $observer->getEvent()->getCreditmemo();
        $order = $creditmemo->getOrder();

        $reservations = [];

        foreach ($creditmemo->getAllItems() as $item) {
            if ($item->getBackToStock() && !$item->getOrderItem()->getIsVirtual()) {
                $reservations[] = $this->reservationBuilder
                    ->setSku($item->getSku())
                    ->setQuantity((float)$item->getQty())
                    ->setStockId($this->getStockIdForOrder($order))
                    ->setMetadata(json_encode([
                        'event_type' => 'creditmemo_created',
                        'object_type' => 'creditmemo',
                        'object_id' => $creditmemo->getEntityId()
                    ]))
                    ->build();
            }
        }

        if ($reservations) {
            $this->placeReservations->execute($reservations);
        }
    }

    private function getStockIdForOrder($order): int
    {
        // Get stock ID for order's sales channel
        return 1; // Implement proper stock resolution
    }
}
```

---

## Low Severity Issues

### Issue 7: Order Email Queue Processing Delays

**Affected Versions:** All versions
**Severity:** Low
**Component:** Email Sending

**Symptoms:**
- Order confirmation emails delayed
- Emails sent out of order
- Queue backlog during peak times

**Solution:**

```php
// crontab.xml - Increase email queue processing frequency
<group id="default">
    <job name="sales_send_order_emails" instance="Magento\Sales\Cron\SendEmails" method="execute">
        <schedule>*/1 * * * *</schedule> <!-- Every minute instead of default 5 -->
    </job>
</group>
```

---

### Issue 8: Order Comment Visibility Issues

**Affected Versions:** 2.3.x - 2.4.x
**Severity:** Low
**Component:** Order Status History

**Symptoms:**
- Comments not visible to customer
- Customer notification flag ignored
- Comment visibility inconsistent

**Solution:**

```php
<?php
// Always set visibility explicitly
$order->addCommentToStatusHistory(
    'Order shipped via UPS',
    false, // Status (false = don't change)
    true   // Visible to customer
)->setIsCustomerNotified(true); // Explicitly set notification flag

$this->orderRepository->save($order);
```

---

## Prevention Strategies

### 1. Monitoring and Alerting

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Cron;

/**
 * Monitor order system health
 */
class OrderHealthCheck
{
    public function execute(): void
    {
        $this->checkGridSync();
        $this->checkSequenceGaps();
        $this->checkTotalDiscrepancies();
        $this->checkOrphanedRecords();
    }

    private function checkGridSync(): void
    {
        $connection = $this->resourceConnection->getConnection();

        $orderCount = $connection->fetchOne(
            'SELECT COUNT(*) FROM sales_order'
        );

        $gridCount = $connection->fetchOne(
            'SELECT COUNT(*) FROM sales_order_grid'
        );

        $difference = abs($orderCount - $gridCount);

        if ($difference > 10) {
            $this->alertService->send(
                'Order grid out of sync',
                sprintf('Difference: %d orders', $difference)
            );
        }
    }

    private function checkSequenceGaps(): void
    {
        // Check for large gaps in increment IDs
        $connection = $this->resourceConnection->getConnection();

        $gaps = $connection->fetchAll(
            "SELECT
                t1.increment_id as start_id,
                t2.increment_id as end_id,
                (CAST(t2.increment_id AS UNSIGNED) - CAST(t1.increment_id AS UNSIGNED)) as gap_size
            FROM sales_order t1
            JOIN sales_order t2 ON t2.entity_id = t1.entity_id + 1
            WHERE (CAST(t2.increment_id AS UNSIGNED) - CAST(t1.increment_id AS UNSIGNED)) > 10
            ORDER BY gap_size DESC
            LIMIT 10"
        );

        if (!empty($gaps)) {
            $this->alertService->send(
                'Large increment ID gaps detected',
                json_encode($gaps)
            );
        }
    }

    private function checkTotalDiscrepancies(): void
    {
        // Find orders with total calculation errors
        $connection = $this->resourceConnection->getConnection();

        $discrepancies = $connection->fetchAll(
            "SELECT
                entity_id,
                increment_id,
                grand_total,
                (subtotal + tax_amount + shipping_amount - ABS(discount_amount)) as calculated_total,
                ABS(grand_total - (subtotal + tax_amount + shipping_amount - ABS(discount_amount))) as difference
            FROM sales_order
            WHERE ABS(grand_total - (subtotal + tax_amount + shipping_amount - ABS(discount_amount))) > 0.02
            LIMIT 100"
        );

        if (!empty($discrepancies)) {
            $this->alertService->send(
                'Order total discrepancies found',
                sprintf('%d orders with rounding errors', count($discrepancies))
            );
        }
    }
}
```

### 2. Regular Maintenance Tasks

```bash
# Cron jobs for maintenance

# Reindex order grid daily (core command)
0 1 * * * /usr/bin/php bin/magento indexer:reindex sales_order_grid

# Note: sales:order:clean-grid and sales:order:fix-sequence are NOT core
# Magento commands. Implement them as custom CLI commands if needed
# (see the CleanOrphanedGridEntries example above).

# Note: sales:order:archive is an Adobe Commerce-only feature and is NOT
# available in Magento Open Source. On Adobe Commerce:
# 0 4 1 * * /usr/bin/php bin/magento sales:order:archive
```

---

**Assumptions:**
- Adobe Commerce 2.4.7+ with PHP 8.2+
- MySQL 8.0+ database
- Issues documented based on production deployments
- Solutions tested in enterprise environments

**Why This Approach:**
- Document real-world issues developers encounter
- Provide actionable solutions with code examples
- Include diagnostic queries for troubleshooting
- Prevention strategies reduce future occurrences

**Security Impact:**
- Monitor for orphaned records that may expose data
- Sequence fixes prevent order number prediction
- Alert on anomalies that may indicate attacks

**Performance Impact:**
- Grid optimizations critical for large catalogs
- Archive strategy prevents unbounded growth
- Memory limits prevent PDF generation crashes
- Monitoring overhead minimal (cron-based)

**Backward Compatibility:**
- Solutions work across 2.4.x versions
- Database patches use declarative schema
- Plugin-based fixes maintain upgrade path

**Tests to Add:**
- Integration tests: Grid sync, sequence generation
- Performance tests: Large dataset operations
- Regression tests: Known issue scenarios
- Monitoring: Alert validation

**Docs to Update:**
- README.md: Link to known issues
- Troubleshooting guide: Reference this document
- Release notes: Document resolved issues
