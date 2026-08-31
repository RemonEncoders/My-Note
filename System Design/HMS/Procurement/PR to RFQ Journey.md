# RFQ পুনর্নির্মাণ — Database Mind Map ও সিদ্ধান্ত-ট্রেস

## ১. এক নজরে সমাধান-পথ

```
                    ┌──── Scenario 2 (আগের মতোই, অপরিবর্তিত) ────┐
                    │                                            ▼
PR (Approved) ──────┤                                    PO (category-pure) ──► GRN ──► Invoice
                    │                                            ▲
                    └── Scenario 1: Float RFQ ──► RFQ/Tender ────┘
                                                 (Award → Create PO)
```

এক PR থেকে একাধিক PO ও একাধিক RFQ হতে পারে — remaining-quantity হিসাব আগে থেকেই আছে।

## ২. সিদ্ধান্ত-ম্যাট্রিক্স — কোন choice নকশাকে কোন দিকে নিল

| #       | প্রশ্ন                         | হাতে থাকা options                                             | ✓ আপনার choice        | ⇒ ফলে design এই দিকে গেল                                                                                                                | অবস্থা      |
| ------- | ------------------------------ | ------------------------------------------------------------- | --------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| **D1**  | PR-এর দুই status desync        | ① এখনই merge ② auto-Final ③ filter শিথিল                      | **এখনই merge**        | একটাই lifecycle enum + data migration; approve হলেই PR PO-screen-এ। RFQ-ও এক-enum দর্শনে                                                | ✅ merged    |
| **D2**  | Orphan PO?                     | ① PR বাধ্যতামূলক ② emergency পথ                               | **PR বাধ্যতামূলক**    | সব procurement-এর শিকড় PR ⇒ **RFQ-তেও `PurchaseRequisitionId` required FK**                                                            | ✅ merged    |
| **D3**  | Fulfillment কোথায়             | ① stored roll-up ② computed                                   | **computed-only**     | নতুন stored state নেই, bulk query। RFQ dashboard-ও একই নীতিতে (D8)                                                                      | ✅ merged    |
| **D4**  | 3-way price match              | ① tolerance+block ② warning ③ যেমন আছে                        | **যেমন আছে**          | Invoice-এ দাম-যাচাই যোগ হয়নি; RFQ-র দাম PO-তে শুধু pre-fill                                                                            | ✅ সিদ্ধান্ত |
| **D5**  | Winner fraud হলে re-award কাকে | ① শুধু participants ② যে কেউ                                  | **শুধু participants** | `IsDisqualified + Reason` field; award-validation: winner অবশ্যই participant; বাইরের লোক লাগলে RFQ cancel → নতুন float                  | 🔜 RFQ      |
| **D6**  | Fraud vendor ভবিষ্যতেও block?  | ① blacklist flag ② শুধু ওই RFQ-তে                             | **blacklist flag**    | vendor master-এ `IsBlacklisted + BlacklistReason`; RFQ invite/award/নতুন PO থেকে বাদ                                                    | 🔜 RFQ      |
| **D7**  | এক PO = এক category            | ① server guard + PR ভেঙে বহু PO ② শুধু client filter (এখনকার) | **server guard**      | PO save-এ category-purity validation (schema বদল নেই); RFQ bridge category-প্রতি আলাদা PO বানাবে                                        | 🔜 ধাপ A    |
| **D8**  | Tender কখন Closed              | ① derived (ClosingDate) ② stored + job ③ manual বাটন          | **derived**           | `ClosingDate`-ই একমাত্র সত্য — আলাদা ClosedDate field **নেই** (D1-এর শিক্ষা); deadline-এর পর bid entry server-এ reject; job লাগে না     | 🔜 RFQ      |
| **D9**  | Deadline extension?            | ① হ্যাঁ, log সহ ② না, চূড়ান্ত                                | **হ্যাঁ, log সহ**     | log-এ `DeadlineExtended` + পুরনো/নতুন তারিখ; শুধু Published-এ, শুধু সামনের দিকে                                                         | 🔜 RFQ      |
| **D10** | Evaluation আলাদা ধাপ?          | ① explicit বাটন ② সরাসরি award                                | **explicit বাটন**     | enum-এ `UnderEvaluation` টিকল; কে কবে মূল্যায়ন শুরু করল তা logged                                                                      | 🔜 RFQ      |
| **D11** | Index dashboard                | ① সব আসল query ② mock রাখা                                    | **সব আসল**            | ২টা নতুন field-এর জন্ম: `EstimatedValue` (PR snapshot → Avg Savings KPI) + `AwardedDate`; vendor-এর দাম **typed decimal** হতে বাধ্য হলো | 🔜 RFQ      |
| **D12** | Winner কে ঠিক করে              | ① মানুষ ② auto-L1                                             | **মানুষ**             | comparison matrix = সিদ্ধান্ত-সহায়ক মাত্র; Award = logged human event (`RFQAwardLog`)                                                  | 🔜 RFQ      |

## ৩. Database Mind Map

```
                         scm_purchase_requisitions (অপরিবর্তিত)
                          │ 1                        │ 1
              (D2 required FK)              (Scenario 2 — আগের মতোই)
                          │ *                        │ *
                 ┌─── scm_rfq_comparisons ───┐    scm_purchase_orders
                 │        │         │        │       ▲ + RFQComparisonId (নতুন, nullable — Phase C)
                 │ *      │ *       │ *      └───────┘
        scm_rfq_items  scm_rfq_  scm_rfq_award_logs
                 │     vendor_responses
                 │        │ *  │ *
     scm_products (FK)    │    └── scm_rfq_custom_attributes (EAV matrix)
                          │
              scm_customer_vendors ◄── + IsBlacklisted, BlacklistReason (নতুন — D6)
```

- 🟢 **নতুন ৫ টেবিল:** `scm_rfq_comparisons`, `scm_rfq_items`, `scm_rfq_vendor_responses`, `scm_rfq_custom_attributes`, `scm_rfq_award_logs`
- 🟠 **বিদ্যমানে additive ছোঁয়া:** vendor master-এ ২ কলাম, PO-তে ১ nullable কলাম
- ⚪ **একদম অপরিবর্তিত:** PR, GRN, Invoice

## ৪. Field-স্তরের ট্রেস — কোন কলামের জন্ম কোন সিদ্ধান্তে

**`scm_rfq_comparisons`** (tender-এর মূল document):

|Field|উৎস|কেন|
|---|---|---|
|`PurchaseRequisitionId` (required FK)|D2|free-text reference বাদ|
|`Status` (Draft→Published→UnderEvaluation→Awarded \| Cancelled)|D8, D10|**"Closed" enum-এ নেই — derived**|
|`ClosingDate`|D8|Closed-এর একমাত্র সত্য; এর পর bid reject|
|`EstimatedValue`|D11|PR-এর TotalPrice snapshot → Avg Savings KPI|
|`AwardedDate` (nullable)|D11, D12|"Awarded This Month" KPI; disqualify-তে মুছে যায়|
|`RowVersion`|PR-এর B4 প্যাটার্ন|একসাথে দুই award-এর race আটকায়|

**`scm_rfq_vendor_responses`** (প্রতি vendor-এর bid):

|Field|উৎস|কেন|
|---|---|---|
|`VendorId` (FK → vendor master)|D6|hard-coded নাম/fake SAP code বাদ; blacklist check সম্ভব|
|`QuotedPrice, Vat, Ait, Discount, TotalPrice` (typed decimal)|D11|EAV string-এ savings গণনা অসম্ভব|
|`IsShortlisted`, `IsWinner`|D12|মানুষের বাছাই|
|`IsDisqualified` + `DisqualificationReason`|D5|মামলা/fraud-এ বাদ; re-award এই টেবিলের ভেতরেই|

**`scm_rfq_award_logs`** (অপরিবর্তনীয় audit trail):

|Field|উৎস|কেন|
|---|---|---|
|`Action`: Published / DeadlineExtended / EvaluationStarted / Awarded / Disqualified / ReAwarded / Cancelled|D9, D10, D12|প্রতিটা ঘটনা এক সারি — তদন্তে পুরো গল্প|
|`OldClosingDate` / `NewClosingDate`|D9|extension-এর আগে-পরে প্রমাণ|
|`Reason`, `ActedById`, `ActionDate`|D5, D12|কে-কখন-কেন; disqualify-তে Reason বাধ্যতামূলক|

**`scm_rfq_items`**: `ProductId, Quantity, MeasurementUnitId` ← PR item snapshot (D2); bridge এই ProductId-র ItemType দেখেই category-প্রতি PO ভাগ করবে (D7)

**`scm_rfq_custom_attributes`** (EAV): warranty/lead-time-এর মতো নমনীয় criteria — পুরনো UI-র শক্তিটা টিকে গেল; **টাকার অঙ্ক বাদে** (ওগুলো typed — D11)

**বিদ্যমান টেবিলে:** vendor master +`IsBlacklisted`+`BlacklistReason` (D6); PO +`RFQComparisonId` nullable (Phase C — nullable বলে Scenario 2-তে শূন্য প্রভাব)

## ৫. Lifecycle — Closed ও Awarded-এর tracking

```
Draft ──[Publish, log]──► Published ──(ClosingDate পেরোলো — স্বয়ংক্রিয়, D8)──► Closed-for-bids*
            │                 │ [Extend deadline — D9, log]                        │
        [Cancel]          [Cancel]                                    [Start Evaluation — D10, log]
            │                 │                                                    ▼
            ▼                 ▼                                            UnderEvaluation ──[Cancel]──► Cancelled
         Cancelled        Cancelled                                                │
                                                       [Award participant — D5/D12, log + AwardedDate]
                                                                                   ▼
                                          [Disqualify + কারণ — D5, log] ◄────── Awarded ──[Create PO — Phase C]──► LOCKED
                                                    (ফেরে UnderEvaluation-এ, re-award)
```

* Closed-for-bids stored status নয় — derived; `ClosingDate`-ই timestamp; ওই মুহূর্ত থেকে server bid entry reject করে।

## ৬. অক্ষত থাকার গ্যারান্টি

- **Scenario 2 হুবহু আগের মতো** — PR→PO সরাসরি পথের এক লাইনও বদলাবে না
- **Schema-তে শুধু additive** — নতুন টেবিল + nullable/flag কলাম; কোনো data rewrite নেই
- **পুরনো প্রোটোটাইপে হারানোর কিছু নেই** — ডেটা static list-এ ছিল, DB-তে শূন্য
- **প্রতি ধাপ আলাদা commit + build যাচাই** — সমস্যায় শুধু সেই commit revert

## ৭. বাস্তবায়ন-ক্রম

|ধাপ|বিষয়বস্তু|কোন সিদ্ধান্ত cover করে|
|---|---|---|
|**A**|PO category-purity server guard (schema বদল নেই)|D7|
|**B** (Phase 1)|৫ টেবিল + migration + service + PR→RFQ + comparison + award/disqualify state machine + blacklist + Open/Closed/Awarded dashboard|D2, D5, D6, D8–D12|
|**C** (Phase 2)|Awarded RFQ → "Create PO": winner+দাম pre-fill, category-প্রতি আলাদা PO, PO-তে nullable FK, award lock|D7, D12|

---

লক্ষ করার মতো একটা প্যাটার্ন: আপনার সিদ্ধান্তগুলো একটা **সুসংগত দর্শন** মেনে চলছে — D3 আর D8 একই নীতি (_"যা গণনা করা যায় তা জমা রেখো না"_), আর D1-এর dual-status শিক্ষাই D8-এ আলাদা ClosedDate field না রাখার কারণ।

কোনো কার্ডে দ্বিমত থাকলে বলুন — নয়তো "proceed" বললেই **ধাপ A** থেকে বাস্তবায়ন শুরু করব।