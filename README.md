# SPC Real-Time Monitoring System

Quality data collection and analysis system for production lines.

---

> ⚠️ **NOTICE: Mock Data**
> 
> All data in this document (IP addresses, database names, credentials, file paths, etc.) are **examples only** for demonstration purposes. Replace with your actual configuration before use.
> 
> **Do not commit real credentials to version control.**

---

## Table of Contents

1. [Overview](#1-overview)
2. [System Architecture](#2-system-architecture)
3. [Database Design](#3-database-design)
4. [Data Flow](#4-data-flow)
5. [Installation](#5-installation)
6. [Configuration](#6-configuration)
7. [Usage](#7-usage)
8. [Visualization](#8-visualization)
9. [Traceability](#9-traceability)
10. [Adding New Production Line](#10-adding-new-production-line)

---

## 1. Overview

This system is part of quality improvement project. It makes inspection data more accessible and uses that data to analyze issues and find ways to improve production.

### What This System Does

- Collects inspection data from CMM machine (Excel files)
- Calculates statistical values (PPK, PP, UCL, LCL)
- Stores data in SQL Server database
- Displays real-time SPC charts on Grafana (shopfloor monitoring)
- Provides data for Power BI reports (weekly/monthly summary)
- Helps identify which measurement points have issues most
- Enables traceability to fix problems at pain points

### Who Uses This System

| User | Purpose |
|------|---------|
| QC Team | Monitor real-time SPC at shopfloor |
| Engineering | Analyze issues and improve process |
| Management | Review weekly/monthly quality reports |

### Data Information

| Item | Description |
|------|-------------|
| Data per item | ~450 measurement points |
| Measurement types | GD&T and XYZ coordinates |
| Data source | CMM machine output (Excel files) |
| 1 Excel file | = 1 product item |

---

## 2. System Architecture

#### Diagram: System Architecture
<!-- Add your system architecture diagram here -->

### How It Works

The system runs as a **Windows Service** on a PC at the shopfloor. It automatically collects Excel files from the CMM machine output folder.

**Process Flow:**

1. **CMM Machine** → Measures product → Outputs Excel file (1 file = 1 item)
2. **Windows Service** → Detects new Excel file → Reads measurement data
3. **Calculation** → Calculates PPK, PP, UCL, LCL using last 125 items
4. **SQL Database** → Stores all data in universal tables
5. **Grafana** → Displays real-time SPC charts at shopfloor
6. **Power BI** → Generates weekly/monthly summary reports

### Data Details

| Item | Description |
|------|-------------|
| Data per Excel | ~450 rows × 2 columns |
| Measurement types | GD&T (form/position) and XYZ (coordinates) |
| Calculation base | Last 125 items in database |
| Update frequency | Follows production cycle time |

---

## 3. Database Design

#### Diagram: Database Schema
<!-- Add your database schema diagram here -->

### Universal Tables

We use **5 universal tables** for ALL production lines. Each table has `production_line` column to separate data by line.

| Table | Purpose |
|-------|---------|
| `QC_Inspection_Header` | Item information (1 row = 1 item) |
| `QC_GDT_Data` | GD&T measurements and control limits |
| `QC_XYZ_Data` | XYZ coordinates and control limits |
| `QC_PPK_PP_Data` | PPK and PP calculation results |
| `QC_Specs` | Specification limits for each point |

### Key Columns

| Column | Description |
|--------|-------------|
| `production_line` | Line identifier (LINE_01, LINE_02, etc.) |
| `run_number` | Item ID for traceability |
| `measurement_code` | Measurement point code |
| `data_type` | LH, RH, LHUCL, LHLCL, RHUCL, RHLCL, PPK, PP |
| `value` | Measured or calculated value |
| `log_date` | When data was recorded |

### Why Universal Design?

- One set of tables for ALL lines (not separate tables per line)
- Easy to compare data across lines
- Add new line = just add data with new `production_line` value
- No need to create new tables for new line

---

## 4. Data Flow

#### Diagram: Data Flow
<!-- Add your data flow diagram here -->

### Step-by-Step Flow

```
CMM Machine
    ↓
Excel File (1 file = 1 product)
    ↓
Windows Service (Python script)
    ↓
├── Read GD&T data
├── Read XYZ data
├── Calculate PPK, PP (using last 125 items)
├── Calculate UCL, LCL (control limits)
    ↓
SQL Server Database
    ↓
├── Grafana → Real-time SPC monitoring (shopfloor)
└── Power BI → Weekly/Monthly reports (management)
```

### Output Usage

| Output | Purpose | User |
|--------|---------|------|
| **Grafana Dashboard** | Real-time SPC monitoring at shopfloor | QC Team |
| **Power BI Report** | Weekly/monthly summary - which points have most issues | Engineering, Management |
| **Traceability Query** | Track back to specific item to find pain point | Engineering |

### Calculation Method

| Value | Formula | Base Data |
|-------|---------|-----------|
| UCL | Mean + 3σ | Last 125 items |
| LCL | Mean - 3σ | Last 125 items |
| PPK | min(PPU, PPL) | Last 125 items |
| PP | (USL - LSL) / 6σ | Last 125 items |

---

## 5. Installation

### Windows Service Setup

The system runs as a Windows Service on a PC at shopfloor. We have a program to install and upgrade the service easily.

#### Diagram: Installation Process
<!-- Add your installation diagram here -->

### Installation Steps

1. Copy program folder to shopfloor PC
2. Run installer program (as Administrator)
3. Configure `.env` file with database connection
4. Start the service

### Service Management

| Action | How |
|--------|-----|
| Install service | Run installer program |
| Upgrade service | Run upgrade program |
| Start service | Windows Services → Start |
| Stop service | Windows Services → Stop |
| Check status | Windows Services → View status |

### Requirements

- Windows 10/11 or Windows Server
- Python 3.8+ (bundled in installer)
- Network access to SQL Server
- Access to CMM output folder

---

## 6. Configuration

### Environment File (.env)

All configuration is in `.env` file. This keeps sensitive information separate from code.

```ini
# ============================================
# EXAMPLE CONFIGURATION - REPLACE WITH ACTUAL
# ============================================

# Database Connection
DB_SERVER=xxx.xxx.xxx.xxx
DB_NAME=your_database_name
DB_USER=your_username
DB_PASS=your_password

# Production Line
PRODUCTION_LINE=LINE_01

# File Paths
INPUT_DIRECTORY=C:\path\to\cmm\output
OUTPUT_DIRECTORY=C:\path\to\reports

# Processing Settings
PROCESS_INTERVAL=120
```

### Configuration Details

| Setting | Description | Example |
|---------|-------------|---------|
| `DB_SERVER` | SQL Server IP address | xxx.xxx.xxx.xxx |
| `DB_NAME` | Database name | QC_Database |
| `DB_USER` | Database username | (your username) |
| `DB_PASS` | Database password | (your password) |
| `PRODUCTION_LINE` | Line identifier | LINE_01 |
| `INPUT_DIRECTORY` | CMM output folder path | C:\CMM_Output |
| `OUTPUT_DIRECTORY` | Report output folder | C:\QC_Reports |
| `PROCESS_INTERVAL` | Seconds between checks | 120 |

### Security Note

- `.env` file contains passwords - **do not commit to Git**
- Each shopfloor PC has its own `.env` file
- Only authorized personnel should have access to modify

---

## 7. Usage

### Automatic Operation

Once installed, the service runs automatically:

1. **Starts with Windows** - Service starts when PC boots
2. **Runs on interval** - Follows production cycle time
3. **Detects new files** - Checks CMM output folder for new Excel files
4. **Processes automatically** - No manual action needed

### Processing Cycle

```
Every cycle:
    ↓
Check for new Excel files
    ↓
If new file found:
    ├── Read measurement data
    ├── Query last 125 items from database
    ├── Calculate PPK, PP, UCL, LCL
    ├── Insert data to database
    ├── Generate Excel summary report
    └── Mark file as processed
    ↓
Wait for next cycle
    ↓
Repeat
```

### Manual Operations

| Task | How |
|------|-----|
| Check if running | Windows Services → Find service → Check status |
| View logs | Check log file in program folder |
| Reprocess file | Delete from processed list and restart service |
| Force refresh | Restart service |

---

## 8. Visualization

### Grafana - Real-Time SPC Monitoring

Grafana displays real-time SPC charts at shopfloor for production monitoring.

#### Diagram: Grafana Dashboard
<!-- Add your Grafana dashboard screenshot here -->

### Dashboard Features

| Feature | Description |
|---------|-------------|
| Real-time update | Data shows as soon as it's in database |
| Control charts | Shows value with UCL/LCL lines |
| PPK trend | Shows PPK value over time |
| Status indicator | Green/Yellow/Red based on limits |
| Line filter | Select which production line to view |
| Point filter | Select which measurement point to view |

### Grafana Variables

| Variable | Query |
|----------|-------|
| `$line` | `SELECT DISTINCT production_line FROM QC_Inspection_Header` |
| `$point` | `SELECT DISTINCT measurement_code FROM QC_GDT_Data WHERE production_line = '$line'` |

### Grafana Query Examples

**Measurement value with control limits:**
```sql
SELECT 
    log_date AS time,
    value,
    data_type
FROM QC_GDT_Data
WHERE production_line = '$line'
  AND measurement_code = '$point'
  AND data_type IN ('LH', 'LHUCL', 'LHLCL')
ORDER BY log_date
```

**PPK trend:**
```sql
SELECT 
    log_date AS time,
    value AS ppk
FROM QC_PPK_PP_Data
WHERE production_line = '$line'
  AND measurement_code = '$point'
  AND data_type = 'LH PPK'
ORDER BY log_date
```

---

### Power BI - Weekly/Monthly Reports

Power BI generates summary reports for management to review quality trends.

#### Diagram: Power BI Report
<!-- Add your Power BI report screenshot here -->

### Report Contents

| Report | Content | Frequency |
|--------|---------|-----------|
| Weekly Summary | Top problem points, trend analysis | Weekly |
| Monthly Summary | Overall quality metrics, improvement tracking | Monthly |
| Issue Report | Points that exceed limits, action required | As needed |

### Using Reports for Improvement

1. **Review report** → Find points with lowest PPK
2. **Use traceability** → Find which items had problems
3. **Analyze pattern** → Time-based? Shift-based? Model-based?
4. **Fix at pain point** → Adjust machine, tooling, or process
5. **Monitor improvement** → Check if PPK improves in next report

---

## 9. Traceability

Every item has its own ID for complete traceability. This ID is stored in database as `run_number`.

#### Diagram: Traceability Flow
<!-- Add your traceability diagram here -->

### How Traceability Works

```
Find issue in Grafana/Power BI
    ↓
Identify measurement point with problem
    ↓
Query database with measurement_code
    ↓
Get run_number (Item ID) of affected items
    ↓
Track back to:
    ├── Original Excel file
    ├── Production date/time
    ├── Machine settings
    └── Other related data
    ↓
Fix at pain point
```

### Traceability Queries

**Find items with problem at specific point:**
```sql
SELECT run_number, log_date, value
FROM QC_GDT_Data
WHERE production_line = 'LINE_01'
  AND measurement_code = 'POINT_001'
  AND value > 1.0  -- Example: over limit
ORDER BY log_date DESC
```

**Get all data for specific item:**
```sql
SELECT *
FROM QC_GDT_Data
WHERE run_number = 'ITEM_ID_HERE'
```

### Benefits of Traceability

| Benefit | Description |
|---------|-------------|
| **Root Cause Analysis** | Find which items have problems |
| **Batch Tracking** | Identify all items from same period |
| **Customer Response** | Quick answer when customer asks about specific item |
| **Process Improvement** | Find pattern in problem items |

---

## 10. Adding New Production Line

Each production line has its own code because measurement data and points are different. When we have a new line, we create new code but reference from existing line.

### Why Each Line Has Own Code

| Reason | Description |
|--------|-------------|
| Different points | Each line measures different points |
| Different specs | Specification limits vary by product |
| Different Excel format | CMM output format may differ |

### Steps to Add New Line

1. **Copy code from reference line**
2. **Update header definitions** (measurement point names)
3. **Update specification limits** (min/max values)
4. **Update data map** (Excel sheet/column settings)
5. **Create configuration** (.env file)
6. **Install service** on shopfloor PC
7. **Test and verify** data appears correctly

### Database - No Changes Needed!

The universal database design means:
- ✅ No new tables needed
- ✅ Just add data with new `production_line` value
- ✅ Grafana/Power BI queries work automatically

### Checklist for New Line

- [ ] Copy code from reference line
- [ ] Update header definitions
- [ ] Update specification limits
- [ ] Update data map settings
- [ ] Create .env configuration
- [ ] Test with sample Excel file
- [ ] Install service on shopfloor PC
- [ ] Verify data appears in Grafana
- [ ] Add to Power BI reports

