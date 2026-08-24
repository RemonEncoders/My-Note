Problem identified by you : 
১. যে ফাঁকটা আপনার নিয়মে এখনো আছে (সবচেয়ে জরুরি)

আপনার নিয়ম: `Σ(active RFQ qty) ≤ PR qty`। কিন্তু quantity **দুইটা চ্যানেল** দিয়ে PR থেকে বের হয়:

```text title:"Scenario of PR to RFQ / PO

                 ┌── RFQ চ্যানেল (solicitation) ────┐
PR (10) ─────────┤                                 ├──► একই ইউনিট
                 └── PO চ্যানেল (commitment) ───────┘

```

দুটো সমস্যা:

**(ক) Direct PR→PO (Scenario 2) উপেক্ষিত।** ধরুন PR=10, সরাসরি একটা PO হলো 5 laptop (RFQ ছাড়া)। আপনার নিয়ম শুধু RFQ যোগ করে → RFQ allocated 2 ≤ 10 pass → নতুন RFQ-তে আরও 8 দেওয়া যাবে। অথচ আসলে বাকি মাত্র **3** (5 তো PO হয়ে গেছে)। আবার over-allocation।

**(খ) RFQ→PO রূপান্তরে double-count।** RFQ-1-এর 2 laptop award হয়ে PO হলে — সেই 2 কি RFQ চ্যানেলে গোনা হবে, নাকি PO চ্যানেলে? দুই জায়গায় গুনলে 2+2=4 ধরা পড়বে, ভুল।

what i catch and think about from my perspective with use case: 
যেহেতু PR থেকে PO এবং RFQ এই দুইটা path রয়েছে। 

কিন্তু এই মুহূর্তে সমস্যা যেটা তা হলো : আমি কতগুলো maximum কতগুলো Iteam quantity র জন্য RFQ করতে পারবো। এই হিসাব টা একটু জটিল কেননা যেহেতু যেই Iteam গুলোর RFQ হয়েছে তা অবশ্যই PO হবে বা হয়েছে। আবার,একটি PR এর যেই Iteam গুলোর RFQ হয়ে গেছে পরবর্তীতে যখন আবার same PR এর উপর RFQ হবে তখন rest of the iteam quantity র উপর হতে হবে। সেক্ষেত্রে কেবল এই জিনিসটা বিবেচনায় নিলে হবে না যে, এখন পর্যন্ত কতগুলো product এর RFQ হয়েছে তাহলে PR against এ থাকা একটি নির্দিষ্ট Iteam Total quantity থেকে - (এ যাবৎ RFQ হয়েছে সেগুলো বাদ দেওয়া )।

এটা করলে ভুল হবে। কেননা। যদি আমি ঔ Iteam এর  PO করে ফেলি RFQ কে পাশ কাটিয়ে কেউ তো মানা করবে না। কারণ, আমার সিস্টেমে এটা করা যায়। এক্ষেত্রে করণীয় হবে PO table এ একটি RFQId column যোগ করা এবং লক্ষ্য রাখা যে,  যদি PO তে RFQId থাকে তারমানে এটা হিসাবে না নিলেও চলবে কারণ আমার তো ঔ PR এর against এ RFQ table কতগুলোর জন্য এখন পর্যন্ত tender হয়েছে আর PO table থেকে ঔ PR against এ  যদি PO থাকে তাহলে তাদের উপর aggregate function add করে দুই রেজাল্ট যোগ করে যা পাওয়া যাবে তার থেকে বাদ দিতে হবে PR quantity তাহলে আমরা সঠিক সংখ্যাটা পাবো যে আর কতগুলোর জন্য আমরা rfq করতে পারবো।

But here is a problem: 
# আপনার Procurement System-কে একটি Warehouse Gate হিসেবে কল্পনা করুন

ধরুন PR হচ্ছে একটি গুদাম যেখানে ১০টি Laptop আছে।

```
                 Purchase Requisition
             ┌──────────────────────────┐
             │      Laptop = 10         │
             └──────────────────────────┘
```

এই ১০টি Laptop **দুইটি Gate** দিয়ে বের হতে পারে।

```
                    PR (10)
                       │
         ┌─────────────┴─────────────┐
         │                           │
         ▼                           ▼
     RFQ Gate                    Direct PO Gate
 (Tender / Quotation)          (Purchase Order)
```

এখন প্রশ্ন হচ্ছে—

> **এই ১০টির মধ্যে কতগুলো এখনো RFQ করার জন্য available?**

---

# Case 1 (শুধু RFQ হয়েছে)

```
PR = 10

          RFQ
           │
           ▼
        Laptop = 2
```

তখন

```
Remaining = 10 - 2 = 8
```

এখানে আপনার formula ঠিক।

---

# Case 2 (Direct PO হয়েছে)

```
PR = 10

              Direct PO
                  │
                  ▼
             Laptop = 5
```

এখন RFQ Table-এ কিছুই নেই।

আপনার formula যদি হয়

```
Remaining

=

PR Qty

-

SUM(RFQ Qty)
```

তাহলে

```
10 - 0 = 10
```

System বলবে

> তুমি ১০টার RFQ করতে পারো।

কিন্তু

বাস্তবে

```
৫টা তো already PO হয়ে গেছে।
```

তাই

```
Remaining হওয়া উচিত

10 - 5

=

5
```

---

# তাই আপনি ভাবলেন

ঠিক আছে।

তাহলে

```
Remaining

=

PR

-

RFQ

-

PO
```

এখানেই Claude-এর আপত্তি।

---

# কেন?

কারণ

ধরুন

```
PR

↓

RFQ

↓

PO
```

Flow

```
PR

10

↓

RFQ

2

↓

PO

2
```

এই ২টা Laptop কি

RFQ?

নাকি

PO?

উত্তর

দুটোই।

---

যদি আপনি করেন

```
Remaining

=

10

-

2(RFQ)

-

2(PO)
```

তাহলে

```
Remaining = 6
```

কিন্তু

বাস্তবে

শুধু

২টাই allocate হয়েছে।

Remaining হওয়া উচিত

```
8
```

আপনি একই quantity দুইবার subtract করলেন।

এটাই Claude-এর

> double-count

---

# তাই আপনার নতুন idea

আপনি বলেছেন

> PO table-এ RFQId রাখবো।

Visual করি।

```
PO

----------------------

PO-001

Qty = 2

RFQId = 5
```

মানে

এই PO এসেছে

RFQ-5 থেকে।

তাহলে

System বুঝবে

```
এই ২টা

RFQ-তে already ধরা আছে।

PO-তে আবার subtract করবো না।
```

---

আর

Direct PO

```
PO

Qty = 5

RFQId = NULL
```

মানে

RFQ bypass করেছে।

তাই

এই ৫টা

subtract করতে হবে।

---

তখন Formula হবে

```
Remaining

=

PR Qty

-

SUM(All Active RFQ Qty)

-

SUM(Direct PO Qty)

where RFQId IS NULL
```

Visual

```
PR = 10

             ┌──────────────┐
             │ RFQ = 2       │
             └──────────────┘

             ┌──────────────┐
             │ Direct PO =5 │
             │ RFQId = NULL │
             └──────────────┘

Remaining

=

10

-

2

-

5

=

3
```

এবার একদম সঠিক।


Here is my daily use cases that i have to handle through this solution: 
my story: 
আমার procurement team 10 টা laptop এর pr করলো। কিন্তু ৩ টা ল্যাপটপ এই মূর্হতে দরকার তাই সরাসরি po করে কিনলাম। আর বাকি ৭ টা ল্যাপটপ আমাকে কিনতে হবে কিন্ত আমার কাছে এতো টাকা নেই তাই আমি কি করলাম আপাতত ৪ টা ল্যাপটপ কেনার জন্য tender publish কটলাম। তাহলে বাকি রইল আর ৩ টা। এই ৩ টা laptop এর জন্য আমি পরে rfq দিবো বা সরাসরি po করবো। আবার এমনও হতে পারে এখন 4 টা ল্যাপটপের জন্য RFQ দিয়েছি কারণ এখন দেখি vendor রা আমাকে কেমন ল্যাপটপ দেয়। 
তাই আমি বুঝে শুনে একটু ধীর স্থির হয়ে কিনতে চাচ্ছি কারণ, এটা business হুট হাট করে সিদ্ধান্ত নেওয়া যায় না। 

 এমনও হতে পারে যে, আমি ১০ টা ল্যাপটপের জন্যই একটা RFQ দিয়েছি। 

আবার আমি কোনো rfq না দিয়ে সরাসরি ১০ টা laptop এর po করে ফেলেছি।


Here is my recomendation: 
# তাহলে আসল Formula কী?

আমি আপনার system-এর জন্য

এই Formula-টাই recommend করবো।

```
Remaining

=

PR Qty

-

Committed Qty

-

Tender Running Qty
```

যেখানে

```
Committed Qty

=

Direct PO

+

PO created without active RFQ
```

এবং

```
Tender Running Qty

=

RFQ

যেগুলো এখনো Award হয়নি।
```

---

RFQ Award হয়ে গেলে

ওটা

Tender Running

থেকে বের হবে।

PO-তে যাবে।

---

Visual

```
                 PR = 10

                     │

        ┌────────────┴─────────────┐

        │                          │

        ▼                          ▼

   Direct PO                  Active RFQ

      3                           4

        │                          │

        └────────────┬─────────────┘

                     │

       Reserved / Allocated = 7

                     │

                     ▼

              Remaining = 3
```

---

# কেন এটা সুন্দর?

কারণ

আপনার

RFQ

এবং

PO

দুটোই

একটা quantity reserve করছে।

কিন্তু

RFQ থেকে

PO হলে

দুইবার reserve করবে না।