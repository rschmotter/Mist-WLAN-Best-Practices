# Scalability Evaluation
## Mist WLAN Best Practices Script at 2,000 Sites / 100,000 Clients

---

## Summary

The script is designed to operate within Mist's published API rate limit of 2,000 calls/hour. At large scale (2,000 sites, ~100,000 clients) the script remains functional but requires careful management of API call budgets. This document provides a full analysis and recommended optimizations.

---

## API Call Budget Estimate – 2,000 Sites

| Operation | Calls | Notes |
|---|---|---|
| Auth (`/self`) | 1 | One-time |
| List sites | ~20 | Paginated at 100/page → 20 pages |
| List WLAN templates | 1–5 | Usually few templates per org |
| WLAN template details | 1 per template | e.g., 10 templates = 10 calls |
| Site WLANs | 2,000 | 1 call per site |
| Site client stats | 2,000 | 1 call per site |
| SLE per site | 2,000 | 1 call per site |
| **Total audit** | **~6,031** | Exceeds 1-hour window |
| Remediation (PUT per WLAN) | up to N×templates | Usually small |

**Conclusion:** A full single-pass audit of 2,000 sites requires approximately 6,000+ API calls — exceeding the 2,000/hour limit by ~3×. The script handles this automatically by sleeping when the limit is approached and resuming in the next window.

**Estimated wall-clock time for 2,000 sites:**
- ~3 hours total (3 × 1-hour API windows)
- At 1 API call per second average: ~6,000 seconds = ~100 minutes of active API time
- With rate-limit sleep intervals: ~180 minutes (3 hours)

---

## Recommended Optimizations for Large Orgs

### 1. Skip Client Stats for Large Orgs (flag: `--skip-clients`)
Client count queries (`/stats/clients`) are the most expensive — one call per site. For 2,000 sites this is 2,000 calls just for counts. Consider using the org-level endpoint for a single aggregate count:
```
GET /api/v1/orgs/{org_id}/stats/clients?limit=1
```
This returns a `total` field in most API versions, avoiding per-site calls.

**Add to script:** Detect when site count > 200 and automatically switch to org-level aggregate.

### 2. Skip SLE for Initial Audit Pass
SLE queries add another 2,000 calls. For a pure best-practices config audit, SLE data is supplementary. Skip SLE on the first pass; collect it only when changes are being applied.

**Add to script:** `--no-sle` flag to skip SLE collection.

### 3. Parallel API Requests (threading)
The current script is sequential (one request at a time). Using `concurrent.futures.ThreadPoolExecutor` with 5–10 concurrent workers could reduce wall-clock time by 5–10×, while staying within rate limits by throttling total calls/second.

**Example:**
```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def fetch_site_data(api, site):
    return {
        "id": site["id"],
        "wlans": get_site_wlans(api, site["id"]),
        "clients": get_site_client_count(api, site["id"]),
    }

with ThreadPoolExecutor(max_workers=5) as ex:
    futures = {ex.submit(fetch_site_data, api, s): s for s in sites}
    for f in as_completed(futures):
        results.append(f.result())
```

**Caution:** Rate-limit tracking must be thread-safe (use `threading.Lock`).

### 4. Incremental / Delta Mode
Store the previous run's WLAN template configs locally (JSON snapshot). On subsequent runs, only check templates that have changed. For an org with 10 templates and 2,000 sites, template checks are typically < 50 calls and the expensive per-site calls can be batched only when a template change is detected.

### 5. Site Filtering
For targeted remediation, allow the user to specify a subset of sites:
```bash
python3 mist_wlan_best_practices.py --sites "Site-A,Site-B,HQ"
```
This limits per-site API calls to only the specified sites.

---

## Memory Footprint

| Data | Size estimate (2,000 sites) |
|---|---|
| Site list JSON | ~500 KB |
| WLAN template details | ~50 KB |
| Site WLAN lists | ~2–4 MB |
| Client stats (counts only) | ~100 KB |
| SLE data | ~200 KB |
| **Total in-memory** | **~5 MB** |

Memory usage is well within normal limits for any modern system. Python objects have overhead; actual RAM usage will be ~20–50 MB including interpreter overhead.

---

## Throughput at 100,000 Clients

The script does **not** retrieve full client details for each client — it only retrieves the **count** per site using the stats endpoint. This keeps the client-related API calls to 2,000 (one per site) or 1 (org-level aggregate) regardless of client count.

If full per-client data is ever needed:
- 100,000 clients at 100/page = 1,000 API calls just for client list
- Response payload ≈ 50–100 MB of JSON
- Recommended: Always use aggregate/count endpoints unless per-client detail is required

---

## Recommended Script Enhancements for Production Scale

| Enhancement | Priority | Est. Development |
|---|---|---|
| Org-level client count fallback (>200 sites) | High | 1 hour |
| `--no-sle` skip flag | High | 30 min |
| Threaded site data collection (5 workers) | Medium | 2 hours |
| Site filter (`--sites`) | Medium | 1 hour |
| Incremental/delta mode (JSON snapshot) | Low | 4 hours |
| Resume on failure (checkpoint file) | Low | 3 hours |

---

## Current Script Behavior at Scale

The current script handles large orgs gracefully through:
- **Auto rate-limit sleep:** Detects when approaching 2,000 calls/hour and pauses automatically
- **Paginated GET (`get_all`):** Handles orgs with >100 sites without truncation
- **Progress bar:** Shows live progress so the operator knows the script is running
- **Elapsed time display:** Tracks how long the run has taken
- **Full logging:** Every API call is logged to debug file for post-run analysis
- **Retry on 429:** If Mist returns HTTP 429 the script reads `Retry-After` header and waits

The script is production-safe at any org size — it will simply take proportionally longer for larger orgs.
