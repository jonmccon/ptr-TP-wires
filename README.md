# TP Prioritize

Prototype https://jonmccon.github.io/ptr-TP-wires/

> A clickable hi-fi prototype for the Priority Tax Relief tax-prep team's individual case-work inbox.

This is a **prototype wireframe**, not a production app. It exists to confirm UX assumptions before backend build — specifically how a preparer should see their assigned queue, how multi-case work should flow, and how the messy spreadsheet status taxonomy should map to a usable interface.

The production build target is **Django** with the PTR DaisyUI theme. The prototype is framework-agnostic visually; this repo's source files are an HTML scaffold used purely as a visual reference.

**Open the prototype:** [`TP Prioritize Dashboard.html`](./TP%20Prioritize%20Dashboard.html)

---

## What's here

| | |
|---|---|
| **The prototype** | [`TP Prioritize Dashboard.html`](./TP%20Prioritize%20Dashboard.html) — open this to see the design |
| **Standalone bundle** | `TP Prioritize Dashboard (standalone).html` — single file, works offline, 2.4 MB |
| **Print variant** | `TP Prioritize Dashboard-print.html` — auto-fires print dialog, Letter landscape |
| **Engineering handoff** | [`HANDOFF.md`](./HANDOFF.md) — full spec for the next dev: data model, status registry, per-surface functional spec, open questions |
| **Source CSV** | `uploads/TP PRIORITIZE - Individual queue.csv` — the spreadsheet the team uses today |

---

## What it does

Persona: **Allen Rose, Senior Tax Preparer**, 18 cases assigned.

- **Inbox** — Allen's cases, grouped by "what action is needed" (Awaiting You / Awaiting Client / In Review / Closed / Parked). Each row shows status, Personal/Business scope chips with year ranges, a 3-step document-collection checklist, fee paid, and last-touched.
- **Filters + saved views** — search, status, request type, work type, "docs in / out" toggles, sort (updated / fee / age). Plus 7 named views like "Awaiting me", "Ready to file", "Overdue 14+ days".
- **Multi-case tabs** — clicking a row opens the case as a new browser-style tab above the workspace. Inbox is pinned. Switch freely.
- **Case detail** — two-column layout. Main: scope cards (what Origination/AS sold), document checklist with FormStack/Intuit/DocuSign links, notes timeline (with composer), activity log. Side rail: client info, financial summary (tax debt + fees), quick actions, linked systems, team.
- **Live preference controls** — toolbar toggle reveals a panel to swap grouping axis (Action / Status / Type / Flat), row density (Compact / Standard / Rich), doc-progress style (Dots / Bar / Text), tab position (Top / Side), and toggle the fee column. In production these become per-user settings.

---

## Why these design choices

Three assumptions the prototype is built to test:

1. **Action-needed grouping beats status grouping.** The spreadsheet has 25+ status values across two parallel taxonomies (lettered `a)…l)` workflow + freeform internal flags like "RAD", "PCOM", "Pending THS"). A preparer doesn't think in 25 buckets — they think in 5: what's on me, what's on the client, what's in review, what's done, what's parked. The "Action" grouping condenses to that. Raw status still rides on every row as a colored pill so nothing is hidden.

2. **Document-collection state is the highest-value row signal.** Origination sold X years of personal + Y years of business. The preparer's single biggest question is "do I have everything I need to start?" — surfaced as three checkpoint dots: **Intuit Link sent → Docs in → Signatures collected**.

3. **Multi-case work is normal.** Tabs at the top, Inbox pinned, cases open and close like browser tabs.

---

## Production build target

- **Backend:** Django (server-rendered templates).
- **Interactivity:** HTMX (or similar) for live filter / saved-view / note-append / status-transition swaps. No full SPA needed.
- **Styling:** Tailwind + DaisyUI v5 against the PTR theme (`uploads/PTR.tokens.json` in the design system project).
- **Design system:** **PTR Daisy** — full guide at `/projects/019e2cea-ee39-704c-9992-a06235c8999a/`.
- **State:** filter/sort/view state in URL querystring (bookmarkable). User preferences in the user profile table. Open-tab set in `sessionStorage`.

See [`HANDOFF.md`](./HANDOFF.md) §2 for the full architecture notes.

---

## Caveats

- **All client data is synthetic** — names, phones, emails, debt amounts, notes. Derived loosely from the CSV but not faithful to any real client.
- **Status colors** are sampled from the team's spreadsheet screenshot and harmonized. Tune them in the prototype's status registry if you want them exact.
- **First names** were synthesized — the CSV only carries last names reliably. Production should pull from the client record.
- **Tax debt** is shown on rows and detail but isn't in the prep CSV; it lives on the resolution side. Confirm the join before wiring.
- **No accessibility audit** beyond basic labels + focus rings. Production needs a proper a11y pass.

---

## File layout

```
.
├── TP Prioritize Dashboard.html         ← main prototype entry
├── HANDOFF.md                           ← full engineering handoff (read this for depth)
├── README.md                            ← you are here
├── uploads/
│   └── TP PRIORITIZE - Individual queue.csv  ← source spreadsheet
└── assets/
    ├── colors_and_type.css              ← PTR design tokens (port to Tailwind theme)
    └── ptr-flag-*.svg                   ← brand mark
```

The prototype's source files (`data.js`, `styles.css`, `*.jsx`, etc) are scaffolding — keep them as a visual reference but don't translate them line-by-line. The functional spec in [`HANDOFF.md`](./HANDOFF.md) is the source of truth for what the Django build should do.
