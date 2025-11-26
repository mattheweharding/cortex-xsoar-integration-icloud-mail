# Cortex XSOAR - iCloud Mail Listener (Resilient Edition)

## Overview
This is a highly customized version of the standard **Mail Listener v2** integration for Cortex XSOAR. It has been re-engineered specifically to handle **iCloud IMAP quirks** and **malformed spam emails** that typically crash standard parsers.

This integration is designed to ingest emails from the **iCloud Junk folder** (or any folder) and force them into a **Phishing workflow**, ensuring that the "Reporter" is correctly identified even if the email headers are spoofed or broken.

---

## Key Features & Modifications

### 1. Resilient "Surgical" Fetching
Standard integrations often fail when fetching spam because the server refuses to return the full RFC822 message blob (resulting in `None`) or the content is malformed.

**Logic:**
- Attempts to fetch the full email.
- If that fails, performs a "surgical" fetch, grabbing the Header and Body Text separately.

**Stitching:**
- Manually stitches these parts together to create a valid email object.
- Ensures "ghost" emails (emails with headers but missing bodies on the server) are still ingested as incidents.

---

### 2. Crash-Proof Parsing
Spam emails frequently violate MIME standards, causing the Python mail-parser library to throw exceptions.

**Logic:**
1. Attempt standard byte parsing.
2. Fallback to string decoding (UTF-8/Latin-1).
3. Fallback to a Dummy Object if parsing fails entirely.

**Result:**
The integration never crashes on a bad email. It simply creates an incident labeled "(Content Parsing Failed)" so you can still investigate the headers.

---

### 3. Forced Phishing Workflow
To ensure the **Phishing - Generic v3** playbook runs correctly, the script overrides standard mapping logic.

- **Hardcoded Type:** Incidents are explicitly set to type `Phishing`.
- **Hardcoded Reporter:** The script injects a configured email address into the `reporteremail`, `emailto`, and Reporter Label fields.

This bypasses issues where the `From` address (the attacker) is accidentally mapped as the `Reporter`.

---

### 4. Enhanced JSON Output
The output JSON has been normalized to ensure Mappers work regardless of field case sensitivity:

- Includes both `html` and `HTML`.
- Includes a coalesced `body` field (contains HTML if Text is empty, and vice versa).
- Explicitly maps the `to` field to the internal Reporter variable.

---

## Configuration

### 1. Script Variables
At the very top of the Python code, you must configure the target Reporter email. This is the internal user who will be credited with reporting the phishing attempt.

```python
# ==============================================================================
# GLOBAL CONFIGURATION VARIABLES
# ==============================================================================
FIXED_REPORTER_EMAIL = "youremail@icloud.com"
```

---

### 2. Integration Instance Settings

- **Mail Server URL:** `imap.mail.me.com` (for iCloud)
- **Port:** `993`
- **Folder:** `Junk` (or `XSOAR_Junk` if using a proxy folder)
- **Credentials:** Your iCloud email and App-Specific Password.
- **First Fetch Time:** 30 days (or 3 days for production).
- **Incident Type:** `Phishing`
- **Mapper:** `Mail Listener v2 - Mapper`

---

### 3. Mapper Configuration (Crucial)
To ensure the playbook does not crash on `!setIncident`:

1. Go to **Settings → Integrations → Classification & Mapping**.
2. Edit the Mapper.
3. Map **Reporter Email Address** to the source field `to`.

> **Note:** The script hijacks the `to` field in the JSON to match the `FIXED_REPORTER_EMAIL` variable.

---

## Debugging Commands
This integration includes custom commands to help troubleshoot IMAP connection issues without waiting for the fetch interval.

| Command | Description |
|---------|-------------|
| `!mail-listener-list-folders` | Lists all available IMAP folders on the server along with their Flags (e.g., `\Noselect`, `\Junk`). Use this to verify the folder path. |
| `!mail-listener-list-emails limit=10` | Connects, fetches, and parses the last 10 emails and prints a preview table to the War Room. Useful for verifying that the parser is working on specific spam emails. |

---

## TIM / Indicator Extraction
This integration supports the **Threat Intel Management (TIM)** module.

- Ensure the Phishing Incident Type has **"Extract Indicators"** enabled.
- The script populates the `emailbody` and `emailhtml` fields, which the Auto-Extraction engine uses to find IPs, Domains, and URLs.

---

## Known Limitations

### iCloud Junk Folder
iCloud occasionally throws an `[UNAVAILABLE] Internal Server Error` on the system Junk folder. If this occurs, create a folder named `XSOAR_Junk`, move emails there, and point the integration to that folder.

### Duplicate Detection
The script uses IMAP UIDs to prevent duplicates. If you **Reset Last Run**, all emails in the folder will be re-ingested.
