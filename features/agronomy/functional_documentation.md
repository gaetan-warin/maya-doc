# Agronomy & Nutritional Management
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-AGRO-001 |
| **Version** | 3.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Agronomy & Nutritional Management** module transforms raw environmental data into actionable agronomic intelligence. It enables turf management professionals to predict disease outbreaks, optimize fertilizer applications, and ensure regulatory compliance for chemical usage.

This document defines the functional requirements for disease risk modeling, precision nutrient application, and agronomic reporting. It establishes three operational modes (Standard, Advanced, Precision) to accommodate varying levels of application complexity and compliance requirements.

**Key Business Outcomes:**
- **Reduced Chemical Costs**: Predictive disease alerts enable targeted treatment, reducing blanket applications by up to 30%.
- **Regulatory Compliance**: Precision mode captures all parameters required for EU chemical application audits.
- **Optimized Turf Health**: Growth Potential-based nitrogen pacing prevents over-fertilization and nutrient leaching.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Disease risk prediction and alerting
- Nutritional input planning and recording
- Precision spraying parameters (nozzle, pressure, speed)
- Nitrogen balance tracking
- Cost and quantity analysis

### 1.2 Out of Scope
- Product procurement (handled by Stock Module)
- Equipment calibration procedures (operational training)

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **Head Greenkeeper / Superintendent** | Disease monitoring, fertilization strategy |
| **Spray Technician** | Application execution, parameter entry |
| **Agronomist Consultant** | Season analysis, recommendation validation |
| **Compliance Officer** | Audit trail review, regulatory reporting |

---

## 2. Disease Risk Modeling

### 2.1 Overview

Maya aggregates weather data (temperature, humidity, leaf wetness, rainfall) to calculate infection probability for turf pathogens. Risk models provide 7-day forecasts based on both historical sensor data and weather predictions.

### 2.2 Supported Disease Models

| Disease | Model Code | Environmental Drivers | Output Scale |
|:---|:---|:---|:---|
| **Dollar Spot** | `DOLLAR_SPOT` | Temperature, Humidity (Smith-Kerns algorithm) | 0–100% probability |
| **Fusarium Patch** | `FUSA_2` | Cool temperatures, prolonged leaf wetness | Low / Medium / High |
| **Anthracnose** | `ANTHRACNOSE` | Heat stress, drought stress index | Probability index |
| **Brown Patch** | `RHIZOCTONIA` | High temperature + high humidity | Risk index |
| **Gray Leaf Spot** | `GLS` | Pyricularia grisea conditions | Risk index |

### 2.3 Alert Logic

> ⚠️ **Important**: Risk models are predictive indicators. A "High Risk" alert signifies that environmental conditions favor pathogen development—it does not confirm active infection.

- **Low Risk (Green)**: No action required
- **Medium Risk (Yellow)**: Increase monitoring frequency
- **High Risk (Red)**: Consider preventive treatment

---

## 3. Nutritional Application Modes

### 3.1 Standard Mode

For granular fertilizer applications requiring minimal technical input.

**User Inputs:**
- Product selection
- Application rate (kg/ha or g/m²)
- Target area (site, zone, hole)

**System Outputs:**
- Total quantity required
- N-P-K units applied

### 3.2 Advanced Spraying Mode

For liquid applications requiring tank mix calculations.

**User Inputs:**
- Product selection and rate
- Dissolution volume per hectare
- Tank capacity
- Tractor speed (km/h)

**System Calculations:**
| Output | Formula |
|:---|:---|
| Total Water Volume | Application Rate × Target Area |
| Number of Tanks | Total Volume ÷ Tank Capacity |
| Recommended Flow Rate | Based on speed, width, and volume |
| Nitrogen Units | Product N% × Quantity Applied |

### 3.3 Precision Management Mode

For high-compliance environments requiring full equipment traceability.

**Additional User Inputs:**
- Nozzle type
- Nozzle spacing
- Operating pressure
- RPM

**Additional System Calculations:**
- Nozzle flow rate validation
- Drift risk assessment
- Coverage uniformity estimate

**Compliance Value:**
This mode captures all parameters required for:
- EU chemical application regulations
- Health & Safety incident investigation
- Legal audit trails

---

## 4. Growth Potential & Nitrogen Pacing

### 4.1 Growth Potential (GP)

A temperature-based metric (0–100%) indicating the turf's theoretical growth rate.

| Grass Type | Optimal Temperature | GP Formula Basis |
|:---|:---|:---|
| Cool Season (C3) | ~20°C (68°F) | Modified for bentgrass, ryegrass |
| Warm Season (C4) | ~31°C (88°F) | Modified for bermudagrass, zoysiagrass |

### 4.2 Nitrogen Pacing Strategy

> 💡 **Best Practice**: Match nitrogen input to Growth Potential. If GP = 50%, the plant can only metabolize 50% of its maximum nitrogen uptake.

Excess nitrogen during low GP periods leads to:
- Soft, disease-susceptible growth
- Nutrient leaching to groundwater
- Wasted chemical investment

### 4.3 Growing Degree Days (GDD)

Used for Plant Growth Regulator (PGR) timing.

| Parameter | Description |
|:---|:---|
| Base Temperature | Configurable (0°C, 6°C, or 10°C) |
| Target GDD | User-defined threshold for reapplication |

---

## 5. Stock Consumption & Cost Tracking

### 5.1 Consumption Categories

| Input Type | Stock Category | Examples |
|:---|:---|:---|
| **Nutritional Input** | Product | Fertilizers, herbicides, fungicides, wetting agents, PGRs |
| **Standard Task** | Non-Product | Fuel, sand, stakes, paint, tools |

### 5.2 Cost Calculation Logic

| Metric | Formula |
|:---|:---|
| Unit Cost | Product price ÷ product quantity |
| Application Cost | Unit cost × quantity applied |
| Cost per Hectare | Application cost ÷ treated area |

### 5.3 Pro-Rata Distribution

When a nutritional input is applied to a zone (e.g., "Greens"), costs are distributed to individual holes proportionally based on surface area.

**Example:**
- Total application: 12 kg/ha
- Green 1: 0.05 ha → 0.6 kg allocated
- Green 2: 0.08 ha → 0.96 kg allocated

---

## 6. Reporting & Analytics

### 6.1 Available Reports

| Report | Purpose |
|:---|:---|
| **Treatment List** | Chronological log of all applications |
| **Nutritional Timeline** | Time-series visualization of nutrient inputs |
| **Actual vs. Planned** | Variance analysis for fertilization programs |
| **Cost Analysis** | Product and category-level expenditure |

### 6.2 Export Capabilities

- PDF: Formatted for audit submission
- CSV: Raw data for external analysis
- API: Integration with third-party systems

---

## 7. Integration Points

| System | Integration Type | Data Flow |
|:---|:---|:---|
| **Weather Module** | Real-time feed | Temperature, humidity, leaf wetness for disease models |
| **Stock Module** | Transactional | Product availability check, consumption recording |
| **Map Module** | Reference | Surface area calculation for zone-based applications |
| **Task Module** | Linked | Application tasks appear in daily schedule |

---

## 8. Acceptance Criteria

1. Disease risk predictions update automatically when new weather data is received.
2. Advanced mode correctly calculates tank numbers and flow rates based on user inputs.
3. Precision mode captures and stores all equipment parameters for audit retrieval.
4. Nitrogen units are calculated automatically based on product composition.
5. Costs are distributed pro-rata when applications target aggregate zones.

---

**Document End**
