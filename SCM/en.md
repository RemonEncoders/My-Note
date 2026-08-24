# Supply Chain (SCM) — Procurement Guide

**Who is this for?** Department Requesters (who create PRs), Approvers/Managers, Procurement Officers, Store/Warehouse In-charges.

**What it covers:** The full procurement chain — from needing something, through Purchase Requisition and approval, an optional RFQ/Tender, a Purchase Order, and finally Goods Receipt.

> ℹ️ When goods are received, the system automatically creates a Journal Entry and sends it to Accounting (debiting the Inventory GL, crediting the GR/IR Clearing GL) — an Accountant must review and Post it. See the **[Accounting](../Accounting/en.md)** guide.

> 🖼️ *[Screenshot: SCM procurement dashboard]*

---

## The whole chain, at a glance

```
Purchase Requisition (PR)  →  [optional] RFQ/Tender  →  Purchase Order (PO)  →  Goods Receipt (GRN)
   Draft → Approval →            Publish → Evaluate →      Direct or                Stock comes in,
   Approved                      Award                      from Tender             GL auto-posts
```

The RFQ/Tender step is **optional** — used only when competitive bidding is needed. For smaller/faster purchases, a Direct PO can be created straight from the PR.

---

## Task 1 — Create a Purchase Requisition (PR)

**Menu:** Supply Chain → **Purchase Requisition** → **Add**  ·  `/SCM/PurchaseRequisition`

1. Fill in the PR's lines — item (product), quantity, expected currency/price.
2. **Save** leaves the PR in **Draft** — freely editable while in Draft.
3. Once ready, **Submit for Approval** (Task 2).

---

## Task 2 — Submit for approval

Clicking **Submit for Approval** makes the system look up an **Approval Policy** matching the PR's **Currency** and **Total Price**.

- No matching policy → Submit is blocked; ask an Admin to create one.
- A match found → the **Approval Workflow** tied to that policy is "snapshotted" onto the PR — later edits to the policy won't retroactively change this PR's approval steps.
- Status becomes **Pending Approval**, and it waits stage by stage (Stage 1, 2, 3…) for someone holding that stage's Role to approve.

> ℹ️ The PR's own creator can approve their own PR if they hold the required Role (self-approval is allowed).

---

## Task 3 — Approve / Reject a PR

**Menu:** Supply Chain → **Pending PR Approvals**  ·  `/SCM/PurchaseRequisition/PendingApprovals`

1. Shows PRs waiting on your Role.
2. **Approve** moves it to the next stage; on the last stage the PR becomes **Approved** — RFQ/PO creation is now possible.
3. **Reject** requires a reason; the PR's owner can edit and resubmit it (starting a new round from stage 1).

Every approve/reject is recorded in the **Purchase Approval** audit trail — `/SCM/PurchaseApproval`.

### PR status

| Status | Meaning |
|--------|---------|
| **Draft** | Being built, freely editable. |
| **Pending Approval** | Approval workflow in progress. |
| **Approved** | All stages passed; ready for RFQ/PO. |
| **Rejected** | Stopped at some stage; can be edited and resubmitted. |
| **Closed** | Fully ordered, or manually closed. |
| **Cancelled** | Called off entirely. |

---

## Task 4 — Manage Vendors (including blacklisting)

**Menu:** Supply Chain → **Vendor**  ·  `/SCM/Vendor`

1. **Add/Edit** vendor details; toggle **Active/Inactive**.
2. If there's an issue with a vendor, **Set Blacklist** with a reason — a blacklisted vendor can't win *new* RFQ awards or POs, but their existing open POs remain editable.

> ℹ️ An older **Supplier** master also exists — it's legacy; use **Vendor** for all current work.

---

## Task 5 — Float an RFQ/Tender (when you need competitive bidding)

**Menu:** From an Approved PR's Details page, click **Float RFQ** (`/SCM/RFQComparison/Add?prId=...`) — the PR's items and their live allocation load automatically.

1. Select which vendors should be asked to quote.
2. **Publish** it — the RFQ is now open for vendors to submit quotes.
3. **Extend Deadline** (with a reason) before the closing date if needed.
4. Once quotes start coming in, **Start Evaluation** — status becomes **Under Evaluation**.
5. **Shortlist** (toggle) the quotes you like, for comparison.

---

## Task 6 — Award the Tender

From the RFQ's Details page:

1. **Award Tender** — this is **single-winner**: the whole tender goes to one vendor. Requirements — that vendor must have quoted *every* item, must not be blacklisted/disqualified, and no item can already be locked into another PO.
2. **Clear Tender Award** fully undoes an award if it was a mistake.
3. To disqualify a vendor later, **Disqualify Vendor** — all their won items return to the pool.
4. If no bid is satisfactory, **Close Without Award**, or **Cancel** the whole RFQ. Both are final — the same RFQ cannot be reopened; a new RFQ must be created.

> ℹ️ Per-item award (different items of the same tender going to different vendors) used to be possible and the data model still supports it, but the current UI uses the **single-winner (whole tender to one vendor)** approach.

### RFQ/Tender status

| Status | Meaning |
|--------|---------|
| **Draft** | Being built, not yet sent to vendors. |
| **Published** | Vendors can submit quotes. |
| **Under Evaluation** | Quotes received, being compared. |
| **Awarded / Partially Awarded** | A vendor (or vendors) has been awarded. |
| **Cancelled / Closed Without Award** | Finally closed, cannot be reopened. |

---

## Task 7 — Create a Purchase Order (PO) — two routes

**Direct PO** (no RFQ) — **Menu:** Supply Chain → **Purchase Order** → **Add**

1. Pick a PR, choose its still-remaining items, and pick a Supplier/Vendor.
2. The system checks: the PR is approved, the vendor isn't blacklisted, and all items on the PO are of the same category (all Goods or all Service — no mixing).

**Tender-sourced PO** (from an RFQ award) — from the RFQ Details' **Create PO** button, or the **Create From RFQ** worklist (`/SCM/PurchaseOrder/CreateFromRFQ`) — this lists awarded tenders not yet turned into a PO, grouped by winning vendor.

1. The vendor and tender are **locked** to the award — cannot be changed.
2. Items come in at the winning bid prices, showing only the ones with RFQ Remaining still available.

> 💡 As many **Draft** POs as you like can be created — they don't lock any PR/RFQ quantity. Quantity is only actually **reserved/locked** when the PO is made **Final**.

---

## Task 8 — Finalise, Cancel/Void, or Reopen a PO

1. Once a Draft PO looks right, click **Final** — this reserves the corresponding PR/RFQ quantity (over-allocation shows an error).
2. If no longer needed, **Cancel PO** (or **Void**) — the reserved quantity is released again.
3. To fix a mistake on a Final PO, **Reopen** it — it goes back to Draft; edit and Finalise again.

### PO status

| Status | Meaning |
|--------|---------|
| **Draft** | Being built, does not yet reserve PR/RFQ quantity. |
| **Final** | Confirmed; quantity reserved; Goods Receipt can be taken against it. |
| **Completed** | Every line fully received — the system sets this automatically. |
| **Cancelled / Void** | Called off; reserved quantity released. |

---

## Task 9 — Goods Receipt (receiving the delivery)

**Menu:** Supply Chain → **Goods Receive**

1. Pick a Final PO and enter what quantity actually arrived.
2. The system blocks over-receipt — a PO line's total received quantity can never exceed its ordered quantity (accounting for quantity already claimed on any other live receipt).
3. Finalising the receipt auto-generates a (Draft) GL entry into Accounting at that moment.
4. Once every line on the PO is fully received, the PO auto-becomes **Completed**; reversing a receipt later reopens it.

> 🖼️ *[Screenshot: Goods Receipt entry against a Final PO]*

---

## Task 10 — What "Remaining" means (understanding allocation)

The PR/RFQ screens show **Requested / Ordered / Open RFQ / Remaining** columns — don't be confused, the rule is simple:

| Column | Meaning |
|--------|---------|
| **Requested** | Total quantity asked for on the PR. |
| **Ordered** | Quantity already on a **Final/Completed** PO (Draft POs don't count). |
| **Open RFQ** | Quantity tied up in a live (Published/Under Evaluation/Awarded) tender that isn't a PO yet. |
| **Remaining** | Requested − Ordered − Open RFQ — the quantity still available for a new RFQ/PO. |

The system re-checks this math again at the moment a PO is made Final — if two people try to claim the same quantity at once, whichever finalises first wins, and the other gets an error.

---

## Handover

Once Goods Receipt happens, that GL entry sits in Accounting as **Draft**, awaiting review.

➡️ Go to the **[Accounting](../Accounting/en.md)** guide to review and Post it.

> ✅ That covers the entire procurement chain, from PR to Goods Receipt.
