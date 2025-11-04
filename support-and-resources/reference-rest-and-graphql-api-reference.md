---
title: "REST and GraphQL API Reference"
description: "Merchant-focused overview of REST and GraphQL usage with setup steps, examples, and links to official references."
type: reference
audience:
  - Magento merchants
  - Magento developers
tags:
  - reference
  - api
last_updated: "2025-09-15T07:56:18.388Z"
---

# REST and GraphQL API Reference

## Overview

Use Magento 2 REST and GraphQL APIs to automate catalog, order, and customer workflows. This guide shows how to
authenticate, choose REST vs GraphQL, and run common operations (price/inventory updates, order exports) with links to
the full API references.

For growing merchants, these APIs reduce manual work, improve data accuracy, and speed up ERP, PIM, and fulfillment
integrations. Automating daily price and inventory updates can cut manual effort by 60–80% and reduce out-of-stock
cancellations, protecting revenue during promotions.

Business impact examples

- Nightly price sync from PIM reduces catalog admin time by ~8–12 hours per week.
- Near real-time inventory push from WMS reduces oversells by 20–40% during promotions.
- Automated shipment creation with tracking reduces order-to-ship SLA by 15–25%.

Version and edition support

- This guidance applies to Magento Open Source and Adobe Commerce 2.4.x unless otherwise noted.
- Some features (for example, B2B GraphQL and selected service contracts) are available only in Adobe Commerce. Always
  verify against your installed version.

Quick start (2-minute test)

1) Get an admin or integration token (REST) and run a small GET products call to verify access.
2) Run a basic GraphQL products query using the Store header to scope to a store view.
3) Confirm you can see the same product in Admin to validate scope alignment.

## Prerequisites

- Admin access to your Magento Admin and the ability to create Integrations (System > Extensions > Integrations).
- A dedicated Integration user role with least-privilege Resource Access.
- Know your store codes (Stores > All Stores) for REST paths; use the Store header for GraphQL.
- For inventory updates, ensure Inventory Management (MSI) modules are enabled (default in 2.4.x).
- Test in a staging environment before production.

## Configuration

Follow these steps to authenticate and find the correct endpoints for your store.

Authentication

- Admin token (REST) — for back-office operations
    - Endpoint: POST https://yourdomain/rest/V1/integration/admin/token
    - Body:
      ```json
      {"username":"<admin_user>","password":"<admin_password>"}
      ```
    - Use the returned token in requests: Authorization: Bearer <token>

- Customer token (REST/GraphQL) — for storefront customer actions (cart, checkout, account)
    - Endpoint: POST https://yourdomain/rest/V1/integration/customer/token
    - Body:
      ```json
      {"username":"customer@example.com","password":"<password>"}
      ```
    - Note: You can also obtain a customer token via GraphQL using the generateCustomerToken mutation.

- Integration access token (UI) — for system-to-system automation via REST
    - In Admin, go to System > Extensions > Integrations > Add New Integration.
    - Configure Name, Callback URL (if needed), and Resource Access (least privilege recommended).
    - Save & Activate, then copy the Access Token. Use Authorization: Bearer <token> for REST.
    - Important: For GraphQL, authenticate as a customer using a customer token (Bearer <customer_token>) or use guest
      access where allowed. Do not use admin or integration tokens with GraphQL.

Which token to use

- Admin/Integration token (REST): Back-office operations (catalog create/update, order export, shipments, invoices).
- Customer token (GraphQL/REST): Storefront customer operations (cart, checkout, account).
- Guest (GraphQL): Public catalog/search queries where allowed.

Base URLs

- REST base path: https://yourdomain/rest/<store_code>/V1/
    - Example: https://yourdomain/rest/default/V1/products
    - REST store scope: You can call /rest/V1/... (no store code) for default scope, or /rest/<store_code>/V1/... to
      target a specific store view. Use the store that matches the catalog/pricing context you intend to read/write.
- GraphQL endpoint: POST https://yourdomain/graphql
    - To target a specific store view, set header Store: <store_code>. Optionally set Content-Currency: <currency_code>
      for price rendering.

Choosing REST vs GraphQL

- Choose GraphQL for storefront reads and cart/checkout flows where you need only specific fields and can leverage CDN
  caching on cacheable queries.
- Choose REST (or Async/Bulk REST) for back-office writes and exports (catalog updates, order/fulfillment,
  price/inventory).
- For 100+ items or time-sensitive jobs, use Async/Bulk REST to avoid timeouts and improve throughput.

Decision checklist

- Need CDN-cached storefront pages and minimal payloads? Use GraphQL.
- Performing bulk catalog or inventory updates under a deadline? Use Async/Bulk REST.
- Integrating with ERP/3PL that expects RESTful endpoints and pagination? Use REST with filters and paging.

## Common merchant workflows (examples)

Each example includes prerequisites, required headers, a purpose, and a cURL you can run.

- Get products (REST)
    - Purpose: Page through products for export to PIM or audit
    - Prerequisites: Admin or Integration token (REST)
    - Endpoint: GET /rest/default/V1/products?searchCriteria[currentPage]=1&searchCriteria[pageSize]=10
    - Headers: Content-Type: application/json, Authorization: Bearer <admin_or_integration_token>
    - cURL:
      ```bash
      curl -X GET "https://yourdomain/rest/default/V1/products?searchCriteria[currentPage]=1&searchCriteria[pageSize]=10" \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer <admin_or_integration_token>"
      ```
    - Verify: Check Admin > Catalog > Products for matching SKUs; or call GET /V1/products/<sku>
    - Business process: Product export to PIM or reporting tools

- Update inventory (MSI, Bulk REST)
    - Purpose: Update on-hand qty for default source from WMS feed
    - Prerequisites: Admin or Integration token (REST); MSI enabled
    - Endpoint: POST /rest/default/async/bulk/V1/inventory/source-items
    - Headers: Content-Type: application/json, Authorization: Bearer <admin_or_integration_token>
    - Body:
      ```json
      [
        {"sku":"SKU1","source_code":"default","quantity":10,"status":1},
        {"sku":"SKU2","source_code":"default","quantity":5,"status":1}
      ]
      ```
    - cURL:
      ```bash
      curl -X POST "https://yourdomain/rest/default/async/bulk/V1/inventory/source-items" \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer <admin_or_integration_token>" \
        -d '[{"sku":"SKU1","source_code":"default","quantity":10,"status":1},{"sku":"SKU2","source_code":"default","quantity":5,"status":1}]'
      ```
    - Response: HTTP 202 with a bulk_uuid to track status
    - Verify: Check Admin > Catalog > Products > [SKU] > Sources; or GET
      /V1/inventory/source-items?searchCriteria[filter_groups][0][filters][0][field]
      =sku&searchCriteria[filter_groups][0][filters][0][value]
      =SKU1&searchCriteria[filter_groups][0][filters][0][condition_type]=eq
    - Business process: Inventory sync from WMS or 3PL
    - Note: When Inventory Management (MSI) is enabled (default in 2.4.x), do not use legacy endpoints like
      /V1/products/:sku/stockItems. Use the Inventory API (/V1/inventory/source-items) to update quantities and status.

- Create shipment (REST)
    - Purpose: Mark an order as shipped and add tracking from ERP/fulfillment
    - Prerequisites: Admin or Integration token (REST); order must be invoiced or invoice upon shipment according to
      your flow
    - Endpoint: POST /rest/default/V1/order/{orderId}/ship
    - Headers: Content-Type: application/json, Authorization: Bearer <admin_or_integration_token>
    - Body (example):
      ```json
      {
        "items": [
          {"order_item_id": 12345, "qty": 1}
        ],
        "notify": true,
        "appendComment": true,
        "comment": {"comment": "Shipped", "is_visible_on_front": 0},
        "tracks": [
          {"track_number": "1Z999...", "title": "UPS", "carrier_code": "ups"}
        ]
      }
      ```
    - cURL:
      ```bash
      curl -X POST "https://yourdomain/rest/default/V1/order/100000123/ship" \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer <admin_or_integration_token>" \
        -d '{"items":[{"order_item_id":12345,"qty":1}],"notify":true,"appendComment":true,"comment":{"comment":"Shipped","is_visible_on_front":0},"tracks":[{"track_number":"1Z999...","title":"UPS","carrier_code":"ups"}]}'
      ```
    - Note: Find order_item_id via GET /V1/orders/{orderId}.
    - Verify: Admin > Sales > Shipments shows the new shipment; GET
      /V1/shipment/shipments?searchCriteria[filter_groups][0][filters][0][field]
      =order_id&searchCriteria[filter_groups][0][filters][0][value]={orderId}
    - Business process: Shipment creation from ERP or 3PL webhook

- GraphQL product query
    - Purpose: Storefront product search with minimal fields for performance and caching
    - Prerequisites: Guest access (for public catalog) or Customer token if required by your store config
    - Endpoint: POST /graphql
    - Headers: Content-Type: application/json, Store: <store_code> (and Authorization: Bearer <customer_token> if
      needed)
    - Body:
      ```json
      {
        "query": "{ products(search: \"shirt\") { items { sku name price_range { minimum_price { regular_price { value currency } } } } } }"
      }
      ```
    - cURL (guest):
      ```bash
      curl -X POST "https://yourdomain/graphql" \
        -H "Content-Type: application/json" \
        -H "Store: default" \
        -d '{"query":"{ products(search: \"shirt\"){ items { sku name price_range { minimum_price { regular_price { value currency } } } } } }"}'
      ```
    - Verify: Compare pricing/currency with storefront; adjust Content-Currency header if needed
    - Business process: Storefront catalog/search rendering or PWA data fetch

## Bulk and Async APIs

- Use async/bulk endpoints for high-volume updates to reduce request time and server load.
- Definitions:
    - Async endpoints: Queue a single request and return HTTP 202 (Accepted). Use when a single large payload must be
      queued.
    - Bulk endpoints: Accept an array of operations to process asynchronously. Use when sending many discrete
      operations.
- Example: POST https://yourdomain/rest/default/async/bulk/V1/products to create or update multiple products in one
  request.
- Responses and tracking:
    - Async/bulk responses include bulk_uuid.
    - Track progress with:
        - GET /rest/<store_code>/V1/bulk/{bulk_uuid}
        - GET /rest/<store_code>/V1/bulk/{bulk_uuid}/status
    - Example async response:
      ```json
      {"bulk_uuid":"e6d1c8c7-5a2b-4f9a-9b30-9f2a1d8b8e01","request_items":[{"id":1,"status":202}],"errors":false}
      ```
- After completion, verify updates with follow-up GET calls and/or Admin grids.

## Error handling and verification

- Common HTTP status codes: 200/201 success, 202 accepted (async/bulk queued), 400/422 validation error, 401
  unauthorized (invalid token), 404 not found, 500 server error.
- REST error responses typically include a message and parameters. Log the full response for troubleshooting.
- Example REST error:
  ```json
  {"message":"Validation Failed","errors":[{"message":"Field \"sku\" is required"}],"parameters":{"fieldName":"sku"}}
  ```
  Action: Log message and parameters; correct payload; retry.
- Verify updates by:
    - Checking the Admin grid, and/or
    - Calling GET endpoints (for example, GET /V1/products/:sku) or equivalent GraphQL queries to confirm changes.

## Security best practices

- Use Integrations with least-privilege Resource Access for third-party systems.
- Create a dedicated API role that grants only required resources (for example, Catalog > Inventory for stock updates)
  and assign it to the Integration.
- Rotate tokens regularly and revoke unused integrations (System > Extensions > Integrations).
- Use HTTPS only; never expose tokens in client-side code.
- Restrict API access by IP where possible (at CDN, firewall, or web server).

## Performance considerations

- Batch writes using Async/Bulk where available. Batching reduces peak load risk during promos and protects page speed.
- Keep payloads minimal; send only changed fields.
- Schedule large updates during off-peak hours to avoid contention with customer traffic.
- Ensure indexers and caches are healthy to avoid slow write/read paths.
- GraphQL: Use persisted queries and keep selections minimal to improve cache hit rates at CDN. Reuse fragments and
  avoid over-fetching.
- REST: Always use filters and pagination; avoid unfiltered GETs on large catalogs.
