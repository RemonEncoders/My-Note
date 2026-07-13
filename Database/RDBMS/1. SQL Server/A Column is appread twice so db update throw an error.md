# Problem : 
what does it mean : 
Column names in each table must be unique. Column name 'PurchaseRequisitionId' in table 'scm_purchase_orders' is specified more than once.  

# Solution : 
এই Error-এর অর্থ হলো **একই টেবিলের মধ্যে `PurchaseRequisitionId` নামে দুইটি (বা তার বেশি) কলাম তৈরি করার চেষ্টা করা হয়েছে**, যা SQL Server অনুমতি দেয় না।

### সহজ ভাষায়

ধরুন `scm_purchase_orders` টেবিলে আপনি এমন কিছু করতে চাইছেন:

|Id|PurchaseRequisitionId|VendorId|PurchaseRequisitionId|
|---|---|---|---|

এটি সম্ভব নয়, কারণ একই টেবিলে **একই নামের দুইটি কলাম থাকতে পারে না**।

তাই SQL Server এই Error দেখাচ্ছে:

```text
Column names in each table must be unique.
Column name 'PurchaseRequisitionId' in table 'scm_purchase_orders' is specified more than once.
```

বাংলায় এর অর্থ:

> **`scm_purchase_orders` টেবিলে `PurchaseRequisitionId` নামে একাধিক কলাম তৈরি করার চেষ্টা করা হয়েছে। কিন্তু একটি টেবিলে একই নামের কলাম একবারই থাকতে পারে।**

---

## কেন এই Error হয়?

সাধারণত নিচের কারণগুলোর মধ্যে একটি হয়।

### ১. একই Property দুইবার Mapping হয়েছে (সবচেয়ে সাধারণ)

উদাহরণ:

```csharp
public Guid PurchaseRequisitionId { get; set; }

public PurchaseRequisition PurchaseRequisition { get; set; }
```

এবং Fluent API-তে ভুলভাবে আবার Foreign Key তৈরি করা হয়েছে।

---

### ২. Fluent API-তে একই Foreign Key দুইবার Configure করা হয়েছে

যেমন:

```csharp
builder.HasOne(x => x.PurchaseRequisition)
       .WithMany()
       .HasForeignKey(x => x.PurchaseRequisitionId);
```

আবার অন্য কোথাও একই সম্পর্ক Configure করা হয়েছে।

---

### ৩. Data Annotation এবং Fluent API-এর Conflict

যেমন:

```csharp
[ForeignKey("PurchaseRequisition")]
public Guid PurchaseRequisitionId { get; set; }
```

এবং আবার Fluent API-তেও একই FK Configure করা হয়েছে।

---

### ৪. Migration-এ একই Column দুইবার Add করা হয়েছে

উদাহরণ:

```csharp
migrationBuilder.AddColumn<Guid>(
    name: "PurchaseRequisitionId",
    table: "scm_purchase_orders",
    ...);
```

এরপর আবার

```csharp
migrationBuilder.AddColumn<Guid>(
    name: "PurchaseRequisitionId",
    table: "scm_purchase_orders",
    ...);
```

---

## আরেকটি সম্ভাবনা

যদি `scm_purchase_orders` টেবিলে **আগেই** `PurchaseRequisitionId` নামে একটি Column থেকে থাকে (আগের Migration বা Database-এ), আর নতুন Migration আবার সেটি `AddColumn()` করতে চায়, তাহলেও একই ধরনের সমস্যা হতে পারে।

চেক করুন:

```sql

SELECT 
	COLUMN_NAMEFROM 
INFORMATION_SCHEMA.COLUMNS
WHERE 
	TABLE_NAME = 'scm_purchase_orders';
	
```

যদি ফলাফলে `PurchaseRequisitionId` আগে থেকেই থাকে, তাহলে এই Migration সেটি আবার যোগ করার চেষ্টা করছে।

---

আপনার SQL Query-এর ফলাফল:

```table title:SQL Result
	COLUMN_NAME
----------------------------
	PurchaseRequisitionId
	
```

এর মানে হলো **`scm_purchase_orders` টেবিলে `PurchaseRequisitionId` নামে কলাম ইতিমধ্যেই আছে।**

এখন এটি খুব গুরুত্বপূর্ণ তথ্য।

### এর মানে কী?

আপনার Migration আবার এই কাজটি করছে:

```csharp
migrationBuilder.AddColumn<Guid>(
    name: "PurchaseRequisitionId",
    table: "scm_purchase_orders",
    ...
);
```

কিন্তু Database-এ একই নামের Column আগে থেকেই আছে। তাই SQL Server বলছে:

> **"Column name 'PurchaseRequisitionId' is specified more than once."**

---

## এখন কী করবেন?

প্রথমে জানতে হবে **কেন কলামটি আগে থেকেই আছে**।

### Scenario 1: Migration আগে Apply হয়ে গেছে (সবচেয়ে সাধারণ)

যদি এই Migration আগে সফলভাবে Run হয়ে থাকে, তাহলে আবার Run করার দরকার নেই।

চেক করুন:

```sql
SELECT * 
FROM __EFMigrationsHistory;
```

দেখুন `UpdateRurchaseOrderByPurchaseRequisitionId` (বা কাছাকাছি নামের) Migration সেখানে আছে কি না।

---

### Scenario 2: Column Database-এ Manual-ভাবে তৈরি করা হয়েছে

যদি কেউ SQL দিয়ে আগে থেকেই Column যোগ করে থাকে, তাহলে EF-এর Migration আবার সেটি যোগ করতে চাইছে।

---

### Scenario 3: আগের Migration-এই Column যোগ হয়েছে

হতে পারে অন্য কোনো Migration-এ ইতিমধ্যেই এই লাইন ছিল:

```csharp
migrationBuilder.AddColumn<Guid>(
    name: "PurchaseRequisitionId",
    table: "scm_purchase_orders",
    ...
);
```

এবং এখন নতুন Migration আবার একই Column যোগ করছে।

---

## আমি এখন যে তথ্যটি জানতে চাই

দয়া করে নিচের SQL Query চালিয়ে ফলাফল দেখান:

```sql
SELECT MigrationId
FROM __EFMigrationsHistory
ORDER BY MigrationId;
```

এবং আরেকটি Query চালান:

```sql
SELECT COLUMN_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'scm_purchase_orders'
ORDER BY COLUMN_NAME;
```

এছাড়া, **`PurchaseOrderConfiguration`** (বা `IEntityTypeConfiguration<PurchaseOrder>`) ক্লাসটিও শেয়ার করুন।

এই তথ্যগুলো দেখলে আমি নিশ্চিতভাবে বলতে পারব সমস্যাটি:

- Migration duplicate,
    
- Database duplicate,
    
- নাকি EF Core-এর relationship mapping-এর কারণে হচ্ছে।
## আপনার ক্ষেত্রে কী কী Check করবেন?

১. `PurchaseOrder` Entity-তে `PurchaseRequisitionId` Property কি একবারই আছে?

২. `PurchaseOrderConfiguration`-এ কি `HasForeignKey()` একই Property দিয়ে দুইবার লেখা হয়েছে?

৩. সর্বশেষ Migration (`Up()` method)-এ কি `PurchaseRequisitionId` দুইবার Add হচ্ছে?

৪. Navigation Property-এর জন্য EF Core কি Shadow Foreign Key তৈরি করছে?

---

যদি আপনি নিচের তিনটি কোড শেয়ার করেন, তাহলে আমি নির্দিষ্ট করে বলতে পারব ঠিক কোথায় সমস্যা হচ্ছে:

1. `PurchaseOrder` Entity
    
2. `PurchaseOrderConfiguration` (বা Fluent API Configuration)
    
3. যে Migration-এ এই Error এসেছে (`Up()` method)