# Question:
```
সে একটা hidden issue ধরেছে।

UpdateStatus

GET

😄

এটা আমি report-এ আশা করিনি।

কিন্তু এটা সত্যিই serious।

Status change

কখনো GET হওয়া উচিত না।

কারণ

Refresh

Browser Prefetch

Crawler

Double click

সব trigger করতে পারে।

এটা business rule-এর বাইরে

HTTP semantics issue।

এটা fix করা উচিত।
```

# Ans:
হ্যাঁ। এটা আসলে খুব গুরুত্বপূর্ণ একটি Web Development concept। অনেক Junior Developer এটা overlook করে, কিন্তু Senior Developer বা Security Review-তে এটা খুব দ্রুত ধরা পড়ে।

চলুন একদম শুরু থেকে বুঝি।

---

# প্রথমে HTTP Method বুঝি

HTTP-তে বিভিন্ন Method আছে।

|Method|কাজ|
|---|---|
|GET|Data পড়বে (Read Only)|
|POST|নতুন Data তৈরি করবে|
|PUT|পুরো Data Update করবে|
|PATCH|আংশিক Update করবে|
|DELETE|Data Delete করবে|

HTTP Specification-এর একটা খুব গুরুত্বপূর্ণ Rule আছে।

> **GET must be Safe.**

অর্থাৎ

GET request কখনো server-এর state change করতে পারবে না।

মানে

```
GET /products

```

এটা শুধু data দেখাবে।

এটা database modify করবে না।

---

# তাহলে UpdateStatus GET কেন সমস্যা?

ধরুন আপনার API

```
GET
/PurchaseOrder/UpdateStatus/5

```

এবং এটা internally করে

```
PO.Status = Approved;

SaveChanges();

```

এখন GET request database change করছে।

এটাই বড় সমস্যা।

---

# কেন এটা dangerous?

## Problem 1 : Browser Refresh

ধরুন User browser-এ গেল

```
GET
/PurchaseOrder/UpdateStatus/5

```

Status Approved হয়ে গেল।

এখন User

```
F5

```

চাপলো।

Browser আবার একই GET request পাঠাবে।

```
GET
/PurchaseOrder/UpdateStatus/5

```

আবার Status Update হবে।

হয়তো

```
Audit Log

Notification

Workflow

Email

```

আবার trigger হবে।

---

## Problem 2 : Browser Prefetch

Modern browser অনেক সময়

আগে থেকেই link load করে।

যেমন

Page-এ আছে

```
<a href="/UpdateStatus/5">

Approve

</a>

```

Chrome ভাবতে পারে

> "User হয়তো click করবে।"

তাই

আগে থেকেই GET request পাঠিয়ে দেয়।

User click-ই করল না।

কিন্তু

Status Approved হয়ে গেল।

এটাকে বলে

```
Prefetch

```

---

## Problem 3 : Search Engine Crawler

Google Bot

Bing Bot

Crawler

সব link follow করে।

ধরুন

```
<a href="/PurchaseOrder/UpdateStatus/5">

Approve

</a>

```

Crawler এলো।

Link follow করল।

GET request গেল।

Database update হয়ে গেল।

Google Bot আপনার Purchase Order approve করে দিল।

শুনতে মজার,

কিন্তু বাস্তবে এমন bug বহুবার হয়েছে।

---

## Problem 4 : Double Click

User

Approve button

দুইবার click করল।

যদি GET হয়

```
GET
UpdateStatus

```

দুইবার execute হবে।

ফলাফল

```
Audit Log
Notification
SMS
Journal Entry
Stock Posting

```

সব duplicate হতে পারে।

---

## Problem 5 : Back Button

Browser history-তে GET request থাকে।

User

Back

Forward

Refresh

করলে

আবার request যেতে পারে।

যদি GET update করে

state আবার change হতে পারে।

---

# HTTP-এর একটা Principle

REST এবং HTTP Specification বলে

GET হওয়া উচিত

> Safe

এবং

> Idempotent

---

## Safe মানে

কোন state change করবে না।

যেমন

```
GET /products

```

শুধু পড়বে।

---

## Idempotent মানে

একই request

১০০ বার করলেও

একই result।

উদাহরণ

```
PUT

```

```
PUT
/Product/5

Price = 100

```

একবার করুন

Price =100

১০ বার করুন

Price =100

State একই।

---

কিন্তু

```
GET
/Approve/5

```

একবার

↓

Approved

আবার

↓

Notification

আবার

↓

Audit

আবার

↓

Email

State change হচ্ছে।

---

# CSRF Problem

আরও dangerous।

ধরুন

আপনি login আছেন।

কেউ একটা malicious website বানালো।

ওখানে আছে

```html
<img src="https://mysystem.com/PO/UpdateStatus/5">
```

Browser image load করার জন্য GET request পাঠিয়ে দিল।

আপনি কিছুই করেননি।

Status update হয়ে গেল।

এটাকে বলে

```
Cross Site Request Forgery (CSRF)
```

State-changing action GET হলে এই ধরনের ঝুঁকি অনেক বেড়ে যায়। তাই সাধারণভাবে state-changing endpoint-এ POST/PUT/PATCH/DELETE ব্যবহার করা হয় এবং CSRF protection (বিশেষ করে cookie-based authentication হলে) প্রয়োগ করা হয়।

---

# তাহলে কী হওয়া উচিত?

ভুল

```
GET
/PurchaseOrder/UpdateStatus/5

```

সঠিক

```
POST
/PurchaseOrder/UpdateStatus

```

অথবা

```
PATCH
/PurchaseOrder/5/status

```

Body

```
{
    "status":"Approved"
}
```

---

# ASP.NET Core-এ

ভুল

```csharp
[HttpGet]
public IActionResult UpdateStatus(int id)
{
    ...
}
```

সঠিক

```csharp
[HttpPost]
public IActionResult UpdateStatus(int id)
{
    ...
}
```

অথবা

```csharp
[HttpPatch("{id}/status")]
public IActionResult UpdateStatus(int id, UpdateStatusDto dto)
{
    ...
}
```

---

# কেন Reviewer এটা Report-এ লিখেছে?

কারণ এটা শুধু Business Logic-এর সমস্যা নয়।

এটা **HTTP protocol semantics**, **Web Security**, এবং **REST API design**-এর সমস্যা।

Business rule ঠিক থাকলেও যদি endpoint এমন হয়—

```
GET
/UpdateStatus
```

তাহলে একজন Senior Reviewer সঙ্গে সঙ্গে বলবে:

> **"State-changing operation must not use GET."**

---

# বাস্তব উদাহরণ (আপনার Procurement System)

ধরুন আপনার URL:

```
GET /SCM/PurchaseOrder/UpdateStatus/125?status=Approved
```

একজন Procurement Officer Approve করতে চাইলেন।

কিন্তু একই URL যদি:

- Browser refresh করে,
    
- Browser prefetch করে,
    
- Search engine crawler follow করে,
    
- কোনো script ভুলবশত call করে,
    

তাহলে Purchase Order আবার "Approve" action execute করতে পারে। যদি এই action-এর সাথে Notification, Audit Log, Journal Entry, বা Inventory Posting যুক্ত থাকে, তাহলে সেগুলোও অপ্রত্যাশিতভাবে পুনরায় চলতে পারে।

তাই সঠিক নকশা হবে:

```
POST /SCM/PurchaseOrder/125/status
```

অথবা

```
PATCH /SCM/PurchaseOrder/125/status
```

যেখানে শুধুমাত্র ব্যবহারকারীর ইচ্ছাকৃত action-ই status পরিবর্তন করবে।

---

## মূল শিক্ষা

একজন Senior Developer সাধারণত এই নিয়মটি মনে রাখেন:

- **GET = শুধু পড়বে (Read Only)**
    
- **POST = Action বা Create**
    
- **PUT/PATCH = Update**
    
- **DELETE = Remove**
    

**যদি কোনো endpoint database-এর state পরিবর্তন করে, তাহলে সেটি কখনোই GET হওয়া উচিত নয়।** এটা শুধু REST-এর best practice নয়—এটা নিরাপত্তা, নির্ভরযোগ্যতা এবং সঠিক HTTP semantics বজায় রাখার জন্যও অত্যন্ত গুরুত্বপূর্ণ।