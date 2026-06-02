# ACC + FIN — User Guide (Bangla)

## 1. কে কোন page ব্যবহার করবে

| Role | Primary screens |
|---|---|
| **Accountant** | JournalEntry → GlReconcileLedger → GL Ledger |
| **Senior Accountant** | JournalEntry (period-end JV) → TrialBalance |
| **Controller / CFO** | ProfitLoss, BalanceSheet, TrialBalance |
| **Admin / Setup** | AccGroup, GlMaster, GlMaster/ImportOB |
| **Inventory Manager** | SCM/ValuationClassTransaction (inventory→GL mapping) |

---

## 2. End-to-end Accounting flow — কোন order-এ কাজ করতে হবে

প্রথমবার system setup করার সময় এই sequence-এ যান। পরবর্তীতে শুধু **Step 5 (Journal Entry)** আর **Step 6 (Reports)** daily ব্যবহার হবে।

  
```

[Step 1] AccGroup  ──────►  [Step 2] GlMaster  ──────►  [Step 3] ImportOB (OB upload)

                                  │

                                  ▼

                     [Step 4] ValuationClassTransaction (SCM auto-JV mapping)

                                  │

                                  ▼

                          [Step 5] JournalEntry

                                  │

            ┌─────────────────────┼─────────────────────┐

            ▼                     ▼                     ▼

      GlReconcileLedger     ProfitLoss / BS     TrialBalance / GlLedger

            (recon)            (FIN reports)        (FIN reports)

```


---
  

## 3. Page-by-Page Journey

### 3.1 Account Group — `/ACC/AccGroup`
  
**কারা:** Admin / Senior Accountant · **কখন:** System setup এর শুরুতে · **Permission:** `Permissions.Conf.AccGroup`
  
Chart of Accounts-এর highest-level grouping। প্রতিটা GL Master কোন না কোন `AccGroup`-এ পড়বে।


| Field                 | কী দিবেন                                                       |
| --------------------- | -------------------------------------------------------------- |
| **AccGroupCode**      | Unique code (e.g., `1000`, `2000`) — remote-validated          |
| **AccGroupName**      | Display নাম (e.g., "Current Assets") — unique per company      |
| **Category**          | Asset / Liability / Revenue / Expense (P&L vs BS দিকনির্দেশনা) |
| **FromAcct / ToAcct** | Numeric range — এই range-এর মধ্যে GL Code পড়লেই auto-allowed  |
| **PlItem / BsItem**   | Checkbox — report-এ কোথায় দেখাবে                              |
| **ParentId**          | Optional — parent group (null হলে top-level)                   |

**গুরুত্বপূর্ণ:**

- `FromAcct ≤ ToAcct` validate হয় (server-side `GetRangeValidationByFromTo`)

- `CompanyId`-scoped — multi-company isolation

- Excel **Import** ও **Export** আছে (Category, `ParentCode`, `GroupCode`, `FromAcct`, `ToAcct` columns)

  
---

### 3.2 GL Master — `/ACC/GlMaster`

**কারা:** Admin / Senior Accountant · **কখন:** `AccGroup` setup-এর পরে · **Permission:** `Permissions.Conf.GL`

প্রকৃত ledger account। প্রতিটা `JournalEntry` line এই GL-এ post হবে।


| Field               | কী দিবেন                                                       |
| ------------------- | -------------------------------------------------------------- |
| **GlCode**          | Numeric — অবশ্যই AccGroup এর FromAcct–ToAcct range-এ পড়তে হবে |
| **GlName**          | Unique per company                                             |
| **AccGroupId**      | Dropdown — code-range সহ                                       |
| **CurrencyId**      | Default = parent currency                                      |
| **PlItem / BsItem** | Report routing                                                 |
| **IsOpenItem**      | Sub-ledger reconciliation প্রয়োজন কিনা                        |
| **PostAuto**        | System auto-post করবে কিনা                                     |
| **ReconType**       | Vendor / Customer / Asset — party-ledger style recon-এর জন্য   |
| **OpeningBalance**  | Optional — সাধারণত ImportOB দিয়ে set করবেন                    |

**Clone action** আছে — একই `AccGroup`-এ নতুন code/name দিয়ে duplicate তৈরি।

**Validation:** `IsGlCodeExists` + `IsGlNameExistsAsync` — uniqueness per company।


---

### 3.3 GL Master — Import Opening Balance — `/ACC/GlMaster/ImportOB`
**কারা:** Admin · **কখন:** Go-live বা new fiscal year setup · **Permission:** `Permissions.Conf.GL`

  
Bulk-upload OB via Excel।

  

**Steps:**

1. Page-এ "Download Sample Excel" — 3 column template: `GL Code | GL Name | Opening Balance`

2. Fill করে upload

3. System read করে — match GL Code with existing `GlMaster`

4. **Update condition:** শুধু তখনই update হবে যখন `existing.OpeningBalance IS NULL` AND `excel value IS NOT NULL`

5. Response message: `"{N} rows updated with opening balance, {M} rows skipped"`


**⚠ গুরুত্বপূর্ণ:**

- `ImportOB`(Import Opening Balance) **কোনো `JournalEntry` তৈরি করে না** — সরাসরি `GlMaster.OpeningBalance` column-এ লেখে

- Existing OB যদি set থাকে, সেটা **overwrite হবে না** (safety)। আগে clear করতে হবে।

- Code mismatch হলে row silently skip হবে — log file চেক করুন

---

### 3.4 Valuation Class Transaction — `/SCM/ValuationClassTransaction`

**কারা:** Inventory Manager + Accountant (joint setup) · **কখন:** Inventory module live করার আগে · **Permission:** `Permissions.Conf.ValuationClassTransaction`

এটা actual transaction নয় — এটা **mapping master**। SCM/IMS-এ যখন material movement (GRN, Issue, Revaluation) হয়, system এই mapping-এ লেগে দেখে কোন GL-এ `Dr/Cr` post করবে।

| Field | কী দিবেন |
|---|---|
| **ValuationClassId** | e.g., RawMaterial, FinishedGoods, ConsumableSpares — unique per company |
| **Transaction rows (multiple)** | প্রতি transaction type-এর জন্য একটা row |
| → TransactionCodeId | e.g., Purchase, Sale, IssueToProduction, Revaluation |
| → GlDrId | যে GL-এ Debit হবে |
| → GlCrId | যে GL-এ Credit হবে |

  

**Example mapping:**

  

| ValuationClass | TransactionCode | GL Dr | GL Cr |
|---|---|---|---|
| RawMaterial | Purchase (GRN) | RawMaterial Inventory | AP — Supplier |
| RawMaterial | Issue | WIP | RawMaterial Inventory |
| FinishedGoods | Sale | COGS | FG Inventory |


**Effect:** যখন GRN save হয়, SCM module এই mapping দেখে automatic JournalEntry তৈরি করে। তাই এই mapping wrong হলে **পুরো inventory GL ভুল direction-এ যাবে** — go-live এর আগে CFO sign-off নিন।

---

### 3.5 Journal Entry — `/ACC/JournalEntry`

  

**কারা:** Accountant (daily), Senior Accountant (period-end) · **Permission:** `Permissions.JE.LV` / `Permissions.JE.C`

  

Daily-use page। Manual Dr/Cr entry।

  

**Header fields:**

  

| Field                           | কী দিবেন                                           |
| ------------------------------- | -------------------------------------------------- |
| **DocDate**                     | Document-এর তারিখ                                  |
| **PostingDate**                 | এই date থেকে FiscalYear + Period auto-detect হবে   |
| **DocTypeId**                   | JournalDocType master থেকে                         |
| **Reference**                   | External reference (invoice no, contract id, etc.) |
| **HeaderText / Remarks**        | Narration                                          |
| **CurrencyId + ConversionRate** | Foreign currency হলে                               |
| **Attachments**                 | Multi-file upload — saved to `/uploads/je/`        |

  

**Line items (multiple, repeating):**

  

| Field | নোট |
|---|---|
| GlMasterId | কোন GL |
| DcType | Debit / Credit |
| AmountInDocCurrency | Doc currency-তে |
| AmountInLocalCurrency | Auto-calculate from rate |
| CostCenterId | Optional — for cost-center P&L |
| Narration | Per-line description |

  

**Save modes:**

- **Park (Draft)** — `Status=Draft`, saved to `JournalEntryTemp` table, approval workflow-এ যায়

- **Post (Final)** — `Status=Posted`, লেখা হয় `JournalEntry` + `JournalEntryItem` table-এ

  

**⚠ Validation:**

- **Balance check:** Posted JV-এর জন্য `Σ(Debit) − Σ(Credit) = 0` মাস্ট

- **Duplicate guard:** একই `FormCode + MaterialTransactionRecordId` থাকলে auto-reject

- **JournalEntryNo** auto-generate হয় `IGenerateNumberService` দিয়ে

  

**Reverse JV:** `ReverseJournalAsync` — Dr/Cr flip করে নতুন entry তৈরি করে, parent JV-এর `Status = Reversed` হয়, audit trail-এ `ParentId` থাকে।

  

---

  

### 3.6 GL Reconcile Ledger — `/ACC/JournalEntry/GlReconcileLedger`

  

**কারা:** Accountant · **কখন:** Daily / weekly party-ledger reconciliation · **Permission:** `Permissions.JE.LV`

  

একটা GL-এর পুরো transaction history + opening + running balance — sub-ledger reconciliation-এর জন্য।

  

| Filter | কী দিবেন |
|---|---|
| GlMasterId | একটা GL select |
| ReportType | Monthly / Yearly / Date |
| MonthFrom-MonthTo / Year / DateFrom-DateTo | Range |
| Currency + ConversionRate | BDT / USD report |

  

**Output columns:** Posting Date · Doc No · Reference · Narration · Offset Particulars · Doc Type · Debit · Credit · Balance

  

**Excel export** আছে। Calculation: `Opening + Σ(Dr) − Σ(Cr) = Closing`। `Opening` আসে `GlMaster.OpeningBalance` থেকে + filter date-এর আগের সব posted JV যোগ করে।

  

---

  

### 3.7 Trial Balance — `/FIN/FinancialReports/TrialBalance`

  

**কারা:** Senior Accountant / Controller · **কখন:** Month-end / period close · **Permission:** `Permissions.RPT.TrialBalance`

  

সব GL-এর Opening, Period Debit, Period Credit, Closing — এক view-এ।

  

| Filter | কী দিবেন |
|---|---|
| ReportType | Monthly / Yearly / Date |
| Date range | ReportType অনুযায়ী |

  

**Output columns:** Account Number · Ledger Name · Opening · Debit · Credit · Balance + grand-total row

  

**Self-check:** `Σ(Debit) = Σ(Credit)` হবেই — না হলে JournalEntry-তে কোথাও balance-violation আছে (manually inserted data বা migration issue)।

  

---

  

### 3.8 GL Ledger — `/FIN/FinancialReports/GlLedger`

  

**কারা:** Accountant / Auditor · **কখন:** Audit / drill-down · **Permission:** `Permissions.RPT.GlLedger`

  

GL Reconcile Ledger-এর মতই, কিন্তু **multiple GL একসাথে** dump করতে পারে।

  

| Filter | কী দিবেন |
|---|---|
| GlMasterId | Single GL · বা ↓ |
| GLMasterCodeFrom / GLMasterCodeTo | GL range |
| ReportType + Date filters | যথারীতি |
| Currency + ConversionRate | |

  

**Output:** প্রতিটা GL-এর জন্য আলাদা section — header (GL name) + opening + transactions table + closing।

  

---

  

### 3.9 Profit & Loss — `/FIN/FinancialReports/ProfitLoss`

  

**কারা:** Controller / CFO · **কখন:** Month-end · **Permission:** `Permissions.RPT.ProfitLoss`

  

`PlItem = true` flag-এর সব GL-এর running P&L।

  

| Filter | কী দিবেন |
|---|---|
| ReportType | Monthly / Yearly / Date |
| Date range | |
| Currency + ConversionRate | Multi-currency consolidation |

  

**Structure:**

- Sales / Revenue rows

- (−) COGS

- = **Gross Profit**

- (−) Overheads

- = **Operating Income**

- (−) Other Expenses

- = **Net Profit**

  

Multi-month report করলে column per-month দেখাবে, subtotal `IsSubTotal` flag দিয়ে।

  

**Note:** Grouping `AccGroup.Category` (Revenue / Expense / etc.) + GL-এর `SerialNo + GroupName` দিয়ে হয়। AccGroup-এ category ঠিক না থাকলে P&L-এ ভুল section-এ যাবে।

  

---

  

### 3.10 Balance Sheet — `/FIN/FinancialReports/BalanceSheet`

  

**কারা:** Controller / CFO · **কখন:** Period close · **Permission:** `Permissions.RPT.BalanceSheet`

  

`BsItem = true` flag-এর সব GL।

  

**Sections:**

- **ASSETS** — CurrentAssets + NonCurrentAssets (subtotal)

- **FINANCED BY** — ShareholdersEquity + AuthorizedCapital + CurrentLiabilities (subtotal)

- **Total Assets = Total Liabilities + Equity** — match না হলে data integrity issue

  

Filter: ReportType + date range (no currency conversion in this view)।

  

---

  

## 4. Daily Routine — Accountant

  

1. **সকাল:** `/ACC/JournalEntry` → previous day-এর সব Dr/Cr entry post

2. **দুপুর:** `/ACC/JournalEntry/GlReconcileLedger` → key GLs (Bank, AR, AP) verify

3. **বিকাল:** `/FIN/FinancialReports/TrialBalance` → balance check

4. **সপ্তাহান্তে:** Party-wise GL Reconcile Ledger export → email to AR/AP

  

---

  

## 5. Month-End Closing Checklist

  

- [ ] সব Draft JournalEntry post বা reject

- [ ] SCM থেকে auto-JV সব generate হয়েছে (ValuationClassTransaction mapping verify)

- [ ] `/FIN/FinancialReports/TrialBalance` — Dr = Cr match

- [ ] Bank Reconcile clear (`/FIN/Reconciliation`)

- [ ] `/FIN/FinancialReports/ProfitLoss` review

- [ ] `/FIN/FinancialReports/BalanceSheet` review → Total Assets = Total Liab + Equity

- [ ] CFO sign-off → Period Lock (`/FIN/FinDashboard` Period Close workflow)

  

---

  

## 6. Common Q&A

  

**Q: GL Code save করতে গিয়ে "out of range" error আসছে।**

A: GL Code অবশ্যই AccGroup এর `FromAcct–ToAcct` range-এর মধ্যে পড়তে হবে। AccGroup-এ range বাড়ান অথবা ভিন্ন AccGroup select করুন।

  

**Q: ImportOB-তে Excel upload করলাম কিন্তু row update হয়নি।**

A: তিনটা কারণ — (1) GL Code DB-তে নেই, (2) এই GL-এর `OpeningBalance` already set আছে (overwrite হবে না), (3) Excel cell empty। Response message-এ `skipped count` দেখুন; Serilog `GlMaster_<date>.log` চেক করুন।

  

**Q: Journal Entry save-এ "Balance mismatch" error।**

A: Posted JV-এর জন্য `Σ(Dr) = Σ(Cr)` মাস্ট। Line items-এ amount + DcType আবার মিলান। Draft-এ park করতে চাইলে এই rule shortcut করা যায়।

  

**Q: SCM-এ GRN save করলাম, কিন্তু GL-এ post হয়নি।**

A: `/SCM/ValuationClassTransaction`-এ ঐ ValuationClass + TransactionCode (Purchase) জোড়ার mapping আছে কিনা check করুন। Mapping না থাকলে auto-JV তৈরি হবে না — material movement হবে, কিন্তু GL untouched থাকবে।

  

**Q: Trial Balance balanced কিন্তু Balance Sheet mismatched।**

A: GL-এ `BsItem` flag সঠিকভাবে set আছে কিনা check করুন। কিছু GL একসাথে BsItem আর PlItem দুটোই হলে double-count হয়। AccGroup `Category` (Asset/Liab/Revenue/Expense) সঠিক হতে হবে।

  

**Q: P&L-এ একটা expense category দেখাচ্ছে না।**

A: GL-এর AccGroup-এ `Category = Expense` এবং `PlItem = true` দুটোই দরকার। তারপর ঐ GL-এ posted JV-line আছে কিনা GlLedger-এ verify করুন।

  

**Q: একটা ভুল JV post হয়ে গেছে — কী করব?**

A: Delete করবেন না (audit trail break হবে)। JournalEntry detail-এ **Reverse** action use করুন — system Dr/Cr flip করে নতুন entry তৈরি করবে, original-এর Status = `Reversed` হবে, `ParentId` link থাকবে।

  

**Q: GL Reconcile Ledger আর GL Ledger-এর পার্থক্য কী?**

A: GL Reconcile (`/ACC`) — single GL, party-recon focused, accountant দৈনিক use করে। GL Ledger (`/FIN`) — single বা range, audit/drill focused, auditor period-end-এ use করে। Data same, presentation আলাদা।

  

---

  

## 7. Related Pages

  

- [ACARE Patient Billing] — Patient invoice settle হলে auto-JV যায় `/ACC/JournalEntry`-তে

- [SCM GRN / Material Receipt] — Auto-JV via ValuationClassTransaction mapping

- [PHAR Sales / Purchase] — Drug movement → auto-JV (same mapping mechanism)

- [HRM Payroll] — Monthly payroll JV

- [AMS Depreciation] — Monthly depreciation JV

- [FIN Period Close (`/FIN/FinDashboard`)] — 8-step workflow which depends on all above

- [FIN Reconciliation (`/FIN/Reconciliation`)] — Bank reconciliation, may generate auto-JVs

  

---

  

## 8. Master Data Setup Order (Admin checklist)

  

| Step | Page | কেন এই order-এ |

|---|---|---|
| 1 | `/ACC/AccGroup` | GL Master-এর parent লাগবে |
| 2 | `/ACC/JournalDocType` | JV save করতে DocType লাগে |
| 3 | `/ACC/GlMaster` | Actual ledger accounts |
| 4 | `/ACC/GlMaster/ImportOB` | Go-live opening balances |
| 5 | `/SCM/ValuationClass` master | ValuationClassTransaction-এর FK |
| 6 | `/SCM/ValuationClassTransaction` | Inventory→GL auto-posting mapping |
| 7 | `/FIN/CostCenter` | Optional — cost-center P&L লাগলে |
| 8 | First test JV → Trial Balance verify | Sanity check |

  

---

  

## 9. Demo / Print Checklist

  

- [ ] AccGroup create → GlMaster create → ImportOB sample Excel দিয়ে OB load

- [ ] ValuationClassTransaction-এ একটা mapping (e.g., RawMaterial Purchase) add

- [ ] JournalEntry — 2-line balanced JV post (Cash Dr, Sales Cr)

- [ ] GlReconcileLedger — Cash GL select → transaction line দেখা যাচ্ছে

- [ ] TrialBalance — Σ(Dr) = Σ(Cr) match

- [ ] ProfitLoss — Sales line দেখা যাচ্ছে

- [ ] BalanceSheet — Cash asset side, Equity financed side

- [ ] GlLedger — same Cash GL → same data, different layout

  

---

  

## 10. Storage / Tables Quick Reference

  

| Page | Main Table | Linked Tables | Direct GL Posting? |
|---|---|---|---|
| AccGroup | `AccGroup` | GlMaster (FK) | না |
| GlMaster | `GlMaster` | AccGroup (FK), JournalEntryItem | না (header only) |
| ImportOB | `GlMaster` (update OpeningBalance column) | — | না (no JV) |
| JournalEntry (Posted) | `JournalEntry` + `JournalEntryItem` | GlMaster, CostCenter | হ্যাঁ |
| JournalEntry (Draft) | `JournalEntryTemp` | — | না (until promoted) |
| ValuationClassTransaction | `ValuationClassTransaction` | MaterialTransactionCode, GlMaster | হ্যাঁ (trigger via SCM movement) |
|GlReconcileLedger / GlLedger / P&L / BS / TB | (query only) | JournalEntryItem, GlMaster, AccGroup | না (read-only reports) |

  

**Key insight:** সব accounting truth `JournalEntry + JournalEntryItem`-এ আছে। বাকি সব report এই দুই table-এর aggregation। তাই data integrity issue হলে JournalEntry-তে যান।