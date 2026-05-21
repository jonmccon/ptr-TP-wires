# TP Prioritize

> A clickable hi-fi prototype for the Priority Tax Relief tax-prep team's individual case-work inbox.

This is a **prototype wireframe**, not a production app. It exists to confirm UX assumptions before backend build — specifically how a preparer should see their assigned queue, how multi-case work should flow, and how the messy spreadsheet status taxonomy should map to a usable interface.

**Open it:** [`TP Prioritize Dashboard.html`](./TP%20Prioritize%20Dashboard.html)

---

## What's here

| | |
|---|---|
| **The prototype** | [`TP Prioritize Dashboard.html`](./TP%20Prioritize%20Dashboard.html) — dev entry, references source files |
| **Standalone bundle** | `TP Prioritize Dashboard (standalone).html` — single file, works offline, 2.4 MB |
| **Print variant** | `TP Prioritize Dashboard-print.html` — auto-fires print dialog, Letter landscape |
| **Engineering handoff** | [`HANDOFF.md`](./HANDOFF.md) — full spec for the next dev/agent: data lineage, status registry, intent per surface, open questions |
| **Source CSV** | `uploads/TP PRIORITIZE - Individual queue.csv` — the spreadsheet the team uses today |

---

## What it does

Persona: **Allen Rose, Senior Tax Preparer**, 18 cases assigned.

- **Inbox** — Allen's cases, grouped by "what action is needed" (Awaiting You / Awaiting Client / In Review / Closed / Parked). Each row shows status, Personal/Business scope chips with year ranges, a 3-step document-collection checklist, fee paid, and last-touched.
- **Filters + saved views** — search, status, request type, work type, "docs in / out" toggles, sort (updated / fee / age). Plus 7 named views like "Awaiting me", "Ready to file", "Overdue 14+ days".
- **Multi-case tabs** — clicking a row opens the case as a new browser-style tab above the workspace. Inbox is pinned. Switch freely.
- **Case detail** — two-column layout. Main: scope cards (what Origination/AS sold), document checklist with FormStack/Intuit/DocuSign links, notes timeline (with composer), activity log. Side rail: client info, financial summary (tax debt + fees), quick actions, linked systems, team.
- **Live tweaks** — toolbar toggle to swap grouping axis (Action / Status / Type / Flat), row density (Compact / Standard / Rich), doc-progress style (Dots / Bar / Text), tab position (Top / Side), and toggle the fee column.

---

## Why these design choices

Three assumptions the prototype is built to test:

1. **Action-needed grouping beats status grouping.** The spreadsheet has 25+ status values across two parallel taxonomies (lettered `a)…l)` workflow + freeform internal flags like "RAD", "PCOM", "Pending THS"). A preparer doesn't think in 25 buckets — they think in 5: what's on me, what's on the client, what's in review, what's done, what's parked. The "Action" grouping condenses to that. Raw status still rides on every row as a colored pill so nothing is hidden.

2. **Document-collection state is the highest-value row signal.** Origination sold X years of personal + Y years of business. The preparer's single biggest question is "do I have everything I need to start?" — surfaced as three checkpoint dots: **Intuit Link sent → Docs in → Signatures collected**.

3. **Multi-case work is normal.** Tabs at the top, Inbox pinned, cases open and close like browser tabs.

---

## Stack

- React 18 + Babel Standalone (no build step — every `.jsx` is a `<script type="text/babel">`)
- Plain CSS in `styles.css`, design tokens in `assets/colors_and_type.css`
- No backend. All data is local seed in `data.js`, derived from the source CSV
- Design system: **PTR Daisy** — DaisyUI-themed. Tokens copied locally so the prototype runs standalone

For a real build, swap Babel Standalone for Vite, replace `CASES` with API calls, and re-implement the components as DaisyUI utility classes against the PTR theme. Full migration notes in [`HANDOFF.md`](./HANDOFF.md).

---

## Caveats

- **All client data is synthetic** — names, phones, emails, debt amounts, notes. Derived loosely from the CSV but not faithful to any real client.
- **Status colors** are sampled from the team's spreadsheet screenshot and harmonized. Tune them in `data.js → STATUS` if you want them exact.
- **First names** were synthesized — the CSV only carries last names reliably. Production should pull from the client record.
- **Tax debt** is shown on rows and detail but isn't in the prep CSV; it lives on the resolution side. Confirm the join before wiring.
- **No accessibility audit** beyond basic labels + focus rings. Production needs a proper a11y pass.

---

## Project structure

```
.
├── TP Prioritize Dashboard.html         ← main entry
├── HANDOFF.md                           ← full handoff doc (read this for depth)
├── data.js                              ← seed cases, status registry, action buckets
├── styles.css                           ← all visual styling
├── App.jsx                              ← top-level state (tabs, search, tweaks)
├── AppShell.jsx                         ← topbar + sidebar
├── TabBar.jsx                           ← browser-style tabs
├── Inbox.jsx                            ← filters, grouping, bulk select
├── CaseRow.jsx                          ← row + status pill + work chips + doc dots
├── CaseDetail.jsx                       ← per-case tab body
├── icons.jsx                            ← inline Lucide-style SVGs
├── tweaks-panel.jsx                     ← live tweaks toolbar
└── assets/
    ├── colors_and_type.css              ← PTR design tokens
    └── ptr-flag-*.svg                   ← brand mark
```

For data lineage (CSV column → UI surface), the full status taxonomy, and every open question deferred to backend build — see [`HANDOFF.md`](./HANDOFF.md).
