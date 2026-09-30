আপডেট: G9 design analysis শেষ। শুধু read-only code/docs/SQL ব্যবহার করেছি। কোনো ফাইল, schema, DB, RBAC, test বা git-এ কিছু বদলাইনি (`git status --short` খালি)। G11–G14 ও C1 সমাধান করিনি।

# A. Current-state evidence

## A1. Domain/code যা আছে
| বিষয় | প্রমাণ |
|---|---|
| **Service** | `MedicalService` ([MedicalService.cs](HMS.Entities/Areas/SVC/MedicalService.cs)): `ItemTypeId`, `MedicalServiceCategoryId`, `DepartmentId`, `LabTestId?`, `RadStudyTypeId?`, `DoctorId?`, `ServiceDuration`, `IsVitalCollection`। কোনো capability/StationType field নেই। Verdict §৮.১ নিজেও লিখেছে "Capability field নেই"। |
| **Package composition** | `HckPackageVersionItem` ([file](HMS.Entities/Areas/HCK/HckPackageVersionItem.cs)): `ServiceId`, `Quantity`, `IsRequired`। G6a `PhaseHint`-কে "G9-G12"-এর জন্য ইচ্ছাকৃতভাবে বাদ রেখেছে (line 11)। |
| **Item snapshot** | `HckScreeningItem`: `ServiceId`, `IsRequired`, `Disposition`, `Fulfilment`, `SourcePackageVersionItemId`। `JourneyStepId` nullable, FK ছাড়া, "reserved slot G9-G12" ([file](HMS.Entities/Areas/HCK/HckScreeningItem.cs)) |
| **Station master** | `HckStation`: `StationType` (enum), `RoomUnitId?`, `Capacity`, `BackupStationId?`, `Sequence`। |
| **Station type enum** | `EnumHckStationType` = ৯টি: Vitals, Phlebotomy, ECG, Echo, Ultrasound, XRay, Eye, Dental, Doctor। |
| **Journey row** | `HckScreeningStation`: `StationId` (non-null), `Sequence`, `Status`। কোনো service/capability/item link নেই। |
| **Enum-এর অন্য ব্যবহার** | `HckScreeningResult.StationType` (result category, report/corporate grouping)। [HckResultService.cs:96,145,160,185](HMS.Services/Areas/HCK/HckResultService.cs#L96) hard-coded `Vitals`/`Phlebotomy`। |
| **পুরনো service→room mapping (অন্য module)** | `ServiceLocationMap` (FACILITY) + `RouteSlipGenerator` (ROUTING): service বা category → Room/RoomUnit, effective-dated, Priority। HCK docs এটার উল্লেখ করেনি। |

## A2. All-stations journey কোথায় তৈরি হয়
`BookIndividualAsync` (version pin) → `ConfirmBookingAsync` ([HckScreeningService.cs:214-230](HMS.Services/Areas/HCK/HckScreeningService.cs#L214-L230)): `_uow.StationRepository.GetAllOrderedAsync()` → প্রতিটি active station-এর জন্য একটি `HckScreeningStation` row (`StationId = st.Id`, `Sequence = seq++`)। এই loop-এ `HckScreeningItem`/package কোথাও ব্যবহার হয় না। ঠিক তার পরের লাইনেই ([:238-257](HMS.Services/Areas/HCK/HckScreeningService.cs#L238-L257)) item snapshot তৈরি হয়, কিন্তু দুটোর মধ্যে কোনো সংযোগ নেই। এটা confirm transaction-এর ভেতরেই হয়। Verdict §৮.৩ এটাকে check-in-এ সরানোর কথা বলেছে, সেটা C1, এখানে decide করছি না।

## A3. HMSDb3 data (read-only, শুধু local dev seed)
- **Stations:** ৯টি, প্রতিটি `StationType`-এ ১টি করে। সবার `RoomUnitId` ও `BackupStationId` null।
- **HealthCheck package (GroupType=4):** ১২টি। সব item `MedicalServiceId`-based, `LabTestId`-only item ০টি।
- **Package-এর distinct non-lab service ৯টি:** Chest X-Ray, ECG, Echocardiogram, Ultrasound Whole Abdomen, Treadmill Test (TMT), Bone Densitometry, Dental Check, Vision/Eye Check, Physician Consultation। এই ৯টির সবই `ItemType = Procedure`। কোনোটির category বা department নেই। `RadStudyTypeId` সব null, `DoctorId` null, `IsVitalCollection` কোনো service-এই নেই।
- **Package-এ Rad-mapped service ০টি।** পুরো DB-তে rad-mapped ১১টি (মোট ১৫৫১ service)।
- **RIS modality:** CR, CT, DXA, MG, MR, NM, US। ECG বা Echo modality নেই।
- **Lab services:** package-এ ১১টি distinct। ২টির `Specimen` = Blood (Venous), ৯টির specimen নেই।
- **Location map:** ১৪৫৩ row, সব service-level। ৯টির মধ্যে Chest X-Ray, ECG, Echo, TMT-এর ১টি করে, USG-র ২টি। Bone Densitometry, Dental, Vision/Eye, Physician Consultation-এর ০টি।
- **Vitals:** কোনো package-এ Vitals service নেই।

**পর্যবেক্ষণ (সিদ্ধান্ত নয়):** Planning §৬/D2 বলেছে "RadStudyTypeId ECG/Echo/US/XRay আলাদা করতে পারে না"। এই DB-তে বাস্তবতা আরও সরু: ECG ও Echo RIS modality-ই নয়, আর package-এর কোনো imaging service-এ `RadStudyTypeId` নেই। TMT ও Bone Densitometry-র সাথে মেলে এমন কোনো `EnumHckStationType` মান নেই।

# B. G9 design options (তুলনা, র‍্যাঙ্কিং নয়)

**Verdict-এ যা আগে থেকে ঠিক আছে (🔒 LOCKED):** তিন-স্তর vocabulary Service / Capability (abstract) / Station (physical); generator শুধু `RequiredCapability` লেখে, `StationId` নয়; pipeline `RequiredCapability → eligible (Capability→[Station]) → …`। **Verdict §৮.৩** "v1 capability shortcut: HckCapability master এখন নয়, `StationType` reuse" লিখেছে। এটা locked list-এ নেই, আর Planning §৬ বলেছে এটা "একটি unbuilt dependency-র ওপর দাঁড়ানো"। G9 নিজেই OPEN।

## Option A — `EnumHckStationType`-ই capability
- **Domain অর্থ:** capability = station type। ৯টি মান।
- **Schema:** `RequiredCapability` column enum (int)। Capability master নেই। Service→capability map আলাদাভাবে লাগবে (D6)।
- **Migration/data:** কম। Planning F8: পরে `HckCapability` FK-তে গেলে live journey row-এ enum→FK migration।
- **Extensibility:** নতুন capability মানে enum মান বাড়ানো (code + migration)। TMT, Bone Densitometry-র মতো capability-র মান আজ নেই।
- **Existing HCK data:** `HckScreeningResult.StationType` একই enum বহন করে। enum বাড়ালে/অর্থ বদলালে result grouping, report ও corporate aggregate প্রভাবিত।
- **Journey Generator:** সরল `switch/map`, তবে capability ও station type একই জিনিস ধরা হয়, তাই Capability→[Station] map কার্যত "একই type-এর station"।
- **ঝুঁকি:** capability ও station type-এর সমার্থক ধরা, উপরের data-gap (TMT/DEXA), F8 migration।

## Option B — `HckCapability` (+ `HckStationCapability`)
- **Domain অর্থ:** capability আলাদা master। Station এক বা একাধিক capability serve করে।
- **Schema:** নতুন ২টি entity/table, `JourneyStep.RequiredCapability` FK। Verdict §৮.৩ এটাকে 🟡 (phased later) বলেছে, আর "map StationType-এর বেশি লাগলে 🔴" (Planning F3)।
- **Migration/data:** master seed, `HckStationCapability` seed, ৯টি station-এর mapping। পরে enum→FK migration লাগে না।
- **Extensibility:** নতুন capability data-হিসেবে যোগ হয়, multi-site/multi-capability station সম্ভব।
- **Existing HCK data:** `HckScreeningResult.StationType` অপরিবর্তিত থাকে (আলাদা concern)।
- **Journey Generator:** capability master ও Capability→[Station] map explicit।
- **ঝুঁকি:** Phase-2-এর scope বড়, Verdict "এখন নয়" বলা জিনিস আগে আনা; master data ও maintenance UI ছাড়া অকার্যকর।

## Option C — enum-ই v1 value-set, কিন্তু column আলাদা ও স্পষ্ট "future FK"
- **Domain অর্থ:** `RequiredCapability` নামের আলাদা column (Option A-র মান-সেট), `HckStation.StationType` থেকে আলাদা রাখা; ভবিষ্যতে FK-তে যাওয়ার পথ নথিভুক্ত।
- **Schema:** A-র মতো, শুধু naming/typing স্পষ্ট। F8-এর নিজস্ব শর্ত ("plan the column deliberately") এটাই।
- **অন্যান্য প্রভাব:** A-র মতো, তবে `HckScreeningResult.StationType`-এর সাথে coupling কমে না, আলাদা নামে শুধু স্পষ্ট হয়।

## Service→capability mapping কোথায় থাকতে পারে (D6-র জন্য তথ্য)
1. `MedicalService`-এ নতুন column: SVC shared table, অন্য মডিউলকে প্রভাবিত করে, verdict §৮.১ "Keep + classification" বলেছে।
2. `HckPackageVersionItem`-এ (per-version): G6a-তে বাদ রাখা, version-এ snapshot, সব item-এ ম্যানুয়াল।
3. HCK-নিজস্ব map table (ServiceId → capability): package/version থেকে স্বাধীন, HCK-owned।
4. `HckScreeningItem`-এ Stage-1 snapshot হিসেবে copy: G8/INV-G3.1-এর "copy once" দর্শনের সাথে মেলে। বাকি ৩টির যেকোনোটির সাথে যুক্ত হতে পারে।

# C. Owner-এর সিদ্ধান্ত লাগবে
নিচের F সারণিতে। এগুলো ছাড়া বাকি সব হয় ইতিমধ্যে locked, নয়তো অন্য gate-এর।

**Already locked (নতুন সিদ্ধান্ত চাইছি না):**
- Capability একটি abstract স্তর, Station থেকে আলাদা।
- Generator `StationId` লেখে না।
- একাধিক item একই capability share করতে পারে ("1 step : N item" grouping, Verdict Stage 2)। grouping-এর predicate নিজে G12।
- Capability→[Station] "eligible stations" pipeline (Verdict §৮.৭)।
- Station-resolution service `StationId`-এর একমাত্র writer।

**Deferred (G9-এ সমাধান করিনি):**
- **G12:** structural steps (Registration, Check-in, Specimen Collection, Doctor consolidation, Report handoff), phase/parallel, "single-encounter feasibility"।
- **G11:** Replan।
- **G13, G14, C1।**
- Station resolution-এর "no eligible station", auto-redirect, manual override (Planning §৬ [MISSING])। এগুলো কোনো নির্দিষ্ট G-number পায়নি, তাই আলাদা করে চিহ্নিত রাখলাম।
- `ServiceLocationMap`/`RoomUnitId` runtime station resolution-এ ব্যবহার হবে কি না।

# D. প্রস্তাবিত G9 lock wording (খসড়া, `[...]` = owner উত্তরের স্লট)

> **G9 — Service → Capability → Station mapping · 🔒 LOCK PENDING**
>
> 1. একটি `HckScreeningItem`-এর `ServiceId` থেকে একটি `RequiredCapability` নির্ধারিত হয়। Capability একটি abstract operational need; Station physical। (Verdict Journey-vocabulary lock অপরিবর্তিত।)
> 2. **Capability abstraction:** `[D1: EnumHckStationType-এর মান-সেট, `RequiredCapability` নামের আলাদা column | নতুন `HckCapability` master]`।
> 3. **Cardinality:** একটি service-এর capability সংখ্যা `[D2: ০..১ | ০..N]`; একাধিক service একই capability ধারণ করতে পারে (আগে থেকে locked)।
> 4. **Capability→Station:** `[D3: station-এর StationType-এর সমতা থেকে derived | explicit HckStationCapability map]`; eligible-station pipeline অপরিবর্তিত।
> 5. **Physical station প্রয়োজন নেই এমন service:** `[D4: capability = null/none হলে কোনো patient-facing step নেই | explicit "no-station" capability মান]`।
> 6. **Mapping-এর মালিকানা ও binding সময়:** `[D6: MedicalService-এ | HckPackageVersionItem-এ | HCK-নিজস্ব map table | Stage-1-এ HckScreeningItem-এ snapshot | সংমিশ্রণ (উল্লেখ করুন)]`।
> 7. **Unmapped service:** `[D7: version authoring-এ block | confirm-এ block | mapped নয় বলে step ছাড়া (audit সহ) | অন্য]`।
> 8. **Existing data:** `[D5: seed data backfill প্রয়োজন কি না, কোন উৎস থেকে]`।
> 9. Generator কখনো `StationId` লেখে না (আগে থেকে locked)। G9 grouping, ordering, structural steps, replan, fulfilment trigger বা idempotency নির্ধারণ করে না।

# E. Implementation boundary

**G9 লক হলে পরে অনুমোদন করবে (আলাদা authorization সাপেক্ষে):**
- Service→capability mapping data model ও storage।
- Capability→Station relationship-এর storage।
- সংশ্লিষ্ট migration/backfill ও authoring/maintenance পথ।

**G9 অনুমোদন করবে না:**
- `IJourneyPlanner`/Journey Generator implementation।
- `JourneyStep`-এ `RequiredCapability`/`StationId` nullable-করার schema রূপান্তর।
- `HckScreeningStation` বা `GetAllOrderedAsync()` loop বদলানো।
- Structural steps, phase/ordering (G12), Replan (G11), Fulfilment trigger (G13), idempotency (G14), check-in বনাম Activate (C1)।
- Station selection strategy বা runtime resolution।
- Data-isolation (F1) aggregation পুনর্লিখন।

**Journey Generator-এ G9 কোথায় জোগান দেবে (design-level):** `GenerateInitial(screening, ctx)` প্রতিটি `HckScreeningItem.ServiceId` পড়বে → G9 map → item প্রতি `RequiredCapability` → (G12 grouping/phase) → `JourneyStep.RequiredCapability`। Layer 2 আলাদাভাবে `RequiredCapability → Capability→[Station]` ব্যবহার করবে (G9-D3)।

# F. REQUIRED DECISION SUMMARY

| Decision ID | Decision required | Options / values | Evidence supporting the decision | Impact if selected | Recommendation status |
|---|---|---|---|---|---|
| G9-D1 | Capability abstraction | (a) `EnumHckStationType`-এর মান-সেট, আলাদা `RequiredCapability` column; (b) নতুন `HckCapability` master (+ `HckStationCapability`) | Verdict §৮.৩ v1 shortcut বনাম §৮.৩ 🟡; Planning §৬ ও F3, F8। ৯টি station ৯ enum মানের সাথে ১:১। TMT ও Bone Densitometry-র সাথে মেলে এমন মান নেই। `HckScreeningResult.StationType` একই enum ব্যবহার করে। | (a): নতুন capability = enum মান + migration; পরে FK-তে গেলে live journey row-এ enum→FK migration; result/report grouping-এর সাথে coupling থাকে। (b): নতুন entity, seed, maintenance; enum→FK migration নেই; Phase-2 scope বড়। | Owner decision |
| G9-D2 | Service → Capability cardinality | ০..১ / ০..N (M:1 আগে থেকে locked, নতুন প্রশ্ন নয়) | Verdict Stage 2 (CBC: patient-facing দিক = Specimen Collection, lab-processing দিকে patient step নেই)। ৯টি non-lab service-এর প্রত্যেকটি স্বাভাবিকভাবে একটি capability-র। ১:N-এর কোনো বাস্তব উদাহরণ data-তে নেই। | ০..১: সরল map, একটি service একটি step-source। ০..N: একটি service একাধিক capability/step-source দিতে পারে, grouping ও `Fulfilment`-এর সাথে G12/G4-এর মিথস্ক্রিয়া বাড়ে। | Owner decision |
| G9-D3 | Capability → Station relationship (কীভাবে persist) | (a) station-এর `StationType`-এর সমতা থেকে derived (এক station = এক capability); (b) explicit `HckStationCapability` map (এক station = একাধিক capability সম্ভব) | Verdict pipeline "Capability→[Station] map" locked। আজ প্রতি type-এ ঠিক ১টি station, সবার `RoomUnitId`/`BackupStationId` null। `ServiceLocationMap` অন্য module-এ service→room map রাখে। | (a): অতিরিক্ত table নেই, কিন্তু D1(a)-এর সাথেই বাঁধা। (b): নতুন table ও seed; D1-এর যেকোনো উত্তরের সাথে সঙ্গত। | Owner decision |
| G9-D4 | Physical station প্রয়োজন নেই এমন service | (a) capability null/none = step নেই; (b) explicit "no-station" মান; (c) (স্বাভাবিক উদাহরণ: lab processing পাশ, structural step-রা G12-র) | Verdict Stage 2: lab processing-এ patient-facing step নেই। Vitals/Check-in/Report handoff structural, package service নয় (G12)। Package data-তে Vitals service নেই। | (a): null-handling সব জায়গায়। (b): নতুন মান, দ্ব্যর্থহীন কিন্তু enum/master-এ অতিরিক্ত entry। Structural steps এই সিদ্ধান্তের বাইরে, G12-তে। | Owner decision |
| G9-D5 | Existing data / backfill | (a) কোনো backfill নয়, নতুন mapping শুধু নতুন data-র জন্য; (b) বিদ্যমান ১২ package-এর ৯টি non-lab service (ও lab service) backfill; (c) Verdict G1 Option D (wipe & reseed) অনুযায়ী re-seed | G1: Option D (wipe & reseed) locked। বিদ্যমান ৯টি service সব `Procedure`, category/dept ছাড়া; শুধু নামই আলাদা করে। ১১টি lab service-এর ৯টির specimen নেই। | (a): পুরনো package mapping-হীন, D7-এর আচরণ কার্যকর হবে। (b): ম্যানুয়াল/seed data-কাজ, DB-write authorization লাগবে। (c): নিয়ম G1-এর সাথে সঙ্গত, কিন্তু scope owner-নির্ধারিত। | Owner decision |
| G9-D6 | Mapping-এর মালিকানা ও binding সময় | (a) `MedicalService`-এ column; (b) `HckPackageVersionItem`-এ; (c) HCK-নিজস্ব map table (ServiceId→capability); (d) Stage-1-এ `HckScreeningItem`-এ snapshot; (a/b/c-এর সাথে (d) যুক্ত হতে পারে) | Verdict §৮.১ "MedicalService: Keep + classification"; G6a-তে `PhaseHint` বাদ; G8/INV-G3.1 "copy once, immutable"; `MedicalService` SVC-shared। | (a): অন্য module-কে স্পর্শ করে; (b): version-এ সাথে immutable ও snapshot, প্রতি item ম্যানুয়াল; (c): HCK-স্বাধীন, live edit-এর ঝুঁকি; (d): booking-এ frozen, পরে map বদলালে বিদ্যমান booking প্রভাবিত নয়, কিন্তু নতুন `HckScreeningItem` field লাগে। | Owner decision |
| G9-D7 | Unmapped service-এর আচরণ | (a) version authoring-এ block; (b) confirm-এ block; (c) step ছাড়া চালু (audit সহ); (d) অন্য | Planning §৬ [MISSING] কেস ও F3: কোনো ব্যাখ্যা নেই। বিদ্যমান ৯টি service আজ কোনো map ছাড়া। | (a): authoring স্ক্রিন আরও কঠোর, বিদ্যমান version-এর সাথে সংঘাত হতে পারে। (b): booking-এ দেরিতে ধরা পড়ে। (c): clinical step নীরবে বাদ পড়ার ঝুঁকি। | Owner decision |

# Owner approval required

G9 লক করার আগে এই ID-গুলোর উত্তর লাগবে: **G9-D1, G9-D2, G9-D3, G9-D4, G9-D5, G9-D6, G9-D7**।

তিনটি নির্ভরতা: D3(a) D1(a)-এর ওপর নির্ভর করে। D5 ও D7 পরস্পর প্রভাবিত। D6-র উত্তর D5-এর কাজের ধরন ঠিক করে।

এখানে থামলাম। Markdown, code, schema, DB, tests, migration, RBAC, commit বা push কিছুই করিনি।