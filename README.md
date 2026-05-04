# Manufacturing Quality Monitoring System
### Real-time defect tracking across 2 workshops · 8 production lines · 7,000+ records/month

---

## Overview

A production-grade quality monitoring system built and deployed at an automotive manufacturing plant.
Covers the full cycle: data collection on the shop floor → real-time analytics → automated alerts → management reporting.

Currently in active use by **50+ people daily** — controllers, line supervisors, engineers, and plant management.

---

## The Problem

Before this system, defect tracking looked like this:

- One production line. One workshop. Paper forms.
- Each car body got a printed sheet — workers filled it in by hand
- No real-time visibility — issues surfaced at the end of a shift, or not at all
- No structured analytics — no trends, no patterns, no DPU/DPR metrics
- Zero cross-line or cross-workshop comparability

The bottleneck wasn't the number of defects. It was reaction speed and visibility.

---

## The Solution

A distributed quality monitoring system, built from scratch:

| What | Details |
|------|---------|
| Coverage | 2 workshops, 8 production lines |
| Volume | 7,000+ records/month, 350+ records/day |
| Users | 50+ daily (controllers, managers, engineers) |
| Data entry | Tablets on the production floor |
| Analytics | Real-time dashboards + on-demand deep dive |
| Alerts | Automated email notifications |
| Backend | PostgreSQL (Neon), normalized schema |
| Automation | Google Apps Script (1,000+ lines of JS) |

---

## Impact

- **Reaction time**: from end-of-shift discovery → near real-time alerts
- **Coverage**: from 1 line → 8 lines across 2 workshops
- **Visibility**: management sees live data, not yesterday's paper
- **Adoption**: the system became part of the workflow — not a side tool
- **Meetings**: weekly reviews are now data-driven, not memory-driven

---

## Architecture

```
[Tablet on line] → [Google Sheets — data entry]
                          ↓
              [Google Apps Script — automation]
                    ↙           ↘
        [Analytics layer]    [Alert engine]
         (Sheets + Looker)    (Email triggers)
                    ↘           ↙
              [Management reporting]

[Mobile app] ↔ [PostgreSQL / Neon — normalized schema]
                  (controllers · managers · post-level access)
```

---

## Core Modules

### 1. Data Collection
Controllers enter defect data from tablets directly on the production line.
Each record: VIN, body ID (SKID), model, color, hatch presence — one row per car body.
Input is normalized automatically on entry (SKID format standardization via Apps Script).

### 2. Analytics — Real-Time
Live dashboards show current shift status across all lines.
Key metrics: DPU (Defects Per Unit), DPR (Defects Per Record), hourly distribution, per-model breakdown.
Available in two modes: automated view and manual drill-down in two clicks.

### 3. Alert System
Anomaly detection engine built on a sliding-window algorithm with per-defect cooldown and subscription tiers.
When a threshold is breached — an alert fires within minutes, not at end of shift.
Replaces manual monitoring. Handles concurrent access via LockService + write-ahead log (ScriptProperties).

### 4. ETL & Data Sync
Cross-spreadsheet ETL with deduplication logic.
Hourly archive sync across 7 source files, 8 production lines.
Prevents data corruption under simultaneous tablet usage.

### 5. Mobile Application
Built for final inspection and loading posts.
Features: body part selection, defect registration, repair workflow (OK / Repair / Rework), status tracking.
Role-based access: controller vs manager, with post-level data isolation.
Backend: PostgreSQL (Neon) — 10-table normalized schema covering users, sessions, defects, repair logs, lookup dictionaries.

---

## Tech Stack

| Layer | Tools |
|-------|-------|
| Data entry | Google Sheets + Apps Script |
| Automation | Google Apps Script (JavaScript) |
| Database | PostgreSQL (Neon) |
| Mobile app | AppSheet |
| Analytics | Looker Studio |
| Alerting | Apps Script + Gmail API |

---

## My Role

This system was designed and built independently, end to end:

- Designed the full architecture from zero
- Built the data model (relational schema, 10+ tables, constraints, normalization)
- Wrote all automation logic (data validation, ETL, deduplication, alerting)
- Implemented concurrent-access handling for simultaneous multi-tablet usage
- Deployed across two workshops, integrated into existing workflows
- Trained operators on tablet-based data entry on the production floor
- Participated in defect review sessions with engineers and management
- Deliver regular reporting to multiple organizational levels — from shift supervisors to plant leadership

---

## Lessons Learned

- **Reaction speed beats data volume.** The real pain wasn't lack of data — it was delay. Real-time alerting changed the game more than any dashboard.
- **Input simplicity drives adoption.** If entry takes more than 30 seconds, people find workarounds. The tablet UX had to be frictionless.
- **Concurrent access is underestimated.** Multi-user writes to shared sheets at scale require explicit locking logic — this was one of the hardest parts.
- **Real users break things unexpectedly.** Production floor conditions are different from a dev environment. Robustness comes from watching real usage, not testing in isolation.
- **AppSheet has UX limits.** For complex data entry flows, AppSheet's constraints became visible. The mobile app is pending UX refinement before full production rollout.

---

## Screenshots

> *Coming soon — interface screenshots with anonymized production data*

| View | Description |
|------|------------|
| 📊 | Real-time dashboard — shift overview |
| 📋 | Data entry form — controller view |
| 🔔 | Alert email — anomaly notification |
| 📱 | Mobile app — defect registration flow |
| 🗄️ | Database schema — ERD diagram |

---

## Status

| Component | Status |
|-----------|--------|
| Data collection (8 lines) | ✅ Production |
| Real-time analytics | ✅ Production |
| Alert system | ✅ Production |
| Shift reporting | ✅ Production |
| Mobile app | 🔄 Pilot — UX refinement in progress |

---

## Note

Full implementation is not public. Code snippets shared here are for demonstration purposes only.  
Available for discussion — feel free to reach out.

---

*Built by [Ildar Islamov](https://linkedin.com/in/islamov-ildar) · Almaty, Kazakhstan*
