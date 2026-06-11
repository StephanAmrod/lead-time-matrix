# Lead Time Matrix Documentation – Issues Report

---

## 1. Structural / File Issues

### 1.1 Orphaned File: `02-configuration-components.md`
- A root-level file `02-configuration-components.md` exists with only 3 lines of content.
- It is **not linked anywhere in README.md**.
- Its content duplicates the opening of `2.1-special-dates-manager.md`.
- **Action needed:** Either remove it or integrate its content properly.

### 1.2 Parent Section Headers Repeated Inside Child Files
The following child files open with their **parent section heading**, creating a duplicate/confusing heading hierarchy:

| File | Incorrect Opening Header |
|------|--------------------------|
| `2.1-special-dates-manager.md` | `# 2. Configuration Components` |
| `3.1-calculation-model-overview.md` | `# 3. Calculations` |
| `4.1-permissions.md` | `# 4. Security and Access Control` |

Each child file should start with its own heading only (e.g. `# 2.1. Special Dates Manager`).

---

## 2. Numbering Issues *(see NUMBERING_REPORT.md for full detail)*

| Section | Current Subsection Numbers | Should Be |
|---------|---------------------------|-----------|
| 3.4 | 1, 8, 9 | 1, 2, 3 |
| 3.6 | 1, 10 | 1, 2 |
| 3.7 | 1, 11 | 1, 2 |
| 3.8 | 1, 12 | 1, 2 |

The same non-sequential numbers appear in README.md anchor links.

---

## 3. Content Inconsistencies

### 3.1 Section 3.1 Pipeline vs Section 3.3 Steps — Mismatch
**File:** `3.1-calculation-model-overview.md` and `3.3-jobcard-lead-time-calculation.md`

The **Consistent Calculation Pipeline** in section 3.1 lists **6 steps**:
1. Determine matrix lead time
2. Apply overrides and adjustments
3. Determine add-on lead times
4. Calculate total (internal) lead time
5. Calculate client lead time
6. Aggregate to order level

But section 3.3 has **7 steps**, including:
- **Step 3: Orders Already Scheduled for Production** — this step does not appear in the pipeline at all.

**Action needed:** Either add this step to the pipeline in 3.1 or clarify that it is a rule, not a pipeline step.

---

### 3.2 Section 3.3 – Due Date Visual Diagram is Incomplete
**File:** `3.3-jobcard-lead-time-calculation.md`

The ASCII diagram for "Jobcard Due Date Calculations" uses a 3-branch split (`├─ ... ├─ ... ┐`) but only labels **2 branches** (Internal due date and Client due date). The third branch is missing its label/content.

---

### 3.3 Section 3.3 Step 3 Feels Out of Place
**File:** `3.3-jobcard-lead-time-calculation.md`

Step 3 ("Orders Already Scheduled for Production") is a **policy rule**, not a calculation step. It describes when *not* to recalculate, which is a different concern from the calculation pipeline.

**Suggestion:** Move this to section 3.7 (Changes to Lead Times) where similar locking/sync behaviour is already described — or clearly label it as a rule, not a step.

---

### 3.4 Section 3.4 – Steps 2–7 Are Missing
**File:** `3.4-order-lead-time-calculation.md`

The section jumps from Step 1 (Determine Order Type) directly to Step 8 (Branded Orders). Steps 2 through 7 are entirely absent with no explanation. This may be the root cause of the numbering issues — if this was originally a global numbering scheme, steps 2–7 may belong to section 3.3.

**Action needed:** Confirm whether steps 2–7 are intentionally omitted, belong elsewhere, or need to be added.

---

### 3.5 Warehouse Permissions Include "Personalisation Rules" — Does Not Apply
**File:** `4.5-warehouse-lead-time-matrix.md`

Three permissions exist for Personalisation Rules:
- Can Add Personalisation Rules
- Can Edit Personalisation Rules
- Can Delete Personalisation Rules

However, **Personalisation is a Production concept**, not a Warehouse concept. Section 2.3 (Warehouse Lead Times – Manage) makes no mention of personalisation rules for warehouse order types.

**Action needed:** Remove personalisation permissions from 4.5, or confirm warehouse lead times also support personalisation.

---

### 3.6 Warehouse View Missing Fields Defined in Manage
**File:** `2.3-lead-time-matrix.md`

The **Warehouse – View** section (Level 3: Quantity Breaks) only shows:
1. Lower quantity limit
2. Upper quantity limit
3. Lead time

But the **Warehouse – Manage** section for Quantity Breaks requires defining:
- Unit (eaches, cases, pallets)
- Production level or production department

These fields are not shown in the view specification.

**Action needed:** Add missing fields to the View specification, or clarify they are manage-only fields.

---

### 3.7 Inconsistent Formatting in Warehouse Quantity Breaks View
**File:** `2.3-lead-time-matrix.md`

The Production Lead Times View (Level 3) uses **bullet points** for quantity break fields.
The Warehouse Lead Times View (Level 3) uses **numbered list items (1. 2. 3.)** for the same fields.

These should be consistent.

---

### 3.8 Section 2.2 – Mixed Terminology (Groups vs Print Codes)
**File:** `2.2-branding-departments-manager.md`

Under **Level 2: Print Codes**, the description reads:
> "Each branding group must be expandable to display invoice and setup codes"

But Level 2 is Print Codes, not Branding Groups. The word "group" is used inconsistently — sometimes referring to print codes, sometimes to branding groups.

**Action needed:** Align terminology throughout. Decide whether Level 2 is called "Print Codes" or "Branding Groups" and use it consistently.

---

### 3.9 Working Day Calendar – Potential Redundancy
**File:** `3.5-working-day-calendar.md`

The rules state:
- Exclude **Weekends**
- Exclude **Public holidays**
- Exclude **All dates in the Special Dates Manager**

Since public holidays are **automatically synced into the Special Dates Manager** (per section 2.1), the second and third bullet points may overlap.

**Action needed:** Clarify whether public holidays are excluded via the Special Dates Manager or independently — to avoid duplicate exclusion logic.

---

### 3.10 Time Unit Handling – No Integration into Jobcard Calculation
**File:** `3.6-time-unit-handling.md` and `3.3-jobcard-lead-time-calculation.md`

Section 3.6 states lead times may be in **Days or Hours**. However:
- Section 3.3 (Jobcard Calculation) only references days throughout.
- There is no specification for how hour-based lead times are handled within the jobcard calculation pipeline.
- It is unclear whether Production lead times can ever be in hours (or only Warehouse).

**Action needed:** Clarify which lead time types (Production / Warehouse) support hours, and how hour-based times integrate into the calculation pipeline.

---

### 3.11 Section 4.3 – No Permission to Add or Delete Branding Departments
**File:** `4.3-branding-departments-manager.md`

Section 2.2 allows managing branding departments (add/edit/delete), but section 4.3 has **no corresponding permissions** for:
- Adding a branding department
- Editing a branding department name
- Deleting a branding department

The only view permission covers the full hierarchy. All manage permissions are scoped only to codes (Invoice, Setup, Additional Charges).

**Action needed:** Add permissions for managing branding departments themselves, or confirm this is intentional (e.g. managed via a different module).

---

### 3.12 Section 4.4 – Adjustments and Overrides Combined Into One Permission
**File:** `4.4-production-lead-time-matrix.md`

> **Can manage Adjustments and Overrides** — Define and update adjustments and overrides

Adjustments and Overrides are **fundamentally different** in behaviour (one adds, the other replaces). Combining them into a single permission means a user who should only manage adjustments also gets access to overrides, which carry much higher impact.

**Action needed:** Consider splitting into separate permissions: `Can manage Adjustments` and `Can manage Overrides`.

---

### 3.13 Section 01-introduction.md – Subsections Not Linked in README
**File:** `README.md` and `01-introduction.md`

The Introduction has three subsections (1. Amtrack, 2. Website, 3. Moyo) but **README.md only links to the top-level introduction file** with no anchor links to subsections — unlike all other sections which include anchor-level links.

**Action needed:** Add anchor links in README.md for the three integration subsections, or confirm this is intentional.

---

## Summary Table

| # | File(s) | Issue Type | Severity |
|---|---------|------------|----------|
| 1.1 | `02-configuration-components.md` | Orphaned file | Medium |
| 1.2 | `2.1`, `3.1`, `4.1` | Duplicate parent headers | Low |
| 2 | `3.4`, `3.6`, `3.7`, `3.8` | Non-sequential numbering | Medium |
| 3.1 | `3.1`, `3.3` | Pipeline step count mismatch | High |
| 3.2 | `3.3` | Incomplete due date diagram | Low |
| 3.3 | `3.3` | Policy rule mixed into calculation steps | Low |
| 3.4 | `3.4` | Steps 2–7 missing with no explanation | High |
| 3.5 | `4.5` | Personalisation permissions on warehouse | High |
| 3.6 | `2.3` | View missing fields defined in Manage | Medium |
| 3.7 | `2.3` | Inconsistent list formatting | Low |
| 3.8 | `2.2` | Mixed terminology (groups vs print codes) | Medium |
| 3.9 | `3.5` | Possible duplicate exclusion logic | Low |
| 3.10 | `3.6`, `3.3` | Hours not integrated into calculation flow | High |
| 3.11 | `4.3` | Missing permissions for branding departments | Medium |
| 3.12 | `4.4` | Adjustments and Overrides combined permission | Medium |
| 3.13 | `README.md`, `01-introduction.md` | Missing anchor links for intro subsections | Low |
