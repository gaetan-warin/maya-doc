# Weather & Environmental Monitoring
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-WEATHER-001 |
| **Version** | 3.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Weather & Environmental Monitoring** module serves as Maya's data backbone, collecting and distributing hyper-local weather information from on-site stations. This data feeds disease prediction models, irrigation decisions, and operational planning across all other modules.

This document defines the functional requirements for weather data acquisition, display, API integration, and the specific metric codes used throughout the Maya platform.

**Key Business Outcomes:**
- **Accurate Disease Prediction**: On-site sensor data produces more reliable disease models than regional weather services.
- **Optimized Spray Timing**: Real-time wind speed and forecast data enable identification of safe spray windows.
- **Reduced Irrigation Waste**: Precise ET and rainfall data optimize water application.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Weather station data collection and display
- Soil sensor integration
- Forecast display and weather alerts
- API endpoint for external data access
- Historical weather data retrieval

### 1.2 Out of Scope
- Weather station hardware installation
- Weather station firmware updates
- Third-party forecast provider contracts

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **Superintendent** | Daily weather review, operational decisions |
| **Spray Technician** | Spray window identification |
| **Irrigation Manager** | ET and rainfall monitoring |
| **External Systems** | API integration for data consumption |

---

## 2. Weather Station Integration

### 2.1 Supported Hardware

Maya integrates with professional-grade weather stations through standardized data protocols:

| Manufacturer | Models | Integration Method |
|:---|:---|:---|
| Davis Instruments | Vantage Pro2 | API/Data logger |
| Pessl Instruments (Metos) | iMETOS series | Fieldclimate API |
| Campbell Scientific | Various | Custom data logger |
| Virtual Station | N/A | Interpolated from nearby sources |

### 2.2 Site Mapping

Each physical weather station is mapped to one or more Maya sites:
- Single station may serve multiple sites within geographical proximity
- Multiple stations may serve a single large property (different microclimates)
- Virtual station option available when physical hardware is unavailable

---

## 3. Available Metrics

### 3.1 Atmospheric Measurements

| Metric | Code | Unit | Description |
|:---|:---|:---|:---|
| Air Temperature | `air_temperature` | °C | Ambient temperature at sensor height |
| Air Humidity | `air_humidity` | % | Relative humidity |
| Wind Speed | `wind_speed` | m/s | Sustained wind velocity |
| Wind Gust | `wind_gust` | m/s | Peak wind velocity |
| Solar Radiation | `radiation` | W/m² | Incoming solar energy |
| Rainfall | `rainfall` | mm | Precipitation amount |
| Daily Light Integral | `dli` | mol/m²/day | Total photosynthetic photon flux |

### 3.2 Soil Measurements

| Metric | Code | Unit | Description |
|:---|:---|:---|:---|
| Soil Temperature | `soil_temperature` | °C | Temperature at 5cm depth |
| Soil Humidity | `soil_humidity` | % | Volumetric water content |
| Soil Oxygen | `soil_oxygen` | % | Oxygen concentration in root zone |
| Soil Salinity | `soil_salinity` | dS/m | Electrical conductivity |

### 3.3 Derived Metrics

| Metric | Code | Unit | Calculation |
|:---|:---|:---|:---|
| Leaf Wetness | `leaf_wetness` | % | Sensor-based or calculated from dew point |
| ET (Evapotranspiration) | `et` | mm | Penman-Monteith equation |
| ET 24-hour | `et24` | mm/24h | Rolling 24-hour ET total |

---

## 4. API Specification

### 4.1 Endpoint

```
GET https://api2.mayaglobal.io/api/v4/metrics
```

### 4.2 Authentication

| Method | Type |
|:---|:---|
| Authorization | Bearer Token |
| Token Source | Generated in Settings → API Token Setting |

### 4.3 Parameters

| Parameter | Required | Description |
|:---|:---|:---|
| `codes` | Yes | Comma-separated list of metric codes |
| `idsite` | No | UUID of specific site (omit for all sites) |

### 4.4 Example Request

```
GET /api/v4/metrics?codes=air_temperature,air_humidity,wind_speed,et24,leaf_wetness&idsite=abc123
```

### 4.5 Response Format

JSON object with requested metrics, including:
- Current value
- Timestamp of last update
- Site identifier (if multi-site query)
- Unit of measurement

---

## 5. Spray Window Analysis

### 5.1 Definition

A "spray window" is a period when environmental conditions are suitable for chemical application.

### 5.2 Criteria

| Parameter | Threshold | Rationale |
|:---|:---|:---|
| Wind Speed | < 15 km/h | Prevents spray drift |
| Rain Forecast | None for 4-6 hours | Prevents wash-off |
| Temperature | < 30°C | Prevents volatilization |
| Humidity | > 50% | Improves uptake |

### 5.3 System Behavior

When conditions meet all criteria, the system can:
- Display "Spray Window Available" indicator on dashboard
- Include in scheduled alerts/notifications
- Log window start/end times for historical analysis

---

## 6. Weather Alerts

### 6.1 Alert Types

| Alert | Trigger | Severity |
|:---|:---|:---|
| Frost Warning | Ground temperature forecast < 0°C | High |
| Heat Stress | Temperature > 35°C | High |
| High Wind | Wind speed > 25 km/h | Medium |
| Rain Incoming | Precipitation forecast within 2 hours | Low |

### 6.2 Notification Channels

Alerts can be delivered via:
- In-app notification
- Email
- WhatsApp (where configured)

---

## 7. Historical Data

### 7.1 Retention

Weather data is retained indefinitely for trend analysis and regulatory compliance.

### 7.2 Query Capabilities

| Query Type | Description |
|:---|:---|
| Date Range | Retrieve data between two dates |
| Rolling Period | Last 24h, 7d, 30d, 365d |
| Comparative | Same period across multiple years |

### 7.3 Export Formats

- **PDF**: Formatted weather summary reports
- **CSV**: Raw data for external analysis
- **API**: Programmatic access for integrations

---

## 8. Integration Points

| System | Integration Type | Data Flow |
|:---|:---|:---|
| **Agronomy Module** | Critical dependency | Temp, humidity, leaf wetness for disease models |
| **Water Module** | Critical dependency | ET and rainfall for Water Balance |
| **Task Module** | Reference | Spray window recommendations |
| **Alarms Module** | Trigger | Weather-based alert thresholds |
| **External Systems** | API | Third-party applications, data warehouses |

---

## 9. Acceptance Criteria

1. Weather data updates display within 5 minutes of station transmission.
2. All 13 standard metric codes are queryable via API.
3. Spray window analysis correctly evaluates all four environmental criteria.
4. Frost and heat stress alerts trigger at defined thresholds.
5. Historical data exports include all requested metrics with timestamps.
6. API returns 401 error for invalid or expired bearer tokens.

---

**Document End**
