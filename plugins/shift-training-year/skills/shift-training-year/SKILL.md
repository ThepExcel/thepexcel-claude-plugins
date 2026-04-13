---
name: shift-training-year
description: Shift years in training data files (CSV + XLSX) by +N years — both filenames and content. Use when updating training data for a new year, "เลื่อนปี", "shift year", "update year in training data", "เปลี่ยนปีใน data", or when preparing Power BI / Excel training files for next year's course.
---

# Shift Training Year

Shift year references in training data files (CSV, XLSX) by a specified amount. Designed for annual updates to course training data — handles filenames, CSV content (Order Numbers, dates), and XLSX year columns.

## When to Use

- Updating training data files so the years match the current course year
- User says "เลื่อนปี", "shift year", "update year in data/training files"
- Preparing Power BI, Excel, or data analytics training materials for a new year

## How It Works

Run the bundled script which handles everything:

```bash
python "<skill-path>/scripts/shift_year.py" "<folder-path>" [--shift N]
```

- `folder-path`: Directory containing the data files (CSV + XLSX)
- `--shift N`: How many years to shift (default: 1). Use negative values to shift backwards.

### What the Script Does

**CSV files:**
- Detects which year(s) appear in each file by scanning quoted strings
- Shifts years **only inside quoted strings** — this protects unquoted numeric values (StoreKey, CustomerKey, ProductKey, etc.) from accidental changes
- Handles Order Numbers (e.g., `"202401011CS952"` → `"202501011CS952"`) and dates in DD/MM/YYYY format (e.g., `"01/01/2024"` → `"01/01/2025"`)
- Renames files that have years in their names (e.g., `contoso-online-2024.csv` → `contoso-online-2025.csv`)

**XLSX files:**
- Shifts any numeric cell value that looks like a year (2000-2099 range) in columns named "Year" or similar
- Also scans all cells for standalone year values if no Year column is found
- Does not rename XLSX files (usually named generically like `TargetReport.xlsx`)

### Important Notes

- The script processes files **in reverse year order** (highest year first) to prevent double-shifting when multiple years exist in the same file
- Always creates a backup (`_backup/`) before modifying files — the backup is placed inside the target folder
- CSV encoding is preserved (UTF-8)

## Workflow

1. User provides the folder path containing training data
2. Confirm with user: which files to process and the shift amount
3. Run the script
4. Verify a sample of the output to make sure it looks correct
5. If user is satisfied, the backup folder can be deleted

## Example

```
User: เลื่อนปีใน training data ไป 1 ปี folder C:\...\Contoso

Step 1: List files in the folder
Step 2: Confirm with user
Step 3: Run script
  python "<skill-path>/scripts/shift_year.py" "C:\...\Contoso"
Step 4: Spot-check a few rows from each file
```

## Related Skills

- `/xlsx-thepexcel` — Read/edit XLSX files directly; use when year-shifted data needs further spreadsheet manipulation
