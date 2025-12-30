# Weather & Environmental Monitoring
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-WEATHER-001 |
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
│  Views           │  Components           │  Store (Pinia)       │
│  └─ weather.vue  │  ├─ WeatherCard.vue   │  └─ weather.js       │
│                  │  ├─ WeatherChart.vue  │                       │
│                  │  └─ SoilSensors.vue   │                       │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│            apiClient + Dashboard API + Public API (V4)           │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│              Weather Station Integration + Calculation           │
├─────────────────────────────────────────────────────────────────┤
│                      External Services                           │
│           Davis | Metos (Fieldclimate) | Virtual Station        │
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│              weather_data | sensor | device                      │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x |
| Chart Library | Chart.js | 4.x |
| State Management | Pinia | 2.x |
| External API | REST (V4) | 4.0 |
| ET Calculation | Penman-Monteith | Standard |

---

## 2. Component Architecture

### 2.1 View Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `weather.vue` | `web/src/views/weather.vue` | Main weather dashboard |

### 2.2 Child Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `WeatherCard.vue` | `web/src/components/weather/WeatherCard.vue` | Individual metric display |
| `WeatherChart.vue` | `web/src/components/weather/WeatherChart.vue` | Historical chart |
| `WeatherForecast.vue` | `web/src/components/weather/WeatherForecast.vue` | 7-day forecast |
| `SoilSensors.vue` | `web/src/components/weather/SoilSensors.vue` | Soil data display |

---

## 3. State Management

### 3.1 Weather Store

**File**: `web/src/store/weather.js`

```javascript
// State Structure
{
  currentConditions: {
    temperature: null,
    humidity: null,
    windSpeed: null,
    windGust: null,
    rainfall: null,
    et24: null,
    leafWetness: null
  },
  soilData: {
    temperature: null,
    humidity: null,
    oxygen: null,
    salinity: null
  },
  forecast: [],
  historicalData: [],
  isLoading: false,
  lastUpdated: null
}
```

### 3.2 Key Actions

| Action | Description | API Call |
|:---|:---|:---|
| `fetchCurrentConditions()` | Load current weather | `GET /dashboardASingleData` |
| `fetchForecast()` | Load 7-day forecast | `GET /weather/forecast` |
| `fetchHistoricalData(range)` | Load historical | `GET /weather/history` |

---

## 4. Data Models

### 4.1 Weather Data Entity

**Table**: `weather_data`

| Column | Type | Description |
|:---|:---|:---|
| `id` | BIGINT | Auto-increment ID |
| `idtenant` | CHAR(36) | Tenant reference |
| `idsite` | CHAR(36) | Site reference |
| `metric_code` | VARCHAR(30) | Metric identifier |
| `value` | DECIMAL(10,4) | Metric value |
| `unit` | VARCHAR(10) | Unit of measure |
| `recorded_at` | TIMESTAMP | Measurement time |

### 4.2 Metric Code Reference

| Code | Name | Unit | Category |
|:---|:---|:---|:---|
| `air_temperature` | Air Temperature | °C | Atmospheric |
| `air_humidity` | Air Humidity | % | Atmospheric |
| `wind_speed` | Wind Speed | m/s | Atmospheric |
| `wind_gust` | Wind Gust | m/s | Atmospheric |
| `radiation` | Solar Radiation | W/m² | Atmospheric |
| `rainfall` | Precipitation | mm | Atmospheric |
| `leaf_wetness` | Leaf Wetness | % | Derived |
| `et` | Evapotranspiration | mm | Derived |
| `et24` | ET 24-Hour | mm/24h | Derived |
| `dli` | Daily Light Integral | mol/m²/day | Derived |
| `soil_temperature` | Soil Temperature | °C | Soil |
| `soil_humidity` | Soil Humidity | % | Soil |
| `soil_oxygen` | Soil Oxygen | % | Soil |
| `soil_salinity` | Soil Salinity | dS/m | Soil |

---

## 5. API Specification

### 5.1 Public API (V4)

**Base URL**: `https://api2.mayaglobal.io/api/v4`

#### Get Metrics
```
GET /metrics?codes=air_temperature,humidity,et24&idsite=uuid
Authorization: Bearer {api_token}

Response: 200 OK
{
  "air_temperature": {
    "value": 22.5,
    "unit": "°C",
    "timestamp": "2025-12-30T10:00:00Z",
    "site_id": "uuid",
    "site_name": "Main Course"
  },
  "air_humidity": {
    "value": 68.0,
    "unit": "%",
    "timestamp": "2025-12-30T10:00:00Z"
  }
}
```

### 5.2 Internal Dashboard API

```
GET /api/v2/dashboardASingleData(code=ETP24)
Authorization: Bearer {session_token}

Response: 200 OK
{
  "ETP24": {
    "daily": {
      "value": 5.8,
      "unit": "mm",
      "timestamp": "2025-12-30T08:00:00Z"
    },
    "weekly": {
      "value": 38.5
    }
  }
}
```

### 5.3 Token Generation

**Endpoint**: Settings → API Token Setting

| Field | Description |
|:---|:---|
| Name | User-defined token name |
| Token | Generated UUID-based string |
| Scopes | Read-only by default |
| Expiry | Optional expiration date |

---

## 6. External Integrations

### 6.1 Weather Station Mapping

```javascript
// Station configuration
{
  id: 'station-uuid',
  provider: 'metos',         // 'davis', 'metos', 'virtual'
  external_id: 'FC12345',    // Provider's station ID
  site_ids: ['site-1', 'site-2'],
  polling_interval: 300,     // seconds
  credentials: {
    api_key: 'encrypted',
    api_secret: 'encrypted'
  }
}
```

### 6.2 Provider Adapters

| Provider | API Type | Data Format |
|:---|:---|:---|
| Davis | REST | JSON |
| Metos (Fieldclimate) | REST | JSON |
| Campbell | Data Logger | CSV |
| Virtual | Interpolation | Calculated |

### 6.3 Virtual Station Logic

When no physical station is available:
```javascript
// Interpolate from nearest stations
const nearbyStations = await findNearestStations(coordinates, limit=3);
const weightedValue = calculateInverseDistanceWeighting(nearbyStations, metric);
```

---

## 7. Calculation Logic

### 7.1 Evapotranspiration (Penman-Monteith)

```javascript
// Simplified approximation
// Full implementation uses FAO-56 standard
const ET0 = (0.408 * delta * (Rn - G) + gamma * (900 / (T + 273)) * u2 * (es - ea)) 
          / (delta + gamma * (1 + 0.34 * u2));
```

**Input Parameters**:
| Symbol | Parameter | Source |
|:---|:---|:---|
| Rn | Net radiation | Solar sensor |
| T | Air temperature | Weather station |
| u2 | Wind speed at 2m | Weather station |
| es | Saturation vapor pressure | Calculated |
| ea | Actual vapor pressure | Humidity sensor |

### 7.2 Daily Light Integral (DLI)

```javascript
// Convert instantaneous radiation to daily total
DLI = (radiation_avg * daylight_hours * 3600) / 1000000;
// Result in mol/m²/day
```

---

## 8. Sequence Diagrams

### 8.1 Current Conditions Fetch

```
┌──────┐      ┌─────────────┐      ┌─────────┐      ┌───────────┐
│ User │      │ weather.vue │      │  Store  │      │    API    │
└──┬───┘      └──────┬──────┘      └────┬────┘      └─────┬─────┘
   │  Load Page      │                  │                  │
   │────────────────>│                  │                  │
   │                 │  fetchConditions │                  │
   │                 │─────────────────>│                  │
   │                 │                  │  GET (parallel)  │
   │                 │                  │─────────────────>│
   │                 │                  │  Merge Results   │
   │                 │  Update Cards    │                  │
   │                 │<─────────────────│                  │
   │  Display Data   │                  │                  │
   │<────────────────│                  │                  │
```

### 8.2 External Data Polling

```
┌────────────┐      ┌─────────────┐      ┌────────────────┐
│ Scheduler  │      │   Backend   │      │ Weather Station│
└─────┬──────┘      └──────┬──────┘      └───────┬────────┘
      │  Cron Trigger      │                     │
      │───────────────────>│                     │
      │                    │  Request Data       │
      │                    │────────────────────>│
      │                    │  JSON Response      │
      │                    │<────────────────────│
      │                    │  Transform + Store  │
      │                    │                     │
      │  Complete          │                     │
      │<───────────────────│                     │
```

---

## 9. Alert System

### 9.1 Threshold Configuration

```javascript
const alertThresholds = {
  frost_warning: {
    metric: 'soil_temperature',
    operator: '<',
    value: 2,
    severity: 'high'
  },
  heat_stress: {
    metric: 'air_temperature',
    operator: '>',
    value: 35,
    severity: 'high'
  },
  high_wind: {
    metric: 'wind_speed',
    operator: '>',
    value: 25,
    severity: 'medium'
  }
};
```

### 9.2 Notification Channels

| Channel | Implementation |
|:---|:---|
| In-App | WebSocket push |
| Email | SMTP service |
| WhatsApp | Twilio API |

---

## 10. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Current conditions load | < 2 seconds |
| Historical chart render | < 3 seconds (365 days) |
| API response time | < 500ms |
| Data freshness | < 5 minutes lag |

---

## 11. Dependencies

| Dependency | Type | Purpose |
|:---|:---|:---|
| Weather Stations | External | Data source |
| Agronomy Module | Consumer | Disease model inputs |
| Water Module | Consumer | ET for Water Balance |
| Alarms Module | Consumer | Threshold triggers |

---

**Document End**
