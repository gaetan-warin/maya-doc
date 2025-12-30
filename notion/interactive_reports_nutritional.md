# Interactive Reports - Nutritional Inputs (Known Bugs)

## General Issues
- [ ] Time in X axis should be date
- [ ] Filter to keep only "completed" tasks
- [ ] Find a better solution than random colors

## Nutritional Input Cost Timeline
- [ ] Useless pie chart
- [ ] Units → should show cost

## Nutritional Input Quantity Timeline
- [ ] Units not the same ("kg/ha" vs "g/sqm")
- [ ] Is it correct to sum in kg/ha?
- [ ] Useless pie chart

---

## SECTION: List of Treatments

### Bugs:
- [ ] **Triplicate rows** — duplicate entries appearing
- [ ] Add **Total Cost Column** (reference price per unit × total quantity)
- [ ] Not showing if product is empty NPK
  - Example: 09/04/2024 → Growth Max S not showing
  - Shows if there is at least ONE SUBSTANCE but query does not return product if substance not included

### TOTAL Quantity at Pro Rate
When no filter applied, total quantity should be divided pro rata to the surface area of the zone.

**Example:**
- Greens = 1.2 ha
- Fairways = 16 ha
- Quantity = 12 kg/ha

**Calculation:**
- Total quantity greens = 12 × 1.2 = 14.4 kg
- Total quantity fairways = 12 × 16 = 192 kg

> **Fix:** Use `quantity × surface_area` of the zone in question (`quantity_area_unit` needs correction)

---

## SECTION: Réel vs Planifié (Actual vs Planned)
- When you select a zone → correct
- When no zone selected → seems to multiply by a number
- **Issue:** Since N kg/ha, it should not change

---

## SECTION: Nutritional Input by Composition Over Time
- **Status:** NOT CORRECT → Consider removing?
- NPK values fixed to match correct values
- **Decision needed:** How should NPK (which is unit/ha) behave? Convert to kg based on surface and applied quantity?

---

## SECTION: Product Nutritional Composition Comparison
- **Status:** HIDE FROM UI FOR NOW

---

## SECTION: Product Level and Category Level Cost Analysis

### Bugs:
- [ ] Pricing are **NOT CORRECT**
  - GROWTH S MAX = 100 EUR/L → 412.8 units = should show 412,800 EUR (not correct)
  - ENDURANCE = 46 EUR/L → 602 units = should show 27,692 EUR (not correct)
- [ ] Amounts aren't right
- [ ] Currency should show

---

## NUTRITIONAL INPUT COST TIMELINE
- [ ] Amounts aren't right
- [ ] Currency should show

---

(Archived from Notion on 2025-12-22)
