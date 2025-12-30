# Surface Area Calculations Logic

Surface calculation depends on the **type of business** (FOOTBALLPITCH or GOLF) and the context (submitting a spraying or mode).

**Divider:** `divider = 10000` (normalizes to hectares)

---

## Spraying Calculations

### FOOTBALLPITCH Logic

#### When holes are selected:
1. Filter pitches (sub-sites) related to the selected site and holes using `filteredPitches`
2. Identify total number of zones: `siteSelected[0].areas.length`
3. **If areas are selected** (`sprayingAreaIds.length > 0`):
   - Distribute surface of each pitch across zones
   - Multiply by number of selected areas
4. **If no areas selected:**
   - Sum the surface of all filtered pitches directly
5. Divide calculated surface by divider

#### When only sites are selected (no holes):
- Use `calculateFootballPitchSurface` to aggregate surface of all pitches
- Divide total by divider

**`calculateFootballPitchSurface` Logic:**
1. Get all sub-sites (pitches) related to selected sites
2. Sum their surface values

### GOLF Logic

#### When holes are selected:
- API call `getAreaHolesSurface` fetches surface data for selected holes
- Total surface = `totalSurface?.data?.total_surface / divider`

#### When only sites or areas are selected:
- Use `calculateTotalSurface`:
  1. Match selected areas with areas associated with each site
  2. Sum surface of matched areas across all selected sites

---

## Clipping Yield Calculations

### GOLF Logic

| Selection | Calculation Method |
|-----------|-------------------|
| Site only (no hole/zone) | Return `null` |
| Zones only (no holes) | `getZoneSurface` → `calculateGrandTotal` (filter areas, sum totals) |
| Zones + Holes | `getZoneHoleSurface` API call → returns `total_surface` |

### FOOTBALLPITCH Logic

| Selection | Calculation Method |
|-----------|-------------------|
| Site only (has sub-sites) | `getSumOfSubsites` → sum all pitch surfaces |
| Holes only | `getSumOfHoles` → filter + sum matching pitches |
| Holes + Zones | `getSurfaceofHolesByZones` → distribute per zone × selected zones |

---

## Common Functions

| Function | Description |
|----------|-------------|
| `getAreaHolesSurface` | API call for holes/zones surface (Golf uses API, Football uses local calc) |
| `totalSurfacePerSite` | FOOTBALLPITCH: `calculateFootballPitchSurface`; Others: aggregate from areas |
| `formularSurface` | Main dispatcher based on mode (create/edit) and business type |
| `calculateGrandTotal` | Filter areas by site/zone IDs, sum total values |
| `getSumOfSubsites` | Filter sub-sites by parent, sum surfaces |
| `getSumOfHoles` / `getSurfaceofHolesByZones` | Filter pitches by holes, optionally distribute across zones |

---

## Final Value Update Logic

1. Calculate `finalValue` using appropriate business type logic
2. If `finalValue` is 0 or null → revert to original value
3. Otherwise → adjust proportionally, round to 2 decimal places

---

(Archived from Notion on 2025-12-22)
