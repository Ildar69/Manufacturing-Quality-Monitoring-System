# Plastic Parts Paint Shop

*2 lines · entirely self-initiated · different logic · different approach*

---

## How it started

Before this — Excel. No automation, no analytics. Pivot tables built by hand.

I wrote a letter to management, had the conversation, found out what they needed, found the time, built it, rolled it out, set up the analytics. Every decision here is mine. There was support, but the initiative, architecture, and implementation — done alone.

This is the second system. By that point I already had the experience of the first — the body paint shop. I didn't repeat the same decisions blindly. I looked at a different shop, different people, a different process — and designed for it.

---

## Two lines — two input tables

The shop has two lines with different specs. Each got its own input table. The logic is similar, but adapted to the reality of each line.

Above them — one analytics table and one main file where everything comes together: dashboards, charts, data from both lines.

```
[Line 1 table — input]       [Line 2 table — input]
          ↓                             ↓
     [Analytics]               [Analytics]
          ↓                             ↓
          └──────────→ [Main file — dashboards, charts] ←──────────┘
```

---

## The key difference: a different input logic

In the body paint shop, the operator types `1 2` — defect code and count. The script normalises it.

Here it works differently. The operator uses dropdown lists that are generated dynamically based on context. No need to know codes, no need to remember what `1` or `5` means. Select a model — the list narrows. Select a part — you see only what belongs to that model.

This was a deliberate decision. Different shop, different people, different familiarity with digital tools. The interface had to be simpler.

---

## Smart input: how dynamic dropdowns work

Row structure:

| Column | Data |
|---|---|
| A | Row number |
| B | Date and time — set automatically when SKID is entered |
| C | SKID |
| D | Model — dropdown list |
| E | Colour — generated from model |
| F | Part name — generated from model |
| G | Defect |
| H | Count |
| + | Coating layer, resolution method, status |

**One row — one defect on one part.** This is a fundamental difference from the body paint shop where one row covers one body with all its defects.

### Dynamic dropdowns — why it matters

When the operator selects a model — the script looks into named ranges (`COL_{model}` for colours, `_{model}` for parts) and builds a dropdown only from values for that model. One model has 3 colours, another has 10 — the operator sees only what's relevant.

```javascript
function setupRowValidation(ss, sheet, row, model) {
  var namedRanges = ss.getNamedRanges();
  for (var i = 0; i < namedRanges.length; i++) {
    var name = namedRanges[i].getName();
    if (name == "COL_" + model) // colours for this model
    if (name == "_"   + model)  // parts for this model
  }
}
```

### Auto-copy row

If the operator adds another defect on the same body — the script automatically copies the SKID, model, colour, part, and dropdown rules from the previous row. No need to re-enter everything — just the defect and count.

### Automatic status

When the operator selects a resolution method (OK / Repair / Rework on line) — a machine-readable status (`ok` / `not ok`) is automatically written to a service column. This is needed for correct counting in analytics.

---

## Deep analytics — detailed report on demand

A separate file for each line. Select a period — get a full report. 850+ lines of JS, version v7.

**Skid counting logic — non-trivial:**

One body can appear multiple times in a row (multiple defects) and then reappear after another body. These are different visits — different skids. A simple unique value count gives the wrong result.

Solution: sequential grouping. One skid = an uninterrupted sequence of rows with the same VIN within a day. If the same VIN reappears after a break — it's a new skid.

```
060 → skid 060_d_1
060 → skid 060_d_1  (consecutive — same)
020 → skid 020_d_1
060 → skid 060_d_2  (interrupted — new)
Total: 3 skids, not 2
```

**What's inside the report:**

1. Overall stats: total skids, OK, repair, rework on line, DPR, DPR-2, DPU
2. Stats by model
3. DPR by day
4. DPU by day broken down by model
5. Defect trend by day — with hourly and model breakdown
6. All defects over the period in descending order
7. DPU by defect per model — comparison table
8. Detail by part: part → defect → count → DPU
9. Defects by resolution status: what went to repair, what was reworked on line
10. Summary matrix: defect × status

Two DPR metrics calculated differently:

- **DPR** = OK / Total × 100 — clean pass only
- **DPR-2** = (Total − Repair) / Total × 100 — including rework on line

Conditional formatting: DPU ≥ 0.3 — red, 0.1–0.3 — yellow, < 0.1 — green. Problems are visible immediately without reading the numbers.

---

## On the Microsoft stack

I considered deploying the system on the Microsoft stack. Looked into it — ruled it out. The tools don't offer the flexibility this kind of work requires, and the workarounds turn into constructions that are hard to maintain. Google Workspace turned out to be the right choice in this context.

---

## Mobile application

A mobile app was also developed for this shop — following the same logic as for the body paint shop. Frozen for the same reasons: organisational readiness, input process, inertia.

More detail in the mobile app section.

---

## What sets this system apart from the first

| | Body paint shop | Plastic parts shop |
|---|---|---|
| Initiative | Natural extension of the role | Entirely self-initiated |
| Row logic | One body — all defects | One defect — one part |
| Input | Free text + normalisation | Dynamic dropdown lists |
| Unit counting | By records | Sequential skid grouping |
| DPR | Single metric | Two metrics (DPR and DPR-2) |
| Analytics | Centralised, shared | Two separate files per line |

Two systems. One person. Different problems — different solutions.

---

*Next: README — the case that brings everything together.*
