# Clozr HR — Master Rule Book (v1.0)
### Single Source of Truth for UI/UX Designers & Backend API Engineers

This document specifies the complete business logic and core UX of the **Clozr HR module** — the HRMS product appended to the existing Clozr CRM SaaS. It follows the same conventions as the *Clozr CRM Master Rule Book v1.0* and is designed to be read alongside it.

The rules below were extracted from an end-to-end analysis of a mature open-source HRMS reference implementation (Frappe HR v17, `develop` branch: 118 HR entities + 44 payroll entities, employee self-service mobile app, shift-roster app, and public careers portal). Every untagged rule reflects proven behaviour of that reference system, translated out of its framework specifics into product language. Where the reference had to be adapted to Clozr's multi-tenant SaaS architecture — or a genuine product decision had to be made — the rule is tagged.

**Explicitly out of scope (per product decision):** subscription/billing, user roles, and user-permission scoping. These are shared platform concerns already governed by the CRM Master Rule Book (§2, §3, §13 there). Note carefully: **approval chains (who approves a leave/expense/shift request) are business logic, not permission scoping — they are specified here in full.**

**Legend for inline tags**
- `[STD]` — a gap the reference system did not answer; filled here using industry standard. **Review and confirm — these are decisions made on your behalf.**
- `[ADAPT]` — reference behaviour adapted to Clozr's multi-tenant SaaS architecture (single-tenant framework concept → SaaS concept). Review the mapping.
- `[UI]` — a note specifically for designers.
- `[API]` — a note specifically for backend/data engineers.

---

## 0. Decisions Made on Your Behalf — Read This First

Each of the following is a business/architecture decision you should explicitly accept or override before build.

1. **Tenant = Company (§1.2)** — the reference system models multiple legal companies per installation (with parent/subsidiary hierarchies). Clozr is row-level multi-tenant; resolved as: **one tenant = one company** in Phase 1. Cross-company staffing-plan ceilings and inter-company transfers are reduced to future-phase seams.
2. **Document immutability model (§1.4)** — the reference uses a universal Draft → Submitted → Cancelled lifecycle where *submitted = locked*. This is retained as a product rule (it is what makes payroll/leave ledgers auditable), implemented as an explicit `record_state` on every transactional entity — not as a framework feature.
3. **Accounting integration seam (§1.6)** — the reference posts double-entry GL journals (expense claims, payroll accrual, settlements). Clozr has **no accounting module** (CRM rulebook §9.1). Resolved: Phase 1 records all monetary events as first-class payment/settlement records with statuses, and every GL-posting point is specified as a named **accounting seam** for a future accounting module or export.
4. **Approval engine (§1.7)** — the reference ships single-approver flows (employee-level approver → department chain fallback) plus an optional configurable multi-level workflow engine. Phase 1 ships the single-approver flows exactly as specified; the multi-level workflow engine is a seam.
5. **India-first tax & statutory scope (§18, §20)** — reference supports pluggable regional rules. Clozr is an Indian SaaS: ship India (income-tax slabs, HRA exemption, marginal relief, PF/PT components, Indian gratuity) in Phase 1; UAE gratuity rules documented as the template for future regions. **Confirm.**
6. **Attendance-vs-Leave payroll basis default (§15.2.1)** — reference default is "Leave". For a product with mobile check-in + auto-attendance as a headline feature, the recommended tenant default remains **Leave** (safest for tenants who don't adopt check-in), switchable per tenant to "Attendance". **Confirm default.**
7. **Employee ↔ platform user link (§3.1.9)** — every employee *may* be linked to a Clozr platform user (for self-service); employees without users are fully supported (blue-collar / bulk-managed staff). User management itself is the CRM platform's concern.
8. **Careers portal (§10.9)** — the public jobs page + application form is included in Phase 1 scope (it is load-bearing for the recruitment funnel), served on a tenant-specific public route.
9. **Async execution thresholds (§23.5)** — the reference hard-codes bulk-action thresholds (>10/20/30/50 records → background job). Numbers are retained as sensible defaults; tune per infrastructure.
10. **Notification channels (§22)** — reference splits email (leave/interview/reminders) vs in-app push (approvals). Retained; all events routed through one notification service with per-event channel defaults as specified.
11. **Timesheet-based payroll & lending (loans) (§15.4.4, §23.7)** — depend on modules Clozr does not have (project timesheets, lending). Specified as seams, not Phase 1 scope.
12. **Biometric device ingestion (§8.3.7)** — the check-in ingestion API accepts biometric-device punches keyed by an employee device-ID field. Included in Phase 1 as an API-only integration (no device management UI). **Confirm.**

---

## 1. Architecture & Cross-Module Ground Rules

1.1 The HR module lives inside the existing Clozr multi-tenant SaaS: row-level tenancy in PostgreSQL, `tenant_id` on every table, ORM + RLS double enforcement — all per CRM rulebook §1. No HR data ever crosses tenants.

1.2 `[ADAPT]` **Tenant = Company.** Every HR entity that the reference scoped by "company" is scoped by `tenant_id` in Clozr. Consequences, resolved here:
- Payroll periods, leave periods, staffing plans, holiday-list defaults, salary components' account mappings: one set per tenant.
- Reference rules involving parent/subsidiary company hierarchies (staffing-plan ceilings §10.1.9–11, inter-company transfer §5.3.6) are **deferred** — the schema keeps a nullable `legal_entity_id` so multi-entity tenants can be added later without migration. `[STD]`

1.3 **Org master data** (Department tree, Designation, Branch/Location, Employee Grade, Employment Type) are tenant-scoped masters — see §2.

1.4 `[ADAPT]` **Record states.** Every transactional HR document carries `record_state ∈ {Draft, Submitted, Cancelled}` in addition to its domain `status`:
- **Draft** — fully editable.
- **Submitted** — locked/immutable; the document has taken effect (created ledger entries, attendance, salary impact…). Editing requires **Cancel → Amend** (a new draft linked via `amended_from`).
- **Cancelled** — all side effects reversed (ledger entries deleted/reversed, generated child records cancelled). Cancellation is blocked wherever downstream records depend on the document (each blocking rule is listed in its section).
- `[API]` State transitions are the ONLY way side effects fire; side-effect logic must be idempotent per document.

1.5 **Employee-centricity.** Every transaction references exactly one Employee. A global guard: **no transaction may be created for an employee whose status is `Inactive`** (§3.1.2). Date-range transactions are clamped to the employee's `date_of_joining` … `relieving_date` window.

1.6 `[ADAPT]` **Accounting seam.** Wherever this book says *"accounting seam"*, Phase 1 must: (a) record the monetary event with amount, currency, date, counterparty (employee), and settlement status; (b) expose it via API/report for external accounting; (c) NOT attempt double-entry GL. The named seams: expense-claim posting & reimbursement (§14), employee-advance payment/return (§14.4), payroll accrual & disbursement (§17.1.8–9), leave-encashment payout (§7.10.5), gratuity payout (§20.3), full-&-final settlement (§6.3.10), exchange gain/loss (§14.3.4).

1.7 **Approver resolution (used by Leave, Expense, Shift Request).** A request's valid approvers = union of:
1. The approver named on the Employee record for that request type (`leave_approver` / `expense_approver` / `shift_request_approver`), and
2. All approvers configured on the employee's Department **or any ancestor department** in the department tree (each department holds an ordered approver list per request type; the first entry is the default).
- If no approver is resolvable and the tenant setting makes approvers mandatory → hard error on save.
- `[API]` The request document is auto-shared with the chosen approver (grants them action rights on that one document); re-assignment removes the old approver's access. A user selected as an approver is auto-granted the corresponding approver capability.
- Self-approval: blocked per request type when the tenant enables `prevent_self_*_approval` and no custom workflow is configured (§22.4).

1.8 `[API]` **Holiday resolution is one shared service** (§2.5) consumed by leave, attendance, shifts, payroll, and reminders. It must never be re-implemented per feature.

1.9 **Currency.** Tenant base currency INR. Multi-currency is supported exactly where the reference supports it: salary structures/slips, expense claims, and employee advances carry `currency + exchange_rate`, with every money field shadowed by a base-currency twin (`base_* = value × exchange_rate`). Everything else is base-currency only.

---

## 2. Organization & Shared Masters

### 2.1 Department
2.1.1 Departments form a **tree** (nested hierarchy) per tenant. A department may be disabled; disabled departments are excluded from approver resolution.
2.1.2 Each department carries: ordered approver lists for **Leave**, **Expense**, and **Shift Request** (§1.7); an optional default **payroll cost center**; an optional **leave block list** (§7.12).

### 2.2 Designation
2.2.1 Designation (job title) master carries: description, an optional **Appraisal Template** link (drives §11), and an expected **skills list** (drives §12).

### 2.3 Employee Grade
2.3.1 Grade carries a **default salary structure** and **default base pay** — used as defaults when assigning salary structures in bulk (§15.5) and on onboarding.

### 2.4 Employment Type & other simple masters
2.4.1 Employment Type is a plain unique-name master. Seeded per tenant: Full-time, Part-time, Probation, Contract, Commission, Piecework, Intern, Apprentice (§24).
2.4.2 Other unique-name masters: Health Insurance Provider, Identification Document Type, Grievance Type, Purpose of Travel, Vehicle Service Item, Job Applicant Source, Offer Term, KRA, Skill, Employee Feedback Criteria.

### 2.5 Holiday Lists & Holiday List Assignment
2.5.1 A **Holiday List** = named set of dated holidays within a from/to range; each holiday is flagged `weekly_off` (recurring weekend) or a described public holiday. Optionally a half-day flag per holiday (auto-attendance halves its thresholds on such days — §8.6.4).
2.5.2 Holiday lists are attached via **Holiday List Assignment** records (`record_state`-bearing): `applicable_for ∈ {Employee, Company}`, `assigned_to`, `holiday_list`, `from_date`. The assignment start date must fall within the holiday list's own date range. No duplicate assignment for the same target + from_date.
2.5.3 **Resolution order for "employee's holiday on date D":** most recent submitted assignment for the *employee* with from_date ≤ D; else most recent for the *tenant/company*; else error (or empty when the caller tolerates absence). The effective end of each assignment = min(list's to_date, next assignment's from_date − 1).
2.5.4 **Gap-filling:** for date ranges not covered by an employee-level assignment, the company-level assignment fills the gap; employee-level wins wherever present. Range queries crossing an assignment boundary split at each from_date and union the holidays.
2.5.5 `[API]` Holiday lookups are cached; any holiday-list or assignment change synchronously invalidates the cache (payroll working-day math depends on it).
2.5.6 `[UI]` Employees never pick their own holiday list; HR assigns. The employee-visible surface is a read-only "Holidays" list (§21.4).

---

## 3. Employee Master

### 3.1 Identity, status, key fields
3.1.1 **Employee status enum: `Active | Inactive | Suspended | Left`.**
- **Active** — normal. Only Active employees: receive reminder emails, accrue earned leave, appear in bulk-tool pickers, can be referenced by new transactions.
- **Inactive** — hard block: no transaction of any type may be created against an Inactive employee.
- **Suspended** — temporary hold; treated as non-Active for reminders/accruals but visible in reporting. `[STD]` Suspended blocks new self-service requests but allows HR-side corrections. **Confirm.**
- **Left** — exited. Requires `relieving_date`. Excluded from earned-leave accrual, payroll employee pools (beyond final settlement), and reminders.
3.1.2 The Active/Inactive guard (§1.5) is enforced in one shared validator used by every transactional entity.
3.1.3 **Employee ID generation** — tenant setting `employee_naming_by ∈ {Naming Series (default), Employee Number, Full Name}`. Unset → error on first employee creation.
3.1.4 Key dates: `date_of_birth`, `date_of_joining`, `relieving_date`, `resignation_letter_date`, `exit_interview_held_on`, `date_of_retirement` (computed = DOB + tenant `retirement_age`, default **60** years).
3.1.5 Compensation context fields: `ctc`, `salary_currency`, `grade`, plus org placement: `company/tenant`, `department`, `designation`, `branch`, `employment_type`, `reports_to` (manager).
3.1.6 Per-employee approver fields: `leave_approver`, `expense_approver`, `shift_request_approver` (all optional; department chain is the fallback — §1.7).
3.1.7 Attendance/shift fields: `default_shift`, `attendance_device_id` (biometric key — §8.3.7), payroll fields: `payroll_cost_center`, `employee_advance_account` seam.
3.1.8 **Internal Work History**: an append-only child table {branch, department, designation, from_date, to_date} maintained **automatically** by Promotion/Transfer (§5.1.3) — never hand-edited.
3.1.9 `[ADAPT]` `user_id` links the employee to a Clozr platform user for self-service. Optional. One user ↔ at most one Active employee per tenant. The self-service app shows an "Invalid Employee" state if a logged-in user has no Active employee record.

### 3.2 Cascades wired to the Employee record
3.2.1 Creating an Employee linked to a Job Applicant: applicant status → **Accepted**; the applicant's latest non-cancelled Job Offer → **Accepted** (prompt to complete it if still draft). (§10.8.3)
3.2.2 If a not-yet-completed Employee Onboarding exists for the linked applicant, the onboarding's mandatory-task gate applies to employee creation (§4.2.4) and the onboarding is stamped with the new employee id.
3.2.3 Setting `leave_approver`/`expense_approver` on an employee auto-grants that user the corresponding approver capability (kept in sync when users are edited).
3.2.4 `[UI]` Employee detail shows grouped related-record links (Attendance, Leave, Lifecycle: Onboarding/Transfer/Promotion/Grievance, Exit: Separation/Exit Interview/Full & Final/Salary Withholding, Shift, Expense, Benefits, Payroll, Training, Evaluation) and a 12-month attendance heat-map (Present + Half Day).

---

## 4. Employee Lifecycle — Onboarding & Separation

### 4.1 Templates & activities (shared model)
4.1.1 **Onboarding Template** and **Separation Template** are reusable activity checklists, optionally scoped by department/designation/grade. Selecting a template copies its activities into the onboarding/separation record.
4.1.2 **Activity row**: `activity_name` (required), assignee = a **user XOR a role** (one or the other), `begin_on` (offset days from start), `duration` (days), `task_weight`, `required_for_employee_creation` (onboarding only), description.
4.1.3 On submit of an Onboarding/Separation, the system generates one **project** with one **task per activity**: task start = begin_date + begin_on; task end = start + duration; **dates landing on a holiday roll forward to the next working day**. Tasks are assigned to the named user plus every enabled user holding the named role; assignees get to-dos, and email notification is sent only if `notify_users_by_email` is checked.
4.1.4 **Progress rolls back up**: as tasks complete, the parent record's `boarding_status` auto-derives from project % complete — `Pending` (0%) → `In Process` (0<x<100) → `Completed` (100%).
4.1.5 Activities may be appended after submission (tasks regenerate for new rows). Cancelling the record deletes the generated project + tasks.

### 4.2 Employee Onboarding
4.2.1 Mandatory: job applicant (status must be **Accepted**), job offer (submitted, same applicant), employee_name, date_of_joining, boarding_begins_on.
4.2.2 **One onboarding per job applicant** (non-cancelled duplicate blocked).
4.2.3 `boarding_status`: Pending → In Process → Completed (system-derived — §4.1.4). A **"Mark as Completed"** action force-completes all tasks + project.
4.2.4 **Employee-creation gate:** "Create Employee" is available only when the onboarding is submitted AND every activity flagged `required_for_employee_creation` has its task Completed/Cancelled. Otherwise: hard error listing pending mandatory tasks.
4.2.5 Employee creation from onboarding maps name/grade/personal email (from applicant), status = Active.

### 4.3 Employee Separation
4.3.1 Mandatory: employee, separation_begins_on. Pulls resignation_letter_date and org fields from the employee.
4.3.2 Identical template/task/status engine as onboarding (§4.1). Task holiday-shifts always use the **employee's** holiday list.
4.3.3 Separation does not auto-create Exit Interview or Full & Final — they are independent records grouped on the employee's exit dashboard.

---

## 5. Promotions, Transfers & Grievances

### 5.1 Property-diff engine (shared by Promotion & Transfer)
5.1.1 Both records carry a table of property changes: {employee field, current value, new value}. The UI dialog fetches the current value live, and blocks duplicate-field rows and no-change rows.
5.1.2 Fields excluded from the property-change picker: identity fields (names, employee id), status, gender, DOB, DOJ, marital status, ctc (handled explicitly on promotion), and non-data fields.
5.1.3 On submit, each change is applied to the Employee (type-safe conversion). Changes to **department / designation / branch** append an Internal Work History row effective from the promotion/transfer date; the previous history row is auto-closed (to_date = new from_date − 1). The first history row is seeded from current values + DOJ.
5.1.4 On cancel, the changes are reverted and appended history rows removed.

### 5.2 Employee Promotion
5.2.1 Mandatory: employee, promotion_date. Blocked for Inactive employees.
5.2.2 **Cannot submit before the promotion date** (future-dated promotions stay draft until the date arrives).
5.2.3 If `revised_ctc` is set, employee `ctc` ← revised value on submit; reverted on cancel.

### 5.3 Employee Transfer
5.3.1 Mandatory: employee, transfer_date, ≥1 transfer detail. Cannot submit before transfer date.
5.3.2 Default mode: apply property changes to the same employee record.
5.3.3 `create_new_employee_id` mode: clone the employee to a **new employee record** (new id), apply changes, move the platform-user link to the new record, and **relieve the old record** (status Left, relieving_date = transfer_date).
5.3.4 Cancel of a new-id transfer is blocked while the new employee record still exists; after deleting it, the old employee is restored to Active and relieving date cleared.
5.3.5 `[UI]` The transfer form warns that new-id transfers relieve the old record.
5.3.6 `[ADAPT]` Reference "transfer to another company" is deferred with multi-entity support (§1.2).

### 5.4 Employee Grievance
5.4.1 Status enum: **Open (default) → Investigated → Resolved | Invalid | Cancelled.**
5.4.2 Mandatory: subject, raised_by (employee), date, grievance_type, grievance_against_party ∈ {Company, Department, Employee Group, Grade, Employee} + the specific party, description. An employee cannot file a grievance against themself.
5.4.3 Conditional mandatory: `cause_of_grievance` when Investigated/Resolved; `resolved_by`, `resolution_date`, `resolution_detail` when Resolved.
5.4.4 Only **Resolved** or **Invalid** grievances can be submitted (closed); discard → Cancelled.
5.4.5 `[UI]` Grievances are employee-raisable via self-service; sensitive — keep visibility restricted to the raiser + HR.

---

## 6. Exit: Exit Interview, Questionnaire & Full-and-Final Settlement

### 6.1 Exit gates
6.1.1 An employee's exit flow requires `relieving_date` to be set — both Exit Interview and Full & Final hard-require it.
6.1.2 **One exit interview per employee** (non-cancelled duplicate blocked).

### 6.2 Exit Interview
6.2.1 Status enum: **Pending → Scheduled → Completed | Cancelled.** Conditional mandatory: interview date + interviewers when Scheduled; final decision when Completed.
6.2.2 Final decision (`employee_status`) enum: **Employee Retained | Exit Confirmed**.
6.2.3 Only **Completed** interviews can be submitted; submit stamps the employee's `exit_interview_held_on`; cancel clears it.
6.2.4 **Exit questionnaire:** a "Send Exit Questionnaire" action emails the employee a link to the tenant-configured exit web form using the configured email template (both settings mandatory to use the action — §22.2). Sent-once flag prevents duplicates; failures (e.g. missing employee email) are reported per employee.

### 6.3 Full & Final Statement (FnF)
6.3.1 Status enum: **Unpaid (default) → Paid | Cancelled** (system-derived). Mandatory: employee, transaction_date; relieving date required to populate.
6.3.2 **Payables auto-seeded** on creation: withheld salary slips (§17.2 — amount = net pay, flagged `paid_via_salary_slip`), Gratuity, unsettled Expense Claims, Bonus (additional salary), Leave Encashment — each row starts `Unsettled`.
6.3.3 **Receivables auto-seeded**: outstanding Employee Advances (+ loans via lending seam).
6.3.4 **Company assets held by the employee** are listed; per asset `action ∈ {Return | Recover Cost}`, status ∈ {Owned, Returned}; `cost` editable only for Recover Cost.
6.3.5 Per-row outstanding amount formulas: Salary Slip = net pay; Expense Claim = grand total − reimbursed − advances; Advance = paid − claimed − returned; Gratuity/Encashment = their amounts.
6.3.6 Totals: `total_payable = Σ payables`; `total_receivable = Σ receivables + Σ recover-cost assets`.
6.3.7 **Submit gates:** every payable & receivable row must be marked `Settled`; every Return-asset must be `Returned`. Violations block submission.
6.3.8 `[UI]` FnF is a working checklist — design for iterative settling, not one-shot entry.
6.3.9 Cancel reverses any settlement records created.
6.3.10 **Settlement (accounting seam):** a "Create Settlement" action (only when submitted & Unpaid) generates one settlement payment record netting payables minus receivables (per-row references retained). When the settlement is confirmed → FnF status **Paid**; if voided → back to **Unpaid**; the linked Gratuity/Leave Encashment rows have their paid-status pushed in sync.

---

## 7. Leave Management

### 7.1 Leave Type (per-tenant master)
7.1.1 Unique name. Mutual exclusions enforced on save: cannot be both **Compensatory** and **Earned**; cannot be both **LWP** (Leave Without Pay) and **PPL** (Partially Paid Leave); cannot be flipped to LWP while an active allocation of this type exists.
7.1.2 **Payment flags:** `is_lwp` — unpaid, needs no allocation to apply; `is_ppl` — pays a fraction of daily salary, requires `fraction_of_daily_salary_per_leave` ∈ [0,1].
7.1.3 **Allocation flags:** `is_carry_forward` (+ `maximum_carry_forwarded_leaves` cap + `expire_carry_forwarded_leaves_after_days`); `allow_negative` (insufficient balance becomes a warning, not an error); `allow_over_allocation` (allocating more days than the period has becomes a warning); `include_holiday` (holidays inside a leave count as leave days); `is_optional_leave` (must be taken on dates from the period's Optional Holiday List); `is_compensatory` (granted only via Compensatory Leave Request — §7.11).
7.1.4 **Limits:** `max_leaves_allowed` per leave period; `max_continuous_days_allowed` per application (chained across date-adjacent applications of the same type); `applicable_after` N calendar days of service.
7.1.5 **Earned leave config:** `is_earned_leave`, `earned_leave_frequency ∈ {Monthly, Quarterly, Half-Yearly, Yearly}`, `allocate_on_day ∈ {First Day, Last Day (default), Date of Joining (Monthly only)}`, per-increment `rounding ∈ {none, 0.25, 0.5, 1.0}`.
7.1.6 **Encashment config:** `allow_encashment`, `earning_component` (salary component used to pay), `max_encashable_leaves`, `non_encashable_leaves` (floor that can never be encashed).

### 7.2 Leave Period
7.2.1 Tenant-scoped {from_date, to_date, is_active, optional_holiday_list}. to > from. **No overlapping active periods** per tenant.

### 7.3 Leave Policy & Assignment
7.3.1 Leave Policy = titled set of rows {leave_type, annual_allocation}; each row's allocation must not exceed the type's `max_leaves_allowed`.
7.3.2 **Leave Policy Assignment** grants a policy to an employee for an effective window derived from `assignment_based_on ∈ {Leave Period, Joining Date, Custom}` (Joining Date → DOJ + 12 months default). No overlapping assignments per employee.
7.3.3 On submit, one **Leave Allocation** per non-LWP policy row is created (LWP types never allocate; compensatory types allocate 0 initially). Per-type carry-forward is forced off when the type disallows it. Zero-quantity allocations are skipped unless earned/negative-allowed.
7.3.4 **Mid-period joining pro-rata:** allocation = annual × (days from DOJ to period end) / (days in period), rounded (whole-number for normal types; exact for earned). Capped at annual allocation (except earned+Yearly). Earned types assigned mid-period retro-allocate all already-elapsed sub-periods.
7.3.5 Bulk assignment runs per-employee savepoints with partial-success reporting.

### 7.4 Leave Allocation
7.4.1 Key fields: employee, leave_type, from/to dates, `new_leaves_allocated`, `unused_leaves` (carried in), `total_leaves_allocated`, `carry_forward`, `total_leaves_encashed`, `expired`.
7.4.2 to > from. LWP types cannot be allocated. **No overlapping allocation** per employee+type.
7.4.3 `total = carried-forward unused + new`. Carry-forward pulls the immediately-previous allocation's remaining balance, capped by the type's CF cap; total capped at `max_leaves_allowed` (excess trims the CF part). Submitting a CF allocation **expires the previous allocation's remainder** and stamps how much was carried out.
7.4.4 Over-allocation (> days in period) is a hard error unless the type allows it (then a warning). `max_leaves_allowed` is enforced across all allocations overlapping the leave period.
7.4.5 Post-submit edits to allocated quantity: forbidden for scheduler-owned earned allocations; otherwise allowed but never below leaves already taken (unless negative allowed); every delta writes a ledger entry.
7.4.6 Cancel reverses ledger entries and unwinds CF stamping; blocked if leave applications already consumed within the window.

### 7.5 Leave Ledger — the balance source of truth
7.5.1 `[API]` An **append-only signed ledger** records every movement: transaction_type ∈ {Allocation, Application, Encashment, Adjustment}, ± leaves, from/to dates, flags {is_carry_forward, is_expired, is_lwp}. **Balances are always derived from the ledger, never stored.**
7.5.2 Credits: allocations, CF, positive adjustments, earned-leave increments, comp-off grants. Debits: approved applications, encashments, expiry, negative adjustments.
7.5.3 A CF credit gets its own ledger entry whose validity ends at min(alloc start + CF-expiry-days − 1, alloc end).
7.5.4 Cancelling a source document **deletes** its ledger entries (with the paired expiry entry); deletion is blocked if later applications fall inside the window.
7.5.5 **Daily expiry job:** for allocations whose window (or CF sub-window) has passed, write negative `is_expired` entries for the remaining balance and mark the allocation expired.
7.5.6 **Consumable balance** on a date additionally caps the raw balance by days remaining until allocation/CF expiry (balance 10 but expiring tomorrow → 1 consumable).

### 7.6 Leave Application
7.6.1 Status enum: **Open (default) → Approved | Rejected → (Cancelled)**. Only Approved/Rejected can be submitted; cancel sets Cancelled.
7.6.2 Smallest unit = **half day** (`half_day` + `half_day_date`, which must lie in range and not on a holiday). No quarter days.
7.6.3 `total_leave_days = calendar days − 0.5 (if half day) − holidays in range (unless type includes holidays)`; ≤ 0 → error ("all days are holidays").
7.6.4 **Balance check** (non-LWP): consumable balance (§7.5.6) must cover total days, else hard error — downgraded to a warning if the type allows negative balance. Applications spanning two allocations are only allowed when negative-allowed and the allocations are date-contiguous; ledger entries split per allocation.
7.6.5 Application must fall inside a submitted allocation window (non-LWP, non-negative types).
7.6.6 **Overlap:** no overlapping Open/Approved application per employee; two half-days on one date allowed only while that date's total leave < 1 day.
7.6.7 Other guards: max continuous days (§7.1.4, chained); minimum service days (`applicable_after`); optional-leave dates must be in the period's Optional Holiday List; **block-list dates** warn while Open and hard-block approval (§7.12); LWP cannot overlap a period already covered by a submitted salary slip; cannot apply over days already marked Present/WFH in attendance.
7.6.8 **Backdating control:** if the tenant restricts backdated applications, only users holding the configured exempt role may file from_date < today (no role configured → fully blocked).
7.6.9 **On approval+submit:** attendance auto-marked per day — `On Leave` (or `Half Day` on the half-day date), linked back to the application; pre-existing Absent records are converted. Holidays inside the range (when not counted) get no attendance. On cancel, those attendance records are cancelled.
7.6.10 Ledger: negative entries for the total; split at CF-expiry boundaries; backdated applications against an already-expired allocation also write a compensating reversal.
7.6.11 **Approver & notifications:** approver resolved per §1.7; approver-mandatory per tenant setting. Email notifications (if enabled + templates configured): to approver on creation/update while Open; to employee on approval/rejection/cancellation. In-app notifications per §22.3.

### 7.7 Leave balance reporting semantics
7.7.1 Period report per employee × type: Opening (balance at from−1; boundary case: on an allocation-expiry date the opening counts only the carried-forward part), Allocated, Taken, Expired, Closing = opening + allocated − taken − expired.
7.7.2 As-of-date matrix report: remaining balance per employee × type on a single date.
7.7.3 `[UI]` The self-service leave form must show the type's live balance **before** submission (§21.4).

### 7.8 Earned-leave accrual (daily scheduler)
7.8.1 At policy assignment, the full period's **accrual schedule** is pre-generated: rows {allocation_date, number_of_leaves, is_allocated, allocated_via, attempted, failed, failure_reason}.
7.8.2 The daily job allocates rows due today (skips employees who Left). Per-period amount = annual ÷ {12, 4, 2, 1}, pro-rated for a partial first sub-period, then rounded per the type's rounding (0.25 → nearest quarter, 0.5 → nearest half, 1 → integer).
7.8.3 Allocation dates: first/last day of month/quarter/half-year/year, or the DOJ day-of-month (Monthly).
7.8.4 Caps per increment: cumulative (excl. CF) ≤ policy annual allocation (except Yearly frequency), and ≤ type `max_leaves_allowed`; remaining quota clamps the increment; nothing left → row marked failed.
7.8.5 Failures are logged per row with reason and **HR managers are emailed a digest**; a retry action re-runs failed rows.

### 7.9 Leave Adjustment (manual correction)
7.9.1 `adjustment_type ∈ {Allocate, Reduce}` against a specific allocation; non-zero quantity; **one submitted adjustment per allocation** (amend to change).
7.9.2 Allocate must not push past the type's `max_leaves_allowed`; Reduce must not exceed available balance on the posting date. Writes a signed ledger entry.

### 7.10 Leave Encashment
7.10.1 Status: **Draft → Unpaid | Paid → (Cancelled)** (Paid when payout fully settled).
7.10.2 Requires an active allocation on the encashment date and an encashment-enabled type.
7.10.3 `actual_encashable = max(balance − non_encashable_leaves, 0)`, then `min(…, max_encashable_leaves)`. User may lower but never exceed.
7.10.4 `encashment_amount = days × per-day rate` where the rate comes from the employee's active salary-structure assignment (`leave_encashment_amount_per_day`, fallback to the structure), else 0. Amount must be > 0 to submit.
7.10.5 Payout: **Additional Salary** on the type's earning component (paid via the next payroll) or direct settlement via the accounting seam.
7.10.6 Submit stamps `total_leaves_encashed` on the allocation + negative ledger entry (with reversal if allocation already expired); cancel reverses all.
7.10.7 **Auto-encashment (daily job, tenant toggle):** the day after an allocation of an encashable type expires, a **draft** encashment is auto-created for employees with a salary structure.

### 7.11 Compensatory Leave Request (comp-off)
7.11.1 Grants comp-off for working on holidays: every day in the claimed work range must be a **holiday** for the employee AND have submitted attendance Present/WFH/Half Day (a Half-Day-only day cannot claim a full comp-off). Half-day claims supported.
7.11.2 The comp leave becomes valid from the day **after** the worked range and requires an active leave period covering that date; grant = extend the existing allocation of the compensatory type, or create one running to the leave period's end.
7.11.3 Comp-off expiry = the allocation's end (leave-period end). Cancel decrements the allocation (floored at 0) + reversal ledger entry.
7.11.4 No overlapping comp requests; work dates cannot be in the future.

### 7.12 Leave Block Lists
7.12.1 A block list = named set of {date, reason} rows + an allow-list of exempt users; optional `leave_type` restriction; `applies_to_all_departments` or attached to specific departments.
7.12.2 Applicability = tenant-wide lists + the employee's department's list, filtered by leave type. A weekly-recurrence helper bulk-adds e.g. every-Monday blocks.
7.12.3 Effect: warning while the application is Open; hard error on approval (§7.6.7).

### 7.13 Bulk tool ("Leave Control Panel")
7.13.1 One screen to bulk-grant: mode A = leave-policy assignments; mode B = direct allocations of one type + N days. Date basis ∈ {Leave Period, Joining Date, Custom Range}.
7.13.2 Employee filters (dept/branch/designation/grade/employment type); only Active employees WITHOUT an existing overlapping allocation/assignment are listed. Per-employee savepoints; partial-success completion report.

---

## 8. Attendance & Check-in

### 8.1 Attendance record
8.1.1 **Status enum: `Present | Absent | On Leave | Half Day | Work From Home`** (required on submit). One record per employee per date per shift.
8.1.2 **Half-day sub-status** `half_day_status ∈ {"" , Present, Absent}` describes the *other* half of a Half Day; `modify_half_day_status` marks it provisional (to be finalised by auto-attendance once check-ins arrive).
8.1.3 Links: shift, leave_application + leave_type, attendance_request, overtime_type. Times: in_time, out_time, working_hours; flags late_entry / early_exit.
8.1.4 Validations: date ≥ DOJ; employee not Inactive; **duplicate** rule per §8.1.1 (completed half-day pairings may coexist; row-locked check); **overlapping-shift** rule — same date, different shift allowed only when the two shifts' timings don't overlap (overnight-aware).
8.1.5 **Leave sync on save:** if an approved leave covers the date, status is forced to On Leave (or Half Day on the half-day date) and linked; claiming On Leave/Half Day without a matching leave sets other-half Absent and warns.
8.1.6 Cancel unlinks all associated check-ins. `[UI]` Calendar view merges attendance records with holidays; updates push in real time.

### 8.2 Bulk marking
8.2.1 Per-employee bulk marking of unmarked days (one status), skipping duplicates/overlaps silently; > 10 days → background job, batched.
8.2.2 A one-date-many-employees tool buckets employees into marked / half-day / unmarked (filters incl. shift), and can set status + late/early flags en masse.
8.2.3 CSV import: template pre-fills existing records and auto-labels holiday rows (stripped on import); imports land as submitted; large files run in background.
8.2.4 "Unmarked days" for any range is always clamped to [DOJ, relieving_date] and optionally excludes holidays.

### 8.3 Employee Check-in
8.3.1 A check-in log = {employee, timestamp (second precision), `log_type ∈ {IN, OUT}` (optional per shift mode), device_id, latitude/longitude, geolocation point, skip_auto_attendance, linked attendance, cached shift snapshot (shift, start/end, actual start/end)}.
8.3.2 Duplicate guard: same employee + timestamp + log_type rejected.
8.3.3 A check-in already consumed by attendance cannot have its time edited (cancel the attendance first).
8.3.4 **Auto shift matching:** on save, the log is matched to the shift whose *actual window* (start − early-buffer … end + late-buffer) contains the timestamp; no match → flagged `offshift` (excluded from auto-attendance).
8.3.5 If the matched shift determines IN/OUT "strictly by log type", a missing log_type is an error; in "alternating" mode log_type is optional.
8.3.6 **Geo rules:** when the tenant enables geolocation tracking, every check-in requires lat/long; if the employee's shift assignment carries a **Shift Location** with `checkin_radius > 0`, the Haversine distance from that location must be ≤ radius, else the check-in is rejected ("must be within N meters").
8.3.7 `[API]` **Ingestion API** for devices/mobile: create a check-in by employee key (id or `attendance_device_id`), timestamp, optional log type / device id / coordinates. This is the single integration entry point for biometric devices and the mobile app.
8.3.8 Mobile check-in is gated by tenant toggle `allow_employee_checkin_from_mobile_app` (default ON). `[UI]` The app's check-in panel shows last IN/OUT, a live clock in the confirm modal, and — when geolocation is on — the captured coordinates on a map; location is required to confirm.

### 8.4 Shift Type (configuration)
8.4.1 Required: start_time, end_time (≠). Overnight shifts (start > end) span midnight. Guard: shift length + both check-in buffers must stay under 24 h.
8.4.2 Buffers: `begin_check_in_before_shift_start_time` (default 60 min) and `allow_check_out_after_shift_end_time` (default 60 min) define the **actual window** used for matching (§8.3.4).
8.4.3 Auto-attendance block: `enable_auto_attendance`; `determine_check_in_and_check_out ∈ {Alternating IN/OUT, Strictly by log type}`; `working_hours_calculation_based_on ∈ {First Check-in & Last Check-out, Every Valid Check-in/out pair}`; half-day threshold (hours); absent threshold (hours; wins over half-day); `mark_auto_attendance_on_holidays`; `process_attendance_after` (date floor); `last_sync_of_checkin` (watermark; only shifts fully ended before it are processed) with optional auto-advance.
8.4.4 Late/early: enable flags + grace minutes each side.
8.4.5 Overtime: `allow_overtime` + mandatory overtime type (§17.3).
8.4.6 Shift timing edits are blocked while unprocessed check-ins exist for the shift. Each shift has a display colour for rosters.

### 8.5 Working-hours math (per shift instance)
8.5.1 Group an employee's non-skipped logs by shift instance; hours = (a) alternating+first/last: last − first; (b) alternating+every-pair: Σ consecutive pairs; (c) strict+first/last: last OUT − first IN; (d) strict+every-pair: Σ IN→OUT pairs. 2-dp rounding.
8.5.2 Adjacent shifts with overlapping buffer windows are trimmed so a punch can only belong to one shift.

### 8.6 Auto-attendance processing (hourly job per enabled shift)
8.6.1 Eligible logs: not skipped, unlinked, ≥ `process_attendance_after`, shift ended before the sync watermark, not offshift.
8.6.2 Status per shift instance: hours < absent-threshold → **Absent**; else < half-day threshold → **Half Day**; else **Present**. Late/early flags per grace config.
8.6.3 Holidays are skipped unless the shift marks attendance on holidays.
8.6.4 On a half-day holiday, both thresholds are **halved**.
8.6.5 A provisional Half Day (from leave, §8.1.2) is updated in place: worked hours recorded and the other half resolved Present/Absent.
8.6.6 **Overtime capture:** if the shift allows OT and status is Present and hours > standard shift duration, the attendance stores `actual_overtime_duration = hours − standard`.
8.6.7 Attendance links back to its constituent check-ins. Failing groups are skipped-and-flagged (with reason) so one bad record never blocks the run; processing is batched with periodic commits.
8.6.8 **Absent for missing check-ins:** working days (post-DOJ, pre-relieving, non-holiday, within processed window) with zero attendance are auto-marked Absent — but only from **one day after** the shift day, leaving a grace window for manual records. Same pass finalises unresolved half-day "other halves" as Absent.
8.6.9 The watermark auto-advance job moves `last_sync_of_checkin` past each completed shift when auto-advance is on.

### 8.7 Attendance Request (regularisation / WFH / on-duty)
8.7.1 Fields: date range (backdating allowed), optional half-day + date, `include_holidays`, optional shift, **reason ∈ {Work From Home, On Duty}** + explanation. No approver — submission itself creates attendance. `[STD]` If tenant governance requires approval on regularisation, configure a workflow (§23.6); default is direct.
8.7.2 If the employee has multiple shift assignments in range and none specified → error; exactly one → auto-filled.
8.7.3 Per-day outcome: skip holidays (unless included) and days on approved leave; else create/update attendance with status Half Day (on the half-day date) / Work From Home / Present.
8.7.4 Overwrite semantics: an existing different-status record is updated in place (linked to the request, comment trail); a Half-Day-Absent other-half flips to Present when the request covers it. If **no** day in the range would produce a change, submission is rejected with a per-day reason table.
8.7.5 Cancel cancels the attendance it created. Overlapping requests per employee(+shift) are rejected.

---

## 9. Shift Management & Rostering

### 9.1 Shift Assignment
9.1.1 Fields: employee, shift_type, start_date, optional end_date (open-ended), `status ∈ {Active, Inactive}`, shift_location, provenance links (shift_request / schedule assignment).
9.1.2 Overlap rules: date-overlapping assignments are blocked when their shift **timings** overlap; if timings don't overlap, allowed only when the tenant enables `allow_multiple_shift_assignments` (else blocked; with the toggle on, a warning). Checked again on post-submit edits.
9.1.3 Cancellation is blocked while any check-in or attendance for that employee+shift exists inside the assignment window.
9.1.4 Daily job: assignments whose end_date has passed flip to **Inactive**.
9.1.5 Effective shift on a date = latest Active submitted assignment starting on/before the date; fallback = employee's `default_shift`.

### 9.2 Shift Request
9.2.1 Fields: shift_type, employee, from_date (to_date optional/open), approver (required), `status ∈ {Draft → Approved | Rejected}`.
9.2.2 Approver must be one of the valid approvers per §1.7 (shift flavour); requesting one's **default shift** is rejected; overlapping requests with overlapping timings are rejected.
9.2.3 Only users with approval rights may move status off Draft. On Approved+submit → a Shift Assignment is auto-created (linked); cancel cancels it. In-app notifications per §22.3.

### 9.3 Shift Schedules (repeating rosters)
9.3.1 Schedule = shift_type + `frequency ∈ {Every Week, Every 2/3/4 Weeks}` + repeat weekdays. Identical schedules are reused rather than duplicated.
9.3.2 **Schedule Assignment** binds employee → schedule (+ location, enabled flag, `create_shifts_after` watermark). Finite date range → generated once, disabled thereafter; open-ended → enabled, auto-extended.
9.3.3 Generation coalesces consecutive repeat-days into single assignments, skips (N−1) weeks between active weeks for multi-week frequencies, and generates 90 days ahead for open-ended schedules; hourly job tops up whenever the watermark reaches today.
9.3.4 Deleting a schedule assignment cancels+deletes all shift assignments it generated. The watermark cannot move backwards past generated assignments.

### 9.4 Bulk shift tool
9.4.1 Actions: **Assign Shift** (excludes employees whose existing assignments would conflict per §9.1.2), **Assign Shift Schedule** (excludes employees already on a schedule sharing repeat-days, timing-aware), **Process Shift Requests** (bulk approve/reject drafts).
9.4.2 Eligibility: Active employees employed for the full assignment window. > 30 employees → background job; per-employee savepoints; live progress + success/failure report.

### 9.5 Roster board (manager UI)
9.5.1 `[UI]` A month-view grid: rows = employees, columns = days; cells show colour-coded shift blocks, holidays and approved leaves. Filters: employee dimensions (status/department/branch/designation) + shift dimensions (type/location); a company/tenant context is required to render.
9.5.2 Creating from a cell: single-day → immediate assignment with **smart-merge** (extends an adjacent assignment of the same shift/status/location instead of fragmenting; absorbs a following adjacent one); ranges ≥ 7 days with custom repeat-days or multi-week frequency → creates a shift schedule assignment instead.
9.5.3 Editing a block: only status and end_date are editable. Deletion granularity: single day (splits the assignment: end the old at date−1, re-create the remainder from date+1), whole consecutive run, or the entire schedule assignment.
9.5.4 **Drag-and-drop swap:** dragging a shift block onto another employee/day moves it (breaking it out of its source run); if the target cell already has a shift, the two swap. Identical source/target shifts are a no-op; same-shift cells are not droppable.
9.5.5 `[API]` Board data endpoint returns per-employee merged {holidays, approved leaves, submitted shift assignments} for the month; only whitelisted filter keys are accepted.

---

## 10. Recruitment

### 10.1 Staffing Plan (headcount & budget)
10.1.1 Plan = {date range, optional department} + rows per **designation**: vacancies, estimated_cost_per_position; computed: current_count (Active employees with that designation), current_openings, number_of_positions = vacancies + current_count, total_estimated_cost = vacancies × cost. Header budget = Σ row costs.
10.1.2 **One active plan per designation per overlapping date window** per tenant (hard block).
10.1.3 A "Get Job Requisitions" action seeds plan rows from approved requisitions (positions → vacancies, expected compensation → cost).
10.1.4 `[ADAPT]` Reference parent/subsidiary-company ceiling rules deferred with multi-entity support (§1.2).

### 10.2 Job Requisition
10.2.1 Fields: designation (req), department, no_of_positions (req), expected_compensation (req), requested_by (employee), posting_date, expected_by, description, reason.
10.2.2 **Status enum: `Pending → Open & Approved → Filled | Rejected | On Hold | Cancelled`.** "Open & Approved" is the approval gate — only then can a Job Opening be created from / associated with the requisition.
10.2.3 Soft duplicate warning when an open requisition already exists for the same designation + department + requester (confirm-to-proceed).
10.2.4 `completed_on` mandatory at Filled; `time_to_fill = completed_on − posting_date` feeds analytics (average time-to-fill).
10.2.5 The requisition auto-flips to **Filled** when its linked Job Opening closes (§10.3.5).

### 10.3 Job Opening
10.3.1 **Status enum: `Open | Closed`.** Fields: job_title, designation, department, employment_type, location, description, vacancies (from requisition), salary range {currency, lower, upper, per Month/Year} + `publish_salary_range`, `publish_applications_received` (default on), publish flag + unique public route, closes_on.
10.3.2 Date sanity: posted_on ≤ closes_on (Open) / ≤ closed_on (Closed).
10.3.3 **Staffing-plan gate:** on save, the active plan for the designation is resolved; if the plan's `number_of_positions` ≤ current filled+open count, opening creation is rejected ("hiring complete per staffing plan").
10.3.4 Templates pre-fill openings (designation, dept, employment type, location, salary range, description).
10.3.5 Transitions: Open→Closed stamps closed_on (clears closes_on) and marks the linked requisition **Filled**; Closed→Open clears closed_on. **Daily job auto-closes** openings past their closes_on.

### 10.4 Job Applicant
10.4.1 **Status enum: `Open | Replied | Shortlisted | Hold | Rejected | Accepted`.** Keyed by email; the same email may re-apply (suffix disambiguation).
10.4.2 Sources: careers portal (default source "Website Listing"), manual entry, **inbound email** (auto-creates applicants from an applications mailbox), employee referral (§10.5).
10.4.3 Cannot apply against a **Closed** opening. Applicant name auto-derives from the email when blank.
10.4.4 `[UI]` Pipeline Kanban grouped by status (default columns Open / Replied / Shortlisted / Accepted); quick actions: Shortlist, Reject, Create Interview (unless Rejected/Accepted), Create Job Offer (when Accepted); applicant rating stars; per-applicant interview summary (round, date, average rating, status).
10.4.5 Applicant-status ↔ referral sync per §10.5.4; applicant→employee conversion per §10.8.

### 10.5 Employee Referral
10.5.1 Fields: candidate name/email/contact, for_designation, referrer (Active employee), résumé, `is_applicable_for_referral_bonus` (default on), payment status. **One referral per candidate email.**
10.5.2 **Status enum: `Pending → In Process → Accepted | Rejected | Cancelled`** (system-managed).
10.5.3 Actions on a Pending referral: Reject, or **Create Job Applicant** (copies details, source = "Employee Referral", links back; referral → In Process).
10.5.4 The linked applicant's status drives the referral: Open/Replied/Hold → In Process; Accepted/Rejected → same value.
10.5.5 **Referral bonus:** when referral is Accepted + bonus-applicable, a one-time **Additional Salary** for the referrer can be created (one per referral, enforced); its submit/cancel flips referral payment status Paid/Unpaid.

### 10.6 Interviews
10.6.1 **Interview Type** = named round definition: default interviewer users, expected average rating, optional designation restriction, expected skill set.
10.6.2 **Interview**: type + applicant (+ auto opening/designation), schedule {date, from, to — set once}, interviewer list, `status ∈ {Pending → Under Review → Cleared | Rejected | Cancelled}`.
10.6.3 One interview per applicant per round type. The round's designation must match the applicant's.
10.6.4 Only **Cleared/Rejected** interviews can be submitted; on submit the system offers to update the applicant (Cleared→Accepted, Rejected→Rejected).
10.6.5 **Reschedule** (while Pending): updates the slot and emails old→new timing to all interviewers + the applicant.
10.6.6 **Interview Feedback** (per interviewer): per-skill star ratings, overall result (Cleared/Rejected), text. Guards: only assigned interviewers; not before the interview date; one submitted feedback per interviewer.
10.6.7 Scores: feedback average = mean of its skill ratings; the interview's `average_rating` = mean of all submitted feedbacks, recomputed on every feedback submit/cancel. `[UI]` Display as 5-star scale with per-skill averages and a reviews-per-rating histogram.
10.6.8 **Reminders:** interviewers + applicant are emailed a reminder `remind_before` (default 15 min) ahead of Pending interviews (sent once per interview); a **daily** reminder chases interviewers of Under-Review interviews who haven't submitted feedback. Both governed by tenant toggles + templates (§22.2).

### 10.7 Job Offer & Appointment Letter
10.7.1 **Job Offer status enum: `Awaiting Response → Accepted | Rejected | Cancelled`** (status editable post-submit). Fields: applicant, offer_date, designation, terms table (from reusable term templates), print/letterhead.
10.7.2 **One non-cancelled offer per applicant.** Offer status syncs the applicant's status (Accepted/Rejected).
10.7.3 **Vacancy gate at offer time** (tenant toggle `check_vacancies`, default OFF): with an active staffing plan covering the offer date, remaining vacancies (plan − already-offered) must be > 0.
10.7.4 **Appointment Letter**: applicant + appointment_date + template-driven {introduction, term paragraphs, closing notes}; templates are tenant-managed.

### 10.8 Applicant → Employee conversion
10.8.1 "Create Employee" unlocks only when the offer is submitted AND Accepted AND no employee is linked yet.
10.8.2 Mapping: applicant name/email → employee identity; offer date → scheduled confirmation date.
10.8.3 Employee creation back-syncs applicant (→Accepted) and the latest offer (→Accepted, prompting completion if draft) (§3.2.1); onboarding gate per §4.2.4.

### 10.9 Public careers portal
10.9.1 `[UI]` Public listing page (per tenant): only **Open + published** openings; 20/page; filters company/department/employment-type/location; free-text search over title/description; sort by posting date. Cards show salary range and applications-received count only when the opening publishes them, plus relative posted-date and closing date.
10.9.2 Public application form (no login): applicant name*, email*, phone, country, cover letter, résumé link (URL-validated) or file upload, salary expectation. Success → thank-you + redirect to listings. Applications from the portal default source "Website Listing".
10.9.3 `[API]` Openings render at their unique public route; application counts computed live.

### 10.10 Recruitment analytics
10.10.1 Pipeline report: Staffing Plan → Openings → Applicants (+status) → Offers (+status/date). KPIs: applicant-to-hire % = Accepted/total applicants; offer acceptance % = accepted/total submitted offers; average time-to-fill from requisitions.

---

## 11. Performance (Appraisals, Goals, Feedback)

### 11.1 Appraisal Cycle
11.1.1 **Status enum: `Not Started → In Progress → Completed`** (action-driven: Start / Mark as Completed / reopen to In Progress). Completion is blocked while any appraisal in the cycle is still Draft.
11.1.2 **A Completed cycle is frozen**: no appraisal, goal, or feedback in it can be created or modified (reopen to amend).
11.1.3 Multiple In-Progress cycles are allowed; features needing "the" active cycle use the latest-started one.
11.1.4 `kra_evaluation_method ∈ {Automated Based on Goal Progress (default), Manual Rating}` — locked once any appraisal exists in the cycle.
11.1.5 Appraisee list: "Get Employees" pulls Active employees per cycle filters (dept/branch/designation) and auto-assigns each their designation's appraisal template (warn if missing; per-row override allowed). Re-running replaces the list.
11.1.6 "Create Appraisals" creates one appraisal per appraisee, idempotently (existing ones skipped); > 30 → background job.
11.1.7 Optional **final-score formula** per cycle (expression over goal_score, average_feedback_score, self_appraisal_score + employee/cycle fields) replacing the default average (§11.3.5).
11.1.8 Cycle dashboard counters: appraisees, self-appraisals pending, employees without goals, employees without feedback.

### 11.2 Appraisal Template
11.2.1 Template = **KRAs with weightages** (must total exactly 100) + **rating criteria with weightages** (must total 100). The same criteria set drives both self-appraisal and 360° feedback. Templates attach to Designations.

### 11.3 Appraisal (per employee per cycle)
11.3.1 One appraisal per employee per cycle (and no overlapping-period duplicates). KRA/criteria weightage totals re-validated (=100) at submission.
11.3.2 **Goal score — automated mode:** per KRA, goal_completion% = average progress of the employee's **top-level, non-archived** goals tagged to that KRA in the cycle; weighted sum → 0–100; `total_score = /20` → **0–5 scale**. Recomputed live whenever a goal changes (§11.5.6).
11.3.3 **Goal score — manual mode:** per-KRA score 0–5 × weight → total_score (0–5).
11.3.4 **Self-appraisal:** per-criterion star rating × 5 × weight → `self_score` (0–5); employee also writes free-text reflections. Pending while 0.
11.3.5 **Final score** = mean(total_score, avg_feedback_score, self_score) — or the cycle's formula. All scores 0–5, precision-rounded, recomputed on every save.
11.3.6 `[UI]` Appraisal shows a feedback timeline (reviewer, score, when) and a 1–5 score-distribution strip.

### 11.4 Employee Performance Feedback (360°)
11.4.1 Reviewer = another Active employee (self-feedback rejected — self-appraisal is separate); appraisal must belong to the subject employee; cycle must not be Completed.
11.4.2 Ratings per template criterion (weightages = 100); `total_score = Σ rating × 5 × weight` (0–5). Feedback text mandatory.
11.4.3 Submitted feedbacks average into the appraisal's `avg_feedback_score` on submit **and** cancel.

### 11.5 Goals
11.5.1 Goals form a **tree** per employee; group nodes aggregate children. Fields: name, employee, dates (default to cycle span), progress %, KRA, appraisal cycle.
11.5.2 **Status enum: `Pending | In Progress | Completed` (auto from progress: 0 / 1–99 / 100) + manual `Archived | Closed`.** Progress > 100 rejected; setting Completed forces 100.
11.5.3 Top-level goals in a cycle must be tagged to a KRA from the employee's appraisal for that cycle; children inherit parent's KRA/cycle/employee (KRA change on a parent cascades to children).
11.5.4 Group-node progress = mean of direct children (Archived excluded; Closed still counted), rolled up recursively; recomputed on save/delete/re-parent.
11.5.5 Goals in Completed cycles are immutable.
11.5.6 **Cascade:** every goal save/delete recomputes the linked appraisal's KRA completion, total and final scores in real time.
11.5.7 `[UI]` Tree hides Archived by default; groups show "X of Y completed".

---

## 12. Skills & Training

12.1 **Skill map:** one per employee — rows {skill, proficiency (stars), evaluated_on} + attended trainings. Picking a designation pre-loads its expected skills at minimum proficiency.
12.2 Designations carry expected skills (§2.2.1); interview rounds rate a defined expected-skill set (§10.6).
12.3 **Training Program** (container; status Scheduled/Completed/Cancelled) → **Training Events**: type ∈ {Seminar, Theory, Workshop, Conference, Exam, Internet, Self-Study}; optional level {Beginner, Intermediate, Advance}; certificate flag; location + start/end (end > start); attendee list (Active employees, no duplicates); `is_mandatory` per attendee.
12.4 Attendee status: **Open → Invited → Completed → Feedback Submitted**; attendance per attendee: Present/Absent.
12.5 Event submission emails all attendees a "Training Scheduled" notice (with mandatory-attendance note). Event completion flips Present attendees to Completed; reverting the event to Scheduled resets attendees to Open.
12.6 **Training Result** (one per event): per-attendee hours, grade, comments; submitting it marks the event + attendees Completed and emails attendees a feedback request.
12.7 **Training Feedback:** only listed participants who were not Absent may submit; submission sets that attendee to Feedback Submitted (reverted on cancel).

---

## 13. Daily Work Summary (email stand-up)

13.1 A summary **group** = user list + daily send hour + subject/message + holiday list; disabled users excluded; requires a tenant inbound-email account (replies are collected by email).
13.2 Hourly job: at each group's configured hour (skipping the group's holidays) create a summary record (status **Open**) and email every member the prompt (default "What did you work on today?"); replies thread onto the record.
13.3 Daily digest job: for each Open summary, compile all replies (quoted history stripped, attributed with avatars), list non-responders, email the digest to all members, and mark the summary **Sent**.

---

## 14. Expense Claims, Advances, Travel & Vehicles

### 14.1 Expense Claim — model
14.1.1 Header: employee, posting_date, `expense_approver`, cost attribution (cost_center/project/task), multi-currency (currency + exchange_rate; base twins per §1.9).
14.1.2 Lines (≥1): expense_date, **Expense Claim Type**, description, `amount` (claimed), `sanctioned_amount` (defaults = claimed; approver may only reduce). Rejection forces all sanctions to 0.
14.1.3 Optional **taxes** rows (rate → tax computed **on sanctioned total**) and **advances** rows (§14.3).
14.1.4 **Expense Claim Type** master: unique name, description, per-company default expense account (accounting seam); claims fail fast if the type lacks an account mapping.

### 14.2 Expense Claim — lifecycle & math
14.2.1 `approval_status ∈ {Draft → Approved | Rejected}` (approver-only field) — must be non-Draft to submit. Derived display status: **Draft | Submitted | Unpaid | Paid | Rejected | Cancelled**; Paid when reimbursed in full, when `is_paid` is flagged (paid immediately outside payroll), or when grand_total = 0 (fully covered by advances).
14.2.2 `grand_total = Σ sanctioned + Σ taxes − Σ allocated advances` (≥ 0 enforced through allocation caps); reimbursement records reduce the outstanding = sanctioned + taxes − reimbursed − advances; over-payment is blocked.
14.2.3 Approver resolution & mandatory/self-approval rules per §1.7 + §22.4; in-app notifications per §22.3.
14.2.4 Cancel reverses postings, re-computes linked-advance consumption, and unlinks payments.

### 14.3 Advance clearing inside a claim
14.3.1 Only the same employee's advances in the **same currency**, submitted, paid, and not fully consumed are linkable; allocation per advance ≤ its unclaimed remainder (paid − claimed − returned); Σ allocations ≤ sanctioned + taxes.
14.3.2 `[UI]` Auto-allocation waterfall fills advances oldest-first up to the claim total.
14.3.3 On claim approval+submit, each advance's `claimed_amount` (and status) recomputes from all approved claims referencing it.
14.3.4 Multi-currency advances allocated at a different rate produce an **exchange gain/loss** record (accounting seam).

### 14.4 Employee Advance
14.4.1 Fields: employee, posting_date, purpose, `advance_amount`, currency, mode_of_payment, `repay_unclaimed_amount_from_salary` toggle.
14.4.2 **Status enum (derived): `Draft | Unpaid | Partially Paid | Paid | Claimed | Returned | Partly Claimed and Returned | Cancelled`** — precedence: fully claimed → Claimed; fully returned → Returned; claimed+returned = paid → Partly …; paid = requested → Paid; some paid → Partially Paid; else Unpaid.
14.4.3 Caps: paid ≤ advance amount; returns ≤ paid − claimed. `pending_amount` surfaces the employee's earlier unpaid advances at creation.
14.4.4 Unclaimed-money recovery, two exclusive paths: (a) toggle OFF → direct repayment record for paid − claimed (accounting seam); (b) toggle ON → **salary deduction**: an Additional Salary deduction (ref = the advance) recovered through payroll; a banner shows scheduled-vs-recovered amounts. Deduction validation ensures scheduled deductions never exceed the recoverable remainder.
14.4.5 Payment disbursement, claim creation shortcuts, and cancel-unlink behaviour follow the seam rules; tenant toggle controls payment unlinking on cancel.

### 14.5 Travel Request
14.5.1 Pure data-capture (no approval flow, no financial effect): travel_type {Domestic, International}, funding {Require Full Funding, Fully Sponsored, Partially Sponsored…}, purpose, identification/passport details, sponsor details, attachments.
14.5.2 **Itinerary rows**: from/to, mode {Flight, Train, Taxi, Rented Car}, departure/arrival, meal preference, advance-required + amount, lodging-required + area/check-in/out. **Costing rows**: expense type, sponsored/funded/total amounts. Informational only. `[STD]` If travel approval is required, attach a workflow (§23.6).

### 14.6 Vehicle Log & vehicle expenses
14.6.1 Vehicle log (per vehicle+employee+date): odometer (must be ≥ vehicle's last recorded), refuelling {qty, price, supplier, invoice}, service rows {item, type ∈ Inspection/Service/Change, frequency, expense}.
14.6.2 Submit advances the vehicle's odometer; cancel rolls it back.
14.6.3 "Create Expense Claim" from a log: amount = services + fuel (qty × price); one claim per log; cancelling the log deletes/cleans a draft claim that only contains vehicle rows.
14.6.4 Reports: advance summary (outstanding = paid − claimed − returned), unpaid claims, monthly fuel-vs-service vehicle expense chart.

---

## 15. Payroll Foundation

### 15.1 Payroll Period
15.1.1 Tenant-scoped {start, end}; start ≤ end; **no overlapping periods**. It is the fiscal container for annual tax projection, YTD windows, benefit accrual, declarations, and mid-year opening balances.
15.1.2 `[API]` **Sub-period factor** (total & remaining pay cycles in the period, clamped to joining/relieving; exact month-diff for monthly non-payment-days components, day-ratio otherwise) is a single shared function — the tax spread (§18.4) and benefit proration (§19) both depend on it.

### 15.2 Payroll Settings (per tenant)
15.2.1 `payroll_based_on ∈ {Leave (default), Attendance}` — where LWP/absence comes from.
15.2.2 `consider_unmarked_attendance_as ∈ {Present, Absent}` (Attendance mode).
15.2.3 `include_holidays_in_total_working_days` (default off — holidays excluded from the denominator; enabling lowers the per-day rate) and `consider_marked_attendance_on_holidays` (default off — holidays paid even if marked absent).
15.2.4 `daily_wages_fraction_for_half_day` (default **0.5**); `disable_rounded_total`; `show_leave_balances_in_salary_slip`; `max_working_hours_against_timesheet` (soft alert).
15.2.5 Slip emailing: `email_salary_slip_to_employee` (default on), optional PDF encryption with a `password_policy` pattern (e.g. `SAL-{first_name}-{date_of_birth.year}`), sender/cc/template.
15.2.6 `process_payroll_accounting_entry_based_on_employee` (per-employee vs aggregated payable — accounting seam); `mandatory_benefit_application` (§19.2); `create_overtime_slip` (adds the overtime step to payroll runs).

### 15.3 Salary Component
15.3.1 Unique name + **abbreviation** (auto-initials, de-duplicated) — the formula token. `type ∈ {Earning, Deduction, Employer Contribution}`.
15.3.2 Value source: fixed `amount` OR `formula` (when `amount_based_on_formula`); optional `condition` expression — falsey ⇒ the component is skipped entirely.
15.3.3 Behaviour flags: `depends_on_payment_days` (default on ⇒ prorated by payment days); `statistical_component` (computed & referenceable, never paid/totalled); `do_not_include_in_total`; `do_not_include_in_accounts`; `remove_if_zero_valued` (default on); `round_to_the_nearest_integer`; `disabled`.
15.3.4 Tax flags: earnings — `is_tax_applicable` (default on), `deduct_full_tax_on_selected_payroll_date`; deductions — `variable_based_on_taxable_salary` (the slab-computed tax component; cannot also carry amount/formula), `exempted_from_income_tax` (pre-tax deduction), `is_income_tax_component` (reporting).
15.3.5 `arrear_component` — participates in arrears/correction differencing (§17.4); incompatible with the tax-variable flag.
15.3.6 `accrual_component` / flexible-benefit flags per §19.
15.3.7 Per-company default account rows (accounting seam); a non-statistical component without accounts draws a warning.
15.3.8 `[API]` Formula/condition text is sanitised; editing a component may bulk-sync the change into all structures referencing it. Safe-eval whitelist (everywhere formulas run): int, float, round, rounded, date, getdate, get_first_day, get_last_day, ceil, floor, min, max — no attribute access, admin-authored only.

### 15.4 Salary Structure (template)
15.4.1 `record_state`-bearing template: currency, `is_active`, `payroll_frequency ∈ {Monthly (default), Fortnightly, Bimonthly, Weekly, Daily}`, mode-of-payment/payment account (seam), earnings/deductions/employer-contribution rows, flexible-benefit rows, `max_benefits`, `leave_encashment_amount_per_day`.
15.4.2 Row flags are overlaid from the component master on save (source of truth for behaviour flags; amount/formula preserved when locally set).
15.4.3 Hard validations: a tax-variable deduction may not carry amount/formula; a payment-days-dependent formula may not reference another payment-days-dependent component's abbr (double-proration guard).
15.4.4 Timesheet-based structures (hour_rate × hours overwrite a designated wage component) are a **seam** (§0.11).
15.4.5 Bulk assignment from a structure: >20 employees → background; skips duplicates; per-employee savepoints. Preview slips can be rendered without persisting.

### 15.5 Salary Structure Assignment (SSA)
15.5.1 Binds employee → structure **from a date**: base & variable amounts, income-tax slab (mandatory when the structure carries a tax component; slab currency must match), payroll cost-center split (must total 100%), per-employee benefit rows, `leave_encashment_amount_per_day` override, payable-account seam.
15.5.2 One submitted SSA per employee per from_date; from_date within employment window; the **active SSA for any date is the latest one starting on/before it**.
15.5.3 **Mid-year joiners:** when the SSA starts after the payroll period began and the structure taxes income, opening balances `taxable_earnings_till_date` + `tax_deducted_till_date` (previous employer) should be captured (warning if blank) — they feed the annual projection (§18.3).
15.5.4 CTC preview: periods/year = {Monthly 12, Fortnightly 26, Bimonthly 24, Weekly 52, Daily 365}; annual gross = per-cycle payable earnings × periods; CTC adds non-payable earnings + employer contributions.
15.5.5 `[API]` The SSA **pre-evaluates every structure row once** against a full-cycle context (payment days = full) producing `default_amount` per row; slips consume these and apply proration only — formulas are not re-evaluated per slip except through this shared engine.
15.5.6 Bulk SSA tool: Active employees employed at from_date without an SSA on that date; base defaults from grade; >30 → background.

---

## 16. Salary Slip Engine

### 16.1 Identity & period
16.1.1 **Status: `Draft | Submitted | Cancelled | Withheld`** (Withheld when caught by a salary-withholding cycle — §17.2).
16.1.2 Period derivation from frequency: Monthly = calendar month; Bimonthly = 1–15 / 16–end; Weekly = start+6; Fortnightly = start+13; Daily = single day.
16.1.3 Guards: start ≤ end; end ≥ joining; relieving (if set) ≥ start — an employee relieved before the period must be status Left or slip creation fails. **One slip per employee per period** (scoped to its payroll run). Effective window clamps: actual_start = max(start, joining), actual_end = min(end, relieving).

### 16.2 Working days & payment days (the proration core)
16.2.1 `total_working_days` = period days − holidays (holidays included instead when the tenant opts in — §15.2.3).
16.2.2 `payment_days` seeds from the clamped window minus holidays (per the same setting), then subtracts:
- **Leave mode:** LWP day-equivalents from approved LWP/PPL leave (half-days count `1 − half_day_fraction`; PPL days weighted by `1 − fraction_of_daily_salary_per_leave`).
- **Attendance mode:** LWP-type On-Leave/Half-Day attendance (same weights) + **Absent** days + unmarked days when `consider_unmarked_attendance_as = Absent` + half-day-absent fractions. Absences on holidays deduct only when the tenant opts in.
16.2.3 Payroll-correction credits add back reversed LWP days, cross-checked against submitted corrections (§17.4.4).
16.2.4 A manually overridden LWP differing from the computed value warns but wins.

### 16.3 Component evaluation
16.3.1 Resolution: the latest active submitted SSA/structure matching the employee + period (+ frequency). No structure → slip renders empty with a "structure missing" notice.
16.3.2 Compute order: sub-period factor → earnings → gross → deductions → loan seam → regional statutory hook → net pay → income-tax breakup (§18.6). Earnings and deductions share one evaluation context so formulas can cross-reference (deductions can use `gross_pay`, any abbr, `base`, employee fields, slip fields — slip wins name collisions; all abbrs zero-seeded).
16.3.3 **Proration formula:** for payment-days-dependent components, `amount = default_amount × payment_days / total_working_days` (likewise the additional-salary part). Zero payment days ⇒ 0. Integer-rounding flag applies. Zero rows drop when `remove_if_zero_valued`.
16.3.4 Statistical/accrual components compute into the context (prorated the same way) but never post as paid rows; accruals write to the benefit ledger (§19.4).

### 16.4 Additional Salary (one-off & recurring overlays)
16.4.1 Fields: employee, component, amount ≥ 0, **one-off** `payroll_date` XOR **recurring** from/to (+ `disabled` kill-switch), `overwrite_salary_structure_amount` (default on), full-tax flag, reference to the source record (incentive, referral bonus, gratuity, encashment, claim, arrear, correction, overtime, advance recovery).
16.4.2 Guards: an SSA must exist at the effective date; overwrite only valid when the component exists in the structure; **no two overwrites of one component in one period**; recurring entries of the same component may not overlap; dates clamp to employment window. A tax-variable component may only be overridden with overwrite on (warning) — never stacked.
16.4.3 Slip merge: overwrite ⇒ replaces the structure row (delta tracked as `additional_amount`); non-overwrite ⇒ adds on top. **No payment-days proration** for amounts referencing Arrear / Payroll Correction / Benefit Claim / accrual components (already period-correct). Recurring entries spanning a partial slip window pay `amount ÷ window days × overlap days`.
16.4.4 Downstream flags: referral-bonus payment status and advance-recovery balances update on submit/cancel (§10.5.5, §14.4.4).

### 16.5 Totals & lifecycle
16.5.1 `gross_pay` = Σ payable earnings; `net_pay = gross − (deductions + loan seam)`; `rounded_total = round(net)`; base twins × exchange rate; amount-in-words. Component-wise + slip-level **YTD** (within payroll period, else fiscal year) and **MTD** aggregates are stored on the slip.
16.5.2 **Negative net pay cannot be submitted** (bulk runs skip such slips and report them).
16.5.3 Submit effects: leave-balance table snapshot (tenant toggle); gratuity/encashment additional-salaries flip Paid; benefit-ledger entries post; slip emailed to the employee's preferred email (individually — or in bulk after a payroll run), PDF optionally password-protected (§15.2.5). Cancel reverses each effect.
16.5.4 `[UI]` Slip detail = tabs: Details / Earnings & Deductions / Net-pay info / **Income-Tax Breakup** (§18.6) / Bank details; PDF download.

---

## 17. Payroll Run, Withholding, Overtime, Arrears & Bonuses

### 17.1 Payroll Entry (bulk run)
17.1.1 **Status: `Draft | Queued | Submitted | Failed | Cancelled`.** Scope: period (frequency-derived), filters {branch, department, designation, grade}, currency + exchange rate, cost attribution, `validate_attendance` toggle, final-cycle proof-enforcement flag (§18.5.3).
17.1.2 **Employee pool:** Active-enough employees with a matching submitted SSA (currency/frequency), employed within the window, minus those already payrolled for it; withheld employees flagged.
17.1.3 Draft-slip generation and slip submission each go **background beyond 30 employees** with per-employee error isolation (one failure never kills the run; failures logged + reported; run flips Failed on systemic error).
17.1.4 With `validate_attendance` on, submission is blocked while any pooled employee has unmarked attendance days in the window.
17.1.5 Negative-net slips are skipped at submission and reported (§16.5.2).
17.1.6 Withheld slips are excluded from disbursement and settle later via the release flow (§17.2.4).
17.1.7 Cancelling a run (background beyond 50 slips) cancels+deletes its slips and its accounting-seam records.
17.1.8 **Accrual posting (accounting seam):** per run — earnings vs deductions aggregated by (account, cost-center split), net payable per employee or aggregated per tenant setting; statistical and excluded components stay out.
17.1.9 **Disbursement record (accounting seam):** Σ (earnings − deductions − loan recovery) of submitted, non-withheld slips; withheld salaries settle through their own release record.

### 17.2 Salary Withholding
17.2.1 Withhold an employee's salary for N cycles from a date (e.g., notice-period disputes): cycles are carved by frequency ({Monthly +1 mo, Bimonthly +2 mo, Fortnightly +14 d, Weekly +7 d, Daily +1 d}); to_date = from + N cycles − 1 day.
17.2.2 **Status: `Draft | Withheld | Released | Cancelled`** — Released once every cycle is released. No overlapping withholdings per employee.
17.2.3 A slip whose period exactly matches a withholding cycle is stamped **Withheld** and skipped by disbursement.
17.2.4 **Release:** a withheld-salary disbursement record ties into the cycles; confirming it marks cycles released and flips slips back to Submitted (reversal on void). Withheld net pay also surfaces in Full & Final (§6.3.2).

### 17.3 Overtime
17.3.1 **Overtime Type:** payout component, max OT hours/day, `calculation_method ∈ {Salary Component Based (default), Fixed Hourly Rate}` (+ hourly rate or the applicable component set), `standard_multiplier`, optional weekend multiplier, optional public-holiday multiplier.
17.3.2 OT hours originate from **auto-attendance** (§8.6.6): Present days on OT-enabled shifts store `actual_overtime_duration`.
17.3.3 **Overtime Slip** (per employee per pay window): rows pulled from OT-bearing attendance (capped at max hours/day); no overlapping slips; no duplicate dates; per-row amount = duration × hourly rate × multiplier (weekend on weekly-offs; public-holiday on non-weekly-off holidays; else standard).
17.3.4 Hourly rate: fixed, or Σ(selected components on a preview slip) ÷ payment days ÷ standard daily hours.
17.3.5 Submission pays out via per-component **Additional Salary** (non-overwrite, ref = the OT slip). Payroll runs can bulk-create/submit OT slips as a pre-step when the tenant enables it (>30 → background).

### 17.4 Arrears & Payroll Correction
17.4.1 **Arrear** (retroactive structure change): scoped to employee + new structure + payroll period, effective from a date inside the period; requires the new SSA; one arrear per employee/structure/period.
17.4.2 Arrear difference = (preview slips under the new structure, honouring historical LWP) − (amounts already paid) per **arrear-flagged** component; only positive diffs pay out — via Additional Salaries (ref = Arrear) + benefit-ledger accrual rows.
17.4.3 **Payroll Correction** (LWP reversal): pick a month's submitted slip with LWP > 0; `days_to_reverse` > 0 and Σ across corrections ≤ that slip's LWP days.
17.4.4 Correction breakup per arrear-flagged component: per-day = default_amount ÷ total working days (accruals ÷ payment days) × days_to_reverse → Additional Salaries (no re-proration — §16.4.3); reversed days credit future payment-day counts (§16.2.3).

### 17.5 Retention Bonus & Employee Incentive
17.5.1 **Retention Bonus:** component + amount + payment date (not in the past) → Additional Salary; multiple bonuses for the same key accumulate onto one entry; cancel subtracts.
17.5.2 **Employee Incentive:** component + amount + payroll date; requires an SSA; → Additional Salary.

---

## 18. Income Tax Engine

### 18.1 Income Tax Slab (config)
18.1.1 Effective-dated, disableable, currency-typed. Contains: **progressive slabs** {from_amount, to_amount (0 = open), percent, optional condition expression (age band, gender, component-value…)}, **other charges** rows {description, percent, min/max taxable bounds} (cess/surcharge levied ON tax, compounding in row order), `standard_tax_exemption_amount` (always applied), `allow_tax_exemption` (enables declarations/proofs), `tax_relief_limit` (taxable ≤ limit ⇒ zero tax), and (India) `marginal_relief_limit`.
18.1.2 A slab is chosen per SSA; it must be enabled and effective on/before the payroll-period start.

### 18.2 Slab math
18.2.1 Zero-tax short-circuit at the relief limit; then per passing slab, band tax = (min(taxable, to) − from + 1) × percent% (open bands use taxable − from + 1) — progressive accumulation with the **inclusive +1 band offset**; then marginal relief (India §18.7.3); then other charges (skip rows whose min/max bounds exclude the income; each row computed on the running total).

### 18.3 Annual taxable projection (per slip)
18.3.1 Annual taxable = **prior actuals** (earlier submitted slips + previous-employer opening earnings, counted only when the SSA starts on/after the period start, − pre-tax exempt deductions) + **current cycle** tax-applicable earnings at actual payment days + **future projection** (current full-cycle taxable × remaining cycles − 1; recurring additional salaries projected forward) + **additional/irregular income** + **declared other income** − **exemptions** (§18.5) − standard deduction.
18.3.2 Additional earnings flagged **full-tax-on-date** are excluded from the spread and taxed entirely in their cycle: full-tax delta = tax(with) − tax(without).

### 18.4 Per-cycle TDS spread
18.4.1 `current cycle tax = (total projected tax − tax already deducted) ÷ remaining cycles (incl. current)` + any full-tax delta; floored at 0 (never negative). Opening `tax_deducted_till_date` counts as already-deducted.
18.4.2 The tax component = the structure's slab-variable deduction (auto-discovered from tenant defaults when absent, with an alert). An overwrite Additional Salary on the tax component pins the cycle's tax directly.

### 18.5 Exemptions: Declaration → Proof
18.5.1 Masters: **Exemption Category** (annual cap) → **Sub-category** (cap ≤ parent). Both deactivateable.
18.5.2 **Declaration** (one per employee per payroll period): rows {sub-category, declared amount}; no duplicate sub-categories. Exemption per row = min(declared, sub-cap), then capped per category; + HRA exemption (India §18.7).
18.5.3 **Proof Submission** (one per period): mirrors the declaration with proof type + attachments and actual amounts; same capping. **Mode switch:** through the year tax uses *declared* amounts; when the payroll run/slip sets `deduct_tax_for_unsubmitted_tax_exemption_proof` — forced automatically in the **final cycle** of the period — tax switches to *proof* amounts.
18.5.4 **Employee Other Income** records (per period, multiple allowed) add to annual taxable income.
18.5.5 Exemptions apply only when the slab allows them; the standard deduction applies regardless.

### 18.6 Income-tax breakup (slip display) `[UI]`
18.6.1 Shows: CTC, other income, total earnings, non-taxable earnings, pre-tax deductions, exemptions + standard deduction, **annual taxable amount**, tax deducted till date, current-cycle tax, future tax deductions, total annual tax. `future = total projected − deducted so far`.

### 18.7 India regional rules (Phase 1)
18.7.1 Statutory components seeded: Professional Tax (pre-tax exempt deduction), Provident Fund (taxable deduction) + PF variants, Basic / HRA / Arrear / Leave Encashment earnings; employee fields PAN, PF account, IFSC/MICR.
18.7.2 **HRA exemption** = min of: (a) actual annual HRA per structure; (b) annual rent − 10% of annual basic; (c) 50% (metro) / 40% (non-metro) of annual basic — basic & HRA aggregated across the period's SSAs, frequency-aware. Declared via monthly rent + metro flag on the Declaration; proof flow validates rent period (≥ 15 days, no overlapping proofs) and converts to months rounded to the nearest 0.5.
18.7.3 **Marginal relief:** when relief_limit < taxable < marginal_relief_limit and the excess over the relief limit is smaller than the computed tax, tax = that excess.
18.7.4 India reports: Professional Tax / Provident Fund / Income Tax deduction registers.

---

## 19. Flexible Benefits

19.1 Benefit components (`is_flexible_benefit`) carry a yearly `max_benefit_amount` and a **payout method**: (a) *Accrue & pay out at period end*; (b) *Accrue per cycle, pay only on claim* (+ optional final-cycle payout of unclaimed accruals); (c) *Allow claim for full benefit amount*.
19.2 Entitlement source: the employee's **Benefit Application** for the period (one per employee per period; row amounts > 0, ≤ per-component cap, Σ ≤ the structure's max benefits) — mandatory when the tenant says so, else falling back to the SSA's benefit rows.
19.3 Per-cycle accrual = yearly ÷ total cycles (payment-days-prorated when flagged; integer-rounding honoured), clamped so cumulative accrual never exceeds the yearly amount.
19.4 `[API]` **Benefit Ledger** (append-only: Accrual | Payout rows per employee/component/period, written on slip submit, deleted on cancel) is the source of truth for accrued-vs-paid and claim eligibility.
19.5 **Benefit Claim:** claimable methods (b)/(c); payroll_date not in the past; amount ≤ eligibility ((accrued + current cycle) − paid for (b); yearly − paid for (c)); **one claim per component per calendar month**; pays via Additional Salary in the next slip.
19.6 Unclaimed balances force-paid in the final cycle (method (a), or (b)+flag) become taxable there (§18.3).

---

## 20. Gratuity & Regional Templates

20.1 **Gratuity Rule:** slab table {from_year, to_year (0 = open), fraction_of_applicable_earnings}, `calculate_based_on ∈ {Current Slab, Sum of all previous slabs}`, work-experience method {Round off, Take exact completed years, Manual}, `total_working_days_per_year` (default 365), minimum qualifying years, applicable earning components. A (0,0) universal slab cannot be mixed with banded slabs.
20.2 Work experience = (relieving − joining − LWP/absent days per the payroll basis) ÷ days-per-year, rounded/exact/manual; below the minimum ⇒ not eligible (hard error).
20.3 Amount basis = Σ default amounts of the applicable earning components on the employee's **last submitted salary slip**. Current-Slab: basis × experience × the containing slab's fraction. Progressive: completed bands at (to−from) years × fraction each + remainder in the containing band. Status: **Draft → Unpaid → Paid | Cancelled**; payout via Additional Salary (through payroll) or direct settlement (accounting seam).
20.4 **India template** (seeded): min 5 years, round-off experience, single open slab, fraction **15/26**.
20.5 **UAE templates** (future region model): exact years, min 1 year, 21/30-day-based fractions — limited-contract progressive ([1–5) = 21/30, 5+ = 1.0), unlimited-termination current-slab, unlimited-resignation reduced fractions (1/3, 2/3, full of 21/30).

---

## 21. Employee Self-Service (Mobile/PWA)

21.1 `[UI]` Navigation = 5 tabs: **Home, Attendance, Leaves, Expenses, Salary** + Profile/Notifications/Settings. A logged-in user without an Active employee record sees a dedicated "invalid employee" state.
21.2 **Home:** greeting; **check-in panel** (per §8.3.8, hidden when the tenant disables mobile check-in); quick links (Request Attendance / Shift / Leave, Claim Expense, Request Advance, View Salary Slips); pending-approvals panel for approvers.
21.3 **Attendance tab:** colour-coded month calendar (attendance status + holidays), recent attendance requests, upcoming shifts, shift requests.
21.4 **Leaves tab:** per-type balance gauges (allocated vs remaining); holiday list; leave list with **"My" vs "Team" tabs** (Team = requests awaiting me as approver) and filters (status/type/employee/department/dates). Leave form: half-day toggle reveals the half-day date (auto-set for single-day spans), leave types offered per selected date, live computed leave-days, **balance shown before submitting**, approver pre-picked from resolution chain and required per tenant setting, attachments.
21.5 **Expenses tab:** claim summary tiles (pending/approved/rejected totals), My/Team claim lists, line items with taxes and advance allocations; advance request/list.
21.6 **Salary tab:** slip list; slip detail tabs incl. income-tax breakup (§16.5.4); PDF download.
21.7 Approvals: approve/reject action sheets on team items; when a custom workflow exists, its states/transitions replace the native buttons automatically (§23.6).
21.8 `[API]` The app is served by a compact API: current user/employee info, settings subset (check-in / geolocation / self-approval flags), notification counts + mark-read, attendance calendar events, shift & attendance requests, leave applications with computed balances & approval metadata, leave types valid per date, expense claims + summary + types + advance balances, salary-slip PDFs (base64), a generic form-metadata engine (fields, states, permitted-write fields, workflow definition), and base64 file upload (JPG/PNG/PDF/TXT/Office; images EXIF-normalised; stored private).
21.9 "My vs Team" list logic `[API]`: Team/approval lists = drafts of other employees in an actionable state where I am the resolved approver (or the workflow permits my role to act).

---

## 22. Notifications & Scheduled Jobs

### 22.1 In-app notifications (+ optional push relay)
22.1.1 Model: {to_user, from_user, rich message, read flag, reference record}. Unread count + mark-all-read + tap-through to the record.
22.1.2 **Matrix:** on create of Leave Application / Expense Claim / Shift Request → the resolved approver ("X raised a new … for approval"); on its status → Approved/Rejected → the employee ("Your … has been Approved/Rejected by Y"). Self-actions never notify. (Employee Advance intentionally has no approval notification.)

### 22.2 Email templates (tenant-configurable, seeded at onboarding)
22.2.1 Leave approval + leave status templates (master toggle `send_leave_notification`); interview reminder; interview-feedback reminder; exit questionnaire. Separate sender identities: general HR sender vs hiring sender.

### 22.3 Scheduled-job catalogue (the complete background surface)
| Frequency | Job |
|---|---|
| ~5 min | Interview reminders (send-once per interview, `remind_before` ahead) |
| Hourly | Daily-work-summary prompts (per group hour) |
| Hourly | Check-in sync-watermark advance; **auto-attendance processing** for every enabled shift; shift-schedule roster top-up |
| Daily | Birthday reminders; work-anniversary reminders; work-summary digests; interview-feedback chase; expire ended shift assignments; auto-close expired job openings |
| Daily (long) | Leave-allocation expiry ledger entries; auto leave-encashment drafts (tenant toggle); **earned-leave accrual** |
| Weekly / Monthly | Upcoming-holiday reminder emails (per tenant frequency setting: 7-day look-ahead vs 1st-of-month monthly look-ahead) |

22.3.1 Reminder audiences: birthdays/anniversaries → everyone in the tenant except the celebrants (celebrant-to-celebrant specials when shared); holidays → each Active employee's own upcoming non-weekly-off holidays.

### 22.4 Tenant settings registry (HR)
22.4.1 Employee: naming scheme, retirement age (60), standard working hours.
22.4.2 Reminders: birthday / anniversary / holiday toggles (all default ON) + holiday frequency (Weekly default) + sender.
22.4.3 Leave: approver mandatory (ON), prevent self-approval (OFF), department-calendar visibility (OFF), leave email notifications (ON + 2 templates), backdating restriction (OFF + exempt role), auto-encashment (OFF).
22.4.4 Expense: approver mandatory (ON), prevent self-approval (OFF), unlink-payment-on-advance-cancel (OFF).
22.4.5 Shift & attendance: multiple shift assignments (OFF), mobile check-in (ON), geolocation tracking (OFF).
22.4.6 Exit: questionnaire web form + notification template.
22.4.7 Recruitment: offer-time vacancy check (OFF), interview reminder (ON + template + 15-min lead), feedback reminder (ON + template), hiring sender.
22.4.8 Payroll settings per §15.2.

---

## 23. Cross-Cutting Requirements

23.1 **Audit trail** `[STD]` — immutable log (actor, timestamp, before/after) for: employee status/date changes, all record_state transitions, approver changes, attendance overwrites, allocation/ledger mutations, payroll run lifecycle, settlement events. The leave ledger (§7.5) and benefit ledger (§19.4) are themselves audit-grade append-only stores.
23.2 **Idempotency** `[STD]` — bulk generators (payroll slips, appraisals, allocations, shift generation, onboarding tasks) must be idempotent per (source, target, window): re-running skips existing records. All approval/submit endpoints accept idempotency keys.
23.3 **Partial-success bulk semantics** — every bulk tool runs per-record savepoints, never aborts the batch on one failure, reports success/failure lists live (progress events), and logs failures with reasons.
23.4 **Derived-not-stored balances** — leave balance, benefit accrual balance, advance outstanding, expense outstanding are always computed from their ledgers/transactions.
23.5 **Async thresholds** `[STD]` — background execution beyond: 10 (bulk attendance days), 20 (structure assignment), 30 (payroll create/submit, SSA bulk, appraisals, shift bulk, OT slips), 50 (payroll-cancel slips), 200 (attendance CSV rows), 1000 (check-in processing batch). Long-job timeouts ≥ 50 min.
23.6 **Workflow seam** — no default multi-level workflows ship, but Leave Application / Expense Claim / Shift Request / Attendance Request must expose {state field, allowed transitions per actor} so a configurable workflow engine can replace native statuses without UI rewrites (the self-service app already renders workflow actions generically).
23.7 **Module seams:** accounting (§1.6), lending/loans (slip deduction + FnF row), project timesheets (§15.4.4), asset management (FnF asset rows §6.3.4), push-notification relay (§22.1), inbound email (applicants §10.4.2, daily work summary §13, exit interview intake).
23.8 **Real-time UX** `[API]` — attendance calendars, request lists, and bulk-tool progress update via server push events, not polling.

---

## 24. Tenant Onboarding Seed Data (created for every new tenant)

24.1 **Leave types:** Casual Leave (encashable, carry-forward, max 3 continuous days), Compensatory Off (compensatory), Sick Leave, Privilege Leave, Leave Without Pay (LWP) — all counting holidays inside leaves.
24.2 **Expense claim types:** Calls, Food, Medical, Others, Travel. **Employment types:** the 8 in §2.4.1. **Applicant sources:** Website Listing, Walk In, Employee Referral, Campaign. **Offer terms:** 12 standard terms (DOJ, salary, probation, benefits, hours, ESOPs, department, JD, responsibilities, leaves/year, notice, incentives). **Vehicle service items:** 6 standard.
24.3 **Email templates** per §22.2; default HR settings per §22.4 (interview reminders pre-enabled).
24.4 **Salary components:** a default Basic; India statutory set per §18.7.1; Indian gratuity rule per §20.4.
24.5 `[STD]` A default Leave Period and Payroll Period for the current fiscal year (April–March, India) are created at tenant onboarding. **Confirm fiscal-year convention.**

---

## 25. Future-Phase Notes (design seams, do not build now)

25.1 Multi-legal-entity tenants (§1.2): staffing-plan hierarchy ceilings, inter-company transfer, per-entity payroll.
25.2 Accounting module: replace every §1.6 seam with true GL postings (the reference implementation's posting rules are documented per section for that day).
25.3 Configurable multi-level approval workflows (§23.6); appraisal/goal deadline reminders (none exist in the reference); two-way calendar sync for interviews/trainings.
25.4 Timesheet-based payroll and lending integration (§0.11).
25.5 Reference reports not carried into Phase 1 scope (project profitability, timesheet utilisation) — revisit with the project-management module.

---

## 26. Open Items Requiring Your Sign-Off

1. Tenant = single company; multi-entity deferred (§0.1 / §1.2).
2. Accounting-seam approach for Phase 1 (§0.3 / §1.6) — every payroll/expense settlement is a status-tracked record, not a GL posting.
3. India-first statutory scope; UAE as future template (§0.5).
4. Payroll basis default = Leave (§0.6) and unmarked-attendance default (Present vs Absent) (§15.2.2).
5. Suspended-employee transaction policy (§3.1.1).
6. Attendance Request without approval by default (§8.7.1) — accept or mandate a workflow.
7. Biometric ingestion in Phase 1 (§0.12).
8. Async thresholds & job cadences (§23.5, §22.3).
9. Seeded fiscal-year convention for leave/payroll periods (§24.5).
10. Careers-portal branding/routing model per tenant (§10.9).

---

*End of Clozr HR Master Rule Book v1.0. Derived end-to-end from the Frappe HR v17 reference implementation; adaptations and standards decisions are tagged `[ADAPT]`/`[STD]` and consolidated in §0 and §26. Read together with the Clozr CRM Master Rule Book v1.0 (shared platform: tenancy, users, roles, billing).*
