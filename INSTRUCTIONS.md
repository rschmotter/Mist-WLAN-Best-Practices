# Run Instructions – Mist WLAN Best Practices Script

## Prerequisites

### Python
Requires Python 3.8 or later.

**Check your version:**
```bash
python3 --version
```

**Install Python (if needed):**
- macOS: `brew install python` or download from https://python.org
- Windows: Download from https://python.org (check "Add to PATH")
- Linux: `sudo apt install python3 python3-pip`

### Install Dependencies
From the project folder:
```bash
pip install -r requirements.txt
```

Or install packages individually:
```bash
pip install requests openpyxl
```

---

## Getting Your Mist API Token

### Read-Only Token (for audit)
1. Log into your Mist dashboard at https://manage.mist.com
2. Go to **Organization → Settings → API Token**
3. Click **Create Token**
4. Set **Privileges** to **Read Only**
5. Copy the token — you will not be able to view it again

### Read/Write (SuperUser) Token (for applying changes)
Same steps as above, but set **Privileges** to **Super User** or **Network Admin**.  
Keep this token secure — it can make changes to your wireless infrastructure.

### Getting Your Org ID
1. In the Mist dashboard, go to **Organization → Settings**
2. The Org ID is displayed at the top of the page (UUID format: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`)

---

## Running the Script

### Interactive Mode (recommended for first use)
```bash
python3 mist_wlan_best_practices.py
```

The script will prompt you for:
1. Mist cloud region
2. Org ID
3. Read-Only API token

It then runs through the full audit and presents options at each step.

### Automated Mode (for scheduled nightly runs)
Set environment variables, then run with `--auto`:

**Linux/macOS:**
```bash
export MIST_CLOUD=1
export MIST_ORG_ID=your-org-id-here
export MIST_TOKEN=your-readwrite-token-here
python3 mist_wlan_best_practices.py --auto
```

**Windows (PowerShell):**
```powershell
$env:MIST_CLOUD="1"
$env:MIST_ORG_ID="your-org-id-here"
$env:MIST_TOKEN="your-readwrite-token-here"
python mist_wlan_best_practices.py --auto
```

### Cloud Region Values for `MIST_CLOUD`
| Value | Cloud |
|---|---|
| `1` | Global 01 / US (api.mist.com) |
| `2` | Global 02 (api.gc1.mist.com) |
| `3` | Europe (api.eu.mist.com) |
| `4` | APAC AC2 (api.ac2.mist.com) |
| `5` | APAC AC5 (api.ac5.mist.com) |
| `6` | Canada (api.ca.mist.com) |

---

## Setting Up Midnight Automation

### Linux/macOS – cron
Edit your crontab:
```bash
crontab -e
```

Add this line (replace path and environment variables):
```
0 0 * * * MIST_CLOUD=1 MIST_ORG_ID=your-org-id MIST_TOKEN=your-rw-token /usr/bin/python3 /path/to/mist_wlan_best_practices.py --auto >> /path/to/logs/cron.log 2>&1
```

**Security tip:** Store your token in a protected env file instead of inline:
```bash
# /etc/mist_secrets  (chmod 600)
export MIST_TOKEN=your-token-here
```
```
0 0 * * * source /etc/mist_secrets && MIST_CLOUD=1 MIST_ORG_ID=your-org-id /usr/bin/python3 /path/to/mist_wlan_best_practices.py --auto
```

### Windows – Task Scheduler
1. Open **Task Scheduler** → **Create Basic Task**
2. Set trigger: **Daily** at **12:00 AM**
3. Set action: **Start a program**
   - Program: `python`
   - Arguments: `"C:\path\to\mist_wlan_best_practices.py" --auto`
   - Start in: `C:\path\to\project-folder`
4. In **Properties → Environment Variables**, add:
   - `MIST_CLOUD` = `1`
   - `MIST_ORG_ID` = your org ID
   - `MIST_TOKEN` = your RW token

---

## Interactive Script Walkthrough

When run interactively, the script follows this flow:

```
1. Select Mist cloud region
2. Enter Org ID
3. Enter Read-Only API token  (hidden input)
4. Loads sites, WLAN templates, site WLANs, client counts, SLE

5. Shows: Total wireless client count by site
6. Shows: Successful Connect SLE (24h) by site

7. Asks: View WLAN Best Practices reference guide?  [y/N]

8. Shows: Best practices status for each WLAN in each template
   (PASS / FAIL / N/A per best practice)

9. Asks: List each site and WLANs present?  [y/N]
   └── Asks: Delete any WLAN by site?  [y/N]
       └── Prompts for RW token, site name, SSID name
       └── WARNING shown before deletion
10. Asks: Show sites with duplicate WLAN names?  [y/N]

11. Asks: Enable failing WLAN Best Practices now?  [y/N]
    └── Prompts for SuperUser API token
    └── Confirms each change per WLAN individually
    └── Applies changes via PUT API
    └── Runs post-change client count and SLE verification

12. Asks: Schedule midnight automation?  [y/N]
    └── Displays cron/Task Scheduler instructions

13. Exports Excel report to project folder
14. Displays run summary (sites, templates, clients, changes, API calls, elapsed time)
```

---

## Output Files

All output files are saved to the project folder:

| File | Description |
|---|---|
| `logs/mist_bp_YYYYMMDD_HHMMSS.log` | Full session log |
| `logs/mist_bp_debug_YYYYMMDD_HHMMSS.log` | API debug trace |
| `mist_bp_report_YYYYMMDD_HHMMSS.xlsx` | Excel best-practices report |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Authentication failed` | Wrong token or cloud | Verify token in Mist portal; check cloud selection |
| `ERROR: 'requests' library not found` | Missing dependency | Run `pip install requests` |
| Excel not generated | Missing `openpyxl` | Run `pip install openpyxl` |
| 429 Too Many Requests | API rate limit | Script auto-retries; reduce concurrent usage |
| Sites return empty WLAN list | Token lacks org access | Use a token with org-level privileges |
| `PUT` returns error on remediation | Token is read-only | Use a SuperUser/RW token for changes |

---

## Important Notes

- **Changes to WLAN templates can briefly bounce AP radios**, causing a momentary wireless disruption. Schedule remediation during a maintenance window.
- **Data rates** (best practice #4) are managed at the RF Template level in Mist and cannot be changed via the WLAN API. The script will flag this and instruct you to update the RF Template manually in the Mist dashboard.
- **Duplicate SSIDs** (best practice #6) must be resolved manually — the script identifies and reports them but cannot automatically determine which WLAN should be removed.
