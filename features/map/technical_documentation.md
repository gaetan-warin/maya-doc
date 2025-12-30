# Spatial Mapping & Visualization
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-MAP-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Engineering Team |
| **Classification** | Internal - Technical |

---

## 1. System Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (Vue 3)                          │
├─────────────────────────────────────────────────────────────────┤
│  Views         │  Components          │  Store (Pinia)          │
│  └─ maps.vue   │  ├─ MapsGoogle.vue   │  ├─ maps.js             │
│                │  ├─ ReportMap.vue    │  └─ all.js (getImport)  │
│                │  └─ ProgressbarCycle │                          │
├─────────────────────────────────────────────────────────────────┤
│                    Google Maps JavaScript API                    │
│                      (load-google-maps-api)                      │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│                    apiClient (Axios)                             │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│              Import Data + Reports + Surface Calc                │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x |
| Map Provider | Google Maps JavaScript API | Latest |
| Map Loader | load-google-maps-api | 2.x |
| State Management | Pinia | 2.x |

---

## 2. Component Architecture

### 2.1 View Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `maps.vue` | `web/src/views/maps.vue` | Main controller, filters, data processing |

### 2.2 Map Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `MapsGoogle.vue` | `web/src/components/maps/MapsGoogle.vue` | Google Maps wrapper, markers, circles |
| `ReportMap.vue` | `web/src/components/reports/ReportMap.vue` | Map for report incident selection |
| `ProgressbarCycle.vue` | `web/src/components/maps/ProgressbarCycle.vue` | VWC distribution display |

### 2.3 Google Maps Integration

**Initialization Pattern**:
```javascript
import loadGoogleMapsApi from "load-google-maps-api";

loadGoogleMapsApi({
  key: VUE_APP_GOOGLE_MAP_KEY,
  libraries: ["marker", "places", "drawing", "geometry"]
}).then((googleMaps) => {
  const map = new google.maps.Map(element, options);
});
```

---

## 3. State Management

### 3.1 Map State (Component Local)

**File**: `web/src/views/maps.vue` (data section)

```javascript
// State Structure
{
  loading: true,
  reactiveMap: {},           // Google Maps instance
  listMaps: {
    centerBy: 'sub',         // Center mode
    centerMep: {},           // Center coordinates
    subSiteMap: [],          // Site markers
    ciclesMap: [],           // VWC data circles
    imagesMap: []            // Incident markers
  },
  form: {
    type: 'VWC_Mode',        // Active layer
    sub: {},                 // Selected sub-site
    hole: {},                // Selected hole
    year: 'all',
    monthDay: 'all'
  },
  vwc: {                     // Threshold settings
    v1: 18,
    v2: 23,
    v3: 30
  }
}
```

### 3.2 Maps Store

**File**: `web/src/store/maps.js`

```javascript
// Actions
{
  async retriveSingleSensor(deviceId) {
    // Fetch sensor counter data
    return await api.get(`/device(${deviceId})/counters`);
  }
}
```

---

## 4. Data Models

### 4.1 Import Data Entity

**Table**: `import_data`

| Column | Type | Description |
|:---|:---|:---|
| `id` | BIGINT | Auto-increment ID |
| `idtenant` | CHAR(36) | Tenant reference |
| `Latitude` | DECIMAL(10,8) | GPS latitude |
| `Longitude` | DECIMAL(11,8) | GPS longitude |
| `VWC` | DECIMAL(6,2) | Volumetric water content % |
| `Temp_Soil` | DECIMAL(5,2) | Soil temperature °C |
| `Time` | DATETIME | Measurement timestamp |

### 4.2 Report Entity (Incidents)

**Table**: `reports`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `latitude` | DECIMAL(10,8) | GPS latitude |
| `longitude` | DECIMAL(11,8) | GPS longitude |
| `reportedon` | DATE | Report date |
| `severity` | TINYINT | 1=Low, 2=Medium, 3=High |

### 4.3 Layer Type Configuration

| Type ID | Name | Mode | Data Source |
|:---|:---|:---|:---|
| `VWC_Mode` | VWC | progressbar | `getImport` |
| `VWC soil map` | VWC Soil Map | progressbar | `getImport` |
| `temperature_points` | Temperature | progressbar | `getImport` |
| `all_incidents` | All Incidents | normal | `getReports` |

---

## 5. API Specification

### 5.1 Get Import Data

```
GET /api/v2/getImport
Authorization: Bearer {token}

Response: 200 OK
{
  "records": [
    {
      "Latitude": 48.8584,
      "Longitude": 2.2945,
      "VWC": 22.5,
      "Temp_Soil": 15.3,
      "Time": "2025-12-30T10:00:00Z"
    }
  ]
}
```

### 5.2 Get Reports (Incidents)

```
GET /api/v2/getReports
Authorization: Bearer {token}

Response: 200 OK
{
  "records": [
    {
      "id": "report-uuid",
      "latitude": 48.8590,
      "longitude": 2.2950,
      "reportedon": "2025-12-30",
      "severity": 2,
      "taggable": [
        { "tag": { "label": "Dollar Spot" } }
      ]
    }
  ]
}
```

### 5.3 Get Area Holes Surface

```
GET /api/v2/getAreaHolesSurface
Authorization: Bearer {token}

Query Parameters:
- site_id: UUID
- hole_ids: comma-separated UUIDs
- zone_ids: comma-separated UUIDs

Response: 200 OK
{
  "total_surface": 12500  // square meters
}
```

---

## 6. Surface Area Calculation Logic

### 6.1 Golf Course Logic

```javascript
// When holes are selected
const totalSurface = await api.get('/getAreaHolesSurface', {
  params: { hole_ids, zone_ids }
});
return totalSurface.data.total_surface / 10000; // Convert to hectares

// When only zones selected
const surface = calculateTotalSurface(selectedSites, selectedAreas);
```

### 6.2 Football Pitch Logic

```javascript
// When pitches selected
const pitchSurfaces = filteredPitches.map(p => p.surface);
const totalSurface = pitchSurfaces.reduce((sum, s) => sum + s, 0);

// When zones selected on pitches
const zoneRatio = selectedZones.length / totalZones.length;
const zoneSurface = pitchSurface * zoneRatio;
```

### 6.3 Divider Constant

```javascript
const DIVIDER = 10000; // Converts m² to hectares
```

---

## 7. Visualization Logic

### 7.1 Circle Color Assignment

```javascript
function getCircleColor(value, thresholds) {
  if (value < thresholds.v1) return '#ff3300';      // Red - Danger
  if (value < thresholds.v2) return '#FFA500';      // Orange - Warning
  if (value < thresholds.v3) return '#85de00';      // Green - Success
  return '#0066ff';                                  // Blue - Saturated
}
```

### 7.2 Incident Icon Mapping

```javascript
const severityIcons = {
  1: 'warning_yellow.png',   // Low
  2: 'warning_orange.png',   // Medium
  3: 'warning_red.png'       // High
};
```

---

## 8. Sequence Diagrams

### 8.1 VWC Data Display Flow

```
┌──────┐      ┌───────────┐      ┌─────────┐      ┌────────────┐
│ User │      │ maps.vue  │      │   API   │      │ MapsGoogle │
└──┬───┘      └─────┬─────┘      └────┬────┘      └──────┬─────┘
   │  Select VWC    │                 │                  │
   │───────────────>│                 │                  │
   │                │  getImport      │                  │
   │                │────────────────>│                  │
   │                │   Import Data   │                  │
   │                │<────────────────│                  │
   │                │  Filter by Date │                  │
   │                │  Calculate Color│                  │
   │                │  listmap.cicles │                  │
   │                │─────────────────────────────────>│
   │                │                 │  Render Circles │
   │  View Map      │                 │                  │
   │<───────────────│                 │                  │
```

---

## 9. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Map initial load | < 3 seconds |
| Data overlay render | < 1 second for 500 points |
| Pan/zoom responsiveness | No visible lag |
| Bounds filter calculation | < 100ms |

---

## 10. Dependencies

| Dependency | Type | Purpose |
|:---|:---|:---|
| Google Maps API | External Service | Map rendering, satellite imagery |
| Weather Module | Data Source | Current conditions (optional overlay) |
| Reports Module | Data Source | Incident marker data |
| Import Service | Data Source | VWC/Temperature readings |

---

**Document End**
