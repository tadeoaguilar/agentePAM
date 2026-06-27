# How to Run PAM_UUID_Agent.py

## Quick Start (3 Steps)

### Step 1: Open Command Prompt or PowerShell
Press `Win + R`, type `cmd` or `powershell`, and press Enter

### Step 2: Navigate to Project Directory
```bash
cd C:\Users\tadeo\gitlab\agentePAM
```

### Step 3: Run the Script
```bash
python PAM_UUID_Agent.py
```

---

## Detailed Instructions

### Prerequisites

1. **Python Installation**
   - Verify Python is installed by running:
   ```bash
   python --version
   ```
   - Should show Python 3.x.x
   - If not found, install from https://www.python.org/downloads/

2. **Required Libraries**
   - Install dependencies with:
   ```bash
   pip install pandas openpyxl xlsxwriter python-dateutil tqdm
   ```
   Or install all at once:
   ```bash
   pip install pandas openpyxl xlsxwriter python-dateutil tqdm -q
   ```

3. **Data Files**
   Ensure these files exist in the `files/` directory:
   - `AuxGastos.xlsx` - GL expense data
   - `Acreedor.xlsx` - Creditor master data
   - `Pagos/3400.xlsx`, `3401.xlsx`, ... `3415.xlsx` (11 FBL1N files)
   - `Compensacion.xlsx` - Compensation bridge data
   - `Auxintercentros1.xlsx`, `Auxintercentros2.xlsx` - Intercompany lookup data

### Running the Script

#### Option 1: From Command Prompt (Recommended)

```bash
# Navigate to project directory
cd C:\Users\tadeo\gitlab\agentePAM

# Run the script
python PAM_UUID_Agent.py
```

**Expected Output:**
```
Imports OK
Config OK  |  Pagos: files/Pagos/  |  Output: output/ReporteFinal_3400_202002.xlsx
Utilities defined
...
[Progress bars showing data loading]
...
[Matching strategy results]
...
Direct override: document 700071613 -> account 11123162
Saved: output/ReporteFinal_3400_202002.xlsx
```

#### Option 2: Using Python Directly

```bash
python -u C:\Users\tadeo\gitlab\agentePAM\PAM_UUID_Agent.py
```
(The `-u` flag shows unbuffered output in real-time)

#### Option 3: Schedule/Automate

Create a batch file `run_script.bat`:
```batch
@echo off
cd C:\Users\tadeo\gitlab\agentePAM
python PAM_UUID_Agent.py
pause
```

Then double-click `run_script.bat` to run it.

---

## Execution Flow

The script will execute in this order:

1. **Load configuration** - Set up paths and column mappings
2. **Load data files** - Read AuxGastos, Acreedor, FBL1N (Pagos), and Compensacion files
3. **Build lookups** - Create intercompany account lookup from Auxintercentros
4. **Matching strategies** (in order):
   - S1: UUID_DIRECTO - Exact UUID matching
   - S2: DOCUMENTO_CRUCE - Document number matching
   - S5: COMPENSACION_BRIDGE - Compensation bridge matching
   - S3: FUZZY_CON_ACREEDOR - Fuzzy matching with creditor
   - S4: FUZZY_SIN_ACREEDOR - Fuzzy matching without creditor
5. **Apply overrides**:
   - Intercompany lookup overrides (136,638 possible matches)
   - Direct override for document 700071613 → account 11123162
6. **Generate output** - Create ReporteFinal_3400_202002.xlsx
7. **Complete** - Show summary statistics

---

## Expected Runtime

**Typical execution time: 5-10 minutes**
- Data loading: ~2 minutes
- Fuzzy matching: ~3-5 minutes
- Output generation: ~1 minute

On faster machines: 3-5 minutes
On slower machines: 10-15 minutes

---

## Output Files

The script generates one main file:

**`output/ReporteFinal_3400_202002.xlsx`**

Contains 3 sheets:

1. **Resultado** (2,404 rows)
   - All matched GL-to-FBL1N records
   - Complete payment information
   - Matching strategy used
   - Confidence level

2. **Pendientes** (112 rows)
   - Unresolved GL records
   - Requires manual review

3. **Métricas** (15 metrics)
   - Matching success rates
   - Strategy breakdown
   - Resolution percentage

---

## Verification

After the script completes, verify the fix:

```bash
# Check that the output file was created
dir output\ReporteFinal_3400_202002.xlsx
```

Or open the file in Excel and search for document 700071613:
- Should show "Cuenta bancaria de pago" = **11123162** ✓
- NOT 11210050 ✗

---

## Troubleshooting

### Error: "can't open file 'PAM_UUID_Agent.py': [Errno 2]"
**Solution:** Make sure you're in the correct directory
```bash
cd C:\Users\tadeo\gitlab\agentePAM
dir PAM_UUID_Agent.py  # Should list the file
python PAM_UUID_Agent.py
```

### Error: "ModuleNotFoundError: No module named 'pandas'"
**Solution:** Install required packages
```bash
pip install pandas openpyxl xlsxwriter python-dateutil tqdm
```

### Error: "FileNotFoundError" for data files
**Solution:** Check that all required files exist in `files/` directory
```bash
dir files\
dir files\Pagos\
```

### Script runs very slowly
**Solution:** This is normal for the first run
- Fuzzy matching on 1,300+ records is computationally intensive
- Typical time: 5-10 minutes
- Subsequent runs are cached if data hasn't changed

### Unicode encoding errors on Windows
**Solution:** Already handled in the script
- Uses UTF-8 encoding with error replacement
- If issues persist, run in PowerShell instead of Command Prompt:
```powershell
python PAM_UUID_Agent.py
```

### Output file not created
**Solution:** Check console output for error messages
```bash
# Run script and save output to file for debugging
python PAM_UUID_Agent.py > debug_log.txt 2>&1

# View the log
type debug_log.txt | more
```

---

## Advanced Usage

### Run with Logging
```bash
python PAM_UUID_Agent.py > execution_log.txt 2>&1
```

### Run in Background (PowerShell)
```powershell
Start-Process python -ArgumentList "PAM_UUID_Agent.py" -NoNewWindow -Wait
```

### Schedule Daily Execution (Task Scheduler)
1. Open Task Scheduler
2. Create Basic Task
3. Name: "PAM_UUID_Agent Daily"
4. Trigger: Daily at desired time
5. Action: Start program = `python`, Arguments = `C:\Users\tadeo\gitlab\agentePAM\PAM_UUID_Agent.py`
6. Settings: Allow on-demand trigger

---

## Contact & Support

If you encounter issues:
1. Check the console output for error messages
2. Verify all data files exist and are readable
3. Ensure Python version is 3.7 or higher
4. Check that dependencies are properly installed
5. Review the debug_log.txt file if created

