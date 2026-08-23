# HCK — Health Check / Preventive Packages

**Module code:** `HCK`
**Area folder:** [HMS.Web/Areas/HCK/](../../HMS.Web/Areas/HCK/)
**Sidebar group:** *Health Check* · icon `fa fa-stethoscope` · order `17`
**Status:** 🟡 ~15% — stub controllers; flow board prototype live

---

## 1. Overview

HCK owns **preventive health packages** — Basic, Executive, Premium, Cardiac Premium, Women's Wellness, Diabetic Care, Pre-Marital, Pre-Employment, Senior Citizen. Individuals or corporates book; patients flow through a station-based journey (vitals → phlebotomy → ECG → Echo → USG → X-ray → Eye → doctor) and receive a consolidated health report. Corporate accounts receive de-identified aggregate snapshots for HR action.

Primary users: **Doctor**, **Nurse**, **Counsellor** (booking + corporate). HCK pulls orders into [LIS](LIS.md) and [RIS](RIS.md) the same way regular consultations do.

---

## 2. Menu & Submenu

```
Health Check (HCK)
├── Dashboard                           → /HCK/HckDashboard
├── Health Screening
│   └── Screening Journey               → /HCK/HealthScreening
├── Vaccination
│   └── Vaccination Drive               → /HCK/VaccinationDrive
└── Insights
    └── Reports                         → /HCK/HckReports
```

### URL Reference

| Route | Page | What it's for |
|-------|------|---------------|
| `/HCK/HckDashboard` | Dashboard | 6 KPIs — bookings today, in-progress, awaiting doctor review, reports issued MTD, corporate bookings, revenue MTD. |
| `/HCK/HealthScreening` | Health Screening Journey | Combined booking + check-in + live patient flow board + station queues + results aggregation + doctor review + final report generation. |
| `/HCK/VaccinationDrive` | Vaccination Drive | Vaccine drive management — schedule, inventory, dose tracking, certificate generation. |
| `/HCK/HckReports` | Reports | 9 reports — Daily Throughput, Package Mix, Station Bottleneck, Corporate Revenue, Critical Findings, Follow-Up Compliance. |

> The prototype covers 7 menu groups (Catalog, Bookings, Patient Journey, Results, Corporate, Insights). In the current build they are tabs / sub-views of the 4 controllers above.

---

## 3. Permissions

| Slug | Scope |
|------|-------|
| `hck` | Module root |
| `hck.screening` | Health Screening Journey |
| `hck.vaccination` | Vaccination Drive |

No dedicated C# permission constants.

---

## 4. Roles with Access

| Role | View | Add | Edit | Delete | Notes |
|------|:---:|:---:|:---:|:---:|-------|
| SuperAdmin | ✅ | ✅ | ✅ | ✅ | Full |
| Admin | ✅ | ✅ | ✅ | ⛔ | — |
| Doctor | ✅ | ✅ | ✅ | ⛔ | **Primary** — doctor review + final report |
| Nurse | ✅ | ✅ | ✅ | ⛔ | Vitals + phlebotomy station work |
| Counsellor | ✅ | ✅ | ✅ | ⛔ | Bookings + corporate accounts |

Slug pattern: `hck`, `pcare`.

---

## 5. Key Features

- **12 active packages** card catalog with price + duration + stations + inclusions.
- **Package Builder** for custom corporate packages with live total / cycle time computation.
- **3 booking modes** — Individual, Corporate Single, Corporate Batch.
- **Live Flow Board** — per-patient horizontal stepper across 9-11 stations; TV-friendly.
- **Station queue** auto-suggestion engine opens backup station when bottleneck forms.
- **Results aggregation** color-coded (Normal / Borderline / Abnormal) with age/gender-aware ranges.
- **Critical findings auto-route** to specialists and jump to top of doctor review queue.
- **8-page consolidated PDF** emailed to patient + corporate HR.
- **Corporate Snapshot** — de-identified aggregate for HR (privacy-protected) with quarterly PDF.
- **Follow-up tracker** auto-schedules doctor-recommended tests with SMS reminders.

---

## 6. User Journeys (Full Workflow)

### Journey A — Individual Executive Package

**Actors:** Patient → Receptionist → Nurse → Doctor · **Trigger:** Patient books online or walk-in · **Outcome:** Consolidated health report emailed.

1. Booking created on `/HCK/HealthScreening` (Individual mode) → pre-visit SMS with fasting + prep instructions.
2. Patient arrives → check-in verifies NID + booking → wristband + personalised station map printed.
3. Patient flows through stations (Vitals → Phlebotomy → ECG → Echo → USG → Eye → Doctor); each station scan updates `/HCK/HealthScreening` flow board.
4. Labs land in [LIS](LIS.md), images in [RIS](RIS.md); results aggregate on `/HCK/HealthScreening` patient page.
5. Doctor review queue — doctor opens patient → reads results → writes opinion + recommendations.
6. 8-page PDF generated; emailed to patient; follow-ups auto-scheduled on tracker.

### Journey B — Corporate Batch (100 employees)

**Actors:** Corporate HR → Counsellor → Hospital Team · **Trigger:** Annual corporate drive · **Outcome:** All 100 screened; HR snapshot delivered.

1. Counsellor on `/HCK/HealthScreening` (Corporate Batch) — uploads CSV of 100 employees + package.
2. System allocates slots over N days; SMS invites + schedule sent; calendar-invites optional.
3. Each employee flows as individual; reports emailed to employee + corporate HR.
4. At end of drive, `/HCK/HealthScreening` Corporate Snapshot compiles de-identified aggregate.
5. HR receives quarterly PDF; privacy-protected — no individual data.

### Journey C — Critical Finding Escalation

**Actors:** Doctor + Specialist · **Trigger:** Severely abnormal result (e.g., HbA1c 13, cardiac ischaemia) · **Outcome:** Immediate specialist referral.

1. Result auto-flags critical on `/HCK/HealthScreening` aggregation.
2. Priority Review button pushes to top of doctor review queue.
3. Doctor reviews within minutes → writes specialist referral; patient called if not on-site.
4. Referral creates [ACARE](ACARE.md) specialist appointment; follow-up tracked.

### Journey D — Station Bottleneck Handling

**Actors:** HCK Coordinator · **Trigger:** Queue depth exceeds threshold at a station · **Outcome:** Backup station opened; flow restored.

1. `/HCK/HealthScreening` flow board shows Echo queue > 6 patients → auto-alert.
2. Auto-suggestion engine recommends opening backup Echo station B.
3. Coordinator confirms; staff reassigned; queue drains.
4. Bottleneck event feeds `/HCK/HckReports` Station Bottleneck analytics.

### Journey E — Package Builder (new offering)

**Actors:** Counsellor + Medical Director · **Trigger:** New package offering requested · **Outcome:** Package published to catalog.

1. Counsellor on `/HCK/HealthScreening` Package Builder → picks components from service catalog.
2. Live total price + cycle time + stations computed.
3. Medical Director approves; version pinned; package goes live on catalog.

### Journey F — Vaccination Drive

**Actors:** Vaccination Nurse · **Trigger:** Scheduled corporate / community drive · **Outcome:** Doses administered + certificates issued.

1. `/HCK/VaccinationDrive` — schedule drive, allocate vials, open registration.
2. Attendees check in; eligibility verified; dose administered; batch + lot captured.
3. Certificate auto-emailed; next-dose schedule set; adverse events monitored 30 min on-site.

### Journey G — Follow-Up Reminder Cycle

**Actors:** System + Counsellor · **Trigger:** Follow-up test due in 7 days · **Outcome:** Patient reminded, scheduled, compliant.

1. `/HCK/HealthScreening` Follow-Up tracker sends SMS 7 days before due date.
2. Day-0 email reminder; Day +3 phone-call attempt on overdue.
3. Appointment booked in [ACARE](ACARE.md); compliance tracked on `/HCK/HckReports`.

---

## 7. Related Modules

- [ACARE](ACARE.md) — patient registration + billing + follow-up appointments.
- [LIS](LIS.md) — labs from health-check package.
- [RIS](RIS.md) — imaging from package.
- [PCARE](PCARE.md) — doctor review environment.
- [CRM](CRM.md) — corporate account management.
- [PHAR](PHAR.md) — vaccines dispensing for drive.

---

## 8. Reference Docs

- Menu structure: [HMS.Web/docs/all/HCK_Menu_Structure.md](../../HMS.Web/docs/all/HCK_Menu_Structure.md)
- Task list: [HMS.Web/docs/all/HCK_Task_List.md](../../HMS.Web/docs/all/HCK_Task_List.md)
- Prototype: [HMS.Web/docs/all/HCK_Prototype.html](../../HMS.Web/docs/all/HCK_Prototype.html)
- Menu index: [HMS.Web/docs/HMS_All_Modules_Menu_Overview.md](../../HMS.Web/docs/HMS_All_Modules_Menu_Overview.md#hck--health-check--preventive-packages)
