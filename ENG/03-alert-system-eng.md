# Alert & Reporting System

*Final inspection only · two notification types · fully autonomous*

---

## Context

The alert system runs on the final inspection line — the most central and most loaded line in the shop. Data arrives there via IMPORTRANGE + QUERY from the source table. Three sheets: BASE (data), Subscriptions (mailing management), Log (send history).

Two notification types solve different problems and operate independently from each other.

---

## Type 1: Anomaly detector → digest

### The idea

The challenge isn't logging a defect. The challenge is catching a pattern — when the same defect on the same element repeats car after car. Manually, you'd only see this at the end of a shift. The system sees it within minutes.

### How the detector works

Runs every 5 minutes. Takes the last 10 bodies that already have a status (OK or Sent to repair) — a sliding window. For each defect + element pair, it counts: how many times did this combination appear in that window.

If 7 out of 10 bodies share the same defect on the same element — the threshold is crossed. The pair is written to the queue.

```
Threshold:  7 / 10 bodies
Window:     last 10 with a status
Interval:   every 5 minutes
```

### Why it doesn't send immediately

Sending on every trigger fire is spam. If the problem isn't fixed, the detector will find it again every 5 minutes.

Solution: queue + digest. The detector writes to the queue; the digest sends everything accumulated once every 2 hours. One email — all current issues.

### Cooldown

Each defect + element pair is blocked for 2 hours after entering the queue. Even if the detector finds it again — it won't be re-added. This eliminates duplicates within a single digest window.

```
Cooldown:  2 hours per defect + element pair
Storage:   ScriptProperties (key ALERT_{defectCode}_{partCol})
Cleanup:   expired keys deleted automatically
```

### What arrives in the email

One email contains all anomalous pairs accumulated over 2 hours. For each:

- Defect name and code
- Body element
- Counter: how many bodies in the window were affected (e.g. 8/10)
- Body list: identifier, skid number, model, variant, time recorded
- Severity level: 90%+ of window — CRITICAL, otherwise — WARNING

The email is HTML-formatted — structured, readable on mobile.

### Working hours

The detector only runs during working hours on working days.

---

## Type 2: End-of-shift summary report

### When it sends

Automatically at the end of each working day. The report is built 18 minutes before the shift ends — management sees the day's picture before the shift is over.

Duplicate protection: the report for a specific date is sent exactly once. A re-triggered run checks ScriptProperties and skips if it's already been sent.

### What's inside the report

**Shift summary:**

- Total bodies, OK, sent to repair, no status
- DPU — defects per unit
- DPR — share of bodies that passed without repair
- Average time on line (minutes)
- Total defects recorded

**Bodies sent to repair:**

- List of each body: identifier, skid, model, variant, time
- Breakdown of causes via the AJ code — not raw codes, but readable element and defect names
- Summary: which defects and elements caused repair most often

**Repair comments:**

- Separate block for bodies where the comment references an "Other" defect — anything outside the standard classifier

**All defects of the shift:**

- Table sorted by count
- For each defect: name, count, DPU, top 3 elements where it appears most

**Bodies by hour:**

- Hourly distribution with a visual bar
- Peak hour highlighted automatically

### Morning report

There's an option to send a morning report — it looks into the database and builds a report for yesterday. This solves a real problem: the evening report is built before the shift ends, so the last bodies don't make it in. The morning report closes that gap.

Currently disabled (`MORNING_REPORT_ENABLED: false`) — under consideration for activation.

---

## Subscription management

Not hardcoded. Managed through the Subscriptions sheet directly in the spreadsheet — no script changes needed.

| Name | Code | Email 1 | Email 2 | ... |
|---|---|---|---|---|
| Shift reports | 0 | ivan@company.com | — | — |
| All alerts | -1 | qa@company.com | — | — |
| Defect type A | 1 | EXCLUDE | — | — |
| Defect type B | 2 | ivan@company.com | qa@company.com | — |

**Code logic:**

- `0` — receives end-of-shift summary reports
- `-1` — receives all alerts regardless of defect type
- `1, 2, 3...` — defect number: receives alerts for that specific defect only
- `EXCLUDE` — defect is fully excluded from the detector, no alerts sent for it

Add a person — one row in the sheet. Exclude a noisy defect — write EXCLUDE. Assign a responsible person for a specific defect — put their email next to the right code.

---

## Log

Every sent email is recorded in the Log sheet: date and time, type (DIGEST_ALERT / REPORT_evening / REPORT_morning), defect code, defect name, body element, recipients, details. History is preserved — you can see how many alerts fired yesterday, the day before, compare the trend right in the spreadsheet.

---

## Trigger architecture

```
Every 5 minutes  →  runDetector()
                     checks last 10 bodies with status
                     writes anomalies to queue (ScriptProperties)

Every 2 hours    →  sendHourlyDigest()
                     reads queue → builds HTML → sends → clears queue

Every 15 minutes →  sendEveningReport()
                     checks: hour == target hour and minute
                     if yes — builds today's report → sends (once)

Every 15 minutes →  sendMorningReport()  [disabled]
                     checks: hour and minute
                     if yes — builds yesterday's report → sends (once)
```

All triggers are time-based — run on schedule. No dependency on user actions.

---

## What makes this system smart

**Sliding window instead of a daily counter.** The system doesn't wait for a defect to hit N occurrences over a shift. It looks at the last 10 bodies right now. If a problem appears — it's visible within minutes, not hours.

**Queue against spam.** Detector and mailing are separated. The detector might fire 20 times in 2 hours — the recipient gets one email with the full picture, not 20 identical ones.

**Cooldown per pair, not per system.** If there's simultaneously a problem with defect A on element 1 and defect B on element 2 — both make it into the digest. The cooldown blocks only the specific pair, not the entire system.

**Subscriptions without code.** A manager can exclude a noisy defect or add a new responsible person directly in the sheet — without touching the script.

**Duplicate report protection.** ScriptProperties stores the fact of sending for a specific date. Even if the trigger fires multiple times — the report goes out once.

**1000+ lines of JS.** Defect parsing, AJ code decoding into readable names, HTML email templates, queue management, logging — all in one script, no external dependencies.

---

## Scale

| Parameter | Value |
|---|---|
| Lines with alerts | 1 (final inspection) |
| Detector runs per day | ~60 (every 5 min, working hours) |
| Digests per day | up to 5 (every 2 hours during working hours) |
| Summary reports per day | 1 (end of shift) |
| Defect types in the registry | 52 |
| Body elements | 26 |
| Lines of code | 1000+ |

---

*Next: mobile app — PostgreSQL, AppSheet, the freeze, and what comes next.*
