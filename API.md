# vending-supply-chain — API Reference

Supply-chain management API for a network of vending machines. Single Cloudflare Worker + KV, no database.

- **Production:** `https://vending-supply-chain.fxp007.workers.dev`
- **Local dev:** `npx wrangler dev` → `http://localhost:8787`

## Conventions

- **No authentication** — this is an internal/back-office API. Put it behind a private network or gateway in production.
- **Money** is integer **fen** (1 CNY = 100 fen): `cost_fen`, `retail_fen`, `price_fen`. The one exception is the supplier's `min_order_yuan`, which is whole **yuan**.
- **IDs:** purchase orders are `PO-<12hex>`, preorders are `PRE-<12hex>`; suppliers / SKUs / machines / lanes use caller-supplied ids.
- **Errors:** `{ "error": "message" }` with HTTP `400` (bad request) or `404` (not found).
- Most `GET` list endpoints return a bare JSON array.

## Endpoints

### Health & dashboard

| Method | Path | Description |
|---|---|---|
| GET | `/health` | `{ "status": "ok", "time": "..." }` |
| GET | `/` · `/dashboard` | Dashboard (redirects to static `index.html`) |
| GET | `/ws` | WebSocket upgrade — live dashboard stream (Durable Object) |

### Suppliers

| Method | Path | Description |
|---|---|---|
| GET | `/suppliers` | List suppliers |
| POST | `/suppliers` | Create. Required: `id`, `name`, `lead_time_days` (number), `min_order_yuan` (number) → `201` |
| GET | `/suppliers/{id}` | One supplier, or `404` |
| PUT | `/suppliers/{id}` | Partial update |
| DELETE | `/suppliers/{id}` | `{ "deleted": true, "id": "..." }` |
| GET | `/suppliers/{id}/skus?keyword=` | SKUs for this supplier (optional name/id `keyword` filter) |

### SKUs

| Method | Path | Description |
|---|---|---|
| GET | `/skus` | List SKUs |
| POST | `/skus` | Create. Required: `sku_id`, `name`, `cost_fen` (number), `retail_fen` (number), `moq` (number), `supplier_id` → `201` |
| GET | `/skus/{id}` | One SKU, or `404` |
| PUT | `/skus/{id}` | Partial update |
| DELETE | `/skus/{id}` | Delete |

### Machines & lanes

| Method | Path | Description |
|---|---|---|
| GET | `/machines` | List machines, each with `total_items` + `low_stock_count` stats |
| POST | `/machines` | Create. Required: `id`, `name`. Optional: `location` → `201` |
| GET | `/machines/{id}` | Machine + stats + its `lanes` |
| PUT | `/machines/{id}` | Partial update |
| DELETE | `/machines/{id}` | Delete |
| GET | `/machines/{id}/lanes` | Lane configs for a machine |
| POST | `/machines/{id}/lanes` | Create lane. Required: `lane_id`, `sku_id`, `name`, `price_fen` (number), `min_qty` (number), `capacity` (number). Optional: `currency` (def `CNY`), `location`. Seeds inventory at qty 0 → `201` |
| PUT | `/machines/{id}/lanes/{laneId}` | Update lane (validates `sku_id` if changed) |
| DELETE | `/machines/{id}/lanes/{laneId}` | Delete lane + its inventory |

### Inventory

| Method | Path | Description |
|---|---|---|
| GET | `/machines/{id}/inventory` | Full inventory for a machine |
| GET | `/inventory?machine_id=&low_stock_only=` | Inventory (flat). `machine_id` def `vm-001`; `low_stock_only=true` filters to low lanes |
| GET | `/inventory/{laneId}?machine_id=` | One lane's live inventory + `low_stock` flag |
| GET | `/inventory/search?name=` | Cross-machine item search (`name` or `q`); returns matching lanes with `available`/`low_stock` |
| POST | `/inventory/restock` | Manual restock (see below) |

**`POST /inventory/restock`**
```json
{ "machine_id": "vm-001", "po_id": "PO-...",
  "items": [ { "lane_id": "A1", "sku_id": "COKE_330", "qty": 20 } ] }
```
Each item is capped at the lane `capacity`; SKU must match the lane. Returns `{ po_id, restocked_at, results: [{ lane_id, sku_id, added, new_qty }] }`. Per-item errors (`Lane not found`, `SKU mismatch`) appear inline in `results`.

### Purchase orders

| Method | Path | Description |
|---|---|---|
| GET | `/purchase-orders?status=&supplier_id=` | List POs, optionally filtered |
| POST | `/purchase-orders` | Create. Required: `supplier_id`, `items:[{sku_id, qty>=1}]`. Optional: `machine_id` (def `vm-001`), `note`, `priority` (def `normal`) → `201`, `status: "draft"` |
| GET | `/purchase-orders/{id}` | One PO |
| POST | `/purchase-orders/{id}/advance` | Advance status (see below) |

**PO status machine:** `draft → submitted → acknowledged → shipped → received → stocked`

```
POST /purchase-orders/{id}/advance        # advance one step
POST /purchase-orders/{id}/advance  { "to_status": "shipped" }   # jump forward (forward-only)
```
Advancing **to `stocked`** automatically adds the ordered quantities to inventory and flips matching preorders to `stock_arrived`. Backward/same-status jumps → `400`.

### Preorders (out-of-stock demand)

| Method | Path | Description |
|---|---|---|
| GET | `/preorders?status=&user_id=` | List preorders, optionally filtered |
| POST | `/preorders` | Create. Required: `sku_id`, `qty>=1`, `user_id`. Optional: `sku_name`, `note`, `notify_channel` (def `none`) → `201`, `status: "pending"` |
| GET | `/preorders/{id}` | One preorder |
| POST | `/preorders/{id}/notify` | Stamp `notified_at`; `stock_arrived → notified` |

### Auto-order (internal / cron)

| Method | Path | Description |
|---|---|---|
| GET | `/internal/daily-order/preview?machine_id=` | Preview what the daily run would order (no writes) |
| POST | `/internal/daily-order` | Run it. Body `{ "machine_id": "vm-001", "dry_run": true }` — `dry_run` → `200` no writes; otherwise creates POs → `201` |

A **cron trigger** (`0 2 * * *`, daily 02:00 UTC) runs the same logic: it creates POs for lanes below `min_qty`, grouped by supplier, skipping any order below the supplier's `min_order_yuan`.

## Data models

```jsonc
Supplier { id, name, lead_time_days, min_order_yuan }
Sku      { sku_id, name, cost_fen, retail_fen, moq, supplier_id }
Machine  { id, name, location }                         // GET adds total_items, low_stock_count
LaneConfig { lane_id, machine_id, sku_id, name, price_fen, currency, min_qty, capacity, location }
Inventory  { lane_id, sku_id, name, price_fen, currency, qty, low_stock, last_restocked_at }
PurchaseOrder { id, supplier_id, supplier_name, machine_id, items[{sku_id,sku_name,qty,cost_fen}],
                note, priority, status, created_at, updated_at, status_history[{status,at}] }
Preorder { id, sku_id, sku_name, qty, user_id, note, notify_channel, status, created_at, updated_at, notified_at }
```

## KV schema

| Key pattern | Description |
|---|---|
| `supplier:{id}` · `sku:{id}` · `machine:{id}` | Core records |
| `lane:{machineId}:{laneId}` | Lane config |
| `inv:{machineId}:{laneId}` | Live inventory `{ qty, last_restocked_at }` |
| `po:{id}` · `pre:{id}` | Purchase orders / preorders |
