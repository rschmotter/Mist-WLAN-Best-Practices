# Juniper Mist WLAN Best Practices Automation

## Overview

`mist_wlan_best_practices.py` is an interactive Python script that connects to the Juniper Mist REST API to audit, report, and optionally remediate WLAN SSID configurations against established best practices for your Mist Org.

---

## Features

| Feature | Description |
|---|---|
| Multi-cloud support | Prompts for cloud region at startup (US, EU, APAC, CA) |
| Read-only audit mode | Checks all WLAN templates and site WLANs without making changes |
| Remediation mode | Applies best-practice fixes with SuperUser/RW token (with confirmation) |
| Client health snapshot | Reports wireless client counts per site before and after changes |
| SLE reporting | Retrieves Successful Connect SLE for past 24 h (and 1 h post-change) |
| Duplicate SSID detection | Flags sites with duplicate WLAN names |
| WLAN deletion | Guided deletion of a WLAN from a specific site (with warning) |
| Excel export | Three-tab report (By WLAN Template / By Site / By AP) |
| Full logging | Session log + debug log saved to `logs/` folder |
| Midnight automation | `--auto` flag for scheduled nightly runs via cron/Task Scheduler |
| API rate limiting | Stays under 2,000 calls/hour; auto-sleeps if limit is approached |
| Progress display | Live progress bar, API call counter, and elapsed time |

---

## WLAN Best Practices Checked

### 1. ARP Filtering (`arp_filter: true`)
The AP acts as an ARP proxy for known wireless clients, converting ARP broadcasts into directed unicast responses. This prevents ARP broadcast storms that degrade Wi-Fi performance in dense environments.  
**Mist API field:** `arp_filter` (boolean on WLAN object)

### 2. Multicast/Broadcast Filtering (`limit_bcast: true`)
Broadcast and multicast frames are sent at the lowest basic data rate, consuming disproportionate airtime on every client. Enabling this filter suppresses unnecessary Layer-2 floods, improving overall SSID throughput and latency.  
**Mist API field:** `limit_bcast` (boolean on WLAN object)

### 3. Allow IPv6 Clients (`allow_ipv6_ndp: true`)
IPv6 Neighbor Discovery Protocol (NDP) replaces ARP in IPv6 networks. If disabled, dual-stack and IPv6-only clients cannot perform address resolution. Modern operating systems prefer IPv6; disabling NDP causes silent connectivity failures.  
**Mist API field:** `allow_ipv6_ndp` (boolean on WLAN object)

### 4. Data Rates – No Legacy 802.11b (`data_rates`)
Legacy 802.11b rates (1, 2, 5.5, 11 Mbps) force APs to transmit beacons and management frames at the lowest supported rate, wasting airtime for all modern clients. Removing 802.11b rates from the supported set and setting the minimum rate to 6 Mbps or higher significantly improves network efficiency.  
**Note:** In Mist this is primarily controlled at the **RF Template** level (not WLAN level). The script checks for explicit WLAN-level `data_rates` overrides and flags RF Template–level settings for manual review.

### 5. 802.11r Fast Transition (Enterprise WPA2/WPA3 only)
802.11r pre-authenticates clients with neighboring APs before roaming, reducing handoff latency from >100 ms to <50 ms — critical for voice, video, and real-time applications. Mist uses Hybrid/Mixed 802.11r so legacy non-FT clients are unaffected.  
**Applies to:** SSIDs with auth type `eap`, `eap-reauth`, `dot1x_eap`, `dot1x_cert`  
**Mist API field:** `auth.disable_ft` — set to `false` to enable FT

### 6. No Duplicate WLAN Names per Site
Duplicate SSID names on the same site cause unexpected client behaviour, authentication loops, and complicate troubleshooting. Each SSID should be unique within a site unless intentionally split (Mist handles 2.4/5 GHz splitting automatically).  
**Resolution:** Must be corrected manually in the Mist UI or API.

---

## Architecture

```
mist_wlan_best_practices.py   ← Main script (single-file, self-contained)
requirements.txt              ← Python dependencies
README.md                     ← This documentation
INSTRUCTIONS.md               ← Step-by-step run guide
SCALABILITY.md                ← Load & scalability evaluation
logs/                         ← Auto-created; session + debug logs written here
```

---

## API Endpoints Used

| Purpose | Endpoint |
|---|---|
| Auth verification | `GET /api/v1/self` |
| List sites | `GET /api/v1/orgs/{org_id}/sites` |
| List WLAN templates | `GET /api/v1/orgs/{org_id}/wlantemplates` |
| WLAN template detail | `GET /api/v1/orgs/{org_id}/wlantemplates/{id}` |
| Site WLANs | `GET /api/v1/sites/{site_id}/wlans` |
| Site client stats | `GET /api/v1/sites/{site_id}/stats/clients` |
| Org client stats | `GET /api/v1/orgs/{org_id}/stats/clients` |
| SLE Successful Connect | `GET /api/v1/sites/{site_id}/sle/wireless/metric/successful-connect/summary` |
| Update WLAN | `PUT /api/v1/orgs/{org_id}/wlantemplates/{tid}/wlans/{wlan_id}` |
| Delete site WLAN | `DELETE /api/v1/sites/{site_id}/wlans/{wlan_id}` |

All requests use: `Authorization: Token <api_token>`

---

## Rate Limiting

The script enforces a hard limit of **2,000 API calls per hour** with a 50-call safety buffer. If the limit is approached, the script automatically sleeps until the rolling window resets. API call count and elapsed time are displayed throughout the run and logged at session end.

---

## Logging

All runs produce two files in the `logs/` subdirectory:

- `mist_bp_YYYYMMDD_HHMMSS.log` — full session log (all console output)
- `mist_bp_debug_YYYYMMDD_HHMMSS.log` — API request/response debug trace

---

## Excel Report Tabs

| Tab | Content |
|---|---|
| **By WLAN Template** | Each WLAN row with PASS/FAIL/N/A per best-practice |
| **By Site** | Each site WLAN row with PASS/FAIL/N/A per best-practice |
| **By AP** | Per-site client count and SLE Successful Connect value |

Color coding: 🟢 PASS (green) / 🔴 FAIL (red) / 🟡 N/A (yellow)

---

## Security Notes

- Read-Only token is used for all audit operations.
- The Read/Write (SuperUser) token is only requested when applying changes or deleting WLANs.
- Tokens are never written to disk or logged.
- In `--auto` mode, tokens are read from environment variables (not command-line arguments).
