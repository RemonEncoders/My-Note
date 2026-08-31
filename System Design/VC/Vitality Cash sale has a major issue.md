## তুমি যা জানতে চেয়েছিলে

**PO-তে যে Unit Price দিলাম, সেটা Sale-এ কীভাবে আসে?**

## Claude-এর উত্তর

**আসেই না।**

---

## Developer যা করতে চেয়েছিল

```text
PO
 ↓
GRN
 ↓
Sales Screen
(Price Auto Fill হবে)
```

---

## কিন্তু Bug-এর কারণে যা হচ্ছে

```text
PO
 ↓
GRN
 ↓
❌ Price খুঁজে পায় না
 ↓
0 টাকা দেখায়
 ↓
Salesman নিজে দাম লিখে
 ↓
Save
```

---

## Bug কেন?

Code-এ ভুল Method Call হয়েছে।

Developer লিখেছে:

```csharp
GetGRProductItemById(productId, CurrentCompanyId)
```

কিন্তু C# এটা বুঝেছে:

```csharp
GetGRProductItemById(productId, GoodsReceiveId)
```

অর্থাৎ

**CompanyId-কে ভুল করে GoodsReceiveId ধরে নিয়েছে।**

তাই Database-এ কিছুই খুঁজে পায় না।

---

## Result

```text
Price = NULL

↓

0 টাকা

↓

Salesman নিজে দাম লিখে
```

---

## কিন্তু COGS ঠিকমতো হয়?

**হ্যাঁ।**

কারণ COGS অন্য Method ব্যবহার করে।

```text
Sale
 ↓
Batch No
 ↓
Goods Receive
 ↓
Purchase Cost
```

এটা ঠিক কাজ করছে।

---

## এক লাইনে পুরো রিপোর্ট

> **VitalityCash-এ Purchase Cost (COGS) ঠিকমতো পাওয়া যায়, কিন্তু Sales-এর Auto Price Suggestion একটি Code Bug-এর কারণে কাজ করে না। তাই Salesman-কে প্রতিবার দাম হাতে লিখতে হয়।**
## প্রমাণ — ৩টি ফাইল, নির্দিষ্ট লাইন নম্বর সহ

### ১) Developer যা লিখেছে (কল সাইট)

**ফাইল:** `VitalityCash.Services/GoodsReceiveService.cs` **লাইন:** ৬০৬-৬০৯ (মূল লাইন: **৬০৮**)

Read [GoodsReceiveService.cs](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/index.html?id=2dbcef70-aaa3-4dda-ada4-dbdf0c1c7529&parentId=1&origin=3227b2e5-9434-432a-8e5f-3c0fed786680&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&session=fdd63756-c375-48db-976b-c05ec3153ef9#) (lines 607-610)

```csharp
// GoodsReceiveService.cs:608
return await _unitOfWork.GoodsReceiveItemRepository.GetGRProductItemById(Id, CurrentCompanyId);
```

Developer-এর উদ্দেশ্য ছিল company-filtered lookup করা — কিন্তু `IGoodsReceiveItemRepository`-তে `(Guid, Guid)` shape-এর যে overload প্রথমে পাওয়া যায় সেটার দ্বিতীয় প্যারামিটারের নাম `CurrentCompanyId` নয়, **`GRId`**।

---

### ২) Interface-এ ৩টি overload-এর ঘোষণা (এখানেই ambiguity তৈরি হয়)

**ফাইল:** `VitalityCash.Repository/Contracts/IGoodsReceiveItemRepository.cs` **লাইন:** ১২-১৫

Read [IGoodsReceiveItemRepository.cs](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/index.html?id=2dbcef70-aaa3-4dda-ada4-dbdf0c1c7529&parentId=1&origin=3227b2e5-9434-432a-8e5f-3c0fed786680&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&session=fdd63756-c375-48db-976b-c05ec3153ef9#) (lines 11-17)

```csharp
// IGoodsReceiveItemRepository.cs:12-15
Task<GoodsReceiveProduct> GetGRProductItemById(Guid Id, Guid GRId, Guid CurrentCompanyId);  // ৩ arg
Task<GoodsReceiveProduct> GetGRProductItemById(Guid Id, Guid GRId);                          // ২ arg  ← এখানে bind
Task<GoodsReceiveProduct> GetGRProductItemById(Guid Id);                                     // ১ arg
```

`GoodsReceiveService.cs:608`-এর কলে ঠিক **২টি** `Guid` আর্গুমেন্ট আছে (`Id`, `CurrentCompanyId`)। C# compiler argument-এর **সংখ্যা ও টাইপ** মিলিয়ে overload বাছে, parameter-এর নাম দেখে নয়। তাই এই কল লাইন ১৩-এর ঘোষণায় bind হয় — যার দ্বিতীয় parameter-এর নাম **`GRId`**, `CurrentCompanyId` নয়।

---

### ৩) প্রকৃত বাস্তবায়ন — যেখানে `CompanyId` আসলে `GRId` হয়ে যায়

**ফাইল:** `VitalityCash.Repository/GoodsReceiveItemRepository.cs` **লাইন:** ২৪-২৭

Read [GoodsReceiveItemRepository.cs](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/index.html?id=2dbcef70-aaa3-4dda-ada4-dbdf0c1c7529&parentId=1&origin=3227b2e5-9434-432a-8e5f-3c0fed786680&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&session=fdd63756-c375-48db-976b-c05ec3153ef9#) (lines 25-28)

```csharp
// GoodsReceiveItemRepository.cs:24-27
public async Task<GoodsReceiveProduct> GetGRProductItemById(Guid Id, Guid GRId)
{
    return await _dbContext.GoodsReceiveProducts...
        .FirstOrDefaultAsync(x => x.ProductId == Id && x.GoodsReceiveId == GRId);
}
```

এই মেথড বডিতে যা লেখা আছে — `x.GoodsReceiveId == GRId` — সেখানে `GRId`-এর জায়গায় প্রকৃতপক্ষে **রানটাইমে `CurrentCompanyId`-এর মান বসে যায়** (কারণ কলার সেটাই পাঠিয়েছে দ্বিতীয় অবস্থানে)। ফলে বাস্তবে যে SQL কন্ডিশন এক্সিকিউট হয়:

```sql
WHERE ProductId = @Id AND GoodsReceiveId = @CompanyIdValue
```

---

## সংক্ষিপ্ত প্রমাণ-চেইন (ফাইল:লাইন সহ)

|ধাপ|ফাইল|লাইন|কী আছে|
|---|---|---|---|
|১ — কল সাইট|`VitalityCash.Services/GoodsReceiveService.cs`|৬০৮|`GetGRProductItemById(Id, CurrentCompanyId)` — ২টি Guid আর্গুমেন্ট|
|২ — Overload ঘোষণা|`VitalityCash.Repository/Contracts/IGoodsReceiveItemRepository.cs`|১২-১৫|`(Guid, Guid)` shape-এর একমাত্র ম্যাচ, প্যারামিটার নাম `GRId`|
|৩ — প্রকৃত কোয়েরি|`VitalityCash.Repository/GoodsReceiveItemRepository.cs`|২৪-২৭|`x.GoodsReceiveId == GRId` — যেখানে `GRId`-এর মান আসলে `CompanyId`|

**উপসংহার:** Developer টাইপ করেছিলেন `GetGRProductItemById(productId, CurrentCompanyId)`, কিন্তু C#-এর overload resolution parameter-নাম দেখে না, শুধু argument-সংখ্যা ও টাইপ দেখে — তাই এই কলটি `GetGRProductItemById(Guid Id, Guid GRId)`-এ bind হয়ে গেছে, এবং `CompanyId`-এর মান `GRId` (GoodsReceiveId) হিসেবে ব্যবহৃত হয়েছে। যেহেতু একটি Company-র Id কখনো কোনো GoodsReceive ডকুমেন্টের Id-র সমান হবে না, এই কোয়েরি বাস্তবে সবসময় `null` রিটার্ন করে।