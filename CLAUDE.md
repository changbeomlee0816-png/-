# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## What this is

**수서교회 중고등부 통합관리시스템** ("Suseo Church Youth Ministry Integrated
Management System") — a member/attendance/finance management PWA for a
church's middle school + high school youth ministry (중등부/고등부).

The entire application is **one file**: `index.html`. There is no build step,
no `package.json`, no bundler, and no test suite. React, ReactDOM, and Babel
are loaded from CDN `<script>` tags, and the app's JSX source lives inline in
a `<script type="text/babel">` block that Babel transpiles **in the browser
at page load**.

```
index.html   # everything: HTML shell, CSS, all React components, seed data, Supabase sync, PWA/service-worker logic
README.md    # one-line project name (Korean)
```

Because it's a single ~330KB/1645-line file, prefer targeted reads/edits
(`grep`/`sed`/`Read` with `offset`/`limit`) over reading the whole file at
once — some data-array lines (seed data) are tens of KB long on their own.

## Running it

There is no dev server, no `npm install`, no build. Just open `index.html`
in a browser (or serve the directory with any static file server, e.g.
`python3 -m http.server`). All dependencies are fetched from CDN at runtime:

- React 18.2 / ReactDOM 18.2 / Babel Standalone 7.23 (`cdnjs.cloudflare.com`)
- No CSS framework — all styling is inline `style={{...}}` objects using the
  `C` color-token object (see below) plus a small global `<style>` reset.

There is nothing to lint, type-check, or test — verify changes by opening the
file in a browser and exercising the affected screen.

## App structure (inside `index.html`)

Everything is defined top-to-bottom in one `<script type="text/babel">`
block, roughly in this order:

1. **Constants & helpers** (`C` color palette, `won`/`dKr`/`pct`/`addDays`
   date-and-format helpers, `lastSunday`/`recentSundays`/`sundaysOfYear`
   for the Sunday-based attendance calendar).
2. **Seed/mock data** — `ACCOUNTS`, `STUDENTS0`, `TEACHERS0`, `ATTEND0`,
   `EXPENSES0`, `BUDGETS0`, `EVENTS0`, `NOTICES0`, `PRAYERS0`, `BULLETINS0`.
   These are the *initial* values used only when there's nothing in
   localStorage/Supabase yet (see Data & sync below). **The seed data
   contains realistic-looking Korean names, birthdates, and phone numbers**
   — treat it as sensitive-looking sample data; don't add more of it or
   propagate it elsewhere without asking.
3. **Reusable UI primitives**: `Chip`, `Btn`, `Card`, `CHdr` (card header),
   `Metrics`, `FI`/`FSel`/`FTA`/`FF`/`FG` (form input/select/textarea/field/
   fieldgroup), `Modal`, `Alert`, `PBar` (progress bar), `DeptBadge`,
   `TrendBars`, `NotifBell`, `SheetBtn`/`SheetModal` (CSV export UI).
4. **Domain helper functions**: `attendanceRate`, `deptAttendance`,
   `consecutiveAbsences`, `deptTrend`, `toCSV`/`downloadCSV`.
5. **Pages** (one function per screen, each takes state + setters as props):
   `LoginPage`, `Sidebar`, `Dashboard`, `AttendancePage`, `BulletinPage`,
   `StudentsPage` (+ `StudentModal`), `TeachersPage`, `PrayerPage`,
   `NoticePage`, `BudgetPage`, `EventsPage`.
6. **`SuseoChurchERP`** — the app root. Owns all top-level state (via
   `useState`), the Supabase sync effects, routing (`page` state, no
   router library — plain conditional rendering), and renders `Sidebar` +
   the active page.
7. Final `<script>` blocks: service worker registration (offline caching,
   Supabase requests explicitly excluded from the cache) and "Add to Home
   Screen" / PWA install-prompt banner logic.

### Roles & routing

Three roles, each with its own sidebar menu (`ADMIN_MENU`, `TEACHER_MENU`,
`PARENT_MENU`) and login path, handled in `LoginPage`:

- **admin** (전도사님/부장님): `admin` / `1234` — full access, including
  budget and teacher management.
- **teacher** (선생님): `suso` / `1234` — everything except `budget` and
  `teachers` pages (blocked in `SuseoChurchERP` via the `blocked` check).
- **parent** (학부모): logs in with the student's 6-digit birthdate
  (YYMMDD) as the ID and password `1234`; matched against `STUDENTS0` by
  birthdate. Parents only see their own child's data (`parentStudentId`).

There is no real auth/backend — credentials are hardcoded constants
(`ACCOUNTS`) and password checks happen client-side in plain JS. **Do not
treat this as a secure login system**; if asked to hard­en it, flag that
this needs a real auth backend rather than patching the client check.

### Data & sync model

App state (`students`, `teachers`, `attendance`, `expenses`, `budgets`,
`events`, `notices`, `prayers`, `bulletins`) lives in React `useState` in
`SuseoChurchERP` and is persisted two ways:

- **localStorage fallback**, key-prefixed `suseo_*` (`lsLoad`/`lsSave`),
  updated on every state change.
- **Supabase** (`SUPA_URL`/`SUPA_KEY`, table `app_data`, single row keyed
  `"main"`) as the source of truth when reachable. The anon key is
  embedded directly in the client JS (`index.html` around line ~1441) —
  this is a public anon key by design for a client-only app, but don't add
  any *more* sensitive keys/secrets this way.
  - On boot: fetch remote row; if present, apply it (overwrites local
    state); if absent, push current local state as the initial row.
  - Every 3 seconds (`setInterval`): if local state changed since the last
    push (`localRev` vs `pushedRev`), push it; otherwise pull remote and
    apply if it differs (`remoteHash`) — a simple last-write-wins polling
    sync, not real-time subscriptions or conflict resolution.
  - `syncStatus` (`connecting`/`online`/`offline`) drives the sync badge
    shown in the header.

When adding a new data domain, follow the existing pattern: add a
`useState` with an `lsLoad` default, include it in `collect()`/`applyData()`,
and add it to the two `lsSave(...)` effect blocks so it round-trips through
both localStorage and Supabase.

## Conventions to follow when editing

- **No build step, no imports/exports** — everything is a top-level
  `function`/`const` in the same Babel script scope. New components just
  get defined alongside the existing ones in the same block, in roughly the
  section that matches their role (primitive vs. page vs. helper).
- **Styling**: inline `style={{}}` objects referencing the `C` palette
  (e.g. `C.primary`, `C.textH`, `C.border`). No Tailwind, no CSS modules,
  no styled-components. Match the existing spacing/radius/font-size scale
  used by sibling components rather than inventing new values.
- **Language**: all UI copy, labels, and most identifiers/comments in the
  data layer are Korean. Keep new user-facing strings in Korean and
  consistent in tone/register with existing copy.
- **Dates**: Sunday-anchored school-year weeks via `SUNDAYS`/
  `recentSundays`/`sundaysOfYear`/`lastSunday` — reuse these instead of
  writing new date math for attendance/bulletin features.
- **CSV export**: use the existing `toCSV`/`downloadCSV`/`SheetBtn`/
  `SheetModal` helpers for any "export to spreadsheet" feature rather than
  building a new exporter.
- **Minified single-line data arrays** (`STUDENTS0`, `TEACHERS0`, `ATTEND0`)
  are machine-generated seed data — edit them with care (they're valid JS
  object literals, not JSON, but formatted as one huge line); avoid
  reformatting them across multiple lines unless asked, since that would
  produce a very large, hard-to-review diff.
- Keep the app framework-free and dependency-free unless explicitly asked
  to introduce a build step — the whole point of this project is that it's
  a single file anyone can open directly or drop on any static host.

## Git

- Default branch: `main`.
- Work happens directly in this repo (no monorepo/workspaces).
