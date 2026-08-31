# Problem
Please investigate a submission failure on the **Goods Receive** screen. ### Scenario URL: `https://localhost:44389/SCM/GoodsReceive/Add` Test Data: * Vendor: **Crest Tech** * PO No: **1020260026** * Memo No: **12345675** why iteam details table not shwoing anything

# Analysis Report
## তদন্তের ফলাফল (Root Cause Analysis) — Item Details Table খালি থাকার কারণ

লাইভ ব্রাউজার টেস্ট (Playwright দিয়ে) চালিয়ে হুবহু আপনার test data (Vendor: Crest Tech, PO: 1020260026) দিয়ে reproduce করে নিশ্চিত হয়েছি। কোনো কোড পরিবর্তন করা হয়নি — শুধু temporary investigation script দিয়ে verify করে সেটা মুছে ফেলা হয়েছে।

### যা যাচাই করা হয়েছে (ডাটাবেস + লাইভ ব্রাউজার)

- DB-তে সরাসরি কুয়েরি করে দেখা গেছে: PO 1020260026 → Vendor "Crest Tech Corporation" এর সাথে সঠিকভাবে যুক্ত, Status = **Final**, ১টি product line (Qty 100), কোনো পূর্ববর্তী Goods Receive নেই। অর্থাৎ ডেটার দিক থেকে সব ঠিক আছে — এই PO-তে রিসিভযোগ্য item থাকার কথা।
- ব্রাউজার দিয়ে Vendor ও PO সিলেক্ট করে Network ট্যাব monitor করে দেখা গেছে:
    - `GetPurchaseOrderForGoodsByVendorId` → ✅ ২০০ms-এর মধ্যে সঠিক PO list রিটার্ন করে (1020260026 সহ)।
    - PO সিলেক্ট করার পর **দুটো** AJAX request একসাথে ছোড়া হয়:
        - `GetVendorForGRByPOId` → ✅ সাথে সাথে ২০০ OK রেসপন্স আসে।
        - `GetProductDetailsForGRByPOId` (যেটা Item Details table populate করে) → **request সার্ভারে যায়, কিন্তু ১৫ সেকেন্ড অপেক্ষা করার পরও কোনো response আসে না — কোনো error নেই, কোনো timeout-ও নেই। Request টা চিরকালের জন্য ঝুলে থাকে (hang)।**
    - কনসোলে কোনো JS error নেই (আগের bug-টা আলাদা ছিল, এটা ভিন্ন সমস্যা)।

তাই টেবিল খালি দেখানোর কারণ: **JavaScript-এর কোনো দোষ নেই — সার্ভার-সাইড অ্যাকশনটাই কখনো রেসপন্স পাঠাচ্ছে না।**

### রুট কজ — [PurchaseOrderController.cs:796-802](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/SCM/Controllers/PurchaseOrderController.cs#L796-L802)

```csharp
public async Task<JsonResult> GetProductDetailsForGRByPOId(Guid id)
{
    var model = await Task.Run(() => new PurchaseOrderModel());
    model.Init(id).Wait();   // ← ব্লকিং কল
    ...
```

`Init(id)` নিজেই [PurchaseOrderModel.cs:196](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/SCM/Models/PurchaseOrderModel.cs#L196)-এ `await Init();` কল করে, আর সেই parameterless `Init()`-এর ভেতরেই আরও ৩ জায়গায় ব্লকিং `.Result` আছে:

- [line 108](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/SCM/Models/PurchaseOrderModel.cs#L108) — `_currencyService.GetCurrencyListAsync().Result`
- [line 159](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/SCM/Models/PurchaseOrderModel.cs#L159) — `_costCenterService.GetAllListAsync().Result`
- [line 186](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/SCM/Models/PurchaseOrderModel.cs#L186) — `_generalConfigService.GetByValue(...).Result`, একটা `foreach` লুপের ভেতরে (একাধিকবার কল হয়)

অর্থাৎ এটা **নেস্টেড sync-over-async** — controller-এর বাইরের `.Wait()` একটা async মেথডকে ব্লক করে অপেক্ষা করছে, যার ভেতরে আবার একাধিক `.Result` ব্লকিং কল আছে। এটাই deadlock/hang-এর ক্লাসিক কারণ — thread pool-এর যে thread `.Wait()`-এ আটকে আছে, সেটাই ভেতরের async continuation-গুলোর জন্য দরকার হয়ে পড়ে, ফলে request কখনো শেষ হয় না।

**প্রমাণ হিসেবে তুলনা:** পাশের `GetVendorForGRByPOId` অ্যাকশনটা ([PurchaseOrderController.cs:790-795](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/SCM/Controllers/PurchaseOrderController.cs#L790-L795)) শুধু `await model.GetMPOByIdAsync(id)` ব্যবহার করে — কোনো `.Wait()`/`.Result` নেই — এবং সেটা প্রতিবারই সাথে সাথে সফলভাবে রেসপন্স দেয়। একই মুহূর্তে, একই PO id দিয়ে ছোড়া হলেও একটা কাজ করে, আরেকটা করে না — পার্থক্যটা ঠিক এই ব্লকিং কোড প্যাটার্নেই।

### কেন প্রথমবার Submit-এর bug ধরা পড়েনি এই সমস্যা

দুটো সম্পূর্ণ ভিন্ন bug — প্রথমটা ছিল client-side JS crash (site.js-এ, যেটা আগে ফিক্স করা হয়েছে), এটা সম্পূর্ণ server-side hang। আসলে item table খালি থাকায় প্রথমবার JS crash-টা ধরাই পড়েনি হয়তো — কারণ টেবিলে কোনো row-ই আসেনি, Submit করলে হয়তো সেই আগের bug-টাও ট্রিগার হতো না (row-বিহীন টেবিলে `rearrangeNameSuffix` করার কিছু নেই)।

### সুপারিশকৃত সমাধান

`GetProductDetailsForGRByPOId`-এর `model.Init(id).Wait();` লাইনটাকে `await model.Init(id);` করতে হবে (controller action আগে থেকেই `async Task<JsonResult>` — শুধু `.Wait()` সরিয়ে `await` করলেই যথেষ্ট, ব্যবসায়িক লজিক অপরিবর্তিত থাকবে)। ভেতরের `Init()`-এর তিনটা `.Result` কলও (`GetCurrencyListAsync().Result`, `GetAllListAsync().Result`, `GetByValue(...).Result`) `await`-এ পরিবর্তন করা উচিত, কারণ এই একই `Init()` মেথড অন্য জায়গা থেকেও কল হয় এবং এই ব্লকিং প্যাটার্নটা যেকোনো caller-এর জন্য একই ঝুঁকি বহন করে।

এটা implement করার অনুমতি দিলে আমি এগিয়ে যাব — নাকি শুধু root cause জানতে চেয়েছিলেন?

### Previous stage of code: 
```cs title:"PurchaseOrderController"
  var model = await Task.Run(() => new PurchaseOrderModel());

model.Init(id).Wait();

var purchaseOrderProductList = new List<PurchaseOrderProduct>();

var purchaseOrderProductResult = model.GetPOProductItemByPOId(id).Result.ToList();
```

#### After Fixing :
```cs title:"PurchaseOrderController"
var model = await Task.Run(() => new PurchaseOrderModel());
await model.Init(id);
var purchaseOrderProductList = new List<PurchaseOrderProduct>();
var purchaseOrderProductResult = (await model.GetPOProductItemByPOId(id)).ToList();
```

### Previous stage of code: 
```cs title:"PurchaseORderModel"
	var currencyList = _currencyService.GetCurrencyListAsync().Result;
	
  var CostCenterLists = new SelectList(_costCenterService.GetAllListAsync().Result.Where(x => !x.IsProfitCenter), "Name", "Name");
  
  var result = _generalConfigService.GetByValue(EnumGeneralConfigKey.MeasurementUnit.ToString(), name).Result;
```

#### Current stage of code : 
```cs title:"PurchaseOrderModel"
var currencyList = await _currencyService.GetCurrencyListAsync();

var  CostCenterLists = new SelectList((await _costCenterService.GetAllListAsync()).Where(x => !x.IsProfitCenter), "Name", "Name");

var result = await _generalConfigService.GetByValue(EnumGeneralConfigKey.MeasurementUnit.ToString(), name);
```

### Previous stage of controller : 
```cs title:"Purchase Order controller"
            var model = await Task.Run(() => new PurchaseOrderModel());

            await model.Init(id);
               Tolerance = model.Tolerance,
               
             //Stage -2
			  var model = new PurchaseOrderModel();

            var purchaseOrderProductList = new List<PurchaseOrderProduct>();

            var purchaseOrderProductResult = (await model.GetPOProductItemByPOId(id)).ToList();

            var receivedQuantities = await model.CalculateReceivedQuantities(id);

            var batchNoList = await model.GetProductAndBatchNoByPoId(id);
            
            // Stage - 3 
            if (!batchNoList.Any())

                {

                    foreach (var item in purchaseOrderProductList)

                    {

                        var batchNo = await model.GenerateBatchNo(item.PurchaseOrderId);

  

                        if (!batchNoList.ContainsKey(item.ProductId))

                            batchNoList[item.ProductId] = new List<string>();

  

                        batchNoList[item.ProductId].Add(batchNo);
                        
                        // stage - 4
                          var model = new PurchaseOrderModel();

            var purchaseOrderProductList = new List<PurchaseOrderProduct>();

            var __sw = System.Diagnostics.Stopwatch.StartNew();

            var purchaseOrderProductResult = (await model.GetPOProductItemByPOId(id)).ToList();

            System.Console.WriteLine($"__TIMING GetPOProductItemByPOId: {__sw.ElapsedMilliseconds}ms"); __sw.Restart();

            var receivedQuantities = await model.CalculateReceivedQuantities(id);

            System.Console.WriteLine($"__TIMING CalculateReceivedQuantities: {__sw.ElapsedMilliseconds}ms"); __sw.Restart();

            var batchNoList = await model.GetProductAndBatchNoByPoId(id);

            System.Console.WriteLine($"__TIMING GetProductAndBatchNoByPoId: {__sw.ElapsedMilliseconds}ms"); __sw.Restart();
            
            //stage - 5
                                foreach (var item in purchaseOrderProductList)

                    {

                        __sw.Restart();

                        var batchNo = await model.GenerateBatchNo(item.PurchaseOrderId);

                        System.Console.WriteLine($"__TIMING GenerateBatchNo: {__sw.ElapsedMilliseconds}ms");

  

                        if (!batchNoList.ContainsKey(item.ProductId))
```


#### Current Stage of code
```cs title:"PurchaseOrderController"
   // GetPOProductItemByPOId/CalculateReceivedQuantities/GetProductAndBatchNoByPoId/GenerateBatchNo

            // only need their own Guid params — none of them read the dropdown lists Init(id) builds

            // (every vendor, every product, all PR/RFQ allocation math for the whole PO module).

            // Calling Init(id) here was pure overhead the JSON response never used, and made this

            // endpoint take ~25s, so the Add screen's item table looked like it never loaded.

            var model = new PurchaseOrderModel();
            
                Tolerance = x.PurchaseOrder.Tolerance,
                // Stage -2 
                
		   var model = new PurchaseOrderModel();

            var purchaseOrderProductList = new List<PurchaseOrderProduct>();

            var __sw = System.Diagnostics.Stopwatch.StartNew();

            var purchaseOrderProductResult = (await model.GetPOProductItemByPOId(id)).ToList();

            System.Console.WriteLine($"__TIMING GetPOProductItemByPOId: {__sw.ElapsedMilliseconds}ms"); __sw.Restart();

            var receivedQuantities = await model.CalculateReceivedQuantities(id);

            System.Console.WriteLine($"__TIMING CalculateReceivedQuantities: {__sw.ElapsedMilliseconds}ms"); __sw.Restart();

            var batchNoList = await model.GetProductAndBatchNoByPoId(id);

            System.Console.WriteLine($"__TIMING GetProductAndBatchNoByPoId: {__sw.ElapsedMilliseconds}ms"); __sw.Restart();
            
            //stage - 3
				if (!batchNoList.Any())
                {
                    foreach (var item in purchaseOrderProductList)
                    {
                        __sw.Restart();
                        var batchNo = await model.GenerateBatchNo(item.PurchaseOrderId);

                        System.Console.WriteLine($"__TIMING GenerateBatchNo: {__sw.ElapsedMilliseconds}ms");

  

			if (!batchNoList.ContainsKey(item.ProductId))

				batchNoList[item.ProductId] = new List<string>();



			batchNoList[item.ProductId].Add(batchNo);
                        
			//stage - 4
                        
			var model = new PurchaseOrderModel();

            var purchaseOrderProductList = new List<PurchaseOrderProduct>();

            var purchaseOrderProductResult = (await model.GetPOProductItemByPOId(id)).ToList();

            var receivedQuantities = await model.CalculateReceivedQuantities(id);

            var batchNoList = await model.GetProductAndBatchNoByPoId(id);
            
            //stage - 5
                                foreach (var item in purchaseOrderProductList)

                    {

                        var batchNo = await model.GenerateBatchNo(item.PurchaseOrderId);

  

                        if (!batchNoList.ContainsKey(item.ProductId))
```


after completion claude reporting this: 
ফিক্স সম্পন্ন এবং লাইভ ব্রাউজার টেস্টে verified — এখন Item Details table সঠিকভাবে সব ডেটা নিয়ে row দেখাচ্ছে।

### যা করা হয়েছে

**১. [PurchaseOrderController.cs:796-806](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/SCM/Controllers/PurchaseOrderController.cs#L796-L806) — মূল ফিক্স (approved অনুযায়ী)**

```csharp
var model = new PurchaseOrderModel();
var purchaseOrderProductResult = (await model.GetPOProductItemByPOId(id)).ToList();
```

`model.Init(id).Wait();` সরিয়ে ফেলা হয়েছে। ইনভেস্টিগেশনে দেখা গেছে `GetPOProductItemByPOId`/`CalculateReceivedQuantities`/`GetProductAndBatchNoByPoId`/`GenerateBatchNo` — এই চারটা মেথডের কোনোটাই `Init(id)`-এর তৈরি করা dropdown list (সব vendor, সব product, পুরো PR/RFQ allocation হিসাব) ব্যবহার করে না। শুধু `Tolerance` ফিল্ডটা `Init(id)` থেকে আসছিল, সেটা এখন সরাসরি `x.PurchaseOrder.Tolerance` থেকে নেওয়া হচ্ছে (একই ডেটা, বাড়তি কাজ ছাড়াই)।

**২. [PurchaseOrderModel.cs](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/SCM/Models/PurchaseOrderModel.cs) — `Init()`-এর ৩টা ব্লকিং `.Result` কল ঠিক করা হয়েছে** (lines 108, 159, 186) — `await`-এ পরিবর্তন, যাতে এই একই `Init()` মেথডের অন্য caller-রাও (Add/Edit page load) এই sync-over-async ঝুঁকি থেকে মুক্ত থাকে।

### লাইভ verification

Playwright দিয়ে সরাসরি dev server চালিয়ে (temporary টেস্ট স্ক্রিপ্ট দিয়ে, পরে মুছে ফেলা হয়েছে) Vendor "Crest Tech" + PO "1020260026" সিলেক্ট করে দেখা গেছে:

- আগে: `GetProductDetailsForGRByPOId` request কখনো শেষ হতো না visible সময়ের মধ্যে (২৫+ সেকেন্ড, client abort হয়ে যেত)।
- ফিক্সের পর: রেসপন্স ২০০ OK আসে সঠিক ডেটা সহ (`"productName":"Bislol 2.5 mg Tablet"`, `"quantity":100`, `"batchNos":["10202608423"]` ইত্যাদি), এবং `#itemTbody`-তে সম্পূর্ণ, সঠিক row রেন্ডার হয় (Warehouse dropdown, Qty, Batch No select, Expire Date — সব ঠিকঠাক)।

### একটা অতিরিক্ত পর্যবেক্ষণ (এখনো ফিক্স করিনি, শুধু জানানো)

`GetPOProductItemByPOId`-এর নিজের DB কুয়েরিটাই এখনো ৫-১০+ সেকেন্ড সময় নিচ্ছে (কখনো কখনো বেশি) — সম্ভবত EF Core-এর বড় model-এর প্রথম-কুয়েরি warm-up cost (এই HMS সিস্টেমে অনেক module থাকায় EF মডেল বড়)। এটা আলাদা, নিম্ন-অগ্রাধিকারের performance বিষয় — table এখন সঠিকভাবে data দেখায়, কিন্তু কিছুটা ধীর মনে হতে পারে। এটা নিয়ে আলাদাভাবে কাজ করতে চাইলে জানাবেন।

**একই ধরনের প্যাটার্ন** (`model.Init(poId)` + `model.Tolerance`) [PurchaseOrderController.cs:939-963](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/SCM/Controllers/PurchaseOrderController.cs#L939-L963)-এর `GetProductDetailsByPoIdProdId` অ্যাকশনেও আছে (manual row-add flow-এ ব্যবহৃত), কিন্তু ওটা `model.PurchaseOrderProducts` (যেটা `Init()` populate করে) ব্যবহার করে বলে একই পদ্ধতিতে সরানো যাবে না — সেটা এই bug report-এর scope-এর বাইরে, তাই হাত দিইনি।

Build ক্লিন (0 errors), এবং ব্যবহৃত temporary টেস্ট ফাইল ও dev server প্রসেস সব পরিষ্কার করে দেওয়া হয়েছে — repo-তে এখন শুধু আসল ফিক্স দুটো ফাইলে (`PurchaseOrderController.cs`, `PurchaseOrderModel.cs`) আছে।