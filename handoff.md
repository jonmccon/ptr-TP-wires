# TP Prioritize — Engineering Handoff

> Prototype for an individual tax preparer's case-work inbox at Priority Tax Relief (PTR). This file ships everything an engineer (human or agent) needs to wire it to a real backend without re-deriving the UX intent.

**Status:** clickable hi-fi prototype. No backend. All data is local seed in `data.js`.
**Audience:** the backend/full-stack team that owns the production build.
**Primary deliverable:** [`TP Prioritize Dashboard.html`](./TP%20Prioritize%20Dashboard.html) (also bundled standalone in `TP Prioritize Dashboard (standalone).html`).

---

## 1. Product context

PTR's tax-prep team currently runs their queue out of a shared spreadsheet (the source CSV: `uploads/TP PRIORITIZE - Individual queue.csv`). Cases flow:

1. **Origination** sells the engagement (years to prep, personal vs business, fees).
2. **Additional Services (AS)** may sell follow-on years or add scope.
3. A **team queue** receives the case.
4. A **router** assigns it to an individual preparer based on case type / urgency / specialty.
5. The preparer drives the case from "Not started" → "Filed."

This prototype is **the individual preparer's view** (step 5). The signed-in persona is **Allen Rose, Senior Tax Preparer, specializes in self-employed**.

The prototype is meant to validate three assumptions before backend build:
- **A1 — Action-needed grouping beats status-letter grouping** for the preparer's day. Statuses are a backend reality (25+ values); the preparer's mental model is "what's on my plate vs what am I waiting on?"
- **A2 — Doc collection completeness is the highest-value signal on the row.** Three checkpoints: Intuit Link sent → Docs in → Signatures collected.
- **A3 — Multi-case work is normal.** A preparer juggles 3–6 cases simultaneously, so the case detail must open as a tab, not a modal or full-screen replacement.

---

## 2. What's in the build

```
.
├── TP Prioritize Dashboard.html         ← main entry (dev: source files referenced)
├── TP Prioritize Dashboard (standalone).html ← single-file offline bundle
├── TP Prioritize Dashboard-print.html   ← print-stylesheet variant (auto opens print dialog)
├── styles.css                           ← all visual styling, no Tailwind
├── data.js                              ← STATUS, BUCKETS, REQUEST_TYPES, SAVED_VIEWS, CASES seed
├── icons.jsx                            ← inline Lucide-style SVG icon set
├── tweaks-panel.jsx                     ← starter component for the live Tweaks toolbar
├── AppShell.jsx                         ← Topbar + Sidebar
├── TabBar.jsx                           ← browser-style tabs above the workspace
├── CaseRow.jsx                          ← StatusPill, WorkChips, DocProgress, CaseRow
├── Inbox.jsx                            ← filters, saved views, grouping, bulk select
├── CaseDetail.jsx                       ← per-case tab body
├── App.jsx                              ← top-level: tabs state, search, tweaks
└── assets/
    ├── colors_and_type.css              ← PTR design-token CSS variables
    ├── ptr-flag-black.svg               ← brand mark (sidebar / print)
    └── ptr-flag-orange.svg              ← favicon
```

**Stack:**
- React 18 + ReactDOM (UMD, pinned, SRI-hashed).
- Babel Standalone for inline JSX (no build step). **The production app should swap this for a real bundler** — every `.jsx` is just a global-scope `<script type="text/babel">` for the prototype.
- No Tailwind in the prototype. The PTR design system describes itself as DaisyUI-themed, but for prototype speed we ported the token values to plain CSS in `styles.css`. **Production should re-implement these as DaisyUI utility classes against the PTR theme** (see `uploads/PTR.tokens.json` in the design system project).
- All design tokens (color, type, spacing, radius, shadows) live in `assets/colors_and_type.css` as CSS custom properties.

**Component scope sharing.** Because we use Babel Standalone, every JSX file ends with `Object.assign(window, { ... })` to expose its components globally. In a real bundle, replace with ES module `import/export`.

---

## 3. Data — what came from the CSV, what was synthesized

### 3.1 CSV columns mapped to UI

The source CSV is messy (many spreadsheet artifacts and merged-cell residue). These columns are **load-bearing**; the rest can be dropped on first migration.

| CSV column | UI surface | Notes |
|---|---|---|
| `Case #` | `#NNNNNN` in row, tab label, detail crumbs | Treat as opaque string, not int (preserve leading zeros). |
| `Last Name` + (inferred first) | Row name, tab label, detail H1 | CSV only has Last Name on most rows; first names were synthesized from the FormStack URLs (`field184002123-first=…`). In production, pull from the client record, not the spreadsheet. |
| `Status` | StatusPill on row, status filter, detail header | See §4 for the full status registry. The CSV mixes lettered workflow stages (`a) Pending Billing` … `k) Filed`) with internal flags (`Tri-Cities`, `Pending THS`, `RAD`, etc.). UI keeps the letter visible on lettered statuses. |
| `Last Updated` | Row "when" cell, detail meta, sort key | Relative ("3d ago") + absolute. |
| `Tax Years` | Scope card year chips, row sub-meta | Parsed into year[] per scope (personal / business). |
| `Pending Signatures` | DocProgress 3rd dot, detail Signatures checkrow | Sub-state ("Mailed" / "Email" / "DocuSign" / "Signed") drives icon + copy. |
| `Intuit Link Sent` | DocProgress 1st dot, detail Documents checkrow | `x` / `TRUE` → done, `FALSE` → todo, mixed personal+business → pending. |
| `Docs in?` | DocProgress 2nd dot, "Docs in / Docs out" chip filter, detail Documents checkrow | Same tri-state. |
| `Total Fee Due` | Row money column (alt mode), detail Financial card | Some rows in the CSV had non-numeric values here (`"Low Priority"`) — treat as data quality issue; not surfaced. |
| `Total Fee Paid` | Row money column (default), detail Financial card, sort key | |
| `Karbon Link` | Detail → Linked Systems card | External system link. |
| `FormStack Business` / `FormStack Personal` | Detail → Documents checkrow action links, scope card | The CSV had **many** FormStack columns; treat as 1 personal + 1 business per case max. |
| `Request Type` | Row sub-meta, "Type" filter, optional grouping axis | Values: `Tax Prep Request`, `AS Casework Ticket`, `Future Prep`, `Payments`, `Note`. |

### 3.2 Synthesized / inferred fields

The prototype shows these on the row or in the detail. They are **not in the CSV** and need to come from elsewhere in the real backend:

| Field | Where shown | Source in production |
|---|---|---|
| `firstName` | Row, detail | Client record / CRM |
| `phone` | Detail side rail | Client record |
| `email` | Detail side rail | Client record |
| `state` | Row sub-meta, detail meta | Client record (the CSV `State` column is unreliable — many `false`/blank) |
| `taxDebt` | Row sub-meta ("$Xk tax debt"), detail Financial cell | Resolution case record (separate from prep) |
| `personal: { years, intuitSent, docsIn }` | Scope card, doc progress | Split out per scope. CSV has flat columns; real schema should be relational. |
| `business: { years, intuitSent, docsIn }` | Scope card, doc progress | Same |
| `signature` enum | DocProgress dot, detail Signatures checkrow | One of `'signed' \| 'mailed' \| 'email' \| 'docusign' \| null` |
| `notes[]` | Detail Notes section | New table — user-authored, append-only timeline. |
| `activity[]` | Detail Activity timeline | Derived from status changes + system events (audit log). |
| `ageDays` | Sort by priority/age, "Overdue" saved view | Server-computed from case open date. |

---

## 4. The status system

Statuses are the gnarliest part of the data — there are two parallel taxonomies in the spreadsheet and **both need to be preserved**. They live together in `data.js → STATUS`.

### 4.1 Lettered workflow stages (a–l)

These are the canonical preparer workflow. Letters are surfaced on the pill (`b) Not Started`) because preparers say "she's a b" or "move to e" — the letter is part of their vocabulary.

| Key | Label | UI color (bg / fg) | Action bucket |
|---|---|---|---|
| `a` | Pending Billing | `#D6E2F8 / #325DBE` | client (Awaiting Client) |
| `b` | Not Started | `#E5E5E5 / #5B5B5B` | me (Awaiting You) |
| `c` | Pending Documents | `#F26FC8 / #FFFFFF` | client |
| `d` | Drafts / Pen Review | `#FFB3AB / #7A1A12` | me |
| `e` | Pending Signatures | `#1F7A4D / #FFFFFF` | client |
| `f` | Rejected | `#F39B23 / #3A1B00` | review |
| `g` | Pending Filing | `#FFC79C / #5D2A00` | me |
| `h` | Reso Review | `#D9C2E8 / #5A3A78` | review |
| `i` | Do Not File | `#1F4DAA / #FFFFFF` | parked |
| `j` | Cold Case | `#C5DCEF / #1F4D7A` | parked |
| `k` | Filed | `#3A3A3A / #FFFFFF` | closed |
| `l` | Dead Case | `#E83A3A / #FFFFFF` | parked |

### 4.2 Non-lettered flags

Internal tags that travel with a case in addition to the workflow stage. The CSV uses them as status values but in production they should probably be a separate `tags[]` column. Today they include: `Tri-Cities`, `Ready for Review`, `All Docs In`, `Future TP`, `Pending THS`, `PCOM`, `Pending Reso`, `Severance`, `Pending Start`, `In Progress`, `RAD`, `No TP Sold`, `Delinquent`. Each has a bespoke color in `STATUS`.

### 4.3 Action buckets (the "Action" grouping)

This is the default grouping for the inbox and is the **single most important interaction-design decision** in the prototype. It maps the 25 raw statuses into 5 actionable buckets:

| Bucket | Statuses in it | Intent |
|---|---|---|
| **Awaiting You** | `b`, `d`, `docs-in`, `progress`, `pstart`, `g` | The preparer's todo list — start, draft, finalize, file. |
| **Awaiting Client** | `c`, `e`, `a`, `delq`, `ths` | Sent — chase reminders, not active work. |
| **In Review** | `ready`, `h`, `f`, `rad`, `reso` | Submitted for peer/Reso review; not actionable until it bounces back. |
| **Closed & Filed** | `k`, `tri`, `pcom`, `sev` | Done. Visible for reference; usually collapsed. |
| **Parked / Inactive** | `j`, `i`, `l`, `future`, `no-tp` | Cold, dead, or out of scope. Visible for reference only. |

The "Status" and "Type" grouping modes (tweaks) are escape hatches if the action mapping is wrong for a given user.

---

## 5. Anatomy by surface

### 5.1 Top bar (`AppShell.jsx → Topbar`)
- **Brand mark** (PTR flag SVG) + product name. Doubles as the "back to inbox" affordance — clicking the pinned Inbox tab is the canonical way.
- **Search**: case# (numeric), last name, OR substring match in note bodies. Lives in `App.jsx` top-level state and is passed into `<Inbox>`. **Production:** debounce, server-side fulltext.
- **Bell / Help / Avatar (AR)**: stubs. The avatar is the user menu placeholder.

### 5.2 Sidebar (`AppShell.jsx → Sidebar`)
The persistent workspace nav. Items:
- **My Inbox** — current view (only screen wired up).
- **Team Queue** — stub for the new-cases pool, before assignment.
- **Awaiting Client** — saved cross-cut view of "Awaiting Client" cases (the high-value one).
- **Reports** — stub.
- Lists: Starred / Flagged / Archive — stubs for user-curated subsets.
- **Bottom panel** — signed-in user (Allen Rose, Senior Tax Preparer · Self-Employed). The bottom-anchored placement is intentional — sign-in identity isn't an action you take often.

### 5.3 Tab bar (`TabBar.jsx`)
- **Inbox tab is pinned** (pin icon, no close button). Pinned indicates persistent destination.
- **Case tabs** open on row click. Each carries: a status-color dot, `#caseNo` in mono, last-name-first label, X close button (visible on hover or active).
- Active tab styling: white fill, bordered, sits flush against the pane below ("page is a continuation of this tab").
- The dot color is sampled from the case's StatusPill so even with the tab text truncated you can see workflow state.
- Tab order is open-order. **Reorder is not implemented** — would be reasonable next step.

**Sidebar-tabs variant** (Tweaks → Tab style → Side): same tabs collapsed into a left column inside the workspace pane. Use this for users who keep many cases open and want full label readability.

### 5.4 Inbox header (`Inbox.jsx → inbox__head`)
- **H1 + sub** — "My Inbox · N of M cases assigned to Allen Rose."
- **Filter row** (left to right): Status (full dropdown grouped by Workflow / Internal flags) → Type → Work (Personal / Business / Both) → "Docs in" / "Docs out" toggle chips. Right side: Sort (Last updated / Fee paid / Priority age).
- **Saved views row**: 7 preset filters (All open, Awaiting me, Awaiting client, Docs ready, Ready to file, Pending signatures, Overdue 14+ days). Each shows live count. Underline indicates active. Definitions in `data.js → SAVED_VIEWS` — each is a `(case) => boolean` predicate.

### 5.5 Case row (`CaseRow.jsx`)
The most-iterated component. Standard density layout (left → right):
1. **Checkbox** — bulk-select. Activates `.bulkbar`.
2. **#caseNo** (mono, muted)
3. **Name + sub-meta** (LAST, First · request type · state · "$Xk tax debt" if any)
4. **StatusPill** — letter + label, color from registry
5. **WorkChips** — `[Personal 22–24]` `[Business 22–24]`, lit if applicable, year range collapsed to `YY–YY` for contiguous, listed otherwise
6. **DocProgress dots** — 3 steps with hover tooltips. Green-filled = done, yellow = pending, hollow = todo. Tooltip reveals which sub-state.
7. **Money** — fee paid (default) or fee due remaining (tweak: hide financials)
8. **When** — relative ("3d ago") + absolute date

**Density variants:**
- **Compact** — single row, no sub-meta. For high-volume scanning.
- **Standard** — default. Two visual lines.
- **Rich** — taller, with a 4px left bar in the status color. Use when status is the dominant axis.

### 5.6 Case detail (`CaseDetail.jsx`)
Opens as the active tab. Two columns, ~70/30 split.

**Main column:**
- **Hero** — crumbs (`My Inbox › #N`), H1 (client name), StatusPill inline, action buttons (Email · Edit · Move to next stage). Meta row: case #, request type, state, last touched, assigned.
- **Tax Prep Scope** — two side-by-side ScopeCards. Personal/Business. Each lights up with year chips and `✓ Organizer sent · ○ Docs pending`-style meta. **Answers the "what did Origination + AS sell?" question (one of the two stated user goals).**
- **Documents & Signatures** — checklist of the same 3 steps as the row dots, expanded: title + descriptive sub + action links (open form / send reminder / send DocuSign / view signed copy). **Answers the "have all docs been collected?" question (the other stated user goal).** Progress bar in the header.
- **Notes** — inline composer + reverse-chrono note list. Each note has avatar (orange = Allen, grey = teammate), author, timestamp, body. **This is the user-curated case memory.**
- **Activity** — system + user events (status changes, "Intuit link sent", "Returns mailed for signature", etc). Read-only audit log.

**Side rail:**
- **Client** — phone (mono), email (link), state.
- **Financial** — money grid: Tax debt (accent cell), Fee paid, Fee due, Status (Balance / Paid in full).
- **Quick Actions** — 2×2 grid of common one-click ops (Mark Filed, Remind client, Request docs, Move to Draft). Production wires these to status-transition endpoints.
- **Linked Systems** — Karbon, FormStack, Intuit Link cards. External-launch icon.
- **Team** — assigned preparer (you), reviewer, billing contact.

---

## 6. Tweaks (live preview controls)

Top-right toolbar toggle exposes a draggable panel with these knobs. State is persisted to `App.jsx → TWEAK_DEFAULTS` between the `/*EDITMODE-BEGIN*/.../*EDITMODE-END*/` markers — host rewrites the JSON in-file when the user changes a tweak.

| Tweak | Values | What it changes |
|---|---|---|
| `grouping` | `action` (default) · `status` · `request` · `flat` | Inbox section axis. See §4.3. |
| `density` | `compact` · `standard` (default) · `rich` | Row height + sub-meta visibility + left status bar. |
| `docProgress` | `dots` (default) · `bar` · `text` | Row doc-progress representation. Three concrete options to A/B. |
| `tabStyle` | `top` (default) · `sidebar` | Tab bar position. |
| `showFinancials` | `true` (default) / `false` | Row money column shows fee paid vs. fee due. |

**For production:** these are user-preference candidates. Persist per-user, expose as Settings instead of an in-page panel.

---

## 7. Open questions / decisions deferred to backend build

1. **Status taxonomy split.** Should lettered workflow + internal flags live in one `status` field or split into `stage` + `tags[]`? The UI handles both today but production schema should pick.
2. **Multi-scope FormStack URLs.** The CSV had ~15 columns of FormStack links per case (duplicates, alternate preparers in URL params). Real schema is 1 personal organizer + 1 business organizer per case.
3. **Tax debt source.** Surfaced on row + detail but **not in the prep CSV** — comes from the resolution side of the business. Need to confirm the join.
4. **Bulk actions semantics.** Reassign / tag / change status — confirm allowed source→target transitions in the state machine.
5. **"Pending Signatures" sub-state at row level.** Today only the dot color hints at it; full sub-state shows in detail. Consider exposing on the row for the "Pending sigs" saved view.
6. **Activity log granularity.** Which system events are worth logging? At minimum: status changes, document upload/receipt, signature events, assignment changes, note added.
7. **Search.** Prototype is client-side substring. Production: case#, name, EIN, SSN-last-4, note fulltext, fee amount.

---

## 8. Design-system anchor

The visual language is **PTR Daisy** — a DaisyUI-themed system documented at `/projects/019e2cea-ee39-704c-9992-a06235c8999a/`. The token CSS is mirrored into `assets/colors_and_type.css` so this prototype runs standalone. Key anchors:

- **Primary green** `#086033` — every actionable affordance, the section accent bars, the "Move to next stage" button.
- **Orange** `#FE5000` — reserved for the avatar bug and the PTR flag. Per brand: _"It's a referee flag — used to draw attention and then gets out of the way."_ The UI never paints buttons orange.
- **Type**: Inter for everything; Newsreader for the rare reflective callout; Source Code Pro for case numbers, fee amounts, year chips.
- **Radii**: 4px on inputs/buttons, 8px on cards. No bigger.
- **Shadows**: very restrained. No glow, no colored shadows.
- **Icons**: Lucide-style at 1.5–2 stroke, in `currentColor`. The PTR brand doesn't ship an icon library — substituted in `icons.jsx`.

---

## 9. Quick start for the next agent

```bash
# the prototype is a single static page — just open it
open "TP Prioritize Dashboard.html"
```

Or use the standalone bundle (single file, works offline):
```bash
open "TP Prioritize Dashboard (standalone).html"
```

**To re-bundle after edits:** the standalone is generated from `TP Prioritize Dashboard (bundle-source).html` which adds a `<template id="__bundler_thumbnail">` for offline splash. Edit the sources, then re-run the bundler against `(bundle-source).html`.

**To wire to a real backend:**
1. Replace the `CASES` array in `data.js` with a fetch from your case API.
2. `STATUS`, `BUCKETS`, `REQUEST_TYPES`, `SAVED_VIEWS` should move to a config endpoint so the team can edit groupings without a redeploy.
3. Replace Babel Standalone with a real bundler (Vite recommended). Convert each `window.Foo = ...` to ES exports.
4. Add a `Tab` state machine for case tabs: `loading | loaded | dirty | error`. Today they're synchronous from the seed array.
5. Saved views (`SAVED_VIEWS`) are client-side predicates today — move to server-side filtered queries to support large case loads.

---

_Generated 2026-05-21 from the prototype source. Maintainer: this file should track the actual UI — update when product decisions change, especially §4 (status) and §6 (tweaks)._
