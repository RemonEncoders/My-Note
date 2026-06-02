পুরো picture পেয়েছি। নিচে আপনার `VitalityCash` project-এর এই ১৩টা entity-এর সম্পূর্ণ database relationship + system design বাংলায় ব্যাখ্যা করছি।

---

# VitalityCash — Accounting + Inventory Entity Map (বাংলা)

## ১. মূল ধারণা — তিনটি Layer

আপনার system-এ এই entity-গুলো তিনটি layer-এ ভাগ হয়েছে:

```
┌─────────────────────────────────────────────────────────────┐
│  LAYER 1 — MASTER / SETUP (একবার configure করে রাখা)        │
│  ChartOfAcc → AccGroup → GlMaster                           │
│  ValuationClass + MaterialTransactionCode                   │
│  ValuationClassTransaction (mapping)                        │
│  GeneralSetting (lookup key-value)                          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  LAYER 2 — BUSINESS TRANSACTION (দৈনন্দিন কাজ)               │
│  GoodsReceive ── GoodsReceiveProduct                        │
│  InvoiceReceive ── InvoiceReceiveGl / InvoiceReceivePO      │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼ (auto-posting)
┌─────────────────────────────────────────────────────────────┐
│  LAYER 3 — ACCOUNTING POSTING (auto-generated)              │
│  MaterialTransaction ── MaterialTransactionItem             │
│  JournalEntry ── JournalEntryItem                           │
└─────────────────────────────────────────────────────────────┘
```

**কেন এই design?** Business user `GoodsReceive` বা `InvoiceReceive` save করে → system **আপনাআপনি** Layer 3-এ stock movement (`MaterialTransaction`) আর accounting entry (`JournalEntry`) তৈরি করে। User-কে separate accounting entry দিতে হয় না।

---

## ২. Entity-by-Entity ব্যাখ্যা

### 🔹 LAYER 1 — MASTER

#### **`ChartOfAcc`** ([ChartOfAcc.cs:11](d:/Web Dev/VitalityCash/VitalityCash.Entities/ChartOfAcc.cs#L11))

একটা company-র **পুরো accounting structure-এর root**। একটা company-র একটাই Chart of Accounts থাকে (BD-তে BAS / IFRS অনুযায়ী)।

|Field|অর্থ|
|---|---|
|`AccName`, `AccNo`|Chart নাম + number|
|`BaseCurrency`|মূল মুদ্রা (e.g., BDT)|
|`FiscalYearStartMonth`|বছর কোন month-এ শুরু (Bangladesh = July, তাই = 7)|
|`LengthOfGl`|GL code কত digit হবে (e.g., 4, 6)|

**Relation:** এর under-এ অনেক `AccGroup` থাকবে।

---

#### **AccGroup** ([AccGroup.cs:11](d:/Web Dev/VitalityCash/VitalityCash.Entities/AccGroup.cs#L11))

Chart of Accounts-কে category-তে ভাগ করার layer।

| Field                          | অর্থ                                                           |
| ------------------------------ | -------------------------------------------------------------- |
| `AccGroupCode`, `AccGroupName` | Group identifier                                               |
| `Category`                     | Asset / Liability / Revenue / Expense                          |
| `FromAcct`, `ToAcct`           | এই group-এর GL code range (e.g., 10000–19999 = Current Assets) |
| `PlItem`, `BsItem`             | P&L-এ যাবে কিনা, Balance Sheet-এ যাবে কিনা                     |
| `ParentId` → `AccGroup`        | **নিজে নিজের parent** — hierarchical tree (self-referencing)   |
| `ChartOfAccId` → `ChartOfAcc`  | কোন chart-এর under                                             |
|                                |                                                                |

**Self-reference-এর কারণ:** আপনি hierarchical structure বানাতে পারবেন:

```
Assets (parent)
├── Current Assets
│   ├── Cash
│   └── Bank
└── Fixed Assets
    └── Equipment
```

---

#### **GlMaster** ([GlMaster.cs:11](d:/Web Dev/VitalityCash/VitalityCash.Entities/GlMaster.cs#L11))

আসল ledger account। সব `JournalEntry` এই GL-এ post হয়।

|Field|অর্থ|
|---|---|
|`GlCode` (long)|Numeric code — `AccGroup.FromAcct ≤ GlCode ≤ AccGroup.ToAcct` মানতে হবে|
|`GlName`|যেমন "Cash in Hand", "AP - Local Supplier"|
|`AccGroupId` → `AccGroup`|কোন group-এর under|
|`CurrencyId` → `Currency`|এই GL কোন currency-তে maintain|
|`OpeningBalance`|বছরের শুরুর balance|
|`PlItem`, `BsItem`|Report routing|
|`PostAuto`|System auto-posting করবে কিনা|
|`ReconType`|Vendor / Customer / Asset — sub-ledger reconciliation type|
|`IsOpenItem`|Open-item (invoice-by-invoice tracking) কিনা|

**Relation chain:** `ChartOfAcc → AccGroup → GlMaster` (one-to-many দুই step)।

---

#### **ValuationClass** ([ValuationClass.cs:11](d:/Web Dev/VitalityCash/VitalityCash.Entities/ValuationClass.cs#L11))

Inventory item-কে accounting category-তে ভাগ করার tag। SAP-style।

|Field|অর্থ|
|---|---|
|`ValuationClassCode`|numeric (e.g., 3000)|
|`ValuationClassName`|"Raw Material", "Finished Goods", "Trading Goods"|

**Relation:** প্রতি `Product`-এর `ValuationClassId` থাকে ([Product.cs:37-38](d:/Web Dev/VitalityCash/VitalityCash.Entities/Product.cs#L37-L38))। মানে — একটা product জানে সে কোন valuation class-এ পড়ে।

---

#### **MaterialTransactionCode** ([MaterialTransactionCode.cs:11](d:/Web Dev/VitalityCash/VitalityCash.Entities/MaterialTransactionCode.cs#L11))

Inventory transaction-এর ধরন। SAP-style codes:

|Code|অর্থ|
|---|---|
|**BSX**|Inventory posting (stock account)|
|**WRX**|GR/IR clearing (Goods Receipt / Invoice Receipt mismatch account)|
|**PRD**|Price difference|
|**GBB**|Offsetting entry (issue/consumption)|

আপনার [GoodsReceiveService.cs:501](d:/Web Dev/VitalityCash/VitalityCash.Services/GoodsReceiveService.cs#L501) code-এ দেখা যাচ্ছে — GR save হলে BSX (Dr Inventory) + WRX (Cr GR/IR) auto-post হয়।

---

#### **ValuationClassTransaction** ([ValuationClassTransaction.cs:11](d:/Web Dev/VitalityCash/VitalityCash.Entities/ValuationClassTransaction.cs#L11)) ⭐ **সবচেয়ে গুরুত্বপূর্ণ mapping**

`ValuationClass` + `MaterialTransactionCode`-এর combination → কোন GL Dr / কোন GL Cr হবে — এটাই decide করে।

|Field|অর্থ|
|---|---|
|`ValuationClassId` → `ValuationClass`|কোন product-category|
|`TransactionCodeId` → `MaterialTransactionCode`|কোন ধরনের movement|
|`GlDrId` → `GlMaster`|Debit GL|
|`GlCrId` → `GlMaster`|Credit GL|
|`CompanyId` → `Company`|Multi-company scope|

**Example rows:**

|ValuationClass|TransactionCode|GlDr|GlCr|
|---|---|---|---|
|Raw Material|BSX|Raw Material Inventory|(null)|
|Raw Material|WRX|(null)|GR/IR Clearing|
|Finished Goods|BSX|FG Inventory|(null)|
|Finished Goods|GBB|COGS|(null)|

**System কীভাবে use করে:** GR save হলে → product-এর ValuationClass বের করে → BSX + WRX এর row লুকআপ করে → সেই Dr/Cr GL-এ JournalEntry তৈরি করে।

---

#### **GeneralSetting** ([GeneralSetting.cs:7](d:/Web Dev/VitalityCash/VitalityCash.Entities/GeneralSetting.cs#L7))

Generic key-value lookup table — অনেক kind-এর dropdown / config একই table-এ:

|Field|অর্থ|
|---|---|
|`ConfigKey`|"DocType", "Size", "Color", "MeasurementUnit", ইত্যাদি|
|`ConfigValue`|Actual value (e.g., "JV", "L", "Red", "Kg")|
|`StatusFlag`|Active / Inactive|

**যেখানে use হয়:**

- `JournalEntry.DocTypeId` → DocType list
- `GoodsReceiveProduct.SizeId / ColorId / MeasurementUnitId` → Size/Color/Unit lookup

**Pattern:** এটা single-table polymorphism — আলাদা আলাদা lookup table না বানিয়ে একটাই table-এ ConfigKey দিয়ে আলাদা করা। হালকা lookup-এর জন্য efficient, কিন্তু complex master-এর জন্য না।

---

### 🔹 LAYER 2 — BUSINESS TRANSACTION

#### **GoodsReceive** ([GoodsReceive.cs:12](d:/Web Dev/VitalityCash/VitalityCash.Entities/GoodsReceive.cs#L12)) — Master / Header

Supplier থেকে মাল আসলে এই entry হয়।

|Field|অর্থ|
|---|---|
|`GrNo`|Auto-generated GR number|
|`GrDate`, `DocDate`|তারিখ|
|`GrType`, `ItemType`|Type classification|
|`VendorId` → `CustomerVendor`|কোন vendor থেকে|
|`PurchaseOrderId` → `PurchaseOrder`|কোন PO-এর against|
|`CurrencyId`, `ConversionRate`|Foreign currency conversion|
|`MemoNo`, `DeliveryNote`, `BillOfLadding`|Reference docs|
|`Status`|enum: Draft / Posted / Reversed|
|`ParentId` → `GoodsReceive`|Reverse GR-এর জন্য (self-ref)|

#### **GoodsReceiveProduct** ([GoodsReceiveProduct.cs:10](d:/Web Dev/VitalityCash/VitalityCash.Entities/GoodsReceiveProduct.cs#L10)) — Child / Lines

প্রতি GR-এর under multiple product lines।

|Field|অর্থ|
|---|---|
|`GoodsReceiveId` → `GoodsReceive`|Parent FK|
|`ProductId` → `Product`|কোন item|
|`SizeId`, `ColorId`, `MeasurementUnitId` → `GeneralSetting`|Variant + unit (GeneralSetting use!)|
|`WarehouseId` → `Warehouse`|কোন warehouse-এ ঢুকবে|
|`BatchNo`, `BoxNo`, `ExpDate`|Batch tracking|
|`Quantity`, `UnitPrice`|কত পেলাম, কত দামে|
|`IsInvoiceRcv`|এই product-এর জন্য Invoice received হয়েছে কিনা|

**গুরুত্বপূর্ণ flag:** `IsInvoiceRcv` — পরে InvoiceReceive-এ 3-way match-এর জন্য।

---

#### **InvoiceReceive** ([InvoiceReceive.cs:12](d:/Web Dev/VitalityCash/VitalityCash.Entities/InvoiceReceive.cs#L12)) — Master / Header

Vendor invoice এলে এই entry — GR এর পরের step।

|Field|অর্থ|
|---|---|
|`IrNo`, `InvoiceDocNo`|Internal IR + Vendor's invoice no|
|`VendorId` → `CustomerVendor`|Vendor|
|`PurchaseOrderId` → `PurchaseOrder`|PO link|
|`GoodsReceiveId` → `GoodsReceive`|GR link (3-way match-এর জন্য)|
|`InvoiceAmount`, `CurrencyId`, `ConversionRate`|Amount|
|`Status`|enum|
|`IsPaid`|Payment হয়েছে কিনা|
|`InvoiceReceiveType`|Direct / Against PO ইত্যাদি|

**Children (তিন রকমের):**

#### **InvoiceReceivePO** ([InvoiceReceivePO.cs:11](d:/Web Dev/VitalityCash/VitalityCash.Entities/InvoiceReceivePO.cs#L11))

এক invoice একাধিক PO-এর against হতে পারে → many-to-many bridge।

```
InvoiceReceive ───┐                      ┌── PurchaseOrder
                  │                      │
                  └─< InvoiceReceivePO >─┘
```

#### **InvoiceReceiveGl** ([InvoiceReceiveGl.cs:8](d:/Web Dev/VitalityCash/VitalityCash.Entities/InvoiceReceiveGl.cs#L8))

এই invoice-এর জন্য GL-wise Dr/Cr distribution।

|Field|অর্থ|
|---|---|
|`InvoiceReceiveId` → `InvoiceReceive`|Parent|
|`GlMasterId` → `GlMaster`|কোন GL|
|`CostCenterId` → `CostCenter`|কোন cost center|
|`DcType`|Dr / Cr|
|`AmountInDocCurrency`, `AmountInLocalCurrency`|Amount in two currencies|

**ব্যবহার:** কোনো invoice direct expense-এ যাবে (e.g., consultancy, courier — PO ছাড়া) তখন `InvoiceReceiveGl` লাগে। PO-based হলে inventory GL আগেই GR-এ post হয়ে গেছে।

#### `InvoiceReceiveProduct` — product-level invoice line (PO + GR match-এর জন্য)

---

### 🔹 LAYER 3 — ACCOUNTING POSTING (Auto-Generated)

#### **MaterialTransaction** ([MaterialTransaction.cs:11](d:/Web Dev/VitalityCash/VitalityCash.Entities/MaterialTransaction.cs#L11)) — Master

প্রতি stock movement-এর single source of truth। GR / Issue / Transfer যেটাই হোক — এখানে এসে জমা হয়।

|Field|অর্থ|
|---|---|
|`TransactionNo`|Auto-generated|
|`PostingDate`, `DocDate`, `FiscalYear`, `Period`|কখন|
|`FormCode` (enum)|কোথা থেকে এসেছে — `GR`, `GI`, `Transfer`, ...|
|`MaterialTransactionRecordId`|Source entity-এর Id (e.g., GoodsReceive.Id)|
|`ParentId` → self|Reverse-এর জন্য|

#### **MaterialTransactionItem** ([MaterialTransactionItem.cs:10](d:/Web Dev/VitalityCash/VitalityCash.Entities/MaterialTransactionItem.cs#L10))

|Field|অর্থ|
|---|---|
|`MaterialTransactionId` → parent||
|`ProductId` → `Product`|কোন item|
|`WarehouseId` → `Warehouse`||
|`Quantity`, `UnitPrice`||
|`TransactionType`, `TransactionCode`|BSX / WRX / GBB string-form|
|`CostCenterId` → `CostCenter`||

**মূল কথা:** ItemStock-এর running balance এই table থেকেই compute হয়।

---

#### **JournalEntry** ([JournalEntry.cs:13](d:/Web Dev/VitalityCash/VitalityCash.Entities/JournalEntry.cs#L13)) — Master

সব accounting truth এখানে।

|Field|অর্থ|
|---|---|
|`JournalEntryNo`|Auto-generated|
|`PostingDate`, `DocDate`, `FiscalYear`, `Period`|কখন|
|`DocTypeId` → **GeneralSetting**|"JV", "PV", "RV" lookup (GeneralSetting দিয়ে!)|
|`CurrencyId`, `ConversionRate`|Multi-currency|
|`FormCode` (enum)|কোথা থেকে — `GR`, `IR`, ...|
|`MaterialTransactionRecordId`|Source entity Id|
|`ParentId` → self|Reverse JV|
|`Status` (enum)|Draft / Posted / Reversed|
|`JournalType` (enum)|Manual / Auto|
|`IsOpenItem`||

#### **JournalEntryItem** ([JournalEntryItem.cs:12](d:/Web Dev/VitalityCash/VitalityCash.Entities/JournalEntryItem.cs#L12))

| Field                                          | অর্থ                              |
| ---------------------------------------------- | --------------------------------- |
| **JournalEntryId** → parent                    |                                   |
| **GlMasterId** → GlMaster                      | কোন GL-এ post                     |
| `CostCenterId` → `CostCenter`                  |                                   |
| `DcType`                                       | "Dr" / "Cr"                       |
| `AmountInDocCurrency`, `AmountInLocalCurrency` |                                   |
| `CustomerVendorId` → `CustomerVendor`          | Sub-ledger party                  |
| `POReference`                                  | PO reference (open-item tracking) |
| `IsOpen`, `GlKnockOfDate`, `GlKnockOfNote`     | Open-item reconciliation tracking |
| `IsBankReconcile`, `ChequeRefNo`               | Bank reconciliation               |

---

## ৩. পুরো System কীভাবে Connect হয় — ER Diagram

```
                      ┌──────────────┐
                      │  ChartOfAcc  │
                      └──────┬───────┘
                             │ 1:N
                      ┌──────▼───────┐
                      │   AccGroup   │◄──┐ (self-ref ParentId)
                      └──────┬───────┘   │
                             │ 1:N       │
                      ┌──────▼───────┐   │
                      │   GlMaster   │   │
                      └──┬───┬───┬───┘   │
                         │   │   │       │
   ┌─────────────────────┘   │    └────────────────────────────┐
   │                         │                                 │
   │  ┌─────────────────┐    │   ┌──────────────────────┐      │
   │  │ ValuationClass  │    │   │ MaterialTransactnCode│      │
   │  └────────┬────────┘    │   └──────────┬───────────┘      │
   │           │             │              │                  │
   │           └─────────┬───┴──────────────┘                  │
   │                     ▼                                     │
   │       ┌──────────────────────────────┐                    │
   │       │  ValuationClassTransaction   │ (Dr GL + Cr GL)    │
   │       │  (mapping master)            │                    │
   │       └──────────────────────────────┘                    │
   │                                                           │
   │                                                           │
   │       ┌────────────────┐         ┌─────────────────┐      │
   │       │  GoodsReceive  │────────►│ GoodsReceiveProd│      │
   │       └────────┬───────┘  1:N    └─────────────────┘      │
   │                │                                          │
   │                │ auto-trigger (via Service)               │
   │                ▼                                          │
   │       ┌────────────────┐         ┌──────────────────┐     │
   │       │ InvoiceReceive │────┬───►│ InvoiceReceivePO │     │
   │       └────────┬───────┘    │    └──────────────────┘     │
   │                │            │    ┌──────────────────┐     │
   │                │            ├───►│ InvoiceReceiveGl │────-┤
   │                │            │    └──────────────────┘     │
   │                │            │    ┌────────────────────┐   │
   │                │            └───►│ InvoiceReceiveProd │   │
   │                │                 └────────────────────┘   │
   │                │                                          │
   │                │ auto-trigger                             │
   │                ▼                                          │
   │     ┌─────────────────────┐                               │
   │     │ MaterialTransaction │────► MaterialTransactionItem  │
   │     └─────────────────────┘                               │
   │                                                           │
   │     ┌─────────────────────┐                               │
   └────►│   JournalEntry      │────► JournalEntryItem ───────-┘
         └─────────┬───────────┘
                   │
                   ▼
            ┌─────────────────┐
            │ GeneralSetting  │ (DocType lookup)
            └─────────────────┘
```

---

## ৪. Live Example — GR Save করলে কী হয়

আপনার [GoodsReceiveService.cs:397-525](d:/Web Dev/VitalityCash/VitalityCash.Services/GoodsReceiveService.cs#L397-L525) থেকে বের করা actual flow:

**Scenario:** ১০টা "Cotton Fabric" (Raw Material valuation class) ১০০ টাকা/পিস দিয়ে কেনা হলো।

**Step 1 — User input (Layer 2)**

```
GoodsReceive { GrNo: "GR-001", Vendor: ABC, PO: PO-100 }
GoodsReceiveProduct { Product: "Cotton Fabric", Qty: 10, UnitPrice: 100 }
```

**Step 2 — Auto MaterialTransaction (Layer 3)**

```csharp
MaterialTransaction {
    TransactionNo: "MT-001",
    FormCode: GR,                          // GoodsReceive থেকে এসেছে
    MaterialTransactionRecordId: GR-001.Id // source link
}
MaterialTransactionItem { Product: Cotton, Qty: 10, TransactionCode: "BSX" }
```

এর ফলে `ItemStock`-এ Cotton Fabric-এর quantity +10 হয়।

**Step 3 — Auto JournalEntry (Layer 3)**

System এখন `ValuationClassTransaction` table check করে:

- Cotton Fabric → ValuationClass = "Raw Material"
- Transaction = BSX (inventory posting)
- লুকআপ → `GlDr = "Raw Material Inventory"` (GL #14000)
- লুকআপ → `GlCr = "GR/IR Clearing"` (GL #21000, via WRX)

```csharp
JournalEntry {
    JournalEntryNo: "JE-001",
    FormCode: GR,
    MaterialTransactionRecordId: GR-001.Id
}
JournalEntryItem { Gl: "Raw Material Inv (14000)",   Dc: Dr, Amount: 1000 }
JournalEntryItem { Gl: "GR/IR Clearing (21000)",     Dc: Cr, Amount: 1000 }
```

**Step 4 — InvoiceReceive আসলে (পরে, vendor invoice দিলে)**

```
InvoiceReceive { GoodsReceiveId: GR-001, InvoiceAmount: 1000, Vendor: ABC }
```

আবার auto-JournalEntry:

```
JournalEntryItem { Gl: "GR/IR Clearing (21000)",    Dc: Dr, Amount: 1000 }  ← knock-off
JournalEntryItem { Gl: "AP - Local Vendor (20100)", Dc: Cr, Amount: 1000 }
```

এতে GR/IR clearing account net 0 হয়ে যায় — এটাই **3-way matching**-এর hallmark।

---

## ৫. কেন এই Design — System Perspective

|Pattern|কেন আছে|
|---|---|
|**MaterialTransaction + JournalEntry আলাদা**|Stock movement audit আর accounting audit আলাদা track করা যায়। Stock recon-এ JE-তে যেতে হয় না।|
|**ValuationClassTransaction mapping table**|New product type যোগ হলে শুধু এই mapping-এ একটা row add করলেই auto-posting কাজ করবে — code change লাগে না।|
|**FormCode + MaterialTransactionRecordId**|প্রতি auto-generated JE / MT জানে সে কোথা থেকে এসেছে। Reverse / Audit / Drill-back সহজ।|
|**ParentId self-reference (সব master-এ)**|Reverse entry parent-কে চিনতে পারে। Audit trail অক্ষুণ্ণ থাকে।|
|**AccGroup.FromAcct–ToAcct range**|GL Code data-driven validation। New GL add করতে গেলে range check করেই AccGroup ঠিক assign হয়।|
|**GeneralSetting (key-value)**|DocType / Size / Color এর জন্য আলাদা table না বানিয়ে একটাই lookup. Drawback: large or complex master হলে এটা ভালো না।|
|**3-table InvoiceReceive children (PO/Gl/Product)**|এক invoice multiple PO + multiple non-PO GL expense + product lines — সব support করে। Real-world invoice complexity match করে।|
|**`AmountInDocCurrency` + `AmountInLocalCurrency`**|Multi-currency reporting + audit। Vendor USD-তে invoice দিলেও local books BDT-তে।|
|**`IsOpenItem`, `IsBankReconcile` flags**|Sub-ledger reconciliation (AR / AP / Bank) এর জন্য। প্রতি JE line আলাদাভাবে "still open" বা "knocked off" track হয়।|

---

## ৬. Quick Reference — কোথায় কী Data যায়

|User Action|যে Table-এ ঢোকে|
|---|---|
|GR save (Posted)|`GoodsReceive` + `GoodsReceiveProduct` + `MaterialTransaction` + `MaterialTransactionItem` + `JournalEntry` + `JournalEntryItem` (২ লাইন)|
|InvoiceReceive save (PO-based)|`InvoiceReceive` + `InvoiceReceivePO` + `InvoiceReceiveProduct` + `JournalEntry` (knock-off + AP)|
|InvoiceReceive save (Direct, non-PO)|`InvoiceReceive` + `InvoiceReceiveGl` + `JournalEntry` (expense + AP)|
|Manual JV|শুধু `JournalEntry` + `JournalEntryItem`|
|Reverse GR|নতুন `GoodsReceive` (Status=Reversed, ParentId=original) + নতুন `MaterialTransaction` (Dr/Cr flip) + নতুন `JournalEntry`|

---

## ৭. মনে রাখার ৫টা Golden Rule

1. **সব accounting truth `JournalEntry + JournalEntryItem`-এ।** P&L, BS, TB — সব এই দুই table-এর aggregation।
2. **সব stock truth `MaterialTransaction + MaterialTransactionItem`-এ।** ItemStock-এর running balance এখান থেকেই।
3. **`ValuationClassTransaction` mapping wrong হলে পুরো auto-posting wrong হবে।** Go-live এর আগে CFO sign-off নিন।
4. **`FormCode + MaterialTransactionRecordId` চেক করলে** যেকোনো auto-JV-র source খুঁজে বের করা যায়।
5. **`GlCode` অবশ্যই AccGroup-এর `FromAcct`–`ToAcct` range-এ পড়তে হবে** — না হলে save fail।

---

আপনার যদি কোনো specific entity/flow নিয়ে আরো deep dive লাগে (যেমন: 3-way matching exactly কীভাবে invoice-এ trigger হয়, বা Reverse JV-এর internal mechanism), বললে দেখাবো।

