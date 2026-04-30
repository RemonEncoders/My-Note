# 🧠 সহজভাবে বুঝো

C# চায়: **default parameter value (ডিফল্ট মান)** হিসেবে শুধু **compile-time constant** গ্রহণ করে।

> “এই ভ্যালুটা কোড compile হওয়ার সময়ই ফিক্সড থাকতে হবে”

---

# ❌ কিন্তু DateTime.UtcNow কী?

```cs
DateTime.UtcNow
```

👉 এটা রানটাইমে (program চলার সময়) জেনারেট হয়  
👉 প্রতিবার আলাদা সময় দেয়

অর্থাৎ:

- এটা constant না
- এটা dynamic value

👉 তাই C# এটাকে default parameter হিসেবে allow করে না

---

# ⚠️ তাই error কেন হচ্ছে?

কারণ তুমি বলছো:

> “compile time এ current time set করো”

কিন্তু C# বলে:

> “না, আমি শুধু fixed value রাখতে পারি”

---
# ❌ তুমি যেটা লিখেছো (সমস্যা কোথায়)

```
DateTime timeStamp = DateTime.UtcNow
```

👉 এখানে সমস্যা হলো:


# ✅ সঠিক সমাধান (তোমার জন্য সবচেয়ে ভালো)

তুমি এটা করো:

```cs
DateTime? timeStamp = null
```

তারপর method এর ভিতরে:

```cs
timeStamp ?? DateTime.UtcNow
```

---

# 💡 পুরো সঠিক কোড (সহজভাবে)

```cs
private EntityOperationResponse<T> EntityResponse<T>(
    T data = default,
    DateTime? timeStamp = null,
    string message = "",
    EnumOperationStatus status = EnumOperationStatus.NotFound,
    EnumEntityOperation operationType = EnumEntityOperation.Update
)
{
    return new EntityOperationResponse<T>
    {
        Data = data,
        TimeStamp = timeStamp ?? DateTime.UtcNow,
        Message = message,
        Operation = operationType,
        Status = status
    };
}
```

---

# 🧠 সহজ করে মনে রাখো

|জিনিস|অনুমতি|
|---|---|
|10, false, "text"|✔ allowed|
|DateTime.UtcNow|❌ not allowed|
|new Guid()|❌ not allowed|
|default(T)|✔ allowed|
