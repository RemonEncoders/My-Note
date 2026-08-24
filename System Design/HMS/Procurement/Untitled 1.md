# Gap Analysis: RFQ Quantity Allocation from Purchase Requisition (PR)

I have identified another business gap in the current RFQ Add/Edit implementation.

## Current Scenario

### Purchase Requisition (PR)

|Item|Requested Qty|
|---|--:|
|Laptop|10|

### RFQ-1

|RFQ|Item|Qty|
|---|---|--:|
|RFQ-001|Laptop|2|

Remaining quantity:

```
10 - 2 = 8
```

### Current Problem

The system currently allows creating another RFQ like this:

|RFQ|Item|Qty|
|---|---|--:|
|RFQ-002|Laptop|10|

Total RFQ Quantity:

```
2 + 10 = 12
```

But the original PR requested only:

```
10
```

So,

```
Requested Qty = 10
Total RFQ Qty = 12
```

This is logically inconsistent because the RFQ quantity exceeds the requested quantity in the originating Purchase Requisition.

---

# Why this is a Business Problem

A Purchase Requisition represents the business requirement.

An RFQ is created to obtain supplier quotations against that requirement.

Therefore, the total quantity allocated across all active RFQs for a PR item should never exceed the requested quantity.

Business Rule:

```
Σ(All Active RFQ Qty)
<=
PR Requested Qty
```

---

# Expected Flow

PR

```
Laptop = 10
```

RFQ-1

```
Laptop = 2
```

Remaining

```
8
```

When creating RFQ-2:

Allowed

```
1
2
3
...
8
```

Not Allowed

```
9
10
20
```

Example

```
Already RFQ = 2
New RFQ = 9

Total = 11

11 > 10

❌ Validation Failed
```

---

# Proposed ERP-Oriented Design

Instead of only validating during RFQ Save, I think the system should expose the allocation status to the user.

A realistic ERP dashboard/page could show:

For every PR Item:

- Requested Quantity
    
- RFQ Allocated Quantity
    
- Remaining Quantity
    
- PO Quantity
    
- Received Quantity
    
- Current Status
    

Example

|Item|Requested|RFQ Allocated|Remaining|PO|Received|
|---|--:|--:|--:|--:|--:|
|Laptop|10|2|8|0|0|

This gives users complete visibility before creating another RFQ.

---

# RFQ Creation Experience

When a user creates an RFQ from a PR, the UI should display:

```
Item : Laptop

Requested Qty : 10

Already Allocated to RFQ : 2

Remaining Qty : 8

Maximum Allowed : 8

RFQ Qty : [________]
```

The user immediately understands how much quantity is still available.

---

# Server-side Validation

Before saving:

```
AlreadyAllocatedRFQQty
+
CurrentRFQQty
<=
RequestedQty
```

Example

```
Requested Qty = 10

Already Allocated = 2

Current RFQ = 10

2 + 10 = 12

12 > 10

❌ Validation Failed
```

Error Message

```
Requested quantity exceeded.

Remaining quantity is 8.
```

---

# Database Calculation

The remaining quantity should never be stored directly.

Instead, calculate it dynamically.

```
RemainingQty =
RequestedQty
-
SUM(All Active RFQ Item Quantities)
```

Example

```
RequestedQty = 10

RFQ1 = 2

Remaining = 8
```

Validation

```
NewRFQQty <= RemainingQty
```

---

# Design Discussion

I would like your opinion on the best ERP-grade design.

Should we:

### Option A

Keep the current RFQ flow and only add quantity validation during RFQ Save.

### Option B

Introduce a PR Allocation Dashboard that tracks:

- Requested Qty
    
- RFQ Qty
    
- Remaining Qty
    
- PO Qty
    
- Received Qty
    

and use it as the source of truth for RFQ creation.

### Option C (or another better ERP design)

If there is a more scalable architecture used by enterprise ERP systems (SAP, Oracle, Microsoft Dynamics, Odoo, etc.), please propose that approach and explain its advantages, trade-offs, required database changes, UI flow, business rules, and validation strategy.

My goal is to implement this using an ERP-standard design rather than just fixing the immediate validation issue.