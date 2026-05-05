# Paint Shop — Quality Control System

*6 lines · 4 files · 10,000+ lines of JS · 4+ months in production*

---

## Where it started

Defects were recorded on paper. On one line out of six. One sheet per body — filled in by hand by several people. At the end of the shift — a stack of paper that no one systematically analyzed.

The other five lines kept no records at all.

My role didn't involve building systems. I was doing statistics. But I could see the data was there — nobody was collecting it. I decided to fix that.

---

## System scale

| Parameter | Value |
|---|---|
| Lines under monitoring | 6 |
| Google Sheets files | 4 (+ supporting sheets inside) |
| Apps Script code | 10,000+ lines of JS total |
| Triggers per file | onEdit + periodicSync (1 min) + up to 2 additional |
| Records per day | 350+ |
| Records per month | 7,000+ |
| Daily active users | 50+ |
| Time in production | 4+ months |

---

## First decision: not one file, but a system of files

The first question I had to answer: how to organize data collection across 6 lines? One shared file — the obvious solution. I rejected it.

The reason: each line has a different data structure. One captures body status, another — coating color, a third — doesn't have certain elements that others do. A single universal file would mean compromises everywhere: extra columns, confusion, harder to explain to operators.

The solution: 4 separate files. Each with its own Apps Script, its own structure, its own logic. Each file accurately reflects the reality of its line — nothing more, nothing less.

Three lines got individual files. The other three — final inspection, repair bay, and exit control — share one file with separate sheets, because they are technologically linked: data flows between them automatically.

| Line | File | Key feature |
|---|---|---|
| Incoming inspection | Separate | Basic set: identifier + model + defects by element |
| Intermediate inspection 1 | Separate | Same + body status → source of DPR metric |
| Intermediate inspection 2 | Separate | Same + additional coating parameter |
| Final inspection | Shared file, sheet 1 | Full structure: 5 ID fields + 26 elements + status + repair code |
| Repair bay | Shared file, sheet 2 | Receives data via QUERY, decodes repair code by element |
| Exit control | Shared file, sheet 3 | Merges two streams: direct pass and post-repair pass |

---

## Normalization: making data fit for analysis

Operators are not analysts. They enter data however is convenient: with a space, comma, slash, period. Without normalization, analytics is impossible — the same defect written in ten different formats can't be counted.

I developed a unified normalization pattern and implemented it in every file. The format: `defect_code(quantity)`.

```
Operator enters:  2 3    or  2,3   or  2/3
Script writes:    2(3)

Multiple defects on one element:
Input:   1 2 5 1
Result:  1(2),5(1)
```

Additionally — body ID normalization. The operator enters just a number, the script adds the prefix and pads to the required format:

```
125    →  BD125
bd125  →  BD125
```

### Two levels of normalization — why not one

The first obvious approach — normalize on input (`onEdit`). The problem: the standard `onEdit` in Google Apps Script has no access to `PropertiesService` or `LockService`. Under fast input or simultaneous editing from multiple devices — data can get corrupted.

The solution: two levels.

- **Level 1 — `onEdit`:** instant normalization on input. The user sees the result immediately. If the format is invalid — the script rolls back the value and shows a notification.
- **Level 2 — `periodicSync` (every minute):** correctness guarantee. Re-checks all rows, recalculates totals, restores timestamps. Runs through `LockService` — only one instance at a time.

Additionally: `onEditInstallable` is installed — a copy of the trigger with full access to services. Both call the same `_handleEdit()` function. This works around the standard `onEdit` limitation.

---

## Concurrent access: the problem you don't expect

Multiple operators work simultaneously on the same line. Each from their own tablet. All writing to the same file. Google Sheets is not built for this scenario out of the box.

The problem manifests like this: two operators edit different rows at the same time — two `onEdit` triggers fire — both try to write a timestamp — one overwrites the other or gets a lock error.

The solution — write-ahead log via `ScriptProperties`:

1. On first edit of the trigger column — the timestamp is saved to `ScriptProperties` under key `TS_A_{row}`
2. `periodicSync` picks up the value and writes it to the cell under `LockService` protection
3. After successful write — the key is deleted from Properties

This guarantees: the exact time of first input is never lost, even under a race condition between multiple `onEdit` calls.

---

## Final inspection: the most complex file

This is the central file of the system. The most data, the largest script, three interconnected sheets. This is also where the only paper-based records existed before the system.

### Record structure

Each row — one body. Five identification fields, then defects across 26 elements, then status and repair data:

| Column | Data |
|---|---|
| A | Body identifier (VIN) |
| B | Skid number — normalized automatically |
| C | Model |
| D | Color |
| E | Additional trim parameter |
| F:AE | Defects across 26 body elements |
| AF | Status: OK / Sent to repair |
| AJ | Repair defect code |
| AK | Free-text comment |

### Defect code — my solution

The task: an operator needs to quickly communicate what exactly needs to be repaired — which element, which defect. Plain text — slow and ambiguous. A table — inconvenient to enter.

I designed a compact code: 26 body elements encoded as letters, defects as numbers. The record is built as a list of pairs:

```
A1,2,B3

A = first element  →  1 = first defect type, 2 = second defect type
B = second element →  3 = third defect type
```

The operator doesn't write in words. Repair workers read the code and already know what to fix — before the body physically arrives. Preparation starts in advance.

---

## Repair bay: automatic decoding

A separate sheet in the same file. Data arrives here automatically via QUERY — no human involvement:

```
=QUERY(ARRAYFORMULA(TO_TEXT(Data)); "select * where Col32 = 'Sent to repair'"; 1)
```

Then — code decoding. The same structure of 26 elements is reproduced. Under each — a formula that looks into column AJ of that body and extracts the relevant numbers:

```
=IFERROR(
  REGEXREPLACE(REGEXREPLACE(
    REGEXEXTRACT($AJ4 & " "; "(?i)" & AU$2 & "([\d\s,]+)");
  "[^\d,]+"; ""); "^,|,$"; ""); "")
```

The result: the repair worker sees a table — element, defect numbers. No need to read and interpret the code manually. After repair: logs what was done, stamps the time, selects the exit status.

---

## Exit control: two streams in one sheet

The final point. Bodies arrive from two sources:

- Bodies with OK status from final inspection — passed without deviation
- Bodies with "Released" status from the repair bay — passed after rework

The `refreshOKSheet` script runs every 5 minutes. Merges both sources, deduplicates by composite key `VIN+SKID+model`, preserves operator comments and marks between updates.

Sliding window — last 3 days in the live sheet. An archive sheet runs in parallel: data is only appended, never deleted. On live sheet refresh — comments from the archive are restored so operator work is never lost.

---

## Data flow through the system

```
[Incoming inspection]       → separate file, data doesn't flow further
[Intermediate inspection 1] → separate file, body status → DPR metric
[Intermediate inspection 2] → separate file
[Final inspection]  ─────────────────────────────────────┐
    ↓ QUERY (status = repair)                            │
[Repair bay]                                             │ one file
    ↓ status = released                                  │
[Exit control] ← also ← [OK status from final]          │
    ↓                                                    │
[Archive] — append only                                 ─┘
```

Data flows in one direction — down through the process. Each file knows only its own part. Analytics brings everything together — but that's a separate block.

---

## Decisions that had to be made

| Challenge | Options | Decision |
|---|---|---|
| Organizing data across 6 lines | One shared file vs separate files | 4 separate files — each accurately reflects its line, no structural compromises |
| Normalizing free-form input | Validation vs normalization | Normalization: any input is brought to format, never rejected |
| Concurrent access | Ignore vs solve | Write-ahead log via ScriptProperties + LockService in periodicSync |
| Timestamp under fast input | Time from onEdit vs separate storage | Save to Properties on first trigger, transfer to cell via periodicSync |
| Passing repair information | Plain text vs structured code | Compact code: letter = element, number = defect — fast input, automatic decoding |
| Data freshness in exit sheet | Full history vs sliding window | 3-day sliding window + separate archive with no deletions |

---

## What this actually is

This is not a set of spreadsheets. It's a distributed data collection system with multiple input points, automatic normalization, concurrent access protection, and linked data flows between files.

Each of the 4 files is an independent module with its own script. Three of them are connected through QUERY and formulas. Everything runs in the browser, no software installation required, on tablets directly on the production floor.

Built by one person in one month. In production for 4+ months. Deployed into a live environment: operator training, management negotiations, iterations based on real usage.

---

*Next: analytics — real-time dashboards, on-demand deep analysis, alert system.*
