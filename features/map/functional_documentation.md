# Spatial Mapping & Visualization
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-MAP-001 |
| **Version** | 3.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Spatial Mapping & Visualization** module provides a satellite-based geographic interface for visualizing operational data across golf courses, stadiums, and sports venues. It serves as both a visualization tool and the computational engine for surface area calculations that drive chemical application quantities and cost distribution.

This document defines the functional requirements for map interaction, data layer overlays, and the critical surface area calculation logic that differs between Golf and Football venue types.

**Key Business Outcomes:**
- **Rapid Problem Identification**: Visual hotspot detection reveals turf stress invisible from ground level.
- **Accurate Application Quantities**: Precise surface area calculations prevent over/under-application of chemicals.
- **Cost Attribution**: Geographic cost distribution enables hole-by-hole or pitch-by-pitch budgeting.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Satellite map display with zoom/pan controls
- Data layer overlays (soil moisture, temperature, incidents)
- Surface area calculation for selected zones
- Site/Hole/Zone navigation and filtering
- Historical data playback

### 1.2 Out of Scope
- Real-time GPS tracking (handled by Mobile Module)
- CAD-style course design tools
- Irrigation control commands

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **Superintendent** | Visual inspection, problem area identification |
| **Spray Technician** | Target area selection for applications |
| **Agronomist** | Pattern analysis across seasons |
| **Operations Manager** | Area-based cost analysis |

---

## 2. Map Interface

### 2.1 Base Layer

The system utilizes Google Maps satellite imagery as the base layer, providing high-resolution aerial photography updated periodically.

| Feature | Implementation |
|:---|:---|
| **Provider** | Google Maps API |
| **Default View** | Satellite |
| **Zoom Range** | Property overview to individual green |
| **Navigation** | Standard pan, zoom, center controls |

### 2.2 Auto-Centering

When a user selects a location from the navigation panel, the map automatically pans and zooms to center on that location.

| Selection | Map Behavior |
|:---|:---|
| Site selected | Center on site centroid |
| Hole/Pitch selected | Center on specific location |
| Zone selected | No re-center (zone is conceptual) |

---

## 3. Data Layers

### 3.1 Soil Moisture (VWC)

Displays volumetric water content readings from hand-held probes or embedded sensors.

**Visualization:**
- Colored circles overlaid at measurement coordinates
- Circle color indicates moisture level relative to thresholds

| Color | Condition | Typical Range |
|:---|:---|:---|
| 🔴 Red | Too Dry (Stress Risk) | < 18% VWC |
| 🟠 Orange | Approaching Dry | 18–23% VWC |
| 🟢 Green | Optimal | 23–30% VWC |
| 🔵 Blue | Saturated (Disease Risk) | > 30% VWC |

**Threshold Configuration:**
VWC thresholds are configurable per site/zone to accommodate different soil types and grass species.

### 3.2 Soil Temperature

Displays interpolated soil temperature readings. Primarily used for root zone health assessment and germination timing.

### 3.3 Incidents

Displays geotagged scout reports for pest damage, disease outbreaks, and infrastructure issues.

| Severity | Icon Color | Example Issues |
|:---|:---|:---|
| Low | Yellow | Minor pest activity, cosmetic damage |
| Medium | Orange | Moderate disease pressure, equipment marks |
| High | Red | Active infestation, safety hazards |

---

## 4. Surface Area Calculation Logic

### 4.1 Purpose

Surface area calculations drive:
- Total chemical quantity for applications
- Cost per hectare calculations
- Pro-rata cost distribution to individual assets

### 4.2 Golf Course Logic

| User Selection | Calculation Method |
|:---|:---|
| Site only | Returns NULL (too broad for applications) |
| Zone only (e.g., "Greens") | Sum of all surfaces for that zone type |
| Hole + Zone (e.g., "Hole 1 Green") | API call to `getAreaHolesSurface` |

**API Response:**
Returns specific surface area in square meters, converted to hectares (÷ 10,000).

### 4.3 Football Stadium Logic

| User Selection | Calculation Method |
|:---|:---|
| Site only | Sum of all pitch surfaces via `getSumOfSubsites` |
| Pitch only | Pitch surface value |
| Pitch + Zone | Proportional distribution based on zone ratio |

**Example (Pitch + Zone):**
- Pitch A total: 1.0 ha
- Zone ratios: Penalty boxes = 20%, Midfield = 60%, Goal areas = 20%
- If "Penalty boxes" selected: 1.0 × 0.20 = 0.20 ha

### 4.4 Pro-Rata Cost Distribution

When a nutritional input targets an aggregate area (e.g., "All Greens"), individual asset costs are calculated as:

```
Asset Cost = (Total Application Cost) × (Asset Surface Area ÷ Total Zone Surface Area)
```

This enables accurate hole-by-hole or pitch-by-pitch budget tracking.

---

## 5. Historical Data

### 5.1 Time Filtering

Users can select specific dates or date ranges to view historical data points.

| Filter | Options |
|:---|:---|
| Year | All / Specific year |
| Month/Day | All / Specific date |

### 5.2 Trend Analysis

By comparing map overlays across different dates, users can identify:
- Progression of disease spread
- Seasonal moisture patterns
- Recurring problem areas

---

## 6. Integration Points

| System | Integration Type | Data Flow |
|:---|:---|:---|
| **Weather Module** | Reference | Current conditions displayed as overlay metadata |
| **Agronomy Module** | Transactional | Selected area feeds into application quantity calculation |
| **Reports Module** | Source | Incident locations populate map markers |
| **Mobile Module** | Inbound | Import of probe measurements with GPS coordinates |

---

## 7. Acceptance Criteria

1. Map loads within 3 seconds and displays satellite imagery at venue-appropriate zoom level.
2. VWC data points display correct colors based on configurable thresholds.
3. Golf surface area calculations return NULL when only site is selected.
4. Football zone calculations correctly apply proportional distribution.
5. Historical date filter updates displayed data without page reload.
6. Incident markers display correct severity colors and are clickable for details.

---

**Document End**
