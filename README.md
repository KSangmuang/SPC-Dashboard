# SPC Real-Time Monitoring System

Quality data collection and analysis system for chassis frame production.

---

## 1. Overview

This system is part of the main project to improve production quality. It makes inspection data more accessible and uses that data to analyze issues and find ways to improve production.

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

### Product Information

- **Product**: Chassis Frame
- **Data per item**: ~450 measurement points
- **Measurement types**: GD&T and XYZ coordinates
- **Data sources**: CMM machine output (Excel files)

---
## 2. System Architecture

#### Diagram: System Architecture
<!-- Add your system architecture diagram here -->

### How It Works

The system runs as a **Windows Service** on a PC at the shopfloor. It automatically collects Excel files from the CMM machine output folder.

**Process Flow:**

1. **CMM Machine** → Measures chassis frame → Outputs Excel file (1 file = 1 item)
2. **Windows Service** → Detects new Excel file → Reads measurement data
3. **Calculation** → Calculates PPK, PP, UCL, LCL using last 125 items (per customer agreement)
4. **SQL Database** → Stores all data in universal tables
5. **Grafana** → Displays real-time SPC charts at shopfloor
6. **Power BI** → Generates weekly/monthly summary reports

### Data Details

| Item | Description |
|------|-------------|
| Data per Excel | ~450 rows × 2 columns |
| Measurement types | GD&T (form/position) and XYZ (coordinates) |
| Calculation base | Last 125 items in database |
| Update frequency | Every 120 seconds (follows production CT) |

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
| `production_line` | Line identifier (P703L1, P703L2, U704L1, etc.) |
| `run_number` | Item KEY_ID (10 digits) for traceability |
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
Excel File (1 file = 1 chassis frame)
    ↓
Windows Service (Python script)
    ↓
├── Read GD&T data (~364 points)
├── Read XYZ data (~556 points)
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

**Note:** Using last 125 items for calculation is based on agreement with customer.

---
## 5. Installation

### Windows Service Setup

The system runs as a Windows Service on a PC at shopfloor. We have created a program to install and upgrade the service easily.

#### Diagram: Installation Process
<!-- Add your installation diagram here -->

### Installation Steps

1. Copy program folder to shopfloor PC
2. Run installer program (as Administrator)
3. Configure `.env` file with database connection
4. Start the service

### Service Management

| Action | Command |
|--------|---------|
| Install service | Run `install_service.exe` |
| Upgrade service | Run `upgrade_service.exe` |
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

All configuration is in `.env` file. This keeps sensitive information (passwords) separate from code.

```ini
# Database Connection
DB_SERVER=10.10.10.18
DB_NAME=WA_QC_Archive
DB_USER=your_username
DB_PASS=your_password

# Production Line
PRODUCTION_LINE=P703L1

# File Paths
INPUT_DIRECTORY=C:\CMM_Output\P703L1
OUTPUT_DIRECTORY=C:\QC_Reports\P703L1

# Processing Settings
PROCESS_INTERVAL=120
```

### Configuration Details

| Setting | Description | Example |
|---------|-------------|---------|
| `DB_SERVER` | SQL Server IP address | 10.10.10.18 |
| `DB_NAME` | Database name | WA_QC_Archive |
| `DB_USER` | Database username | scada |
| `DB_PASS` | Database password | (your password) |
| `PRODUCTION_LINE` | Line identifier | P703L1 |
| `INPUT_DIRECTORY` | CMM output folder path | C:\CMM_Output\P703L1 |
| `OUTPUT_DIRECTORY` | Report output folder | C:\QC_Reports\P703L1 |
| `PROCESS_INTERVAL` | Seconds between checks | 120 |

### Security Note

- `.env` file contains passwords - do not share or commit to Git
- Each shopfloor PC has its own `.env` file
- Only IT/Engineering should have access to modify

---

## 7. Usage

### Automatic Operation

Once installed, the service runs automatically:

1. **Starts with Windows** - Service starts when PC boots
2. **Runs every 120 seconds** - Follows production cycle time (CT)
3. **Detects new files** - Checks CMM output folder for new Excel files
4. **Processes automatically** - No manual action needed

### Processing Cycle

```
Every 120 seconds:
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
Wait 120 seconds
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

### Monitoring

- **Grafana**: Shows real-time data as soon as it's in database
- **Log file**: Records each file processed with timestamp
- **Excel report**: Generated daily summary in output folder

---
## 8. Visualization

### Grafana - Real-Time SPC Monitoring

Grafana displays real-time SPC charts at shopfloor. QC team uses this to monitor production quality.

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

Set up these variables for dropdown filters:

| Variable | Query |
|----------|-------|
| `$line` | `SELECT DISTINCT production_line FROM QC_Inspection_Header ORDER BY production_line` |
| `$point` | `SELECT DISTINCT measurement_code FROM QC_GDT_Data WHERE production_line = '$line' ORDER BY measurement_code` |
| `$data_type` | `LH, RH` |

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
  AND data_type IN ('$data_type', '${data_type}UCL', '${data_type}LCL')
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
  AND measurement_type = 'GDT'
  AND data_type = '$data_type PPK'
ORDER BY log_date
```

**Latest status for all points:**
```sql
SELECT 
    measurement_code,
    value,
    CASE 
        WHEN value < spec_min OR value > spec_max THEN 'NG'
        WHEN value < lcl OR value > ucl THEN 'Warning'
        ELSE 'OK'
    END AS status
FROM (
    SELECT TOP 1 WITH TIES *
    FROM QC_GDT_Data
    WHERE production_line = '$line'
    ORDER BY ROW_NUMBER() OVER (PARTITION BY measurement_code ORDER BY log_date DESC)
) latest
```

---

### Power BI - Weekly/Monthly Reports

Power BI generates summary reports for management to review quality trends.

#### Diagram: Power BI Report
<!-- Add your Power BI report screenshot here -->

### Report Contents

| Report | Content | Frequency |
|--------|---------|-----------|
| Weekly Summary | Top 10 problem points, trend analysis | Every Monday |
| Monthly Summary | Overall quality metrics, improvement tracking | First of month |
| Issue Report | Points that exceed limits, action required | As needed |

### Key Metrics in Reports

| Metric | Description | Target |
|--------|-------------|--------|
| PPK Average | Average PPK across all points | > 1.33 |
| Points below 1.33 | Count of points with PPK < 1.33 | 0 |
| Warning count | Items with values outside control limits | < 5% |
| NG count | Items with values outside spec limits | 0 |

### Power BI Queries

**Top 10 problem points (lowest PPK):**
```sql
SELECT TOP 10
    measurement_code,
    AVG(value) AS avg_ppk,
    COUNT(*) AS sample_count
FROM QC_PPK_PP_Data
WHERE production_line = 'P703L1'
  AND data_type = 'LH PPK'
  AND log_date >= DATEADD(week, -1, GETDATE())
GROUP BY measurement_code
ORDER BY avg_ppk ASC
```

**Weekly trend by point:**
```sql
SELECT 
    DATEPART(week, log_date) AS week_number,
    measurement_code,
    AVG(value) AS avg_ppk
FROM QC_PPK_PP_Data
WHERE production_line = 'P703L1'
  AND log_date >= DATEADD(month, -1, GETDATE())
GROUP BY DATEPART(week, log_date), measurement_code
ORDER BY week_number, measurement_code
```

**Points that need action (PPK < 1.0):**
```sql
SELECT 
    measurement_code,
    AVG(value) AS avg_ppk,
    MIN(value) AS min_ppk,
    MAX(value) AS max_ppk
FROM QC_PPK_PP_Data
WHERE production_line = 'P703L1'
  AND data_type LIKE '%PPK'
  AND log_date >= DATEADD(week, -1, GETDATE())
GROUP BY measurement_code
HAVING AVG(value) < 1.0
ORDER BY avg_ppk ASC
```

### Using Reports for Improvement

1. **Review weekly report** → Find points with lowest PPK
2. **Use traceability** → Find which items had problems
3. **Analyze pattern** → Is it time-based? Shift-based? Model-based?
4. **Fix at pain point** → Adjust machine, tooling, or process
5. **Monitor improvement** → Check if PPK improves in next report

---
## 9. Traceability

Every item (chassis frame) has its own KEY_ID for complete traceability. This ID is stored in database as `run_number`.

#### Diagram: Traceability Flow
<!-- Add your traceability diagram here -->

### KEY_ID Structure

The KEY_ID is a **10-digit code** that uniquely identifies each item.

| Position | Meaning | Example |
|----------|---------|---------|
| 1-4 | Line + Date code | P703 |
| 5-7 | Model code | L1X |
| 8-10 | Sequence number | 144 |

**Example:** `P703L1X144` = Line P703, Model L1X, Item #144

### How Traceability Works

```
Find issue in Grafana/Power BI
    ↓
Identify measurement point with problem
    ↓
Query database with measurement_code
    ↓
Get run_number (KEY_ID) of affected items
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
WHERE production_line = 'P703L1'
  AND measurement_code = 'MTG_1_211'
  AND value > 1.0  -- Over limit
ORDER BY log_date DESC
```

**Get all data for specific item:**
```sql
SELECT *
FROM QC_GDT_Data
WHERE run_number = 'P703L1X144'
```

**Find items in date range with issues:**
```sql
SELECT h.run_number, h.log_date, h.file_name
FROM QC_Inspection_Header h
WHERE h.production_line = 'P703L1'
  AND h.status_check = 'Warning'
  AND h.log_date BETWEEN '2025-12-01' AND '2025-12-04'
```

### Benefits of Traceability

| Benefit | Description |
|---------|-------------|
| **Root Cause Analysis** | Find which items have problems |
| **Batch Tracking** | Identify all items from same period |
| **Customer Response** | Quick answer when customer asks about specific item |
| **Process Improvement** | Find pattern in problem items |

### Data Retention

- All data is kept in database for traceability
- Archive policy: (define your retention period)
- Backup: (define your backup schedule)

---
## 10. Adding New Production Line

Each production line has its own code because measurement data and points are different. When we have a new line, we create new code but reference from existing line.

### Why Each Line Has Own Code

| Reason | Description |
|--------|-------------|
| Different points | Each line measures different points on the product |
| Different specs | Specification limits vary by product/line |
| Different Excel format | CMM output format may differ |
| Different calculation | Some lines may have special requirements |

### Steps to Add New Line

#### Step 1: Copy Reference Code

```bash
# Copy from existing line
cp -r src/P703L1/ src/NEW_LINE/
```

#### Step 2: Update Header Definitions

Edit `header_definitions.py` for new line:

```python
# Add headers for new line
NEW_LINE_headers = [
    "NEWLINE_MTG_1_211",
    "NEWLINE_MTG_1_212",
    # ... add all measurement points
]

NEW_LINE_XYZ_headers = [
    "NEWLINE_XYZ_N_001",
    "NEWLINE_XYZ_N_002",
    # ... add all XYZ points
]
```

#### Step 3: Update Specification Limits

Edit `spec_limits.py` for new line:

```python
# Add spec limits for new line
newline_lh_max = [1.0, 1.0, ...]  # Upper limits
newline_lh_min = [-1.0, -1.0, ...]  # Lower limits
newline_rh_max = [1.0, 1.0, ...]
newline_rh_min = [-1.0, -1.0, ...]
```

#### Step 4: Update Data Map

Edit main script for new Excel format:

```python
NEW_LINE_DATA_MAP = {
    "gdt_sheet": "GD&T Data",      # Sheet name in Excel
    "gdt_cols": "E,I,U",           # Columns to read
    "gdt_skiprows": 20,            # Rows to skip
    "gdt_nrows": 364,              # Number of rows
    "data_sheet": "Data",
    "data_cols": "N,AA",
    "data_skiprows": 12,
    "data_nrows": 556,
    # ... other settings
}
```

#### Step 5: Create Configuration

Create `.env` for new line:

```ini
PRODUCTION_LINE=NEW_LINE
INPUT_DIRECTORY=C:\CMM_Output\NEW_LINE
OUTPUT_DIRECTORY=C:\QC_Reports\NEW_LINE
```

#### Step 6: Install Service

Run installer on shopfloor PC for new line.

### Checklist for New Line

- [ ] Copy code from reference line
- [ ] Update header definitions (all measurement points)
- [ ] Update specification limits (min/max for each point)
- [ ] Update data map (Excel format settings)
- [ ] Create .env configuration
- [ ] Test with sample Excel file
- [ ] Install service on shopfloor PC
- [ ] Verify data appears in Grafana
- [ ] Add to Power BI reports

### Database - No Changes Needed!

The universal database design means:
- ✅ No new tables needed
- ✅ Just add data with `production_line = 'NEW_LINE'`
- ✅ Grafana queries work automatically (filter by line)

### Folder Structure for Multiple Lines

```
SPC-System/
├── src/
│   ├── P703L1/          # Line 1 code
│   │   ├── main.py
│   │   ├── headers.py
│   │   └── specs.py
│   ├── P703L2/          # Line 2 code
│   │   ├── main.py
│   │   ├── headers.py
│   │   └── specs.py
│   └── U704L1/          # Another line
│       ├── main.py
│       ├── headers.py
│       └── specs.py
├── database/            # Shared database scripts
├── docs/                # Shared documentation
└── config/              # Configuration templates
```

---
