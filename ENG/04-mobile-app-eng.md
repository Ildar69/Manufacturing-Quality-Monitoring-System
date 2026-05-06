# Mobile Application

*Two backend versions · not in production · an honest story*

---

## What it is and why

The current data collection system runs through Google Sheets on tablets. That solution works — but it has a ceiling: one operator is simultaneously inspecting for defects and entering data. That's not their fault, and not the system's fault — it's a process problem.

The mobile app was built as the next step: a more professional interface, separate from spreadsheets, with a role model, a body map, and reference books built right in.

---

## Two versions — one logic

The system was developed twice with different backends:

**Version 1: AppSheet + Google Sheets**
Fast start, everything inside the Google ecosystem. AppSheet's limitations became visible as the UX grew more complex — a no-code tool doesn't give the flexibility needed for intricate input scenarios.

**Version 2: AppSheet + PostgreSQL (NeonDB)**
A more professional setup. Relational database, normalised schema, full control over data structure. Analytics connect directly to the database via Looker Studio.

The application logic is identical in both cases — only the backend changed.

---

## How the app works

**Role model and data isolation**

One app — for all lines. But each user sees only their own: every account has a baked-in profile — email, name, role, shift, post, line. Only explicitly added users can log in. Data is filtered automatically — an inspector on post 3 doesn't see data from post 1.

**Main screen — Info**

The logged-in user's profile, a built-in chat for communication between posts, a defect reference with descriptions, and a model reference. This is a fundamental difference from a spreadsheet: the operator doesn't have to hold everything in their head — they can look up right inside the app what a defect looks like, or how one model differs from another.

**Side menu: Active · Archive · Repair**

All work flows through three sections. Active — current shift. Archive — history. Repair — bodies under rework.

**Body card**

Tap the plus button — enter the primary information. A full card opens: all entered information at the top, defect block below. To add a defect: select element → select defect → enter count. The defect reference inside the card shows only the defects relevant to that post — not the full list, just what applies.

**Analytics via Looker Studio**

With the PostgreSQL backend, analytics connect directly to the database through Looker Studio. The same real-time analytics as in the current system — but more readable, with more visualisation options.

---

## Why it's not in production

The app was developed and tested across both shops. The feedback was consistent: everyone liked it, everyone was on board — but the transition never happened.

The reason isn't the app. The reason is the process: right now one person simultaneously inspects the body and enters the data. The app requires a few more steps than the spreadsheet — and in that context, it matters. The spreadsheet is faster in this scenario simply because it's familiar and more primitive.

The right fix is to split roles: one person inspects, another enters data. That's an organisational decision, not a technical one. Until the process changes, the app won't deliver the advantage that outweighs inertia.

**Technical limitation:** AppSheet as a no-code platform has a UX flexibility ceiling. For complex input scenarios, that ceiling is real. The next version — a full web application: HTML, CSS, JavaScript, Supabase or NeonDB, no no-code constraints. Python and AI tools will have a place there too.

---

## Status

| Component | Status |
|---|---|
| AppSheet + Sheets | Built, tested, frozen |
| AppSheet + PostgreSQL | Built, tested, frozen |
| Next version (web) | Planned |

The app is frozen — not abandoned. The organisational conditions for its rollout haven't matured yet. Technically, it works.

---

*Next: [plastic parts paint shop — the second system, different logic, entirely self-initiated](05-plastic-shop-eng.md).*

