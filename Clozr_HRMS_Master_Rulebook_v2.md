# Clozr HRMS — Master Rule Book (v2.0)
### Single Source of Truth for UI/UX Designers & Backend API Engineers

**HRMS is a module of the Clozr platform** — a peer of the CRM, Project, Helpdesk and Accounts modules — and this book follows the conventions of the *Clozr CRM Master Rule Book v1.0*. It supersedes HRMS Rule Book v1.0: that draft was built on assumptions; this version incorporates the answered product-decision questionnaire (55 questions, logged in §0). Archive v1.0.

Business logic is derived from an end-to-end analysis of the Frappe HR v17 reference implementation, adapted to the confirmed product decisions. Untagged rules reflect proven reference behaviour translated into product language.

**Out of scope (shared platform concerns, governed by the CRM rulebook):** subscription/billing, user roles, permission scoping, tenancy mechanics, user management. **Approval routing is business logic and is specified here** (§1.6): the approver is always the reporting manager.

**Structure:** **Part A** (§1–§17) is the Phase 1 build scope. **Part B** (§18–§26) specifies Phase 2 modules — kept in this book so the design leaves seams for them, but **do not build them now**.

**Legend for inline tags**
- `[STD]` — a detail your answers did not cover; filled with an industry-standard default. **Review and confirm** (consolidated in §27).
- `[ADAPT]` — reference behaviour adapted to the Clozr platform or to a confirmed decision. Review the mapping.
- `[UI]` — a note specifically for designers.
- `[API]` — a note specifically for backend/data engineers.

---

## 0. Confirmed Decision Log

The questionnaire answers, condensed. Q-numbers match the questionnaire for traceability.

| # | Decision |
|---|---|
| Q1 | Product = **HRMS module** of the Clozr platform (peer of CRM / Project / Helpdesk / Accounts). |
| Q2–Q3 | **Phase 1:** employee lifecycle, leave, attendance & check-ins, performance, daily work summary, expenses, travel, vehicles, **payroll**, ESS. **Phase 2:** income tax, flexible benefits, gratuity, recruitment, training & skills, roster suite, overtime & incentives. |
| Q4 | India only. |
| Q5–Q6 | One tenant = one company. **No Branch master.** |
| Q7–Q8 | **Departments and the reporting hierarchy already exist in the CRM module — HRMS reuses them, never recreates them.** |
| Q9 | Employee ID scheme tenant-configurable. |
| Q10 | Statuses Active/Inactive/Suspended/Left; **Suspended = app access blocked**. |
| Q11 | Every employee gets a platform login (ESS). |
| Q12 | HR captures basic details; employee **self-completes profile within 30 days of joining** (days configurable per tenant). |
| Q13 | **The reporting manager is the approver** for all requests. No separate approver fields, no department approver lists. |
| Q14–Q15 | Attendance regularisation: **no approval**. Travel request: **pure data capture**. |
| Q16 | Self-approval blocked by default. |
| Q17 | Leave year: tenant setting — **Fiscal (Apr–Mar) or Calendar (Jan–Dec)**. |
| Q18–Q19 | Seed leave **types only**; quantities set by the tenant admin. Carry-forward allowed on chosen types with caps, via settings. **Negative leave balances do not exist.** |
| Q20 | Half day is the smallest leave unit. |
| Q21–Q22 | Comp-off validity and leave-backdating policy are **tenant settings**. |
| Q23–Q24 | Attendance sources: manual, mobile geo check-in, biometric ingestion, CSV import. **Geofencing ON by default.** |
| Q25 | Auto-attendance thresholds & grace periods are settings (per shift). |
| Q26–Q27 | No multiple shifts per employee per day. Roster board → Phase 2. |
| Q28 | Overtime & incentives → Phase 2. |
| Q29–Q30 | **Monthly payroll only.** Basis = **Leave**; unmarked working days without leave are treated as **LOP** (§12.2.4). |
| Q31–Q32 | Statutory deductions (PF/ESI/PT/LWF) are **settings-driven**. **No income tax in Phase 1.** |
| Q33 | Salary slips are **in-app only** (no email PDFs). |
| Q34–Q35, Q46 | No gratuity/FBP in Phase 1. **INR only, everywhere** (no multi-currency). |
| Q36–Q37 | Accounting = **reports only** (Accounts-module integration is a seam). **Bank salary-file export is Phase 1.** |
| Q38–Q40 | Recruitment (careers portal, apply-by-email, offer links) → Phase 2. |
| Q41 | Performance Phase 1 = **manual rating only** (goal-based automation → Phase 2). |
| Q42 | Daily work summary kept, **adapted to in-app submission** (no inbound email) — droppable if the adaptation is unwanted. |
| Q43 | Vehicles module kept. |
| Q44–Q45 | Expense receipts **mandatory**. Advance recovery **via salary deduction** by default. |
| Q47–Q48 | ESS lives **inside the existing Clozr app**; the platform's push-notification infra is used. |
| Q49–Q50 | No WhatsApp for HR. Reminder emails all ON, sent only via the platform's existing SMTP service. |
| Q51 | Go-live imports: **employees, leave balances, attendance history** only. |
| Q52–Q53 | Biometric: **eSSL** devices. No public API. |
| Q54–Q55 | Keep reference terminology. Rounding/display conventions are settings. |

Remaining `[STD]` decisions awaiting sign-off are consolidated in **§27**.

---

# PART A — PHASE 1 (BUILD SCOPE)

## 1. Architecture & Cross-Module Ground Rules

1.1 HRMS is a module inside the Clozr multi-tenant platform: row-level tenancy, `tenant_id` on every table, ORM + RLS double enforcement per CRM rulebook §1. One tenant = one company (Q5); every rule the reference scoped by "company" is tenant-scoped here.

1.2 `[ADAPT]` **Shared platform masters (Q7/Q8).** HRMS **reuses** the platform's Department tree and the CRM reporting hierarchy (CRM rulebook §3.2 — materialized closure table). HRMS adds HR attributes *onto* them (e.g. a department's payroll cost center, leave block list) via extension tables — it never duplicates the org structure. The employee's manager IS the CRM reporting hierarchy parent.

1.3 **No Branch entity (Q6).** Wherever the reference filtered or scoped by branch, that dimension is dropped.

1.4 `[ADAPT]` **Record states.** Every transactional HRMS document carries `record_state ∈ {Draft, Submitted, Cancelled}` in addition to its domain `status`:
- **Draft** — fully editable.
- **Submitted** — locked/immutable; the document has taken effect (ledger entries, attendance, salary impact…). Editing requires **Cancel → Amend** (new draft linked via `amended_from`).
- **Cancelled** — all side effects reversed. Cancellation is blocked where downstream records depend on the document (each blocking rule listed in its section).
- `[API]` State transitions are the only way side effects fire; side-effect logic must be idempotent per document.

1.5 **Employee-centricity.** Every transaction references exactly one Employee. Global guards: no transaction may be created for an **Inactive** employee; date-range transactions clamp to `date_of_joining … relieving_date`. One shared validator enforces this for every entity.

1.6 **Approver = Reporting Manager (Q13, Q16).** For every request type (Leave, Expense Claim, and any future approval flow):
- The approver is the employee's **reporting manager** from the CRM hierarchy — resolved at request creation and stamped on the request.
- `[STD]` **Fallback:** if the employee has no manager (top of tree) or the manager is not Active, the request routes to the tenant's designated **HR fallback approver** (a tenant setting; defaults to the System Admin). **Confirm.**
- **Self-approval is blocked by default** (tenant setting `prevent_self_approval`, default ON, per request type). The fallback rule handles the top-of-tree case.
- `[API]` The request is auto-shared with the stamped approver (action rights on that one document); manager change re-stamps only **Draft/Open** requests. `[STD]`
- `[UI]` The approver field is displayed read-only on request forms (resolved, not chosen).

1.7 **Currency (Q35/Q46).** INR everywhere. No `currency`/`exchange_rate` fields, no base-amount twins. `[API]` Store amounts as INR decimals; display rounding per tenant settings (§17.2.7).

1.8 `[ADAPT]` **Accounting (Q36).** No GL postings in Phase 1. Every monetary event (expense sanction/reimbursement, advance payment/return, payroll accrual/disbursement, encashment payout, FnF settlement) is recorded as a status-tracked payment/settlement record and surfaced through **reports** (§12.10). Each such point is a named seam for the Accounts module (§26.1).

1.9 `[API]` **Holiday resolution is one shared service** (§2.4) consumed by leave, attendance, payroll, and reminders — never re-implemented per feature.

1.10 **ESS placement (Q47).** Employee self-service is a section **inside the existing Clozr app** (web + mobile), not a separate application. It reuses the platform's auth, navigation, notification and push infrastructure (Q48).

---

## 2. Organization & Shared Masters

### 2.1 Department (platform-owned, HR-extended)
2.1.1 The Department tree comes from the platform (§1.2). Disabled departments are excluded from HR pickers.
2.1.2 HR extension attributes per department: default **payroll cost center** (reporting dimension — §12.8.4), optional **leave block list** (§7.11).

### 2.2 Designation & Employee Grade
2.2.1 Designation carries: description and an optional **Appraisal Template** link (§9). *(Skills-per-designation → Phase 2 with Training & Skills.)*
2.2.2 Employee Grade carries a **default salary structure** and **default base pay** — used as defaults in bulk salary-structure assignment (§12.4.6).

### 2.3 Simple masters
2.3.1 Employment Type (seeded: Full-time, Part-time, Probation, Contract, Commission, Piecework, Intern, Apprentice), Health Insurance Provider, Identification Document Type, Grievance Type, Purpose of Travel, Vehicle Service Item, KRA, Employee Feedback Criteria. All tenant-scoped unique-name masters.

### 2.4 Holiday Lists & Assignment
2.4.1 A **Holiday List** = named set of dated holidays within a from/to range; each holiday flagged `weekly_off` or described public holiday; optional half-day flag (halves auto-attendance thresholds that day — §8.5.4).
2.4.2 **Holiday List Assignment** (record-state-bearing): `applicable_for ∈ {Employee, Company}`, assigned_to, holiday_list, from_date (must fall inside the list's own range; no duplicate target+from_date).
2.4.3 **Resolution for employee E on date D:** most recent submitted employee-level assignment with from_date ≤ D; else the tenant-level one; else error (or empty where tolerated). Effective end of an assignment = min(list's to_date, next assignment's from_date − 1).
2.4.4 **Gap-filling:** ranges not covered by employee-level assignments fall back to tenant-level; employee-level wins where present; range queries split at assignment boundaries and union holidays.
2.4.5 `[API]` Holiday lookups are cached; list/assignment changes synchronously invalidate the cache (payroll day-math depends on it).
2.4.6 `[UI]` Employees never pick holiday lists; HR assigns. Employees see a read-only Holidays list in ESS (§13.4).

---

## 3. Employee Master & Profile Completion

### 3.1 Identity, status, key fields
3.1.1 **Status enum: `Active | Inactive | Suspended | Left`.**
- **Active** — normal; receives reminders, accrues leave, transactable.
- **Inactive** — hard block on all new transactions.
- **Suspended (Q10)** — the linked platform user's **app access is blocked** (no ESS, no login-dependent actions); HR-side records/corrections remain possible; excluded from reminders and accruals.
- **Left** — exited; requires `relieving_date`; excluded from accruals, payroll pools (beyond final settlement), reminders.
3.1.2 **Employee ID (Q9)** — tenant setting `employee_naming_by ∈ {Naming Series (default), Employee Number, Full Name}`; series pattern tenant-configurable. Unset → error on first employee creation.
3.1.3 Key dates: date_of_birth, date_of_joining, relieving_date, resignation_letter_date, exit_interview_held_on, date_of_retirement (= DOB + tenant `retirement_age`, default 60).
3.1.4 Org placement: department, designation, grade, employment_type; **manager = CRM hierarchy parent** (§1.2 — not an HR-owned field). Compensation context: ctc. Attendance: default_shift, `attendance_device_id` (biometric key — §8.3.6). Payroll: payroll_cost_center.
3.1.5 **Internal Work History**: append-only rows {department, designation, from_date, to_date}, maintained **only** by Promotion/Transfer (§5.1.3).
3.1.6 **Platform user link (Q11):** every employee is provisioned a platform user for ESS at creation (or linked to an existing one). One user ↔ at most one Active employee per tenant. A logged-in user without an Active employee record sees a dedicated "no employee profile" state in the HRMS section.

### 3.2 Profile self-completion (Q12) `[ADAPT]`
3.2.1 HR creates the employee with **basic details** (name, contact, DOJ, department, designation, grade, employment type, manager via hierarchy).
3.2.2 The employee must **self-complete** the remaining profile (personal details, addresses, bank details, PAN/identification documents, emergency contact, education/work history) within `profile_completion_days` of joining — tenant setting, **default 30**.
3.2.3 `[UI]` ESS shows a profile-completion checklist with % complete and the deadline; incomplete mandatory groups are individually listed.
3.2.4 `[STD]` Enforcement: reminder notifications to the employee at 7 days before deadline, on deadline, and weekly after; HR is notified of overdue profiles (report + notification). No hard lock of ESS. **Confirm enforcement severity.**
3.2.5 `[API]` Which profile fields are HR-only vs self-editable vs self-editable-until-verified is a per-field flag in the employee schema config; bank/PAN edits after payroll onboarding raise an HR review task. `[STD]`

### 3.3 Cascades wired to the Employee record
3.3.1 Employee creation gates/back-syncs tied to onboarding (§4.2.4). *(Applicant/offer back-sync → Phase 2 with Recruitment.)*
3.3.2 Manager relationships, and therefore approver routing (§1.6), come live from the CRM hierarchy — HRMS performs no role grants of its own. `[ADAPT]`
3.3.3 `[UI]` Employee detail: grouped related-record links (Attendance, Leave, Lifecycle, Exit, Expense, Payroll, Evaluation) + 12-month attendance heat-map (Present + Half Day).

---

## 4. Employee Lifecycle — Onboarding & Separation

### 4.1 Templates & activities (shared engine)
4.1.1 **Onboarding Template** / **Separation Template** = reusable activity checklists, optionally scoped by department/designation/grade. Selecting a template copies its activities in.
4.1.2 Activity row: activity_name (required), assignee = **user XOR role**, begin_on (offset days), duration (days), task_weight, `required_for_employee_creation` (onboarding only), description.
4.1.3 On submit, the system generates one **project** with one **task per activity** (task start = begin date + offset; end = start + duration; dates landing on holidays roll forward to the next working day). `[ADAPT]` Projects/tasks are created in the platform's **Project module**. Assignees get to-dos; email only if `notify_users_by_email`.
4.1.4 `boarding_status` auto-derives from project completion: **Pending** (0%) → **In Process** (1–99%) → **Completed** (100%).
4.1.5 Activities may be appended after submission (tasks regenerate for new rows). Cancel deletes the generated project + tasks.

### 4.2 Employee Onboarding
4.2.1 Mandatory: candidate name, company/tenant, date_of_joining, boarding_begins_on. `[ADAPT]` With Recruitment in Phase 2, Phase 1 onboarding starts from a **manually entered candidate** (name + email); the applicant/offer links activate in Phase 2 (§18).
4.2.2 One onboarding per candidate (non-cancelled duplicate blocked, keyed by candidate email `[STD]`).
4.2.3 "Mark as Completed" force-completes all tasks + project.
4.2.4 **Employee-creation gate:** "Create Employee" only when the onboarding is submitted AND every `required_for_employee_creation` activity's task is Completed/Cancelled — else hard error listing pending tasks. Creation maps name/grade/personal email; status Active; triggers user provisioning (§3.1.6) and the profile-completion clock (§3.2).

### 4.3 Employee Separation
4.3.1 Mandatory: employee, separation_begins_on. Pulls resignation_letter_date and org fields from the employee.
4.3.2 Identical template/task/status engine (§4.1); holiday-shifting always uses the employee's holiday list.
4.3.3 Separation does not auto-create Exit Interview or FnF — independent records grouped on the employee's exit view.

---

## 5. Promotion, Transfer & Grievance

### 5.1 Property-diff engine (shared)
5.1.1 Promotion/Transfer carry a table of property changes {employee field, current, new}; the dialog fetches current values live and blocks duplicate-field and no-change rows.
5.1.2 Excluded from the picker: identity fields, status, gender, DOB, DOJ, ctc (explicit on promotion), non-data fields.
5.1.3 On submit each change applies to the Employee; **department/designation changes append an Internal Work History row** effective from the event date, auto-closing the previous row (to_date = new from_date − 1); first row seeded from current values + DOJ. Cancel reverts values and removes appended rows.
5.1.4 `[ADAPT]` A department change updates the employee's HR department; **it does not by itself move the employee in the CRM reporting hierarchy** — manager changes are made in the hierarchy (platform UI) and merely referenced here. A transfer/promotion that intends a manager change must include it as an explicit property row targeting the hierarchy. **Confirm interplay.** `[STD]`

### 5.2 Employee Promotion
5.2.1 Mandatory: employee, promotion_date; Inactive employees blocked; **cannot submit before the promotion date**.
5.2.2 If revised_ctc set → employee ctc updates on submit, reverts on cancel.

### 5.3 Employee Transfer
5.3.1 Mandatory: employee, transfer_date, ≥1 detail; cannot submit before transfer date.
5.3.2 Default: apply changes to the same employee. `create_new_employee_id` mode: clone to a new employee record, move the user link, relieve the old record (status Left, relieving_date = transfer_date). Cancel of a new-id transfer is blocked while the new record exists; afterwards the old record is restored Active.
5.3.3 Inter-company transfer does not exist (one tenant = one company).

### 5.4 Employee Grievance
5.4.1 Status: **Open (default) → Investigated → Resolved | Invalid | Cancelled**.
5.4.2 Mandatory: subject, raised_by, date, grievance_type, against-party (Company/Department/Grade/Employee — not self) + description.
5.4.3 Conditional mandatory: cause when Investigated/Resolved; resolved_by + resolution_date + resolution_detail when Resolved.
5.4.4 Only Resolved/Invalid can be submitted; discard → Cancelled.
5.4.5 `[UI]` Raisable via ESS; visibility restricted to raiser + HR.

---

## 6. Exit: Exit Interview, Questionnaire & Full-and-Final

### 6.1 Gates
6.1.1 Exit Interview and FnF both hard-require the employee's `relieving_date`. One exit interview per employee.

### 6.2 Exit Interview
6.2.1 Status: **Pending → Scheduled → Completed | Cancelled**; date + interviewers mandatory at Scheduled; final decision (**Employee Retained | Exit Confirmed**) mandatory at Completed.
6.2.2 Only Completed interviews submit; submit stamps `exit_interview_held_on` on the employee (cancel clears it).
6.2.3 **Exit questionnaire:** "Send Exit Questionnaire" emails the employee a link to the tenant-configured exit form using the configured template (both settings required). Sent-once flag; per-employee failure reporting. `[ADAPT]` The form is an in-app/public form of the platform; responses attach to the interview.

### 6.3 Full & Final Statement
6.3.1 Status: **Unpaid (default) → Paid | Cancelled** (system-derived). Mandatory: employee, transaction_date.
6.3.2 **Payables auto-seeded:** withheld salary slips (§12.7 — net pay, flagged `paid_via_salary_slip`), unsettled Expense Claims, Bonus (additional salary), Leave Encashment. Each row starts Unsettled. *(Gratuity row joins in Phase 2.)*
6.3.3 **Receivables auto-seeded:** outstanding Employee Advances (paid − claimed − returned).
6.3.4 **Assets held** by the employee are listed (asset seam — §16.7): per asset `action ∈ {Return, Recover Cost}`; cost editable only for Recover Cost.
6.3.5 Totals: total_payable = Σ payables; total_receivable = Σ receivables + Σ recover-cost.
6.3.6 **Submit gates:** every payable/receivable row Settled; every Return asset Returned.
6.3.7 **Settlement:** "Create Settlement" (submitted & Unpaid only) generates one settlement record netting payables − receivables with per-row references (§1.8). Confirmed → **Paid** (pushes paid-status to linked encashment rows); voided → back to Unpaid.
6.3.8 `[UI]` FnF is a working checklist — design for iterative settling.

---

## 7. Leave Management

### 7.1 Leave Type (tenant master)
7.1.1 Unique name. Mutual exclusions on save: not both **Compensatory** and **Earned**; not both **LWP** and **PPL**; cannot flip to LWP while an active allocation of the type exists.
7.1.2 **Payment flags:** `is_lwp` (unpaid; needs no allocation to apply); `is_ppl` (pays a fraction of daily salary; `fraction_of_daily_salary_per_leave` ∈ [0,1] mandatory).
7.1.3 **Allocation flags:** `is_carry_forward` + `maximum_carry_forwarded_leaves` cap + `expire_carry_forwarded_leaves_after_days` (Q19 — CF is configured per type, per tenant, from settings); `include_holiday` (holidays inside a leave count as leave days); `is_optional_leave` (dates must come from the period's Optional Holiday List); `is_compensatory` (granted only via Compensatory Leave Request — §7.10).
7.1.4 `[ADAPT]` **No negative balances (Q19).** The reference's `allow_negative` / `allow_over_allocation` flags are **removed**. Insufficient balance is always a hard error; allocations beyond the period's day count are always rejected.
7.1.5 **Limits:** `max_leaves_allowed` per leave period; `max_continuous_days_allowed` per application (chained across date-adjacent same-type applications); `applicable_after` N days of service.
7.1.6 **Earned-leave config:** `is_earned_leave`, frequency ∈ {Monthly, Quarterly, Half-Yearly, Yearly}, allocate-on ∈ {First Day, Last Day (default), Date of Joining (Monthly only)}, per-increment rounding ∈ {none, 0.25, 0.5, 1.0}.
7.1.7 **Encashment config:** `allow_encashment`, earning salary-component, `max_encashable_leaves`, `non_encashable_leaves`.
7.1.8 Seeded types per tenant (Q18 — **types only, no quantities**): Casual Leave, Sick Leave, Privilege Leave, Compensatory Off, Leave Without Pay. Admin sets annual quantities via Leave Policies.

### 7.2 Leave Period (Q17)
7.2.1 Tenant setting `leave_year ∈ {Fiscal (Apr–Mar), Calendar (Jan–Dec)}` drives the default period generated each year; admins may edit. to > from; **no overlapping active periods** per tenant. Optional `optional_holiday_list` per period.

### 7.3 Leave Policy & Assignment
7.3.1 Policy = titled rows {leave_type, annual_allocation}; each row ≤ the type's `max_leaves_allowed`.
7.3.2 **Policy Assignment** grants a policy to an employee for a window from `assignment_based_on ∈ {Leave Period, Joining Date, Custom}` (Joining Date → DOJ + 12 months default). No overlapping assignments per employee.
7.3.3 On submit: one **Leave Allocation** per non-LWP policy row (LWP never allocates; compensatory starts at 0). Per-type CF forced off when the type disallows it. Zero-quantity allocations skipped unless earned.
7.3.4 **Mid-period joining pro-rata:** allocation = annual × (days from DOJ to period end) ÷ (days in period), rounded (whole numbers for normal types; exact for earned; capped at annual except earned+Yearly). Earned types assigned mid-period retro-allocate elapsed sub-periods.
7.3.5 Bulk assignment: per-employee savepoints, partial-success reporting.

### 7.4 Leave Allocation
7.4.1 Fields: employee, leave_type, from/to, new_leaves_allocated, unused_leaves (carried in), total_leaves_allocated, carry_forward, total_leaves_encashed, expired. to > from; LWP types unallocatable; **no overlapping allocation** per employee+type.
7.4.2 total = carried-forward + new. CF pulls the previous allocation's remainder, capped by the type's CF cap; total capped at `max_leaves_allowed` (excess trims CF). Submitting a CF allocation **expires the previous allocation's remainder** and stamps the carried-out quantity.
7.4.3 Allocating more days than the period contains → hard error (§7.1.4).
7.4.4 Post-submit quantity edits: forbidden for scheduler-owned earned allocations; otherwise never below leaves already taken; every delta writes a ledger entry.
7.4.5 Cancel reverses ledger + CF stamping; blocked if applications already consumed inside the window.

### 7.5 Leave Ledger — balance source of truth
7.5.1 `[API]` Append-only signed ledger: transaction_type ∈ {Allocation, Application, Encashment, Adjustment}, ±leaves, from/to, flags {is_carry_forward, is_expired, is_lwp}. **Balances are always derived, never stored.**
7.5.2 Credits: allocations, CF, positive adjustments, earned increments, comp-off grants. Debits: approved applications, encashments, expiry, negative adjustments.
7.5.3 CF credits carry their own validity window ending at min(alloc start + CF-expiry-days − 1, alloc end).
7.5.4 Cancelling a source document deletes its ledger entries (with paired expiry entries); blocked if later applications fall inside the window.
7.5.5 **Daily expiry job:** past-window allocations (and CF sub-windows) get negative `is_expired` entries for the remainder; allocation marked expired.
7.5.6 **Consumable balance** on a date additionally caps by days remaining until allocation/CF expiry.

### 7.6 Leave Application
7.6.1 Status: **Open (default) → Approved | Rejected → (Cancelled)**. Only Approved/Rejected can be submitted.
7.6.2 Smallest unit = **half day** (Q20): `half_day` + `half_day_date` (inside range, not on a holiday).
7.6.3 `total_leave_days = calendar days − 0.5 (if half day) − holidays in range (unless the type includes holidays)`; ≤ 0 → error.
7.6.4 **Balance check** (non-LWP): consumable balance must cover the total — always a hard error when short (§7.1.4). The application must fall inside one submitted allocation window; spanning two allocations is not supported. `[ADAPT]` (reference allowed it only with negative balances, which no longer exist).
7.6.5 **Overlap:** no overlapping Open/Approved application per employee; two half-days on a date allowed only while that date's total < 1 day.
7.6.6 Other guards: max continuous days; minimum service days; optional-leave dates from the Optional Holiday List; block-list dates (warn while Open, block approval — §7.11); LWP cannot overlap a period already covered by a submitted salary slip; cannot apply over days already marked Present/WFH.
7.6.7 **Backdating (Q22 — tenant setting):** `backdated_leave_policy ∈ {Allowed (default), Blocked, Allowed within N days}` + optional exempt role. `[STD]` default Allowed — confirm.
7.6.8 **Approval & effects:** approver = reporting manager (§1.6). On approval+submit: attendance auto-marked per day (On Leave / Half Day, linked), converting pre-existing Absent records; cancel cancels that attendance. Ledger debits split at CF-expiry boundaries; backdated applications on already-expired allocations write a compensating reversal.
7.6.9 Notifications: in-app+push to manager on creation; to employee on approval/rejection/cancel; optional emails via platform SMTP when leave-notification templates are enabled (§14).

### 7.7 Balance reporting
7.7.1 Period report per employee × type: Opening (balance at from−1; on an allocation-expiry boundary counts only the CF part), Allocated, Taken, Expired, Closing = opening + allocated − taken − expired.
7.7.2 As-of-date balance matrix. `[UI]` ESS shows the live balance on the leave form **before** submission (§13.4).

### 7.8 Earned-leave accrual (daily scheduler)
7.8.1 The policy assignment pre-generates the period's **accrual schedule** rows {allocation_date, quantity, is_allocated, allocated_via, attempted, failed, failure_reason}.
7.8.2 Daily job allocates rows due today (skips Left employees). Per-period amount = annual ÷ {12, 4, 2, 1}, pro-rated for a partial first sub-period, rounded per type.
7.8.3 Caps per increment: cumulative (excl. CF) ≤ policy annual (except Yearly); ≤ type max; remaining quota clamps; nothing left → row fails.
7.8.4 Failures logged per row; HR notified with a digest; manual retry re-runs failed rows.

### 7.9 Leave Adjustment & Encashment
7.9.1 **Adjustment**: {Allocate | Reduce} against one allocation; non-zero; one submitted adjustment per allocation; Allocate ≤ type max; Reduce ≤ available balance on posting date; writes a signed ledger entry.
7.9.2 **Encashment** status: **Draft → Unpaid | Paid → (Cancelled)**. Requires an active allocation on the encashment date + an encashment-enabled type.
7.9.3 `actual_encashable = max(balance − non_encashable, 0)` capped by `max_encashable_leaves`; user may lower, never exceed. Amount = days × the employee's `leave_encashment_amount_per_day` (from the active salary-structure assignment, fallback structure); must be > 0 to submit.
7.9.4 Payout via **Additional Salary** on the type's earning component (next payroll) or direct settlement record (§1.8). Submit stamps the allocation's encashed total + negative ledger entry; cancel reverses.
7.9.5 **Auto-encashment (daily job, tenant toggle, default OFF):** the day after an encashable allocation expires, a draft encashment is created for employees with a salary structure.

### 7.10 Compensatory Leave Request
7.10.1 Grants comp-off for working on holidays: every claimed day must be a **holiday** for the employee AND have submitted Present/WFH/Half-Day attendance (a half-day presence cannot claim a full day). Half-day claims supported. No overlapping requests; no future work dates.
7.10.2 The comp leave is valid from the day after the worked range; grant extends the existing compensatory allocation or creates one.
7.10.3 **Comp-off validity (Q21 — tenant setting):** `compoff_validity ∈ {End of leave period (default), N days from the worked day}`. With N-days mode, the grant's ledger window ends at worked_day + N. `[STD]` default = leave-period end — confirm N default (90) if enabled.
7.10.4 Cancel decrements the allocation (floored at 0) + reversal entry.

### 7.11 Leave Block Lists
7.11.1 Block list = named {date, reason} rows + exempt-user allow-list; optional leave-type restriction; tenant-wide or attached to departments.
7.11.2 Applicability = tenant-wide lists + the employee's department's list, filtered by type. Weekly-recurrence helper bulk-adds weekday blocks.
7.11.3 Effect: warn while Open; block approval.

### 7.12 Bulk grant tool
7.12.1 One screen to bulk-create policy assignments (mode A) or direct allocations of one type + N days (mode B); date basis ∈ {Leave Period, Joining Date, Custom}.
7.12.2 Filters (department/designation/grade/employment type); only Active employees without an overlapping allocation/assignment. Per-employee savepoints; partial-success report.

---

## 8. Attendance, Check-in & Shifts

*Phase 1 ships attendance + check-ins + the shift configuration needed to power them. The rostering suite (shift requests, repeating schedules, roster board, bulk shift tools) is Phase 2 — §21.* `[ADAPT]`

### 8.1 Attendance record
8.1.1 **Status: `Present | Absent | On Leave | Half Day | Work From Home`.** One record per employee per date (per shift where shifts are used).
8.1.2 **Half-day sub-status** `half_day_status ∈ {"", Present, Absent}` describes the other half; `modify_half_day_status` marks it provisional pending auto-attendance.
8.1.3 Links: shift, leave_application + leave_type, attendance_request. Times: in/out, working_hours; flags late_entry / early_exit.
8.1.4 Validations: date ≥ DOJ; employee not Inactive; duplicate rule (completed half-day pairings may coexist; row-locked check). **No multiple shifts per day (Q26)** — a second same-day attendance on a different shift is rejected.
8.1.5 **Leave sync on save:** an approved leave covering the date forces On Leave / Half Day + links it; claiming leave-status without a matching leave sets other-half Absent and warns.
8.1.6 Cancel unlinks associated check-ins. `[UI]` Calendar merges attendance + holidays; real-time updates.

### 8.2 Bulk marking & import (Q23, Q51)
8.2.1 Per-employee bulk marking of unmarked days (one status); >10 days → background; duplicates skipped silently.
8.2.2 One-date-many-employees tool: buckets marked / half-day / unmarked (filters incl. shift); mass status + late/early flags.
8.2.3 **CSV import** (also the Q51 migration path for attendance history): template pre-fills existing records and labels holidays (stripped on import); imports land Submitted; >200 rows → background.
8.2.4 Unmarked-day queries clamp to [DOJ, relieving_date], optionally excluding holidays.

### 8.3 Employee Check-in
8.3.1 Log = {employee, timestamp (second precision), log_type ∈ {IN, OUT} (optionality per shift mode), device_id, latitude/longitude, skip_auto_attendance, linked attendance, cached shift-window snapshot}.
8.3.2 Duplicate guard: same employee + timestamp + log_type rejected. A log consumed by attendance cannot have its time edited.
8.3.3 **Auto shift matching:** the log attaches to the shift whose actual window (start − early-buffer … end + late-buffer) contains the timestamp; no match → `offshift` (excluded from auto-attendance).
8.3.4 Strict-log-type shifts require log_type; alternating shifts don't.
8.3.5 **Geofencing (Q24 — default ON):** every check-in requires lat/long; when the employee's shift assignment carries a **Shift Location** with `checkin_radius > 0`, the Haversine distance must be ≤ radius or the check-in is rejected. `[STD]` Default radius for new shift locations: 200 m — confirm.
8.3.6 `[API]` **Ingestion API (Q52):** create a check-in by employee key (id or `attendance_device_id`), timestamp, optional log type/device/coordinates — the single entry point for the mobile app and the **eSSL device connector** (an adapter service that maps eSSL push records onto this API; device registry kept minimal: device_id ↔ location). `[STD]` Connector delivery semantics: at-least-once with the duplicate guard absorbing replays.
8.3.7 Mobile check-in gated by tenant toggle (default ON). `[UI]` ESS check-in panel: last IN/OUT, live clock in the confirm modal, map preview of captured location; confirmation blocked without location when geofencing applies.

### 8.4 Shift Type (Phase 1 subset)
8.4.1 Required: start_time ≠ end_time; overnight shifts span midnight; shift length + both buffers < 24 h. Buffers default 60 min each side.
8.4.2 Auto-attendance block (**all thresholds are settings, no fixed defaults — Q25**): `enable_auto_attendance`; IN/OUT determination ∈ {Alternating, Strictly by log type}; working-hours basis ∈ {First in & last out, Every valid pair}; `working_hours_threshold_for_half_day`; `working_hours_threshold_for_absent` (wins over half-day); `mark_auto_attendance_on_holidays`; `process_attendance_after` (date floor); `last_sync_of_checkin` watermark + optional auto-advance; late/early grace minutes.
8.4.3 Shift timing edits are blocked while unprocessed check-ins exist.
8.4.4 **Shift Assignment** (basic): employee, shift_type, start_date, optional end_date, status Active/Inactive, shift_location. Date-overlapping assignments are **always blocked** (Q26). Cancellation blocked while check-ins/attendance exist in the window. Daily job expires past-end assignments to Inactive.
8.4.5 Effective shift on a date = latest Active submitted assignment starting on/before it; fallback = employee `default_shift`.

### 8.5 Auto-attendance processing (hourly job per enabled shift)
8.5.1 Eligible logs: not skipped, unlinked, ≥ process-after date, shift ended before the watermark, not offshift; grouped per employee + shift instance.
8.5.2 Working hours per §8.4.2 basis: first/last or summed pairs (strict mode pairs IN→OUT), 2-dp.
8.5.3 Status: hours < absent-threshold → **Absent**; else < half-day threshold → **Half Day**; else **Present**. Late/early flags per grace.
8.5.4 Holidays skipped unless the shift marks them; **half-day holidays halve both thresholds**.
8.5.5 Provisional Half Days (from leave) are updated in place — hours recorded, other half resolved Present/Absent.
8.5.6 Attendance links back to its check-ins. Failing groups are skipped-and-flagged with reason; batched processing with periodic commits.
8.5.7 **Absent for missing check-ins:** working days with zero attendance inside the processed window auto-mark Absent — only from one day **after** the shift day (grace for manual records). Unresolved half-day other-halves finalise Absent.
8.5.8 Watermark auto-advance moves `last_sync_of_checkin` past each completed shift when enabled.

### 8.6 Attendance Request (regularisation / WFH / on-duty) — no approval (Q14)
8.6.1 Fields: date range (backdating allowed), optional half-day + date, include_holidays, optional shift, **reason ∈ {Work From Home, On Duty}** + explanation. Submission itself creates attendance — no approver.
8.6.2 Multiple active shift assignments in range with none specified → error; exactly one → auto-filled.
8.6.3 Per-day outcome: skip holidays (unless included) and approved-leave days; else create/update attendance (Half Day on the half-day date / WFH / Present).
8.6.4 Overwrite semantics: an existing different-status record updates in place (linked, comment trail); a Half-Day-Absent other-half flips Present when covered. If no day would change → rejected with a per-day reason table.
8.6.5 Cancel cancels the attendance it created. Overlapping requests per employee(+shift) rejected.

### 8.7 Reports
8.7.1 **Monthly attendance sheet** (month or ≤ 90-day range): detailed per-day grid or summarized. Abbreviations: P, A, HD/A, HD/P, WFH, L, H, WO. Unmarked working days show blank (detailed) / count as `unmarked_days` (summary). Summary: present = P + WFH + 0.5×HD; leaves = L + 0.5×HD; absents; holidays; per-leave-type columns; late/early counts. Pre-DOJ days excluded.
8.7.2 **Shift attendance report:** per-attendance shift window vs actual in/out, hours, late/early (with a consider-grace toggle), optional rows without check-ins; summary tiles.
8.7.3 **Employees working on holidays:** attendance Present/Half-Day/WFH dated on the employee's holidays.

---

## 9. Performance — Manual Rating (Q41)

*Phase 1 ships appraisal cycles with **manual KRA rating**, self-appraisal and 360° feedback. Goal trees and goal-progress-automated KRA scoring are Phase 2 (§20).*

### 9.1 Appraisal Cycle
9.1.1 Status: **Not Started → In Progress → Completed** (action-driven: Start / Mark as Completed / reopen). Completion blocked while any appraisal in the cycle is Draft.
9.1.2 **Completed cycles are frozen** — no appraisal or feedback in them can be created/modified (reopen to amend).
9.1.3 Multiple In-Progress cycles allowed; "the" active cycle = latest started.
9.1.4 `[ADAPT]` KRA evaluation method is fixed to **Manual Rating** in Phase 1 (the method field ships hidden, defaulted, and becomes selectable in Phase 2).
9.1.5 Appraisee list: "Get Employees" pulls Active employees per cycle filters (department/designation), auto-assigning each their designation's appraisal template (warn if missing; per-row override). Re-running replaces the list.
9.1.6 "Create Appraisals" creates one appraisal per appraisee, idempotently; >30 → background.
9.1.7 Optional per-cycle **final-score formula** (expression over goal_score, average_feedback_score, self_appraisal_score) replacing the default average.
9.1.8 Cycle dashboard: appraisee count, self-appraisals pending, employees without feedback.

### 9.2 Appraisal Template
9.2.1 Template = **KRAs with weightages (must total exactly 100)** + **rating criteria with weightages (total 100)**. The same criteria drive self-appraisal and 360° feedback. Templates attach to Designations.

### 9.3 Appraisal (per employee per cycle)
9.3.1 One appraisal per employee per cycle (no overlapping-period duplicates). Weightage totals re-validated at submission.
9.3.2 **Goal score (manual):** per-KRA score 0–5 × weight → `total_score` (0–5 scale). Appraiser remarks captured.
9.3.3 **Self-appraisal:** per-criterion star rating × 5 × weight → `self_score` (0–5); employee writes free-text reflections. Pending while 0.
9.3.4 **Final score** = mean(total_score, avg_feedback_score, self_score) — or the cycle formula. All 0–5, precision-rounded, recomputed on every save.
9.3.5 `[UI]` Appraisal shows the feedback timeline (reviewer, score, when) and a 1–5 score-distribution strip.

### 9.4 Employee Performance Feedback (360°)
9.4.1 Reviewer = another Active employee (self-feedback rejected); the appraisal must belong to the subject; the cycle must not be Completed.
9.4.2 Ratings per template criterion (weightages = 100); `total_score = Σ rating × 5 × weight` (0–5); feedback text mandatory.
9.4.3 Submitted feedback averages into the appraisal's `avg_feedback_score` on submit **and** cancel.

---

## 10. Daily Work Summary — in-app (Q42) `[ADAPT]`

10.1 The reference collected stand-ups by **reply email**; the platform has outbound SMTP only (Q50), so Phase 1 adapts to **in-app submission**: reminders go out by notification + email, employees reply in ESS. *(Drop the module entirely if this adaptation is unwanted — flagged §27.)*
10.2 A summary **group** = member list + daily prompt hour + prompt text (default "What did you work on today?") + holiday list. Disabled members excluded.
10.3 Hourly job: at each group's hour (skipping its holidays), create the day's summary record (status **Open**), send members the prompt (in-app + email), and surface a reply box in ESS.
10.4 Daily digest job: for each Open summary, compile submitted replies (attributed, oldest-first), list non-responders, deliver the digest to all members (in-app + email), mark **Sent**.

---

## 11. Expenses, Advances, Travel & Vehicles

### 11.1 Expense Claim — model
11.1.1 Header: employee, posting_date, approver = reporting manager (§1.6), cost attribution (cost_center/project — reporting dimensions). INR only.
11.1.2 Lines (≥1): expense_date, **Expense Claim Type**, description, `amount` (claimed), `sanctioned_amount` (defaults = claimed; approver may only reduce). Rejection forces all sanctions to 0.
11.1.3 **Receipts are mandatory (Q44): every expense line requires an attachment**; submission is blocked without one.
11.1.4 Optional taxes rows (rate → tax computed on the sanctioned total) and advance-allocation rows (§11.3).
11.1.5 **Expense Claim Type** master: unique name + description (seeded: Calls, Food, Medical, Others, Travel). *(Account mapping joins with the Accounts module — seam.)*

### 11.2 Lifecycle & math
11.2.1 `approval_status ∈ {Draft → Approved | Rejected}` (approver-only field) — must be non-Draft to submit. Display status: **Draft | Submitted | Unpaid | Paid | Rejected | Cancelled**; Paid when fully reimbursed, marked paid-immediately, or grand_total = 0 (fully advance-covered).
11.2.2 `grand_total = Σ sanctioned + Σ taxes − Σ allocated advances` (≥ 0 via allocation caps); outstanding = sanctioned + taxes − reimbursed − advances; over-reimbursement blocked. Reimbursement is a settlement record (§1.8).
11.2.3 Self-approval blocked (Q16); in-app + push notifications per §14.2.
11.2.4 Cancel reverses settlement links and re-computes linked-advance consumption.

### 11.3 Advance clearing inside a claim
11.3.1 Linkable: the same employee's submitted, paid, not-fully-consumed advances; per-row allocation ≤ unclaimed remainder (paid − claimed − returned); Σ allocations ≤ sanctioned + taxes.
11.3.2 `[UI]` Auto-allocation waterfall fills advances oldest-first.
11.3.3 On claim approval+submit, each advance's claimed amount and status recompute from all approved claims referencing it.

### 11.4 Employee Advance
11.4.1 Fields: employee, posting_date, purpose, advance_amount, `repay_unclaimed_amount_from_salary` (**default ON — Q45**).
11.4.2 **Status (derived): `Draft | Unpaid | Partially Paid | Paid | Claimed | Returned | Partly Claimed and Returned | Cancelled`** — precedence: fully claimed → Claimed; fully returned → Returned; claimed+returned = paid → Partly…; paid = requested → Paid; some paid → Partially Paid; else Unpaid.
11.4.3 Caps: paid ≤ advance amount; returns ≤ paid − claimed. Prior unpaid advances surface at creation.
11.4.4 Unclaimed-money recovery: default = **salary deduction** (an Additional Salary deduction referencing the advance, recovered through payroll; scheduled deductions never exceed the recoverable remainder; a banner shows scheduled-vs-recovered). Alternative when the toggle is off: a direct repayment record (§1.8).
11.4.5 Disbursement is a payment record (§1.8); a tenant toggle controls payment unlinking on cancel (default OFF).

### 11.5 Travel Request — pure data capture (Q15)
11.5.1 No approval, no financial effect. Fields: travel_type {Domestic, International}, funding {Require Full Funding, Fully Sponsored, Partially Sponsored…}, purpose, identification details, sponsor details, attachments.
11.5.2 Itinerary rows: from/to, mode {Flight, Train, Taxi, Rented Car}, departure/arrival, meal preference, advance-required + amount, lodging-required + area/check-in/out. Costing rows: expense type, sponsored/funded/total. Informational only.

### 11.6 Vehicle Log (Q43)
11.6.1 Per vehicle+employee+date: odometer (≥ the vehicle's last reading), refuelling {qty, price, supplier, invoice}, service rows {item, type ∈ Inspection/Service/Change, frequency, expense}.
11.6.2 Submit advances the vehicle's odometer; cancel rolls it back.
11.6.3 "Create Expense Claim" from a log: amount = services + fuel (qty × price); one claim per log; cancelling the log deletes/cleans a draft claim containing only vehicle rows.
11.6.4 Reports: advance summary (outstanding = paid − claimed − returned), unpaid claims, monthly fuel-vs-service chart.

---

## 12. Payroll (Monthly, Leave-based, No Income Tax)

*Phase 1 payroll = monthly gross-to-net with settings-driven statutory deductions, in-app slips, and bank-file export. Income tax/TDS, flexible benefits, gratuity, overtime and incentives are Phase 2 (§22–§25).*

### 12.1 Payroll Period & Settings
12.1.1 **Payroll Period**: tenant-scoped {start, end}, non-overlapping; generated per the tenant's year convention (aligned to the leave-year setting by default `[STD]`). It anchors YTD windows and month generation.
12.1.2 **Payroll settings (per tenant):** basis = **Leave** (fixed in Phase 1 — Q30); `include_holidays_in_total_working_days` (default OFF); `daily_wages_fraction_for_half_day` (default 0.5); `disable_rounded_total`; `show_leave_balances_in_salary_slip`; `treat_unmarked_days_as_lop` (**default ON — Q30**, see §12.2.4); rounding conventions (Q55 — §17.2.7).
12.1.3 `[ADAPT]` Frequency is **Monthly only** (Q29): slips always span a calendar month; the reference's other frequencies are removed (schema keeps the field for the future).

### 12.2 Working days & payment days
12.2.1 `total_working_days` = days in month − holidays (holidays included instead when the tenant opts in).
12.2.2 `payment_days` seeds from the employment-clamped window (max(start, DOJ) … min(end, relieving)) minus holidays per the same setting.
12.2.3 **LWP (Leave basis):** approved LWP/PPL leave days subtract from payment_days — half-days count (1 − half-day fraction); PPL days weighted by (1 − fraction_of_daily_salary_per_leave).
12.2.4 **Unmarked days = LOP (Q30)** `[ADAPT]`: with `treat_unmarked_days_as_lop` ON, working days inside the clamped window having **neither an attendance record nor an approved leave** also subtract from payment_days. `[STD]` Precise intent flagged for confirmation (§27) — this makes attendance-marking effectively mandatory for pay.
12.2.5 Payroll-correction credits add back reversed LOP days (§12.6.3). A manual LWP override warns but wins. payment_days floors at 0.
12.2.6 An employee relieved before the month must be status Left or slip creation fails.

### 12.3 Salary Component
12.3.1 Unique name + **abbreviation** (auto-initials, de-duplicated) — the formula token. `type ∈ {Earning, Deduction, Employer Contribution}`.
12.3.2 Value source: fixed amount OR formula (`amount_based_on_formula`); optional condition expression — falsey ⇒ component skipped.
12.3.3 Behaviour flags: `depends_on_payment_days` (default ON ⇒ prorated); `statistical_component` (computed & referenceable, never paid); `do_not_include_in_total`; `remove_if_zero_valued` (default ON); `round_to_the_nearest_integer`; `arrear_component` (participates in arrear/correction differencing — §12.6); `disabled`.
12.3.4 Tax-related flags exist in schema but are **dormant until Phase 2** (§23). `[ADAPT]`
12.3.5 `[API]` Formula safe-eval whitelist (everywhere formulas run): int, float, round, rounded, date, getdate, get_first_day, get_last_day, ceil, floor, min, max — no attribute access; admin-authored only. Editing a component can bulk-sync into structures referencing it.
12.3.6 **Statutory components (Q31)** are settings-driven (§12.9), seeded: Basic (earning), HRA (earning), PF (deduction), ESI (deduction), Professional Tax (deduction), LWF (deduction) — each activating only when its statutory setting is enabled.

### 12.4 Salary Structure & Assignment
12.4.1 **Structure** (record-state-bearing template): `is_active`, earnings/deductions/employer-contribution rows, `leave_encashment_amount_per_day`, mode-of-payment. Row flags overlay from the component master on save.
12.4.2 Guard: a payment-days-dependent formula may not reference another payment-days-dependent component's abbr (double-proration).
12.4.3 **Assignment (SSA)** binds employee → structure **from a date**: base & variable amounts, payroll cost-center split (must total 100%), `leave_encashment_amount_per_day` override.
12.4.4 One submitted SSA per employee per from_date; from_date inside the employment window; the active SSA for a date = latest starting on/before it.
12.4.5 CTC preview: annual gross = monthly payable earnings × 12; CTC adds non-payable earnings + employer contributions.
12.4.6 Bulk SSA tool: Active employees employed at from_date without an SSA there; base defaults from grade (§2.2.2); >30 → background; per-employee savepoints.
12.4.7 `[API]` The SSA **pre-evaluates each structure row once** (full-cycle context) producing `default_amount`; slips consume these and apply proration only.

### 12.5 Salary Slip engine
12.5.1 **Status: `Draft | Submitted | Cancelled | Withheld`**. One slip per employee per month (scoped to its payroll run). Duplicate guard enforced.
12.5.2 Compute order: working/payment days → earnings → gross → deductions (incl. statutory §12.9) → net → totals. Earnings/deductions share one eval context (abbrs zero-seeded; `base`, employee and slip fields available; slip wins collisions).
12.5.3 **Proration:** payment-days-dependent components: `amount = default_amount × payment_days ÷ total_working_days`; zero payment days ⇒ 0; integer-rounding flag honoured; zero rows drop when `remove_if_zero_valued`.
12.5.4 **Additional Salary merge:** overwrite ⇒ replaces the structure row; non-overwrite ⇒ adds on top; amounts referencing Arrear / Payroll Correction / advance recovery skip re-proration (already period-correct); recurring entries spanning a partial window pay per-day × overlap.
12.5.5 Totals: gross = Σ payable earnings; net = gross − deductions; `rounded_total = round(net)`; words fields; slip-level + component-wise **YTD** (payroll-period window) and **MTD**.
12.5.6 **Negative net pay cannot be submitted** (bulk runs skip and report such slips).
12.5.7 Submit effects: leave-balance snapshot (tenant toggle); encashment/bonus additional-salaries flip Paid; advance-recovery deductions register against their advances. Cancel reverses.
12.5.8 **In-app only (Q33):** slips are viewed/downloaded (PDF) inside ESS — **no slip emails**. `[UI]` Slip detail tabs: Details / Earnings & Deductions / Net-pay info; download button. An in-app + push notification announces slip availability. `[STD]`

### 12.6 Additional Salary, Arrears & Corrections
12.6.1 **Additional Salary:** employee, component, amount ≥ 0, one-off payroll_date XOR recurring from/to (+ disable switch), `overwrite_salary_structure_amount`, reference to source (encashment, advance recovery, arrear, correction, FnF bonus). Guards: SSA must exist at the date; overwrite only for components present in the structure; no two overwrites of one component in one month; recurring same-component overlap blocked; dates clamp to employment.
12.6.2 **Arrear** (retroactive structure change): employee + new structure + payroll period, effective from a date inside it; requires the new SSA; one per employee/structure/period. Difference = preview slips under the new structure (honouring historical LWP) − amounts already paid, per **arrear-flagged** component; only positive diffs pay out via Additional Salaries.
12.6.3 **Payroll Correction** (LOP reversal): pick a month's submitted slip with LOP > 0; days_to_reverse > 0 and Σ across corrections ≤ that slip's LOP days. Breakup per arrear-flagged component: per-day = default_amount ÷ total_working_days × days_to_reverse → Additional Salaries (no re-proration). Reversed days credit later payment-day counts.

### 12.7 Salary Withholding
12.7.1 Withhold an employee's salary for N monthly cycles from a date; to_date = from + N months − 1 day. **Status: `Draft | Withheld | Released | Cancelled`**; no overlapping withholdings per employee.
12.7.2 A slip whose month matches a cycle is stamped **Withheld** and excluded from disbursement/bank file.
12.7.3 **Release:** a withheld-salary disbursement record ties into the cycles; confirming marks cycles released and flips slips to Submitted (reversal on void). Withheld net pay also surfaces in FnF (§6.3.2).

### 12.8 Payroll Run (Payroll Entry)
12.8.1 **Status: `Draft | Queued | Submitted | Failed | Cancelled`.** Scope: month, filters {department, designation, grade}, `validate_attendance` toggle.
12.8.2 Employee pool: Active-enough employees with a matching submitted SSA, employed within the month, minus those already payrolled; withheld employees flagged.
12.8.3 Draft-slip creation and slip submission each go **background beyond 30 employees**, with per-employee error isolation (failures logged + reported; systemic error → Failed).
12.8.4 With `validate_attendance` ON, submission is blocked while pooled employees have unmarked days. **Payroll register report** (accounting seam): per-run component-wise earnings/deductions per employee + cost-center dimension — Phase 1's replacement for GL postings (Q36).
12.8.5 Cancelling a run (background beyond 50 slips) cancels+deletes its slips and settlement records.

### 12.9 Statutory deductions — settings-driven (Q31) `[ADAPT]` `[STD]`
12.9.1 A **Statutory Settings** panel per tenant; each block independently toggleable, each computing its seeded deduction component on the slip. Defaults follow current law but **must be confirmed with your CA** and be effective-dated so rate changes don't rewrite history:
- **PF:** employee 12% of PF wages (Basic + DA); PF wage ceiling toggle (₹15,000 default) with "restrict to ceiling" or "actual wages" option; employer share (12%) shown as Employer Contribution; EPS split display optional.
- **ESI:** applies while gross ≤ threshold (₹21,000 default); employee 0.75%, employer 3.25%; contribution-period exit rules (remain covered till period end) `[STD]`.
- **Professional Tax:** state-wise monthly slab table (tenant selects state(s); editable slab grid; e.g. Kerala/Karnataka/Maharashtra presets).
- **LWF:** state preset {amounts, frequency}.
12.9.2 Statutory components are non-editable on individual slips (recompute-only). Registers: PF, ESI, PT reports per month (§12.10).
12.9.3 `[API]` All statutory parameters are versioned (effective-from) tenant settings; slips snapshot the parameter version used.

### 12.10 Bank file & reports (Q36–Q37)
12.10.1 **Bank salary-file export (Phase 1):** from a submitted run, generate a disbursement file of net pays for submitted, non-withheld slips. `[STD]` Format: a generic bank-upload CSV (account number, IFSC, beneficiary name, amount, narration) + per-bank templates as a config table. **Confirm the first bank formats to ship.** Regeneration allowed until the run is marked disbursed; the file event is logged.
12.10.2 Reports: payroll register (§12.8.4), salary register per employee/month, statutory registers (§12.9.2), LOP/payment-days summary.

---

## 13. Employee Self-Service — inside the Clozr app (Q47)

13.1 `[UI]` ESS is the HRMS section of the existing Clozr app (web + mobile), reusing platform auth, navigation, notifications and push (Q48). Surfaces: **Home, Attendance, Leaves, Expenses, Salary, Profile, Notifications**.
13.2 **Home:** greeting; **check-in panel** (§8.3.7 — hidden when the tenant disables mobile check-in); quick links (Request Attendance, Request Leave, Claim Expense, Request Advance, View Salary Slips, Daily Work Summary reply when due); **pending-approvals panel** for managers.
13.3 **Attendance:** colour-coded month calendar (attendance + holidays), recent attendance requests, my shift info.
13.4 **Leaves:** per-type balance gauges (allocated vs remaining); holidays list; leave list with **"My" vs "Team" tabs** (Team = my direct reports' requests awaiting me — driven by the reporting hierarchy) and filters (status/type/employee/department/dates). Leave form: half-day toggle reveals the half-day date (auto-set for single-day spans), types offered per selected date, live computed leave-days, **balance shown before submitting**, approver displayed read-only (= manager), attachments.
13.5 **Expenses:** summary tiles (pending/approved/rejected totals), My/Team claim lists, line items with mandatory receipts, taxes, advance allocations; advance request/list.
13.6 **Salary:** slip list + slip detail (§12.5.8) with PDF download. In-app only.
13.7 **Profile:** employee info with the **profile-completion checklist + deadline** (§3.2); self-editable fields per schema config.
13.8 Approvals: approve/reject action sheets on team items; if a tenant configures a custom workflow (§16.6), its states/transitions replace the native buttons automatically.
13.9 `[API]` ESS endpoints: current employee info; settings subset (check-in / geolocation / self-approval flags); notification counts + mark-read; attendance calendar events; attendance requests; leave applications with computed balances + approver metadata; leave types valid per date; expense claims + summary + types + advance balances; salary-slip PDF; profile-completion state; generic form-metadata engine (fields, states, permitted-write fields, workflow definition); file upload (JPG/PNG/PDF/TXT/Office; images EXIF-normalised; stored private). **No public API (Q53)** — these endpoints are first-party only.
13.10 "My vs Team" logic `[API]`: Team lists = other employees' actionable drafts where I am the stamped approver (or the workflow permits my role).

---

## 14. Notifications & Scheduled Jobs

### 14.1 Channels (Q48–Q50)
14.1.1 Channels: **in-app + push** (platform infra) and **email via the platform's existing SMTP service only**. No WhatsApp for HR (Q49). No per-module sender accounts — one platform sender identity. `[ADAPT]`

### 14.2 In-app/push matrix
14.2.1 On create of Leave Application / Expense Claim → the reporting manager ("X raised a new … for approval"). On status → Approved/Rejected → the employee ("Your … has been Approved/Rejected by Y"). Self-actions never notify. Advance requests carry no approval notification (reference behaviour).
14.2.2 Salary-slip availability → employee (§12.5.8). Profile-completion reminders (§3.2.4). Daily-work-summary prompts/digests (§10).

### 14.3 Email reminders (all ON by default — Q50)
14.3.1 **Birthdays** (daily): everyone in the tenant except the celebrants; celebrant-to-celebrant specials when shared.
14.3.2 **Work anniversaries** (daily): same audience pattern; completed-years computed.
14.3.3 **Holiday look-ahead** (weekly, or monthly on the 1st — tenant frequency setting, default Weekly): each Active employee's own upcoming non-weekly-off holidays.
14.3.4 Leave approval/status emails when the tenant enables them + templates (§7.6.9). Exit questionnaire email (§6.2.3).

### 14.4 Scheduled-job catalogue
| Frequency | Job |
|---|---|
| Hourly | Daily-work-summary prompts (per group hour); check-in watermark advance; **auto-attendance processing** per enabled shift |
| Daily | Birthday + anniversary reminders; work-summary digests; expire ended shift assignments; profile-completion reminder sweep `[STD]` |
| Daily (long) | Leave-allocation expiry entries; auto leave-encashment drafts (toggle); **earned-leave accrual** |
| Weekly / Monthly | Holiday look-ahead emails (per tenant frequency) |

---

## 15. Migration & Device Integration

15.1 **Go-live imports (Q51)** — exactly three, CSV-based, validating and partial-success (§16.3):
- **Employees** (basic profile + org placement + DOJ + device id; user provisioning follows §3.1.6).
- **Opening leave balances** — imported as opening Leave Allocations (+ ledger credits) per employee × type × current period.
- **Attendance history** — via the attendance CSV import (§8.2.3).
15.2 Explicitly NOT imported: salary structures, payroll history, YTD figures (payroll starts fresh in Clozr — first slips have no prior-slip YTD). `[ADAPT]`
15.3 **Biometric — eSSL (Q52):** an eSSL connector maps device push records → the check-in ingestion API (§8.3.6): device serial → registered device_id → employee via `attendance_device_id`; at-least-once delivery absorbed by the duplicate guard; unknown device/employee records land in an unmatched-punches queue for HR review. `[STD]`
15.4 No public/partner API (Q53).

---

## 16. Cross-Cutting Requirements

16.1 **Audit trail** `[STD]` — immutable log (actor, timestamp, before/after) for: employee status/date changes, record_state transitions, approver actions, attendance overwrites, allocation/ledger mutations, payroll-run lifecycle, statutory-setting changes, bank-file generation, settlement events. The leave ledger is itself audit-grade.
16.2 **Idempotency** — bulk generators (payroll slips, appraisals, allocations, onboarding tasks) are idempotent per (source, target, window); approval/submit endpoints accept idempotency keys.
16.3 **Partial-success bulk semantics** — per-record savepoints; never abort a batch on one failure; live progress; success/failure lists; failures logged with reasons.
16.4 **Derived-not-stored balances** — leave balance, advance outstanding, expense outstanding always computed from ledgers/transactions.
16.5 **Async thresholds** `[STD]` — background beyond: 10 (bulk attendance days), 30 (payroll create/submit, SSA bulk, appraisals), 50 (payroll-cancel slips), 200 (CSV rows). Long-job timeouts ≥ 50 min.
16.6 **Workflow seam** — no multi-level workflows ship (Q13), but Leave Application and Expense Claim expose {state field, allowed transitions per actor} so a configurable workflow engine can replace native statuses later without UI rewrites.
16.7 **Module seams:** Accounts module (all §1.8 settlement points + component account mapping), Project module (onboarding/separation tasks — §4.1.3), asset management (FnF asset rows), Recruitment/Training/Goals/Income-tax/Benefits/Gratuity (Part B).
16.8 **Real-time UX** `[API]` — attendance calendars, request lists, bulk-tool progress update via server push, not polling.

---

## 17. Tenant Seed Data & Settings Registry

### 17.1 Seed data (created per tenant at HRMS activation)
17.1.1 Leave types (§7.1.8 — no quantities). Expense claim types: Calls, Food, Medical, Others, Travel. Employment types (§2.3.1). Vehicle service items: Brake Oil, Brake Pad, Clutch Plate, Engine Oil, Oil Change, Wheels.
17.1.2 Salary components: Basic, HRA (earnings); PF, ESI, Professional Tax, LWF (statutory deductions, dormant until enabled — §12.9).
17.1.3 Email templates: leave approval, leave status, exit questionnaire. Default leave + payroll periods per the tenant's year setting (§7.2, §12.1.1).

### 17.2 HR settings
17.2.1 Employee: naming scheme (Q9), retirement age (60), `profile_completion_days` (30 — Q12), HR fallback approver (§1.6).
17.2.2 Reminders: birthday / anniversary / holiday toggles (ON — Q50) + holiday frequency (Weekly).
17.2.3 Leave: leave year FY/Calendar (Q17); per-type carry-forward caps (Q19); comp-off validity mode (Q21); backdating policy (Q22); self-approval block (ON — Q16); auto-encashment (OFF); leave email notifications + templates.
17.2.4 Attendance: mobile check-in (ON); **geolocation tracking (ON — Q24)**; default check-in radius (200 m `[STD]`); per-shift auto-attendance thresholds & grace (Q25).
17.2.5 Expense: mandatory receipts (ON — Q44); self-approval block (ON); advance recovery via salary (ON — Q45); unlink-payment-on-cancel (OFF).
17.2.6 Payroll: §12.1.2 settings; statutory blocks (§12.9); bank-file format selection (§12.10.1).
17.2.7 Display & rounding (Q55): net-pay rounding (nearest rupee default), leave display precision (0.5), date format — all tenant settings.
17.2.8 Exit: questionnaire form + template.

---

# PART B — PHASE 2 MODULES (SPECIFIED FOR PLANNING — DO NOT BUILD NOW)

*Rules below are complete enough to size and design seams against; re-validate before Phase 2 build. All Part A conventions (tenancy, record states, manager-as-approver, INR) apply.*

## 18. Recruitment & Careers Portal (Q38–Q40)
18.1 **Staffing Plan:** per-designation {vacancies, cost/position} for a date window; one active plan per designation per window; budget roll-up; seeds from requisitions.
18.2 **Job Requisition:** `Pending → Open & Approved → Filled | Rejected | On Hold | Cancelled`; "Open & Approved" gates opening creation; auto-Filled when its opening closes; time-to-fill analytics.
18.3 **Job Opening:** `Open | Closed`; staffing-plan gate (positions ≤ planned); templates; publish flag + unique public route; daily auto-close past closes_on.
18.4 **Job Applicant:** `Open | Replied | Shortlisted | Hold | Rejected | Accepted`; keyed by email (re-application suffixes); no applications against Closed openings; pipeline Kanban; apply-by-email intake (needs inbound email).
18.5 **Employee Referral:** one per candidate email; `Pending → In Process → Accepted | Rejected | Cancelled` synced from the applicant; referral bonus via one-per-referral Additional Salary, payment status synced.
18.6 **Interviews:** typed rounds (default interviewers, expected rating, skill set); one interview per applicant per round; only Cleared/Rejected submit (offering applicant sync); reschedule notifications; per-interviewer feedback (assigned interviewers only, not before the date, one each; per-skill ratings averaging into the interview); reminder + feedback-chase jobs.
18.7 **Job Offer:** `Awaiting Response → Accepted | Rejected | Cancelled`; one active offer per applicant; offer↔applicant status sync; optional staffing-plan vacancy check at offer; term templates; **Appointment Letter** from templates.
18.8 **Applicant → Employee:** unlocked by a submitted Accepted offer; employee creation back-syncs applicant + offer; onboarding gate applies.
18.9 **Careers portal:** public per-tenant listing (Open + published; filters, search, pagination) + application form (mandatory name/email, résumé, salary expectation); salary-range and application-count visibility flags per opening.

## 19. Training & Skills
19.1 Skill master; designation skill expectations; per-employee **skill map** (proficiency stars, evaluated-on; designation pre-load).
19.2 **Training Program → Events** (type/level/certificate; attendees with `is_mandatory`; scheduling emails) → attendee status `Open → Invited → Completed → Feedback Submitted`; attendance Present/Absent.
19.3 **Training Result** (one per event; hours/grade/comments per attendee) completes the event + attendees and requests feedback; **Training Feedback** only from non-absent participants.

## 20. Goals & Automated KRA Evaluation
20.1 Goal **tree** per employee (group nodes aggregate; progress 0–100; statuses Pending/In Progress/Completed auto + Archived/Closed manual; parent rollup = mean of non-archived children).
20.2 Top-level cycle goals must tag a KRA from the employee's appraisal; children inherit KRA/cycle; completed-cycle goals immutable.
20.3 **Automated appraisal mode:** per-KRA completion = mean progress of top-level non-archived goals → weighted → total_score = %/20 (0–5); recomputed live on every goal change. Cycle method becomes selectable (§9.1.4).

## 21. Shift Scheduling Suite & Roster Board (Q27)
21.1 **Shift Request** (`Draft → Approved | Rejected`; approver = manager; approval auto-creates the assignment; cancel cascades).
21.2 **Shift Schedules**: shift + frequency {every 1–4 weeks} + repeat weekdays; schedule assignments generate coalesced shift assignments 90 days ahead (hourly top-up); deleting cascades.
21.3 **Roster board**: month grid (employees × days; shift blocks colour-coded with holidays/leaves), smart-merge creation, split/run/schedule deletion granularity, **drag-and-drop swap**; whitelisted filters.
21.4 Bulk shift tool: assign shift / assign schedule / process requests; >30 → background.

## 22. Overtime, Incentives & Retention Bonus (Q28)
22.1 **Overtime Type:** payout component, max OT hours/day, method {Fixed hourly rate, Component-based (Σ selected components ÷ payment days ÷ standard hours)}, multipliers {standard, weekend, public holiday}.
22.2 OT hours captured by auto-attendance (Present + hours > shift standard); **Overtime Slip** per employee per month (rows from OT-bearing attendance, capped/day; no overlaps/duplicate dates) pays via per-component Additional Salary; payroll runs can bulk create/submit OT slips.
22.3 **Employee Incentive** (component + amount + payroll date → Additional Salary; SSA required) and **Retention Bonus** (payment date not past; duplicates accumulate).

## 23. Income Tax Engine (Q31–Q32)
23.1 Effective-dated **tax slab** masters (progressive bands with optional condition expressions; other-charges/cess rows compounding on tax; standard deduction; relief limit; marginal-relief limit), old + new regime sets, **Clozr-managed and pushed to tenants** `[STD]`.
23.2 Slab per SSA; annual projection = prior actuals + current (payment-days) + future cycles + additional income + other income − exemptions − standard deduction; per-cycle TDS = (projected − deducted) ÷ remaining cycles; full-tax-on-date additionals excluded from spread; never negative.
23.3 Exemption categories/sub-categories with caps; **Declaration** (planned, one per employee per period) vs **Proof Submission** (actuals; forced in the final cycle); Employee Other Income; HRA exemption (min of actual HRA / rent − 10% basic / 50-40% basic metro-split); marginal relief; slip income-tax breakup tab; PF/PT interplay.

## 24. Flexible Benefits (Q34)
24.1 Benefit components with yearly caps and payout methods {accrue & pay at period end, accrue per cycle & pay on claim (+ final-cycle payout), claim full amount}; entitlement via per-period Benefit Application (Σ ≤ max benefits) or SSA rows.
24.2 Per-cycle accrual = yearly ÷ cycles (payment-days-prorated), cumulative-capped; append-only **Benefit Ledger** (Accrual|Payout) as source of truth; claims (one per component per month, ≤ eligibility) pay via Additional Salary; unclaimed force-payouts become taxable in the final cycle.

## 25. Gratuity (Q34)
25.1 Rules: slab table {year bands, fraction of applicable earnings}, method {Current Slab, Sum of previous slabs}, experience calc {round off, exact, manual}, days/year (365), minimum years; India template: min 5 years, fraction 15/26.
25.2 Experience = (relieving − joining − LOP days) ÷ days-per-year; below minimum ⇒ ineligible. Basis = applicable components on the last submitted slip. Status `Draft → Unpaid → Paid | Cancelled`; payout via Additional Salary or settlement record; FnF integration.

## 26. Other deferred items
26.1 **Accounts-module integration:** replace §1.8 settlement records + reports with real postings; component/expense-type account mappings activate.
26.2 Multi-currency (excluded for India-only scope), WhatsApp notifications (declined — Q49), public API (declined — Q53), inbound-email intake (applicants, work-summary replies), offer e-sign links, appraisal/goal deadline reminders, two-way calendar sync.

---

## 27. Open Items Requiring Your Sign-Off

Remaining `[STD]` decisions — everything else is confirmed via §0.

1. **HR fallback approver** when an employee has no (active) manager — default System Admin? (§1.6)
2. **Manager-change mechanics** in promotions/transfers vs the CRM hierarchy (§5.1.4).
3. **Profile-completion enforcement** — reminder cadence only, or escalate (e.g. ESS lock) after deadline? (§3.2.4)
4. **Unmarked-day LOP rule** — confirm §12.2.4 exactly: any working day with neither attendance nor approved leave is unpaid. This makes attendance capture effectively mandatory for payroll.
5. **Statutory defaults** (PF ceiling ₹15k, ESI ₹21k / 0.75%+3.25%, PT state presets, LWF) — confirm with your CA; pick the state presets to ship first (§12.9).
6. **Bank-file formats** to ship first (generic CSV + which banks?) and the "marked disbursed" workflow (§12.10.1).
7. **Backdated-leave default** (Allowed vs window-limited) (§7.6.7) and **comp-off N-days default** if that mode is used (§7.10.3).
8. **Default geofence radius** 200 m (§8.3.5) and eSSL unmatched-punch queue handling (§15.3).
9. **Daily Work Summary**: accept the in-app adaptation, or drop the module (§10.1).
10. **Onboarding without recruitment**: candidate keyed by email, applicant/offer links dormant until Phase 2 (§4.2.1–4.2.2).
11. **Payroll-period generation** aligned to the leave-year setting (§12.1.1).
12. **Slip-availability notification** (in-app + push) on payroll submission (§12.5.8).

---

*End of Clozr HRMS Master Rule Book v2.0 — supersedes v1.0. Derived from the Frappe HR v17 reference implementation and the answered product questionnaire (§0). Read together with the Clozr CRM Master Rule Book v1.0 (shared platform: tenancy, users, roles, departments, reporting hierarchy, billing).*
