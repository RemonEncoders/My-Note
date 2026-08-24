নিশ্চিত হলাম — আমার প্রস্তাবিত স্থাপত্যের ভিত্তিটা কোডে ইতিমধ্যেই আছে। এবার আপনার ডিজাইন-প্রশ্নের সরাসরি উত্তর দিই।

---

# আমার মত: Option B — তবে "stored dashboard" নয়, **computed allocation service**

সংক্ষেপে: **আপনার Option B-ই সঠিক দিক**, কিন্তু আপনার লেখা নিয়মে একটা গুরুত্বপূর্ণ ফাঁক আছে যেটা ঠিক না করলে এন্টারপ্রাইজে আবার over-allocation হবে। আর dashboard-টা আলাদা টেবিলে **store করা যাবে না** — compute করতে হবে (এটা আপনার নিজের D1/D3/D8 দর্শনেরই ধারাবাহিকতা: "যা গণনা করা যায় তা জমা রেখো না")।

## ১. যে ফাঁকটা আপনার নিয়মে এখনো আছে (সবচেয়ে জরুরি)

আপনার নিয়ম: `Σ(active RFQ qty) ≤ PR qty`। কিন্তু quantity **দুইটা চ্যানেল** দিয়ে PR থেকে বের হয়:

```text title:"Scenario of PR to RFQ / PO

                 ┌── RFQ চ্যানেল (solicitation) ────┐
PR (10) ─────────┤                                 ├──► একই ইউনিট
                 └── PO চ্যানেল (commitment) ───────┘

```

দুটো সমস্যা:

**(ক) Direct PR→PO (Scenario 2) উপেক্ষিত।** ধরুন PR=10, সরাসরি একটা PO হলো 5 laptop (RFQ ছাড়া)। আপনার নিয়ম শুধু RFQ যোগ করে → RFQ allocated 2 ≤ 10 pass → নতুন RFQ-তে আরও 8 দেওয়া যাবে। অথচ আসলে বাকি মাত্র **3** (5 তো PO হয়ে গেছে)। আবার over-allocation।

**(খ) RFQ→PO রূপান্তরে double-count।** RFQ-1-এর 2 laptop award হয়ে PO হলে — সেই 2 কি RFQ চ্যানেলে গোনা হবে, নাকি PO চ্যানেলে? দুই জায়গায় গুনলে 2+2=4 ধরা পড়বে, ভুল।

## ২. সঠিক ERP নিয়ম (double-count-মুক্ত)

আপনার কোডে ইতিমধ্যে `RFQItem.PurchaseOrderId` আছে ([RFQItem.cs:36](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Entities/Areas/SCM/RFQItem.cs#L36)) — PO হয়ে গেলে এই lock সেট হয়। এটাই চাবি। প্রতি product-এ:

```
Committed = PO_Ordered                        // সব PO (direct + RFQ-উৎস), বিদ্যমান CalculateOrderedQuantities দিয়ে
          + Open_RFQ                           // Σ RFQItem.Quantity  যেখানে  tender ≠ Cancelled  এবং  PurchaseOrderId IS NULL

Remaining = RequestedQty − Committed
নতুন RFQ বৈধ যদি:  newQty ≤ Remaining
```

কেন নিখুঁত:

- RFQ item **PO হয়ে গেলে** → `PurchaseOrderId` সেট → RFQ চ্যানেল থেকে বাদ, PO চ্যানেলে গোনা (কারণ RFQ-উৎস PO-তেও `PurchaseRequisitionId` থাকে)। **কোনো double-count নেই।**
- RFQ item **এখনো PO হয়নি** (Draft/Published/Awaited-award) → শুধু RFQ চ্যানেলে গোনা → qty "reserve" থাকে।
- Cancelled tender/PO → স্বয়ংক্রিয়ভাবে বাদ (Cancelled RFQ filter + `CalculateOrderedQuantities` ইতিমধ্যে Inactive PO বাদ দেয়, [PurchaseOrderItemRepository.cs:64](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Repository/Areas/SCM/PurchaseOrderItemRepository.cs#L64)) → qty মুক্ত হয়ে ফেরত।

এটাই SAP-এর "open quantity" / commitment-ledger ধারণা — কখনো stored নয়, সবসময় computed।

## ৩. কেন stored dashboard নয় (Option B-এর naive রূপ বাদ)

আলাদা `RFQAllocatedQty` কলাম রাখলে সেটা RFQ/PO প্রতিটা create/edit/cancel-এ sync করতে হবে — একটা মিস হলেই ভুল সংখ্যা। এটা হুবহু আপনার **D1-এর dual-status বিপর্যয়**। তাই dashboard একটা **read-time computed view**, টেবিল নয়।

## ৪. ডেটাবেস পরিবর্তন: **শূন্য** ✅

সব বিদ্যমান কলাম থেকে গণনা — `RFQItem.Quantity/PurchaseOrderId`, `RFQComparison.Status`, `PurchaseOrderProduct`। নতুন টেবিল/migration লাগবে না। এটাই এই পথের বড় সুবিধা।

## ৫. যা লাগবে (প্রস্তাবিত phased plan)

|ধাপ|কাজ|ফাইল|
|---|---|---|
|**১. Allocation service (core)**|`GetPrAllocationAsync(prId, excludeRfqId?)` → প্রতি product: Requested / OpenRfq / PO Ordered / Remaining। এক জায়গায় সত্য। নতুন repo query: PR-এর open (non-cancelled, PO-হীন) RFQ item গুলো product-ভিত্তিক sum, নিজের RFQ বাদ।|RFQRepository, RFQService|
|**২. Validator upgrade**|আমার লেখা `ValidateItemQuantitiesAgainstPRAsync`-কে raw PR qty-র বদলে **Remaining**-এর বিপরীতে চেক করানো (Add + Draft-Edit; edit-এ `excludeRfqId = self`)। মেসেজ: `"Laptop: only 8 remaining against the PR (2 already allocated)."`|RFQService|
|**৩. RFQ create UX**|`GetPRItemsForRFQ`-কে allocation সহ ফেরাতে বলা → Add/Edit ফর্মে কলাম: Requested / Allocated / Remaining / **Max Allowed**, আর qty ইনপুটে `max=Remaining`।|RFQComparisonController, Add/Edit.cshtml|
|**৪. (ঐচ্ছিক) PR Allocation view**|আপনার dashboard টেবিল — Requested / RFQ / Remaining / PO / Received — একই service থেকে, শুধু render। Received আনতে GRN qty যোগ করতে হবে।|নতুন view|

ধাপ ১–৩ মূল সমস্যা পুরো সমাধান করে; ধাপ ৪ বাড়তি visibility (আলাদা commit-এ করা যায়)।

## ৬. একটা edge আপনি ঠিক করতে চাইবেন

PO **cancel/remove** হলে সংশ্লিষ্ট `RFQItem.PurchaseOrderId` clear হওয়া উচিত (নইলে qty PO চ্যানেল থেকে মুক্ত হলেও RFQ চ্যানেলে ফেরে না — একটা units "হারিয়ে" যায়)। entity-র কমেন্ট বলছে "until that PO is removed" — তাই সম্ভবত hook দরকার। এটা ধাপ ১-এ যাচাই করে নেব।

---

আমার সুপারিশ: **ধাপ ১–৩ এখন করি** (schema বদল নেই, আপনার আগের Option-A validator-টাকেই সঠিক cumulative নিয়মে upgrade করা হয়), ধাপ ৪ পরে। এতে "Max Allowed: 8" UX আর server গ্যারান্টি দুটোই পাবেন।

কীভাবে এগোব, সিদ্ধান্ত দিন:

AskUserQuestion