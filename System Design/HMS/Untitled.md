# RFQ → PO Enhancement Implementation Instructions

Please implement the following requirements carefully. These requirements are **only for the RFQ → Purchase Order flow** and **must not break or modify the existing PR → PO business rules**.

---

# 1. Scope

The existing **Direct PR → PO** workflow is already working correctly and **must remain unchanged**.

All new business rules described below apply **only** to the RFQ → PO workflow (CreateFromTender / EditFromTender).

Do not modify the existing PR validation logic unless absolutely necessary.

---

# 2. RFQ Quantity Consumption Rule

An RFQ Item may be fulfilled through multiple Purchase Orders.

The business rule is:

```
Remaining Qty = RFQ Qty − Σ(Committed PO Qty)
```

Where **Committed PO Qty** includes only Purchase Orders that are business-committed.

An RFQ Item is considered completed only when:

```
Remaining Qty == 0
```

Until then, additional Purchase Orders may still be created for the remaining quantity.

The system must never allow:

```
Σ(Committed PO Qty) > RFQ Qty
```

---

# 3. Which Purchase Orders Consume RFQ Quantity

Only business-committed Purchase Orders consume RFQ quantity.

Include:

- Approved
    
- Confirmed
    
- Issued
    
- Any equivalent business-committed status
    

Do NOT consume quantity for:

- Draft
    
- Rejected
    
- Cancelled
    
- Deleted
    

If a previously committed Purchase Order becomes Cancelled/Rejected/Deleted (according to the business workflow), its quantity must be released automatically and become available again.

---

# 4. Draft Purchase Orders

Draft Purchase Orders are **work in progress only**.

They DO NOT reserve quantity.

Creating a Draft PO must never reduce RFQ Remaining Quantity.

---

# 5. Draft Confirmation Validation

When a Draft PO is going to be Confirmed/Approved/Issued, the system MUST validate the latest remaining quantity again.

Validation:

```
Draft Qty <= Current Remaining Qty
```

If validation succeeds:

- Confirm PO
    
- Consume quantity
    

If validation fails:

Reject the confirmation.

Keep the Purchase Order in Draft status.

Do not automatically modify the quantity.

The user must manually revise the Draft and confirm again.

---

# 6. User-Friendly Validation Messages

If only partial quantity remains:

Example:

```
Remaining Qty = 2
Draft Qty = 5
```

Display:

> Only 2 units remain available. Your draft requests 5 units. Please reduce the quantity to 2 or less before confirming.

---

If no quantity remains:

Example:

```
PR Qty = 10
Already Committed = 10
Remaining = 0
Draft Qty = 10
```

Display:

> This Purchase Order can no longer be confirmed because the requested quantity has already been committed by other confirmed Purchase Orders. No quantity is currently available. Please revise or cancel this draft.

Do not use generic validation messages.

---

# 7. Ajax Validation Before Confirmation

Before changing a Draft Purchase Order to a committed status:

Perform an Ajax validation using the latest database values.

Flow:

```
Click Confirm

↓

Read Current Remaining Qty

↓

Validate

↓

Success

or

Friendly Validation Message
```

This avoids unnecessary page reloads and provides a better user experience.

---

# 8. Mixed RFQ (Service + Product)

A single RFQ may contain both Product and Service items.

However, a single Purchase Order may contain only one category.

Business Rules:

If PO Category = Service

→ Show only Service Items.

If PO Category = Local Goods / Asset

→ Show only Product Items.

Backend validation must enforce this rule.

Do not rely only on frontend filtering.

---

# 9. Multiple Purchase Orders

A single RFQ may generate multiple Purchase Orders.

Example:

```
RFQ

Laptop = 10

↓

PO1 = 4

↓

PO2 = 3

↓

PO3 = 3
```

The system must always calculate Remaining Quantity using committed Purchase Orders.

---

# 10. Create PO Availability

The Create PO action should only be available when:

```
Awarded Vendor

AND

Remaining Qty > 0
```

If Remaining Qty becomes zero,

that RFQ (or awarded vendor) should no longer appear in any "Create PO from RFQ" list.

---

# 11. Fully Ordered Status

When all awarded quantities for an awarded vendor have been committed through Purchase Orders,

the awarded vendor should be treated as:

```
Fully Ordered
```

Such vendors should not appear in Create PO lists anymore.

---

# 12. Single Source of Truth

Avoid duplicate business logic.

Use a single aggregate calculation:

```
Ordered Qty = Σ(Committed PO Qty)
Remaining Qty = RFQ Qty − Ordered Qty
```

This single calculation should be reused by:

- Tender Item Source
    
- Remaining Quantity Validation
    
- RFQ Remaining Calculation
    
- Create PO Availability
    
- Fully Ordered Detection
    

---

# 13. Existing PR → PO Flow Must Remain Intact

This enhancement must NOT break:

- Direct PR → PO
    
- Existing PR Remaining Validation
    
- Existing Formula M
    
- Existing business behaviour
    

Only RFQ → PO should receive these new capabilities.

---

# 14. Regression Scenarios

Please verify all of the following scenarios after implementation:

- Direct PR → PO
    
- RFQ → Single PO
    
- RFQ → Multiple Partial POs
    
- Mixed Product + Service RFQ
    
- Draft → Confirm
    
- Draft confirmation failure because remaining changed
    
- Cancelled PO releases quantity
    
- Rejected PO does not consume quantity
    
- Deleted PO does not consume quantity
    
- Edit Confirmed PO
    
- Edit Draft PO
    
- Fully Ordered RFQ disappears from Create PO list
    
- Remaining Quantity recalculates correctly
    
- Backend category validation
    
- Concurrent users confirming Purchase Orders
    

---

# 15. Implementation Goal

The final solution should:

- Preserve all existing PR → PO behaviour.
    
- Support multiple partial Purchase Orders from a single RFQ.
    
- Prevent over-ordering.
    
- Treat Draft as work-in-progress only.
    
- Consume quantity only after business commitment.
    
- Provide clear validation messages.
    
- Be concurrency-safe.
    
- Avoid duplicate business logic.
    
- Use a single source of truth for Remaining Quantity.
    
- Maintain a clean, extensible architecture suitable for future enhancements.