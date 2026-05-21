# TP Prioritize — Engineering Handoff

> Specification for an individual tax preparer's case-work inbox at Priority Tax Relief (PTR). This document is **framework-agnostic** — it describes _what_ each surface needs to do and _what data_ it draws from, not how the prototype happens to be assembled. Target implementation: **Django** (server-rendered templates + HTMX or similar for interactivity).

**Status:** the HTML prototype is a clickable visual reference. This doc is the source of truth for behavior.
**Audience:** the engineering team building the production app.
**Visual reference:** [`TP Prioritize Dashboard.html`](./TP%20Prioritize%20Dashboard.html) — open it to see the intended look-and-feel.

---

## 1. Product context

PTR's tax-prep team currently runs their queue out of a shared spreadsheet (the source CSV: `uploads/TP PRIORITIZE - Individual queue.csv`). Cases flow:

1. **Origination** sells the engagement (years to prep, personal vs business, fees).
2. **Additional Services (AS)** may sell follow-on years or add scope.
3. A **team queue** receives the case.
4. A **router** assigns it to an individual preparer based on case type / urgency / specialty.
5. The preparer drives the case from "Not started" → "Filed."

This document specifies **the individual preparer's view** (step 5). The signed-in persona used throughout the prototype is **Allen Rose, Senior Tax Preparer, specializes in self-employed**.

The build is meant to validate three assumptions:
- **A1 — Action-needed grouping beats status-letter grouping** for the preparer's day. Statuses are a backend reality (25+ values); the preparer's mental model is "what's on my plate vs what am I waiting on?"
- **A2 — Doc collection completeness is the highest-value signal on the row.** Three checkpoints: Intuit Link sent → Docs in → Signatures collected.
- **A3 — Multi-case work is normal.** A preparer juggles 3–6 cases simultaneously, so the case detail must open as a tab (or equivalent persistent, side-by-side surface), not a modal or full-screen replacement.

---

## 2. Architecture notes for the Django build

The prototype is a single static HTML page. The Django build is a multi-view app. Recommended high-level shape:

**URL routes (suggested):**
| Route | View | Purpose |
|---|---|---|
| `/inbox/` | InboxView | The preparer's inbox (default landing). Honors querystring for filters/sort/view. |
| `/inbox/?view=ready-file&sort=updated` | InboxView | All filter/sort/group state is URL-bound — shareable, bookmarkable. |
| `/cases/<case_no>/` | CaseDetailView | A single case's detail page. |
| `/cases/<case_no>/notes/` | NoteCreate (POST) | Append a note. |
| `/cases/<case_no>/transitions/` | StatusTransition (POST) | Move to next stage / mark filed / etc. |
| `/api/cases/?q=…` | Async search/filter (HTMX or JSON) | Powers the live filter + saved-view tabs without full page reloads. |

**The "tab" pattern in a Django world.** The prototype uses an in-page tab bar so the preparer can have multiple cases open at once. Two reasonable production translations:

- **A.** **Client-side tab shell.** The Django app renders the inbox and case-detail templates as normal, but a small JS layer keeps a tab strip in the browser. Each tab fetches its content via HTMX into a target div. Tab state lives in `sessionStorage` so a reload restores the open set.
- **B.** **Native browser tabs.** Open each case in a new browser tab (`target="_blank"` on inbox rows). Drop the custom tab bar entirely. Simpler, but the user loses the in-product "Inbox" pin.

The prototype demonstrates option A's visual model. The team should pick before build — both are valid; A is closer to the prototype.

**State persistence:** the prototype uses local component state for filters. Production should put filter/sort/view state in **querystring params** so users can deep-link saved searches. The "Saved views" feature (§5.4) becomes server-side `SavedView` records owned by the user.

**Real-time:** none of this requires websockets. The inbox can poll every 30–60s, or simply refresh on tab switch.

---

## 3. Data — what came from the CSV, what was synthesized

### 3.1 CSV columns mapped to UI

The source CSV is messy (many spreadsheet artifacts and merged-cell residue). These columns are **load-bearing**; the rest can be dropped on first migration.

| CSV column | UI surface | Notes |
|---|---|---|
| `Case #` | `#NNNNNN` in row, tab label, detail crumbs | Treat as opaque string, not int (preserve leading zeros). |
| `Last Name` + (inferred first) | Row name, tab label, detail H1 | CSV only has Last Name on most rows; first names were synthesized from the FormStack URLs (`field184002123-first=…`). In production, pull from the client record, not the spreadsheet. |
| `Status` | Status pill on row, status filter, detail header | See §4 for the full status registry. The CSV mixes lettered workflow stages (`a) Pending Billing` … `k) Filed`) with internal flags (`Tri-Cities`, `Pending THS`, `RAD`, etc.). UI keeps the letter visible on lettered statuses. |
| `Last Updated` | Row "when" cell, detail meta, sort key | Relative ("3d ago") + absolute. |
| `Tax Years` | Scope card year chips, row sub-meta | Parsed into year[] per scope (personal / business). |
| `Pending Signatures` | Doc-progress 3rd step, detail Signatures checkrow | Sub-state ("Mailed" / "Email" / "DocuSign" / "Signed") drives icon + copy. |
| `Intuit Link Sent` | Doc-progress 1st step, detail Documents checkrow | `x` / `TRUE` → done, `FALSE` → todo, mixed personal+business → pending. |
| `Docs in?` | Doc-progress 2nd step, "Docs in / Docs out" chip filter, detail Documents checkrow | Same tri-state. |
| `Total Fee Due` | Row money column (alt mode), detail Financial card | Some rows in the CSV had non-numeric values here (`"Low Priority"`) — treat as a data-quality issue; not surfaced. |
| `Total Fee Paid` | Row money column (default), detail Financial card, sort key | |
| `Karbon Link` | Detail → Linked Systems card | External system link. |
| `FormStack Business` / `FormStack Personal` | Detail → Documents checkrow action links, scope card | The CSV had **many** FormStack columns; treat as 1 personal + 1 business per case max. |
| `Request Type` | Row sub-meta, "Type" filter, optional grouping axis | Values: `Tax Prep Request`, `AS Casework Ticket`, `Future Prep`, `Payments`, `Note`. |

### 3.2 Suggested Django model shape

Not prescriptive — adapt to the existing data layer — but this is the rough decomposition the UI assumes:

```
Client
  first_name, last_name, phone, email, state
  total_tax_debt        (decimal; from resolution side, joined in)

Case
  case_no               (string, opaque, unique)
  client                FK Client
  request_type          enum (tax-prep | as-ticket | future | payments | note)
  status                FK Status  ← the lettered workflow stage
  tags                  M2M StatusFlag  ← the non-lettered flags (RAD, PCOM, Future TP…)
  assigned_to           FK User
  opened_at             datetime
  last_updated          datetime  (auto-bumped on any child write)
  date_completed        datetime, nullable
  fee_due               decimal
  fee_paid              decimal
  karbon_url            url, nullable
  intuit_url            url, nullable
  signature_state       enum (signed | mailed | email | docusign | null)

CaseScope                ← one row per personal/business scope on a case
  case                  FK Case
  kind                  enum (personal | business)
  years                 int[] or M2M Year
  intuit_sent           bool
  formstack_url         url, nullable
  docs_in               bool

Note                     ← user-authored
  case                  FK Case
  author                FK User
  body                  text
  created_at            datetime

ActivityEvent            ← system + user events for the timeline
  case                  FK Case
  actor                 FK User, nullable (null = system)
  kind                  enum (status_change | doc_received | signature_sent | … )
  payload               json
  occurred_at           datetime

SavedView                ← per-user named filter set
  user                  FK User
  label                 string
  query                 json   (filter + sort + grouping bag)
```

### 3.3 Synthesized / inferred fields

The prototype shows these. They are **not in the CSV** and need a source in production:

| Field | Where shown | Source |
|---|---|---|
| `first_name` | Row, detail | Client record / CRM |
| `phone`, `email` | Detail side rail | Client record |
| `state` | Row sub-meta, detail meta | Client record (the CSV `State` column is unreliable — many `false`/blank) |
| `total_tax_debt` | Row sub-meta ("$Xk tax debt"), detail Financial cell | Resolution case record (separate from prep) |
| `intuit_sent`, `docs_in` per scope | Scope card, doc progress | Split out per scope. CSV has flat columns; real schema should be relational. |
| `signature_state` | Doc-progress step, detail Signatures checkrow | One of `signed | mailed | email | docusign | null` |
| `notes[]` | Detail Notes section | New table — user-authored, append-only timeline. |
| `activity[]` | Detail Activity timeline | Derived from status changes + system events (audit log). |
| `age_days` | Sort by priority/age, "Overdue" saved view | Computed `now() - opened_at`. |

---

## 4. The status system

Statuses are the gnarliest part of the data — there are two parallel taxonomies in the spreadsheet and **both need to be preserved**.

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

Implement as a `Status` model seeded via a data migration. The letter, label, and color values should live in the DB so admins can tune them without a redeploy.

### 4.2 Non-lettered flags

Internal tags that travel with a case in addition to the workflow stage. The CSV uses them as status values but in production they should be a separate `tags[]` (many-to-many) relation. Today they include: `Tri-Cities`, `Ready for Review`, `All Docs In`, `Future TP`, `Pending THS`, `PCOM`, `Pending Reso`, `Severance`, `Pending Start`, `In Progress`, `RAD`, `No TP Sold`, `Delinquent`. Each has a bespoke color (full registry in the prototype's `data.js → STATUS`).

### 4.3 Action buckets (the "Action" grouping)

This is the default grouping for the inbox and is the **single most important interaction-design decision** in the prototype. It maps the 25 raw statuses into 5 actionable buckets:

| Bucket | Statuses in it | Intent |
|---|---|---|
| **Awaiting You** | `b`, `d`, `docs-in`, `progress`, `pstart`, `g` | The preparer's todo list — start, draft, finalize, file. |
| **Awaiting Client** | `c`, `e`, `a`, `delq`, `ths` | Sent — chase reminders, not active work. |
| **In Review** | `ready`, `h`, `f`, `rad`, `reso` | Submitted for peer/Reso review; not actionable until it bounces back. |
| **Closed & Filed** | `k`, `tri`, `pcom`, `sev` | Done. Visible for reference; usually collapsed. |
| **Parked / Inactive** | `j`, `i`, `l`, `future`, `no-tp` | Cold, dead, or out of scope. Visible for reference only. |

Store as a `Status.action_bucket` column. The "Status" and "Type" grouping modes (see §6) are escape hatches if the action mapping is wrong for a given user.

---

## 5. Functional spec by surface

Each subsection describes one surface from the prototype: **what it shows**, **what data it needs**, **what actions it accepts**, **how it changes state**.

### 5.1 Top bar

**Shows:** PTR brand mark + product name, global search box, notifications bell, help link, current-user avatar.

**Data:** signed-in user (initials, full name, role).

**Actions:**
- **Global search** — query: substring across `case_no`, `last_name`, `first_name`, `note.body`. Debounced 250ms. Server-side fulltext recommended (Postgres `tsvector`). Submits to `/inbox/?q=…` or replaces the inbox table via HTMX.
- **Notifications bell** — opens a popover with unread events (status changes by teammates on the user's cases, mentions in notes, etc). Not implemented in prototype — stub for now.
- **Avatar** — user menu (profile / sign out).

### 5.2 Sidebar

**Shows:** persistent workspace nav.

**Items:**
- **My Inbox** — `/inbox/`, current view (only screen wired up in prototype).
- **Team Queue** — `/team-queue/`, stub for the pre-assignment pool.
- **Awaiting Client** — saved cross-cut view of "Awaiting Client" cases. High-value because it's a "what should I poke today?" list.
- **Reports** — stub.
- **Lists**: Starred / Flagged / Archive — stubs for user-curated subsets.

**Data:** counts beside each nav item (open cases assigned to me, team queue size, awaiting-client count).

**Bottom panel:** signed-in user identity. Bottom-anchored placement is intentional — sign-in identity isn't an action you take often.

### 5.3 Tab bar (workspace tabs)

**Shows:** the open tab strip above the case workspace. Always includes a **pinned Inbox tab**. Each open case is its own tab with a status-color dot, `#caseNo` in mono, and the client's name. Hover/active reveals an X to close.

**Data:** the set of open case_nos in this session.

**Actions:**
- Click a tab → activate it.
- Click X → close.
- Click an inbox row → open that case as a new tab.
- Close last case tab → fall back to Inbox.

**Persistence:** session-scoped (`sessionStorage`) so a refresh restores the working set. Not per-user-DB-persisted — tabs are ephemeral, like browser tabs.

**Production note:** an alternative is to drop the in-product tab bar and use native browser tabs (anchor links with `target="_blank"`). The prototype demonstrates the in-product version because it keeps Inbox always reachable via the pin.

### 5.4 Inbox — filters & saved views

**Shows:**
- Title + count: `My Inbox · N of M cases assigned to <user>`.
- Filter controls: **Status** (full dropdown grouped by Workflow / Internal flags), **Type** (Request Type), **Work** (Personal only / Business only / Both), **Docs in / Docs out** toggle chips.
- Sort dropdown: **Last updated** (default), **Fee paid**, **Priority / age**.
- Saved-views tab strip: 7 preset filters, each shows live count (All open, Awaiting me, Awaiting client, Docs ready, Ready to file, Pending signatures, Overdue 14+ days).

**Data:** the user's cases + all filter options + counts per saved view.

**State:** every filter, sort, view, and grouping decision lives in the URL querystring. Example: `/inbox/?view=ready-file&type=tax-prep&sort=paid&group=action`. This makes views bookmarkable and shareable.

**Saved views — recommended implementation:**
1. Seed the 7 named views as system-owned `SavedView` rows.
2. Allow users to save their current filter combination as a new view (button on the filter bar: "Save view").
3. Display system views first, then user views.

**Server-side filtering:** the prototype filters in memory. With realistic case loads (hundreds per preparer, thousands across the team), filter to the DB. The querystring → Django QuerySet translation should be a single `apply_filters(qs, request.GET)` helper to keep all surfaces consistent.

### 5.5 Case row

The most-iterated component. Standard density layout (left → right):

1. **Checkbox** — bulk-select. Activates the bulk-action bar (§5.6).
2. **#caseNo** (mono, muted).
3. **Name + sub-meta** — `LAST, First · request type · state · "$Xk tax debt"` (last bit shown only if any).
4. **Status pill** — letter + label, color from registry (§4).
5. **Work chips** — `[Personal 22–24]` `[Business 22–24]`. Lit if applicable. Year range collapsed to `YY–YY` for contiguous, listed otherwise. Data source: `CaseScope` per kind.
6. **Doc-progress dots** — 3 steps with hover tooltips. Green-filled = done, yellow = pending, hollow = todo. Tooltip reveals which sub-state.
7. **Money** — fee paid (default) or fee due remaining (configurable).
8. **When** — relative ("3d ago") + absolute date.

**Click behavior:** the entire row is clickable; opens the case as a tab.
**Hover:** subtle border darken + shadow lift.

**Density variants** (user preference):
- **Compact** — single row, no sub-meta. For high-volume scanning.
- **Standard** — default. Two visual lines.
- **Rich** — taller, with a 4px left bar in the status color. Use when status is the dominant axis.

**Doc-progress computation:** for each of the 3 steps, aggregate across personal + business scopes:
- `intuit_sent`: `done` if all sold scopes have `intuit_sent=true`, `pending` if some, `todo` if none.
- `docs_in`: same logic.
- `signature`: `done` if `signature_state='signed'`, `pending` if any other non-null value, `todo` if null.

### 5.6 Bulk action bar

**Triggers:** appears when ≥1 row is checkbox-selected.

**Actions:** Reassign… · Tag… · Change status… · Clear.

Each opens a small dialog. The status-change action must respect the workflow state machine (define allowed `(from, to)` transitions in a `STATUS_TRANSITIONS` config).

### 5.7 Case detail — main column

Opens as the active tab body (or `/cases/<case_no>/` if using native tabs).

#### 5.7.1 Hero
- Crumbs: `My Inbox › #<case_no>`.
- H1: client full name. Status pill inline.
- Action buttons: **Email client** · **Edit** · **Move to next stage** (primary). The "Move to next stage" button computes the next status from the current one + the workflow transition table.
- Meta row: case #, request type, state, last touched, assigned-to.

#### 5.7.2 Tax Prep Scope
Two side-by-side scope cards: **Personal (1040)** and **Business return**. Each lights up if sold, dims if not. Shows year chips and `✓ Organizer sent · ○ Docs pending`-style status meta. **Answers "what did Origination + AS sell?" — one of the two stated user goals.**

Data: `CaseScope` rows for this case.

#### 5.7.3 Documents & Signatures
Expanded version of the 3 row dots:
- **Intuit organizer link sent** — `done` / `pending` / `todo` + descriptive sub + actions: open personal form, open business form.
- **Client documents received** — `done` / `pending` / `todo` + sub + actions: open FormStack, send reminder.
- **Signatures collected** — `done` / `pending` / `todo` + sub describing sub-state + actions: send DocuSign / view signed copy.

Each "action" link is a separate POST endpoint (`/cases/<n>/send-reminder/`, etc) that returns the updated checklist HTML (HTMX swap target).

**Answers "have all docs been collected?" — the second stated user goal.**

#### 5.7.4 Notes (user-authored)
Inline composer + reverse-chrono list. Each note: avatar (orange for self, grey for others), author name, timestamp, body (preserved whitespace).

**This is the user-curated case memory.** The single most-used write surface in the prototype.

Endpoint: `POST /cases/<n>/notes/` with `body`. Response: the new note's HTML for HTMX append.

#### 5.7.5 Activity timeline (system events)
Read-only audit log. Each event: actor (or "system"), message, timestamp.

Event kinds to emit:
- `status_change` — `Status → Pending Signatures`
- `assignment_change` — `Case assigned to Allen Rose`
- `intuit_sent` — `Intuit link sent (personal + business)`
- `doc_received` — `All docs received and verified`
- `signature_sent` — `Returns mailed for signature`
- `signature_received` — `Returns signed by client`
- `filed` — `Status → Filed`

Implement as `ActivityEvent` rows; auto-create on the relevant model `post_save` signals or in a `case.transition_to(new_status)` service method.

### 5.8 Case detail — side rail

A second column (~30% width) with read-mostly summaries. **Order matters** — most-asked information at top.

- **Client** — phone (mono, click-to-copy), email (mailto), state.
- **Financial** — money grid: Tax debt (accent cell, biggest), Fee paid, Fee due, Status (Balance / Paid in full).
- **Quick Actions** — 2×2 grid of common one-click ops: Mark Filed, Remind client, Request docs, Move to Draft. Each maps to a transition endpoint.
- **Linked Systems** — Karbon, FormStack, Intuit Link cards. External-launch icons. Pull the URLs from the case record.
- **Team** — assigned preparer (you), reviewer, billing contact. Static for now; could pull from case-team assignments.

---

## 6. User preferences (what the prototype calls "Tweaks")

These are knobs the prototype exposes via a floating panel for design exploration. In production, surface as **per-user settings** (saved to the User profile / preferences table) and expose via a Settings page or a small kebab menu on the inbox.

| Preference | Values | What it controls |
|---|---|---|
| `inbox_grouping` | `action` (default) · `status` · `request` · `flat` | Inbox section axis. See §4.3. |
| `row_density` | `compact` · `standard` (default) · `rich` | Row height + sub-meta visibility + left status bar. |
| `doc_progress_style` | `dots` (default) · `bar` · `text` | Row doc-progress representation. |
| `tab_style` | `top` (default) · `sidebar` | Tab bar position. (Only relevant if using in-product tabs.) |
| `show_financials_column` | `true` (default) / `false` | Show fee paid column on the row. |

---

## 7. Visual + design-system requirements

The visual language is **PTR Daisy** — a DaisyUI-themed system documented at `/projects/019e2cea-ee39-704c-9992-a06235c8999a/` (in the design ecosystem). Production build should:

- Use the **PTR DaisyUI theme** (`uploads/PTR.tokens.json` in the design system project). All visual tokens — colors, type, radii, shadows — come from there.
- Use **Tailwind + DaisyUI v5** for component classes (`btn btn-primary`, `card`, `input`, `badge`, etc). The prototype hand-rolled CSS for speed; production should use the framework's class API.
- Inter for everything; Newsreader for the rare reflective callout; Source Code Pro for case numbers, fee amounts, year chips.
- Radii: 4px on inputs/buttons, 8px on cards. No bigger.
- Shadows: very restrained. No glow, no colored shadows.
- Icons: Lucide-style at 1.5–2 stroke, in `currentColor`. The PTR brand doesn't ship an icon library — Lucide is the substitute.

**Key brand rules to honor:**
- **Primary green** `#086033` is the only "go" color. Every actionable affordance, section accent bar, primary button.
- **Orange** `#FE5000` is reserved for the avatar/brand mark. Per the brand book: _"It's a referee flag — used to draw attention and then gets out of the way."_ Never paint general UI orange.

---

## 8. Open questions / decisions deferred to backend build

1. **Status taxonomy split.** Should lettered workflow + internal flags live in one `status` field or split into `status` + `tags[]`? The UI handles both today; the proposed schema in §3.2 splits them. Confirm before migration.
2. **Multi-scope FormStack URLs.** The CSV had ~15 columns of FormStack links per case (duplicates, alternate preparers in URL params). Real schema is 1 personal organizer + 1 business organizer per case.
3. **Tax debt source.** Surfaced on row + detail but **not in the prep CSV** — comes from the resolution side of the business. Confirm the join.
4. **Bulk actions semantics.** Reassign / tag / change status — confirm allowed source→target transitions in the state machine. Define the `STATUS_TRANSITIONS` table.
5. **"Pending Signatures" sub-state at row level.** Today only the dot color hints at it; full sub-state shows in detail. Consider exposing on the row for the "Pending sigs" saved view.
6. **Activity log granularity.** Confirm which system events are worth logging. Suggested minimum list in §5.7.5.
7. **Search scope.** Prototype is client-side substring. Production: case#, name, EIN, SSN-last-4, note fulltext, fee amount. Postgres `tsvector` recommended.
8. **In-product tabs vs native browser tabs.** Pick one (see §2). All other surfaces are unaffected.

---

## 9. Reading the prototype as a reference

When in doubt about pixel-level styling or exact copy, open [`TP Prioritize Dashboard.html`](./TP%20Prioritize%20Dashboard.html). The prototype is the source of truth for:

- Exact colors of every status pill
- Action-bucket assignments for every status
- Row layout proportions
- Detail card ordering
- Copy on every label, button, empty state
- Filter dropdown labels and option lists
- Doc-progress tooltip text

When in doubt about behavior or data, this document (`HANDOFF.md`) is the source of truth.

The prototype's seed data lives in `data.js`. Treat it as a fixture set for migration testing — 18 cases covering most status combinations, both personal-only and personal+business engagements, the full range of doc-progress states, and a variety of fee-paid/due/tax-debt combinations.

---

_Generated 2026-05-21. Maintainer: this file should track the actual UI — update when product decisions change, especially §4 (status), §5 (per-surface specs), and §6 (preferences)._
