---
title: "Building Admin UI Components in Magento 2"
description: "Developer guide: Building Admin UI Components in Magento 2"
type: "how-to"
tier: 2
difficulty: "advanced"
version: "2.4.7+"
estimated_time: "30 minutes"
topics:
  - admin ui
  - ui components
  - admin grids
  - admin forms
  - data providers
  - mass actions
last_updated: "2026-02-07"
---

# Building Admin UI Components in Magento 2

## Overview

Magento's UI Component framework provides a powerful, declarative system for building admin interfaces. This guide covers admin grids, forms, data providers, mass actions, inline editing, and AJAX operations with production-ready examples.

**What you'll learn:**
- Admin grid structure with ui_component XML
- Admin form configuration (fieldsets, fields, data providers)
- Mass actions implementation
- Inline editing and custom columns
- Custom filters and search
- AJAX operations in admin
- Data provider patterns and optimization

**Prerequisites:**
- PHP 8.2+ knowledge (typed properties, interfaces)
- Magento module structure and DI
- Understanding of XML configuration
- Basic JavaScript/Knockout.js knowledge

---

## UI Component Architecture

### Component Hierarchy

```
[UI Component XML] → [Data Provider] → [Repository/Resource Model] → [Database]
                  ↓
              [Layout XML]
                  ↓
            [Template/Block]
                  ↓
          [JavaScript Component]
```

### Core Concepts

1. **UI Component XML:** Declarative configuration for grid/form structure
2. **Data Provider:** Fetches and prepares data for UI components
3. **Layout XML:** Integrates UI component into admin page
4. **JavaScript Component:** Client-side behavior (KnockoutJS-based)

---

## Admin Grid Implementation

### Step 1: Define Grid UI Component

**File:** `view/adminhtml/ui_component/vendor_module_order_listing.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<listing xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Ui:etc/ui_configuration.xsd">

    <!-- Grid Settings -->
    <argument name="data" xsi:type="array">
        <item name="js_config" xsi:type="array">
            <item name="provider" xsi:type="string">vendor_module_order_listing.vendor_module_order_listing_data_source</item>
        </item>
    </argument>

    <!-- Data Source -->
    <settings>
        <buttons>
            <button name="add">
                <url path="*/*/new"/>
                <class>primary</class>
                <label translate="true">Add New Order</label>
            </button>
        </buttons>
        <spinner>vendor_module_order_columns</spinner>
        <deps>
            <dep>vendor_module_order_listing.vendor_module_order_listing_data_source</dep>
        </deps>
    </settings>

    <!-- Data Provider -->
    <dataSource name="vendor_module_order_listing_data_source" component="Magento_Ui/js/grid/provider">
        <settings>
            <storageConfig>
                <param name="indexField" xsi:type="string">entity_id</param>
            </storageConfig>
            <updateUrl path="mui/index/render"/>
        </settings>
        <aclResource>Vendor_Module::order_view</aclResource>
        <dataProvider class="Vendor\Module\Ui\Component\Listing\DataProvider" name="vendor_module_order_listing_data_source">
            <settings>
                <requestFieldName>id</requestFieldName>
                <primaryFieldName>entity_id</primaryFieldName>
            </settings>
        </dataProvider>
    </dataSource>

    <!-- Listing Toolbar -->
    <listingToolbar name="listing_top">
        <settings>
            <sticky>true</sticky>
        </settings>

        <!-- Bookmark (saved views) -->
        <bookmark name="bookmarks"/>

        <!-- Columns toggle -->
        <columnsControls name="columns_controls"/>

        <!-- Full-text search -->
        <filterSearch name="fulltext"/>

        <!-- Filters -->
        <filters name="listing_filters">
            <settings>
                <templates>
                    <filters>
                        <select>
                            <param name="template" xsi:type="string">ui/grid/filters/elements/ui-select</param>
                            <param name="component" xsi:type="string">Magento_Ui/js/form/element/ui-select</param>
                        </select>
                    </filters>
                </templates>
            </settings>

            <!-- Status filter -->
            <filterSelect name="status" provider="${ $.parentName }">
                <settings>
                    <captionValue>0</captionValue>
                    <options class="Vendor\Module\Model\Source\OrderStatus"/>
                    <label translate="true">Status</label>
                    <dataScope>status</dataScope>
                    <imports>
                        <link name="visible">componentType = column, index = ${ $.index }:visible</link>
                    </imports>
                </settings>
            </filterSelect>

            <!-- Date range filter -->
            <filterRange name="created_at" provider="${ $.parentName }">
                <settings>
                    <label translate="true">Created Date</label>
                    <dataScope>created_at</dataScope>
                    <rangeType>date</rangeType>
                </settings>
            </filterRange>
        </filters>

        <!-- Mass actions -->
        <massaction name="listing_massaction">
            <action name="delete">
                <settings>
                    <confirm>
                        <message translate="true">Are you sure you want to delete selected orders?</message>
                        <title translate="true">Delete Orders</title>
                    </confirm>
                    <url path="vendor_module/order/massDelete"/>
                    <type>delete</type>
                    <label translate="true">Delete</label>
                </settings>
            </action>
            <action name="status">
                <settings>
                    <type>status</type>
                    <label translate="true">Change Status</label>
                    <actions>
                        <action name="0">
                            <type>pending</type>
                            <label translate="true">Pending</label>
                            <url path="vendor_module/order/massStatus">
                                <param name="status">pending</param>
                            </url>
                        </action>
                        <action name="1">
                            <type>processing</type>
                            <label translate="true">Processing</label>
                            <url path="vendor_module/order/massStatus">
                                <param name="status">processing</param>
                            </url>
                        </action>
                        <action name="2">
                            <type>complete</type>
                            <label translate="true">Complete</label>
                            <url path="vendor_module/order/massStatus">
                                <param name="status">complete</param>
                            </url>
                        </action>
                    </actions>
                </settings>
            </action>
        </massaction>

        <!-- Pagination -->
        <paging name="listing_paging"/>
    </listingToolbar>

    <!-- Columns -->
    <columns name="vendor_module_order_columns">
        <!-- Selection column -->
        <selectionsColumn name="ids">
            <settings>
                <indexField>entity_id</indexField>
            </settings>
        </selectionsColumn>

        <!-- ID column -->
        <column name="entity_id">
            <settings>
                <filter>textRange</filter>
                <label translate="true">ID</label>
                <sorting>desc</sorting>
            </settings>
        </column>

        <!-- Increment ID column -->
        <column name="increment_id">
            <settings>
                <filter>text</filter>
                <label translate="true">Order #</label>
            </settings>
        </column>

        <!-- Customer name column -->
        <column name="customer_name">
            <settings>
                <filter>text</filter>
                <label translate="true">Customer</label>
            </settings>
        </column>

        <!-- Status column with options -->
        <column name="status" component="Magento_Ui/js/grid/columns/select">
            <settings>
                <options class="Vendor\Module\Model\Source\OrderStatus"/>
                <filter>select</filter>
                <dataType>select</dataType>
                <label translate="true">Status</label>
            </settings>
        </column>

        <!-- Grand total column with price formatting -->
        <column name="grand_total" class="Vendor\Module\Ui\Component\Listing\Column\Price">
            <settings>
                <filter>textRange</filter>
                <label translate="true">Grand Total</label>
            </settings>
        </column>

        <!-- Created at column with date formatting -->
        <column name="created_at" class="Magento\Ui\Component\Listing\Columns\Date" component="Magento_Ui/js/grid/columns/date">
            <settings>
                <filter>dateRange</filter>
                <dataType>date</dataType>
                <label translate="true">Created</label>
            </settings>
        </column>

        <!-- Actions column -->
        <actionsColumn name="actions" class="Vendor\Module\Ui\Component\Listing\Column\Actions">
            <settings>
                <indexField>entity_id</indexField>
                <resizeEnabled>false</resizeEnabled>
                <resizeDefaultWidth>107</resizeDefaultWidth>
            </settings>
        </actionsColumn>
    </columns>
</listing>
```

### Step 2: Create Data Provider

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Ui\Component\Listing;

use Magento\Framework\View\Element\UiComponent\DataProvider\DataProvider as AbstractDataProvider;
use Magento\Ui\DataProvider\Modifier\PoolInterface;
use Vendor\Module\Model\ResourceModel\Order\Grid\CollectionFactory;

class DataProvider extends AbstractDataProvider
{
    private PoolInterface $modifierPool;

    public function __construct(
        string $name,
        string $primaryFieldName,
        string $requestFieldName,
        CollectionFactory $collectionFactory,
        PoolInterface $modifierPool,
        array $meta = [],
        array $data = []
    ) {
        parent::__construct($name, $primaryFieldName, $requestFieldName, $meta, $data);
        $this->collection = $collectionFactory->create();
        $this->modifierPool = $modifierPool;
    }

    /**
     * Get data
     *
     * @return array
     */
    public function getData(): array
    {
        $data = parent::getData();

        // Apply modifiers
        foreach ($this->modifierPool->getModifiersInstances() as $modifier) {
            $data = $modifier->modifyData($data);
        }

        return $data;
    }

    /**
     * Get meta
     *
     * @return array
     */
    public function getMeta(): array
    {
        $meta = parent::getMeta();

        // Apply modifiers
        foreach ($this->modifierPool->getModifiersInstances() as $modifier) {
            $meta = $modifier->modifyMeta($meta);
        }

        return $meta;
    }
}
```

### Step 3: Grid Collection

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\ResourceModel\Order\Grid;

use Magento\Framework\View\Element\UiComponent\DataProvider\SearchResult;
use Vendor\Module\Model\ResourceModel\Order as OrderResource;

class Collection extends SearchResult
{
    /**
     * Initialize select
     *
     * @return $this
     */
    protected function _initSelect()
    {
        parent::_initSelect();

        // Join customer name
        $this->getSelect()->joinLeft(
            ['customer' => $this->getTable('customer_entity')],
            'main_table.customer_id = customer.entity_id',
            ['customer_name' => 'CONCAT(customer.firstname, " ", customer.lastname)']
        );

        return $this;
    }
}
```

**Register collection in `di.xml`:**

```xml
<virtualType name="Vendor\Module\Model\ResourceModel\Order\Grid\Collection" type="Magento\Framework\View\Element\UiComponent\DataProvider\SearchResult">
    <arguments>
        <argument name="mainTable" xsi:type="string">vendor_module_order</argument>
        <argument name="resourceModel" xsi:type="string">Vendor\Module\Model\ResourceModel\Order</argument>
    </arguments>
</virtualType>

<type name="Magento\Framework\View\Element\UiComponent\DataProvider\CollectionFactory">
    <arguments>
        <argument name="collections" xsi:type="array">
            <item name="vendor_module_order_listing_data_source" xsi:type="string">Vendor\Module\Model\ResourceModel\Order\Grid\Collection</item>
        </argument>
    </arguments>
</type>
```

### Step 4: Custom Column Renderers

**Price column:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Ui\Component\Listing\Column;

use Magento\Framework\View\Element\UiComponent\ContextInterface;
use Magento\Framework\View\Element\UiComponentFactory;
use Magento\Ui\Component\Listing\Columns\Column;
use Magento\Framework\Pricing\PriceCurrencyInterface;

class Price extends Column
{
    public function __construct(
        ContextInterface $context,
        UiComponentFactory $uiComponentFactory,
        private readonly PriceCurrencyInterface $priceCurrency,
        array $components = [],
        array $data = []
    ) {
        parent::__construct($context, $uiComponentFactory, $components, $data);
    }

    /**
     * Prepare data source
     *
     * @param array $dataSource
     * @return array
     */
    public function prepareDataSource(array $dataSource): array
    {
        if (isset($dataSource['data']['items'])) {
            $fieldName = $this->getData('name');

            foreach ($dataSource['data']['items'] as &$item) {
                if (isset($item[$fieldName])) {
                    $item[$fieldName] = $this->priceCurrency->format(
                        (float) $item[$fieldName],
                        false
                    );
                }
            }
        }

        return $dataSource;
    }
}
```

**Actions column:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Ui\Component\Listing\Column;

use Magento\Framework\View\Element\UiComponent\ContextInterface;
use Magento\Framework\View\Element\UiComponentFactory;
use Magento\Ui\Component\Listing\Columns\Column;
use Magento\Framework\UrlInterface;

class Actions extends Column
{
    private const URL_PATH_EDIT = 'vendor_module/order/edit';
    private const URL_PATH_DELETE = 'vendor_module/order/delete';

    public function __construct(
        ContextInterface $context,
        UiComponentFactory $uiComponentFactory,
        private readonly UrlInterface $urlBuilder,
        array $components = [],
        array $data = []
    ) {
        parent::__construct($context, $uiComponentFactory, $components, $data);
    }

    /**
     * Prepare data source
     *
     * @param array $dataSource
     * @return array
     */
    public function prepareDataSource(array $dataSource): array
    {
        if (isset($dataSource['data']['items'])) {
            foreach ($dataSource['data']['items'] as &$item) {
                if (isset($item['entity_id'])) {
                    $item[$this->getData('name')] = [
                        'edit' => [
                            'href' => $this->urlBuilder->getUrl(
                                static::URL_PATH_EDIT,
                                ['id' => $item['entity_id']]
                            ),
                            'label' => __('Edit')
                        ],
                        'delete' => [
                            'href' => $this->urlBuilder->getUrl(
                                static::URL_PATH_DELETE,
                                ['id' => $item['entity_id']]
                            ),
                            'label' => __('Delete'),
                            'confirm' => [
                                'title' => __('Delete Order'),
                                'message' => __('Are you sure you want to delete this order?')
                            ]
                        ]
                    ];
                }
            }
        }

        return $dataSource;
    }
}
```

### Step 5: Layout XML

**File:** `view/adminhtml/layout/vendor_module_order_index.xml`

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <update handle="styles"/>
    <body>
        <referenceContainer name="content">
            <uiComponent name="vendor_module_order_listing"/>
        </referenceContainer>
    </body>
</page>
```

---

## Admin Form Implementation

### Step 1: Define Form UI Component

**File:** `view/adminhtml/ui_component/vendor_module_order_form.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<form xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Ui:etc/ui_configuration.xsd">

    <!-- Form Settings -->
    <argument name="data" xsi:type="array">
        <item name="js_config" xsi:type="array">
            <item name="provider" xsi:type="string">vendor_module_order_form.vendor_module_order_form_data_source</item>
        </item>
        <item name="label" xsi:type="string" translate="true">Order Information</item>
        <item name="template" xsi:type="string">templates/form/collapsible</item>
    </argument>

    <settings>
        <buttons>
            <button name="back">
                <url path="*/*/"/>
                <class>back</class>
                <label translate="true">Back</label>
            </button>
            <button name="delete" class="Vendor\Module\Block\Adminhtml\Order\Edit\DeleteButton"/>
            <button name="save" class="Vendor\Module\Block\Adminhtml\Order\Edit\SaveButton"/>
            <button name="save_and_continue" class="Vendor\Module\Block\Adminhtml\Order\Edit\SaveAndContinueButton"/>
        </buttons>
        <namespace>vendor_module_order_form</namespace>
        <dataScope>data</dataScope>
        <deps>
            <dep>vendor_module_order_form.vendor_module_order_form_data_source</dep>
        </deps>
    </settings>

    <!-- Data Source -->
    <dataSource name="vendor_module_order_form_data_source">
        <argument name="data" xsi:type="array">
            <item name="js_config" xsi:type="array">
                <item name="component" xsi:type="string">Magento_Ui/js/form/provider</item>
            </item>
        </argument>
        <settings>
            <submitUrl path="vendor_module/order/save"/>
        </settings>
        <dataProvider class="Vendor\Module\Ui\Component\Form\DataProvider" name="vendor_module_order_form_data_source">
            <settings>
                <requestFieldName>id</requestFieldName>
                <primaryFieldName>entity_id</primaryFieldName>
            </settings>
        </dataProvider>
    </dataSource>

    <!-- Fieldsets -->
    <fieldset name="general">
        <settings>
            <label translate="true">General Information</label>
            <collapsible>true</collapsible>
            <opened>true</opened>
        </settings>

        <!-- Increment ID field -->
        <field name="increment_id" formElement="input">
            <settings>
                <dataType>text</dataType>
                <label translate="true">Order Number</label>
                <validation>
                    <rule name="required-entry" xsi:type="boolean">true</rule>
                </validation>
            </settings>
        </field>

        <!-- Customer email field -->
        <field name="customer_email" formElement="input">
            <settings>
                <dataType>text</dataType>
                <label translate="true">Customer Email</label>
                <validation>
                    <rule name="required-entry" xsi:type="boolean">true</rule>
                    <rule name="validate-email" xsi:type="boolean">true</rule>
                </validation>
            </settings>
        </field>

        <!-- Status field (select) -->
        <field name="status" formElement="select">
            <settings>
                <dataType>text</dataType>
                <label translate="true">Status</label>
                <validation>
                    <rule name="required-entry" xsi:type="boolean">true</rule>
                </validation>
            </settings>
            <formElements>
                <select>
                    <settings>
                        <options class="Vendor\Module\Model\Source\OrderStatus"/>
                    </settings>
                </select>
            </formElements>
        </field>

        <!-- Grand total field (price) -->
        <field name="grand_total" formElement="input">
            <settings>
                <dataType>text</dataType>
                <label translate="true">Grand Total</label>
                <validation>
                    <rule name="validate-number" xsi:type="boolean">true</rule>
                    <rule name="validate-zero-or-greater" xsi:type="boolean">true</rule>
                </validation>
            </settings>
        </field>

        <!-- Created at field (date) -->
        <field name="created_at" formElement="date">
            <settings>
                <dataType>text</dataType>
                <label translate="true">Created Date</label>
                <validation>
                    <rule name="required-entry" xsi:type="boolean">true</rule>
                </validation>
            </settings>
            <formElements>
                <date>
                    <settings>
                        <options>
                            <option name="dateFormat" xsi:type="string">MM/dd/yyyy</option>
                            <option name="timeFormat" xsi:type="string">HH:mm:ss</option>
                            <option name="showsTime" xsi:type="boolean">true</option>
                        </options>
                    </settings>
                </date>
            </formElements>
        </field>
    </fieldset>

    <!-- Items fieldset -->
    <fieldset name="items">
        <settings>
            <label translate="true">Order Items</label>
            <collapsible>true</collapsible>
            <opened>true</opened>
        </settings>

        <!-- Dynamic rows for order items -->
        <dynamicRows name="order_items">
            <settings>
                <addButtonLabel translate="true">Add Item</addButtonLabel>
                <deleteProperty>false</deleteProperty>
                <recordTemplate>record</recordTemplate>
            </settings>
            <container name="record" component="Magento_Ui/js/dynamic-rows/record">
                <argument name="data" xsi:type="array">
                    <item name="config" xsi:type="array">
                        <item name="isTemplate" xsi:type="boolean">true</item>
                        <item name="is_collection" xsi:type="boolean">true</item>
                        <item name="componentType" xsi:type="string">container</item>
                    </item>
                </argument>

                <field name="sku" formElement="input">
                    <settings>
                        <dataType>text</dataType>
                        <label translate="true">SKU</label>
                        <validation>
                            <rule name="required-entry" xsi:type="boolean">true</rule>
                        </validation>
                    </settings>
                </field>

                <field name="qty" formElement="input">
                    <settings>
                        <dataType>text</dataType>
                        <label translate="true">Qty</label>
                        <validation>
                            <rule name="required-entry" xsi:type="boolean">true</rule>
                            <rule name="validate-number" xsi:type="boolean">true</rule>
                        </validation>
                    </settings>
                </field>

                <field name="price" formElement="input">
                    <settings>
                        <dataType>text</dataType>
                        <label translate="true">Price</label>
                        <validation>
                            <rule name="validate-number" xsi:type="boolean">true</rule>
                        </validation>
                    </settings>
                </field>

                <actionDelete>
                    <settings>
                        <componentType>actionDelete</componentType>
                        <fit>false</fit>
                    </settings>
                </actionDelete>
            </container>
        </dynamicRows>
    </fieldset>
</form>
```

### Step 2: Form Data Provider

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Ui\Component\Form;

use Magento\Framework\App\Request\DataPersistorInterface;
use Magento\Ui\DataProvider\AbstractDataProvider;
use Vendor\Module\Model\ResourceModel\Order\CollectionFactory;
use Vendor\Module\Model\Order;

class DataProvider extends AbstractDataProvider
{
    private DataPersistorInterface $dataPersistor;
    private array $loadedData = [];

    public function __construct(
        string $name,
        string $primaryFieldName,
        string $requestFieldName,
        CollectionFactory $collectionFactory,
        DataPersistorInterface $dataPersistor,
        array $meta = [],
        array $data = []
    ) {
        parent::__construct($name, $primaryFieldName, $requestFieldName, $meta, $data);
        $this->collection = $collectionFactory->create();
        $this->dataPersistor = $dataPersistor;
    }

    /**
     * Get data
     *
     * @return array
     */
    public function getData(): array
    {
        if (!empty($this->loadedData)) {
            return $this->loadedData;
        }

        $items = $this->collection->getItems();
        /** @var Order $order */
        foreach ($items as $order) {
            $this->loadedData[$order->getId()] = $order->getData();

            // Load order items
            $orderItems = $order->getItems();
            $itemsData = [];
            foreach ($orderItems as $item) {
                $itemsData[] = [
                    'sku' => $item->getSku(),
                    'qty' => $item->getQty(),
                    'price' => $item->getPrice()
                ];
            }
            $this->loadedData[$order->getId()]['order_items'] = $itemsData;
        }

        // Load data from session (after validation failure)
        $data = $this->dataPersistor->get('vendor_module_order');
        if (!empty($data)) {
            $order = $this->collection->getNewEmptyItem();
            $order->setData($data);
            $this->loadedData[$order->getId()] = $order->getData();
            $this->dataPersistor->clear('vendor_module_order');
        }

        return $this->loadedData;
    }
}
```

### Step 3: Form Buttons

**Save button:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Block\Adminhtml\Order\Edit;

use Magento\Framework\View\Element\UiComponent\Control\ButtonProviderInterface;

class SaveButton implements ButtonProviderInterface
{
    /**
     * Get button data
     *
     * @return array
     */
    public function getButtonData(): array
    {
        return [
            'label' => __('Save'),
            'class' => 'save primary',
            'data_attribute' => [
                'mage-init' => ['button' => ['event' => 'save']],
                'form-role' => 'save',
            ],
            'sort_order' => 90,
        ];
    }
}
```

**Delete button:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Block\Adminhtml\Order\Edit;

use Magento\Framework\View\Element\UiComponent\Control\ButtonProviderInterface;
use Magento\Backend\Block\Widget\Context;

class DeleteButton implements ButtonProviderInterface
{
    public function __construct(
        private readonly Context $context
    ) {}

    /**
     * Get button data
     *
     * @return array
     */
    public function getButtonData(): array
    {
        $orderId = (int) $this->context->getRequest()->getParam('id');

        if (!$orderId) {
            return [];
        }

        return [
            'label' => __('Delete'),
            'class' => 'delete',
            'on_click' => sprintf(
                "deleteConfirm('%s', '%s')",
                __('Are you sure you want to delete this order?'),
                $this->context->getUrlBuilder()->getUrl('*/*/delete', ['id' => $orderId])
            ),
            'sort_order' => 20,
        ];
    }
}
```

---

## Mass Actions Implementation

### Controller

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Adminhtml\Order;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\Controller\ResultFactory;
use Magento\Ui\Component\MassAction\Filter;
use Vendor\Module\Model\ResourceModel\Order\CollectionFactory;
use Vendor\Module\Api\OrderRepositoryInterface;
use Psr\Log\LoggerInterface;

class MassDelete extends Action
{
    public const ADMIN_RESOURCE = 'Vendor_Module::order_delete';

    public function __construct(
        Context $context,
        private readonly Filter $filter,
        private readonly CollectionFactory $collectionFactory,
        private readonly OrderRepositoryInterface $orderRepository,
        private readonly LoggerInterface $logger
    ) {
        parent::__construct($context);
    }

    /**
     * Execute mass delete action
     *
     * @return \Magento\Framework\Controller\ResultInterface
     */
    public function execute()
    {
        try {
            $collection = $this->filter->getCollection($this->collectionFactory->create());
            $collectionSize = $collection->getSize();
            $deletedCount = 0;

            foreach ($collection as $order) {
                try {
                    $this->orderRepository->delete($order);
                    $deletedCount++;
                } catch (\Exception $e) {
                    $this->logger->error('Failed to delete order', [
                        'order_id' => $order->getId(),
                        'error' => $e->getMessage()
                    ]);
                }
            }

            $this->messageManager->addSuccessMessage(
                __('A total of %1 record(s) have been deleted.', $deletedCount)
            );

            if ($deletedCount < $collectionSize) {
                $this->messageManager->addWarningMessage(
                    __('%1 record(s) could not be deleted.', $collectionSize - $deletedCount)
                );
            }
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage(__('An error occurred while deleting orders.'));
            $this->logger->critical($e);
        }

        $resultRedirect = $this->resultFactory->create(ResultFactory::TYPE_REDIRECT);
        return $resultRedirect->setPath('*/*/');
    }
}
```

**Mass status update:**

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Adminhtml\Order;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\Controller\ResultFactory;
use Magento\Ui\Component\MassAction\Filter;
use Vendor\Module\Model\ResourceModel\Order\CollectionFactory;

class MassStatus extends Action
{
    public const ADMIN_RESOURCE = 'Vendor_Module::order_edit';

    public function __construct(
        Context $context,
        private readonly Filter $filter,
        private readonly CollectionFactory $collectionFactory
    ) {
        parent::__construct($context);
    }

    /**
     * Execute mass status update
     *
     * @return \Magento\Framework\Controller\ResultInterface
     */
    public function execute()
    {
        $status = $this->getRequest()->getParam('status');

        if (!$status) {
            $this->messageManager->addErrorMessage(__('Invalid status parameter.'));
            return $this->resultFactory->create(ResultFactory::TYPE_REDIRECT)->setPath('*/*/');
        }

        try {
            $collection = $this->filter->getCollection($this->collectionFactory->create());
            $updatedCount = 0;

            foreach ($collection as $order) {
                $order->setStatus($status);
                $order->save();
                $updatedCount++;
            }

            $this->messageManager->addSuccessMessage(
                __('A total of %1 record(s) have been updated.', $updatedCount)
            );
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage(__('An error occurred while updating orders.'));
        }

        return $this->resultFactory->create(ResultFactory::TYPE_REDIRECT)->setPath('*/*/');
    }
}
```

---

## Inline Editing

### Enable Inline Editing in Grid

```xml
<column name="status" component="Magento_Ui/js/grid/columns/select">
    <settings>
        <editor>
            <editorType>select</editorType>
            <validation>
                <rule name="required-entry" xsi:type="boolean">true</rule>
            </validation>
        </editor>
        <options class="Vendor\Module\Model\Source\OrderStatus"/>
        <filter>select</filter>
        <dataType>select</dataType>
        <label translate="true">Status</label>
    </settings>
</column>
```

### Inline Edit Controller

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Adminhtml\Order;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\Controller\Result\JsonFactory;
use Vendor\Module\Api\OrderRepositoryInterface;
use Psr\Log\LoggerInterface;

class InlineEdit extends Action
{
    public const ADMIN_RESOURCE = 'Vendor_Module::order_edit';

    public function __construct(
        Context $context,
        private readonly JsonFactory $jsonFactory,
        private readonly OrderRepositoryInterface $orderRepository,
        private readonly LoggerInterface $logger
    ) {
        parent::__construct($context);
    }

    /**
     * Execute inline edit
     *
     * @return \Magento\Framework\Controller\ResultInterface
     */
    public function execute()
    {
        $resultJson = $this->jsonFactory->create();
        $error = false;
        $messages = [];

        $postItems = $this->getRequest()->getParam('items', []);

        if (!($this->getRequest()->getParam('isAjax') && count($postItems))) {
            return $resultJson->setData([
                'messages' => [__('Please correct the data sent.')],
                'error' => true,
            ]);
        }

        foreach (array_keys($postItems) as $orderId) {
            try {
                $order = $this->orderRepository->get((int) $orderId);
                $order->setData(array_merge($order->getData(), $postItems[$orderId]));
                $this->orderRepository->save($order);
            } catch (\Exception $e) {
                $messages[] = __('Error updating order %1: %2', $orderId, $e->getMessage());
                $error = true;
                $this->logger->error('Inline edit error', [
                    'order_id' => $orderId,
                    'error' => $e->getMessage()
                ]);
            }
        }

        return $resultJson->setData([
            'messages' => $messages,
            'error' => $error
        ]);
    }
}
```

**Register route in `routes.xml`:**

```xml
<route id="vendor_module" frontName="vendor_module">
    <module name="Vendor_Module"/>
</route>
```

---

## AJAX Operations

### Custom AJAX Controller

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Adminhtml\Order;

use Magento\Backend\App\Action;
use Magento\Backend\App\Action\Context;
use Magento\Framework\Controller\Result\JsonFactory;
use Vendor\Module\Model\OrderProcessor;

class Process extends Action
{
    public const ADMIN_RESOURCE = 'Vendor_Module::order_process';

    public function __construct(
        Context $context,
        private readonly JsonFactory $jsonFactory,
        private readonly OrderProcessor $orderProcessor
    ) {
        parent::__construct($context);
    }

    /**
     * Process order via AJAX
     *
     * @return \Magento\Framework\Controller\ResultInterface
     */
    public function execute()
    {
        $resultJson = $this->jsonFactory->create();
        $orderId = (int) $this->getRequest()->getParam('order_id');

        if (!$orderId) {
            return $resultJson->setData([
                'success' => false,
                'message' => __('Invalid order ID')
            ]);
        }

        try {
            $this->orderProcessor->process($orderId);

            return $resultJson->setData([
                'success' => true,
                'message' => __('Order processed successfully')
            ]);
        } catch (\Exception $e) {
            return $resultJson->setData([
                'success' => false,
                'message' => __('Error processing order: %1', $e->getMessage())
            ]);
        }
    }
}
```

### JavaScript Component for AJAX

**File:** `view/adminhtml/web/js/process-order.js`

```javascript
define([
    'jquery',
    'Magento_Ui/js/modal/alert',
    'mage/translate'
], function ($, alert, $t) {
    'use strict';

    return function (config, element) {
        $(element).on('click', function (e) {
            e.preventDefault();

            $.ajax({
                url: config.processUrl,
                type: 'POST',
                dataType: 'json',
                data: {
                    order_id: config.orderId,
                    form_key: window.FORM_KEY
                },
                showLoader: true,
                success: function (response) {
                    if (response.success) {
                        alert({
                            title: $t('Success'),
                            content: response.message
                        });
                        location.reload();
                    } else {
                        alert({
                            title: $t('Error'),
                            content: response.message
                        });
                    }
                },
                error: function () {
                    alert({
                        title: $t('Error'),
                        content: $t('An error occurred while processing the order.')
                    });
                }
            });
        });
    };
});
```

**Add button in form:**

```xml
<field name="process_button" formElement="container">
    <settings>
        <label/>
        <additionalClasses>
            <class name="admin__field-small">false</class>
        </additionalClasses>
    </settings>
    <formElements>
        <container>
            <settings>
                <template>ui/form/components/button/container</template>
            </settings>
            <buttonAdapter>
                <settings>
                    <title translate="true">Process Order</title>
                    <actions>
                        <action name="process">
                            <settings>
                                <targetName>${ $.parentName }</targetName>
                                <actionName>processOrder</actionName>
                                <params>
                                    <param name="0" xsi:type="string">processUrl</param>
                                </params>
                            </settings>
                        </action>
                    </actions>
                </settings>
            </buttonAdapter>
        </container>
    </formElements>
</field>
```

---

## Assumptions

- **Magento version:** 2.4.7+ (UI Component framework 2.x)
- **PHP version:** 8.2+ (typed properties, readonly)
- **JavaScript:** KnockoutJS 3.5+, RequireJS 2.3+
- **Modules:** `Magento_Ui`, `Magento_Backend` enabled
- **ACL:** Proper ACL resources defined in `acl.xml`

## Why This Approach

**UI Component XML over programmatic grids:**
- Declarative configuration is easier to maintain and extend
- Built-in features (pagination, filtering, sorting) with zero code
- Consistent UX across Magento admin
- Plugin-friendly for third-party extensions

**Data providers pattern:**
- Separation of concerns (data fetching vs presentation)
- Testable (mock data provider in unit tests)
- Modifiers allow post-processing without touching collection
- Supports multiple data sources (DB, API, cache)

**Mass actions with Filter class:**
- Handles UI Component selected IDs automatically
- Supports "select all" across pages
- Consistent error handling and user feedback
- Transaction-safe (partial success reporting)

**Inline editing:**
- Reduces clicks for admin users (no form navigation)
- AJAX-based (no full page reload)
- Validation feedback inline
- Audit trail preserved (modified_at timestamps)

## Security Impact

- **ACL resources:** All controllers must define `ADMIN_RESOURCE` constant; unauthorized access returns 403
- **CSRF protection:** Form key validated on all POST requests (Magento framework handles automatically)
- **XSS prevention:** Use `translate="true"` and `__()` for all user-facing strings; never output raw HTML
- **SQL injection:** Use repositories/collections (ORM); never concatenate SQL strings
- **Authorization:** Verify entity ownership before edit/delete (check store scope, customer group)
- **AJAX endpoints:** Validate `isAjax()` request; return JSON only (no HTML to prevent CSRF)

## Performance Impact

- **Grid collection:** Use indexes on filtered/sorted columns; avoid N+1 queries with joins
- **Data provider:** Lazy load data (only fetch when needed); cache metadata
- **Static content:** UI Component JS/CSS minified in production mode
- **Mass actions:** Batch operations where possible (single UPDATE for status changes)
- **Inline editing:** Delta updates (only changed fields); avoid full entity load
- **Pagination:** Limit grid to 20-50 rows per page; use `getLimitedData()` for large datasets

## Backward Compatibility

- **UI Component schema:** Stable across 2.4.x; breaking changes in major versions only (2.5+)
- **Data provider interface:** Implement `DataProviderInterface` for forward compatibility
- **JavaScript components:** Use RequireJS mixins for extensions; avoid direct overwrites
- **Deprecations:** `Magento_Ui/js/lib/ko/bind/scope` deprecated in 2.4.6 (use `scope` binding)
- **Migration path:** UI Components replace legacy grids (`Magento\Backend\Block\Widget\Grid`); migrate before 2.5

## Tests to Add

**Unit tests:**
- Data provider returns correct data structure
- Column renderers format values correctly
- Mass action controllers handle errors gracefully

**Integration tests:**
- Grid collection filters apply correctly
- Form data provider loads entity data
- Inline edit saves changes to database
- AJAX endpoints return valid JSON

**MFTF tests:**
- Grid displays records and pagination works
- Mass actions execute and show success message
- Form validates required fields
- Inline editing updates grid row

## Documentation to Update

- **README:** List all admin routes and ACL resources
- **Admin user guide:** Screenshots of grid, form, mass actions
- **Developer guide:** How to extend grid with custom columns, add form fields
- **CHANGELOG:** Document UI Component schema changes, new form fields
- **Architecture diagram:** Data flow from controller → data provider → collection → UI Component

---

## Additional Resources

- [Magento DevDocs: UI Components](https://developer.adobe.com/commerce/frontend-core/ui-components/)
- [UI Component XML Reference](https://developer.adobe.com/commerce/frontend-core/ui-components/concepts/xml-declaration/)
- [Data Provider Guide](https://developer.adobe.com/commerce/php/development/components/data-provider/)
- [KnockoutJS Documentation](https://knockoutjs.com/documentation/introduction.html)

## Related Documentation

### Related Guides

- [Layout XML Deep Dive: Mastering Magento's View Layer](layout-xml-deep-dive.md)
- [GraphQL Resolver Patterns in Magento 2](../explanations/graphql-resolver-patterns.md)

### Related Module Documentation

- [Magento_Catalog Overview](../../modules/catalog/README.md)
- [Magento_Customer Overview](../../modules/customer/README.md)
- [Magento_Sales Overview](../../modules/sales/README.md)
