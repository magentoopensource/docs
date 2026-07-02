---
title: "Declarative Schema & Data Patches: Modern Database Management in Magento 2"
description: "Complete guide to declarative schema (db_schema.xml) and data/schema patches in Magento 2.4.7+: create tables, manage indexes, foreign keys, versioning, migrations, and rollback strategies"
type: "tutorial"
tier: 1
difficulty: "intermediate"
version: "2.4.7+"
estimated_time: "75 minutes"
topics:
  - declarative-schema
  - db-schema
  - data-patches
  - schema-patches
  - database-migrations
  - setup-scripts
  - mysql
  - indexing
  - foreign-keys
last_updated: "2026-02-07"
---

# Declarative Schema & Data Patches: Modern Database Management in Magento 2

## Learning Objectives

By completing this tutorial, you will:

- Understand declarative schema architecture and advantages over legacy InstallSchema/UpgradeSchema
- Master `db_schema.xml` syntax for tables, columns, indexes, and foreign keys
- Create data patches (DataPatchInterface) for idempotent data modifications
- Create schema patches (SchemaPatchInterface) for complex schema changes
- Implement patch dependencies and versioning strategies
- Migrate legacy setup scripts to declarative schema
- Design rollback strategies for schema and data changes
- Optimize database performance with proper indexing and constraints
- Ensure upgrade-safe database modifications

## Introduction

Magento 2.3+ introduced **declarative schema**, a paradigm shift from procedural setup scripts (InstallSchema, UpgradeSchema) to declarative configuration files (`db_schema.xml`). Instead of writing PHP code to describe **how** to change the database, you declare **what** the database should look like, and Magento calculates the necessary changes.

### What is Declarative Schema?

Declarative schema defines the **desired state** of your database structure in XML format. Magento compares this desired state with the current database state and generates the SQL needed to migrate from current to desired state.

**Key files:**

1. **`etc/db_schema.xml`**: Declares tables, columns, indexes, constraints
2. **`etc/db_schema_whitelist.json`**: Tracks schema changes (auto-generated, never manually edit)
3. **`Setup/Patch/Data/*.php`**: Data patches for inserting/modifying data
4. **`Setup/Patch/Schema/*.php`**: Schema patches for complex changes that can't be expressed in XML

### Why Declarative Schema?

**Legacy approach (Magento 2.0-2.2):**

```php
// InstallSchema.php - Procedural, version-dependent
public function install(SchemaSetupInterface $setup, ModuleContextInterface $context)
{
    $setup->startSetup();
    $table = $setup->getConnection()->newTable(
        $setup->getTable('vendor_table')
    )->addColumn(
        'entity_id',
        \Magento\Framework\DB\Ddl\Table::TYPE_INTEGER,
        null,
        ['identity' => true, 'unsigned' => true, 'nullable' => false, 'primary' => true],
        'Entity ID'
    );
    $setup->getConnection()->createTable($table);
    $setup->endSetup();
}
```

**Problems:**
- Version-dependent: Must track module versions
- Error-prone: Easy to forget columns, indexes
- No rollback: Upgrades are one-way
- Difficult to maintain: Changes scattered across multiple upgrade scripts

**Declarative approach (Magento 2.3+):**

```xml
<!-- db_schema.xml - Declarative, version-independent -->
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    <table name="vendor_table" resource="default" engine="innodb" comment="Vendor Table">
        <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" identity="true" comment="Entity ID"/>
        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="entity_id"/>
        </constraint>
    </table>
</schema>
```

**Advantages:**
- **Version-independent**: No module version tracking
- **Declarative**: Clear "what" not "how"
- **Rollback support**: Can revert schema changes
- **Single source of truth**: All schema in one file
- **Validation**: XSD schema validation prevents errors
- **Dry-run**: Preview changes before applying

### When to Use db_schema.xml vs Patches

| Task | Use | Reason |
|------|-----|--------|
| Create table | `db_schema.xml` | Declarative structure |
| Add column | `db_schema.xml` | Simple schema change |
| Add index | `db_schema.xml` | Declarative indexing |
| Add foreign key | `db_schema.xml` | Declarative constraint |
| Insert initial data | Data Patch | Data operations |
| Complex column transform | Schema Patch | Complex SQL logic |
| Migrate data between tables | Data Patch | Data operations |
| Add computed column | Schema Patch | SQL expressions not supported in XML |

## db_schema.xml Syntax

### Complete Example

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">

    <!-- Main entity table -->
    <table name="vendor_product_review" resource="default" engine="innodb" comment="Product Review Table">

        <!-- Columns -->
        <column xsi:type="int" name="review_id" unsigned="true" nullable="false" identity="true" comment="Review ID"/>
        <column xsi:type="int" name="product_id" unsigned="true" nullable="false" comment="Product ID"/>
        <column xsi:type="int" name="customer_id" unsigned="true" nullable="true" comment="Customer ID"/>
        <column xsi:type="varchar" name="nickname" nullable="false" length="128" comment="Nickname"/>
        <column xsi:type="varchar" name="title" nullable="false" length="255" comment="Review Title"/>
        <column xsi:type="text" name="detail" nullable="false" comment="Review Detail"/>
        <column xsi:type="smallint" name="rating" unsigned="true" nullable="false" default="0" comment="Rating (1-5)"/>
        <column xsi:type="smallint" name="status" unsigned="true" nullable="false" default="0" comment="Status (0=pending, 1=approved)"/>
        <column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP" comment="Creation Time"/>
        <column xsi:type="timestamp" name="updated_at" nullable="false" default="CURRENT_TIMESTAMP" on_update="true" comment="Update Time"/>

        <!-- Primary Key -->
        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="review_id"/>
        </constraint>

        <!-- Foreign Keys -->
        <constraint xsi:type="foreign" referenceId="FK_REVIEW_PRODUCT_ID"
                    table="vendor_product_review" column="product_id"
                    referenceTable="catalog_product_entity" referenceColumn="entity_id"
                    onDelete="CASCADE"/>

        <constraint xsi:type="foreign" referenceId="FK_REVIEW_CUSTOMER_ID"
                    table="vendor_product_review" column="customer_id"
                    referenceTable="customer_entity" referenceColumn="entity_id"
                    onDelete="SET NULL"/>

        <!-- Indexes -->
        <index referenceId="VENDOR_REVIEW_PRODUCT_ID" indexType="btree">
            <column name="product_id"/>
        </index>

        <index referenceId="VENDOR_REVIEW_CUSTOMER_ID" indexType="btree">
            <column name="customer_id"/>
        </index>

        <index referenceId="VENDOR_REVIEW_STATUS_CREATED" indexType="btree">
            <column name="status"/>
            <column name="created_at"/>
        </index>

        <!-- Unique Constraint -->
        <constraint xsi:type="unique" referenceId="VENDOR_REVIEW_UNIQUE_CUSTOMER_PRODUCT">
            <column name="customer_id"/>
            <column name="product_id"/>
        </constraint>

    </table>

    <!-- Join table for many-to-many relationship -->
    <table name="vendor_review_tag" resource="default" engine="innodb" comment="Review Tag Relation Table">

        <column xsi:type="int" name="review_id" unsigned="true" nullable="false" comment="Review ID"/>
        <column xsi:type="int" name="tag_id" unsigned="true" nullable="false" comment="Tag ID"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="review_id"/>
            <column name="tag_id"/>
        </constraint>

        <constraint xsi:type="foreign" referenceId="FK_REVIEW_TAG_REVIEW_ID"
                    table="vendor_review_tag" column="review_id"
                    referenceTable="vendor_product_review" referenceColumn="review_id"
                    onDelete="CASCADE"/>

        <constraint xsi:type="foreign" referenceId="FK_REVIEW_TAG_TAG_ID"
                    table="vendor_review_tag" column="tag_id"
                    referenceTable="vendor_tag" referenceColumn="tag_id"
                    onDelete="CASCADE"/>

        <index referenceId="VENDOR_REVIEW_TAG_TAG_ID" indexType="btree">
            <column name="tag_id"/>
        </index>

    </table>

    <!-- Tag table -->
    <table name="vendor_tag" resource="default" engine="innodb" comment="Tag Table">

        <column xsi:type="int" name="tag_id" unsigned="true" nullable="false" identity="true" comment="Tag ID"/>
        <column xsi:type="varchar" name="name" nullable="false" length="64" comment="Tag Name"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="tag_id"/>
        </constraint>

        <constraint xsi:type="unique" referenceId="VENDOR_TAG_UNIQUE_NAME">
            <column name="name"/>
        </constraint>

    </table>

</schema>
```

### Table Definition

```xml
<table name="vendor_table_name"
       resource="default"
       engine="innodb"
       comment="Human-readable description">
    <!-- columns, constraints, indexes -->
</table>
```

**Attributes:**

| Attribute | Required | Description | Values |
|-----------|----------|-------------|--------|
| `name` | Yes | Table name (with vendor prefix) | `vendor_module_entity` |
| `resource` | No | Database resource (default: `default`) | `default`, `sales`, `checkout` |
| `engine` | No | Storage engine (default: `innodb`) | `innodb`, `memory` |
| `comment` | No | Table comment | Any string |

**Naming convention**: `{vendor}_{module}_{entity}` (e.g., `acme_blog_post`)

### Column Types

```xml
<!-- Integer types -->
<column xsi:type="smallint" name="status" unsigned="true" nullable="false" default="0"/>
<column xsi:type="int" name="entity_id" unsigned="true" nullable="false" identity="true"/>
<column xsi:type="bigint" name="large_number" unsigned="false" nullable="true"/>

<!-- Decimal/Float -->
<column xsi:type="decimal" name="price" precision="12" scale="4" nullable="false" default="0.0000"/>
<column xsi:type="float" name="weight" nullable="true"/>

<!-- String types -->
<column xsi:type="varchar" name="name" length="255" nullable="false"/>
<column xsi:type="text" name="description" nullable="true"/>
<column xsi:type="mediumtext" name="content" nullable="true"/>
<column xsi:type="longtext" name="large_content" nullable="true"/>

<!-- Date/Time -->
<column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP"/>
<column xsi:type="timestamp" name="updated_at" nullable="false" default="CURRENT_TIMESTAMP" on_update="true"/>
<column xsi:type="date" name="birth_date" nullable="true"/>
<column xsi:type="datetime" name="event_time" nullable="true"/>

<!-- Binary -->
<column xsi:type="blob" name="binary_data" nullable="true"/>
<column xsi:type="mediumblob" name="medium_binary_data" nullable="true"/>

<!-- Boolean (stored as smallint) -->
<column xsi:type="boolean" name="is_active" nullable="false" default="false"/>
```

**Column attributes:**

| Attribute | Required | Description | Example |
|-----------|----------|-------------|---------|
| `xsi:type` | Yes | Column data type | `int`, `varchar`, `text`, `timestamp` |
| `name` | Yes | Column name | `entity_id`, `customer_name` |
| `unsigned` | No | Unsigned integer (int types only) | `true`, `false` |
| `nullable` | No | Allow NULL values (default: true) | `true`, `false` |
| `identity` | No | Auto-increment (int types only) | `true`, `false` |
| `default` | No | Default value | `0`, `''`, `CURRENT_TIMESTAMP` |
| `length` | Conditional | Length for varchar (required for varchar) | `255`, `128` |
| `precision` | Conditional | Total digits for decimal (required) | `12` |
| `scale` | Conditional | Decimal places for decimal (required) | `4` |
| `on_update` | No | Update timestamp on row update | `true` (timestamp only) |
| `comment` | No | Column comment | Any string |

### Constraints

#### Primary Key

```xml
<constraint xsi:type="primary" referenceId="PRIMARY">
    <column name="entity_id"/>
</constraint>

<!-- Composite primary key -->
<constraint xsi:type="primary" referenceId="PRIMARY">
    <column name="entity_id"/>
    <column name="store_id"/>
</constraint>
```

#### Foreign Key

```xml
<constraint xsi:type="foreign" referenceId="FK_ORDER_CUSTOMER_ID"
            table="vendor_order" column="customer_id"
            referenceTable="customer_entity" referenceColumn="entity_id"
            onDelete="CASCADE"/>
```

**onDelete options:**

| Option | Behavior | Use Case |
|--------|----------|----------|
| `CASCADE` | Delete child rows when parent deleted | Order items deleted when order deleted |
| `SET NULL` | Set child column to NULL when parent deleted | Customer ID set to NULL when customer deleted |
| `NO ACTION` | Prevent parent deletion if children exist | Prevent category deletion if products exist |
| `RESTRICT` | Same as NO ACTION (MySQL default) | Prevent deletion |

**Attributes:**

- `referenceId`: Unique constraint identifier (e.g., `FK_VENDOR_ORDER_CUSTOMER_ID`)
- `table`: Current table name
- `column`: Current table column
- `referenceTable`: Parent table name
- `referenceColumn`: Parent table column
- `onDelete`: Action on parent deletion

#### Unique Constraint

```xml
<!-- Single column unique -->
<constraint xsi:type="unique" referenceId="VENDOR_TAG_UNIQUE_NAME">
    <column name="name"/>
</constraint>

<!-- Composite unique (multiple columns) -->
<constraint xsi:type="unique" referenceId="VENDOR_REVIEW_UNIQUE_CUSTOMER_PRODUCT">
    <column name="customer_id"/>
    <column name="product_id"/>
</constraint>
```

### Indexes

```xml
<!-- Single column index -->
<index referenceId="VENDOR_REVIEW_PRODUCT_ID" indexType="btree">
    <column name="product_id"/>
</index>

<!-- Composite index (order matters for query performance) -->
<index referenceId="VENDOR_REVIEW_STATUS_CREATED" indexType="btree">
    <column name="status"/>
    <column name="created_at"/>
</index>

<!-- Full-text index (for text search) -->
<index referenceId="VENDOR_REVIEW_FULLTEXT" indexType="fulltext">
    <column name="title"/>
    <column name="detail"/>
</index>
```

**Index types:**

| Type | Description | Use Case |
|------|-------------|----------|
| `btree` | B-tree index (default) | Standard lookups, range queries |
| `fulltext` | Full-text search index | Text search with MATCH AGAINST |
| `hash` | Hash index (Memory engine only) | Exact match lookups |

**Index best practices:**

1. **Foreign key columns**: Always index foreign key columns
2. **Where clause columns**: Index columns frequently used in WHERE
3. **Composite indexes**: Order by cardinality (most selective first)
4. **Covering indexes**: Include columns used in SELECT to avoid table lookups
5. **Limit indexes**: Too many indexes slow down INSERT/UPDATE

**Example: Optimize query**

```sql
-- Query: Find approved reviews for product, sorted by date
SELECT * FROM vendor_product_review
WHERE product_id = 123 AND status = 1
ORDER BY created_at DESC;

-- Optimal index: (product_id, status, created_at)
<index referenceId="VENDOR_REVIEW_PRODUCT_STATUS_DATE" indexType="btree">
    <column name="product_id"/>
    <column name="status"/>
    <column name="created_at"/>
</index>
```

## Generating db_schema_whitelist.json

The whitelist tracks which schema changes are applied. Magento auto-generates this file; **never edit manually**.

### Generate Whitelist

```bash
# Generate whitelist for all modules
bin/magento setup:db-declaration:generate-whitelist

# Generate whitelist for specific module
bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module
```

**Output**: `etc/db_schema_whitelist.json`

```json
{
    "vendor_product_review": {
        "column": {
            "review_id": true,
            "product_id": true,
            "customer_id": true,
            "nickname": true,
            "title": true,
            "detail": true,
            "rating": true,
            "status": true,
            "created_at": true,
            "updated_at": true
        },
        "constraint": {
            "PRIMARY": true,
            "FK_REVIEW_PRODUCT_ID": true,
            "FK_REVIEW_CUSTOMER_ID": true,
            "VENDOR_REVIEW_UNIQUE_CUSTOMER_PRODUCT": true
        },
        "index": {
            "VENDOR_REVIEW_PRODUCT_ID": true,
            "VENDOR_REVIEW_CUSTOMER_ID": true,
            "VENDOR_REVIEW_STATUS_CREATED": true
        }
    }
}
```

### Apply Schema Changes

```bash
# Dry-run: Preview SQL without executing
bin/magento setup:db-declaration:generate-patch --revertable Vendor_Module

# Apply schema changes
bin/magento setup:upgrade

# Verify schema
bin/magento setup:db:status
```

## Data Patches

Data patches insert or modify data in a version-independent, idempotent manner.

### Data Patch Structure

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\Patch\PatchRevertableInterface;
use Magento\Framework\Setup\Patch\PatchVersionInterface;

/**
 * Install default product review statuses
 */
class InstallReviewStatuses implements
    DataPatchInterface,
    PatchRevertableInterface,
    PatchVersionInterface
{
    /**
     * @param ModuleDataSetupInterface $moduleDataSetup
     */
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup
    ) {
    }

    /**
     * Apply patch: Insert default review statuses
     *
     * @return void
     */
    public function apply(): void
    {
        $this->moduleDataSetup->getConnection()->startSetup();

        $tableName = $this->moduleDataSetup->getTable('vendor_review_status');

        // Check if data already exists (idempotency)
        $select = $this->moduleDataSetup->getConnection()->select()
            ->from($tableName)
            ->where('status_id = ?', 1);

        if ($this->moduleDataSetup->getConnection()->fetchOne($select)) {
            $this->moduleDataSetup->getConnection()->endSetup();
            return; // Data already exists, skip
        }

        // Insert data
        $data = [
            ['status_id' => 1, 'status_code' => 'pending', 'label' => 'Pending Approval'],
            ['status_id' => 2, 'status_code' => 'approved', 'label' => 'Approved'],
            ['status_id' => 3, 'status_code' => 'rejected', 'label' => 'Rejected'],
        ];

        $this->moduleDataSetup->getConnection()->insertMultiple($tableName, $data);

        $this->moduleDataSetup->getConnection()->endSetup();
    }

    /**
     * Revert patch: Remove installed statuses
     *
     * @return void
     */
    public function revert(): void
    {
        $this->moduleDataSetup->getConnection()->startSetup();

        $tableName = $this->moduleDataSetup->getTable('vendor_review_status');

        $this->moduleDataSetup->getConnection()->delete(
            $tableName,
            ['status_id IN (?)' => [1, 2, 3]]
        );

        $this->moduleDataSetup->getConnection()->endSetup();
    }

    /**
     * Get patch dependencies (execute after these patches)
     *
     * @return array<class-string>
     */
    public static function getDependencies(): array
    {
        return [
            // Example: Run after another patch
            // \Vendor\Module\Setup\Patch\Data\OtherPatch::class
        ];
    }

    /**
     * Get patch version (optional, for documentation)
     *
     * @return string
     */
    public static function getVersion(): string
    {
        return '1.0.0';
    }

    /**
     * Get aliases (old patch class names, for BC)
     *
     * @return array<class-string>
     */
    public function getAliases(): array
    {
        return [];
    }
}
```

### Data Patch Interfaces

| Interface | Required | Purpose | Methods |
|-----------|----------|---------|---------|
| `DataPatchInterface` | **Yes** | Mark class as data patch | `apply()`, `getDependencies()`, `getAliases()` |
| `PatchRevertableInterface` | No | Enable rollback | `revert()` |
| `PatchVersionInterface` | No | Document version | `getVersion()` |

### Data Patch Best Practices

1. **Idempotency**: Always check if data exists before inserting

```php
public function apply(): void
{
    // Check if data already exists
    $select = $this->connection->select()
        ->from($tableName)
        ->where('identifier = ?', 'my_data');

    if ($this->connection->fetchOne($select)) {
        return; // Skip if exists
    }

    // Insert data
    $this->connection->insert($tableName, $data);
}
```

2. **Transactions**: Wrap in startSetup/endSetup for atomicity

```php
public function apply(): void
{
    $this->moduleDataSetup->getConnection()->startSetup();
    try {
        // Data operations
        $this->moduleDataSetup->getConnection()->commit();
    } catch (\Exception $e) {
        $this->moduleDataSetup->getConnection()->rollBack();
        throw $e;
    }
    $this->moduleDataSetup->getConnection()->endSetup();
}
```

3. **Batch processing**: For large datasets, process in chunks

```php
public function apply(): void
{
    $batchSize = 1000;
    $offset = 0;

    do {
        $data = $this->getDataBatch($offset, $batchSize);

        if (empty($data)) {
            break;
        }

        $this->connection->insertMultiple($tableName, $data);
        $offset += $batchSize;
    } while (count($data) === $batchSize);
}
```

4. **Use repositories, not raw SQL**: Prefer service contracts for complex logic

```php
public function apply(): void
{
    // AVOID: Raw SQL for complex business logic
    $this->connection->query("INSERT INTO ...");

    // PREFER: Use repository/service contract
    $review = $this->reviewFactory->create();
    $review->setData($data);
    $this->reviewRepository->save($review);
}
```

### Data Patch Example: Migrate Configuration

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\App\Config\Storage\WriterInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

/**
 * Migrate old config path to new path
 */
class MigrateConfigPath implements DataPatchInterface
{
    private const OLD_PATH = 'vendor/old_section/old_field';
    private const NEW_PATH = 'vendor/new_section/new_field';

    public function __construct(
        private readonly WriterInterface $configWriter,
        private readonly \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
    ) {
    }

    public function apply(): void
    {
        $oldValue = $this->scopeConfig->getValue(self::OLD_PATH);

        if ($oldValue !== null) {
            // Copy to new path
            $this->configWriter->save(self::NEW_PATH, $oldValue);

            // Delete old path
            $this->configWriter->delete(self::OLD_PATH);
        }
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

## Schema Patches

Schema patches handle complex schema changes that can't be expressed in `db_schema.xml`.

### Schema Patch Structure

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Schema;

use Magento\Framework\Setup\SchemaSetupInterface;
use Magento\Framework\Setup\Patch\SchemaPatchInterface;
use Magento\Framework\Setup\Patch\PatchRevertableInterface;

/**
 * Add computed column for review sentiment score
 */
class AddReviewSentimentScore implements
    SchemaPatchInterface,
    PatchRevertableInterface
{
    public function __construct(
        private readonly SchemaSetupInterface $schemaSetup
    ) {
    }

    /**
     * Apply patch: Add generated column
     *
     * @return void
     */
    public function apply(): void
    {
        $this->schemaSetup->startSetup();

        $connection = $this->schemaSetup->getConnection();
        $tableName = $this->schemaSetup->getTable('vendor_product_review');

        // Check if column already exists
        if ($connection->tableColumnExists($tableName, 'sentiment_score')) {
            $this->schemaSetup->endSetup();
            return;
        }

        // Add computed column (MySQL 5.7+)
        // sentiment_score = rating * 20 (convert 1-5 to 20-100)
        $connection->addColumn(
            $tableName,
            'sentiment_score',
            [
                'type' => \Magento\Framework\DB\Ddl\Table::TYPE_SMALLINT,
                'unsigned' => true,
                'nullable' => false,
                'default' => 0,
                'comment' => 'Sentiment Score (0-100)',
            ]
        );

        // Update existing rows
        $connection->query(
            "UPDATE {$tableName} SET sentiment_score = rating * 20"
        );

        $this->schemaSetup->endSetup();
    }

    /**
     * Revert patch: Remove column
     *
     * @return void
     */
    public function revert(): void
    {
        $this->schemaSetup->startSetup();

        $connection = $this->schemaSetup->getConnection();
        $tableName = $this->schemaSetup->getTable('vendor_product_review');

        if ($connection->tableColumnExists($tableName, 'sentiment_score')) {
            $connection->dropColumn($tableName, 'sentiment_score');
        }

        $this->schemaSetup->endSetup();
    }

    /**
     * Get dependencies
     *
     * @return array<class-string>
     */
    public static function getDependencies(): array
    {
        return [];
    }

    /**
     * Get aliases
     *
     * @return array<class-string>
     */
    public function getAliases(): array
    {
        return [];
    }
}
```

### Schema Patch Use Cases

| Use Case | Why Schema Patch | Example |
|----------|------------------|---------|
| Computed columns | Not supported in db_schema.xml | Generated columns, triggers |
| Data migration | Transform data during schema change | Split column into two |
| Complex constraints | CHECK constraints | Validate enum values |
| Stored procedures | Not supported in db_schema.xml | Create stored procedure |
| Partitioning | Not supported in db_schema.xml | Partition large table |

### Schema Patch Example: Split Column

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Schema;

use Magento\Framework\Setup\SchemaSetupInterface;
use Magento\Framework\Setup\Patch\SchemaPatchInterface;

/**
 * Split full_name into first_name and last_name
 */
class SplitCustomerName implements SchemaPatchInterface
{
    public function __construct(
        private readonly SchemaSetupInterface $schemaSetup
    ) {
    }

    public function apply(): void
    {
        $this->schemaSetup->startSetup();
        $connection = $this->schemaSetup->getConnection();
        $tableName = $this->schemaSetup->getTable('vendor_customer');

        // Add new columns
        if (!$connection->tableColumnExists($tableName, 'first_name')) {
            $connection->addColumn(
                $tableName,
                'first_name',
                [
                    'type' => \Magento\Framework\DB\Ddl\Table::TYPE_TEXT,
                    'length' => 128,
                    'nullable' => true,
                    'comment' => 'First Name',
                ]
            );
        }

        if (!$connection->tableColumnExists($tableName, 'last_name')) {
            $connection->addColumn(
                $tableName,
                'last_name',
                [
                    'type' => \Magento\Framework\DB\Ddl\Table::TYPE_TEXT,
                    'length' => 128,
                    'nullable' => true,
                    'comment' => 'Last Name',
                ]
            );
        }

        // Migrate data: Split full_name on first space
        $connection->query("
            UPDATE {$tableName}
            SET
                first_name = SUBSTRING_INDEX(full_name, ' ', 1),
                last_name = TRIM(SUBSTRING(full_name, LOCATE(' ', full_name) + 1))
            WHERE full_name IS NOT NULL
              AND full_name != ''
        ");

        $this->schemaSetup->endSetup();
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

## Patch Dependencies and Versioning

### Patch Dependencies

Control execution order with `getDependencies()`:

```php
class MyDataPatch implements DataPatchInterface
{
    /**
     * Execute after these patches
     *
     * @return array<class-string>
     */
    public static function getDependencies(): array
    {
        return [
            \Vendor\Module\Setup\Patch\Data\PatchA::class,
            \Vendor\Module\Setup\Patch\Data\PatchB::class,
        ];
    }
}
```

**Execution order:**

1. Magento builds dependency graph
2. Executes patches in topological order (dependencies first)
3. Tracks applied patches in `patch_list` table

**View applied patches:**

```sql
SELECT * FROM patch_list WHERE patch_name LIKE 'Vendor_Module%';
```

### Patch Versioning

Use `PatchVersionInterface` for documentation (not enforced by Magento):

```php
class MyPatch implements DataPatchInterface, PatchVersionInterface
{
    public static function getVersion(): string
    {
        return '1.0.0'; // Semantic version
    }
}
```

**Use case**: Track which patches correspond to module versions for rollback planning.

### Patch Aliases

Use `getAliases()` for backward compatibility when renaming patches:

```php
class NewPatchName implements DataPatchInterface
{
    public function getAliases(): array
    {
        return [
            \Vendor\Module\Setup\Patch\Data\OldPatchName::class,
        ];
    }
}
```

Magento treats aliases as the same patch, preventing duplicate execution.

## Migration from Legacy Setup Scripts

### Legacy Setup Structure (Magento 2.0-2.2)

```
Vendor/Module/Setup/
├── InstallSchema.php       # Initial schema installation
├── UpgradeSchema.php       # Schema upgrades (version-based)
├── InstallData.php         # Initial data installation
├── UpgradeData.php         # Data upgrades (version-based)
└── Recurring.php           # Runs on every setup:upgrade
```

### Migration Steps

#### Step 1: Convert InstallSchema to db_schema.xml

**Legacy InstallSchema.php:**

```php
public function install(SchemaSetupInterface $setup, ModuleContextInterface $context)
{
    $setup->startSetup();

    $table = $setup->getConnection()->newTable(
        $setup->getTable('vendor_review')
    )->addColumn(
        'review_id',
        \Magento\Framework\DB\Ddl\Table::TYPE_INTEGER,
        null,
        ['identity' => true, 'unsigned' => true, 'nullable' => false, 'primary' => true],
        'Review ID'
    )->addColumn(
        'title',
        \Magento\Framework\DB\Ddl\Table::TYPE_TEXT,
        255,
        ['nullable' => false],
        'Title'
    )->addIndex(
        $setup->getIdxName('vendor_review', ['title']),
        ['title']
    );

    $setup->getConnection()->createTable($table);
    $setup->endSetup();
}
```

**New db_schema.xml:**

```xml
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    <table name="vendor_review" resource="default" engine="innodb" comment="Review Table">
        <column xsi:type="int" name="review_id" unsigned="true" nullable="false" identity="true" comment="Review ID"/>
        <column xsi:type="varchar" name="title" length="255" nullable="false" comment="Title"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="review_id"/>
        </constraint>

        <index referenceId="VENDOR_REVIEW_TITLE" indexType="btree">
            <column name="title"/>
        </index>
    </table>
</schema>
```

#### Step 2: Convert UpgradeSchema to db_schema.xml

**Legacy UpgradeSchema.php:**

```php
public function upgrade(SchemaSetupInterface $setup, ModuleContextInterface $context)
{
    $setup->startSetup();

    if (version_compare($context->getVersion(), '1.0.1', '<')) {
        $setup->getConnection()->addColumn(
            $setup->getTable('vendor_review'),
            'status',
            [
                'type' => \Magento\Framework\DB\Ddl\Table::TYPE_SMALLINT,
                'unsigned' => true,
                'nullable' => false,
                'default' => 0,
                'comment' => 'Status',
            ]
        );
    }

    $setup->endSetup();
}
```

**New db_schema.xml (add status column):**

```xml
<table name="vendor_review" resource="default" engine="innodb" comment="Review Table">
    <column xsi:type="int" name="review_id" unsigned="true" nullable="false" identity="true" comment="Review ID"/>
    <column xsi:type="varchar" name="title" length="255" nullable="false" comment="Title"/>
    <!-- NEW: Add status column -->
    <column xsi:type="smallint" name="status" unsigned="true" nullable="false" default="0" comment="Status"/>

    <constraint xsi:type="primary" referenceId="PRIMARY">
        <column name="review_id"/>
    </constraint>

    <index referenceId="VENDOR_REVIEW_TITLE" indexType="btree">
        <column name="title"/>
    </index>
</table>
```

No version checks needed! Magento compares desired state with current state.

#### Step 3: Convert InstallData to Data Patch

**Legacy InstallData.php:**

```php
public function install(ModuleDataSetupInterface $setup, ModuleContextInterface $context)
{
    $setup->startSetup();

    $data = [
        ['status_id' => 1, 'label' => 'Pending'],
        ['status_id' => 2, 'label' => 'Approved'],
    ];

    $setup->getConnection()->insertMultiple(
        $setup->getTable('vendor_review_status'),
        $data
    );

    $setup->endSetup();
}
```

**New Data Patch (Setup/Patch/Data/InstallStatuses.php):**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class InstallStatuses implements DataPatchInterface
{
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup
    ) {
    }

    public function apply(): void
    {
        $this->moduleDataSetup->getConnection()->startSetup();

        $tableName = $this->moduleDataSetup->getTable('vendor_review_status');

        // Check if data exists (idempotency)
        $select = $this->moduleDataSetup->getConnection()->select()
            ->from($tableName)
            ->limit(1);

        if ($this->moduleDataSetup->getConnection()->fetchOne($select)) {
            $this->moduleDataSetup->getConnection()->endSetup();
            return;
        }

        $data = [
            ['status_id' => 1, 'label' => 'Pending'],
            ['status_id' => 2, 'label' => 'Approved'],
        ];

        $this->moduleDataSetup->getConnection()->insertMultiple($tableName, $data);
        $this->moduleDataSetup->getConnection()->endSetup();
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

#### Step 4: Remove Legacy Files

After migration:

1. Delete `Setup/InstallSchema.php`
2. Delete `Setup/UpgradeSchema.php`
3. Delete `Setup/InstallData.php`
4. Delete `Setup/UpgradeData.php`
5. Keep `Setup/Recurring.php` if still needed (rare)

#### Step 5: Generate Whitelist and Test

```bash
# Generate whitelist
bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module

# Dry-run to verify changes
bin/magento setup:db-declaration:generate-patch --revertable Vendor_Module

# Apply changes
bin/magento setup:upgrade
```

## Rollback Strategies

### Schema Rollback

**Revertable patches** (implement `PatchRevertableInterface`):

```php
class MyPatch implements DataPatchInterface, PatchRevertableInterface
{
    public function apply(): void
    {
        // Apply changes
    }

    public function revert(): void
    {
        // Undo changes
    }
}
```

**Revert patch:**

```bash
# Revert specific patch
bin/magento module:uninstall Vendor_Module --non-composer

# Or manually delete from patch_list and run revert logic
DELETE FROM patch_list WHERE patch_name = 'Vendor\\Module\\Setup\\Patch\\Data\\MyPatch';
```

**Limitations:**
- Magento doesn't auto-revert patches
- Must manually trigger revert logic or uninstall module
- Data loss risk (e.g., dropping column loses data)

### Safe Rollback Pattern

**Phase 1: Add new column (backward compatible)**

```xml
<!-- Version 1.0.1: Add new column, keep old column -->
<table name="vendor_customer">
    <column xsi:type="varchar" name="old_field" length="255" nullable="true"/>
    <column xsi:type="varchar" name="new_field" length="255" nullable="true"/>
</table>
```

**Phase 2: Migrate data**

```php
// Data patch: Copy old_field to new_field
public function apply(): void
{
    $connection->query("
        UPDATE vendor_customer
        SET new_field = old_field
        WHERE new_field IS NULL
    ");
}
```

**Phase 3: Dual-write period (application code writes to both fields)**

```php
// In application code
$customer->setData('old_field', $value);
$customer->setData('new_field', $value); // Dual write
```

**Phase 4: Remove old column (next major version)**

```xml
<!-- Version 2.0.0: Remove old column -->
<table name="vendor_customer">
    <column xsi:type="varchar" name="new_field" length="255" nullable="false"/>
    <!-- old_field removed from db_schema.xml -->
</table>
```

**Rollback**: Revert to version 1.x.x, old column still exists.

### Backup Before Schema Changes

```bash
# Backup database before upgrade
mysqldump -u root -p magento2 > backup_before_upgrade.sql

# Apply upgrade
bin/magento setup:upgrade

# If issues, restore backup
mysql -u root -p magento2 < backup_before_upgrade.sql
```

## Production Checklist

Before deploying declarative schema changes to production:

### 1. Test in Staging

- Apply changes in staging environment identical to production
- Run full regression test suite
- Verify data integrity after migration

### 2. Review Generated SQL

```bash
# Generate SQL preview
bin/magento setup:db-declaration:generate-patch Vendor_Module --revertable

# Review output in Setup/Patch/Schema/*.php
# Ensure no destructive operations (DROP COLUMN, etc.)
```

### 3. Backup Database

```bash
mysqldump -u root -p magento2 > pre_upgrade_backup.sql
```

### 4. Estimate Downtime

- Large tables: Adding columns with defaults can lock table (MySQL 5.7)
- MySQL 8.0+: Instant ADD COLUMN (most cases)
- Foreign keys: Can cause cascading locks

**Test duration in staging:**

```bash
time bin/magento setup:upgrade
# Measure time for production planning
```

### 5. Maintenance Mode

```bash
bin/magento maintenance:enable
bin/magento setup:upgrade
bin/magento cache:flush
bin/magento maintenance:disable
```

### 6. Monitor Performance

After deployment:

- Check slow query log for new N+1 queries
- Verify indexes are used: `EXPLAIN SELECT ...`
- Monitor database CPU/memory usage

## Performance Optimization

### Index Strategy

**Query patterns:**

```sql
-- Frequent query: Get approved reviews for product
SELECT * FROM vendor_review
WHERE product_id = 123 AND status = 1
ORDER BY created_at DESC;

-- Optimal index: Covering index
<index referenceId="VENDOR_REVIEW_PRODUCT_STATUS_DATE" indexType="btree">
    <column name="product_id"/>
    <column name="status"/>
    <column name="created_at"/>
</index>
```

**Verify index usage:**

```sql
EXPLAIN SELECT * FROM vendor_review
WHERE product_id = 123 AND status = 1
ORDER BY created_at DESC;

-- Check "key" column shows your index name
```

### Foreign Key Performance

**Cascading deletes** can lock many tables:

```xml
<!-- SLOW: Cascade delete thousands of rows -->
<constraint xsi:type="foreign" referenceId="FK_REVIEW_PRODUCT"
            table="vendor_review" column="product_id"
            referenceTable="catalog_product_entity" referenceColumn="entity_id"
            onDelete="CASCADE"/>

<!-- FASTER: SET NULL (no cascade), handle in application -->
<constraint xsi:type="foreign" referenceId="FK_REVIEW_PRODUCT"
            table="vendor_review" column="product_id"
            referenceTable="catalog_product_entity" referenceColumn="entity_id"
            onDelete="SET NULL"/>
```

Manually delete orphaned rows in batch:

```php
// Cron job or scheduled task
$connection->delete(
    'vendor_review',
    ['product_id IS NULL']
);
```

### Large Table Considerations

For tables >1M rows:

1. **Add indexes before data**: Faster than adding after
2. **Use ALGORITHM=INPLACE** (MySQL 8.0): Non-blocking schema changes
3. **Partition tables**: Split by date range for better performance
4. **Archive old data**: Move to separate archive table

**Example: Partition by date**

```php
// Schema patch: Partition review table by year
public function apply(): void
{
    $connection->query("
        ALTER TABLE vendor_review
        PARTITION BY RANGE (YEAR(created_at)) (
            PARTITION p2023 VALUES LESS THAN (2024),
            PARTITION p2024 VALUES LESS THAN (2025),
            PARTITION p2025 VALUES LESS THAN (2026),
            PARTITION pmax VALUES LESS THAN MAXVALUE
        )
    ");
}
```

## Assumptions

- Target: Adobe Commerce / Magento Open Source 2.4.7
- Database: MySQL 8.0 (declarative schema requires 2.3+)
- PHP: 8.2+
- Environment: Development and production
- Scope: Backend (PHP modules, database schema)

## Why This Approach

- **Declarative over Procedural**: Version-independent, single source of truth
- **Patches over Upgrade Scripts**: Idempotent, explicit dependencies
- **db_schema.xml for Structure**: Standard schema changes
- **Patches for Complex Logic**: Data transformations, computed columns
- **Revertable Patches**: Rollback support for safe deployments
- **Whitelist Auto-Generation**: Tracks applied changes, prevents drift

Alternatives considered:
- **Legacy InstallSchema/UpgradeSchema**: Deprecated, version-dependent, no rollback
- **Raw SQL in patches**: Less portable, no validation
- **Manual schema changes**: Not tracked, breaks deployments

## Security Impact

### Authentication/Authorization

- Schema changes don't directly affect auth, but ensure ACL resources updated if adding admin features
- Data patches: Don't expose sensitive data in logs

### CSRF/Form Keys

- N/A for database schema

### XSS Escaping

- N/A for database schema

### PII/GDPR

- **Sensitive columns**: Mark with appropriate comment for GDPR compliance
- Example: `<column xsi:type="varchar" name="email" comment="Email (PII)"/>`
- Data patches: Anonymize customer data if loading from production dumps
- Implement data export/deletion per GDPR requirements in application layer

### Secrets Management

- Never store plaintext secrets in database
- Use encrypted columns for sensitive data (payment tokens, API keys)
- Magento encryption: `Magento\Framework\Encryption\EncryptorInterface`

## Performance Impact

### Full Page Cache (FPC)

- Schema changes don't affect FPC directly
- Data patches: Flush cache after modifying config data

### Database Load

- **Adding indexes**: Can lock table (test duration in staging)
- **Foreign keys**: Cascading deletes can be slow (prefer SET NULL + manual cleanup)
- **Large tables**: Adding columns with defaults can be slow (MySQL 5.7)
- **MySQL 8.0 Instant ADD COLUMN**: Most ADD COLUMN operations instant (no table rebuild)

### Core Web Vitals (CWV)

- Database performance affects page load times
- Ensure proper indexing for frontend queries
- Use query profiler to measure impact: `EXPLAIN SELECT ...`

### Cacheability

- Data patches: Ensure cache invalidation if modifying cached data
- Example: After updating config, call `bin/magento cache:flush config`

## Backward Compatibility

### API/DB Schema Changes

- **Adding columns**: Backward compatible (existing code ignores new columns)
- **Removing columns**: **Breaking change** (code referencing column fails)
- **Renaming columns**: **Breaking change** (requires dual-write period)
- **Changing column types**: **Risky** (data loss if incompatible)

**Safe migration pattern** (rename column):

1. Add new column (BC)
2. Dual-write both columns (BC)
3. Migrate data (BC)
4. Switch code to new column (BC)
5. Remove old column (next major version, breaking)

### Upgrade Path

- Magento 2.4.6 → 2.4.7: Declarative schema fully supported
- Magento 2.4.7 → 2.4.8/2.4.9: Continue using declarative schema
- MySQL 5.7 → 8.0: Test schema changes (syntax differences minimal)

### Migration Notes

- Review MySQL 8.0 reserved keywords if upgrading from 5.7
- Test foreign key constraints (behavior may differ)
- Verify index types supported (FULLTEXT requires InnoDB)

## Tests to Add

### Unit Tests

- Test data patch idempotency: Run `apply()` twice, verify no errors
- Test schema patch column checks: Verify skips if column exists

### Integration Tests

```php
/**
 * @magentoDbIsolation enabled
 */
public function testSchemaTableExists(): void
{
    $connection = $this->getConnection();
    $tableName = $this->resource->getTableName('vendor_review');

    $this->assertTrue($connection->isTableExists($tableName));
}

/**
 * @magentoDbIsolation enabled
 */
public function testSchemaColumnExists(): void
{
    $connection = $this->getConnection();
    $tableName = $this->resource->getTableName('vendor_review');

    $this->assertTrue($connection->tableColumnExists($tableName, 'review_id'));
}
```

### Functional Tests (MFTF)

- Test end-to-end flows that depend on new schema
- Example: Create review, verify data saved correctly

### Performance Tests

- Measure index usage with `EXPLAIN`
- Measure query duration before/after index addition
- Load test with realistic data volumes

## Documentation to Update

### Module README

```markdown
## Database Schema

This module adds the following tables:

- `vendor_product_review`: Product reviews
- `vendor_review_status`: Review status types
- `vendor_review_tag`: Many-to-many review-tag relation

To install schema:

1. `bin/magento setup:upgrade`
2. `bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module`
```

### CHANGELOG

```markdown
## [1.0.0] - 2026-02-05
### Added
- Database schema: `vendor_product_review`, `vendor_review_status`, `vendor_review_tag`
- Data patch: Install default review statuses
- Schema patch: Add sentiment score computed column

### Changed
- Migrated from legacy InstallSchema to declarative schema
```

### Upgrade Guide

```markdown
## Upgrading from 0.9.x to 1.0.0

**Breaking Changes:**

- Removed `old_field` column from `vendor_customer` table. Applications must use `new_field`.

**Migration Steps:**

1. Backup database: `mysqldump magento2 > backup.sql`
2. Run: `bin/magento setup:upgrade`
3. Verify data integrity in `vendor_customer` table
4. Update custom code to use `new_field`
```

### Admin User Guide

- Document any user-visible changes caused by schema updates
- Example: "Review statuses now include 'Rejected' option (v1.0.0+)"

## Related Documentation

### Related Guides

- [EAV System Architecture: Understanding Entity-Attribute-Value in Magento 2](../explanations/eav-system.md)
- [Upgrade Guide: Adobe Commerce 2.4.6 to 2.4.7/2.4.8](../references/upgrade-guide-247-248.md)
- [Plugin System Deep Dive: Mastering Magento 2 Interception](plugin-system-deep-dive.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Customer Overview](../../modules/customer/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
