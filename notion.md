# **TO-BE Model (Golf & Football)**

## **1. Introduction**

Maya is used daily in golf courses, football stadiums and multisport venues to plan, execute and analyse operational work.

To remain credible, compliant, and scalable, we must define **one single unified logic** for:

- Tasks
- Nutritional inputs
- Staff
- Machines
- Sites, holes, pitches, and zones
- Planned vs actual hours
- Overtime
- Reporting and analytics
- Stock usage and legal compliance

*This document describes the **TO-BE behaviour,** the future, corrected, consistent model, and finishes with the **AS-IS inconsistencies** currently being fixed.*

# **2. Core Concepts: One Model for All Work**

Across golf and football, all operational work falls into the same structure:

**A task = planned work + actual work, carried out by specific staff, machines, and parameters, across one or multiple locations.**

This includes:

- mowing
- fertilising
- spraying
- topdressing
- bunker raking
- hand-watering
- line marking
- grooming artificial pitches
- aeration
- verticutting
- repairs
- renovations
- irrigation checks
- inspections

And all **nutritional / chemical / amendment applications** (fertilisers, wetting agents, PGRs, herbicides, fungicides, etc.).

All of these should behave the same way in Maya, with additional parameters depending on the operation type.

---

# **3. Planned Duration — What It Means**

### **3.1 One task = one planned duration**

The planned duration expresses the expected time **per staff member**, not the total labour.

Example (golf):

Aeration of **5 greens on Course A**

Planned duration: typically 8am → 11am **( i.e. planned duration = 3h)**

Assigned to: **2 staff**

👉 Planned labour = **3h × 2 = 6 labour hours**

Example (football):

Morning pitch grooming before training

Planned: **7am → 9am (2h)**

Staff: **3 groundskeepers**

👉 Planned labour = **2h × 3 = 6 labour hours**

**UI Consideration (Optional Input Method):** To align with how turf managers naturally plan work (and because we have hit the issue a few times), it would help to allow user a flexible way to   define planned duration. Along defining time per staff, users may specify : 

- the *total estimated labour hours (how our users think day to day…)* for a task and
- the *number of staff assigned*

upon which Maya automatically calculates the planned duration per staff member. This improves clarity, reduces planning errors, and remains fully consistent with mobile tracking, multi-site splitting, and all reporting logic. Users who prefer the traditional start-end method can continue using it seamlessly without friction.

### **3.2 Multi-site / Multi-hole / Multi-pitch planned duration**

A planned duration can cover:

- Multiple **golf holes** (e.g. greens 1–9)
- Multiple **pitches** in a football academy
- Multiple **zones** (fairways + roughs)
- Multiple **sites** (e.g. Stadium A + Stadium B)

The planned duration always means:

> “This is the amount of work per person to complete this operation across all selected areas.”
> 

***If the user expects different durations for different locations, they must create separate tasks.***

→ This model is simple, predictable, and aligned with industry workflows.

---

# **4. Real Labour Hours = Planned × Staff (with example)**

This is the main labour accounting rule.

Example with your requested times:

**Task:** Morning mowing

**Planned:** 8am → 11am = **3 hours**

**Staff:** 4

👉 **Planned labour = 3h × 4 = 12 labour hours**

In practice:

- 1 person mowing greens
- 1 person mowing approaches
- 2 people mowing fairways

Even though they cut different areas, they work one shared block of time.

---

# **5. Individual Mobile Tracking (current behaviour, TO-BE preserved)**

This logic is **already implemented**, correct, and must remain unchanged.

- Each staff member **starts and completes** the task individually on mobile.
- This produces **actual hours per person**, which override all assumptions.
- This is why the schedule page already shows **an individual status icon** next to each staff member. If task completed through web app then it completes all ongoing tasks at the same time. (current implementation and working great!).

Example:

Planned duration = 3h

Actual mobile logs:

- A = 2.5h
- B = 3h
- C = 4h

👉 Real labour = **9.5 hours.**

---

# **6. Multi-Site / Multi-Hole / Multi-Zone Tasks (critical rule)**

A staff member logs **one work session**, not multiple.

Therefore:

> When a task covers multiple locations, the real duration is always split evenly across each location.
> 

Examples:

### **Golf example:**

Topdressing **Greens 1, 2, 3**

Staff: 1

Mobile duration: 3 hours

👉 Per-green distribution: **1h per green**

### **Football example:**

Grooming **Pitch A + Pitch B**

2 staff

Actual mobile: both work 4 hours

👉 Total labour: 8h

👉 Per-pitch breakdown: 4h split → **2h per pitch per staff**

This rule is the only defensible option because:

- Mobile app records *one* work session
- Labour cannot logically be multiplied
- Site-level reporting must remain consistent
- Users who want separate time allocations must create separate tasks

This will be communicated clearly to users.

---

# **7. Planned vs Actual — Across Multiple Locations**

Because the task duration reflects **the overall operation**, not each sub-area, the system can finally calculate:

- planned vs actual duration
- planned vs actual labour
- planned vs actual distribution across areas (via splitting)

Example (golf):

Fertilising Greens 1–18

Planned: 6h

2 staff → planned labour 12h

Actual (mobile):

- A = 5.5h
- B = 6.5h → actual labour = 12h

At a site/green level, splitting allows:

- planned per-green duration
- actual per-green duration
- variance identification
- cost attribution
- legal reporting

Same logic applies in stadium management (multi-pitch operations).

---

# **8. Overtime — TO-BE Logic**

We need to support two forms of overtime calculation:

### **8.1 Overtime due to the task being flagged as OT**

If a task is explicitly marked **“Overtime”**, then the *entire duration* of that task is overtime.

### **8.2 Overtime due to exceeding the staff member’s daily schedule**

Even if the task is **not** marked as overtime:

Example:

Staff scheduled to finish at 4pm.

Task runs from 2pm to 6pm.

👉 Last 2 hours = **overtime**

Even if the task itself is “normal”.

The system must check:

- staff calendar
- individual mobile logs
- task metadata

Both models must co-exist because they represent different operational realities in golf & stadium management.

***UI consideration: user should be able to flag this clearly and easily.*** 

---

# **9. Nutritional Inputs (Fertilisation / Spraying) — TO-BE Normalisation**

Today these live in a **separate table**, which is a historical artefact.

They (task + nutritional input) should be unified.

A nutritional input *is a task* with additional parameters…

BUT both pages should remain separate but should be sync fully.

### **9.1 Standard parameters (always available)**

- site / hole / work zone /  pitch / zone
- staff
- machine
- product(s) *⇒ (to be enhanced in new UI)*
- quantity
- comments
- time and OT consideration (see above)

### **9.2 “Advanced mode” parameters (optional)**

Used in many countries for legal compliance:

- nozzle type
- tractor speed
- RPM
- pressure
- flow rate
- tank size
- application rate
- …

*! Note: the feature of extracting and sharing weather conditions on the nutritional input record is very important.* 

*These parameters are essential for: legal audits, chemical application logs, detecting misuse, ML modelling (coverage, uniformity, drift risk, labour prediction, etc.)*

### **9.3 Future evolution (standardised task templates)**

In the future, like MAYA SPORT, we will introduce predefined task types (not for this version!):

- Mowing (with height of cut, blade type, etc)

⇒ see Maya Sport for more inspiration… ! 

This enables consistent data schemas, benchmarking, increased agronomic intelligence, ML predictions, operational optimisation 

---

# **10. Machines & Staff — Legal/H&S Considerations**

Current UX allows multi-staff + multi-machine selection without linking them.

This is insufficient for:

- spraying compliance (mandatory operator → machine assignment)
- chain of liability
- insurance
- Health & Safety inspections (UK/ USA/ Asia)
- accident investigations

TO-BE model:

- Machines should be assigned **per staff member**
    
    OR
    
- Machines can be **shared** (transporters, utility vehicles)

Examples:

Spraying (high legal sensitivity):

- Staff A → Sprayer 1
- Staff B → Sprayer 2

Bunker repair (simple logistics):

- Staff A + B → Utility Vehicle 1 (shared)

This must be captured at database level even if UI remains simple.

---

# **11. Free-Text Tasks (TO-BE Retained)**

Although structured tasks will grow in importance, **free-text tasks remain and will always remain..**.

These will still act like normal tasks, with:

- planned duration
- staff
- machines
- parameters (if needed)
- actual logs
- labour accounting

---

# **12. Future Check-In / Check-Out System (Not in current epic, future enhancement later down the road)**

We are likely to introduce:

- Check-in (arrival)
- Pause / lunch
- Resume
- Check-out (end of day)

This would allow true timesheeting

This is not included in the current refactor but is fully compatible with the TO-BE model.

---

# **13. Stock Logic for Tasks & Nutritional Inputs**

Reminder: Both **tasks** and **nutritional inputs** may consume items from stock.

### **Rules:**

- **Nutritional inputs** consume stock items of type **“Product”**
- **Tasks** consume stock items of type **“Non-product”** (tools, sand, stakes, paint, fuel, etc.) All stock items appart from MACHINERY PARTS + PRODUCT

---

# 

---