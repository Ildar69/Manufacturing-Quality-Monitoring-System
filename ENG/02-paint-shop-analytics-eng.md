# Paint Shop — Analytics

*Two levels: real-time · deep analysis on demand*

---

## Two levels — two different questions

The analytics in this system are split into two independent layers, and that's by design.

**Real-time** answers the question "what's happening right now". Charts update automatically, no human involvement required. The lag between an operator entering data and it appearing on the dashboard — seconds.

**Deep analytics** answers the question "what was happening over a period". Triggered manually: pick a line, set a date range, get a detailed report. Used when you need to spot a trend, compare periods, or dig into a specific problem.

These are different tools for different questions. Combining them in one place means doing both worse.

---

## Architecture: why not one file

The obvious first move would be to put all analytics in a single file. I ruled that out.

The reason is practical: the central data file covering all lines already only opens reliably on desktop. On mobile, Google Sheets crashes — too much data, the app can't handle it. One file for everything would mean a single point of failure: it goes down, everything goes down.

The solution: each analytics file does exactly one job. If one file is unavailable, the rest keep running. Data is always accessible.

Total in the analytics layer: 10+ separate files, each with a distinct purpose.

---

## Layer 1: data centralization

First step — bring data from all lines into one place.

Each source table (where operators enter data) pushes data via IMPORTRANGE + QUERY directly — to the central file and to analytics tables in parallel. The central file is not the sole recipient: data fans out from the source.

```
[Line table — data entry]
        ↓ IMPORTRANGE + QUERY
        ├──→ [Central file — all lines]
        ├──→ [Analytics: top defects]
        ├──→ [Analytics: by body side]
        └──→ [Analytics: by model]
```

Example query pulling data from a source table:

```
=QUERY(
  {
    ARRAYFORMULA(TO_TEXT(IMPORTRANGE("source_id"; "'Sheet'!A4:E")));
    IMPORTRANGE("source_id"; "'Sheet'!F4:AK")
  };
  "select * where Col1 is not null";
  0
)
```

The reason for splitting TO_TEXT and numeric data: identifiers (VIN, model, variant) need to be text so QUERY can work with them correctly. Numbers and dates go separately to preserve their types for calculations.

---

## Layer 2: calculations

Between the raw data and the charts — a calculations layer. Separate tables compute the metrics: DPU by day, DPR, distribution by element, top defects today, side-by-side comparison.

These calculations feed back into the central file — into dedicated sheets that the charts are built on.

```
[Central file — raw data]
        ↓
[Calculation tables — metrics]
        ↓
[Central file — chart data sheets]
        ↓
[Charts — built automatically]
```

All automatic. Nothing is run manually. Data comes in — calculations update — charts refresh.

---

## Layer 3: real-time dashboards

The central file has three dashboards — three chart sheets answering different questions.

**Dashboard 1 — line overview:**

- DPU, unit count, direct pass, repairs — broken down by day
- DPR by day
- Top 10 elements by defect count — today
- Top 10 defect types — today
- Right vs left side comparison for key defects — today

**Dashboard 2 — breakdown by model:**

- Unit count per model — table view
- Top 10 defects separately for each model — today

**Dashboard 3 — defect symmetry by model:**

- For each model — a separate left vs right side comparison chart
- Makes it visible: is the defect systemic (both sides) or localised (one side)

All three dashboards update on their own. Every day — current data, no action required.

---

## Code protection: Apps Script libraries

All calculation logic is extracted into Apps Script Libraries. Inside each analytics file's script editor — only library calls:

```javascript
MyLib.generateReport();
MyLib.updateDashboard();
```

The actual implementation lives in the library, which has no public access. Opening the file and reading the logic is not possible — only the call is visible.

File access follows standard Google permissions: invite by specific email. No one can open any file in the system by accident.

---

## Layer 4: deep analytics on demand

A separate tool — for detailed breakdown over any arbitrary period.

**How it works:**

1. Open the file, see the Dashboard sheet — a control table
2. Enter "From" and "To" dates next to the target line
3. Via menu: Analytics → line → Refresh data
4. Then: Analytics → line → Report: General or By model
5. Get a fully built sheet with complete breakdown

**What's inside the General report:**

- Summary metrics: unit count in the dataset, total defects, DPU, sent to repair, DPR
- Defects by element: defect count per element, DPU per element
- Breakdown by defect type: number, name, count, DPU, right side, left side, right-left delta, breakdown by each paired element and unpaired elements

**What's inside the By model report:**

- Same blocks, but separately for each model
- Lets you compare: is one model the problem, or is the issue systemic across all

**How the code is structured:**
850+ lines of JS. Configuration is extracted into a `CONFIGS_` object — each line has its own parameters: data source, column ranges, body elements, defect types, whether DPR applies. Adding a new line means adding a config block; the logic is fully reused.

```javascript
const CONFIGS_ = {
  Line1: {
    sourceId:    "...",
    defectStart: 3,
    defectEnd:   25,
    hasDpr:      false,
    elements:    [...],
    defectNames: {...}
  },
  Line2: { ... }
};
```

Data parsing, report generation, conditional formatting (right side orange, left side blue, delta highlighted) — all generated programmatically from config.

---

## What this delivers in practice

Real-time answers the question right now: how many defects, on which element, for which model, which side is the problem today.

Deep analytics answers the question over time: what changed over the past month, which defect appeared more often, which model performs worse, is there a systemic issue on one side.

The combination of both levels gives the full picture — operational and historical at the same time.

---

## Analytics layer scale

| Parameter | Value |
|---|---|
| Analytics files | 10+ |
| Real-time dashboards | 12 (in the central file) |
| Charts total | 56 |
| Lines in deep analytics | 4 (all lines in the shop) |
| Report types | 2 (General + By model) |
| Lines of code (deep analytics) | 850+ |
| Dashboard refresh | Automatic, 1–5 sec lag |
| Human involvement | Not required for real-time |

---

*Next: alert system — [automated anomaly notifications and end-of-shift summary reports](03-alert-system-eng.md).*

