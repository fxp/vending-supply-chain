# AGENTS.md — vending-supply-chain

> 独立供应链管理 API（单 Cloudflare Worker + KV）。Project Vend 的库存/采购后端。
> 本文件供**独立开发本 repo 的 Agent** 上手；完整 API 见 [API.md](API.md) / [README.md](README.md)。

## 角色定位
- harness 的 `inventory`（已落地 `InventoryHTTPAdapter`）+ 未来 `sourcing`/`demand`/`group_buy` 能力，`VSC_URL` 指向本服务。
- 提供：机器/货道/库存、SKU（成本→毛利、MOQ→凑箱）、供应商（lead_time→SLA）、采购单、代订（缺货需求）、每日补货预览/触发。
- **无鉴权**（内部/后台 API）；生产请置于私网/网关后。

## 技术栈 / 绑定（`wrangler.jsonc`）
- TS + Cloudflare Worker，`src/index.ts`。
- **KV** `SC_KV`（全部持久化，无 DB）· **Durable Object** `DASHBOARD: DashboardRoom`（`/ws` 实时看板）· `assets: ./public`（内嵌看板）。
- `migrations: [{tag:"v1", new_classes:["DashboardRoom"]}]`。**`account_id` 在 `wrangler.jsonc`**。

## 关键端点（详见 API.md）
```
GET  /health                          {status:"ok", time}
GET  / · /dashboard · /ws             内嵌看板
GET/POST /suppliers · /skus · /machines        CRUD
GET  /machines/{id}/inventory · /inventory?machine_id=   库存
GET  /inventory/search?name=          跨机找货
POST /inventory/restock               手动补货
GET/POST /purchase-orders · /preorders           采购单 / 代订
GET  /internal/daily-order/preview · POST /internal/daily-order(dry_run)   每日补货
```
金额：`price_fen`/`cost_fen`/`retail_fen` **整数分**；`moq` 凑箱；`lead_time_days` 算 SLA。

## 开发 / 测试 / 部署
```bash
npm install
npm run dev
npm run deploy    # 需 wrangler 已登录；或走 CI
```

## CI/CD（GitHub Actions，本 repo 自带 `.github/workflows/ci.yml`）
- push `main` → typecheck → deploy → curl `/health` 冒烟。
- secret **`CLOUDFLARE_API_TOKEN`**：有 **KV 绑定** → token **必须含 Workers KV Storage:Edit + Workers Scripts**（否则 `code 10023`）。account_id 已在 `wrangler.jsonc`。

## 不可破坏的契约（harness 依赖）
- `GET /health` 含 `"status":"ok"`（看板判活）。
- `/machines`、`/inventory`、`/skus`、`/suppliers`、`/preorders` 的字段（`lane_id`/`machine_id`/`sku_id`/`price_fen`/`low_stock`…）匹配 harness `InventoryHTTPAdapter`。
- ⚠️ **id 命名空间**：harness mock 用 `prod_*`、本服务用自有 `sku_id`、UVM 用 catalog id；harness 侧靠 `SkuMap` 归一。改 id 字段语义前先看 harness `models/sku.py` 与 `docs/external-services.md` 的字段映射表。

## 关系
同族外部服务见 `vending-status-dashboard` 监控列表；与 `ucp-vending-machine` 经 catalog/lane 对齐。
