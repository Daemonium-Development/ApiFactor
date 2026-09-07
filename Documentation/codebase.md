# CargoHUB WMS API — Codebase Documentation

Reference documentation for the API in `Codebase/code`. It describes what the API does, how it is built, and every path it exposes.

---

## 1. Overview

The codebase is a Warehouse Management System (WMS) backend called **CargoHUB**. It exposes a versioned HTTP/JSON API over a warehousing domain: warehouses and their storage locations, item master data (items, lines, groups, types), suppliers, inventory per location, clients, orders, shipments and internal stock transfers.

Key characteristics:

- Written in Python 3 using only the standard library (`http.server`, `socketserver`, `json`, `threading`, `datetime`). No framework, no external dependencies.
- Persistence is a set of flat JSON files in `Codebase/code/data/` — there is no database.
- Authentication is a static API key passed in a request header; authorisation is a per-application permission matrix stored alongside the data.
- All routes live under the prefix `/api/v1/`.
- The server listens on port **3000**.

### Running

```sh
Codebase/code/start-system.sh
```

The script `cd`s into `Codebase/code/api` (so the `providers`, `models` and `processors` packages resolve) and runs `python3 main.py`. On start it prints `Serving on port 3000...` and starts the notification processor.

---

## 2. Project layout

```
Codebase/code/
├── start-system.sh                     launcher
├── api/
│   ├── main.py                         HTTP server + all routing/handlers
│   ├── providers/
│   │   ├── auth_provider.py            API-key lookup and permission check
│   │   └── data_provider.py            factory functions returning model pools
│   ├── processors/
│   │   └── notification_processor.py   in-process notification queue
│   └── models/
│       ├── base.py                     shared timestamp helper
│       ├── warehouses.py  locations.py
│       ├── items.py  item_lines.py  item_groups.py  item_types.py
│       ├── inventories.py  suppliers.py
│       ├── orders.py  order_items.py
│       ├── shipments.py  shipment_items.py
│       └── transfers.py  transfer_items.py
└── data/                               JSON files, one per table
    ├── user.json          warehouse.json     location.json
    ├── item.json          item_line.json     item_group.json   item_type.json
    ├── inventory.json     supplier.json      client.json
    ├── order.json         order_item.json
    ├── shipment.json      shipment_item.json
    └── transfer.json      transfer_item.json
```

---

## 3. Architecture and request flow

### 3.1 Layers

| Layer | Responsibility |
| --- | --- |
| `main.py` (`ApiRequestHandler`) | Parses the URL, authenticates, authorises, dispatches to a model pool, writes the HTTP response |
| `providers/auth_provider.py` | Resolves an API key to a user record and answers "may this user perform this method on this resource" |
| `providers/data_provider.py` | Factory functions (`fetch_*_pool()`) that construct a model instance bound to the data directory |
| `models/*` | One class per table; loads its JSON file into memory, offers query/mutate methods, writes back on `save()` |
| `processors/notification_processor.py` | Background queue that prints queued messages |

### 3.2 Request lifecycle

Each of `do_GET`, `do_POST`, `do_PUT`, `do_DELETE` follows the same shape:

1. Read the `API_KEY` request header.
2. `auth_provider.get_user(api_key)` scans `data/user.json` for a record whose `api_key` matches. `None` → **401** and the request ends.
3. Split `self.path` on `/`, giving `["", "api", "v1", <resource>, ...]`. The handler proceeds only when the split yields more than 3 segments and segments 1 and 2 are exactly `api` and `v1`. Anything else produces no response at all.
4. The remainder (`paths[3:]`, i.e. resource segment onwards) is passed to the version-1 handler for that verb.
5. The handler calls `auth_provider.has_access(user, paths, method)`. `False` → **403**.
6. The handler branches on `paths[0]` (the resource) and, for GET/PUT, on the number of remaining segments, then delegates to a model pool.
7. Any exception raised anywhere in steps 3–6 is caught and answered with **500**.

### 3.3 Routing mechanics

Routing is hand-written `if/elif` chains on `paths[0]`, with a `match len(paths)` inside for resources that have sub-paths. Consequences that are visible from the outside:

- Path segments are matched literally; there is no query-string parsing. A query string stays attached to the final path segment.
- Numeric ids are converted with `int(...)`. A non-numeric id raises and is answered **500**.
- A trailing slash adds an empty final segment, which changes the segment count and, where an id is expected, fails the `int()` conversion → **500**.
- Unknown resource names, unknown sub-paths and unsupported segment counts return **404**.
- Only GET, POST, PUT and DELETE are implemented. Other verbs fall through to `BaseHTTPRequestHandler`'s default handling (**501**).

### 3.4 Persistence model

`data_provider.fetch_*_pool()` constructs a **new** model instance on every call, and each constructor reads its JSON file from disk immediately. There is no caching or connection pooling: a single request may read the same file several times, and reads are always fresh from disk.

Mutating handlers follow the pattern *load pool → mutate in memory → `save()`*, where `save()` rewrites the whole JSON file with `json.dump`. There is no locking, no transaction and no rollback; the server is a single-threaded `TCPServer`, so requests are processed one at a time.

Every model derives from `Base`, which supplies `get_timestamp()` returning a UTC ISO-8601 string ending in `Z`. `add_*` sets both `created_at` and `updated_at`; `update_*` sets `updated_at`. `update_*` replaces the stored record wholesale with the submitted body — it is not a merge.

Creation does **not** generate ids for the main entities: the `id` must be supplied in the POST body. Ids for the junction tables (`order_item`, `shipment_item`, `transfer_item`) are generated internally as `max(existing id) + 1`.

### 3.5 Composite records

Orders, shipments and transfers are stored as a header row plus rows in a junction table. The models hide this:

- `get_orders()`, `get_order()`, `get_shipments()`, `get_shipment()`, `get_transfers()`, `get_transfer()` each return the header record with an added `items` array of `{item_id, amount}` objects.
- `add_*` pops `items` off the submitted body, stores the header, then writes the junction rows.
- `update_*` pops `items`; when present it replaces the junction rows, when absent the existing item rows are left untouched.
- `remove_*` deletes the header and all its junction rows.

Because the enrichment is done per header and each enrichment constructs a fresh junction pool, listing endpoints re-read the junction file once per record in the list.

### 3.6 Inventory side effects

Inventory is keyed on the composite `(item_id, location_id)`; there is no surrogate inventory id. Three operations write to inventory as a side effect:

| Trigger | Effect |
| --- | --- |
| `PUT /api/v1/orders/{id}/items` (and `PUT /api/v1/orders/{id}` with an `items` field) | For each item, the delta between the new and old ordered amount is applied to `quantity_allocated`. The delta lands on the inventory row for that item with the highest `quantity_on_hand`; the result is clamped at 0. Items removed from the order get their previous amount subtracted. |
| `PUT /api/v1/shipments/{id}/items` (and `PUT /api/v1/shipments/{id}` with an `items` field) | Same mechanism, applied to `quantity_ordered`. |
| `PUT /api/v1/transfers/{id}/commit` | For each item in the transfer: subtract `amount` from `quantity_on_hand` at `from_location_id` (only when a row exists), and add it at `to_location_id`, creating a row with zeroed `quantity_expected`/`quantity_ordered`/`quantity_allocated` when none exists. The transfer's `transfer_status` becomes `Processed`. |

### 3.7 Notification processor

`notification_processor` holds a module-level list, seeded with `"Dummy message"`. `push(message)` appends to it. `start()` — called once at server start — begins a self-rescheduling `threading.Timer` chain that fires every **30 seconds**, popping at most one message per tick and printing it to stdout. Nothing is delivered off-machine; the module notes push/e-mail delivery as future work.

Two events push a notification:

- `POST /api/v1/transfers` → `Scheduled batch transfer {id}`
- `PUT /api/v1/transfers/{id}/commit` → `Processed batch transfer with id:{id}`

---

## 4. Authentication and authorisation

### 4.1 API key

Every request must carry the header:

```
API_KEY: <key>
```

The key is matched against the `api_key` field of the records in `data/user.json`. No match (including a missing header) → **401 Unauthorized** with an empty body.

### 4.2 Permissions

Each user record carries an `endpoint_access` object mapping a resource name to a `{get, post, put, delete}` object of booleans:

```json
{
  "api_key": "d4s2a0b0a1n4a0l0y7t",
  "app": "analytics_dashboard",
  "endpoint_access": {
    "warehouses": { "get": true, "post": false, "put": false, "delete": false },
    "...": {}
  }
}
```

`has_access` takes the **first path segment after `/api/v1/`** as the resource name. A resource absent from `endpoint_access`, or a method flag that is `false`, yields **403 Forbidden** with an empty body.

Because only the first segment is inspected, sub-resources are governed by their parent's permissions: `/api/v1/warehouses/{id}/locations` is checked against `warehouses`, and `/api/v1/items/{id}/inventory` against `items`.

### 4.3 Configured applications

Six application keys ship in `data/user.json`. `G` = get, `P` = post, `U` = put, `D` = delete.

| Resource | analytics_dashboard | smartglass_reader | order_picker | receiving_station | shipping_station | facility_management |
| --- | --- | --- | --- | --- | --- | --- |
| warehouses | G | G | G | G | G | G P U D |
| locations | G | G | G | G | G | G P U D |
| items | G | G | G | G P U | G | G |
| item_lines | G | G | G | G P U | G | G P U D |
| item_groups | G | G | G | G P U | G | G P U D |
| item_types | G | G | G | G P U | G | G P U D |
| inventories | G | G U | G U | G P U | G U | G |
| suppliers | G | G | G | G P U | G | G P U D |
| clients | G | G | G | G | G P U | G P U D |
| orders | G | G | G U | G P U | G U | G |
| shipments | G | G | G | G P U | G P U | G |
| transfers | G | G U | G U | G P U | G P U | G |

Keys as stored:

| App | API key |
| --- | --- |
| analytics_dashboard | `d4s2a0b0a1n4a0l0y7t` |
| smartglass_reader | `s8m3a9r2t7g1l4a5s0s` |
| order_picker | `o3r5d4e2r1p6i8c0k0e` |
| receiving_station | `r2e4c6e8i0v3i5n7g9s` |
| shipping_station | `s1h3i5p7p9i2n4g6s8t` |
| facility_management | `f4a5c6i7l8i9t0y1m2a3n4a5g6` |

---

## 5. Conventions

### 5.1 Request

| Aspect | Behaviour |
| --- | --- |
| Base path | `/api/v1` |
| Auth header | `API_KEY` (required on every verb) |
| Body | Raw JSON on POST and PUT; `Content-Length` must be present and correct |
| Content negotiation | None; the request `Content-Type` is ignored |
| Query parameters | Not supported |

### 5.2 Response

| Status | When |
| --- | --- |
| `200 OK` | Successful GET (with `Content-type: application/json` and a JSON body), successful PUT, successful DELETE |
| `201 Created` | Successful POST; empty body |
| `401 Unauthorized` | `API_KEY` missing or unknown; empty body |
| `403 Forbidden` | Key valid but the resource/method is not permitted; empty body |
| `404 Not Found` | Unknown resource, unknown sub-path, or an unsupported number of path segments; empty body |
| `500 Internal Server Error` | Any unhandled exception (malformed id, malformed JSON body, missing `Content-Length`, missing referenced record, …); empty body |

Only successful GET responses carry a body and a `Content-type` header. A GET for an id that does not exist is **not** a 404: the lookup returns `null`, so the response is `200` with the body `null`.

---

## 6. Data model

Entities as stored in `data/`. Every top-level entity carries `created_at` and `updated_at` ISO-8601 UTC strings.

### warehouse (10 records)
`id`, `code`, `name`, `address`, `city`, `zip_code`, `province`, `country`, `contact_name`, `contact_phone`, `contact_email`

### location (400 records)
`id`, `warehouse_id` → warehouse, `code`, `name`

### item (600 records)
`id`, `code`, `description`, `barcode`, `model_number`, `commodity_code`, `unit_weight`, `item_line_id` → item_line, `item_group_id` → item_group, `item_type_id` → item_type, `min_purchase_qty`, `case_size`, `packaging_type` (`Single`, `Carton`, `Case`, `Pallet`; observed values in the data are `Carton`, `Case`, `Pallet` and the literal string `None`), `order_multiple`, `supplier_id` → supplier, `supplier_sku`

### item_line (14 records), item_group (3 records), item_type (3 records)
`id`, `name`, `description`. Item groups are handling regimes (`Vers`, …); item types are packaging tiers (`Single`, …).

### supplier (21 records)
`id`, `code`, `name`, `address`, `city`, `zip_code`, `province`, `country`, `contact_name`, `phone_number`, `reference`

### client (300 records)
`id`, `name`, `address`, `city`, `zip_code`, `province`, `country`, `contact_name`, `contact_phone`, `contact_email`

### inventory (4 800 records)
`item_id` + `location_id` (composite key, no `id`), `quantity_on_hand`, `quantity_expected`, `quantity_ordered`, `quantity_allocated`

### order (4 854 records) + order_item (26 498 records)
Order: `id`, `client_id`, `order_date`, `request_date`, `reference`, `customer_po_number`, `order_status` (`Pending`, `Shipped`), `shipping_notes`, `warehouse_id`, `ship_to_client_id`, `bill_to_client_id`
Order item: `id`, `order_id`, `item_id`, `amount`, `unit_price`

### shipment (6 132 records) + shipment_item (33 660 records)
Shipment: `id`, `reference`, `order_id` (a shipment references at most one order), `shipment_date`, `shipment_type` (`Incoming`, `Outgoing`), `shipment_status` (`Scheduled`, `Transit`, `Delivered`), `carrier_name`, `shipping_method` (`Standard`, `Express`, `Overnight`, `Freight`), `payment_type` (`Automated`, `Manual`)
Shipment item: `id`, `shipment_id`, `item_id`, `amount`

### transfer (800 records) + transfer_item (2 370 records)
Transfer: `id`, `reference`, `from_location_id` → location, `to_location_id` → location, `transfer_status` (`Scheduled`, `Processed`)
Transfer item: `id`, `transfer_id`, `item_id`, `amount`

### user (6 records)
`api_key`, `app`, `endpoint_access`

---

## 7. Endpoints

All paths are prefixed with `/api/v1`. `{id}` is an integer.

### 7.1 Warehouses

| Method | Path | Description |
| --- | --- | --- |
| GET | `/warehouses` | All warehouses |
| GET | `/warehouses/{id}` | One warehouse, or `null` |
| GET | `/warehouses/{id}/locations` | All location objects whose `warehouse_id` matches |
| POST | `/warehouses` | Create from the JSON body; `id` must be supplied |
| PUT | `/warehouses/{id}` | Replace the record |
| DELETE | `/warehouses/{id}` | Remove the record |

### 7.2 Locations

| Method | Path | Description |
| --- | --- | --- |
| GET | `/locations` | All locations |
| GET | `/locations/{id}` | One location, or `null` |
| POST | `/locations` | Create |
| PUT | `/locations/{id}` | Replace |
| DELETE | `/locations/{id}` | Remove |

### 7.3 Items

| Method | Path | Description |
| --- | --- | --- |
| GET | `/items` | All items |
| GET | `/items/{id}` | One item, or `null` |
| GET | `/items/{id}/inventory` | All inventory rows for the item, one per location |
| GET | `/items/{id}/inventory/totals` | Aggregate across all locations: `{total_expected, total_ordered, total_allocated, total_available}`, where `total_available` sums `quantity_on_hand - quantity_allocated` |
| POST | `/items` | Create |
| PUT | `/items/{id}` | Replace |
| DELETE | `/items/{id}` | Remove |

### 7.4 Item lines

| Method | Path | Description |
| --- | --- | --- |
| GET | `/item_lines` | All item lines |
| GET | `/item_lines/{id}` | One item line, or `null` |
| GET | `/item_lines/{id}/items` | Array of item **ids** belonging to the line |
| POST | `/item_lines` | Create |
| PUT | `/item_lines/{id}` | Replace |
| DELETE | `/item_lines/{id}` | Remove |

### 7.5 Item groups

| Method | Path | Description |
| --- | --- | --- |
| GET | `/item_groups` | All item groups |
| GET | `/item_groups/{id}` | One item group, or `null` |
| GET | `/item_groups/{id}/items` | Array of item **ids** belonging to the group |
| POST | `/item_groups` | Create |
| PUT | `/item_groups/{id}` | Replace |
| DELETE | `/item_groups/{id}` | Remove |

### 7.6 Item types

| Method | Path | Description |
| --- | --- | --- |
| GET | `/item_types` | All item types |
| GET | `/item_types/{id}` | One item type, or `null` |
| GET | `/item_types/{id}/items` | Array of item **ids** of that type |
| POST | `/item_types` | Create |
| PUT | `/item_types/{id}` | Replace |
| DELETE | `/item_types/{id}` | Remove |

### 7.7 Inventories

| Method | Path | Description |
| --- | --- | --- |
| GET | `/inventories` | All inventory rows |
| GET | `/inventories/{id}` | **404** — inventory has no surrogate id |
| POST | `/inventories` | Upsert on `(item_id, location_id)`: merges into the existing row when one exists, otherwise appends |
| PUT | `/inventories/...` | **404** for every form |
| DELETE | `/inventories/...` | **404** for every form |

Per-item inventory is read through `/items/{id}/inventory` and `/items/{id}/inventory/totals`.

### 7.8 Suppliers

| Method | Path | Description |
| --- | --- | --- |
| GET | `/suppliers` | All suppliers |
| GET | `/suppliers/{id}` | One supplier, or `null` |
| GET | `/suppliers/{id}/items` | Full item **objects** supplied by this supplier |
| POST | `/suppliers` | Create |
| PUT | `/suppliers/{id}` | Replace |
| DELETE | `/suppliers/{id}` | Remove |

### 7.9 Clients

| Method | Path | Description |
| --- | --- | --- |
| GET | `/clients` | All clients |
| GET | `/clients/{id}` | One client, or `null` |
| GET | `/clients/{id}/orders` | Orders where the client appears as `client_id`, `ship_to_client_id` or `bill_to_client_id`, each enriched with `items` |
| POST | `/clients` | Create |
| PUT | `/clients/{id}` | Replace |
| DELETE | `/clients/{id}` | Remove |

### 7.10 Orders

| Method | Path | Description |
| --- | --- | --- |
| GET | `/orders` | All orders, each with an `items` array |
| GET | `/orders/{id}` | One order with its `items`, or `null` |
| GET | `/orders/{id}/items` | The order's items as `{item_id, amount}` |
| POST | `/orders` | Create. `items` in the body is stored as `order_item` rows; the header keeps the remaining fields |
| PUT | `/orders/{id}` | Replace the header. When the body carries `items`, they are replaced too and inventory `quantity_allocated` is adjusted (§3.6) |
| PUT | `/orders/{id}/items` | Replace only the order's items and adjust `quantity_allocated` |
| DELETE | `/orders/{id}` | Remove the order and all its `order_item` rows |

### 7.11 Shipments

| Method | Path | Description |
| --- | --- | --- |
| GET | `/shipments` | All shipments, each with an `items` array |
| GET | `/shipments/{id}` | One shipment with its `items`, or `null` |
| GET | `/shipments/{id}/items` | The shipment's items as `{item_id, amount}` |
| GET | `/shipments/{id}/orders` | Array holding the shipment's single `order_id`, or empty when unset |
| POST | `/shipments` | Create; `items` becomes `shipment_item` rows |
| PUT | `/shipments/{id}` | Replace the header. When the body carries `items`, they are replaced too and inventory `quantity_ordered` is adjusted (§3.6) |
| PUT | `/shipments/{id}/items` | Replace only the shipment's items and adjust `quantity_ordered` |
| PUT | `/shipments/{id}/orders` | Body is an array of order ids; the first entry becomes `order_id`, an empty array clears it to `null` |
| DELETE | `/shipments/{id}` | Remove the shipment and all its `shipment_item` rows |

### 7.12 Transfers

| Method | Path | Description |
| --- | --- | --- |
| GET | `/transfers` | All transfers, each with an `items` array |
| GET | `/transfers/{id}` | One transfer with its `items`, or `null` |
| GET | `/transfers/{id}/items` | The transfer's items as `{item_id, amount}` |
| POST | `/transfers` | Create; `transfer_status` is forced to `Scheduled` regardless of the body, `items` becomes `transfer_item` rows, and a notification is queued |
| PUT | `/transfers/{id}` | Replace the header; when the body carries `items` the `transfer_item` rows are replaced (no inventory change) |
| PUT | `/transfers/{id}/commit` | Execute the transfer: move each item's `amount` from the source location's `quantity_on_hand` to the destination's, set `transfer_status` to `Processed`, queue a notification, and persist both transfer and inventory. No request body is read |
| DELETE | `/transfers/{id}` | Remove the transfer and all its `transfer_item` rows |

### 7.13 Path summary

```
GET     /api/v1/warehouses
GET     /api/v1/warehouses/{id}
GET     /api/v1/warehouses/{id}/locations
POST    /api/v1/warehouses
PUT     /api/v1/warehouses/{id}
DELETE  /api/v1/warehouses/{id}

GET     /api/v1/locations
GET     /api/v1/locations/{id}
POST    /api/v1/locations
PUT     /api/v1/locations/{id}
DELETE  /api/v1/locations/{id}

GET     /api/v1/items
GET     /api/v1/items/{id}
GET     /api/v1/items/{id}/inventory
GET     /api/v1/items/{id}/inventory/totals
POST    /api/v1/items
PUT     /api/v1/items/{id}
DELETE  /api/v1/items/{id}

GET     /api/v1/item_lines
GET     /api/v1/item_lines/{id}
GET     /api/v1/item_lines/{id}/items
POST    /api/v1/item_lines
PUT     /api/v1/item_lines/{id}
DELETE  /api/v1/item_lines/{id}

GET     /api/v1/item_groups
GET     /api/v1/item_groups/{id}
GET     /api/v1/item_groups/{id}/items
POST    /api/v1/item_groups
PUT     /api/v1/item_groups/{id}
DELETE  /api/v1/item_groups/{id}

GET     /api/v1/item_types
GET     /api/v1/item_types/{id}
GET     /api/v1/item_types/{id}/items
POST    /api/v1/item_types
PUT     /api/v1/item_types/{id}
DELETE  /api/v1/item_types/{id}

GET     /api/v1/inventories
POST    /api/v1/inventories

GET     /api/v1/suppliers
GET     /api/v1/suppliers/{id}
GET     /api/v1/suppliers/{id}/items
POST    /api/v1/suppliers
PUT     /api/v1/suppliers/{id}
DELETE  /api/v1/suppliers/{id}

GET     /api/v1/clients
GET     /api/v1/clients/{id}
GET     /api/v1/clients/{id}/orders
POST    /api/v1/clients
PUT     /api/v1/clients/{id}
DELETE  /api/v1/clients/{id}

GET     /api/v1/orders
GET     /api/v1/orders/{id}
GET     /api/v1/orders/{id}/items
POST    /api/v1/orders
PUT     /api/v1/orders/{id}
PUT     /api/v1/orders/{id}/items
DELETE  /api/v1/orders/{id}

GET     /api/v1/shipments
GET     /api/v1/shipments/{id}
GET     /api/v1/shipments/{id}/items
GET     /api/v1/shipments/{id}/orders
POST    /api/v1/shipments
PUT     /api/v1/shipments/{id}
PUT     /api/v1/shipments/{id}/items
PUT     /api/v1/shipments/{id}/orders
DELETE  /api/v1/shipments/{id}

GET     /api/v1/transfers
GET     /api/v1/transfers/{id}
GET     /api/v1/transfers/{id}/items
POST    /api/v1/transfers
PUT     /api/v1/transfers/{id}
PUT     /api/v1/transfers/{id}/commit
DELETE  /api/v1/transfers/{id}
```

---

## 8. Examples

List warehouses:

```sh
curl -H "API_KEY: d4s2a0b0a1n4a0l0y7t" http://localhost:3000/api/v1/warehouses
```

Inventory totals for item 1:

```sh
curl -H "API_KEY: d4s2a0b0a1n4a0l0y7t" \
     http://localhost:3000/api/v1/items/1/inventory/totals
```

```json
{ "total_expected": 1191, "total_ordered": 858, "total_allocated": 876, "total_available": 1018 }
```

Create a transfer (queued as `Scheduled`):

```sh
curl -X POST -H "API_KEY: r2e4c6e8i0v3i5n7g9s" \
     -d '{"id": 9001, "reference": "TRF-009001", "from_location_id": 12, "to_location_id": 34,
          "items": [{"item_id": 190, "amount": 10}]}' \
     http://localhost:3000/api/v1/transfers
```

Commit it, moving the stock:

```sh
curl -X PUT -H "API_KEY: r2e4c6e8i0v3i5n7g9s" \
     http://localhost:3000/api/v1/transfers/9001/commit
```

Replace an order's lines (adjusting `quantity_allocated`):

```sh
curl -X PUT -H "API_KEY: o3r5d4e2r1p6i8c0k0e" \
     -d '[{"item_id": 82, "amount": 12}, {"item_id": 119, "amount": 4}]' \
     http://localhost:3000/api/v1/orders/1/items
```
