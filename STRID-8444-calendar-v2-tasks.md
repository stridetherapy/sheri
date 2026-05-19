# STRID-8444 — Calendar V2 — Tasks

> Stable IDs `S<section>.<n>`. **Never renumber** — IDs are referenced across sessions and PRs.
> `Done-when:` is the acceptance check. `Depends-on:` references other task IDs.
> Companion design doc: `STRID-8444-calendar-v2.md`. Working state: `STRID-8444-calendar-v2.local.md` (gitignored).
> When complete, strike the task heading: `### ~~S0.1 ...~~ ✅`. Capture commit/PR refs in the local file.

---

## S0 — Foundation & Feature Gate

### S0.1 FullCalendar v6 license — procurement track (parallel, non-blocking for dev)
Dev work proceeds with FC's GPL-compatible key during development. Commercial license required **before prod rollout** (Stride is closed-source SaaS).

1. File procurement ticket for FC commercial license (Standard vs Premium — Premium needed for `resource-timegrid` + `scrollgrid` in closed-source prod).
2. Track procurement separately; do not block other S0.* tickets.
3. Once procured, swap GPL key for commercial key in `VITE_FULLCALENDAR_LICENSE` env (used by `S0.5`).

**Done-when:** Commercial key in env config + 1Password before SX.1 pilot rollout.
**Depends-on:** —
**Blocks:** SX.1 (pilot rollout) only — does **not** block S0.2–S0.15 or feature tickets.

### S0.2 Add `SCHEDULING_HUB` app setting
1. Backend: extend `DEFAULT_APP_SETTINGS` in `backend/utils/constants.py` with `"SCHEDULING_HUB": {"visible": False, "smartScheduling": False, "bulkOps": False, "rooms": False, "teams": False, "search": False}`.
2. Frontend: add `SCHEDULING_HUB = "SCHEDULING_HUB"` to `AppSettings` enum in `entities/AppSetting/index.ts`.
3. Frontend: define `TSchedulingHubSettings` type for value shape.

**Done-when:** New AppSetting row exists on any practice queried via `/api/appsettings/`; FE enum compiles.
**Depends-on:** S0.1

### S0.3 `useSchedulingHubEnabled` hook
1. Create `frontend/src/utils/hooks/useSchedulingHubEnabled.ts` returning `{ enabled, flags, ghostMode }`.
2. Combine logic: `enabled = appSettings?.[AppSettings.SCHEDULING_HUB]?.visible === true` **OR** (`user.isStaff === true` **AND** query param `?scheduling_hub=true`). Mirror `useHasInternalFeature.ts` for the param-read pattern.
3. Sub-flag accessors: `flags.smartScheduling`, etc. — true if (staff ghost mode) OR app setting sub-flag true.
4. Vitest unit test cases: ghost on + staff / ghost on + non-staff → disabled / app setting on only / both off / sub-flag scenarios.

**Done-when:** Hook returns expected combinations; tests pass; **non-staff users with `?scheduling_hub=true` get `enabled: false`**.
**Depends-on:** S0.2

### S0.4 Lazy-load v2 route (POC: PORT)
1. Modify `frontend/src/pages/calendar/index.tsx` to swap V1↔V2 based on `useSchedulingHubEnabled().enabled` and lazy-import the V2 entry via `reactLazyWithRetry`. **POC's version already implements this pattern using `appSettings[SCHEDULING_HUB]`** — adapt it to use the new `useSchedulingHubEnabled` hook (S0.3) so it picks up staff ghost mode too.
2. Wrap in `<Suspense fallback={<AppLoadingScreen />}>` per existing pattern.
3. Verify bundle: v2 chunk only loads when gate is on.
4. Keep route at `/calendar`; do not create `pages/schedulingHub/`. Hub renders at the existing calendar URL behind the flag.

**Done-when:** Network tab shows v2 chunk loaded only with gate on; v1 route unchanged with gate off.
**Depends-on:** S0.3
**POC source:** `git show CH/fullcalendar-poc:frontend/src/pages/calendar/index.tsx`

### S0.5 Install FullCalendar v6 + plugins (POC: KEEP)
1. Add FC deps to `frontend/package.json`: `@fullcalendar/{core,daygrid,interaction,react,resource,resource-timegrid,scrollgrid,timegrid}@^6.1.20`. **POC `package.json` already has these** — copy the lines, do not copy POC's `yarn.lock`. Run `yarn install` fresh.
2. Salvage the engine-agnostic types from POC: `git checkout CH/fullcalendar-poc -- frontend/src/components/schedulingHub/calendar/types.ts frontend/src/components/schedulingHub/calendar/index.ts`. These are dependency-free and unlock S2.6.
3. Add FC license key env var (`VITE_FULLCALENDAR_LICENSE`) — GPL-compatible key for dev; commercial key swapped before SX.1.
4. PR is `yarn install` + the two type files + env var stub only. No runtime code yet.

**Done-when:** `yarn build` succeeds; `calendar/types.ts` + `calendar/index.ts` importable; no runtime FC instance yet.
**Depends-on:** S0.1 (procurement parallel, GPL key OK for dev)
**POC source:** `frontend/package.json`, `frontend/src/components/schedulingHub/calendar/{types,index}.ts`

### S0.6 V2 shell + theming (POC: PORT — small)
1. Salvage POC root + toolbar + mappers + barrel:
   - `git checkout CH/fullcalendar-poc -- frontend/src/components/schedulingHub/{SchedulingHub.tsx,SchedulingHubToolbar.tsx,mappers.ts,index.ts,scheduling-hub-overrides.css}`
2. Port `scheduling-hub-overrides.css`: replace 5 hex literals (`#F44336`, `#fafbfc`, `#64748b`, `#70778b`, `#fff`) with `tokens.color.*` references from `contexts/theme/stride-v2`.
3. Port `SchedulingHub.tsx`: the single `api/appointments.getAppointmentTypes()` axios call → `useQuery` from `api/schedulingHub/queries.ts` (stub returns `[]` until S0.8 lands real query).
4. Hub root wraps in `<ThemeProvider theme={strideThemeV2}>` (POC already does this — verify after salvage).
5. No business logic yet beyond what POC ships.

**Done-when:** Staff loads `/calendar?scheduling_hub=true` on demo → sees themed v2 shell, no console errors; lint passes (no hex in CSS).
**Depends-on:** S0.4, S0.5
**POC source:** `components/schedulingHub/{SchedulingHub.tsx,SchedulingHubToolbar.tsx,mappers.ts,scheduling-hub-overrides.css,index.ts}`

### S0.7 V2 Zustand store (UI-only)
1. Create `stores/schedulingHub.ts` for client UI state: view mode, slot duration, selected location/therapist/team/admin, bulk-mode set, dialog open registry, polling-pause registry.
2. **Do not** cache appointment/availability data here — that lives in TanStack Query.
3. Use `useShallow` patterns in selectors.

**Done-when:** Store exists with tests for state transitions; consumer hooks compile.
**Depends-on:** S0.6

### S0.8 Query hook layer skeleton (POC: PORT — medium)
1. Create `api/schedulingHub/queries.ts` for read hooks (`useSchedulingHubAppointments`, `useSchedulingHubAppointmentTypes`, `useSchedulingHubAvailabilities`, `useSchedulingHubCalendarSettings`) with sensible `staleTime` per data-fetching rules. `refetchOnWindowFocus: true`; `refetchInterval: 30_000`.
2. Create `api/schedulingHub/mutations.ts` (`useCreateAppointment`, `useEditAppointment`, `useDeleteAppointment`, `useGetAppointmentDetails`).
3. **Port `useSchedulingHub.ts` from POC**: replace direct axios CRUD (`getAppointments`/`addAppointment`/`editAppointment`/`deleteAppointment`/`getAppointmentDetails`, lines 9–13) with the mutation/query hooks. Drop manual `fetchIdRef` cancellation (RQ handles it).
4. Verify global `QueryErrorHandler` is already mounted in `App.tsx`.

**Done-when:** Hub fetches appointments via RQ; opening tooltip / creating / editing all route through mutations; no axios in `components/schedulingHub/**`.
**Depends-on:** S0.7
**POC source:** `components/schedulingHub/useSchedulingHub.ts` (logic preserved, network layer replaced)

### S0.9 Backoffice admin UI for `SCHEDULING_HUB` toggle
1. New view under `backend/backoffice/` listing practices and exposing toggle for `SCHEDULING_HUB.visible` plus each sub-flag (`smartScheduling`, `bulkOps`, `rooms`, `teams`, `search`).
2. Every write audit-logged (who flipped what, when, for which practice).
3. Permission: Stride internal staff only (existing backoffice permission class).

**Done-when:** Stride staff can flip per-practice flag + sub-flags from backoffice without DB access; audit-log row written.
**Depends-on:** S0.2

### S0.10 Backend `feature_flags.py` helper
1. New module `backend/utils/feature_flags.py` with `scheduling_hub_enabled(request) -> bool`.
2. Logic: `request.user.is_staff AND request.query_params.get('scheduling_hub','').lower()=='true'` OR `AppSetting.objects.filter(name='SCHEDULING_HUB', practice=request.practice).first().value.get('visible')`.
3. Add `scheduling_hub_feature(request, name) -> bool` for sub-flag checks.
4. Unit tests covering staff/non-staff + with/without setting + each sub-flag.

**Done-when:** Helper importable from any view; unit tests pass.
**Depends-on:** S0.2

### S0.11 Saved-view schema versioning + round-trip test
1. Update `stores/schedulingHub.ts` URL-sync (and `userProfile.defaultCalendarUrlParams` writer) to emit `{v: 2, ...}` discriminator.
2. V1 readers ignore unknown `v` (verify existing V1 deserialization is tolerant; add defensive check if not).
3. Vitest unit test: V2 write → V2 read round-trip preserves all state; V2 blob read by V1 deserializer does **not** crash (returns null or default).
4. Vitest test: V1 blob read by V2 deserializer gracefully returns defaults.

**Done-when:** Tests pass; staff can toggle V1↔V2 without saved-view corruption.
**Depends-on:** S0.7

### S0.12 Staff-gate ghost mode (FE + BE)
1. FE: `useSchedulingHubEnabled` returns `ghostMode: true` only when `appContext.state.user.isStaff === true` and `?scheduling_hub=true` present.
2. BE: `scheduling_hub_enabled(request)` (S0.10) gates query-param branch on `request.user.is_staff`.
3. Add Vitest case to S0.3's suite covering non-staff with query param → `enabled: false`.

**Done-when:** Non-staff PracticeAdmin loading `/calendar?scheduling_hub=true` sees V1. Staff loading the same sees V2.
**Depends-on:** S0.3, S0.10

### S0.13 Pause-source registry
1. Extend `stores/schedulingHub.ts` with `pauseSources: Set<string>`, `addPauseSource(id)`, `removePauseSource(id)`, `isPaused()`.
2. Wire consumers: tooltip open/close, modal open/close, drag-confirm dialog, cancel-reason dialog, bulk-mode active.
3. `useSchedulingHubAppointments`'s `refetchInterval` returns `false` when `isPaused()` true.

**Done-when:** Opening any of {tooltip, modal, drag dialog, bulk mode} suspends 30s polling; closing resumes.
**Depends-on:** S0.7, S0.8

### S0.14 Recurring-expansion authority decision
1. Document in `components/schedulingHub/utils/fcAdapters.ts` header: BE remains authoritative; FC renders flat instances; do **not** enable FC native `rrule` plugin.
2. Add ESLint custom rule or grep-CI guard forbidding imports from `@fullcalendar/rrule`.

**Done-when:** Header doc present; CI guard catches a deliberate offending import in a smoke commit.
**Depends-on:** S0.5

### S0.15 Timezone adapter (`utils/timezone.ts`) + DST tests
1. Build `components/schedulingHub/utils/timezone.ts` for UTC ↔ practice-TZ conversion used by `fcAdapters`.
2. Use `date-fns-tz` only; no moment.
3. Unit tests cover spring-forward / fall-back transitions, midnight boundary, cross-TZ practice scenarios.

**Done-when:** Tests pass; helper exported and consumed by S0.8 query hooks.
**Depends-on:** S0.5

---

## S1 — General

### S1.1 Refresh appts on tab switch (existing)
Port the window-focus refetch behavior from `useFetchCalendarData`. With react-query, set `refetchOnWindowFocus: true` on the appointments query and respect the pause registry from S0.7.
**Done-when:** Window blur → focus triggers refetch unless pause registry is non-empty.
**Depends-on:** S0.8

### S1.2 30s polling (existing)
Use `refetchInterval` on `useSchedulingHubAppointments`. Honor pause registry (return `false` when paused).
**Done-when:** Network shows requests every 30s; pause registry suspends correctly.
**Depends-on:** S0.8

### S1.3 No double-booking same case same day (existing)
Port validation from current `useDragEdit` / form submit. Centralize in `utils/validation/appointmentOverlap.ts`. Pure function, fully unit-testable.
**Done-when:** Util has unit tests covering same-case-same-day, different-case, different-day; integrated into Create/Edit submit.
**Depends-on:** S0.7

### S1.4 Time snapping side-by-side (existing)
Document current DX behavior, implement FC equivalent (config `slotLabelInterval`, `slotDuration`, custom event positioning) so simultaneous appointments stack side-by-side.
**Done-when:** Two overlapping appts render side-by-side as in v1.
**Depends-on:** S0.5

### S1.5 Mobile friendly (existing)
Apply responsive breakpoints to FilterBar / Sidebar / Calendar layout. Test at 375px, 768px.
**Done-when:** Day view usable on iPad-portrait; FilterBar collapses overflow to popover on narrow widths.
**Depends-on:** S0.6

### S1.6 Theme-only colors (no hardcoded) — guardrail
Lint rule or PR-review checklist forbidding hex colors in `components/schedulingHub/**`. All colors via theme palette / `tokens` / user-configured payment colors.
**Done-when:** ESLint config or pre-commit check; review of merged v2 PRs shows no hex literals.
**Depends-on:** —

### S1.7 Team management (w/ backend)
1. Backend: new `Team` model under tenant-scoped app (probably `api/practice/teams/`). Fields: name, members (M2M to clinicians), color?, ordering.
2. ViewSet + serializer + URL route.
3. Frontend: `api/teams/queries.ts` + `mutations.ts`; TeamManageDialog component per PRD §30.
4. Sidebar Team dropdown reads from query.

**Done-when:** Practice admin can CRUD teams via dialog; teams persist; backend tests pass.
**Depends-on:** S0.7, S3.1

---

## S2 — Schedule View

### S2.1 Today / Prev / Next (existing)
FC API `getApi().today() / .prev() / .next()`. Toolbar buttons in FilterBar.
**Done-when:** Buttons advance/retreat current view; Today returns to today regardless of view.
**Depends-on:** S0.5, S0.6

### S2.2 Day/Week view selector (existing)
View modes: `resourceTimeGridDay`, `resourceTimeGridWeek`, `resourceTimeGridWorkWeek`. Persist current view in store.
**Done-when:** Selector switches without remount thrash; current date preserved.
**Depends-on:** S2.1

### S2.3 Slot-duration selector (existing)
Options 5/10/15/30/60 min. Wire FC `slotDuration` prop. Persist in store + survive view-switch.
**Done-when:** Duration changes re-grid the calendar; user pref persists in saved-view payload.
**Depends-on:** S2.2

### S2.4 Hide cancelled appts toggle (existing)
Filter at query-hook level OR client-side filter on rendered events (decide for perf). Persist toggle.
**Done-when:** Toggle visibly hides/shows cancelled blocks.
**Depends-on:** S2.6

### S2.5 Save current view (existing)
Reuse `defaultCalendarUrlParams` write on user profile. Floppy-disk icon in FilterBar serializes current store slice to user-profile field via existing mutation.
**Done-when:** Click save → reload → calendar restores view/filters/duration.
**Depends-on:** S2.2, S2.3, S3.x as relevant

### S2.6 Render appointments (existing) (POC: PORT — small)
1. Salvage adapter: `git checkout CH/fullcalendar-poc -- frontend/src/components/schedulingHub/calendar/FullCalendarAdapter.tsx`
2. Port: replace inline styles on `eventContent` renderer and `resourceLabelContent` with `sx` prop. Keep FC imports + imperative ref API + `queueMicrotask` patterns.
3. Map `IAppointment[]` from `useSchedulingHubAppointments` (S0.8) through `mappers.ts` (S0.6) to FC `EventInput[]`. Resource = clinician/admin per view mode. Hand off to `<AppointmentBlock>` via `eventContent` (S5.1).

**Done-when:** Day view renders all current appts at correct times on correct resource columns; lint clean (no inline color literals).
**Depends-on:** S0.8, S5.1
**POC source:** `components/schedulingHub/calendar/FullCalendarAdapter.tsx`

### S2.7 Drag & drop appointment (existing)
Wire FC `editable`, `eventDrop`, `eventResize`. On drop, open DragConfirmDialog (PRD §23) with old→new diff. Reuse overlap validation from S1.3. On confirm, fire PATCH; on cancel, revert.
**Done-when:** Drag to new slot/column → diff dialog → confirm persists; cancel reverts.
**Depends-on:** S2.6, S1.3, S2.10

### S2.8 IE/Total appt header (existing)
Compute from currently-rendered events; render in FilterBar.
**Done-when:** Counts update on filter/view change.
**Depends-on:** S2.6

### S2.9 Now-indicator (existing)
Enable FC `nowIndicator`. Override CSS so line spans all columns (PRD §3) and color = indigo brand.
**Done-when:** Red/indigo line spans full grid; updates each minute.
**Depends-on:** S2.6

### S2.10 DragConfirmDialog (existing pattern, new component)
Build dialog showing diff. Reuse `CancelAppointmentModal` reason pattern only where relevant.
**Done-when:** Storybook story shows old→new diff; confirm/cancel handlers wired.
**Depends-on:** S0.6

### S2.11 Availability rendering on calendar (existing)
Port `useCellAvailability` logic. FC equivalent: `businessHours` + background events for PTO/closures.
**Done-when:** Available windows shaded white; unavailable grey; PTO red; closures slate — matches v1.
**Depends-on:** S2.6

### S2.12 My Calendar saved-view toggle (existing)
Toggle in FilterBar that filters to `selectedTherapistId = currentUserId`. Saved per-user.
**Done-when:** Toggle on → single-resource view of current user.
**Depends-on:** S2.5

### S2.13 Validate appt on inline change (no sidebar) (existing)
Wire validation chain on inline status change / drag / resize — same authorization/overlap checks as in Create/Edit.
**Done-when:** Inline status change to `CHECKED_IN` runs auth/payer checks; errors surface as toast.
**Depends-on:** S1.3, S6.x auth ticket

### S2.14 Download schedule (w/o backend)
Generate CSV/PDF client-side from current visible events (date range + filters). Add toolbar action.
**Done-when:** Click → file downloads with correct rows.
**Depends-on:** S2.6

### S2.15 Show clinician location in header (w/o backend)
Per-resource column header shows location chip from `clinician.primaryLocation`.
**Done-when:** Each column header shows location text.
**Depends-on:** S2.6

### S2.16 Show clinician credentials in header (w/o backend)
Per-resource column header shows credentials suffix (PT, DPT, OTR/L) from clinician entity.
**Done-when:** Headers show name + credentials.
**Depends-on:** S2.6

### S2.17 Total appt count in current view (w/o backend)
Already covered by S2.8 if we expose total separately.
**Done-when:** Total count chip visible in FilterBar.
**Depends-on:** S2.8

### S2.18 Legend (w/o backend)
Popover or inline legend mapping category accent + payer badge + status icon. PRD §3 triple encoding.
**Done-when:** Legend reachable from FilterBar overflow; matches actual rendered colors.
**Depends-on:** S5.x payer badge ticket

### S2.19 Keybinds (w/o backend)
Implement shortcuts: ⌘K (palette), ⌘B (sidebar), arrow keys (prev/next), `T` (today), `D/W` (view). Single `useKeyboardShortcuts` hook.
**Done-when:** Each shortcut fires expected action; non-conflicting with browser.
**Depends-on:** S2.1, S3.2

### S2.20 Dynamic business hours view (w/ backend)
Pull clinic hours per location from new `ClinicHours` model (or existing if any). Apply to FC `businessHours` prop per location.
**Done-when:** Switching location updates grid's shaded business-hours window.
**Depends-on:** S2.11, BE: ClinicHours endpoint

### S2.21 Rooms (w/ backend)
1. Backend: `Room` model + viewset under `api/practice/rooms/`. Fields: name, location, capacity, active.
2. Frontend: query hook; render room columns as alternate resource axis (toggle in FilterBar: clinician vs room).
3. AppointmentBlock displays room chip.

**Done-when:** Room view shows rooms as resource columns; appointments slot under their room.
**Depends-on:** S2.6

### S2.22 Search (w/ backend)
Build/use practice-wide patient + appointment search endpoint. Surface via Command Palette (⌘K, PRD §13).
**Done-when:** Search returns hits; selecting one navigates calendar to that appt's day/resource.
**Depends-on:** S2.19, BE: search endpoint

### S2.23 View Selector — location/clinician/team/admin (w/ backend)
PRD's `calendarViewMode` dropdown. byLocation / byTherapist / byTeam / byAdmin. Per-mode resource list source.
**Done-when:** Each mode renders correct resource columns; selection persists in saved view.
**Depends-on:** S1.7, S2.21, S3.1

### S2.24 Bulk Cancel (w/ backend)
1. Toolbar bulk-mode toggle (icon `LibraryAddCheckOutlinedIcon`).
2. Click blocks adds to `bulkSelectedIds`.
3. "Cancel" action → CancelReasonDialog (S2.25) with bulk count → PATCH N at once (new endpoint or N-loop with single reason).

**Done-when:** Bulk-cancel of 3 appts updates all 3 with same reason; cancellations log audit row.
**Depends-on:** S2.25, BE: bulk-update endpoint

### S2.25 CancelReasonDialog (existing pattern)
Port from current `CancelAppointmentModal`. Chip list + optional note + "Other" requires note.
**Done-when:** Single + bulk both invoke; reason persists; dialog enforces "Other → note".
**Depends-on:** S0.6

### S2.26 Bulk Editing (w/ backend)
Status changes (Unmarked, Checked In) over selected set, no reason dialog. New bulk endpoint or N-loop PATCH.
**Done-when:** Select N → "Checked In" updates all N.
**Depends-on:** S2.24

---

## S3 — Sidebar

### S3.1 Sidebar shell + open/close (existing)
PRD §12 drawer pattern. Toggle via FilterBar arrow + ⌘B. Persist collapsed state to user profile saved view.
**Done-when:** Drawer slides; state survives reload.
**Depends-on:** S0.6

### S3.2 Filter — All Locations (existing)
Multi-select locations list. Wires `selectedLocationIds` in store.
**Done-when:** Toggling locations filters resource columns + events.
**Depends-on:** S3.1, S2.6

### S3.3 Drag-drop clinician order (existing, w/ backend)
Reuse `@dnd-kit/sortable` pattern. Persist order to user profile (new field or part of saved-view JSON).
**Done-when:** Reorder persists across reload; resource columns reflect order.
**Depends-on:** S3.1, BE: persist order

### S3.4 Filter — Admin Staff (existing)
List of admin workers when in `byAdmin` view mode.
**Done-when:** Selection filters admin column visibility.
**Depends-on:** S3.1, S2.23

### S3.5 Drag-drop admin worker order (w/ backend)
Same dnd-kit pattern; persist to user profile.
**Done-when:** Reorder persists.
**Depends-on:** S3.4

### S3.6 Sidebar filter (search-within) (w/ backend)
Text input that filters visible clinicians/admins/locations in the drawer. If practice has >N clinicians, may need server-side fuzzy search.
**Done-when:** Typing narrows the list; clear restores full list.
**Depends-on:** S3.1

### S3.7 Hide unavailable therapists (w/ backend)
Toggle that removes from sidebar list any therapist with no availability windows on current date range. Requires availability query to expose "has any availability today" flag.
**Done-when:** Toggle on → unavailable clinicians disappear from resource columns.
**Depends-on:** S3.1, S2.11

---

## S4 — Settings

### S4.1 Appt color preference (existing)
Per-practice or per-user color mapping for service categories. Read existing setting; surface in v2 SettingsPage (PRD §14).
**Done-when:** Saved color appears on appointment block accents.
**Depends-on:** S5.x category accent

### S4.2 Default appt duration (existing)
Existing setting → wire to CreateDialog prefill.
**Done-when:** New appts default to setting's duration.
**Depends-on:** S7.x CreateDialog

### S4.3 Event type creation (existing)
Existing flow; surface entry point in v2 SettingsPage.
**Done-when:** Event types CRUD reachable from v2 settings.
**Depends-on:** —

### S4.4 Payment type colors (existing)
Existing setting → wires to payer badge color on AppointmentBlock.
**Done-when:** Payer badge color = user-configured value.
**Depends-on:** S5.x payer badge

### S4.5 Service-type restricted no-double-booking (w/ backend)
Setting that disallows overlap of two appts with the same service type for one clinician.
1. BE: add to OverlapSettings model/JSON; enforce in validator.
2. FE: toggle in SettingsPage; client-side check mirrors BE.

**Done-when:** With setting on, overlap of same service-type blocks save with error; off allows.
**Depends-on:** S1.3

### S4.6 Booking-in-unavailable-slots setting (w/ backend)
Existing `wb-allow-booking-unavailable` in PRD localStorage → move to per-practice AppSetting or OverlapSettings.
**Done-when:** Toggle persisted server-side; Create/Edit respects.
**Depends-on:** S4.5

### S4.7 Max concurrent appts (w/ backend)
Per-clinician cap on overlapping appts (e.g. 2 = double-booking allowed; 3+ blocked).
1. BE: field on OverlapSettings; enforced.
2. FE: setting input + Create/Edit validation.

**Done-when:** Try to book 3rd overlap when cap=2 → blocked with clear error.
**Depends-on:** S4.5

### S4.8 Payer-restricted no-double-booking (w/ backend)
Same shape as S4.5 but per-payer.
**Done-when:** Blocks overlap by same payer when setting on.
**Depends-on:** S4.5

---

## S5 — Appointment Block

### S5.1 Block shell (existing) (POC: KEEP)
1. Salvage: `git checkout CH/fullcalendar-poc -- frontend/src/components/schedulingHub/AppointmentBlock.tsx frontend/src/components/schedulingHub/AppointmentBlock.stories.tsx`
2. Verify `hexToRgba()` util on POC resolves on current main (or replace with `theme.palette.alpha()` if missing).
3. Wire as FC `eventContent` renderer (S2.6). Patient name dominant per PRD §3.

**Done-when:** Block displays patient name; ResizeObserver toggles `isNarrow`/`isShort`; Storybook story renders.
**Depends-on:** S2.6
**POC source:** `components/schedulingHub/AppointmentBlock.{tsx,stories.tsx}`

### S5.2 Tags (existing)
Render existing appointment tags as chips/icons in icon row.
**Done-when:** Tags render; truncate per density.
**Depends-on:** S5.1

### S5.3 Appt status icon (existing)
Right-edge icon by status (None/Confirmed/Checked In/Cancelled/Late/No Show). Click → status menu.
**Done-when:** Icon reflects status; menu changes it (with reason dialog as needed).
**Depends-on:** S5.1, S2.25

### S5.4 Appt resize (existing)
FC `eventResize`. Reuse DragConfirmDialog (kind=duration).
**Done-when:** Drag bottom edge → confirm dialog → persists duration.
**Depends-on:** S2.7

### S5.5 Block badges — service category accent + payer badge (w/o backend)
PRD §3 triple encoding. Service category drives left accent stripe; payer drives badge color.
**Done-when:** Three appt of different categories/payers render distinguishable.
**Depends-on:** S5.1

### S5.6 Inline notes preview (w/o backend)
Show first ~40 chars of appointment note inline when block tall enough.
**Done-when:** Tall block shows note snippet; short block drops it.
**Depends-on:** S5.1

### S5.7 Responsive icon list (w/o backend)
Icon row: payer, note status, auth flag, payment status. Order by priority; drop low-priority icons in narrow blocks.
**Done-when:** Block at 60px wide drops icons in order; >120px shows all.
**Depends-on:** S5.1

### S5.8 Responsive rendering (w/o backend)
ResizeObserver toggling display modes (long/short/narrow). Drop icon row before name per dominance rule.
**Done-when:** 15-min slot still shows readable name.
**Depends-on:** S5.1

### S5.9 Check-in patient from corner (w/o backend)
Small corner action on block: click checkmark → status=CHECKED_IN with auth validation (S2.13).
**Done-when:** One-click check-in from block; respects validation.
**Depends-on:** S5.3, S2.13

### S5.10 Visit count badge (w/ backend)
Pull `visitsUsed/visitsTotal` from authorization data; render as `3/12` badge.
**Done-when:** Badge present and correct for appts with auth.
**Depends-on:** S5.5, BE: auth fields in appointment serializer

### S5.11 Secondary insurance badge (w/ backend)
Render mini-badge when appt has secondary payer.
**Done-when:** Badge shows on secondary-payer appts only.
**Depends-on:** S5.5, BE: secondary payer field exposed

### S5.12 Tooltip on hover (w/ backend)
FC `eventMouseEnter` → after 320ms, show `<AppointmentTooltip>`. Pre-fetch tooltip data with react-query (`useAppointmentDetails(id)`).
**Done-when:** Hover ≥320ms → tooltip; leave → dismiss.
**Depends-on:** S6.x tooltip ticket

### S5.13 "Co" badge if user is co-treater (w/ backend)
Read appointment's `coTreaters[]`; if currentUser in list, show "Co" badge.
**Done-when:** Badge appears only for current user when they're co-treater.
**Depends-on:** S5.5, BE: co-treater field

---

## S6 — Appointment Tooltip

### S6.1 Tooltip shell (existing) (POC: KEEP)
1. Salvage: `git checkout CH/fullcalendar-poc -- frontend/src/components/schedulingHub/AppointmentDetailDialog/`
2. POC ships `AppointmentDetailDialog.tsx`, `AppointmentDetailDialog.stories.tsx`, `DetailContent.tsx`, `DetailRow.tsx`, `constants.ts`, `types.ts`, `index.ts` — all KEEP-clean per salvage matrix.
3. Hub renders this on event click (S0.1 wiring). Section split (`Patient`/`Auth`/`Insurance`/`Charges`/`Warnings`/`Actions` per S6.17) is layered on later via additional section components in the same folder; POC's `DetailContent`/`DetailRow` are the starting pattern.

**Done-when:** Click event → dialog shows core fields; Storybook story renders.
**Depends-on:** S5.12
**POC source:** `components/schedulingHub/AppointmentDetailDialog/*`

### S6.2 Click view appointment (existing)
Tooltip "View" → opens `AppointmentModal` (PRD §18).
**Done-when:** Click opens modal with full data.
**Depends-on:** S6.1, S7.x modal

### S6.3 Copy phone/email + download schedule (existing)
Buttons copy to clipboard; download schedule = patient's upcoming appts CSV/PDF.
**Done-when:** Copy fills clipboard; download produces file.
**Depends-on:** S6.1

### S6.4 Copy appointment (existing)
Clone appointment to a future slot. Opens CreateDialog prefilled.
**Done-when:** Copy → CreateDialog opens with same patient/case/service.
**Depends-on:** S6.1, S7.x CreateDialog

### S6.5 Delete appointment (existing)
Confirm dialog → DELETE.
**Done-when:** Confirmed delete removes from calendar.
**Depends-on:** S6.1

### S6.6 Cancellation details modal (existing)
View existing cancellation reason if appt cancelled.
**Done-when:** Cancelled appt's tooltip exposes "View cancellation reason" → modal shows code + note.
**Depends-on:** S6.1, S2.25

### S6.7 Change status (existing)
Status radio/menu, with cancellation routed through reason dialog.
**Done-when:** Any status change persists; cancellations always capture reason.
**Depends-on:** S6.1, S2.25

### S6.8 Collect payment button + flow (existing)
Existing payment flow wired into tooltip.
**Done-when:** Button opens existing payment modal; flow ends in updated payment status.
**Depends-on:** S6.1

### S6.9 Payment modal after check-in (existing)
On status→CHECKED_IN, if balance owed, auto-open payment modal.
**Done-when:** Check-in with outstanding balance → modal appears.
**Depends-on:** S6.7, S6.8

### S6.10 Download upcoming appointments (w/o backend)
Patient's future appts → CSV/PDF.
**Done-when:** Action produces correct file for the patient.
**Depends-on:** S6.3

### S6.11 Open flowsheet from case (w/o backend)
Link to flowsheet page for the case.
**Done-when:** Click opens flowsheet in new tab.
**Depends-on:** S6.1

### S6.12 Send SMS to patient (w/o backend)
Open existing SMS modal prefilled with patient phone.
**Done-when:** Modal opens with phone prefilled.
**Depends-on:** S6.1

### S6.13 Scheduling warnings — cert expiring, progress note due (w/ backend)
BE exposes warning flags on appointment serializer (cert expiration date, progress-note-due flag). Tooltip renders alert section.
**Done-when:** Tooltip shows warnings only when flags present; clinically accurate.
**Depends-on:** S6.1, BE: warning fields

### S6.14 Authorization info panel (w/ backend)
Visits used/total, expiration, KX threshold, Workers' Comp written-auth status. Inline.
**Done-when:** Authorization panel renders correct values per payer type.
**Depends-on:** S6.1, BE: auth fields

### S6.15 Patient emailing option for files (w/ backend)
Send files from case to patient via existing email infra.
**Done-when:** Files sent; backend logs delivery.
**Depends-on:** S6.1, BE: email endpoint

### S6.16 DOB, diagnosis, insurance member ID (w/ backend)
Surface these existing fields in tooltip.
**Done-when:** Tooltip shows three fields when present.
**Depends-on:** S6.1, BE: serializer exposes

### S6.17 Recompose AppointmentTooltip into sections
1. V1's `components/shared/Scheduler/AppointmentTooltip.tsx` is a 50KB monolith. V2 splits into props-only section components: `Patient.tsx`, `Auth.tsx`, `Insurance.tsx`, `Charges.tsx`, `Warnings.tsx`, `Actions.tsx`.
2. Root `AppointmentTooltip.tsx` composes sections; data flows via props.
3. No `tss-react`; `sx` / `styled` only.

**Done-when:** Sections each have a Storybook story (`SQ.2`); root tooltip renders same content as V1 with no regressions.
**Depends-on:** S6.1

---

## S7 — Appointment Create / Edit

### S7.1 CreateDialog shell (existing) (POC: PORT — large)
1. Salvage tree: `git checkout CH/fullcalendar-poc -- frontend/src/components/schedulingHub/CreateAppointmentDialog/`
2. POC ships `CreateAppointmentDialog.tsx`, `FormContent.tsx` (already uses RHF V2 — `RHFSelect`, `RHFTextField`, `RHFDateTimeField`, `RHFAutocomplete`), `useAppointmentForm.ts`, `constants.ts`, `types.ts`, `index.ts`.
3. **Port `useAppointmentForm.ts`** (the meaty one): replace direct `searchPatients`/`getPatient` axios calls (lines 6–10) with `useQuery` hooks; the manual `patientSearch`/`patientOptions`/`patientLoading` state collapses into `useQuery` result shape. Keep `useDebounce` for typing. Extract case-loading effect (lines 116–131) into its own `useCases(patientId)` hook.
4. Dialog renders as right-side drawer per PRD §19.

**Done-when:** Open via "+" button → form renders; typing in patient field hits `searchPatients` via RQ (cached/debounced); selecting patient loads cases via RQ; submitting creates appt via `useCreateAppointment` mutation (S0.8).
**Depends-on:** S0.6, S0.8
**POC source:** `components/schedulingHub/CreateAppointmentDialog/*`

### S7.2 Auto-select location from clicked spot (existing)
Double-click → CreateDialog prefilled with the slot's date/time/clinician/location.
**Done-when:** Click empty 9:00 slot on Therapist A → form has 9:00, today, Therapist A, A's location.
**Depends-on:** S7.1, S2.6

### S7.3 Double-click to create (existing)
FC `dateClick` (with double-click detection) → S7.2 flow.
**Done-when:** Two quick clicks → dialog opens prefilled.
**Depends-on:** S7.2

### S7.4 Attendee availability warning (existing)
On chosen time, check selected clinician/attendees' availability; show inline warning if outside their windows.
**Done-when:** Picking 6am for Therapist A (8–5 hours) shows warning.
**Depends-on:** S7.1, S2.11

### S7.5 Add from waitlist on create (existing)
"Add from waitlist" → list of matching waitlist items → click to prefill form from item.
**Done-when:** Selecting a waitlist item populates patient/case/service/preferences.
**Depends-on:** S7.1, S8.x

### S7.6 CreateEvent sidebar (existing)
Toggle CreateDialog into event mode. Different fields per PRD §19 event variant.
**Done-when:** Toggle works; event creates with correct shape.
**Depends-on:** S7.1

### S7.7 New patient from create (existing)
"New patient" link → inline patient-create flow → returns to CreateDialog with new patient pre-selected.
**Done-when:** Create new patient → land back in dialog.
**Depends-on:** S7.1

### S7.8 Additional attendees (existing)
Add co-treaters / multi-clinician.
**Done-when:** Multiple clinicians visible on appt block (S5.13 "Co" badge correct).
**Depends-on:** S7.1

### S7.9 Repeat / recurring create (existing)
Recurrence options (daily/weekly/etc.). Generates N appts.
**Done-when:** Weekly repeat × 4 → 4 appts created.
**Depends-on:** S7.1

### S7.10 Double-click edit appt (existing)
Double-click block → AppointmentModal → "Edit" → CreateDialog prefilled in edit mode. Cancel returns to modal (PRD §3 reopen-modal pattern).
**Done-when:** Edit cycle preserves modal context; save updates appt.
**Depends-on:** S7.1, S6.2

### S7.11 Booking Wizard (w/o backend) — disabled per PRD
PRD §20 says wizard exists but is disabled. Keep disabled in v2 too — but document entry point for future ticket.
**Done-when:** No active wizard in UI; design doc captures decision.
**Depends-on:** —

### S7.12 Case auto-select (w/o backend)
On patient pick, auto-select the patient's first active case.
**Done-when:** Patient with one active case → case prefilled.
**Depends-on:** S7.1

### S7.13 Events can have additional attendees (w/o backend)
Lift attendee multi-select into event mode.
**Done-when:** Event with 3 attendees renders on all 3 columns.
**Depends-on:** S7.6, S7.8

### S7.14 Block scheduling based on availability (w/ backend)
Hard block (not just warn) when chosen time outside availability AND `allowBookingInUnavailable=false`.
**Done-when:** Save disabled with error when out-of-window and setting off.
**Depends-on:** S4.6, S7.4

---

## S8 — Waitlist

### S8.1 View scheduled (online-scheduling queue) (existing)
Reuse `api/waitlist/pendingRequests.ts`. WaitlistPanel "Patient Scheduled" tab (PRD §31).
**Done-when:** Tab lists pending requests.
**Depends-on:** S0.6

### S8.2 View waitlist active/expired (existing)
Tabs Active/Expired filtered from `getWaitlistItems`.
**Done-when:** Tab switch filters list.
**Depends-on:** S8.1

### S8.3 Approve/decline scheduled (existing)
Approve → materializes Appointment via patient's first active case; decline removes row. Reuse existing mutation.
**Done-when:** Approve creates appt visible on calendar.
**Depends-on:** S8.1

### S8.4 New waitlist item (existing)
WaitlistDialog (PRD §32) — RHF V2.
**Done-when:** Create persists; appears in Active list.
**Depends-on:** S8.2

### S8.5 Approve waitlist (existing)
From active item → open Create prefilled (S7.5).
**Done-when:** Approve flow ends with new appt created and waitlist item resolved.
**Depends-on:** S8.4, S7.5

---

## S9 — Smart Scheduling

### S9.1 Smart Scheduling button (w/o backend)
Toolbar icon in FilterBar. Opens SmartSchedulingDialog (PRD §21).
**Done-when:** Click opens empty dialog.
**Depends-on:** S0.6

### S9.2 SmartSchedulingDialog shell (w/o backend)
Inputs: duration, location filter, date tabs (5 business days). Therapist cards with time pills.
**Done-when:** Dialog renders inputs; date tabs cycle 5 days.
**Depends-on:** S9.1

### S9.3 Pre-filter based on existing appt (w/o backend)
When opened in rebook flow, prefill duration / location / service from existing appt.
**Done-when:** Smart Rebook (S9.5) opens with prefilled filters.
**Depends-on:** S9.2

### S9.4 Rebook on bad status (w/o backend)
Cancelled/No-Show status → option to "Smart Rebook". Opens SmartRebookDialog (PRD §22) with prefill.
**Done-when:** From a cancelled appt's status menu → Smart Rebook → dialog prefilled.
**Depends-on:** S9.3

### S9.5 SmartRebookDialog (w/o backend)
Picks date+clinician+slot → chains into CreateDialog for confirmation (PRD §22).
**Done-when:** End-to-end rebook produces new appt; original cancelled state preserved.
**Depends-on:** S9.4, S7.10

### S9.6 Practice-facing availability ranking endpoint (w/ backend)
Backend endpoint returning ranked open slots given filters (duration, locations, service type, date range). Heuristic only (no AI per scope decision).
1. BE: `POST /api/availability/search` returning `[{clinicianId, start, end, score}]`. Score = simple weighted (earliest first, balanced clinician load).
2. FE: query hook + integrate into S9.2 cards.

**Done-when:** Dialog populates clinician cards from endpoint; results sortable by earliest/current-therapist.
**Depends-on:** S9.2

---

## SQ — Quality (cross-cutting)

### SQ.1 Playwright E2E parity suite
1. Create `frontend/tests/e2e/calendar-v2/` with one happy-path test per category: toggle (V1↔V2), view switch, drag-drop confirm, create, edit, cancel-with-reason, bulk cancel, smart schedule, waitlist approve.
2. Suite gated by staff impersonation + `?scheduling_hub=true`.

**Done-when:** Suite green in CI; runs in <8 min wall-clock.
**Depends-on:** Per category as it lands.

### SQ.2 Storybook stories for V2 atomic components (POC: partial — KEEP)
1. POC already ships: `AppointmentBlock.stories.tsx`, `AppointmentDetailDialog.stories.tsx`, `ScheduleCalendar.stories.tsx` (salvaged in S5.1 / S6.1 / S0.6).
2. Add new stories for components built fresh: badge primitives, each tooltip section (S6.17), FilterBar, sidebar `StaffList`, `Legend`, `KeybindHelp`, `DragConfirmDialog`, `CancelReasonDialog`, `SmartSchedulingDialog`, `SmartRebookDialog`.

**Done-when:** Every new V2 atomic has a `*.stories.tsx`; Chromatic (or local Storybook build) clean.
**Depends-on:** Per component as it lands.

### SQ.3 a11y audit (axe pass)
1. Run `@axe-core/playwright` on calendar route under both V1 and V2.
2. Document and fix any serious/critical violations on V2.

**Done-when:** Zero serious/critical axe violations on V2 route.
**Depends-on:** S0.6; re-run as features land.

### SQ.4 Performance budget + CI bundle-size guard
1. Measure baseline V1 chunk size and FCP/TBT on `/calendar`.
2. Add CI step that fails PR if V2 chunk size exceeds baseline + 150KB gz.
3. Budget: V2 FCP within ±10% of V1; smart-scheduling endpoint ≤500ms p95 (20 therapists × 14 days).

**Done-when:** CI guard active; smart-scheduling load test in place.
**Depends-on:** S0.5, S9.6

### SQ.5 Telemetry events
1. Emit `scheduling_hub.session_start` on V2 mount (via existing analytics util).
2. Emit `scheduling_hub.feature_used.{name}` for key actions: `drag`, `create`, `edit`, `cancel`, `bulk_cancel`, `bulk_edit`, `search`, `smart_schedule`, `smart_rebook`, `save_view`, `waitlist_approve`.
3. Include `ghostMode: boolean` and sub-flag state in event metadata.

**Done-when:** Events visible in analytics dashboard during smoke test.
**Depends-on:** S0.6

### SQ.6 Unit-test coverage for critical utils
1. Target ≥90% coverage on: `utils/timezone.ts` (S0.15), `utils/validation/appointmentOverlap.ts` (S1.3), `utils/fcAdapters.ts` (S0.14), `urlSync` (S0.11), `slotFinder.ts` (S9.2), `therapistSort.ts` (S9.2).

**Done-when:** Coverage report ≥90% on listed files in CI.
**Depends-on:** Each util ticket.

### SQ.7 ESLint guardrails for V2 hygiene
1. ESLint rule (or pre-commit grep) forbidding in `components/schedulingHub/**`: hex color literals (S1.6), `tss-react`/`makeStyles` imports, `useApi`/`useApiOld`/`useFetcher` imports, `@fullcalendar/rrule` import.

**Done-when:** Lint catches deliberate offending commit in smoke test.
**Depends-on:** S0.6

---

## Epic Close — Cutover

### SX.1 Pilot rollout
1. Flip `SCHEDULING_HUB.visible = true` on one pilot practice.
2. Monitor for 1 week.

**Done-when:** Pilot signs off; bug backlog < critical threshold.

### SX.2 General rollout
1. Flip remaining practices in batches.

**Done-when:** All practices on V2.

### SX.3 Delete V1
1. Remove `components/shared/Scheduler/`, `components/calendar/Calendar.tsx`, `pages/calendar/index.tsx` (replace with v2), `stores/calendar.tsx`, `hooks/calendar/`, V1-only legacy `useApi*` calendar code.
2. Remove DevExpress deps (`@devexpress/dx-react-scheduler*`) from `package.json`.
3. Remove `react-beautiful-dnd` if no other consumer.
4. Drop `useSchedulingHubEnabled` gate; v2 becomes the only path.
5. Drop the `{v: 2, ...}` saved-view discriminator (S0.11) and migrate stored blobs to the flat shape.

**Done-when:** `yarn build` size shrinks; no DevExpress in dependency graph; existing tests deleted/replaced; saved-view migration runs cleanly across practices.
**Depends-on:** SX.2

### SX.4 Internal QA runbook + Loom
1. Doc under `frontend/docs/calendar-v2-qa.md`: toggle path (AppSetting + ghost mode), per-category regression checklist, smart-scheduling validation steps, mobile fallback expectations, rollback procedure.
2. Loom recording of golden paths so QA/PM can self-verify without engineering shadowing.

**Done-when:** Doc merged; Loom linked from doc; PM signs off on coverage.
**Depends-on:** SX.1
