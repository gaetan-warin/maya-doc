Unified documentation for tenant-level water configuration, water sources, readings lifecycle, and dashboard metrics.

---

### Base Path

```
{baseUrl}/{apiVersion}
```

Unless otherwise specified (e.g. explicit `/api/v2/` in some endpoints), all paths are relative to this base.

### Standard Response Wrapper

All successful and error responses (except low-level framework errors) use:

```json
{
    "success": true,
    "message": null,
    "data": {
        /* payload */
    }
}
```

On errors:

```json
{
    "success": false,
    "message": "Validation failed",
    "data": null
}
```

### Shared Enums

| Field | Allowed Values |
| --- | --- |
| source_type | inflow, outflow |
| measurement_type | daily_consumption, meter_reading |
| meter_unit | cubic_meters |
| irrigation_frequency | daily, every_other_day, once_per_week, specific_weekdays |
| weekday | mon, tue, wed, thu, fri, sat, sun |

---

---

## ⚙️ 1. Get Default Water Settings

Retrieve system-wide (global) defaults for tenant-level and source-level configuration.

GET `/{apiVersion}/settings/water/defaults`

Response:

```json
{
    "success": true,
    "message": "Default water settings retrieved successfully",
    "data": {
        "tenant_defaults": {
            "et_correction_factor": 1,
            "mobile_notifications_enabled": true,
            "preferred_reminder_time": "15:00:00"
        },
        "source_defaults": {
            "annual_water_allowance": "0",
            "irrigation_period_from": "01-01",
            "irrigation_period_to": "12-31",
            "irrigation_frequency": "once_per_week",
            "irrigation_weekdays": ["wed"]
        }
    }
}
```

Field Notes:

- `annual_water_allowance` may be sent by clients as numeric; system can return string.

## 🧩 2–3. Tenant Water Settings

### 2. Get Tenant Settings

GET `/{apiVersion}/settings/tenant/{tenantId}`

Example:

```json
{
    "success": true,
    "message": null,
    "data": {
        "et_correction_factor": "0.870",
        "mobile_notifications_enabled": false,
        "preferred_reminder_time": "16:00:00",
        "default": false
    }
}
```

### 3. Update Tenant Settings

PUT `/{apiVersion}/settings/tenant/{tenantId}`

Request:

```json
{
    "et_correction_factor": 0.87,
    "mobile_notifications_enabled": false,
    "preferred_reminder_time": "16:00"
}
```

Response:

```json
{
    "success": true,
    "message": "Tenant water settings updated",
    "data": {
        "idtenant": "a8b94cd9-2421-46ea-bfd2-71da150ad027",
        "et_correction_factor": 0.87,
        "preferred_reminder_time": "16:00",
        "mobile_notifications_enabled": false
    }
}
```

## 💦 4–6. Water Sources

### 4. List Water Sources

GET `/{apiVersion}/settings/water-sources?tenant_id={tenantId}`

Response (truncated):

```json
{
    "success": true,
    "message": "Water sources retrieved successfully",
    "data": [
        {
            "id": 327,
            "name": "1111",
            "source_type": "outflow",
            "measurement_type": "daily_consumption",
            "last_meter_reading": "111.000000000000",
            "last_reading_date": "2025-10-01",
            "settings": {
                "annual_water_allowance": "100.000000000000",
                "irrigation_frequency": "once_per_week",
                "irrigation_weekdays": ["wed"]
            }
        }
    ]
}
```

### 5. Create Water Source

POST `/{apiVersion}/settings/water-sources`

Request:

```json
{
    "name": "New Source",
    "source_type": "outflow",
    "measurement_type": "meter_reading",
    "last_meter_reading": 0,
    "last_reading_date": "2025-08-12",
    "meter_unit": "cubic_meters",
    "default_associations": [
        {
            "site_id": "33a179b0-ca3b-4e38-b085-7880fdcde571",
            "tags": ["b9f68c65-e9d9-489a-8ed8-057a91241b5f"]
        }
    ],
    "settings": {
        "annual_water_allowance": 125000.5,
        "irrigation_period_from": "03-15",
        "irrigation_period_to": "10-30",
        "irrigation_frequency": "every_other_day",
        "irrigation_start_date": "2025-08-01"
    }
}

```

Response:

```json
{
    "success": true,
    "message": "Water source created successfully",
    "data": {
        "id": 345,
        "name": "New Source",
        "source_type": "outflow",
        "measurement_type": "meter_reading",
        "last_meter_reading": "0.000000000000",
        "last_reading_date": "2025-08-12T00:00:00.000000Z",
        "status": 1,
        "is_legacy": false
    }
}

```

### 6. Update Water Source

PUT `/{apiVersion}/settings/water-sources/{id}`

Request:

```json
{
    "name": "Updated Source",
    "measurement_type": "meter_reading",
    "last_meter_reading": 8000,
    "settings": {
        "annual_water_allowance": 125000.6,
        "irrigation_period_from": "03-15",
        "irrigation_period_to": "10-30",
        "irrigation_frequency": "every_other_day",
        "irrigation_start_date": "2025-08-01"
    }
}

```

## 💧 7–10. Water Readings

### 7. List Readings (Paginated)

GET `/{apiVersion}/water/readings`

Query Params:

- `tenant_id` (UUID, required)
- `current_page` (int, default 1)
- `per_page` (int, 1–100)
- `sort_by` (reading_date | name | reading_value)
- `sort_order` (asc | desc)
- `filters[start_date]`, `filters[end_date]` (YYYY-MM-DD)
- `filters[measurement_type]` (meter_reading | daily_consumption)

### 8. Create Reading

POST `/api/v2/water/readings`

Request:

```json
{
    "water_source_id": 42,
    "reading_value": 1575.3,
    "reading_date": "2025-10-18",
    "reading_time": "06:30:00",
    "notes": "Manual entry after maintenance"
}
```

Response:

```json
{
    "success": true,
    "message": "Water reading created successfully.",
    "data": {
        "id": 781,
        "water_source_id": 42,
        "reading_value": "1575.3000",
        "consumption_value": "125.3000",
        "measurement_type": "meter_reading",
        "reading_date": "2025-10-18",
        "reading_time": "06:30:00"
    }
}
```

### 9. Update Reading

PUT `/api/v2/water/readings/{id}`

Request:

```json
{
    "reading_value": 1582.7,
    "notes": "Adjusted after QA check"
}

```

Response:

```json
{
    "success": true,
    "message": "Water reading updated successfully.",
    "data": {
        "id": 781,
        "reading_value": "1582.7000",
        "consumption_value": "132.7000"
    }
}

```

### 10. Delete Reading

DELETE `/api/v2/water/readings/{id}`

Response:

```json
{
    "success": true,
    "message": "Water reading deleted successfully.",
    "data": null
}

```

## 📈 11. Water Dashboard

### Endpoint

- Method: `GET`
- URL: `/api/v2/water/dashboard`
- Query: `tenant_id={tenant_id}`

Retrieves aggregated telemetry, usage, and budget metrics for a tenant's water dashboard. Successful responses use the standard API envelope: `success`, `message`, and `data`.

### Query Parameters

| Name | Required | Type | Description |
| --- | --- | --- | --- |
| `tenant_id` | Yes | UUID (string) | Tenant identifier whose dashboard metrics should be returned. |

### Response Shape

`data` contains the metrics captured by `WaterDashboardService::getDashboardData()`.

### Example Response

```json
{
  "success": true,
  "message": "Water dashboard data retrieved successfully",
  "data": {
    "generated_at": "2025-11-24T07:15:32Z",
    "water_usage": {
      "last_24h": {
        "total": 2495.0,
        "percent_of_month_forecast": 0
      },
      "month_to_date": {
        "total": 10852.0,
        "trend_vs_last_month": -1712.0
      },
      "month_forecast": {
        "value": 0,
        "status": "coming_soon"
      },
      "fallback_last_reading": null
    },
    "watering_activity": {
      "days_watered_month": 12,
      "days_watered_year": 110,
      "avg_water_per_day_month": 904.3,
      "avg_water_per_day_last_month": 932.1,
      "trend_vs_last_month": -27.8
    },
    "annual_budget": {
      "year_to_date": 45498.0,
      "allowance": 75000.0,
      "progress_percent": 60.7,
      "last_year_same_period": 47210.0,
      "difference_vs_last_year": -1712.0
    },
    "et": {
      "last_24h": 5.2,
      "total_since_last_irrigation": {
        "value": 18.7,
        "days_since_irrigation": 4
      },
      "tomorrow_forecast": 4.9,
      "adjusted": {
        "value": 4.42,
        "reference": 5.2,
        "correction_factor": 0.85
      }
    },
    "rainfall": {
      "last_24h": 0.0,
      "total_since_last_irrigation": {
        "value": 2.3,
        "days_since_irrigation": 4
      },
      "tomorrow_forecast": {
        "value": 5.0,
        "probability": 50
      }
    },
    "site_conditions": {
      "air_temperature": {
        "now": 15.0,
        "min": 14.0,
        "max": 18.0
      },
      "soil_temperature": {
        "now": 16.0,
        "min": 15.5,
        "max": 19.2
      },
      "soil_moisture": {
        "now": 30.0,
        "min": 25.0,
        "max": 35.0
      },
      "last_updated": "2025-11-24T07:15:32Z"
    }
  }
}
```

### Field Reference

**Core Metadata**

- `generated_at`: ISO 8601 timestamp emitted when the dashboard payload is assembled.

**Water Usage**

- `water_usage.last_24h`: Object with `total` (float) and `percent_of_month_forecast` (float, currently `0`). Set to `null` when the 24-hour consumption resolves to `0.0`, in which case the fallback block is surfaced.
- `water_usage.month_to_date.total`: Cumulative consumption for the current month. `trend_vs_last_month` captures the raw delta between the current month total and the previous month total (current minus previous).
- `water_usage.month_forecast`: Static placeholder `{ "value": 0, "status": "coming_soon" }` until forecasting is implemented.
- `water_usage.fallback_last_reading`: Populated only when `last_24h` is `null` and a recent reading exists. Provides `value` (float), `days_ago` (int or `null` when the reading date is unavailable), `timestamp` (ISO 8601 or `null`), and a constant `reason` of `no_recent_reading`.

**Watering Activity**

- `watering_activity.days_watered_month`: Total irrigated days in the current month.
- `watering_activity.days_watered_year`: Total irrigated days in the current year.
- `watering_activity.avg_water_per_day_month`: Average consumption per irrigated day for the current month.
- `watering_activity.avg_water_per_day_last_month`: Same metric for the previous month.
- `watering_activity.trend_vs_last_month`: Difference between the current and previous month averages (current minus previous).

**Annual Budget**

- `annual_budget.year_to_date`: Year-to-date consumption.
- `annual_budget.allowance`: Annual allowance derived from tenant water sources.
- `annual_budget.progress_percent`: Percentage of allowance used, rounded to one decimal. Defaults to `0.0` when allowance is zero.
- `annual_budget.last_year_same_period`: Consumption for the same period in the prior year.
- `annual_budget.difference_vs_last_year`: Difference between current year-to-date consumption and the same prior-year period.

**Evapotranspiration (ET)**

- `et.last_24h`: Average evapotranspiration across the last 24 hours calculated from hourly ET metrics. Returns `null` when the data set is empty.
- `et.total_since_last_irrigation.value`: Aggregated ET values from the last irrigation event forward, or `null` when the event date is unknown.
- `et.total_since_last_irrigation.days_since_irrigation`: Elapsed days since the last irrigation event, or `null` if not determinable.
- `et.tomorrow_forecast`: Next-day ET forecast pulled from OpenMeteo. `null` when tenant coordinates or provider data are unavailable.
- `et.adjusted`: Object with `value` (corrected ET), `reference` (raw ET), and `correction_factor` (tenant setting or configurable default). `value` and `reference` become `null` when `et.last_24h` is `null`, while `correction_factor` always resolves to a float.

**Rainfall**

- `rainfall.last_24h`: Aggregated rainfall for the previous 24 hours from hourly rainfall metrics, or `null` when no readings exist.
- `rainfall.total_since_last_irrigation.value`: Rainfall accumulation since the last irrigation event; `null` when the event date is unknown.
- `rainfall.total_since_last_irrigation.days_since_irrigation`: Days since the last irrigation event, or `null` when unavailable.
- `rainfall.tomorrow_forecast`: Object with `value` (float or `null`) and `probability` (integer percentage or `null`) provided by OpenWeather. Both fields default to `null` if the provider call fails or coordinates are missing.

**Site Conditions**

- `site_conditions.air_temperature`: Summary with `now`, `min`, and `max` averages rounded to two decimals; each value becomes `null` when telemetry is absent.
- `site_conditions.soil_temperature`: Follows the same structure and nullability as air temperature.
- `site_conditions.soil_moisture`: Follows the same structure and nullability as the temperature metrics.
- `site_conditions.last_updated`: Timestamp of the aggregation process (never `null`).

### Null Handling

- Missing hourly ET or rainfall metrics yield `null` for `et.last_24h`, `rainfall.last_24h`, and their `total_since_last_irrigation.value` counterparts.
- Absence of irrigation history returns `null` for `days_since_irrigation` and the associated aggregated values.
- A 24-hour consumption total of `0.0` suppresses `water_usage.last_24h` and attempts to populate `fallback_last_reading` with the latest known reading.
- `fallback_last_reading` remains `null` when no readings exist or when the latest reading lacks usable metadata.
- Unset tenant coordinates or upstream forecast errors produce `null` for `et.tomorrow_forecast` and for both fields inside `rainfall.tomorrow_forecast`.

## Dashboard Fallback Variant

When the last 24-hour consumption resolves to `0.0`, the API returns a fallback payload similar to the following:

```json
{
  "water_usage": {
    "last_24h": null,
    "month_to_date": {
      "total": 10852.0,
      "trend_vs_last_month": -1712.0
    },
    "month_forecast": {
      "value": 0,
      "status": "coming_soon"
    },
    "fallback_last_reading": {
      "value": 2410.0,
      "days_ago": 4,
      "timestamp": "2025-10-16T08:00:00Z",
      "reason": "no_recent_reading"
    }
  }
}

```

## Null Scenario Example

The following response illustrates the extreme case where every optional field resolves to `null` while required values default to `0` or standard placeholders:

```json
{
  "success": true,
  "message": "Water dashboard data retrieved successfully",
  "data": {
    "generated_at": "2025-11-24T09:00:00Z",
    "water_usage": {
      "last_24h": null,
      "month_to_date": {
        "total": 0.0,
        "trend_vs_last_month": 0.0
      },
      "month_forecast": {
        "value": 0,
        "status": "coming_soon"
      },
      "fallback_last_reading": null
    },
    "watering_activity": {
      "days_watered_month": 0,
      "days_watered_year": 0,
      "avg_water_per_day_month": 0.0,
      "avg_water_per_day_last_month": 0.0,
      "trend_vs_last_month": 0.0
    },
    "annual_budget": {
      "year_to_date": 0.0,
      "allowance": 0.0,
      "progress_percent": 0.0,
      "last_year_same_period": 0.0,
      "difference_vs_last_year": 0.0
    },
    "et": {
      "last_24h": null,
      "total_since_last_irrigation": {
        "value": null,
        "days_since_irrigation": null
      },
      "tomorrow_forecast": null,
      "adjusted": {
        "value": null,
        "reference": null,
        "correction_factor": 1.0
      }
    },
    "rainfall": {
      "last_24h": null,
      "total_since_last_irrigation": {
        "value": null,
        "days_since_irrigation": null
      },
      "tomorrow_forecast": {
        "value": null,
        "probability": null
      }
    },
    "site_conditions": {
      "air_temperature": {
        "now": null,
        "min": null,
        "max": null
      },
      "soil_temperature": {
        "now": null,
        "min": null,
        "max": null
      },
      "soil_moisture": {
        "now": null,
        "min": null,
        "max": null
      },
      "last_updated": "2025-11-24T09:00:00Z"
    }
  }
}

```

## Error Responses

- `400 Bad Request` – The required `tenant_id` query parameter is missing or invalid.
- `404 Not Found` – The tenant exists but has no associated water dashboard configuration.
- `500 Internal Server Error` – Unexpected failure while fetching dashboard aggregates; the controller logs the exception and returns `success: false` with the message "Failed to retrieve water dashboard data".

## 📊 12. Water Usage

`GET {{baseUrl}}/{{apiVersion}}/water/usage`

- **Purpose** Return usage readings aggregated by source for a tenant, with optional filtering and cumulative totals.
- **Query Parameters**
    - `tenant_id` (string, required) – Tenant identifier whose usage to load.
    - `filters[start_date]` (string, optional) – ISO-8601 date/time; inclusive range start.
    - `filters[end_date]` (string, optional) – ISO-8601 date/time; inclusive range end; must be ≥ start.
    - `filters[source_ids][]` (array<string>, optional) – One or more water source IDs to include; omit for all sources.
    - `filters[source_type]` (string, optional) – Filter by `inflow` or `outflow`.
    - `filters[granularity]` (string, optional) – Aggregation window; accepts `daily` (default) or `hourly`. `daily` returns YYYY-MM-DD dates, `hourly` returns full timestamps.
- **Sample Request**

    ```bash
    GET {{baseUrl}}/{{apiVersion}}/water/usage?tenant_id=abc-123&filters[start_date]=2024-01-01&filters[end_date]=2024-01-31&filters[source_ids][]=1&filters[source_type]=inflow&filters[granularity]=daily
    ```

- **Success Response (200)**

    ```json
    {
      "success": true,
      "message": "Water usage retrieved successfully",
      "data": [
        {
          "id": "1",
          "name": "Pump 1",
          "readings": [
            {
              "timestamp": "2024-01-01",
              "consumption_value": 12500,
              "cumulative_value": 12500
            },
            {
              "timestamp": "2024-01-02",
              "consumption_value": 13200,
              "cumulative_value": 25700
            },
            {
              "timestamp": "2024-01-03",
              "consumption_value": 11800,
              "cumulative_value": 37500
            }
          ]
        }
      ]
    }
    ```

- **Notes** Values represent consumption or production depending on `source_type`; cumulative values reset per source at the start of the filtered range.

## 📆 13. Water Budget

- Endpoint: `GET /api/water/budget`
    - Purpose: return monthly consumption and cumulative totals per water source, grouped by year for a tenant.
- Query parameters:
    - `tenant_id` (required, uuid/int): tenant that owns the meters.
    - `filters[years][]` (optional, array<int>): restrict response to specific calendar years; defaults to the last 3 full years if omitted.
    - `filters[source_ids][]` (optional, array<int>): restrict response to specific water sources; defaults to all tenant sources.
- Behavior:
    - Response envelope: `success` (bool), `message` (string), `data` (array).
    - `data` is an array of objects, each keyed by a year string (e.g. `{ "2025": [...] }`).
    - Source arrays ordered by name; readings ordered by `timestamp` ascending; cumulative reset on Jan 1.
    - `404` if tenant not found; `422` when validation fails.

**Response Examples**

- Success (`200 OK`)

```json
{
  "success": true,
  "message": "Water usage retrieved successfully",
  "data": [
    {
      "2025": [
        {
          "id": "1",
          "name": "Pump 1",
          "readings": [
            {
              "timestamp": "2025-01",
              "consumption_value": 12500,
              "cumulative_value": 12500
            },
            {
              "timestamp": "2025-02",
              "consumption_value": 13200,
              "cumulative_value": 25700
            },
            {
              "timestamp": "2025-03",
              "consumption_value": 11800,
              "cumulative_value": 37500
            }
          ]
        }
      ]
    },
    {
      "2024": [
        {
          "id": "2",
          "name": "Reservoir North",
          "readings": [
            {
              "timestamp": "2024-11",
              "consumption_value": 9800,
              "cumulative_value": 9800
            }
          ]
        }
      ]
    }
  ]
}
```

- Error (`404 Not Found`)

    ```json
    {"success": false, "message": "Tenant not found", "data": []}
    ```

- Error (`422 Unprocessable Entity`)

    ```json
    {"success": false,"message": "Validation failed","data": [],"errors": {"tenant_id": ["The tenant_id field is required."]}}
    ```


## 🗓️ 14. Days Watered

### Status Definitions

- `irrigated` – Irrigation completed and recorded for the period.
- `confirmed` – Irrigation intent acknowledged (for example by an operator) but not yet executed.
- `denied` – Irrigation request rejected or canceled before execution.
- `planned` – Irrigation scheduled but pending confirmation or execution.
- `unknown` – Status not provided by the source system.

### GET `/api/v2/water/calendar`

### Overview

Returns irrigation calendar data for the month that begins on `start_date`. Results are limited to past and current dates; future dates are excluded. When `source_ids[]` is provided, only the specified outflows are returned. Otherwise, all tenant outflows are included.

### Query Parameters

- `tenant_id` (string, required, `UUID`) – Tenant identifier.
- `filters[start_date]` (string, required, `YYYY-MM-DD`) – First day of the month to fetch.
- `filters[source_ids][]` (integer, optional) – Filter to a specific water source.

### Examples

### Single water source

Sample request:

```bash
curl "{{baseUrl}}/api/v2/water/calendar \
  ?tenant_id=8b66cbe8-1234-4d7c-9f3b-2f7b5d6d9abc \
  &filters[start_date]=2025-04-01 \
  &filters[source_ids][]=42"
```

Sample response:

```json
{
  "success": true,
  "message": "Irrigation calendar retrieved successfully",
  "irrigated_days_count": 3
  "data": [
    {
      "timestamp": "2025-04-01",
      "consumption_value": 12500,
      "sources": [
        {
          "id": 42,
          "name": "North Field Pivot",
          "status": "irrigated"
        }
      ]
    },
    {
      "timestamp": "2025-04-02",
      "consumption_value": 0,
      "sources": [
        {
          "id": 42,
          "name": "North Field Pivot",
          "status": "planned"
        }
      ]
    },
    {
      "timestamp": "2025-04-03",
      "consumption_value": 0,
      "sources": [
        {
          "id": 42,
          "name": "North Field Pivot",
          "status": "confirmed"
        }
      ]
    }
  ]
}
```

### All outflows

Sample request:

```bash
curl "{{baseUrl}}/api/v2/water/calendar \
  ?tenant_id=8b66cbe8-1234-4d7c-9f3b-2f7b5d6d9abc \
  &filters[start_date]=2025-04-01"
```

Sample response:

```json
{
  "success": true,
  "message": "Irrigation calendar retrieved successfully",
  "irrigated_days_count": 3
  "data": [
    {
      "timestamp": "2025-04-01",
      "consumption_value": 12500,
      "sources": [
        {
          "id": 42,
          "name": "North Field Pivot",
          "status": "irrigated"
        },
        {
          "id": 84,
          "name": "South Orchard Drip",
          "status": "planned"
        }
      ]
    },
    {
      "timestamp": "2025-04-02",
      "consumption_value": 0,
      "sources": [
        {
          "id": 42,
          "name": "North Field Pivot",
          "status": "denied"
        },
        {
          "id": 84,
          "name": "South Orchard Drip",
          "status": "unknown"
        }
      ]
    }
  ]
}
```

### Error Responses

- Error (`404 Not Found`)

    ```json
    { "success": false, "message": "Tenant not found", "data": [] }
    ```

- Error (`422 Unprocessable Entity`)

    ```json
    {
      "success": false,
      "message": "Validation failed",
      "data": [],
      "errors": { "tenant_id": ["The tenant_id field is required."] }
    }
    ```


## 🌧️ 15. Rainfall

**Endpoint**: `GET /api/water/rainfall`

**Description**: Returns daily rainfall metrics for the requested tenant and date range. Values are aggregated by day and include cumulative totals for the period.

**Authentication**: Requires a valid application bearer token (see global authentication guide).

### Query Parameters

- `tenant_id` (uuid, required): Tenant identifier. Must reference an existing tenant record.
- `filters[start_date]` (date, required): Start of the reporting window in `YYYY-MM-DD` format.
- `filters[end_date]` (date, required): End of the reporting window in `YYYY-MM-DD` format. Must be on or after `filters[start_date]`.

### Successful Response

- HTTP 200
- Body shape:
    - `success` (boolean): Always `true` on success.
    - `message` (string): Human readable confirmation.
    - `data` (array): Ordered list with one entry per day in the requested range.
        - `timestamp` (string): Date in `YYYY-MM-DD`.
        - `rainfall_value` (number): Daily rainfall total (millimeters).
        - `cumulative_value` (number): Sum of `rainfall_value` from the beginning of the range to `timestamp`.

```bash
curl -X GET "https://{host}/api/water/rainfall" \\
  -H "Accept: application/json" \\
  -H "Authorization: Bearer <token>" \\
  --get \\
  --data-urlencode "tenant_id=11111111-2222-3333-4444-555555555555" \\
  --data-urlencode "filters[start_date]=2024-06-01" \\
  --data-urlencode "filters[end_date]=2024-06-15"
```

```json
{
  "success": true,
  "message": "Rainfall data retrieved successfully",
  "data": [
    {
      "timestamp": "2024-06-01",
      "value": 2.1,
      "cumulative_value": 2.1
    },
    {
      "timestamp": "2024-06-02",
      "value": 0.0,
      "cumulative_value": 2.1
    }
  ]
}
```

### Error Responses

- HTTP 422: Returned when validation fails (missing tenant, invalid date range, etc.).

```json
{
  "message": "Validation failed",
  "errors": {
    "filters.end_date": [
      "The filters.end_date field is required."
    ]
  }
}
```

- HTTP 500: Returned when the service is unable to retrieve rainfall metrics. The response body follows the standard error envelope: `{ "success": false, "message": "Failed to retrieve rainfall data" }`.

## **💨 16.** ET

**Endpoint**: `GET /api/water/et`

**Description**: Returns daily evapotranspiration (ET) metrics for the requested tenant and date range. Values are aggregated by day and include cumulative totals. When an ET correction factor exists for the tenant, adjusted values are also returned.

**Authentication**: Requires a valid application bearer token (see global authentication guide).

### Query Parameters

- `tenant_id` (uuid, required): Tenant identifier. Must reference an existing tenant record.
- `filters[start_date]` (date, required): Start of the reporting window in `YYYY-MM-DD` format.
- `filters[end_date]` (date, required): End of the reporting window in `YYYY-MM-DD` format. Must be on or after `filters[start_date]`.

### Successful Response

- HTTP 200
- Body shape:
    - `success` (boolean): Always `true` on success.
    - `message` (string): Human readable confirmation.
    - `data` (array): Ordered list with one entry per day in the requested range.
        - `timestamp` (string): Date in `YYYY-MM-DD`.
        - `et_value` (number): Daily ET value (millimeters).
        - `cumulative_value` (number): Sum of `et_value` from the beginning of the range to `timestamp`.
        - `adjusted_et_value` (number, optional): Daily ET value after applying the tenant ET correction factor (present only when a factor is configured).
        - `adjusted_cumulative_value` (number, optional): Sum of `adjusted_et_value` from the beginning of the range to `timestamp`.

```bash
curl -X GET "https://{host}/api/water/et" \\
  -H "Accept: application/json" \\
  -H "Authorization: Bearer <token>" \\
  --get \\
  --data-urlencode "tenant_id=11111111-2222-3333-4444-555555555555" \\
  --data-urlencode "filters[start_date]=2024-06-01" \\
  --data-urlencode "filters[end_date]=2024-06-15"
```

```json
{
  "success": true,
  "message": "Evapotranspiration data retrieved successfully",
  "data": [
    {
      "timestamp": "2024-06-01",
      "value": 4.2,
      "cumulative_value": 4.2,
      "adjusted_value": 4.62,
      "adjusted_cumulative_value": 4.62
    },
    {
      "timestamp": "2024-06-02",
      "value": 3.7,
      "cumulative_value": 7.9,
      "adjusted_value": 4.07,
      "adjusted_cumulative_value": 8.69
    }
  ]
}
```

### Error Responses

- HTTP 422: Returned when validation fails (missing tenant, invalid date range, etc.).

```json
{
  "message": "Validation failed",
  "errors": {
    "filters.start_date": [
      "The filters.start_date field is required."
    ]
  }
}
```

- HTTP 500: Returned when the service is unable to retrieve ET metrics. The response body follows the standard error envelope: `{ "success": false, "message": "Failed to retrieve evapotranspiration data" }`.

## 🗓️ **17.** Water Calendar Status API

## Overview

Update irrigation calendar status for a water source on a specific date. Returns mock data mirroring calendar entries for front-end integration before persistence.

## Endpoint

| Method | Endpoint | Description |
| --- | --- | --- |
| PUT | /water/calendar/sources/{source}/status | Updates irrigation status for source and date. |

## Path Parameters

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| source | integer | Yes | Water source ID. |

## Request Body

```json
{
  "date": "2025-01-18",
  "status": "planned"
}
```

### Body Fields

| Field | Type | Required | Allowed Values | Description |
| --- | --- | --- | --- | --- |
| date | string (Y-m-d) | Yes | Valid date | Irrigation date. |
| status | string | Yes | planned, denied, confirmed | New status. |

### Status Rules

- Only `planned`, `denied`, `confirmed` are mutable.
- Other statuses (e.g. `irrigated`, `unknown`) are rejected.
- Transitions allowed only among mutable statuses.

## Success Response

```json
{
  "success": true,
  "message": "Irrigation calendar status updated successfully",
  "data": {
    "id": 312,
    "name": "Outflow-Daily consumption",
    "status": "planned"
  }
}
```

### Response Fields

| Field | Type | Description |
| --- | --- | --- |
| success | boolean | Update succeeded. |
| message | string | Outcome message. |
| [data.id](http://data.id/) | integer | Source ID. |
| [data.name](http://data.name/) | string | Source name. |
| data.status | string | New status. |

## Error Responses

| Status | Body | When |
| --- | --- | --- |
| 400 | `{"success": false, "message": "Failed to update irrigation calendar status."}` | Server issue. |
| 404 | `{"success": false, "message": "Water source not found."}` | Source not found. |
| 422 | `{"message": "Validation failed", "errors": {"status": ["The selected status may only be planned, denied, or confirmed."]}}` | Invalid payload. |

## Notes

- Returns mock data; persistence pending.
- Dates in `Y-m-d` format.
- Use numeric source IDs; convert UUIDs beforehand.
- Errors follow platform validation structure.

---

## ✅ Error Handling

| HTTP Code | Meaning |
| --- | --- |
| 200 | Successful operation |
| 201 | Resource created successfully |
| 422 | Validation or business rule error |
| 500 | Internal error — operation failed |

---

## 🔌 Integration Tips

1. Prefer using explicit `/api/v2/` endpoints where provided; mixed-version base paths should align with backend routing config.
2. Times without seconds may be normalized to `HH:mm:00` server-side.
3. Numeric fields sometimes returned as strings (precision). Treat `reading_value`, `consumption_value`, `annual_water_allowance`, `last_meter_reading` as high-precision decimals.
4. When updating readings, ensure id corresponds to latest reading if business rule restricts modification.
5. For irrigation scheduling UI: combine `irrigation_frequency`, `irrigation_weekdays`, and period window to derive next run date.

# Water 2.0  Phase 2

[Water 2.0 Phase 2](https://www.notion.so/Water-2-0-Phase-2-2ee58558330a80acac5df35f2c5407aa?pvs=21)

[Water 2.0](https://www.notion.so/Water-2-0-2f758558330a80c197a8d50f059041a8?pvs=21)


# Irrigation Notifications API (Water)

This document describes the irrigation notifications endpoints used by the Water module. These endpoints are loaded under the `api/v2` namespace and protected by the standard authentication middleware. See the route setup in [app/Providers/RouteServiceProvider.php](https://www.notion.so/gvegroup/app/Providers/RouteServiceProvider.php#L18-L22).

## Base URL

- `https://<host>/api/v2/water`

## Authentication

- All endpoints require a valid JWT access token in the `Authorization: Bearer <token>` header.
- Tenant and caller context are derived from the token by middleware; the client does not need to send `tenantId` or `callerId` explicitly.

## Response Envelope

Successful responses are wrapped using a consistent envelope:

```json
{
	"success": true,
	"message": "<human readable message>",
	"data": <payload>
}

```

Error responses:

```json
{
	"success": false,
	"message": "<error description>"
}

```

---

## GET /irrigation-notifications

List planned irrigation notifications for the authenticated user's tenant. Only notifications with status `planned` and `expires_at > now` are returned. Results include water source context and adjacent readings.

### Request

- Method: `GET`
- URL: `/api/v2/water/irrigation-notifications`
- Headers:
    - `Authorization: Bearer <token>`
- Query: none

### Response

- Status: `200 OK`
- Envelope: success
- `data`: array of notification objects

Notification object shape:

```json
{
	"id": 123,
	"irrigation_date": "2026-01-30",
	"irrigation_status": "planned",
	"expires_at": "2026-02-05T12:00:00Z",
	"water_source": {
		"id": 55,
		"name": "North Pump",
		"source_type": "well",
		"measurement_type": "meter_reading" | "daily_consumption",
		"prev_reading": {
			"id": 9876,
			"reading_date": "2026-01-29",
			"value": 1234.5
		},
		"next_reading": {
			"id": 9880,
			"reading_date": "2026-01-31",
			"value": 1240.0
		},
		"suggested_value": 1234.5
	}
}

```

Notes:

- `prev_reading` and `next_reading` are included when available and shaped to `{ id, reading_date, value }`. The `value` corresponds to `meter_reading_value` for `meter_reading` sources, or `consumption_value` for `daily_consumption` sources.
- `suggested_value` is present only when no `next_reading` exists for the notification date; it mirrors the previous reading value and helps the user quickly confirm a reading.

### Example

```bash
curl -H "Authorization: Bearer $TOKEN" \\
	https://<host>/api/v2/water/irrigation-notifications

```

---

## PATCH /irrigation-notifications/{notification}

Update a single planned notification to `confirmed` or `denied`. If `confirmed` and an irrigation value is provided, the system may create a water reading for that date (subject to validation rules and source measurement type).

### Request

- Method: `PATCH`
- URL: `/api/v2/water/irrigation-notifications/{notification}`
- Headers:
    - `Authorization: Bearer <token>`
- Path params:
    - `notification`: integer ID of the irrigation notification
- Body (JSON):

```json
{
	"status": "confirmed" | "denied",
	"irrigation_value": 42.0
}

```

Validation:

- `status` is required; must be one of `confirmed`, `denied`.
- `irrigation_value` must be a number if present, and is prohibited when `status` is `denied`.

Behavior:

- Only notifications in `planned` state can be updated.
- Tenant scoping is enforced: the notification must belong to a water source in the caller's tenant.
- When `status = confirmed` and `irrigation_value` is provided:
    - For `daily_consumption` sources: creates a daily consumption reading on the notification date.
    - For `meter_reading` sources: creates a meter reading on the notification date only if a reading does not already exist for that date; otherwise just updates the status.
    - Reading creation follows strict validation rules relative to previous/next readings and the source's `last_meter_reading`/`last_reading_date`.

### Response

- Status: `200 OK` (on success)
- Envelope: success
- `data`: updated notification object (same shape as in GET)

### Error Statuses

- `401 Unauthorized`: missing/invalid token
- `403 Forbidden`: notification does not belong to the tenant
- `422 Unprocessable Entity`: validation failed (invalid status/value or reading validation)
- `500 Internal Server Error`: unexpected failure

### Examples

Confirm with a value:

```bash
curl -X PATCH -H "Authorization: Bearer $TOKEN" \\
	-H "Content-Type: application/json" \\
	-d '{
				"status": "confirmed",
				"irrigation_value": 38.5
			}' \\
	https://<host>/api/v2/water/irrigation-notifications/123

```

Deny without a value:

```bash
curl -X PATCH -H "Authorization: Bearer $TOKEN" \\
	-H "Content-Type: application/json" \\
	-d '{ "status": "denied" }' \\
	https://<host>/api/v2/water/irrigation-notifications/123

```

---

## POST /irrigation-notifications/batch-update

Batch update multiple planned notifications. This endpoint is intended for bulk `confirmed` updates where each item carries an `irrigation_value`. Denying in batch is not supported (see validation).

### Request

- Method: `POST`
- URL: `/api/v2/water/irrigation-notifications/batch-update`
- Headers:
    - `Authorization: Bearer <token>`
- Body (JSON):

```json
{
	"status": "confirmed",
	"items": [
		{ "id": 101, "irrigation_value": 37.2 },
		{ "id": 102, "irrigation_value": 41.9 }
	]
}

```

Validation:

- `status` is required; must be `confirmed` or `denied`.
- When `status = confirmed`:
    - `items` is required and must be an array.
    - For each item: `id` is required (integer), `irrigation_value` is required (numeric).
- When `status = denied`:
    - `items` is prohibited (omit the field). Denying in batch is effectively a no-op.

Processing:

- Items are grouped by `water_source_id` and processed in chronological order (`irrigation_date`).
- For each notification, applies the same confirmation logic and validations described in the single `PATCH` endpoint.

### Response

- Status: `200 OK`
- Envelope: success
- `data`:

```json
{
	"updated": [
		{ /* updated notification object */ },
		{ /* updated notification object */ }
	]
}

```

### Error Statuses

- `401 Unauthorized`: missing/invalid token
- `403 Forbidden`: one or more notifications not in tenant scope
- `422 Unprocessable Entity`: validation failed (payload shape or reading validation)

### Example

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \\
	-H "Content-Type: application/json" \\
	-d '{
				"status": "confirmed",
				"items": [
					{ "id": 101, "irrigation_value": 37.2 },
					{ "id": 102, "irrigation_value": 41.9 }
				]
			}' \\
	https://<host>/api/v2/water/irrigation-notifications/batch-update

```

---

## Frontend Integration Notes

- Use `GET /irrigation-notifications` to populate the UI with pending items and their `suggested_value` and adjacent readings for context.
- For confirmation flows:
    - Prefer the `suggested_value` when appropriate; allow manual override with validation feedback.
    - Use `PATCH /irrigation-notifications/{id}` for single updates.
    - Use `POST /irrigation-notifications/batch-update` for bulk confirmations with values.
- Handle validation errors (`422`) by displaying the server `message`. Meter reading validations may include bounds like "must be greater than previous and less than next/last meter".
- Always include `Authorization: Bearer <token>`; the backend derives tenant and user context.