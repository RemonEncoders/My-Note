# Supply Chain (SCM) — Procurement গাইড

**কার জন্য?** Department Requester (যে PR বানান), Approver/Manager, Procurement Officer, Store/Warehouse In-charge।

**কী কী আছে:** একটা জিনিস দরকার হওয়া থেকে শুরু করে Purchase Requisition, Approval, (দরকার হলে) RFQ/Tender, Purchase Order, আর শেষে Goods Receipt পর্যন্ত পুরো procurement chain।

> ℹ️ Goods Receipt হলে system নিজে থেকেই একটা Journal Entry বানিয়ে Accounting-এ পাঠিয়ে দেয় (Inventory GL debit, GR/IR Clearing GL credit) — সেটা Accountant-কে review করে Post করতে হয়। দেখুন **[Accounting](../Accounting/bn.md)** গাইড।

> 🖼️ *[Screenshot: SCM procurement dashboard]*

---

## পুরো chain-টা একনজরে

```
Purchase Requisition (PR)  →  [ঐচ্ছিক] RFQ/Tender  →  Purchase Order (PO)  →  Goods Receipt (GRN)
   Draft → Approval →           Publish → Evaluate →      Direct অথবা            Store-এ মাল ঢোকে,
   Approved                     Award                     Tender থেকে             GL-এ auto-post
```

RFQ/Tender ধাপটা **ঐচ্ছিক** — শুধু competitive bidding দরকার হলে ব্যবহার হয়। ছোট/দ্রুত কেনাকাটায় PR থেকে সরাসরি Direct PO বানানো যায়।

---

## কাজ ১ — Purchase Requisition (PR) বানানো

**Menu:** Supply Chain → **Purchase Requisition** → **Add**  ·  `/SCM/PurchaseRequisition`

1. দরকারি item (product), quantity, expected currency/price দিয়ে PR-এর line ভরুন।
2. **Save** করলে PR **Draft** অবস্থায় থাকে — Draft-এ থাকা অবস্থায় যত খুশি edit করা যায়।
3. তৈরি হয়ে গেলে **Submit for Approval** করুন (কাজ ২)।

---

## কাজ ২ — Approval-এর জন্য পাঠানো

**Submit for Approval** click করলে system PR-এর **Currency** আর **Total Price**-এর সাথে মিলিয়ে একটা **Approval Policy** খোঁজে।

- মিলে যাওয়া policy না থাকলে Submit আটকে যাবে — Admin-কে policy বানাতে বলুন।
- মিলে গেলে, ওই policy-র সাথে বাঁধা **Approval Workflow**-টা PR-এর গায়ে "snapshot" হয়ে বসে যায় — পরে কেউ policy বদলালেও এই PR-এর approval ধাপ বদলাবে না।
- Status হয় **Pending Approval**, আর ধাপে ধাপে (Stage 1, 2, 3...) সেই stage-এর জন্য নির্ধারিত Role-এর কারো approve করার অপেক্ষায় থাকে।

> ℹ️ PR যিনি বানিয়েছেন, নিজের PR নিজে approve করতে পারেন যদি তার সেই Role থাকে (self-approval অনুমোদিত)।

---

## কাজ ৩ — PR Approve / Reject করা

**Menu:** Supply Chain → **Pending PR Approvals**  ·  `/SCM/PurchaseRequisition/PendingApprovals`

1. আপনার Role-এর জন্য অপেক্ষমান PR-গুলোর list দেখবেন।
2. **Approve** করলে পরের stage-এ যায়; শেষ stage হলে PR **Approved** হয়ে যায় — তখনই RFQ/PO বানানো যায়।
3. **Reject** করলে reason দিতে হয়; PR-এর মালিক এটা edit করে আবার নতুন করে submit করতে পারেন (নতুন round, stage ১ থেকে আবার শুরু)।

সব approve/reject-এর record **Purchase Approval** (audit trail)-এ থেকে যায় — `/SCM/PurchaseApproval`।

### PR status

| Status | মানে |
|--------|------|
| **Draft** | তৈরি হচ্ছে, স্বাধীনভাবে edit করা যায়। |
| **Pending Approval** | Approval workflow চলছে। |
| **Approved** | সব stage pass; RFQ/PO বানানোর জন্য প্রস্তুত। |
| **Rejected** | কোনো stage-এ বাতিল; edit করে resubmit করা যায়। |
| **Closed** | পুরো quantity PO হয়ে গেছে বা manually বন্ধ করা হয়েছে। |
| **Cancelled** | পুরোপুরি বাতিল। |

---

## কাজ ৪ — Vendor ঠিক করা (blacklist সহ)

**Menu:** Supply Chain → **Vendor**  ·  `/SCM/Vendor`

1. **Add/Edit** করে vendor-এর তথ্য রাখুন; **Active/Inactive** টগল করা যায়।
2. কোনো vendor-এর সাথে সমস্যা হলে **Set Blacklist** করে reason দিন — blacklisted vendor দিয়ে *নতুন* কোনো RFQ award বা PO করা যাবে না, তবে তার আগের চলমান PO ঠিকই edit করা যাবে।

> ℹ️ পুরনো একটা **Supplier** নামের আলাদা master আছে — সেটা legacy, এখন সব কাজের জন্য **Vendor** ব্যবহার করুন।

---

## কাজ ৫ — RFQ/Tender float করা (competitive bidding দরকার হলে)

**Menu:** Approved PR-এর Details পেজ থেকে **Float RFQ** (`/SCM/RFQComparison/Add?prId=...`) — PR-এর item ও তার তখনকার live allocation নিজে থেকেই load হয়ে যায়।

1. যেসব vendor-কে quote দিতে বলবেন, তাদের select করুন।
2. **Publish** করুন — এখন RFQ vendor-দের quote জমা দেওয়ার জন্য খোলা।
3. **Closing Date**-এর আগে দরকার হলে **Extend Deadline** করা যায় (reason সহ)।
4. Vendor-দের quote জমা পড়া শুরু হলে **Start Evaluation** করুন — status **Under Evaluation** হয়।
5. ভালো লাগা quote **Shortlist** (toggle) করে রাখুন তুলনা করার জন্য।

---

## কাজ ৬ — Tender Award করা

RFQ-এর Details পেজ থেকে:

1. **Award Tender** — এটা **single-winner**: পুরো tender একটা vendor-কে দেওয়া হয়। শর্ত — ওই vendor-কে *প্রতিটা* item-এ quote দিতে হবে, blacklisted/disqualified থাকা যাবে না, আর কোনো item আগে থেকে অন্য PO-তে lock থাকা যাবে না।
2. ভুল হলে **Clear Tender Award** দিয়ে award পুরোপুরি সরানো যায়।
3. কোনো vendor পরে disqualify করতে হলে **Disqualify Vendor** — তার জেতা সব item আবার pool-এ ফিরে যায়।
4. কোনো bid-ই সন্তোষজনক না হলে **Close Without Award**, বা পুরো RFQ-টাই **Cancel** করুন। এই দুটোই চূড়ান্ত — একই RFQ আবার খোলা যায় না, নতুন RFQ বানাতে হবে।

> ℹ️ Item-ভিত্তিক আলাদা award (একই tender-এর ভিন্ন item ভিন্ন vendor-কে) আগে সম্ভব ছিল আর এখনো data-তে টিকে আছে, কিন্তু বর্তমান UI-তে **single-winner (পুরো tender একজনকে)** পদ্ধতিই ব্যবহার হয়।

### RFQ/Tender status

| Status | মানে |
|--------|------|
| **Draft** | তৈরি হচ্ছে, এখনো vendor-দের কাছে যায়নি। |
| **Published** | Vendor-রা quote দিতে পারছে। |
| **Under Evaluation** | Quote জমা পড়েছে, তুলনা চলছে। |
| **Awarded / Partially Awarded** | একজন (বা কিছু) vendor-কে জেতানো হয়েছে। |
| **Cancelled / Closed Without Award** | চূড়ান্তভাবে বন্ধ, পুনরায় খোলা যায় না। |

---

## কাজ ৭ — Purchase Order (PO) বানানো — দুই রাস্তা

**Direct PO** (RFQ ছাড়াই সরাসরি) — **Menu:** Supply Chain → **Purchase Order** → **Add**

1. PR বেছে নিয়ে তার এখনো-বাকি (remaining) item pick করুন, Supplier/Vendor বেছে নিন।
2. System check করে: PR approved কিনা, vendor blacklisted কিনা, আর PO-র সব item একই category-র (সব Goods অথবা সব Service — মিশ্রণ চলবে না)।

**Tender-sourced PO** (RFQ award থেকে) — RFQ Details-এর **Create PO** বাটন থেকে, অথবা worklist **Create From RFQ** (`/SCM/PurchaseOrder/CreateFromRFQ`) থেকে — এখানে জেতা কিন্তু এখনো PO না হওয়া tender-গুলো vendor-ভিত্তিক গ্রুপ করা থাকে।

1. Vendor আর tender এখানে award অনুযায়ী **lock করা** — বদলানো যায় না।
2. Item-গুলো সেই winning bid-এর দামেই আসে, শুধু যেগুলোর RFQ Remaining এখনো আছে সেগুলোই দেখাবে।

> 💡 PO **Draft** অবস্থায় যতগুলো খুশি বানিয়ে রাখতে পারেন — এতে PR/RFQ-এর quantity আটকায় না। Quantity সত্যিকারের **reserve/lock** হয় শুধু PO **Final** করার সময়।

---

## কাজ ৮ — PO Final করা, Cancel/Void, Reopen

1. Draft PO ঠিকঠাক থাকলে **Final** করুন — এই মুহূর্তে PR/RFQ-এর against quantity reserve হয় (over-allocation হলে error দেখাবে)।
2. আর দরকার না হলে **Cancel PO** (বা **Void**) করুন — reserve করা quantity আবার মুক্ত হয়ে যায়।
3. Final PO-তে ভুল ঠিক করতে হলে **Reopen** করুন — আবার Draft-এ ফিরে যায়, edit করে আবার Final করতে হবে।

### PO status

| Status | মানে |
|--------|------|
| **Draft** | তৈরি হচ্ছে, PR/RFQ quantity এখনো আটকায়নি। |
| **Final** | Confirmed; quantity reserve হয়েছে; Goods Receipt নেওয়া যায়। |
| **Completed** | সব line পুরোপুরি received — system নিজে থেকেই এই status দেয়। |
| **Cancelled / Void** | বাতিল; reserve করা quantity মুক্ত। |

---

## কাজ ৯ — Goods Receipt (মাল বুঝে নেওয়া)

**Menu:** Supply Chain → **Goods Receive**

1. Final হওয়া PO বেছে নিন, যা যা এসেছে তার quantity ঢুকান।
2. System over-receipt আটকে দেয় — কোনো PO-line-এর মোট received quantity তার order করা quantity-র বেশি হতে পারবে না (অন্য কোনো live receipt-এ ইতিমধ্যে claim করা quantity বাদ দিয়ে হিসাব হয়)।
3. Receipt **Final** করলে সেই মুহূর্তেই একটা GL entry (Draft) auto-generate হয়ে Accounting-এ চলে যায়।
4. PO-এর সব line পুরোপুরি received হয়ে গেলে PO নিজে থেকেই **Completed** হয়ে যায়; কোনো receipt পরে reverse হলে PO আবার খুলে যায়।

> 🖼️ *[Screenshot: Goods Receipt entry against a Final PO]*

---

## কাজ ১০ — "Remaining" মানে কী (allocation বোঝা)

PR/RFQ-এর screen-এ **Requested / Ordered / Open RFQ / Remaining** কলাম দেখবেন — বিভ্রান্ত হবেন না, নিয়মটা সহজ:

| কলাম | মানে |
|-------|------|
| **Requested** | PR-এ মোট চাওয়া quantity। |
| **Ordered** | ইতিমধ্যে **Final/Completed** PO-তে যত quantity গেছে (Draft PO গোনায় ধরা হয় না)। |
| **Open RFQ** | কোনো live (Published/Under Evaluation/Awarded) tender-এ আটকে থাকা quantity, যেটা এখনো PO হয়নি। |
| **Remaining** | Requested − Ordered − Open RFQ — এটাই নতুন RFQ/PO-তে এখনো নেওয়া যায় এমন quantity। |

PO **Final** করার মুহূর্তে system আবার এই হিসাব যাচাই করে — দুইজন একসাথে একই quantity নিয়ে নিলে যেটা আগে Final হয় সেটাই জেতে, অন্যটা error পাবে।

---

## Handover

Goods Receipt হয়ে গেলে ওই GL entry Accounting-এ **Draft** অবস্থায় বসে থাকে, review-এর জন্য।

➡️ Entry review ও Post করতে **[Accounting](../Accounting/bn.md)** গাইডে যান।

> ✅ এতেই PR থেকে Goods Receipt পর্যন্ত পুরো procurement chain কভার হলো।
