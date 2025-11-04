---
title: "How to Set Up Google Analytics 4"
description: "Configure GA4 with enhanced ecommerce tracking for complete customer insights"
type: how-to
audience:
  - "Magento merchants"
  - "Advanced users"
difficulty: Intermediate
estimated_time: "6 minutes"
prerequisites:
  - "Access to Magento Admin Panel"
  - "Basic PHP knowledge"
  - "Magento development environment"
tags:
  - how-to
  - customer-management
  - analytics
last_updated: "2025-09-16T18:03:17.602Z"
---

# How to Set Up Google Analytics 4

## Overview

Set up Google Analytics 4 (GA4) with enhanced ecommerce tracking to see the full customer journey—from product views to
purchases—so you can optimize conversion rates, reduce acquisition costs, and improve store performance. With purchase
events in GA4, you can: track revenue and average order value by channel, build remarketing audiences (for example, cart
abandoners or high-value buyers), import conversions into Google Ads for Smart Bidding, and diagnose drop-off in the
product-to-checkout funnel. Many merchants see improved ROAS within 2–6 weeks after importing reliable GA4 purchase
conversions into Google Ads.

Version compatibility and path selection:

- Magento 2.4.5+: Use Stores > Configuration > Sales > Google API > Google Tag to add your GA4 Measurement ID (
  recommended for most merchants). Do not also install GA4 via GTM unless intentionally architected to avoid duplicates.
- Magento < 2.4.5: Use Google Tag Manager (GTM) or a Marketplace extension to deploy GA4.
- Choose a single path to avoid duplicate tags. If you must use both Google Tag and GTM, ensure only one source sends
  page_view and purchase.

## Prerequisites

Before you begin, make sure you have:

- Admin-only path (no code):
    - A GA4 property with a Web data stream and Measurement ID (G-XXXXXXXXXX)
    - Access to Magento Admin (appropriate scope permissions)
    - A GTM Web container (only if you choose the GTM path)
- Code path (theme/module):
    - Magento development environment and deployment access (staging and production)
    - Ability to clear caches and deploy static content in production mode
    - Basic PHP/XML/JS familiarity (for theme or module edits)
- Testing:
    - Access to a staging store and a test payment method to place orders
    - Ability to use Tag Assistant (GTM Preview) and GA4 DebugView

## What You'll Accomplish

By following this guide, you will:

- Set up Google Analytics 4 with ecommerce tracking for your Magento store, including page views and purchase events.

## Step-by-Step Instructions

Before you start the steps, choose one implementation path to avoid duplicate data:

- Option A (recommended for Magento 2.4.5+): Use Magento’s native Google Tag (GA4) integration for a no-code install.
- Option B: Use Google Tag Manager if you manage multiple tags or need advanced control.
- Option C: Manual theme/module edits (developers) when Admin/GTM options are unavailable.

Follow only one option or coordinate carefully to prevent duplicate events.

### Step 0: Create a GA4 property and web data stream

In Google Analytics, create a new GA4 property. Add a Web data stream for your storefront domain and note the
Measurement ID (format: G-XXXXXXXXXX). Enable Enhanced Measurement.

### Step 1: Create a Google Tag Manager (GTM) Web container (recommended path)

If you plan to use GTM (Option B), in Google Tag Manager, create a new container for your domain (target type: Web).
Note your container ID (format: GTM-XXXXXXX).

Note: If you’re on Magento 2.4.5+ and do not need GTM, you can use Magento’s Google Tag integration in Step 2 instead.
Avoid installing both unless you understand and prevent duplicate events.

### Step 2: Install GTM on Magento via Admin (if available)

For Magento 2.4.5+ (Option A: Google Tag GA4, recommended):

- Go to Stores > Configuration.
- In the top-left Scope dropdown, select the Website/Store View you want to configure.
- Navigate to Sales > Google API > Google Tag.
- Enter your GA4 Measurement ID (G-XXXXXXXXXX), enable the integration, and Save.
- Go to System > Cache Management and Flush Magento Cache.
- Multi-store note: Configure per Website/Store View as needed. Uncheck “Use Default” to override values at lower
  scopes.

About GTM via Admin:

- The legacy Google Tag Manager module is not included by default in 2.4.4+ unless installed separately. If you don’t
  see Google Tag or GTM settings in Admin, proceed to Step 3 to add the container via your theme.

### Step 3: Install GTM on Magento via theme (alternative)

If your Admin lacks Google Tag or GTM settings, add the GTM snippets via theme/layout.

- Head script snippet:
    - Preferred: add the GTM <script> to the head via Admin > Content > Design > Configuration > Edit Scope > Other
      Settings > HTML Head > Scripts and Style Sheets.
    - Or in layout XML, reference the correct container:
        - Container: head.additional
        - File example (app/design/frontend/<Vendor>/<theme>/Magento_Theme/layout/default.xml or
          default_head_blocks.xml):

```xml

<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <head>
        <!-- Optionally include a block/template to output the GTM head script -->
    </head>
    <body>
        <!-- See after.body.start below for the noscript iframe -->
    </body>
</page>
```

- Body noscript snippet (immediately after <body>):
    - Add a block to the after.body.start container in app/design/frontend/<Vendor>/<theme>
      /Magento_Theme/layout/default.xml:

```xml

<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceContainer name="after.body.start">
            <block class="Magento\Framework\View\Element\Template"
                   name="gtm.noscript"
                   template="Magento_Theme::gtm/noscript.phtml"/>
        </referenceContainer>
    </body>
</page>
```

- Important: Do not place the noscript iframe in Footer > Miscellaneous HTML; it renders at the end of <body>, not
  immediately after it.

- After making changes in production mode, deploy and clear caches:

```bash
bin/magento cache:clean
bin/magento cache:flush
bin/magento setup:static-content:deploy -f
```

- Verify the container loads using Tag Assistant (GTM Preview).

### Step 4: Add a GA4 Configuration tag in GTM

In GTM, create a Tag: GA4 Configuration. Enter your Measurement ID (G-XXXXXXXXXX). Trigger: All Pages. Optionally enable
“Send a page view event when this configuration loads” if you are not managing page views elsewhere.

Duplicate page_view warning: If GA4 is also installed via Magento Google Tag (Step 2) or you already have another GA4
Configuration tag, disable “Send a page view event when this configuration loads” in GTM or remove the duplicate
install. Ensure a single source sends page_view to avoid double counting.

Consent Mode (EEA/UK): If you serve EEA/UK users, implement Consent Mode v2 so GA4 respects user consent. Default to
denied until consent is granted, then update on acceptance. Coordinate with your CMP/cookie banner.

```html

<script>
    // Default consent
    gtag('consent', 'default', {
        ad_user_data: 'denied',
        ad_personalization: 'denied',
        ad_storage: 'denied',
        analytics_storage: 'denied'
    });
    // On user consent, update to granted
    // gtag('consent', 'update', { ad_user_data: 'granted', ad_personalization: 'granted', ad_storage: 'granted', analytics_storage: 'granted' });
</script>
```

### Step 5: Confirm or add a GA4-compatible ecommerce data layer

Check if your site pushes GA4-ready events (for example, view_item, add_to_cart, begin_checkout, purchase) with these
minimum parameters:

- items: array of item objects
- value: number (no currency symbols)
- currency: ISO 4217 code (for example, "USD")

Use GTM Preview to inspect the dataLayer. If you do not see GA4-style events, plan to:

- Install a GA4-ready data layer extension from Marketplace, or
- Use the low-code purchase fallback in Step 7 and phase in more events over time.

### Step 6: Create GA4 ecommerce event tags in GTM (if data layer is present)

For each event you detect in the dataLayer, create a GA4 Event tag and map parameters using GTM Data Layer Variables.

- view_item: trigger on product detail view; map items (array), value, currency.
- add_to_cart: trigger on add-to-cart; map items, value, currency, quantity.
- begin_checkout: trigger at checkout start; map items, value, currency, coupon (if present).
- purchase: trigger on order success; map transaction_id (unique), affiliation (for example, 'online'), value (number),
  tax, shipping, currency, coupon, and items array.

Create GTM Variables (Data Layer Variable type) for each field you need to map.

### Step 7: Low-code fallback: add a purchase event on the success page (if no data layer)

Implement at least the purchase event to unlock revenue reporting:

- In your theme or a custom module, add JS on the checkout success page (handle: checkout_onepage_success).
- Output a dataLayer push with GA4 fields using correct types and names:
    - value, tax, shipping, price, quantity must be numbers
    - currency must be a 3-letter ISO code (for example, "USD")
    - transaction_id must be unique per order (use the Magento increment_id)
    - GA4 item parameters: item_id (SKU), item_name (product name), item_brand (if available), item_category, price,
      quantity

Example purchase event (pseudocode using Magento variables):

```html

<script>
    window.dataLayer = window.dataLayer || [];
    window.dataLayer.push({
        event: 'purchase',
        ecommerce: {
            transaction_id: '{{ order.increment_id }}',
            affiliation: 'online',
            value: {
    {
        order.grand_total_number
    }
    },
    tax: {
        {
            order.tax_amount_number
        }
    }
    ,
    shipping: {
        {
            order.shipping_amount_number
        }
    }
    ,
    currency: '{{ order.currency_code }}',
            coupon
    :
    '{{ order.coupon_code }}',
            items
    :
    [
        {{#each order.items}
    }
    {
        item_id: '{{ this.sku }}',
                item_name
    :
        '{{ this.name }}',
                item_brand
    :
        '{{ this.brand }}',
                item_category
    :
        '{{ this.category }}',
                price
    :
        {
            {
                this.price_number
            }
        }
    ,
        quantity: {
            {
                this.qty_number
            }
        }
    }
    {
        {
            #unless
        @last
        }
    }
    ,
    {
        {
            /unless}}
            {
                {
                    /each}}
                ]
                }
            }
        )
            ;
</script>
```

Then in GTM, create a GA4 Event tag 'purchase' triggered when event equals purchase, mapping the parameters accordingly.
Clear caches and test.

### Step 8: Prevent duplicate purchase events

Ensure the purchase code runs only on the initial load of the success page and not on refresh or back navigation. Use a
server-rendered flag or a client-side sessionStorage/localStorage key to guard against duplicates. Verify only one GTM
or GA4 configuration is installed to avoid double firing.

### Step 9: Preview, publish, and test

Use GTM Preview (Tag Assistant) and navigate a full flow: Home > Product page (view_item) > Add to Cart (add_to_cart) >
Checkout start (begin_checkout) > Order success (purchase).

- Confirm GA4 tags fire with correct parameters at each step.
- Validate that only one page_view and one purchase fire per order.
- Publish the container when satisfied.

### Step 10: Validate in GA4

Open GA4 DebugView to confirm events and parameters in real time. Check Real-time for active users and event names.
After 24–48 hours, review Monetization > E-commerce purchases and funnel reports for revenue, items, and conversion
steps. Confirm transaction_id uniqueness and expected revenue totals.

### Step 11: Document and operationalize

Record your Measurement ID, container ID, and the events implemented. Note any exclusions (for example, missing
view_item_list). Define ownership: assign a Marketing Operations owner for GTM/GA4. Create a monthly QA checklist (for
example, verify purchase volume, revenue, item counts) and alerts for sudden drops in purchase events. Schedule periodic
QA after theme/extension updates and plan the next events if you used the low-code fallback.

After events fire reliably, link GA4 to Google Ads (GA4 > Admin > Product Links > Google Ads) and import conversions (
purchase) into Google Ads for optimized bidding. Ensure consistent currency and conversion window settings.

## Verification

Use this checklist to confirm everything is working correctly:

1. Tag Assistant (GTM Preview):
    - Start a preview session and navigate: Home > PDP > Add to Cart > Checkout > Success.
    - Pass: GA4 tags fire on expected events (view_item, add_to_cart, begin_checkout, purchase) with items, value (
      number), currency (ISO), and transaction_id.
    - Pass: Only one page_view per page and a single purchase per order.
2. GA4 DebugView (GA4 > Admin > DebugView):
    - Pass: Events appear in order with correct parameters.
3. Reports (after 24–48 hours):
    - Pass: Monetization > E-commerce purchases shows orders and item data; revenue aligns with your store (within
      expected variance).
4. Multi-store scope:
    - Pass: Correct Measurement ID/GTM container per Website/Store View; “Use Default” overridden where required.
5. Duplicate detection:
    - Pass: Refreshing the success page does not create a second purchase event.

## Common Issues and Solutions

- Duplicate page views or purchases:
    - Ensure only one GA4 configuration is installed. In GTM, disable the auto page_view if Magento Google Tag also
      sends page_view.
    - Guard success page code with a one-time flag (for example, sessionStorage key).
- No events in DebugView:
    - Check that the GTM container is published and not blocked by consent.
    - Verify dataLayer event names and that GTM triggers match them.
- Wrong revenue or currency:
    - Ensure value is numeric (no strings) and currency is a 3-letter ISO code. Align value with your reporting
      preference (incl./excl. tax).
- Missing items:
    - Confirm the items array exists and each object includes item_id and item_name at minimum.
- Multi-store mix-ups:
    - Verify the correct store view scope is selected in Admin and that you’re using the correct Measurement
      ID/container per store.
