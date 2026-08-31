**slug হলো একটা permission-এর ছোট, মেশিন-পাঠযোগ্য নাম-ID।** মানুষের জন্য যেমন display name থাকে, প্রোগ্রামের জন্য তেমন slug।

## আপনার নিজের codebase থেকেই দেখুন

```csharp
new() { Name = "Stock Levels",           // ← মানুষ পড়বে (RBAC admin UI-তে দেখায়)
        Slug = "ims.stock-levels",       // ← সিস্টেম চেনে (স্থায়ী ID)
        HttpMethod = "GET",
        HttpPath = "/ims/stocklevels*" } // ← কোন URL-এ প্রযোজ্য
```

এখানে ৩টা জিনিস তিন কাজের:

- **`Name`** — শুধু প্রদর্শনের জন্য; কাল বদলে "Stock Level Report" করলেও কিছু ভাঙবে না
- **`Slug`** — আসল **চাবি (key)**; DB-র role-permission mapping টেবিল, menu item, code — সব জায়গা এই string দিয়েই permission-টাকে refer করে
- **`HttpPath`** — enforcement: কোন route-এ এই permission লাগবে

## কেন "slug" বলে, আর গঠনটা এমন কেন

শব্দটা এসেছে সংবাদপত্র/URL-জগত থেকে — লম্বা শিরোনামের ছোট, ফাঁকা-স্থানহীন, সব-ছোট-হাতের সংস্করণকে slug বলে (যেমন blog URL-এ `/my-first-post`)। বৈশিষ্ট্য:

```
ims.stock-levels
└┬┘ └────┬─────┘
module   পেজ/ফিচার (kebab-case: ছোট হাতের, শব্দ-মাঝে হাইফেন)
```

- **space নেই, বড় হাতের অক্ষর নেই** — তাই URL, config, DB — সব জায়গায় নিরাপদে বসে
- **`module.feature` কাঠামো** — এক নজরেই বোঝা যায় কোনটা কার (`hrm.payroll`, `otm.seed`, `scm.supplier`)

## বাস্তবে কীভাবে খাটে (আপনার সিস্টেমে)

ধরুন স্টোরকিপার role-কে শুধু Stock Levels দেখার অনুমতি দেবেন:

1. RBAC admin UI-তে role-এর সাথে **`ims.stock-levels`** permission map করলেন → DB-তে জমা হলো এই slug-টাই
2. ইউজার `/IMS/StockLevels` খুলল → middleware দেখল এই URL কোন permission-এর `HttpPath`-এ পড়ে → পেল `ims.stock-levels` → ইউজারের role-এ এই slug আছে কি না check → আছে, ঢুকতে দিল
3. Sidebar menu আঁকার সময়ও একই check — এই slug না থাকলে menu item-টাই দেখায় না

অর্থাৎ **slug হলো সেই সুতো যা role ↔ permission ↔ menu ↔ URL — চারটাকে এক জায়গায় বাঁধে।** আমি যে টেবিলে "নতুন ১১টা slug" প্রস্তাব করেছি, তার মানে: ১১টা নতুন permission-চাবি বানানো হবে, যেগুলো পরে আপনি যেকোনো role-কে দিতে/কাড়তে পারবেন।