সব verify করা শেষ — এখন পুরো execution plan দিচ্ছি। তার আগে শেষ investigation-এ পাওয়া **দুটো গুরুত্বপূর্ণ তথ্য**, যেগুলো plan-কে প্রভাবিত করেছে:

1. **StockOut-এ `PurchasePrice` populate হয় না** — [StockOutModel.cs:126-136](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/IMS/Models/StockOutModel.cs#L126-L136)-এ শুধু `UnitPrice`, `Vat` set হয়, `PurchasePrice` বাদ। আর stock আসলে কমে `UpdateStatusAsync`-এ Completed status-এ ([StockOutService.cs:271](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Services/Areas/IMS/StockOutService.cs#L271))।
2. **`SalesStockOutItem`-এ `WarehouseId` নেই**, এবং `StockOutByStockIdAsync` ([ItemStockService.cs:167](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Services/ItemStockService.cs#L167)) warehouse ignore করে শুধু `ProductId + BatchNo` দিয়ে প্রথম row থেকে কমায় — মানে ভুল store থেকে stock কমতে পারে। Stock Levels যেহেতু per-store report, এটা ঠিক করা দরকার।

---

# Stock Levels Development — Full Execution Plan

## Phase 0 — Price Infrastructure (foundation, সবার আগে)

এটা আগে না করলে ABC আর Stock Value কখনোই নির্ভরযোগ্য হবে না।

**Step 0.1 — `ItemStock`-এ `Vat` column**

- [ItemStock.cs](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Entities/ItemStock.cs)-এ `public double? Vat { get; set; }` যোগ (base `PurchasePrice` already আছে, VAT-ছাড়া দাম রাখব)
- Migration: `Add-Migration UpdateItemStockWithVat`

**Step 0.2 — `SalesStockOutItem`-এ `WarehouseId` column** _(decision লাগবে, নিচে দেখুন)_

- `public Guid? WarehouseId { get; set; }` + navigation + migration
- StockOut Add/Edit UI-তে warehouse dropdown (batch select করলে auto-set করা যায়)

**Step 0.3 — Moving Weighted Average সহ নতুন stock-in method**

- [IItemStockService.cs](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Services/Contracts/IItemStockService.cs) + [ItemStockService.cs](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Services/ItemStockService.cs)-এ:

```csharp
Task<Guid> StockInWithPriceExpDateAsync(Guid userId, Guid productId, Guid warehouseId,
    double quantity, DateTime? expDate, string batchNo, double? purchasePrice, double? vat);
```

- Logic: row না থাকলে নতুন row দাম-সহ; থাকলে —

```csharp
var totalQty = stock.QuantityStock + quantity;
if (purchasePrice != null && totalQty > 0)
    stock.PurchasePrice = ((stock.QuantityStock * (stock.PurchasePrice ?? purchasePrice.Value))
                          + (quantity * purchasePrice.Value)) / totalQty;   // একই logic Vat-এর জন্যও
stock.QuantityStock = totalQty;
```

- পুরনো `StockInWithExpDateAsync` অক্ষত থাকবে (অন্য caller ভাঙবে না)

**Step 0.4 — GRN flow-তে দাম পাস করা**

- [GoodsReceiveService.cs:307](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Services/Areas/SCM/GoodsReceiveService.cs#L307)-এ নতুন method call
- দাম source: `GoodsReceiveProduct.UnitPrice`; null হলে `GoodsReceive.PurchaseOrderId` → `PurchaseOrderProduct` (ProductId match) থেকে `UnitPriceWithoutVat`
- VAT: PO line-এর `VatPercentage` থেকে per-unit VAT amount

**Step 0.5 — StockOut-এ cost stamping**

- [StockOutService.cs](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Services/Areas/IMS/StockOutService.cs)-এর `UpdateStatusAsync`-এ (Completed হওয়ার মুহূর্তে, stock কমানোর ঠিক আগে): প্রতিটা item-এর জন্য `ItemStock` row (ProductId+BatchNo+WarehouseId) থেকে `PurchasePrice` পড়ে `SalesStockOutItem.PurchasePrice`-এ লিখে save
- Draft-এ stamp করব না — draft আর completion-এর মাঝে দাম বদলাতে পারে; **stock যখন কমে তখনকার দামই সঠিক cost**

**Step 0.6 — `StockOutByStockIdAsync` fix**

- Warehouse-aware overload: `ProductId + BatchNo + WarehouseId` দিয়ে সঠিক row থেকে কমানো
- Insufficient stock হলে এখন silently কিছু হয় না ([ItemStockService.cs:170](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Services/ItemStockService.cs#L170)) — exception/result return করে UpdateStatus-এ handle করা

**Step 0.7 — Backfill script (one-time SQL)**

- `ItemStock.PurchasePrice IS NULL` → `GoodsReceiveProduct`-এর সাথে `ProductId + BatchNo` join করে সর্বশেষ received দাম
- Match না হলে → `Product.PurchasePrice`
- পুরনো Completed `SalesStockOutItem.PurchasePrice`-ও একইভাবে
- Script টা `docs/` বা migration-এ রাখা, dev DB-তে আগে test

## Phase 1 — Stock Levels Core Grid (real data)

**Step 1.1 — Query objects** — `HMS.Entities/NotMapped/StockLevelQuery.cs`:

- `StockLevelQuery : IQueryObject` → `CategoryId`, `WarehouseId`, `Status (enum)`, `Ved (EnumProductImportance?)`, `SearchText`, `Page`, `PageSize`
- `StockLevelReportItem` DTO → ProductId, ProductCode, ProductName, CategoryName, WarehouseName, CurrentQty, UnitName, MinimumQty, MaximumQty, DoC, Status, Ved, Abc, LastIssuedDate, StockValue
- `StockLevelKpi` → TotalSku, OkCount, LowCount, CriticalCount, StockOutCount, ExcessCount, ExpiringCount

**Step 1.2 — Status enum** — `HMS.Common/Enums/EnumStockLevelStatus.cs`: OK, Low, Critical, StockOut, Excess। Derivation rule (server-side, LINQ-এ):

|শর্ত|Status|
|---|---|
|CurrentQty ≤ 0|Stock-Out|
|CurrentQty < MinimumQty × 0.5|Critical|
|CurrentQty < MinimumQty|Low|
|CurrentQty > MaximumQty (Max set থাকলে)|Excess|
|নাহলে|OK|
|MinimumQty null হলে|শুধু Stock-Out/OK|

**Step 1.3 — Repository** — [ProductRepository.cs](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Repository/ProductRepository.cs)-এ `StockLevelReportAsync(StockLevelQuery)`:

- `ItemStock` group by `ProductId + WarehouseId` (batch rows collapse) → SUM(QuantityStock), SUM(Qty×PurchasePrice)
- `Product` join (Category, MeasurementUnit include), filters apply, status severity-তে sort (Stock-Out → Critical → Low আগে), paging → `QueryResult<StockLevelReportItem>`
- KPI-র জন্য একই filtered query-র `GroupBy(Status).Count()` — আলাদা lightweight call
- Interface update: `IProductRepository`

**Step 1.4 — Service passthrough** — `IProductService` + [ProductService.cs](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Services/ProductService.cs) (existing `ProductStockReport`-এর pattern-এই)

**Step 1.5 — Controller + Model** — codebase-এর pattern মেনে `HMS.Web/Areas/IMS/Models/StockLevelModel.cs` (service call wrapper) এবং [StockLevelsController.cs](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/IMS/Controllers/StockLevelsController.cs):

- `Index(StockLevelQuery query)` → data + KPI + dropdown data (Category list, Warehouse list) → View
- Paging: `StaticPagedList` (StockOutController-এর মতো)

**Step 1.6 — View rewrite** — [Index.cshtml](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/IMS/Views/StockLevels/Index.cshtml):

- Static ১৫ row মুছে `@foreach`, badge-গুলো enum switch দিয়ে
- Filter dropdowns DB-bound, form GET submit, search textbox
- KPI cards model-bound; প্রতিটা card clickable → সেই status filter
- Pagination partial, "Showing X of Y"
- Help banner-এর DEV NOTE অনুযায়ী banner remove
- ABC আর DoC column আপাতত "—" (Phase 2/3-এ আসবে)

## Phase 2 — Consumption Metrics (DoC, Last Issued, Expiring)

**Step 2.1 — Consumption aggregate** — repository-তে: গত ৯০ দিনের Completed `SalesStockOutItem` group by `ProductId (+WarehouseId)` → `SUM(Quantity)/90` = avg daily, `MAX(StockOutDate)` = Last Issued **Step 2.2 — DoC** = CurrentQty ÷ avgDaily; consumption না থাকলে "N/A" **Step 2.3 — StockOutType filter** — কোন type গুলো আসল consumption (transfer/damage বাদ) সেটা define করে `Where` clause-এ; type-এর distinct value গুলো আগে DB-তে দেখে নেব **Step 2.4 — Expiring KPI** — `ItemStock.Exp <= Today+30 && Exp >= Today && QuantityStock > 0` → distinct product count; সাথে "Expiring" filter option

## Phase 3 — ABC Classification

**Step 3.1** — `EnumAbcClass` (A=1,B=2,C=3) + [Product.cs](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Entities/Product.cs)-এ nullable `AbcClass` + `AbcCalculatedOn` date + migration **Step 3.2** — নতুন `AbcAnalysisService`:

- গত ১২ মাসের Completed consumption value per product = `SUM(Quantity × PurchasePrice)`; PurchasePrice null হলে WAC fallback (`GoodsReceiveProduct` GROUP BY ProductId), তারপর `Product.PurchasePrice`
- Value descending sort → cumulative % → ≤70% = A, ≤90% = B, বাকি = C → Product-এ save **Step 3.3** — Trigger: [AbcVedAnalysisController](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/IMS/Controllers/AbcVedAnalysisController.cs) page-এ "Recalculate" button (POST) + calculation summary দেখানো **Step 3.4** — Stock Levels view-তে ABC badge (`Product.AbcClass` থেকে — VED-এর মতোই, query-তে কোনো বাড়তি cost নেই)

## Phase 4 — Export ও Polish

**Step 4.1** — Excel Export: ClosedXML already ব্যবহার হয় ([MaterialController](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/IMS/Controllers/MaterialController.cs)-এর pattern) — একই filter, paging ছাড়া full data, `ExportToExcel(StockLevelQuery)` action **Step 4.2** — Row click → product-এর batch-wise details (existing `ProductStockDetailsReport` reuse/link) **Step 4.3** — RBAC verify — `RbacDataSeederService`-এ StockLevels permission entry আছে কি না, menu link ঠিক আছে কি না **Step 4.4** — `docs/modules/IMS.md` update

## Testing Checklist (প্রতি phase শেষে)

- [ ]  Solution build + migration dev DB-তে apply
- [ ]  **GRN cycle:** PO → GRN → UpdateStatus → `ItemStock`-এ দাম+VAT ঢুকেছে; একই batch-এ ভিন্ন দামে ২য় GRN → weighted average ঠিক (হাতে হিসাব মিলিয়ে)
- [ ]  **StockOut cycle:** Draft → Completed → `SalesStockOutItem.PurchasePrice` stamped, সঠিক warehouse-এর row থেকে qty কমেছে
- [ ]  Stock Levels: প্রতিটা filter একা+combined, paging, KPI সংখ্যা table-এর সাথে মেলে, status rule-এর ৫টা case-ই manually verify
- [ ]  Backfill: আগে-পরে row count, null price count
- [ ]  Regression: Material Stock Report, ProductStockReport, GoodsReceive আগের মতোই কাজ করে
- [ ]  Export-এর data grid-এর সাথে মেলে

## আপনার কাছ থেকে ৪টা সিদ্ধান্ত লাগবে

1. **Stock-out-এ WarehouseId যোগ করব?** (Step 0.2) — আমার মত: হ্যাঁ, নাহলে per-store report কখনোই accurate হবে না। কিন্তু এতে StockOut-এর UI-তেও change লাগবে।
2. **Reorder/Max product-level থাকবে, নাকি per-store?** — আমার মত: এখন product-level (`MinimumQty`/`MaximumQty` যেমন আছে), পরে দরকার হলে per-store override।
3. **Cost-এ VAT ধরা হবে কি?** — hospital input VAT credit না পেলে VAT-ও cost; আলাদা column রাখছি বলে পরে সিদ্ধান্ত বদলানো যাবে। Default: ABC-তে VAT-ছাড়া।
4. **Window:** DoC-র জন্য ৯০ দিন, ABC-র জন্য ১২ মাস — ঠিক আছে?

## Execution Order

Phase 0 (Step 0.1→0.7) → Phase 1 → Phase 2 → Phase 3 → Phase 4 — এই ক্রমেই, কারণ প্রতিটা পরেরটার foundation। Phase 0+1 একসাথে করলেই page টা কাজ শুরু করবে (ABC/DoC ছাড়া); বাকিগুলো incremental। আনুমানিক ছোঁয়া লাগবে ~২০-২২টা ফাইল, ৩টা migration, ১টা backfill script।

৪টা সিদ্ধান্ত জানালেই আমি Phase 0 থেকে implementation শুরু করে দেব।