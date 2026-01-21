# Mediflow Healthcare Analytics Pipeline

## Summary

End-to-end cloud-based healthcare data analytics solution leveraging AWS services (S3, Glue, Athena) and Power BI to analyze 25,000 hospital admissions, tracking severity patterns, length of stay, and 30-day readmissions across multiple cities and diagnoses.

**Key Findings:** 33% severe case rate, 5.52-day average length of stay, 1.66% 30-day readmission rate, with chronic conditions (kidney disease, heart failure) showing highest readmission risk.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Data Pipeline](#data-pipeline)
- [Technologies Used](#technologies-used)
- [Setup & Configuration](#setup--configuration)
- [SQL Implementation](#sql-implementation)
- [Power BI Analytics](#power-bi-analytics)
- [Key Insights](#key-insights)
- [Challenges & Solutions](#challenges--solutions)
- [Future Enhancements](#future-enhancements)

---

## Project Overview

### Objective

Transform raw healthcare admission data into actionable clinical and operational insights, focusing on:
- Hospital utilization patterns
- Disease severity distribution
- 30-day readmission analytics
- Geographic and demographic trends

### Dataset Characteristics

- **Total Records:** 25,000 admissions
- **Unique Patients:** 24,636
- **Time Period:** 2010-2020
- **Key Attributes:** Patient demographics, admission/discharge dates, diagnosis, severity, lab procedures, medications, readmission indicators

---

## Architecture

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐      ┌──────────────┐
│   Raw Data  │ ───> │  AWS Glue    │ ───> │   Athena    │ ───> │   Power BI   │
│   (S3)      │      │  (Crawler)   │      │  (SQL Views)│      │  (Dashboard) │
└─────────────┘      └──────────────┘      └─────────────┘      └──────────────┘
                            │                      │
                            ▼                      ▼
                     ┌──────────────┐      ┌─────────────┐
                     │ Glue Catalog │      │  ODBC DSN   │
                     │  (Metadata)  │      │ (Connector) │
                     └──────────────┘      └─────────────┘
```

### Component Roles

| Component | Purpose |
|-----------|---------|
| **Amazon S3** | Raw data storage (CSV format) |
| **AWS Glue Crawler** | Schema inference & metadata cataloging |
| **AWS Glue Data Catalog** | Centralized metadata repository |
| **Amazon Athena** | Serverless SQL query engine |
| **Athena ODBC Driver** | Power BI connectivity bridge |
| **Power BI** | Interactive dashboards & DAX analytics |

---

## Data Pipeline

### Stage 1: Data Ingestion

```
S3 Bucket Structure:
└── mediflow-healthcare-data/
    └── raw/
        └── hospital_admissions.csv
```

**Raw Data Format:**
- Dates stored as strings (DD-MM-YYYY)
- No schema enforcement at S3 level
- Patient-level admission records with clinical attributes

### Stage 2: Schema Discovery (AWS Glue)

**Glue Crawler Configuration:**
```
Crawler Name: mediflow-healthcare-crawler
Data Source: s3://mediflow-healthcare-data/raw/
Database: mediflow_database
Table Prefix: raw_data_
Schedule: On-demand
```

**IAM Permissions Required:**
- `AWSGlueServiceRole`
- `AmazonS3FullAccess`
- `AmazonAthenaFullAccess`

### Stage 3: Data Transformation (Athena SQL)

Four analytical views created to support different analysis needs:

1. **Core Power BI View** - Basic admission analytics
2. **Readmission View** - 30-day readmission logic
3. **Clinical Analysis View** - Extended clinical attributes
4. **Validation Queries** - Data quality checks

---

## SQL Implementation

### 1. Core Power BI View

```sql
CREATE OR REPLACE VIEW mediflow_powerbi_view AS
SELECT
    patient_id,
    age,
    gender,
    CAST(date_parse(admission_date, '%d-%m-%Y') AS DATE) AS admission_date,
    CAST(date_parse(discharge_date, '%d-%m-%Y') AS DATE) AS discharge_date,
    length_of_stay,
    primary_diagnosis,
    severity_level,
    city
FROM raw_data_mediflow_;
```

**Purpose:** Date conversion and basic dimensional attributes for Power BI

---

### 2. Readmission Analytical View

```sql
CREATE OR REPLACE VIEW mediflow_readmission_view AS
WITH ordered_admissions AS (
    SELECT
        patient_id,
        city,
        primary_diagnosis,
        severity_level,
        CAST(date_parse(admission_date, '%d-%m-%Y') AS DATE) AS admission_date,
        CAST(date_parse(discharge_date, '%d-%m-%Y') AS DATE) AS discharge_date,
        length_of_stay,
        
        LEAD(
            CAST(date_parse(admission_date, '%d-%m-%Y') AS DATE)
        ) OVER (
            PARTITION BY patient_id
            ORDER BY CAST(date_parse(admission_date, '%d-%m-%Y') AS DATE)
        ) AS next_admission_date
        
    FROM raw_data_mediflow_
)

SELECT
    patient_id,
    city,
    primary_diagnosis,
    severity_level,
    admission_date,
    discharge_date,
    length_of_stay,
    
    CASE
        WHEN next_admission_date IS NOT NULL
             AND date_diff('day', discharge_date, next_admission_date) <= 30
        THEN 1
        ELSE 0
    END AS readmitted_within_30_days,
    
    date_diff('day', discharge_date, next_admission_date) AS days_to_readmission
    
FROM ordered_admissions;
```

**Key Features:**
- Window function (`LEAD`) to capture next admission per patient
- 30-day readmission flag calculation
- Days-to-readmission metric for temporal analysis

---

### 3. Clinical Analysis View (Extended)
Full view includes all clinical attributes:

```sql
CREATE OR REPLACE VIEW mediflow_clinical_analysis_view AS
WITH ordered_admissions AS (
    SELECT
        patient_id,
        CAST(date_parse(admission_date, '%d-%m-%Y') AS DATE) AS admission_date,
        CAST(date_parse(discharge_date, '%d-%m-%Y') AS DATE) AS discharge_date,

        length_of_stay,
        primary_diagnosis,
        severity_level,
        lab_procedures,
        procedures,
        medications,
        outpatient,
        inpatient,
        emergency,
        medical_specialty,
        diag_1,
        diag_2,
        diag_3,
        glucose_test,
        a1c_test,
        change,
        diabetes_med,

        LEAD(
            CAST(date_parse(admission_date, '%d-%m-%Y') AS DATE)
        ) OVER (
            PARTITION BY patient_id
            ORDER BY CAST(date_parse(admission_date, '%d-%m-%Y') AS DATE)
        ) AS next_admission_date

    FROM raw_data_mediflow_
)

SELECT
    patient_id,
    admission_date,
    discharge_date,
    length_of_stay,
    primary_diagnosis,
    severity_level,
    lab_procedures,
    procedures,
    medications,
    outpatient,
    inpatient,
    emergency,
    medical_specialty,
    diag_1,
    diag_2,
    diag_3,
    glucose_test,
    a1c_test,
    change,
    diabetes_med,

    CASE
        WHEN next_admission_date IS NOT NULL
             AND date_diff('day', discharge_date, next_admission_date) <= 30
        THEN 1
        ELSE 0
    END AS readmitted_within_30_days,

    date_diff('day', discharge_date, next_admission_date) AS days_to_readmission

FROM ordered_admissions;
```
---
### City-wise analysis table:
```sql 
CREATE OR REPLACE VIEW mediflow_city_analysis_view AS
SELECT
    city, COUNT(DISTINCT patient_id) AS total_admissions,
    AVG(length_of_stay) AS avg_length_of_stay,
    SUM(CASE WHEN readmitted_within_30_days = 1 THEN 1 ELSE 0 END) AS readmissions_30_days,
    CAST(SUM(CASE WHEN readmitted_within_30_days = 1 THEN 1 ELSE 0 END) AS DOUBLE) / COUNT(DISTINCT patient_id) AS readmission_rate,

    AVG(CASE WHEN readmitted_within_30_days = 1 THEN days_to_readmission ELSE NULL END) AS avg_days_to_readmission,
    COUNT(CASE WHEN severity_level = 'Severe' THEN 1 END) AS severe_case_count

FROM raw_mediflow_data
GROUP BY city;
```

---

### 4. Validation Queries

**Record Count Validation:**
```sql
SELECT
    COUNT(*) AS rows,
    COUNT(DISTINCT patient_id) AS patients
FROM mediflow_clinical_analysis_view;
```

**Expected Output:** 25,000 rows, 24,636 unique patients

**Date Range Validation:**
```sql
SELECT
    MIN(admission_date) AS min_date,
    MAX(admission_date) AS max_date
FROM mediflow_clinical_analysis_view;
```

**Expected Output:** 2010-01-XX to 2020-12-XX

---

## Technologies Used

### Cloud Infrastructure
- **Amazon S3** - Object storage
- **AWS Glue** - ETL and data catalog
- **Amazon Athena** - Serverless SQL engine
- **IAM** - Access management

### Data Connectivity
- **Simba Athena ODBC Driver (64-bit)** - Power BI connection
- **AWS Region:** us-east-1
- **Authentication:** IAM Access Keys

### Analytics & Visualization
- **Power BI Desktop** - Dashboard development
- **DAX** - Measure creation
- **Power Query** - Data modeling

---

## Setup & Configuration

### Prerequisites

- AWS account with appropriate permissions
- Power BI Desktop installed
- Athena ODBC driver (64-bit Simba)
- IAM user with programmatic access

### Step-by-Step Setup

#### 1. AWS S3 Configuration

```bash
# Create S3 bucket
aws s3 mb s3://mediflow-healthcare-data

# Upload raw data
aws s3 cp hospital_admissions.csv s3://mediflow-healthcare-data/raw/
```

#### 2. IAM Setup

Create IAM user with policies:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "glue:*",
        "athena:*"
      ],
      "Resource": "*"
    }
  ]
}
```

Generate Access Keys:
- Navigate to IAM → Users → Security Credentials
- Create Access Key (store securely)

#### 3. AWS Glue Crawler

1. Create database: `mediflow_database`
2. Create crawler pointing to S3 bucket
3. Run crawler to generate table: `raw_data_mediflow_`

#### 4. Athena Views

Execute SQL scripts in Athena console:
1. Core Power BI View
2. Readmission View
3. Clinical Analysis View
4. Run validation queries

#### 5. ODBC Connection

**Configure DSN:**
```
Data Source Name: mediflow_athena
AWS Region: us-east-1
Database: mediflow_database
Workgroup: primary
Authentication: IAM Credentials
Access Key ID: [Your Access Key]
Secret Access Key: [Your Secret Key]
```

**Test Connection** before proceeding to Power BI.

#### 6. Power BI Connection

1. Get Data → ODBC → Select DSN
2. Load views: `mediflow_powerbi_view`, `mediflow_readmission_view`
3. Create relationships and date dimension
4. Build DAX measures
5. Design dashboard

---

## Power BI Analytics

### DAX Measures

```dax
// Total Admissions
Total Admissions = COUNTROWS(mediflow_powerbi_view)

// Severe Cases
Severe Cases = 
CALCULATE(
    COUNTROWS(mediflow_powerbi_view),
    mediflow_powerbi_view[severity_level] = "Severe"
)

// Severe Case Percentage
Severe Case % = 
DIVIDE(
    [Severe Cases],
    [Total Admissions],
    0
) * 100

// 30-Day Readmission Rate
30-Day Readmission Rate = 
VAR TotalReadmissions = 
    CALCULATE(
        COUNTROWS(mediflow_readmission_view),
        mediflow_readmission_view[readmitted_within_30_days] = 1
    )
VAR TotalAdmissions = COUNTROWS(mediflow_readmission_view)
RETURN
    DIVIDE(TotalReadmissions, TotalAdmissions, 0) * 100

// Average Length of Stay
Avg LOS = AVERAGE(mediflow_powerbi_view[length_of_stay])

// Days to Readmission (for readmitted cases only)
Avg Days to Readmission = 
CALCULATE(
    AVERAGE(mediflow_readmission_view[days_to_readmission]),
    mediflow_readmission_view[readmitted_within_30_days] = 1
)
```

### Dashboard Components

**KPI Cards:**
- Total Admissions: 25,000
- Avg Length of Stay: 5.52 days
- Severe Case %: 33%
- 30-Day Readmission Rate: 1.66%

**Time-Series Charts:**
- Admissions trend by year
- Severity percentage over time
- Readmission rate trend

**Geographic Analysis:**
- Admissions by city
- Inpatient vs outpatient by city
- Severity distribution by city

**Clinical Insights:**
- Admissions by diagnosis
- Severity % by diagnosis
- Length of stay by diagnosis
- 30-day readmission rate by diagnosis

**Demographic Breakdown:**
- Age group distribution
- Gender-based severity analysis

**Scatter Plots:**
- Severity % vs Length of Stay (by diagnosis)
- Clinical complexity vs outcomes

---

## Key Insights

### Overall Utilization

- **Stable Admission Volume:** 25,000 admissions with minimal year-over-year fluctuation
- **Moderate Length of Stay:** 5.52 days average, indicating efficient patient flow
- **High Acuity Population:** 33% severe case rate suggests tertiary care facility

### Severity Patterns

- **Stable Severity Over Time:** No systemic escalation in disease acuity
- **Diagnosis-Specific Severity:** Kidney disease and diabetes show highest severe case percentages
- **Consistent Case Mix:** Severity distribution remains stable across years

### Readmission Analysis

- **Low 30-Day Readmission Rate:** 1.66% indicates effective discharge planning
- **Total Readmissions:** 362 cases, with only 6 within 30 days
- **Declining Trend:** Total readmissions decrease over time
- **High-Risk Diagnoses:** Kidney disease and heart failure show elevated readmission rates

### Geographic Insights

- **Uniform Distribution:** Similar admission volumes across cities
- **Consistent Severity:** No geographic concentration of severe cases
- **Inpatient-Heavy Model:** All cities show higher inpatient than outpatient utilization

### Clinical Priorities

**High-Impact Diagnoses** (for intervention focus):
1. Kidney Disease - Highest readmission rate
2. Heart Failure - High severity + readmission risk
3. Diabetes - High severe case percentage

---

## Challenges & Solutions

### Challenge 1: Date Format Incompatibility

**Problem:** Dates stored as VARCHAR in DD-MM-YYYY format caused CAST errors

**Solution:** 
```sql
CAST(date_parse(admission_date, '%d-%m-%Y') AS DATE)
```

### Challenge 2: IAM Authentication for ODBC

**Problem:** AWS console credentials cannot be used for ODBC connections

**Solution:** Generated IAM Access Keys for programmatic access

### Challenge 3: Readmission Logic Implementation

**Problem:** No explicit readmission flag in raw data

**Solution:** Implemented window function with LEAD to calculate next admission date per patient

### Challenge 4: Power BI Direct Query Limitations

**Problem:** Complex DAX calculations slow with direct query

**Solution:** Pre-calculated readmission flags in Athena views, imported data mode

---

## Future Enhancements

### Technical Improvements

- [ ] Implement AWS Lambda for automated crawler triggering
- [ ] Add AWS Glue ETL jobs for complex transformations
- [ ] Set up Amazon QuickSight as alternative BI tool
- [ ] Create CloudWatch dashboards for pipeline monitoring
- [ ] Implement data versioning in S3 with lifecycle policies

### Analytical Extensions

- [ ] Predictive readmission risk modeling (ML)
- [ ] Cost analysis per diagnosis and severity
- [ ] Provider performance benchmarking
- [ ] Real-time alerting for readmission spikes
- [ ] Patient pathway analysis (sequence mining)

### Dashboard Enhancements

- [ ] Drill-through to patient-level details
- [ ] What-if analysis for capacity planning
- [ ] Mobile-optimized report views
- [ ] Automated report distribution (Power BI Service)
- [ ] Row-level security by city/department

---

## Project Structure

```
mediflow-healthcare-analytics/
├── sql/
│   ├── athena_views/
│   │   ├── core_powerbi_view.sql
│   │   ├── readmission_view.sql
│   │   └── clinical_analysis_view.sql
│   └── validation_queries.sql
├── power_bi/
│   ├── mediflow_dashboard.pbix
│   └── dax_measures.txt
├── docs/
│   ├── architecture_diagram.png
│   ├── data_dictionary.md
│   └── setup_guide.md
├── config/
│   └── odbc_dsn_template.txt
└── README.md
```

---

## Contributors

**Project Lead:** [Sourav Mondal]  
---

## License

This project is licensed under the MIT License - see LICENSE file for details.

---

## Acknowledgments

- AWS Documentation for Glue and Athena best practices
- Power BI Community for DAX optimization techniques
- Healthcare Analytics frameworks for clinical metric definitions

---

## Contact

For questions or collaboration opportunities:
- **Email:** your.email@example.com
- **LinkedIn:** [Your LinkedIn Profile]
- **Portfolio:** [Your Portfolio Website]

---

**Last Updated:** January 2026
