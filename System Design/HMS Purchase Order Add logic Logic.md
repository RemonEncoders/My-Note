নিচে পুরো সমাধান-প্রক্রিয়া আর প্রতিটি সিদ্ধান্তের যুক্তি একসাথে দিলাম — এটা দেখেই আপনি চূড়ান্ত সিদ্ধান্ত নিতে পারবেন।

---

## ১. সমস্যাটা কী ছিল (রিক্যাপ)

- **তাৎক্ষণিক বাগ:** PR সিলেক্ট করলেই "Unable to load PR Items" — কারণ AJAX শুধু `prId` পাঠায়, কিন্তু [PurchaseController.cs:925](vscode-webview://1hkp3pmg1rkq084uobqs2co5n6qneo67lqs04s1r9984lj96o1kt/HMS.Web/Areas/SCM/Controllers/PurchaseController.cs#L925)-এর অ্যাকশন `poCategory` প্যারামিটারও চায়। সেটা null হয়ে `poCategory.Contains("Service")`-এ **NullReferenceException** → HTTP 500 → এরর টোস্ট।
- **আসল রিকোয়ারমেন্ট:** এক PR-এ Product/Asset/Service সব থাকতে পারে, কিন্তু এক PO-তে শুধু এক ধরনের আইটেম। PR সিলেক্টের পর PO Category বাছলে শুধু সেই ধরনের আইটেম দেখাবে। আর এক PR-এর বিপরীতে একাধিক আংশিক PO (ভিন্ন ভেন্ডরে) কাটা যাবে — তাই remaining quantity হিসাবটা অক্ষত থাকতে হবে।

## ২. চূড়ান্ত সমাধান — ধাপে ধাপে

**ধাপ ১ — Controller:** [PurchaseController.cs](vscode-webview://1hkp3pmg1rkq084uobqs2co5n6qneo67lqs04s1r9984lj96o1kt/HMS.Web/Areas/SCM/Controllers/PurchaseController.cs#L925)-এ `GetProductDetailsForPO(Guid prId)` — `poCategory` প্যারামিটার ও Goods/Service/Asset ভাগ করার পুরো সার্ভার-সাইড ব্লক বাদ। অ্যাকশনটা PR-এর `remainingQty > 0` এমন **সব** আইটেম একবারে JSON-এ দেবে। আপনার তিনটি ফিল্ড হুবহু থাকবে (`QuantityOrder`, `OrderedQuantity`, `Quantity = মূল − অর্ডারকৃত`), শুধু `ItemType` যাবে **int** হিসেবে (`.ToString()` নয়) এবং null-safe করে। পুরনো কমেন্ট-আউট ড্রাফট মেথডটা (লাইন 793–923) মুছে দেব।

**ধাপ ২ — View (Add.cshtml):** ফিল্টারিং সম্পূর্ণ ক্লায়েন্ট-সাইডে:

```js
Service PO  (ParentCategory === "Service") → itemType === 3 আইটেম
Local Goods / Asset (ParentCategory === "Goods") → itemType !== 3 আইটেম (সব প্রোডাক্ট)
কোনো Category সিলেক্ট নেই → খালি লিস্ট
```

সাথে `$(document).ready`-তে `LoadProductAndService()` — পেজ খোলামাত্র ড্রপডাউনে সিস্টেমের সব প্রোডাক্টের ফুল লিস্ট আর দেখাবে না; PR + Category বাছার পরই আইটেম আসবে। PR বা Category যেটাই বদলান, সব রো-র ড্রপডাউন রি-ফিল্টার হবে (বর্তমান ভ্যালু বৈধ থাকলে টিকে থাকবে)।

**ধাপ ৩ — সেফটি ফিক্স ক (ঐচ্ছিক কিন্তু সুপারিশকৃত):** [PurchaseOrderItemRepository.cs:33-44](vscode-webview://1hkp3pmg1rkq084uobqs2co5n6qneo67lqs04s1r9984lj96o1kt/HMS.Repository/Areas/SCM/PurchaseOrderItemRepository.cs#L33-L44)-এ `CalculateOrderedQuantities`-এ `Status != Inactive` শর্ত — বাতিল PO-র quantity যেন remaining থেকে আটকে না থাকে। Draft গণনায় থাকবে (over-order ঠেকাতে)।

**ধাপ ৪ — সেফটি ফিক্স খ (ঐচ্ছিক কিন্তু সুপারিশকৃত):** PO সেভের POST-এ সার্ভার-সাইড রি-ভ্যালিডেশন — প্রতিটি আইটেমের সাবমিটকৃত qty ≤ remaining (+Tolerance)। দুই ইউজার একসাথে PO বানালে বা কেউ হাতে বড় সংখ্যা বসালে PR ছাড়িয়ে অর্ডার হবে না।

**ধাপ ৫ — বিল্ড + টেস্ট:** মিক্সড PR দিয়ে তিন ক্যাটাগরির ফিল্টার, আংশিক PO-র পর remaining, পূর্ণ হলে আইটেম উধাও — এই সিনারিওগুলো।

**পরিবর্তিত ফাইল:** ৪টি — `PurchaseController.cs`, `Add.cshtml`, `PurchaseOrderItemRepository.cs`, `PurchaseOrderController.cs`/`PurchaseOrderModel.cs` (ধাপ ৪-এর জন্য)। Repository-র `GetPurchaseByIdAsync`-এ **কোনো হাত পড়বে না।**

## ৩. আগের সমাধান থেকে এটা কেন আলাদা, এবং কেন ভালো

আলোচনার মধ্য দিয়ে সমাধানটা তিন দফা বিবর্তিত হয়েছে:

||কীভাবে Asset/Goods/Service চেনা হতো|সমস্যা কী ছিল|
|---|---|---|
|**সংস্করণ ১** (বিদ্যমান কোড)|সার্ভারে `poCategory` **নামের স্ট্রিং** ম্যাচ (`Contains("Service")`)|null হয়ে ক্র্যাশ; ক্যাটাগরির নাম বদলালেই ভাঙে; প্রতি ক্যাটাগরি-বদলে সার্ভার কল লাগত|
|**সংস্করণ ২** (আমার প্রথম প্রস্তাব)|`itemType` (2/3) + `isAsset` ফ্ল্যাগ — Asset চেনা হতো `Product.Category.Name`-এ "Asset" খুঁজে|Repository-তে বাড়তি `Include` লাগত; নাম-ভিত্তিক ম্যাচিং ভঙ্গুর; পুরনো প্রোডাক্টের জন্য ডেটা-ফিক্স মাইগ্রেশন লাগত; "Chair" সমস্যার সমাধান ছিল না|
|**সংস্করণ ৩** (চূড়ান্ত — আপনার দুই ইনপুটের পর)|শুধু `Product.ItemType`: Service = 3, বাকি সব = Goods; Asset PO আর Local Goods PO **একই** প্রোডাক্ট লিস্ট দেখে|—|

চূড়ান্তটা বেছে নেওয়ার কারণগুলো:

1. **আপনার ডেটা মডেলের সাথে সামঞ্জস্য:** আপনার সিদ্ধান্ত — এক প্রোডাক্ট এক রেকর্ড, "Asset কিনা" সেটা প্রোডাক্টের ধর্ম নয়, **PO-র সিদ্ধান্ত** (PO Category-ই সেটা ধারণ করে)। তাই আইটেম-লেভেলে Asset আলাদা করার চেষ্টা (সংস্করণ ১, ২) ধারণাগতভাবেই ভুল ছিল — একই চেয়ার দুই লিস্টে ভাগ করা যায় না।
2. **কম কোড, কম নির্ভরতা:** নাম-ম্যাচিং নেই, `isAsset` নেই, Repository-তে নতুন `Include` নেই, ডেটা-ফিক্স মাইগ্রেশন নেই। ভাঙার জায়গা যত কম, বাগ তত কম।
3. **এক PR = এক সার্ভার কল:** PR-এর সব আইটেম একবারে আসে; ইউজার Category বদলে বদলে দেখলেও আর নেটওয়ার্ক কল হয় না — দ্রুত, আর সার্ভারে চাপ কম।
4. **আপনার partial-PO লজিক স্পর্শই করা হয়নি:** remaining-qty ক্যালকুলেশন যেখানে ছিল সেখানেই, বরং ধাপ ৩–৪ সেটাকে আরও নির্ভরযোগ্য করে (বাতিল PO + concurrent ইউজার কেস)।
5. **ট্রেড-অফ যেটা মেনে নিচ্ছি (জেনে-বুঝে):** Asset PO আর Local Goods PO-তে একই লিস্ট দেখাবে — অর্থাৎ সিস্টেম ইউজারকে "ভুল" ক্যাটাগরিতে প্রোডাক্ট কিনতে বাধা দেবে না। এটা দুর্বলতা নয়, আপনার ব্যবসায়িক সিদ্ধান্তেরই প্রতিফলন — বাছাইয়ের দায়িত্ব ইউজারের, আর কোনটা অ্যাসেট হিসেবে কেনা হলো তা PO-র `POCategoryId` থেকেই পরে ট্রেস/রিপোর্ট করা যাবে।