## এর পেছনের বিজনেস যুক্তি

Goods আর Service-এর খরচ ট্র্যাকিং আলাদা:

||Goods PO|Service PO|
|---|---|---|
|গন্তব্য|Store/Inventory-তে ঢোকে, GRN হয়|কোনো স্টক নেই|
|খরচ কার|পরে issue হলে যে ডিপার্টমেন্ট নেয়|**PO-র সময়েই ঠিক করতে হয়**|

| বিষয়                         | **Goods PO**                                    | **Service PO**                                              |
| ---------------------------- | ----------------------------------------------- | ----------------------------------------------------------- |
| **গন্তব্য (Destination)**    | Store/Inventory-তে ঢোকে                         | কোনো Inventory/Stock-এ যায় না                              |
| **GRN (Goods Receipt Note)** | ✅ GRN করতে হয়                                  | ❌ GRN লাগে না (অনেক ERP-তে Service Entry/Acceptance হয়)    |
| **Stock Movement**           | ✅ Stock বৃদ্ধি পায়                             | ❌ কোনো Stock Movement হয় না                                |
| **খরচ (Expense) কখন হয়**    | Item Issue/Consumption-এর সময়                  | **PO/Service Receipt/Invoice-এর সময়ই Expense ধরা হয়**     |
| **খরচ কার (Cost Ownership)** | পরে যে Department/Cost Center Item ব্যবহার করবে | **PO তৈরির সময়ই Department/Cost Center নির্ধারণ করতে হয়** |
| **Accounting Flow**          | **PO → GRN → Inventory → Issue → Expense**      | **PO → Service Receipt/Invoice → Expense**                  |

সার্ভিসের কোনো ফিজিক্যাল ইনভেন্টরি নেই, তাই "কোন ডিপার্টমেন্টের খরচ" সেটা ইস্যুর সময় ধরার সুযোগ থাকে না — PO লেভেলেই Cost Center-এ ভাগ করতে হয়। সেজন্যই টেবিলে **Cost Center + Amount** দুটো কলাম — যেমন এক সার্ভিস আইটেমের ৳১,০০,০০০ কে OT ৳৬০,০০০ + ICU ৳৪০,০০০ ভাগ করা যায়। ডেটা যায় `PurchaseOrderCostCenter` টেবিলে, সেভ হয় [PurchaseOrderController.cs:567](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/SCM/Controllers/PurchaseOrderController.cs#L567) `ServiceItemSave` দিয়ে।

লক্ষণীয়: [Add.cshtml:190-195](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/SCM/Views/PurchaseOrder/Add.cshtml#L190-L195)-এ Cost Center ড্রপডাউনটা `@* *@` দিয়ে কমেন্ট করা — অর্থাৎ ডিজাইনই ছিল PO তৈরির সময় নয়, **তৈরির পর Details/Edit থেকে** কস্ট সেন্টার বসানো।

আপনি কি চান এই সেকশনটা রাখা হোক, নাকি নতুন RFQ→PO ফ্লোতে (`CreateFromTender`) এটা নিয়ে ভিন্ন কিছু ভাবব?