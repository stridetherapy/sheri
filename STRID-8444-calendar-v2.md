# STRID-8444 — Calendar V2 (Scheduling Hub) — Design

> Companion tasks doc: `STRID-8444-calendar-v2-tasks.md`.
> Handoff/working state: `STRID-8444-calendar-v2.local.md` (gitignored — created by whoever is actively executing).

---

## Context

Current main scheduling calendar (`/calendar` → `components/calendar/Calendar.tsx` → `components/shared/Scheduler/`) is built on **DevExpress DX React Scheduler v4**, legacy `useApi`/`useFetch`, and Zustand. Epic 8444 rebuilds it on **FullCalendar v6 resource-timegrid** to ship the unified Scheduling Hub described in PM's PRD (`SCHEDULING_HUB_PRD.md`, 4192 lines, 46 sections — lives on Clayton's Desktop, not in repo). The new surface adds inline auth/payer alerts, drag-rebook with diff confirm, smart scheduling, bulk ops, per-scope saved views, and a unified waitlist + online-scheduling queue.

**Out of scope this epic** (confirmed):
- Wellness / group classes (PRD §33–43)
- AI features: no-show prediction, slot ranking, message generation
- Saved view persistence beyond what current calendar already does (mirror current behavior — reuse `userProfile.defaultCalendarUrlParams`)

**Goal:** Replace the DevExpress calendar with FullCalendar v6 behind an app setting + ghost-mode query param. Stride staff can preview in prod/demo/dev via `?scheduling_hub=true`; per-practice rollout flips via `AppSettings.SCHEDULING_HUB`.

---

## Pushback on locked decisions

The locked decisions are followed in this plan, but each carries a known risk worth flagging:

1. **Query-param name precedent is split.** KPI_DASHBOARD_V2 reads `?v2_dashboard=true` on BE but `?feature=v2` on FE. We use `?scheduling_hub=true` on **both** sides; do not propagate the precedent drift.
2. **Ghost-mode is not staff-gated in the KPI precedent** — `?feature=v2` works for any PracticeAdmin. For SCHEDULING_HUB the query param check requires `request.user.is_staff` so customer admins cannot toggle a half-shipped V2. See `S0.3` + `S0.12`.
3. **`defaultCalendarUrlParams` schema collision** — V1 and V2 store shapes diverge. The serialized blob gets a `{v: 2, ...}` discriminator so V1 readers ignore unknown fields. Round-trip unit test in `S0.11`.
4. **Sub-flag shape** is flat top-level keys on the AppSetting `value` JSON (`{visible, smartScheduling, bulkOps, rooms, teams, search}`), not a nested `features.*` map — one fewer JSON path lookup for callers.

---

## Key Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Calendar engine | FullCalendar v6 (`@fullcalendar/resource-timegrid`, `interaction`, `scrollgrid`) | PRD design built on FC v6; mature drag/resize, resource cols, sticky headers |
| License | Commercial FC license (open question — see Risks) | Stride is closed-source SaaS; GPL key insufficient |
| Feature gate (per-practice) | `AppSettings.SCHEDULING_HUB` with `{ visible, smartScheduling, bulkOps, … }` sub-flags | Matches `KPI_DASHBOARD_V2` precedent (`backend/utils/constants.py` `DEFAULT_APP_SETTINGS`). Sub-flags let us progressively enable individual features per practice without code redeploy |
| Staff ghost mode | Query param `?scheduling_hub=true` consumed via new `useSchedulingHubEnabled()` hook (mirrors `useHasInternalFeature.ts` precedent). **Gated on `user.is_staff`** so customer admins cannot self-enable. BE counterpart in `backend/utils/feature_flags.py::scheduling_hub_enabled(request)`. | Lets Stride staff QA in prod/demo without flipping the app setting |
| Route splitting | `routes/base.tsx` lazy-loads `pages/schedulingHub` separately via `reactLazyWithRetry` | Keeps existing bundle untouched for non-v2 practices |
| State | Zustand stores + TanStack Query for server data | Current calendar uses legacy `useApi` — v2 modernizes to react-query per `frontend-data-fetching.md`. Client UI state stays in Zustand |
| Theme | `strideThemeV2` wrapped at v2 calendar root only | Per `frontend-theming.md` — no nested ThemeProviders inside |
| Forms | RHF V2 components (`RHFTextField`, `RHFAutocomplete`, etc.) | Better typing; convention for new features |
| Saved views | Reuse `userProfile.defaultCalendarUrlParams` (existing). V2 writes a `{v: 2, ...}`-discriminated blob; V1 readers ignore unknown `v`. | Confirmed: mirror current behavior. PRD's localStorage `stride:*` keys not needed. Discriminator prevents V1↔V2 toggle from corrupting saved views |
| Pause-polling registry | Centralize in `stores/schedulingHub.ts` with `addPauseSource(id)/removePauseSource(id)` API consumed by tooltip, modal, drawer, dialogs | V2 has more open surfaces than V1's tooltip-only pause; raw `refetchOnWindowFocus` alone is insufficient |
| Tooltip composition | Recompose monolithic V1 `AppointmentTooltip.tsx` into props-only section components (`Patient`, `Auth`, `Insurance`, `Charges`, `Warnings`, `Actions`) | 50KB monolith; section split enables progressive port + testability |
| Recurring appointments | Backend remains authoritative for expansion; FC renders flat instances (no native FC recurrence rules) | Matches current architecture; FC native recurrence has subtle TZ/DST gaps |
| Mobile fallback | Below `md` breakpoint, FC switches to `listWeek`; toolbar becomes bottom sheet; sidebar becomes drawer | `resource-timegrid` is unusable on phone widths |
| Compatibility | V2 lives alongside V1 routes. No deletion of DevExpress code until rollout complete | Safe rollback |
| Plan doc style | Design + companion tasks doc with stable IDs `S<cat>.<n>` | Multi-session epic; stable IDs let conversations reference work units across sessions |

---

## Architecture

```
frontend/src/
├── pages/calendar/
│   └── index.tsx                          # MODIFY: V1/V2 lazy swap based on useSchedulingHubEnabled
│                                          #   (POC version already does this — see S0.4)
├── components/
│   ├── calendar/                          # V1 — untouched until SX.3 deletion
│   ├── shared/Scheduler/                  # V1 — untouched until SX.3 deletion
│   └── schedulingHub/                     # NEW — anchored on CH/fullcalendar-poc salvage
│       ├── SchedulingHub.tsx              # root w/ <ThemeProvider strideThemeV2>  (POC ✓)
│       ├── SchedulingHubToolbar.tsx       # toolbar                                (POC ✓)
│       ├── useSchedulingHub.ts            # hub state hook — port axios → RQ        (POC PORT)
│       ├── mappers.ts                     # Appointment ↔ FC EventInput            (POC ✓)
│       ├── scheduling-hub-overrides.css   # FC CSS — port hex → tokens             (POC PORT)
│       ├── AppointmentBlock.tsx + story   #                                        (POC ✓)
│       ├── AppointmentDetailDialog/       # tooltip-equivalent w/ section split    (POC ✓)
│       ├── CreateAppointmentDialog/       # RHF V2 form — port useAppointmentForm axios → RQ (POC PORT)
│       ├── Sidebar/                       # NEW — @dnd-kit-based
│       ├── DragConfirmDialog.tsx          # NEW
│       ├── CancelReasonDialog.tsx         # NEW
│       ├── SmartSchedulingDialog.tsx      # NEW
│       ├── SmartRebookDialog.tsx          # NEW
│       ├── WaitlistPanel.tsx              # NEW wrapper over components/waitlist/*
│       ├── BulkOps/                       # NEW
│       └── calendar/                      # engine-agnostic adapter (POC contribution)
│           ├── FullCalendarAdapter.tsx    # only module that imports @fullcalendar/* (POC ✓)
│           ├── types.ts                   # CalendarEngineProps/Ref/Event/View      (POC ✓)
│           └── index.ts                                                            #(POC ✓)
├── api/
│   ├── schedulingHub/                     # NEW: queries.ts + mutations.ts (RQ)
│   └── appSettings/                       # existing — extend with SCHEDULING_HUB
├── stores/
│   └── schedulingHub.ts                   # NEW Zustand: UI state only (filters, view, bulk, pause registry)
├── utils/hooks/
│   └── useSchedulingHubEnabled.ts         # NEW: app setting + staff-gated query param
└── routes/base.tsx                        # MODIFY: conditional v1/v2 lazy import based on gate
```

**Engine-agnostic adapter pattern (POC contribution).** `components/schedulingHub/calendar/FullCalendarAdapter.tsx` is the **only** module in the hub that imports `@fullcalendar/*`. Hub code consumes the adapter through `CalendarEngineProps` / `CalendarEngineRef` interfaces in `calendar/types.ts`. This isolates any future engine swap to a single file.

Backend additions land in their own apps under `backend/api/` (no new top-level packages):
- `appsettings` defaults extended (new entry in `backend/utils/constants.py`)
- `appointments` views/serializers extended for new fields (visit count, secondary insurance, co-treater flag, scheduling-warning fields, authorization expand). Validation hooks extended on existing `OverlapSettings` (do **not** introduce a new `SchedulingRule` model — reuse the existing extension point)
- `teams` (new model + viewset under `api/practice/teams/` or `api/teams/`)
- `rooms` (likely new under `api/practice/rooms/`)
- `availability` ranking endpoint for Smart Scheduling
- `utils/feature_flags.py` — new helper module exposing `scheduling_hub_enabled(request)` that any endpoint can use to branch V1/V2 behavior consistently
- `backoffice/` — new view to flip per-practice `SCHEDULING_HUB` (and sub-flags) without DB access; writes are audit-logged

---

## Rollout phasing

1. **S0 — Foundation** lands first and unblocks everything. App setting, ghost-mode hook, lazy route, FC v6 install, theme wiring, empty v2 shell. Ghost-mode preview available to staff at this point.
2. **S1–S9** run in any order after S0. Backend tickets within a category can run parallel to FE shell tickets once contract is agreed.
3. **Cutover** (separate epic-close ticket): flip default `SCHEDULING_HUB.visible = true`, delete DevExpress Scheduler folder, remove v1 route.

## Salvage from `CH/fullcalendar-poc` (2026-03-03)

Prior POC by Clayton at `CH/fullcalendar-poc` contains `frontend/src/components/schedulingHub/` (~24 files, ~2300 LOC) — substantively the V2 shell. Branch is 394 commits behind main; **do not rebase**. Instead, each S0/S2/S5/S6/S7 ticket cherry-picks its slice from POC via `git checkout CH/fullcalendar-poc -- <path>` and lands as its own PR.

POC files split into:
- **KEEP** (19 files — drop in clean): all types, constants, barrels, stories, MUI components (`SchedulingHubToolbar`, `mappers.ts`, `AppointmentBlock.tsx`, `AppointmentDetailDialog/*`, `CreateAppointmentDialog.tsx`, `FormContent.tsx`, `calendar/types.ts`, `calendar/index.ts`, etc.). `FormContent.tsx` already uses RHF V2.
- **PORT — small** (1 file): `scheduling-hub-overrides.css` (5 hex literals → tokens).
- **PORT — medium** (2 files): `SchedulingHub.tsx` (one axios call → RQ), `useSchedulingHub.ts` (axios CRUD → RQ).
- **PORT — large** (1 file): `useAppointmentForm.ts` (patient-search axios + manual state → RQ).
- **PORT — small** (1 file): `FullCalendarAdapter.tsx` (inline styles → `sx`; FC imports stay).

No POC file uses `useApi`/`useApiOld`/`useFetcher`, `tss-react`, `makeStyles`, `moment`, or `react-beautiful-dnd`. No broken imports on current `main`. No overlap with V1 `components/calendar/` or `components/shared/Scheduler/`.

Per-ticket-PR strategy: each cherry-pick batch ships with its parent ticket (e.g. `S0.5` adds FC deps + `calendar/types.ts`; `S0.6` brings in `SchedulingHub.tsx` + `Toolbar` + `mappers.ts` + CSS-with-hex-port; `S2.6` brings in `FullCalendarAdapter`; `S5.1` brings in `AppointmentBlock`; `S6.1` brings in `AppointmentDetailDialog/*`; `S7.1` brings in `CreateAppointmentDialog/*` with the form-hook port). Tickets are annotated `(POC: KEEP|PORT)` in the tasks doc.

---

## Risks & Open Questions

| Risk | Mitigation |
|---|---|
| **FullCalendar commercial license** — Stride closed-source; commercial key required for prod. **Dev proceeds with GPL-compatible key**; procurement is a parallel non-blocking track (S0.1). Only `SX.1` pilot rollout gates on the commercial key being in env. | S0.1 demoted to parallel procurement track; S0.2–S0.15 unblocked |
| **DevExpress → FC behavior diffs** — drag-drop, availability rendering, time-snapping side-by-side, copy-paste all currently bespoke on DX. Direct port may not be 1:1 | Each "existing" task does a behavior-parity check vs current; document FC equivalent in task notes |
| **Now-indicator CSS override** spans `400vw` per PRD §3 | Validate via prototype before relying on |
| **Resource view vs Individual view dual render** — PRD says two memoized FC instances. Risk of state-sync bugs across instances | Encapsulate in `<ScheduleCalendar>` with single source of truth in Zustand store |
| **Drag-confirm dialog adds friction** vs current drag-and-go | Confirm with PM whether dialog is opt-in (user setting) or always-on |
| **Concurrent edits during polling** — 30s poll currently pauses on tooltip open. New v2 has more open surfaces (modal, drawer, panels). Need a unified "pause polling" registry | Centralize in Zustand `stores/schedulingHub.ts` with `addPauseSource(id) / removePauseSource(id)` |
| **Saved views mirror current** — current uses `defaultCalendarUrlParams` (URL search params). PRD adds per-Location / per-Team / All-Clinicians scopes. Confirm whether scope-keyed prefs require new BE field or can be URL-encoded | Open question for product/BE |
| **V1↔V2 saved-view schema collision** — V2 stores fields V1 doesn't recognize | `{v: 2, ...}` discriminator + V1 readers ignore unknown `v`; round-trip unit test (`S0.11`) |
| **DevExpress GroupingPanel column totals** have no FC equivalent | Custom `resourceLabelContent` + derived counts from React Query cache |
| **Timezone correctness in `resource-timegrid` across DST transitions** | Build `utils/timezone.ts` adapter early; DST-transition unit tests in `S0.15` |
| **Bundle size** — FC plugins add ~150KB gz | Lazy V2 chunk; CI bundle-size guard in `SQ.4`; budget +150KB max delta |
| **Smart scheduling perf** — algorithm is O(therapists × business_days × windows × slots) | Budget ≤500ms p95 (20 therapists × 14 days); paginate by day on BE if exceeded (`S9.6`) |
| **Mobile UX of `resource-timegrid` is weak** | Fallback to FC `listWeek` below `md` (`S1.5`) |
| **a11y of custom event content** — FC default a11y OK, custom `eventContent` adds gaps | Explicit ARIA labels in renderer; axe pass per `SQ.3` |
| **Customer-admin self-enable of half-shipped V2** via copying URL with `?scheduling_hub=true` | Ghost-mode gated on `user.is_staff` server- and client-side |
| **TenantBaseModel migrations across many practice DBs** for new `teams`/`rooms` apps | Use `--database=migrations`; coordinate fan-out with infra owner before merge |
| **Recurring expansion divergence** if FC native recurrence is later enabled | Document BE-as-authoritative rule in `utils/fcAdapters.ts` header (`S0.14`) |
| **No backoffice toggle UI** — staff need to flip SCHEDULING_HUB per practice | Build backoffice view in `S0.9` with audit logging |

Open questions to resolve during S0:
- FullCalendar license status?
- Drag confirm dialog: always-on or user toggle?
- Per-scope saved views (location/team/all): URL-encode under existing field, or new `calendarV2Preferences` JSON field on user profile?
- Bulk operations limit — cap N at 50? 100? PRD doesn't specify
- Does "Refresh on tab switch" mean OS tab focus or in-app tab? (Current = window focus)

---

## Verification (epic-level)

Each S0–S9 task carries its own done-when. Epic-wide acceptance:
1. With `SCHEDULING_HUB.visible = false` and no query param → `/calendar` renders V1 unchanged (zero regression in current test suite).
2. With `?scheduling_hub=true` (any practice) → `/calendar` renders V2 shell.
3. With `SCHEDULING_HUB.visible = true` for a practice → all users at that practice see V2 by default.
4. Run existing `frontend/src/components/shared/Scheduler/__tests__` against V1 path (untouched). Add new Vitest suites under `components/schedulingHub/**/__tests__` per `frontend-testing.md`.
5. Backend: `CI=true pipenv run pytest tests/test_appsettings.py tests/test_teams.py tests/test_rooms.py` etc. pass.
6. Manual smoke (S0 done-when): staff loads `/calendar?scheduling_hub=true` on demo → sees empty v2 shell with theme applied, no errors.
7. Final cutover smoke: staff flips one pilot practice's `SCHEDULING_HUB.visible = true`, validates full day-of-life workflow in prod.
8. Playwright E2E parity suite under `frontend/tests/e2e/calendar-v2/` green in CI (`SQ.1`) — one happy-path per category (toggle, view switch, drag, create, edit, cancel, smart schedule, waitlist).
9. axe-core a11y pass clean on V2 route (`SQ.3`).
10. V2 chunk size within `SQ.4` budget (+150KB gz max vs V1 baseline).
11. Telemetry events `scheduling_hub.session_start` + `feature_used.*` visible in analytics (`SQ.5`).

Internal QA runbook + Loom (`SX.4`) captures the toggle path, ghost-mode usage, regression checklist, and screen recording of golden paths so QA/PM can self-verify without engineering shadowing.

---

## Critical files to reference

**Frontend (existing, to read/reuse):**
- `frontend/src/components/shared/Scheduler/Scheduler.tsx` — current DX scheduler
- `frontend/src/components/shared/Scheduler/Toolbar.tsx` — current toolbar
- `frontend/src/components/shared/Scheduler/AppointmentTooltip.tsx` — current tooltip behaviors
- `frontend/src/components/shared/Scheduler/CancelAppointmentModal.tsx` — cancellation reason flow
- `frontend/src/components/shared/Scheduler/useDragEdit.tsx` — current drag handling
- `frontend/src/stores/calendar.tsx` — current Zustand store (model to mirror, then modernize)
- `frontend/src/hooks/calendar/index.ts` — `serializeCalendarStateToSearchParams` / `useFetchCalendarData` / polling
- `frontend/src/components/waitlist/{WaitlistDrawer,WaitlistItemModal,AddFromWaitlistModal}.tsx` — port to v2 panel
- `frontend/src/api/waitlist/{index,pendingRequests}.ts` — existing API; reuse
- `frontend/src/entities/AppSetting/index.ts` — extend with `SCHEDULING_HUB`
- `frontend/src/hooks/useHasInternalFeature.ts` — pattern for query-param hook
- `frontend/src/contexts/app/state.ts` — `appSettings` slot
- `frontend/src/routes/base.tsx` — lazy load split
- `frontend/src/contexts/theme/stride-v2/` — theme tokens
- `frontend/src/utils/hooks/` — destination for `useSchedulingHubEnabled`

**Backend (existing, to extend):**
- `backend/api/appsettings/{models,serializers,views}.py`
- `backend/utils/constants.py` — `DEFAULT_APP_SETTINGS`
- `backend/api/appointments/` — extend serializers/views for new block badges (visit count, secondary ins, co-treater)
- `backend/api/practice/` — likely home for `teams` and `rooms` apps
- `backend/api/availability/` — Smart Scheduling ranking endpoint

**PRD reference (not in repo):** `~/Desktop/drive-download-20260514T152326Z-3-001/SCHEDULING_HUB_PRD.md`. Component sections (§10–§43) describe individual component contracts. Companion HTML render at `StrideSchedulingHub.html`. Claude-design prototype source at `07-wellness-booking-v6/`.
