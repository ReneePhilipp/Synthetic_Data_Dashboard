# Fleet Maintenance Dashboard - Technical Documentation

**Project**: FleetInsights Synthetic Data  
**Date**: January 2025  
**Purpose**: Data dictionaries and SQL logic for customer UI integration  

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Data Architecture Overview](#data-architecture-overview)
3. [Dataset Documentation](#dataset-documentation)
   - [SYNTHETIC_OBD2 (Base Dataset)](#synthetic_obd2-base-dataset)
   - [SYNTHETIC_OBD2_DAILY (Summary Dataset)](#synthetic_obd2_daily-summary-dataset)
4. [Dashboard Metrics & SQL Logic](#dashboard-metrics--sql-logic)
5. [Implementation Notes](#implementation-notes)

---

## Executive Summary

This document provides comprehensive technical documentation for the Fleet Maintenance Dashboard, including:

- **Data Dictionaries** for both base and summary datasets
- **SQL Statements** for all dashboard metrics and visualizations
- **Field Definitions** with examples and business context
- **Implementation Guidelines** for customer UI integration

The dashboard provides public safety fleet managers with real-time insights into vehicle maintenance needs, reliability risks, and operational status.

---

## Data Architecture Overview

```
┌─────────────────┐    Daily ETL    ┌──────────────────────┐
│  SYNTHETIC_OBD2 │ ────────────► │ SYNTHETIC_OBD2_DAILY │
│  (Raw Telemetry)│               │   (Fleet Summary)    │
│  ~100K records  │               │    One per VIN       │
└─────────────────┘               └──────────────────────┘
         │                                    │
         │                                    │
         ▼                                    ▼
┌─────────────────┐               ┌──────────────────────┐
│   Historical    │               │    Dashboard KPIs    │
│    Analysis     │               │   & Visualizations   │
└─────────────────┘               └──────────────────────┘
```

**Key Principles:**
- **Base Table**: Raw vehicle telemetry (SYNTHETIC_OBD2)
- **Summary Table**: Daily aggregated metrics (SYNTHETIC_OBD2_DAILY)
- **Real-time Updates**: Base table updated continuously
- **Daily Refresh**: Summary table refreshed once daily

---

## Dataset Documentation

### SYNTHETIC_OBD2 (Base Dataset)

**Purpose**: Raw telemetry data from fleet vehicles with OBD2 diagnostic information  
**Update Frequency**: Real-time (per driving event)  
**Data Source**: `obd2_data.py` script  
**Record Count**: ~100,000 records  
**Data Retention**: 30 days of operational data  

#### Field Specifications

| Field Name | Data Type | Length | Nullable | Description | Example Values |
|------------|-----------|--------|----------|-------------|----------------|
| **VEHICLE_VIN** | STRING | 17 | No | Vehicle Identification Number (Primary Key) | 1CH1234567890123 |
| **TENANT** | STRING | 50 | No | Fleet organization identifier | FIRE_DEPT_001, POLICE_002 |
| **VEHICLE_MAKE** | STRING | 20 | No | Vehicle manufacturer | CHEVROLET, FORD |
| **VEHICLE_MODEL** | STRING | 30 | No | Vehicle model name | EXPRESS, F150, TAHOE |
| **VEHICLE_YEAR** | INTEGER | - | No | Model year | 2019, 2023 |
| **REPORTED_DATE_UTC** | TIMESTAMP | - | No | When data was collected (UTC) | 2025-08-15 14:30:00 |
| **ENGINE_LIGHT_ON** | BOOLEAN | - | No | Check engine light status | TRUE, FALSE |
| **ENGINE_LIGHT_CODES** | STRING | 1000 | Yes | JSON of active diagnostic codes | {"P0171": "SYSTEM TOO LEAN"} |
| **ODOMETER_MILES** | FLOAT | - | No | Current vehicle mileage | 45,231.7 |
| **ENGINE_COOLANT_TEMP_F** | FLOAT | - | No | Engine temperature (Fahrenheit) | 195.2 |
| **OIL_PRESSURE_PSI** | FLOAT | - | No | Oil pressure (PSI) | 42.1 |
| **TRANSMISSION_TEMP_F** | FLOAT | - | No | Transmission temperature (Fahrenheit) | 185.4 |
| **TIRE_PRESSURE_FL_PSI** | FLOAT | - | No | Front left tire pressure (PSI) | 34.2 |
| **TIRE_PRESSURE_FR_PSI** | FLOAT | - | No | Front right tire pressure (PSI) | 33.8 |
| **TIRE_PRESSURE_RL_PSI** | FLOAT | - | No | Rear left tire pressure (PSI) | 35.1 |
| **TIRE_PRESSURE_RR_PSI** | FLOAT | - | No | Rear right tire pressure (PSI) | 34.9 |
| **RPM** | INTEGER | - | No | Engine RPM | 750, 2400 |
| **SPEED_MPH** | FLOAT | - | No | Vehicle speed (MPH) | 35.2 |
| **THROTTLE_PERCENTAGE** | FLOAT | - | No | Throttle position (%) | 25.3 |
| **MAF_G_SEC** | FLOAT | - | No | Mass Air Flow (grams/second) | 4.2 |
| **FUEL_LEVEL_PERCENTAGE** | FLOAT | - | No | Fuel tank level (%) | 67.3 |
| **MILES_SINCE_CLEARED** | FLOAT | - | No | Miles driven since current codes appeared | 1,247.3 |
| **DAYS_SINCE_CLEARED** | FLOAT | - | No | Days since current codes appeared | 12.4 |
| **MOST_SIGNIFICANT_ISSUE** | STRING | 200 | Yes | Highest priority diagnostic issue | TRANSMISSION OVERHEATING |
| **SERVICE_PRIORITY** | STRING | 50 | No | Maintenance urgency level | IMMEDIATE SERVICE REQUIRED |
| **CREATE_DATE** | DATE | - | No | Record creation date | 2025-08-15 |

#### Service Priority Values

| Priority Level | Description | Action Required |
|----------------|-------------|-----------------|
| **NO SERVICE REQUIRED** | Vehicle operating normally | Continue monitoring |
| **ROUTINE MAINTENANCE** | Scheduled maintenance due | Schedule within 30 days |
| **MONITOR - SCHEDULE SERVICE** | Potential issue detected | Schedule within 14 days |
| **SERVICE NEEDED WITHIN 7 DAYS** | Active problem requiring attention | Schedule within 7 days |
| **IMMEDIATE SERVICE REQUIRED** | Critical issue, safety risk | Service immediately |

---

### SYNTHETIC_OBD2_DAILY (Summary Dataset)

**Purpose**: Daily fleet maintenance summary with vehicle-level aggregations  
**Update Frequency**: Daily (latest vehicle status)  
**Data Source**: `obd2_data_daily.py` script  
**Record Count**: One record per active VIN (~62 vehicles)  
**Data Retention**: 90 days of daily summaries  

#### Field Specifications

| Field Name | Data Type | Length | Nullable | Description | Example Values |
|------------|-----------|--------|----------|-------------|----------------|
| **VEHICLE_VIN** | STRING | 17 | No | Vehicle Identification Number (Primary Key) | 1CH1234567890123 |
| **TENANT** | STRING | 50 | No | Fleet organization identifier | FIRE_DEPT_001 |
| **MAKE** | STRING | 20 | No | Vehicle manufacturer | CHEVROLET, FORD |
| **MODEL** | STRING | 30 | No | Vehicle model name | EXPRESS, F150, TAHOE |
| **YEAR** | INTEGER | - | No | Model year | 2019, 2023 |
| **CURRENT_ENGINE_TEMP** | FLOAT | - | Yes | Latest engine temperature reading | 195.2 |
| **MOST_URGENT_ISSUE** | STRING | 200 | Yes | Highest priority active issue | TRANSMISSION OVERHEATING |
| **LAST_READING_TIMESTAMP** | TIMESTAMP | - | No | When last telemetry was received | 2025-08-15 14:30:00 |
| **CURRENT_ODOMETER** | FLOAT | - | No | Latest odometer reading | 45,231.7 |
| **DAYS_OFFLINE** | INTEGER | - | No | Days since last communication | 0, 3, 15 |
| **VEHICLE_STATUS** | STRING | 20 | No | Operational status | ACTIVE, RECENT_OFFLINE, LONG_OFFLINE |
| **TOTAL_DELINQUENT_MILES** | FLOAT | - | No | Miles driven with active issues | 1,247.3 |
| **TOTAL_DELINQUENT_DAYS** | FLOAT | - | No | Days with active issues | 12.4 |
| **ACTIVE_CODE_COUNT** | INTEGER | - | No | Number of active diagnostic codes | 0, 2, 5 |
| **MOST_DELINQUENT_CODE** | STRING | 10 | Yes | Longest-persisting diagnostic code | P0171 |
| **MOST_DELINQUENT_MILES** | FLOAT | - | No | Miles for most persistent issue | 1,247.3 |
| **SERVICE_PRIORITY** | STRING | 50 | No | Maintenance urgency level | IMMEDIATE SERVICE REQUIRED |
| **RELIABILITY_RISK_SCORE** | FLOAT | - | No | Mission failure risk (0.0-1.0) | 0.347 (34.7% risk) |
| **CURRENT_OIL_PRESSURE** | FLOAT | - | Yes | Latest oil pressure reading | 42.1 |
| **CURRENT_TRANSMISSION_TEMP** | FLOAT | - | Yes | Latest transmission temperature | 185.4 |
| **CURRENT_TIRE_PRESSURE_FL** | FLOAT | - | Yes | Latest front left tire pressure | 34.2 |
| **CURRENT_TIRE_PRESSURE_FR** | FLOAT | - | Yes | Latest front right tire pressure | 33.8 |
| **CURRENT_TIRE_PRESSURE_RL** | FLOAT | - | Yes | Latest rear left tire pressure | 35.1 |
| **CURRENT_TIRE_PRESSURE_RR** | FLOAT | - | Yes | Latest rear right tire pressure | 34.9 |
| **ACTIVE_ENGINE_CODES** | STRING | 500 | Yes | Comma-separated list of active codes | P0171, P0300, P0420 |
| **ACTIVE_DELINQUENT_DETAILS** | STRING | 2000 | Yes | JSON with detailed issue tracking | {"P0171": {"miles": 1247, "days": 12}} |
| **CREATED_DATE** | DATE | - | No | Summary creation date | 2025-08-15 |

#### Vehicle Status Definitions

| Status | Description | Days Offline | Color Code |
|--------|-------------|--------------|------------|
| **ACTIVE** | Currently transmitting data | 0-1 days | 🟢 Green |
| **RECENT_OFFLINE** | Recently went offline | 2-7 days | 🟡 Yellow |
| **LONG_OFFLINE** | Extended offline period | 8+ days | 🔴 Red |

#### Reliability Risk Score

**Range**: 0.0 to 1.0 (0% to 100% risk)  
**Purpose**: Quantifies probability of mission failure during emergency response  

**Risk Categories**:
- **0.0 - 0.30**: Low Risk (Mission Ready)
- **0.31 - 0.60**: Moderate Risk (Monitor Closely)  
- **0.61 - 1.0**: High Risk (Immediate Action Required)

**Calculation Factors**:
- Base risk by service priority
- Miles driven with active issues
- Time elapsed since issues first appeared

---

## Dashboard Metrics & SQL Logic

### KPI Cards (Top Row)

#### 1. Vehicles Needing Work
**Definition**: Count of vehicles requiring any level of service  
```sql
SELECT COUNT(*) as vehicles_needing_work
FROM SYNTHETIC_OBD2_DAILY 
WHERE SERVICE_PRIORITY != 'NO SERVICE REQUIRED';
```

#### 2. Vehicles To Fix Now  
**Definition**: Count of vehicles requiring immediate attention  
```sql
SELECT COUNT(*) as vehicles_to_fix_now
FROM SYNTHETIC_OBD2_DAILY 
WHERE SERVICE_PRIORITY = 'IMMEDIATE SERVICE REQUIRED';
```

#### 3. Days Overdue (Median)
**Definition**: Median days that active issues have persisted  
```sql
SELECT MEDIAN(TOTAL_DELINQUENT_DAYS) as median_days_overdue
FROM SYNTHETIC_OBD2_DAILY 
WHERE ACTIVE_CODE_COUNT > 0;
```

#### 4. Miles Overdue (Median)  
**Definition**: Median miles driven with active diagnostic codes  
```sql
SELECT MEDIAN(TOTAL_DELINQUENT_MILES) as median_miles_overdue
FROM SYNTHETIC_OBD2_DAILY 
WHERE ACTIVE_CODE_COUNT > 0;
```

#### 5. Reliability Risk Percentage
**Definition**: Average fleet-wide reliability risk score  
```sql
SELECT ROUND(AVG(RELIABILITY_RISK_SCORE) * 100, 1) as avg_reliability_risk_pct
FROM SYNTHETIC_OBD2_DAILY;
```

### Vehicle Maintenance Table

**Purpose**: Detailed list of vehicles requiring service, prioritized by urgency  

```sql
SELECT 
    SERVICE_PRIORITY,
    VEHICLE_VIN,
    MAKE,
    MODEL, 
    YEAR,
    MOST_URGENT_ISSUE,
    ACTIVE_CODE_COUNT as num_codes,
    ACTIVE_ENGINE_CODES as all_active_codes,
    CURRENT_ODOMETER as last_odometer
FROM SYNTHETIC_OBD2_DAILY 
WHERE SERVICE_PRIORITY != 'NO SERVICE REQUIRED'
ORDER BY 
    CASE SERVICE_PRIORITY
        WHEN 'IMMEDIATE SERVICE REQUIRED' THEN 1
        WHEN 'SERVICE NEEDED WITHIN 7 DAYS' THEN 2  
        WHEN 'MONITOR - SCHEDULE SERVICE' THEN 3
        WHEN 'ROUTINE MAINTENANCE' THEN 4
        ELSE 5
    END,
    TOTAL_DELINQUENT_DAYS DESC;
```

### Vehicle Operating Range Gauges

#### Transmission Temperature
**Purpose**: Show min/median/max transmission temperatures across fleet  
```sql
SELECT 
    MIN(CURRENT_TRANSMISSION_TEMP) as min_temp,
    ROUND(MEDIAN(CURRENT_TRANSMISSION_TEMP), 0) as median_temp,
    MAX(CURRENT_TRANSMISSION_TEMP) as max_temp
FROM SYNTHETIC_OBD2_DAILY;
```

#### Oil Pressure  
**Purpose**: Show min/median/max oil pressure readings across fleet  
```sql
SELECT 
    MIN(CURRENT_OIL_PRESSURE) as min_pressure,
    ROUND(MEDIAN(CURRENT_OIL_PRESSURE), 0) as median_pressure,
    MAX(CURRENT_OIL_PRESSURE) as max_pressure
FROM SYNTHETIC_OBD2_DAILY;
```

#### Engine Coolant Temperature
**Purpose**: Show min/median/max engine temperatures across fleet  
```sql
SELECT 
    MIN(CURRENT_ENGINE_TEMP) as min_temp,
    ROUND(MEDIAN(CURRENT_ENGINE_TEMP), 0) as median_temp, 
    MAX(CURRENT_ENGINE_TEMP) as max_temp
FROM SYNTHETIC_OBD2_DAILY;
```

#### Mass Air Flow (MAF)
**Purpose**: Show min/median/max air flow readings from latest telemetry  
```sql
SELECT 
    MIN(MAF_G_SEC) as min_maf,
    ROUND(MEDIAN(MAF_G_SEC), 2) as median_maf,
    MAX(MAF_G_SEC) as max_maf
FROM SYNTHETIC_OBD2 s1
WHERE s1.REPORTED_DATE_UTC = (
    SELECT MAX(s2.REPORTED_DATE_UTC) 
    FROM SYNTHETIC_OBD2 s2 
    WHERE s2.VEHICLE_VIN = s1.VEHICLE_VIN
);
```

#### Engine RPM
**Purpose**: Show min/median/max RPM readings from latest telemetry  
```sql
SELECT 
    MIN(RPM) as min_rpm,
    ROUND(MEDIAN(RPM), 1) as median_rpm,
    MAX(RPM) as max_rpm
FROM SYNTHETIC_OBD2 s1
WHERE s1.REPORTED_DATE_UTC = (
    SELECT MAX(s2.REPORTED_DATE_UTC)
    FROM SYNTHETIC_OBD2 s2 
    WHERE s2.VEHICLE_VIN = s1.VEHICLE_VIN  
);
```

#### Intake Air Temperature
**Purpose**: Show min/median/max intake air temperatures from latest telemetry  
```sql
SELECT 
    MIN(INTAKE_AIR_TEMP_F) as min_temp,
    ROUND(MEDIAN(INTAKE_AIR_TEMP_F), 0) as median_temp,
    MAX(INTAKE_AIR_TEMP_F) as max_temp  
FROM SYNTHETIC_OBD2 s1
WHERE s1.REPORTED_DATE_UTC = (
    SELECT MAX(s2.REPORTED_DATE_UTC)
    FROM SYNTHETIC_OBD2 s2
    WHERE s2.VEHICLE_VIN = s1.VEHICLE_VIN
);
```

#### Tire Pressure (All Four Tires)
**Purpose**: Show min/median/max tire pressure for each tire position  

**Front Left:**
```sql
SELECT 
    MIN(CURRENT_TIRE_PRESSURE_FL) as min_psi,
    ROUND(MEDIAN(CURRENT_TIRE_PRESSURE_FL), 0) as median_psi,
    MAX(CURRENT_TIRE_PRESSURE_FL) as max_psi
FROM SYNTHETIC_OBD2_DAILY;
```

**Front Right:**
```sql
SELECT 
    MIN(CURRENT_TIRE_PRESSURE_FR) as min_psi,
    ROUND(MEDIAN(CURRENT_TIRE_PRESSURE_FR), 0) as median_psi,
    MAX(CURRENT_TIRE_PRESSURE_FR) as max_psi
FROM SYNTHETIC_OBD2_DAILY;
```

**Rear Left:**
```sql
SELECT 
    MIN(CURRENT_TIRE_PRESSURE_RL) as min_psi,
    ROUND(MEDIAN(CURRENT_TIRE_PRESSURE_RL), 0) as median_psi,
    MAX(CURRENT_TIRE_PRESSURE_RL) as max_psi
FROM SYNTHETIC_OBD2_DAILY;
```

**Rear Right:**
```sql
SELECT 
    MIN(CURRENT_TIRE_PRESSURE_RR) as min_psi,
    ROUND(MEDIAN(CURRENT_TIRE_PRESSURE_RR), 0) as median_psi,
    MAX(CURRENT_TIRE_PRESSURE_RR) as max_psi
FROM SYNTHETIC_OBD2_DAILY;
```

### Dashboard Filtering Logic

#### Filter by Tenant
```sql
WHERE TENANT = 'FIRE_DEPT_001'
```

#### Filter by Service Priority  
```sql
WHERE SERVICE_PRIORITY IN ('IMMEDIATE SERVICE REQUIRED', 'SERVICE NEEDED WITHIN 7 DAYS')
```

#### Filter by Vehicle Make
```sql
WHERE MAKE = 'CHEVROLET'
```

#### Filter by Vehicle Model
```sql
WHERE MODEL IN ('TAHOE', 'EXPRESS') 
```

#### Filter by Year Range
```sql
WHERE YEAR BETWEEN 2020 AND 2024
```

#### Combined Filter Example
```sql
SELECT COUNT(*) as critical_vehicles
FROM SYNTHETIC_OBD2_DAILY 
WHERE TENANT = 'FIRE_DEPT_001'
  AND SERVICE_PRIORITY = 'IMMEDIATE SERVICE REQUIRED'
  AND MAKE = 'CHEVROLET'
  AND YEAR >= 2020;
```

---

## Implementation Notes

### Data Refresh Schedule
- **Base Table (SYNTHETIC_OBD2)**: Real-time updates as vehicle data streams in
- **Summary Table (SYNTHETIC_OBD2_DAILY)**: Daily refresh at 6:00 AM UTC
- **Dashboard Refresh**: Every 15 minutes during business hours

### Performance Considerations
- **Indexing**: Primary indexes on VEHICLE_VIN, TENANT, SERVICE_PRIORITY
- **Partitioning**: Base table partitioned by REPORTED_DATE_UTC (daily partitions)
- **Data Retention**: Base table retains 30 days, summary table retains 90 days

### Data Quality Checks
- **VIN Validation**: All VINs must be 17 characters, alphanumeric
- **Timestamp Validation**: All timestamps must be within last 30 days
- **Diagnostic Code Validation**: All codes must match standard OBD2 format (P####)

### Security & Access
- **Row-Level Security**: Users can only access data for their assigned tenant
- **API Rate Limiting**: Dashboard queries limited to 100 requests per minute
- **Data Encryption**: All data encrypted at rest and in transit

### Mobile Optimization
- **Responsive Design**: Dashboard adapts to mobile screen sizes
- **Touch Interface**: All interactions optimized for touch devices
- **Offline Mode**: Critical metrics cached for offline viewing

### Error Handling
- **Missing Data**: NULL values handled gracefully in all calculations
- **Connection Timeouts**: Automatic retry logic with exponential backoff
- **Data Validation**: Invalid records logged and excluded from aggregations

---

**Document Version**: 1.0  
**Last Updated**: January 2025  
**Contact**: Fleet Analytics Team  
**Next Review**: Quarterly 