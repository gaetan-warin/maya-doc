# Maya Weather API Documentation

## Steps to Fetch Metrics

### 1. Generate API Token (Frontend)
1. Open: `https://app.mayaglobal.io/setting`
2. Go to "API token Setting"
3. Click **Add token** → give it a name → **Create**
4. Copy and save the generated token (used as Bearer token)

### 2. Create Request in Postman/Client

**Method:** `GET`

**URL:**
```
https://api2.mayaglobal.io/api/v4/metrics
```

**Query Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `codes` | **Yes** | Comma-separated list of metric keys |
| `idsite` | No | Site UUID. If omitted, returns data for all sites |

**Example URL:**
```
https://api2.mayaglobal.io/api/v4/metrics?codes=air_temperature,air_humidity,wind_speed,wind_gust,radiation,rainfall,et24,et,leaf_wetness,soil_temperature,soil_humidity,soil_oxygen,soil_salinity,dli
```

### 3. Authorization
- **Type:** Bearer Token
- **Token:** Paste the token from step 1

### 4. Response
JSON with requested metric data. If `idsite` not provided, each metric entry includes site id and site name.

---

## Available Weather Codes

| Code | Metric | Unit |
|------|--------|------|
| `air_temperature` | Air temperature | °C |
| `air_humidity` | Air humidity | % |
| `wind_speed` | Wind speed | m/s |
| `wind_gust` | Wind gust | m/s |
| `radiation` | Solar radiation | W/m² |
| `rainfall` | Precipitation amount | mm |
| `et24` | 24-hour evapotranspiration | mm/24h |
| `et` | Evapotranspiration | mm |
| `leaf_wetness` | Leaf wetness | % |
| `soil_temperature` | Soil temperature at 5cm depth | °C |
| `soil_humidity` | Soil humidity at 5cm depth | % |
| `soil_oxygen` | Soil Oxygen | % |
| `soil_salinity` | Soil salinity | dS/m |
| `dli` | Daily Light Integral | mol/m²/day |

---

## Notes
- Ensure `codes` only contains valid metric keys from the allowed list
- If you get **401** — verify the token and that it wasn't expired or revoked

---

(Archived from Notion on 2025-12-22)
