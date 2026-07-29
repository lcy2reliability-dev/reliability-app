# Reliability App — Changelog

**Repository:** GitHub → GitHub Pages (lcy2reliability-dev.github.io/reliability-app)
**Platform:** Single-page HTML app with Firebase Realtime Database
**Site:** LCY2, Tilbury
**Maintained by:** Tittus Streuti (strtt)
**Note:** From v1.04 onward, changes are made via Aki, working from a copy in `Reliability App - AKI/`. The Quick-built file is kept untouched as a fallback. Every app change is logged here in parallel — version bumps reflect feature changes; bug fixes are noted under the version they were fixed in without a version bump unless bundled with a feature change.


## v1.07 — 2026-07-16
**Status:** Deploying in OPEN mode (login / permissions stay off until the secure-mode rollout).

**Highlights:**
- **Parts Associated** — imported catalogue reference list on every position; link parts straight from it (see below).
- **Route data audit** — all 372 active routes reconciled to the master CBM sheet.
- **Accessibility** — modal focus management, keyboard/Escape handling, ARIA roles.
- **Login switch (dormant)** — `AUTH_ENABLED` flag; ships open, flip to secure later.
- **Technician permissions** + **security-rules compatibility** groundwork (active only in secure mode).
- **Fix: thermal readings failed to save on any dotted-name position** (SLAM, CB, AFE, ESTOP, etc.) — see below.
- **Fix: unlinked/linked parts lingered until manual refresh** — stale cache; the list now updates instantly, see below.
- **Add to Work ↔ Remove from Work toggle** — on a position/route detail page, once a job is added the green button becomes a red “Remove from Work” button (and flips back on removal). No more hunting through Work Assigned to undo a mis-tap.

Full detail of each item below.


## Feature — Add to Work / Remove from Work toggle (2026-07-28)
**Status:** In working file, deploys with v1.07

Previously, adding a position or route to Work Assigned from its detail page simply hid the “+ Add to Work” button; removing the job meant navigating to the Work Assigned tab. Now:

- Clicking **+ Add to Work** adds the job and the button immediately becomes red **− Remove from Work**.
- Clicking **− Remove from Work** deletes your matching job(s) and the button flips back — both with a confirmation toast.
- Only your own jobs are affected (matched on login + route/position number). Completed jobs are untouched — they already move to the work log.


## Fix — Recently Deleted now actually purges after 7 days (post-v1.07 scan)
**Date:** 2026-07-28
**Status:** Fixed in working file, deploys with v1.07

The Recently Deleted screen has always said "Items are permanently removed after 7 days", but no purge code existed — trashed items from early June were still in the database. Opening the screen (admin) now deletes anything trashed more than 7 days ago before the list renders. Items without a valid `deletedAt` are left alone.

## Database cleanup — full app + database scan (2026-07-28, no code change except the purge fix above)
A record-by-record integrity scan of all Firebase nodes. App code came back clean (no ID collisions, no stale-cache paths, syntax OK). Database fixes applied directly:

- **Orphaned thermal history recovered** — routes 506 and 888 had readings stored under legacy bare-number keys from before the APM rename; invisible to history/charts/anomaly baselines. Migrated to `CBM_THERMO_506` / `CBM_THERMO_888` (3 recording sessions) and the legacy keys removed.
- **Ghost record deleted** — `equipment/undefined` (an orphaned 65 KB screenshot with no position data).
- **Duplicate part removed** — APN 67846 existed twice (`parts/64846` was a key-typo duplicate); the correctly-keyed record kept.
- **Unreachable trash item rescued** — an old code version wrote one deleted route to `trash/route` (singular), which the Recently Deleted screen never lists. Moved to `trash/routes`.
- **Leftover TEST job deleted** from Work Assigned (created 03/07 during testing).
- **13 orphaned import stubs deleted** (`VFD01`–`VFD11`, `SLAM116`, `284504`) — empty position records auto-created by the 06/07 route import and no longer referenced by any route, part link, catalogue entry or reading. The other 477 description-less stubs (e.g. `01`–`25`, `ALIGN`) ARE referenced by route waypoints/motors and the parts catalogue, so they were kept.

Full pre-cleanup backup of every node: `Downloads/db_scan_backup_2026-07-28/`.


## Fix — thermal readings crash on dotted position names (post-v1.06)
**Date:** 2026-07-16
**Status:** Fixed in working file, not yet deployed
**Trigger:** Recording a reading on `SLAM.306.DVT1` (and every other position whose name contains a dot) failed with `push failed: value argument contains an invalid key ... Keys must be non-empty strings and can't contain ".", "#", "$", "/", "[", or "]"`.

### Cause
Each reading was stored under a key built from the position name plus " Motor" / " Gearbox" (e.g. `SLAM.306.DVT1 Motor`). Firebase forbids `.` `#` `$` `/` `[` `]` in keys, so the whole save was rejected. Only purely numeric positions (e.g. `140060 Motor`) saved — they have no illegal characters — which is why the bug looked intermittent.

### Fix
Reading keys are now run through the existing `sanitizeKey()` (dots → underscores) so they are always Firebase-legal, and a parallel `labels` map is saved alongside each reading batch mapping the sanitized key back to its original display name (`SLAM.306.DVT1 Motor`). History, charts, Excel export, the edit dialog and anomaly alerts all read the display name from `labels`, so the UI is unchanged — positions still show their real dotted names.

- **Backward compatible:** old readings (numeric positions) have no dots, so `sanitizeKey` is a no-op and the value lookups are identical; where a `labels` map is absent the code falls back to the key itself, which for old data equals the display name.
- Touched: `saveThermalReadings`, `saveRecordedReadings`, `renderThermalHistory`, `renderChart`, `exportThermalExcel`, `editThermalReading`, `checkThermalAnomaly` (16 edits total).


## Fix — linked/unlinked parts didn't update until manual refresh (post-v1.06)
**Date:** 2026-07-16
**Status:** Fixed in working file, not yet deployed
**Trigger:** After unlinking a part from a position, it stayed in the Linked Parts list until the page was manually refreshed.

### Cause
Parts are cached in memory for 1 minute (`CACHE_TTL`). `unlinkPart` and `confirmLinkPart` wrote the change to Firebase but re-rendered from the still-valid cache, so the screen showed the old list. Linking from the Parts Associated card already refreshed correctly because it invalidated the cache.

### Fix
- `unlinkPart` and `confirmLinkPart` now invalidate the parts cache before re-rendering, so the list reflects the change immediately.
- `unlinkPart` (only ever used on the Edit screen) now refreshes the Edit screen's linked list **in place** instead of jumping to the read-only Detail view, so you can unlink several parts in a row without losing your place.

### Also covered (2026-07-16 follow-up)
Audited every action that changes cached data and re-draws the screen. All now refresh instantly:
- **SCADA screenshot** add / replace / remove (`confirmQuickSS`, `replacePicture`, `removePicture`) now invalidate the equipment/route cache so the search list and edit screen never show a stale image.
- Verified already-correct: edit position/route/part, delete, decommission/reactivate, link from Parts Associated, remove from Parts Associated. No remaining gaps.


## Fix — "Route not found" when adding a CBM job (post-v1.06)
**Date:** 2026-07-20
**Status:** Fixed in working file, not yet deployed
**Trigger:** On Work Assigned, "+ Add New Job" > CBM would reject a valid route with "Route not found".

### Cause
The Route Number field was free text with a misleading placeholder ("e.g. 42"), but routes are stored under their full identifier (route `CBM.THERMO.575` is keyed `CBM_THERMO_575`, `routeNumber: "CBM.THERMO.575"`). The lookup only matched if you typed the exact full name, so a short number or different casing failed.

### Fix
- The Route Number field is now a **searchable picker** (datalist) listing the real routes, filtered by the selected CBM Type, so you choose an existing route instead of guessing its exact name.
- The lookup is now **forgiving**: if the exact key misses, it falls back to a case-insensitive / `routeNumber` match against the route catalogue and uses the canonical route before ever showing "Route not found".
- Typing **just the number** (e.g. `575`) now works: the lookup resolves it to the full route by matching the trailing number, disambiguated by the selected CBM Type when the same number exists under multiple types (verified: 575/897/850/875 unique; 42 disambiguated Thermal vs Ultrasound vs Ranger). Ambiguous cases fall back to the picker rather than guessing.


## Security hardening — red-team items 3–8 (2026-07-28)
**Status:** Applied in working file, ships with v1.07 deploy.
Pre-rollout hardening ahead of the 60–70 user expansion. No functional behaviour changes for normal use.

| # | Item | Change |
|---|---|---|
| 3 | **5 MB image cap** | `fileToBase64` now rejects files over 5 MB with a toast before reading — prevents quota exhaustion on Firebase free tier |
| 4 | **sanitizeKey strips `'` and `"`** | Single and double quotes in position/route names can no longer break `onclick` attribute strings |
| 5 | **CSP + X-Frame-Options** | Added `Content-Security-Policy` and `X-Frame-Options: SAMEORIGIN` meta tags — blocks clickjacking and limits script injection surface |
| 6 | **Save cooldown** | 5-second cooldown after each thermal reading save (`saveThermalReadings`, `saveRecordedReadings`) — prevents accidental or malicious write flooding |
| 7 | **Auto-clear label corrected** | Removed "All entries auto-clear at 04:00 daily" — the feature is not implemented; the misleading label is gone |
| 8 | **`maxlength` on key inputs** | Added limits to all main text inputs: identifiers 30–60 chars, descriptions/notes 300–500 chars, observations 300 chars |

**Remaining (require Firebase console / deploy decision):**
- C1/C2: Lock Firebase security rules + enable Auth (`AUTH_ENABLED=true`) — see `SECURITY_SETUP.md`
- C2: Restrict Firebase API key to your GitHub Pages origin in GCP Console
- H1: `currentUser` spoof protection resolved automatically by locking rules + Auth

---

## Parts Associated (imported catalogue reference) (post-v1.06)
**Date:** 2026-07-16  
**Status:** Built in working file, not yet deployed  
**Trigger:** Tittus wants each position to show the parts the maintenance catalogue says belong to it, as a reference the technician can turn into real linked parts.

### What it does
Each position detail now shows a second card, **Parts Associated**, below Linked Parts. It is an imported, read-only reference list (APN, class, quantity, description) pulled from the site parts catalogue. The catalogue is known to be noisy, so the list is treated as a suggestion, not truth.

- **Link** (technician+): opens the normal qty/observations popup, pre-filled with the imported quantity (which the tech can correct). On confirm it links the part to the position and the entry moves up into Linked Parts. If the APN is not yet a real part record, a minimal one is created from the reference first (promotion), then linked.
- **Unlink** (from the Edit screen, unchanged location): now also **restores** the part to Parts Associated, so the reference is never lost.
- **Remove ✕** (admin only): prunes a wrong entry from the associated list without touching any linked part.
- **One list per position:** an APN is either Linked or Associated for a position, never both. Linking (by any path) drops it from Associated; unlinking sends it back.

### Storage / performance
- Stored in a **separate top-level node** `associatedParts/{positionKey}`, **not** nested in `equipment`, so the app's startup load is unchanged.
- Loaded **lazily** — one small read only when a position detail is opened (a few KB), never the whole 10 MB.
- Import was filtered from a 94 MB / 661k-row region BOM down to **10.2 MB / 107,633 entries across 3,371 positions** by keeping only positions that exist in the app (matched by the last dotted segment / trailing digits of the catalogue equipment tag) and de-duping by APN.

### Data model
`associatedParts/{sanitizeKey(conveyorNumber)}` = array of `{apn, description, partType, qty}`.

### Re-import note
Re-import is an Aki-run REST operation (admin file → filtered → written). A future re-import must **merge**, not overwrite, or it would clobber admin removals and the Linked/Associated state. Not scheduled yet.

## Login Switch — Open vs Secure Mode (post-v1.06)
**Date:** 2026-07-16  
**Status:** Ready to deploy  
**Trigger:** Tittus wants to ship the v1.07 improvements now without turning on the login gate — the app is contained (3 users) and the login/PIN/permissions rollout can wait.

### What changed
A single flag, **`AUTH_ENABLED`**, near the top of the login code decides how the app behaves:
- **`false` (the shipped default) — OPEN mode.** No Firebase login. On first load the app asks once for a name (remembered in the browser, used to attribute readings/notes), then gives full access — exactly like the original pre-auth build. Deployable immediately with **zero** Firebase console work.
- **`true` — SECURE mode.** The full Firebase login (Login ID + PIN) with viewer/technician/admin roles. Flip this only when doing the login rollout, and only after enabling Email/Password in the console and publishing the locked rules (see `SECURITY_SETUP.md`).

### How it works
- Startup dispatches to `initNoAuth()` in open mode instead of the Firebase auth listener.
- The existing login overlay is reused as a name-only prompt (the PIN field, its label, and the account hint are hidden); the prompt reads "Enter your name to continue".
- "Logout" in open mode simply switches the remembered name (no Firebase sign-out).
- Nothing else changes: same screens, same data, same features. Flipping the flag is the only action needed to move between modes.

## Technician Permissions Expansion (for v1.07 deploy)
**Date:** 2026-07-14  
**Status:** Ready to deploy  
**Trigger:** Requested by Tittus — technicians should be able to do day-to-day data upkeep, not just record readings. Deletion, decommissioning, bulk imports, and user management stay admin-only.

### What changed
Technicians (not just admins) can now:
- **Add and edit positions, parts, and routes** — the create forms and the edit forms both save for technicians now (previously the edit form opened but the Save button was blocked).
- **Add, replace, and remove SCADA location screenshots.**
- **Link and unlink parts to/from positions.**
- **Edit route waypoints** (start point, waypoints, end point).

Still **admin-only**: deleting positions/parts/routes, decommissioning (the active/inactive toggle), restoring or permanently purging from Recently Deleted, bulk position imports, full-database backup, and managing users/roles.

### How it's enforced (both layers)
- **In the app:** 16 functions were moved from an admin-only guard (`requireAdmin`) to a write-access guard (`requireWrite`, i.e. technician or admin): the position save/edit paths, the part save/edit paths, the route save/edit paths (`smartSaveRoute` and the manual/APM/CSV single-route savers it dispatches to), the SCADA picture add/replace/remove functions, and the link/unlink-part functions. The delete, trash, decommission, bulk-import, backup, and user-management functions keep their admin-only guard.
- **In the Firebase rules** (`SECURITY_SETUP.md`): for `equipment`, `parts`, and `routes`, the collection-level write stays admin-only, and a new per-item (`$key`) rule additionally allows a technician to write an item **only while the result still exists** (`newData.exists()`). Creating and editing satisfy that; deleting (nulling a node) does not — so technicians can create/edit but the database itself still refuses to let them delete. Single-item paths under those nodes (a route's waypoints, a part's `conveyorNotes` link) are covered because rules cascade down to child paths.

## Accessibility (Modal/Keyboard) & Security-Rules Compatibility (post-v1.06)
**Date:** 2026-07-14  
**Status:** Ready to deploy  
**Trigger:** Completing the 3 Critical + 4 High accessibility findings that Round 6 explicitly **deferred** ("tackle as its own focused pass"), plus preparing the app so it stays fully functional once the locked server-side Firebase rules in `SECURITY_SETUP.md` are published.

### Accessibility — dialog & tab semantics (was deferred in Round 6)
- **The confirmation modal and the three full-screen overlays (detail page, add-item form, login) now announce themselves as dialogs.** Added `role="dialog"` + `aria-modal="true"` to each, gave the confirm modal `aria-labelledby` pointing at its title, and gave the overlays descriptive `aria-label`s. Screen readers now correctly announce "dialog" and its purpose instead of reading it as an anonymous block of page content.
- **The Search / Work Assigned tabs are now proper tabs.** The tab strip is marked `role="tablist"`, each tab is `role="tab"` with `aria-selected` reflecting the active state (kept in sync by `switchTab`/`switchTabDirect`), and the global keyboard handler now activates a focused tab on Enter/Space the same way it does buttons. Previously they were generic `role="button"` elements with no selected-state announced.

### Accessibility — modal focus management (was deferred in Round 6)
- **The confirmation modal now traps and restores keyboard focus.** When it opens, focus moves into the dialog; Tab / Shift+Tab cycle only within the dialog's own controls instead of leaking to the page behind it; and when it closes, focus returns to whatever element the user was on before it opened. Keyboard and screen-reader users can no longer get "lost" behind an open modal.
- **Escape now closes the modal.** Pressing Escape triggers the modal's Cancel action (its secondary button) when one is present, matching standard dialog behaviour. Previously the only way out was a mouse click.

### Security-rules compatibility (so nothing breaks when the locked rules go live)
The locked rules in `SECURITY_SETUP.md` set the database root to no-read / no-write and only open specific top-level nodes. Four code paths assumed root-level or unconditional access and were adjusted so they keep working (and stop generating permission-denied noise) once the rules are published:
- **"Download Backup" no longer reads the database root.** It read `db.ref('/')` for a whole-tree dump; under the locked rules that single call is denied even for an admin (read permission does not cascade up from child nodes to the root). It now reads each of the nine covered nodes individually and assembles them into the same backup file — same output, but permitted.
- **The three login-time maintenance routines now check role before writing.** `oneTimeCleanup` (writes equipment) and `cleanupDuplicateEndpoints` (writes routes) are now admin-only; `cleanupOldWork` (removes stale Work Assigned entries) now requires write access. They run for every user on login, so without these guards a viewer's login would have fired writes that the locked rules deny. (`migratePartsData` was already guarded.)

### Verified
- Cross-checked every Firebase path the app reads/writes against the rules in `SECURITY_SETUP.md`: all nine top-level nodes the app touches (`equipment`, `parts`, `routes`, `readings`, `trash`, `users`, `workAssigned`, `workLog`, `notes`) are covered, and all dynamic `db.ref(collection + '/' + key)` calls resolve to a covered node. The only uncovered access was the root-level backup read, now fixed above.
- File passes a JavaScript syntax check with balanced braces (1422/1422) and parentheses (4819/4819).

### Still outstanding (not in this batch)
- **Predictive search dropdown keyboard navigation** (arrow-key / Enter selection) — one of the Round 6 High findings, still deferred. Genuinely separate work from the modal/tab pass and lower risk to users than the modal trap, which is why it wasn't bundled here.


## Performance, Quality & Contrast Fixes (post-v1.06) — Council Audit Round 6
**Date:** 2026-07-14  
**Status:** Ready to deploy  
**Trigger:** Second half of the follow-up verification audit — Performance, Code Quality, and UX/Accessibility auditors, run alongside the Security/Correctness/Domain pass above. Zero Critical findings from Performance and Quality. UX/Accessibility found 3 Critical (modal focus/keyboard-trap gaps) which are **not** included in this batch — see "Deferred" below.

### Performance
- **Work Assigned no longer re-downloads the whole job list on every check.** Eight different actions — saving a thermal reading, the daily cleanup, opening a detail page, clearing all jobs, adding an item to Work Assigned, adding a thermal note, adding a route, and editing a thermal reading — each independently pulled the *entire* `workAssigned` collection from Firebase just to check "is this already in my queue?". Since the Work Assigned tab already keeps a live, continuously-updated copy of that same data (via its real-time listener), all eight now reuse that in-memory copy instead of making their own network round trip. On a busy day this cuts roughly 8 redundant full-collection downloads down to 0–1.
- **Recording thermal readings no longer re-reads the same history multiple times.** Each temperature measurement was checked against its own 3-reads-of-history lookup, so a route with 6 motors/gearboxes fired 6 near-identical Firebase reads for one save. History is now fetched once per save and reused for every component's anomaly check.

### Code quality — cache freshness
- **New positions, routes, deletions, and reactivations now show up immediately instead of after up to 60 seconds.** Four write paths (`saveNewPosition`, `saveNewRoute`, `deleteEntry`, `reactivateItem`) wrote to Firebase but never told the app's 60-second in-memory cache that its copy was stale — so a newly added position, a freshly created route (and the positions it auto-creates), a deleted item, or a reactivated item could still show as absent/present/inactive in search and duplicate-checks for up to a minute. All four now invalidate the relevant cache right after saving.

### Accessibility
- **Low-contrast secondary text now meets readability guidelines.** A light grey (`#9ca3af`) used throughout the app for hints, timestamps, and empty-state text failed the minimum WCAG contrast ratio against a white background (roughly 2:1 vs. the 4.5:1 required). Swapped app-wide for a darker grey (`#6b7280`, already used elsewhere in the app for the same purpose) — same visual hierarchy, actually readable.
- **Burger menu button no longer silently drops its positioning style.** The hamburger button had two `style=` attributes on one element; browsers only honour the first, so the `position:relative` needed by the "what's new" notification dot was being discarded. Merged into one attribute.
- **A dead real-time listener and a page-init failure now tell the user something's wrong** instead of only logging to the browser console. If the Work Assigned list's live connection drops, or if login setup fails partway through, the user now sees a toast telling them to refresh, instead of a silently frozen or empty screen.

### Deferred (flagged, not fixed this round — bigger scope, higher regression risk if rushed)
The UX/Accessibility auditor found 3 Critical and 4 High findings that are all real but represent a bigger, riskier unit of work: the confirmation modal has no `role="dialog"`, no keyboard focus-trap, and no Escape-to-close; the predictive search dropdown has no keyboard navigation; the Search/Work Assigned tabs and three full-screen overlays (burger menu, detail page, add-item form) are missing dialog semantics and don't manage keyboard focus. Recommending this be tackled as its own focused pass — touching focus/keyboard behaviour across every overlay in one hurried batch is exactly the kind of change that should get its own attention and testing rather than being bundled in.

---

## XSS Hardening & Follow-up Fixes (post-v1.06) — Council Audit Round 5
**Date:** 2026-07-14  
**Status:** Ready to deploy  
**Trigger:** Follow-up verification audit (3 focused auditors — security, correctness, domain) run against the Round 2 fixes above, to confirm no Critical/High issues remained before deploy. Zero Critical findings. One live HIGH (DOM-XSS) and 8 live Medium/Low escaping gaps fixed below; several other reported findings turned out to be dead code from the pre-v1.03 tabbed UI (see "Investigated, no action needed" below) and were left alone.

### Security — remaining XSS gaps closed
- **Search: typing HTML into the search box no longer risks injecting markup.** The "No results found for…" message echoed the raw search text into the page; a search for something like `<img onerror=...>` would have executed. The searched text, and an inactive-item name shown in the "Found in inactive items" panel, are now escaped.
- **Thermal recording form now escapes everything sourced from Firebase.** The route number and notes in the recording form's heading, each measurement point's label, and the hidden component identifiers on the Motor/Gearbox inputs were all inserted unescaped — now all escaped.
- **Work Assigned list now escapes job type, equipment name, and linked part fields** (APN and part type) shown on each job card.
- **Position detail's Linked Parts table** now escapes the Bin Location/Location column (every other column in that row was already escaped — this one was missed).
- **Route detail** now escapes each route waypoint's display name and the component names listed in the "View trend for" dropdown.
- **`editNote` now checks write permission directly** (it was only ever called from an already-guarded caller, but the guard belongs on the function itself, matching the pattern used by `deleteNote` right next to it).
- **Burger menu now escapes the displayed username** when the menu is reopened (the very first render already escaped it; this only affected the secondary update).

### Domain — thermal anomaly detection
- **A reading of exactly 0°C no longer silently skips anomaly checking.** `checkThermalAnomaly`'s guard used `!currentValue`, and in JavaScript `!0` is `true` — so a genuine 0°C reading (which, per the bilateral fix above, is exactly the kind of under-temperature/sensor-detachment signal this check exists to catch) was treated as "no value" and ignored. The guard now checks for null/undefined/empty-string/NaN explicitly instead of falsiness.

### Investigated, no action needed
- The verification audit flagged several functions (`loadEquipmentList`, `loadPartsList`, `loadRoutesList`, the bulk equipment CSV/XLSX import, the QR scanner, and the old standalone Thermal Readings tab) as referencing HTML elements that don't exist. Traced each one: they all belong to the "Manage Equipment/Parts/Routes" and "Thermal Readings" tabs that were removed in v1.03 — the markup was left in the file but wrapped in `display:none!important` with no code path that ever un-hides it. Confirmed dead and unreachable by any current user action; no fix applied. Candidate for a future cleanup pass (delete the orphaned HTML/JS) but zero runtime risk today.
- `showPartsPopup`'s missing permission guard was also traced to the same dead "Add Part Used" button inside the removed Manage Equipment tab — unreachable, no fix needed.

---

## Security Hardening & Consolidated Fixes (post-v1.06) — Council Audit Round 2
**Date:** 2026-07-14  
**Status:** Ready to deploy  
**Trigger:** Consolidated Critical/High/Medium findings from a two-round, six-angle audit (security, correctness, UX, performance, code quality, reliability-engineering domain). Applied as 12 guarded patch batches, each verifying exact match counts before writing; final file passes a JS syntax check with balanced braces/parens.

### Security — permission guards (client-side)
- **12 state-changing actions now check the user's role before running.** Functions that edit or delete data — editing a thermal reading, marking a job done/undone, editing a note, reactivating a deleted item, removing/replacing a picture, toggling a route active, clearing all jobs, opening the edit form, running the parts-data migration, downloading a full backup, and opening the Recently Deleted screen — previously ran regardless of whether the user was a viewer, editor, or admin. Each now enforces the appropriate write- or admin-level guard. (These are client-side guards; server-side Firebase Security Rules are the real enforcement layer and are tracked separately in SECURITY_SETUP.md.)
- **PIN validation tightened.** Login now requires a PIN of 6 or more digits (digits only), rejecting short or non-numeric input that the previous length-only check would have let through.

### Security — HTML escaping (XSS hardening)
- **26 places that displayed user-entered text now escape it first.** Anywhere a record's free-text fields (descriptions, notes, areas, serial numbers, criticality, store/bin locations, manufacturer/supplier, conveyor numbers, route waypoints, search queries reflected back on screen) were dropped straight into the page, a crafted value could have injected markup or broken the layout. All of these display sites now run through `escapeHtml()`. Covers position detail, part detail, route detail, the link-part popup, inactive-item search, autocomplete, and the Recently Deleted list.

### Correctness — data safety
- **CRITICAL: deleting from the detail page no longer skips the trash.** `confirmDetailDelete` was hard-deleting the record outright, bypassing the soft-delete `trash/` collection that every other delete path uses — so those items were unrecoverable and never appeared in Recently Deleted. It now moves the record to `trash/{collection}/{key}` with a `deletedAt` timestamp first, matching the rest of the app.

### Correctness — dead/mismatched code
- **Work Assigned drag-to-reorder, overdue detection, and the tab count now actually work.** A cluster of functions (`updateWorkCount`, `renumberJobs`, `saveJobOrder`, `checkOverdueJobs`, and all the drag handlers in `initJobDragDrop`) targeted DOM that doesn't exist — the class `.work-job-card`, the id `#work-jobs-list`, and `dataset.jobId` — while the actual render uses `.work-card`, `#work-list`, and `data-key`. These had silently never worked; the selectors are now corrected to match what's rendered.
- **Programmatic tab switches to Routes no longer depend on a stray click event.** Three call sites (`editRoute`, `relinkRoute`, and one more) called `switchTab('routes')`, which relies on the implicit `window.event` — undefined when the call isn't triggered by an inline click, so the switch could misbehave. They now call `switchTabDirect('routes')`.
- **Route detail loads thermal history consistently.** The route branch of `openDetail` used one condition to decide whether to *show* the thermal-history card and a different one to decide whether to *load* it. Both now use the shared `routeReadingRequired(route)` helper.
- **Editing a note with an apostrophe no longer breaks the note list.** `refreshNotesList` was substituting `&apos;` into an inline onclick attribute, which broke the handler's parsing. It now escapes correctly.
- **Role changes refresh the user list.** `changeUserRole` now awaits `openUsersScreen()` so the list reflects the change.
- **"Done by" is now recorded.** Marking a job done now saves `completedBy` so it's clear who completed it.

### Domain — thermal anomaly detection (CBM correctness)
- **Cold faults are now detected, not just hot ones.** The anomaly check only ever alerted on over-temperature. Per ISO 13373, a sudden drop (sensor detachment, a cooling fault reading unrealistically low) is just as diagnostic. The check is now bilateral — it alerts on a 10 degC deviation from a component's own historical average in *either* direction, with the alert wording and colour reflecting hot vs cold.
- **Valid cold readings are no longer discarded from the baseline.** The historical-average calculation filtered readings with `value > 0`, silently dropping legitimate near-zero/cold readings. It now filters on `value >= TEMP_MIN`, so the baseline reflects real history.

### Performance
- **Fewer full-database reads.** Several paths that re-pulled entire collections from Firebase now reuse the in-memory caches: route/part lookups when opening a detail page, duplicate-APN checks in `savePart`/`saveNewPart`, the edit-linked-parts list, part-to-link search, and the three-collection inactive-item search. `saveNewPart` also now invalidates the parts cache after saving, so new parts appear immediately.

### UX & accessibility
- **Login fields are now labelled** (`<label>` elements for the login ID and PIN inputs) for screen-reader and autofill support.
- **Toast notifications announce themselves** to screen readers (`role="alert"`, `aria-live="assertive"`).
- **Viewers no longer see controls they can't use** — the note-input box and Add-note button are hidden from view-only users.
- **Cleaner logout.** `burgerLogout` guards its listener-detach calls (`.off()`) against a missing database handle and re-applies role-based visibility after logging out.

---

## UX & Accessibility (post-v1.06) — Council Audit Round 4
**Date:** 2026-07-14  
**Status:** Ready to deploy  
**Trigger:** UX/accessibility findings from the multi-angle audit. Aimed at glove/thumb use on the shop floor and basic keyboard/screen-reader support.

- **Bigger touch targets.** Tabs and search autocomplete rows are now at least 44px tall (Apple/WCAG minimum), and the small remove/add buttons get a 44px minimum height on mobile, so they're easier to hit with a thumb or glove.
- **No more iOS zoom-jump on inputs.** iOS auto-zooms when you focus an input smaller than 16px. All inputs are now forced to 16px on mobile, so focusing a field no longer yanks the layout around.
- **Numeric keypad for temperature entry.** The four thermal reading fields now request a decimal numeric keypad (`inputmode="decimal"`), so technicians get digits instead of the full keyboard.
- **Spinner can't get stuck.** The loading overlay now has a 15-second safety timeout — if a Firebase call hangs (bad Wi-Fi), the spinner clears itself instead of trapping the user on a frozen screen.
- **Offline indicator.** A red "You are offline" banner now appears automatically when the device loses connectivity and disappears when it returns, so technicians know their changes may not be saving.
- **Keyboard access.** The tabs and burger-menu items are now reachable by Tab and can be activated with Enter or Space (they carry `role="button"` and `tabindex`), with a single shared key handler.

---

## Performance (post-v1.06) — Council Audit Round 3
**Date:** 2026-07-14  
**Status:** Ready to deploy  
**Trigger:** Performance findings from the multi-angle audit. No user-facing behaviour change; the app just loads and runs lighter, especially on poor warehouse Wi-Fi.

- **Faster first load — heavy libraries no longer block the page.** Chart.js (~200KB) and the XLSX/Excel library (~860KB) were loaded up front via a render-blocking `document.write`, stalling the whole app behind ~1.3MB of downloads before anything appeared. Only Firebase now loads up front; Chart.js and XLSX load lazily on demand (and quietly preload in the background a second after login), via a small `loadScriptOnce` helper. All chart and Excel import/export code now `await`s the relevant library, so nothing breaks if you use a feature before its library finishes loading.
- **Search is dramatically cheaper.** `searchAll` re-downloaded the entire equipment, parts, and routes database (including any base64 images stored on records) from Firebase on every single search. It now reuses the same 60-second in-memory cache the autocomplete uses, so repeated searches are instant and don't re-pull the whole DB each tap.
- **Route detail opens faster.** The route detail page validated each route point with its own sequential Firebase round-trip (N points = N blocking reads). It now reads the equipment cache once and checks all points in memory — one fetch instead of N.
- **Work Assigned list no longer stacks listeners / flickers.** `loadWorkAssigned` is called after every work add/edit/delete, and each call attached a *new* realtime listener without removing the old one, so the list re-rendered multiple times (visible flicker) and leaked memory. It now detaches the previous listener before attaching a fresh one.
- **Thermal chart is cleaned up on close.** Leaving a route detail page now destroys the Chart.js instance, freeing the canvas and avoiding a slow memory creep across many route views.

---

## Bug Fixes (post-v1.06) — Council Audit Round 1
**Date:** 2026-07-14  
**Status:** Ready to deploy  
**Trigger:** Full multi-angle audit (security, correctness, UX, performance, code quality, reliability-engineering domain).

### Thermal / Anomaly Detection (core safety feature — was silently broken)
- **Anomaly detection now fires on the main Thermal tab.** Readings recorded from the Thermal tab were checked against a component key of `reading-0`, `reading-1`... which never matched the stored keys (e.g. `"<point> Motor"`), so no alert could ever trigger from that path. The check now maps each input to its real component via `currentThermalItems`, so it matches the stored reading key exactly.
- **Anomaly alert notes now actually save.** The anomaly modal's buttons used `{className, action}`, but `showCustomModal` reads `{class, value}` and resolves with `value` (it never calls `action`). The "Save Note" callback therefore never ran. Rewritten to check the resolved value and save the note.
- **Anomaly baseline no longer self-contaminates.** The new reading was written to Firebase *before* the anomaly check read the last-10 history, so the reading polluted its own baseline average. The check now runs before the save.

### Work Assigned
- **Job notes no longer corrupt after the first save.** Notes were stored as a JS array via `.set([...])`; Firebase returns that as an object on re-read, so the second `.push()` threw and the list stopped rendering. Now appended with `.push()` and rendered via `Object.values()`.

### Recently Deleted
- **Deleted Positions/Parts/Routes are now restorable again.** `detailDelete` wrote soft-deletes to `trash/part/` and `trash/route/` (singular), but the Recently Deleted screen reads `trash/parts/` and `trash/routes/` (plural), so those items were invisible and unrestorable. The write path now uses the plural collection names to match.

### Search freshness
- **Edits and deletes now appear in search immediately.** `detailDelete`, `confirmDetailDelete`, `saveEditPosition`, `saveEditRoute`, and `saveEditPart` now invalidate the relevant 60-second search cache, so changes show up in the predictive dropdown and full search right away instead of after up to a minute.

### Reliability / memory
- **Firebase listeners no longer leak on logout.** `equipment`, `parts`, `routes`, and `workAssigned` real-time listeners are now detached (`.off()`) in `burgerLogout`, preventing them stacking up (and the associated memory growth / post-logout `currentUser` crash) across repeated logins on a shared device.

### Consistency (native dialogs removed)
- **Replaced all 6 remaining native `confirm()`/`prompt()` calls with the app's custom modal** (`promptEditNote`, `addWorkNote` note + save-to-entity, `addThermalNoteToWork`, `saveRecordedReadings`, `saveWorkJobNote`). Native dialogs block the JS thread, can't be styled, and misbehave on mobile/PWA. All now use styled `showCustomModal` prompts and Yes/No confirmations.

### Also fixed
- **Help & Reference page can now be closed.** `openHelpPage` showed the overlay with an inline `style.display='block'`, but every close path only removes the `active` class — so the Back button did nothing. It now opens via `classList.add('active')` to match.

### Round 1 (continued) — deferred structural fixes
- **Thermal recording UI now appears on new/imported routes immediately.** Routes were created with the field `needsTemp`, but the Record button, thermal-history section, and Work Assigned Record button all gated on a *different* field, `readingRequired` — which only the Edit form ever wrote. A freshly created or imported route showed no recording UI until someone opened Edit and re-saved. Added a single `routeReadingRequired(route)` helper (prefers `readingRequired`, then `needsTemp`, then falls back to `type === 'Thermal'`) used at all 6 read sites, and made all 5 write sites set **both** fields to the same value. `saveEditRoute` now derives both from the checkbox instead of hardcoding `needsTemp` to `type === 'Thermal'`.
- **Editing a Work Assigned job no longer risks permanent data loss.** `editWork` previously deleted the record and opened a blank Add form — if you closed without re-saving, the job was gone. It now fetches the job, opens the modal pre-filled in a true "Edit Work Assigned" mode, and saves via an `update()` (preserving `createdAt`, adding `updatedAt`) instead of delete-then-recreate.
- **`saveRecordedReadings` anomaly baseline no longer self-contaminates.** Like the main Thermal tab fix, the route-detail recording path now runs all anomaly checks *before* writing the new readings to Firebase, so a reading can't pollute its own historical average. The old post-save anomaly loop was removed.

---

## v1.06 — Edit-Only Delete/Unlink + Design Polish
**Date:** 2026-07-09  
**Status:** Ready to deploy  
**File:** `index.html` (334,131 bytes)

### Security / UX
- **Delete and Unlink restricted to Edit screen only:** removed all Del buttons from the Positions, Parts, and Routes list tables (search/browse views). Removed the ✕ Unlink button from the Linked Parts table in position detail, and from the linked-position chips in part detail. Removed the Delete button from thermal reading history. All destructive actions are now only accessible through the Edit overlay (accessed via the ✎ button or Edit button from the detail page).
- **Unlink sections added to Edit overlay:** when editing a Position, the Edit screen now shows all linked parts with an Unlink button next to each. When editing a Part, it shows all linked positions with Unlink buttons. Unlinking from here triggers the full confirmation modal (below).
- **Unlink confirmation modal:** all Unlink actions now open a warning modal (`⚠️ Unlink Part`) showing the position name and any saved Qty/Observations for that link before confirming. Cancel/Unlink buttons. The part itself is never deleted by an unlink.

### Design Polish
- **Active tab highlight:** selected tab now shows a blue bottom border (`#00A8E1`) in addition to the text colour change, making the active tab immediately obvious.
- **Touch targets enlarged:** `.btn-sm` buttons now have `min-height: 36px` (desktop) and `44px` (mobile), meeting minimum touch-target guidelines.
- **Loading overlay:** the spinner is now presented inside a white card with a "Loading..." label and a subtle `backdrop-filter: blur(2px)` behind it — easier to read against any background.
- **Header title centred:** the "RME App" title in the header is now absolutely centred regardless of the logo or burger-menu button width.
- **Position Number in detail view:** the Position ID field (conveyor number) is now displayed at `font-size: 20px; font-weight: 700` — matching the APN styling on part detail pages.
- **Predictive search dropdown CSS:** `.autocomplete-item` class added; dropdown items have hover state and bottom border separators.

### Internal
- **liveSearchSuggest now uses Firebase cache:** predictive search reads from `getEquipmentCache()`, `getPartsCache()`, and `getRoutesCache()` instead of making fresh Firebase calls every 300ms. Reduces read overhead during live typing.

---

## v1.05 — RME App Rebrand + UX Fixes
**Date:** 2026-07-09  
**Status:** Ready to deploy  
**File:** `index.html` (324,005 bytes)

### Features
- **Predictive (typeahead) search restored** on the main Search bar: results appear in a dropdown as you type (2+ characters), showing matching Positions, Parts, and Routes with type badges. Clicking a result opens the detail page directly. Pressing Enter still runs the full search as before. Dropdown dismisses on outside click.
- **Logo is now interactive:** clicking the Amazon RME logo in the header returns to the Search tab, closes any open overlays, and clears the search state — equivalent to a home button.
- **Link Part to Conveyor — Quantity + Observations fields added:** when linking a part to a conveyor (from position detail page), two new fields now appear after confirming the part: *Qty on this conveyor* (how many of that part are used) and *Observations* (optional free text, e.g. “described as end pulley, but actually tension roller”). Both values are saved to Firebase under `parts/{key}/conveyorNotes/{conveyorKey}`.

### Bug Fixes
- **Pinch-to-zoom on mobile now works:** removed `maximum-scale=1.0, user-scalable=no` from the viewport meta tag, which was blocking native zoom on touchscreens.
- **Deleted item now disappears from screen immediately:** after deleting any item (from detail page or search results table), the search results auto-refresh so the deleted record is no longer visible. Previously the stale row / card remained until the next manual search.
- **Burger menu dividers corrected:** removed the duplicate divider between Tools and Help; added the missing divider between Help and Account. Menu sections are now consistently separated.

### Rebrand
- **App renamed from “Reliability App” to “RME App”** across all surfaces: browser tab title, header, login screen. More inclusive naming for the site.

### Notes
- Menu section header font sizes confirmed consistent: all section labels (Add New, Tools, Help, Account) are `12px` uppercase; all menu items are `14px`. No changes needed.

---

## v1.04 — Thermal History Toggle + Carrier Position Import
**Date:** 2026-07-08  
**Status:** Deployed  
**File:** `index.html` (317,935 bytes)

### Bug Fix
- **Thermal History section on route detail page** now correctly respects the `readingRequired` toggle: hidden when off, shown when on, and re-hides if toggled off again. Previously it could remain visible regardless of the toggle state.

### Data
- Imported 17 **CARRIER**-class positions from the RME building position spreadsheet (`CB.SRT.01.CARR.01`–`CARR.16` + `CB.SRT.01.LN.FLAT`), each enriched with manufacturer (DEMATIC), model (SC3CAR.GRP), and criticality data. Import skips any position already in the database.
- Remaining RME position classes from the same spreadsheet (CONV, GUARD, CNTRL, STN, BLDG, SUB, GATE, CHUTE, SRT, STCKR — roughly 3,900 positions) were reviewed but **not imported** — decision deferred, on hold until requested.

---

## v1.03 — UI Restructure: Simplified Navigation
**Date:** 2026-07-03  
**Status:** Deployed  
**File:** `index.html` (235 KB)

### Overview
Major UI simplification. Removed all data-entry tabs. Everything is now accessed through Search + burger menu. Two-tab interface: Search + Work Assigned.

### Navigation Restructure
- **6 tabs → 2 tabs:** Search + Work Assigned only
- **Burger menu (hamburger icon, top-right)** replaces all other navigation:
  - Account: logged-in user, Switch User, Logout
  - Add New: Position, Part, Route (opens overlay form)
  - Tools: Bulk Import (APM), Download Backup
- **"Reliability App" title** centered in header
- **Clear/Backup buttons removed from header** (Clear is automatic on tab switch; Backup moved to burger menu)
- **Login/username removed from header** (now in burger menu Account section)

### Detail Page Redesign (View Mode)
- **Screenshot displayed** if one exists (full image visible)
- **"Screenshot missing — click to add"** warning link when no image. Clicking opens inline upload (drag & drop, paste, or browse) with preview + Save confirmation. No need to go to Edit page for this.
- **Edit button** opens the Edit form overlay (see below)
- **Add to Work button** adds to Work Assigned queue
- **Record Readings button** (thermal routes) opens inline recording form
- **Notes section** at bottom (chronological, author-only edit/delete)
- **No Delete button** on the view page (moved to Edit page Danger Zone only)

### Edit Page (from Edit button)
- Opens as an overlay (same detail panel, different content)
- **All fields pre-populated** with existing data
- **Screenshot:** if exists → shown with "Replace" button. If none → upload box shown.
- **Route waypoints:** editable, add/remove
- **Active/Inactive toggle** (routes only)
- **Danger Zone at bottom:** requires typing "DELETE" + confirmation modal
- **Save + Cancel buttons**

### Add New Forms (from burger menu)
- **Add New Position:** ID, Type, Description, Notes, Screenshot upload
- **Add New Part:** Position, APN, Part Type, Store Location, Notes
- **Add New Route:** Route Number, Type, Description, Start/Waypoints/End
- **Bulk Import:** APM Checklist drag & drop with preview
- All open as overlay forms, close back to Search

### Active/Inactive System (All Entity Types)
- **Positions, Parts, and Routes** all now have an `active` toggle in their Edit page
- **Search hides inactive items** by default (equipment, parts, routes all filtered)
- **"Found in inactive" prompt:** when a SPECIFIC search (contains numbers/dots) returns no active results, the app checks inactive items and shows: "⚠ Found in inactive items" with Reactivate/View buttons
- **Generic searches** (like "Thermal", "CBM", "Manual Induct") do NOT trigger the inactive prompt — only specific IDs/numbers do
- **"Manage Inactive Items"** in burger menu: dedicated page with search box + filters (Positions/Parts/CBM Routes) to find and reactivate archived items
- **Reactivate button:** instantly sets `active: true` and opens the detail page

### Removed
- Manage Equipment tab (Edit from detail page)
- Add Parts tab (Add from burger menu)
- Manage CBM Routes tab (Edit from detail page)
- Thermal Readings tab (Record from route detail)
- Clear button (tab switch auto-clears)
- Header username display (now in burger menu)
- Remove/Replace Picture buttons from detail view top

---

## v1.02 — Notes System, Work Assigned Overhaul, Manual Induct Mapping
**Date:** 2026-07-03  
**Status:** Deployed  
**File:** `index.html` (187 KB)

### Database Cleanup (one-time, automatic)
- All equipment **Type** fields wiped (were incorrectly set during bulk imports)
- All equipment **Notes** fields wiped (will be re-added by users with correct data)
- Runs automatically on first load after update - no user action needed

### UI Changes
- Tab renamed: "Add Equipment" -> **"Manage Equipment"**
- Label renamed: "Conveyor Type" -> **"Type"**
- Removed "Re-Import Route Points" button from route detail page (all edits via Edit menu only)
- Removed "Saved Positions" table from Manage Equipment tab (use Search to find equipment instead)
- Removed "Saved Routes" table from Add CBM Routes tab
- Removed "Saved Parts" table from Add Parts tab
- **Tab switch clears everything:** switching tabs now clears all inputs, search results, detail overlays, and form previews across ALL tabs — fresh start every time



### Tab & Navigation Changes
- Tab renamed: "Add CBM Routes" → **"Manage CBM Routes"**
- **Thermal Readings tab REMOVED** — recording is now accessed from:
  - Route detail page ("Record Readings" button)
  - Work Assigned tab ("Record" button on thermal CBM jobs)
  - Search results → route detail → record
- Route type dropdown now includes: **Oil, Ranger, Strobe, Vibration** (in addition to Ultrasound, Thermal, Other)

### Active/Inactive Routes
- All routes now have an `active` field (true/false)
- **Toggle switch** on each route's detail page (above Danger Zone)
- Inactive routes are **hidden from search** and cannot be added to Work Assigned
- All 950 routes imported as `active: true` by default
- After uploading PM-assigned routes file, non-PM routes can be toggled inactive

### Thermal Recording Redesign
- Recording form now appears **inside the route detail overlay** (no separate tab)
- Shows grouped by point: Motor above Gearbox
- Notes field with Enter-to-submit
- After saving: auto-adds to Work Assigned, runs anomaly check, returns to route detail
- "Record" button available on thermal CBM jobs in Work Assigned tab

### Manual Induct Search Mapping
- Search "Manual Induct 6" or "MI 6" or "ID 6" -> returns all CB.SRT.01.ID21.XX equipment
- Search "Manual Induct" or "MI" -> returns ALL induct stations
- Full mapping: MI 1-5 = North (ID11-15), MI 6-10 = South (ID21-25)
- Searching by internal ID also works (e.g. "ID21" returns the same results)

### Notes/Comments System (NEW)
- **Notes on every entity** - Routes, Equipment, Parts all have a Notes section at the bottom of their detail page
- **Add notes from:** Detail page, Thermal Readings, Work Assigned
- **Ownership:** Only the author can Edit/Delete their own notes
- **Timestamps:** Today/Yesterday/Day name/Full date (relative formatting)
- **Thermal Readings notes:** auto-adds the route to Work Assigned if not already there; prompts to save note permanently to route page


### Bug Fixes (continued)
- **Duplicate route in Work Assigned:** Fixed — route can no longer be added twice. Existence check now correctly matches on routeNumber/equipment field. addThermalNoteToWork no longer creates its own job entry.
- **Ctrl+V paste fix:** Paste now works in Edit mode (detail overlay open), uses FileReader directly as fallback if DataTransfer isn't supported, shows error toast if paste fails. Also works with ._pastedFile fallback for save functions.
- **Delete safeguards:** Delete button removed from top of detail pages. Deletion now requires: (1) scroll to bottom "Danger Zone", (2) type "DELETE" in confirmation field, (3) confirm via modal dialog. Triple-safeguard prevents accidental deletion.


### Bug Fixes
- **Work Assigned auto-add:** Fixed — route now correctly appears in Work Assigned after saving thermal readings. Root cause: Firebase `orderByChild` queries require a pre-configured index; replaced with client-side filtering.
- **Delete refreshes screen:** Fixed thermal history container ID — deleting/editing readings now immediately updates the displayed list without needing a tab switch or page refresh.
- **Enter key support:** Enter now triggers actions on: Search box, Thermal route input, Note input fields. No need to click buttons.
- **User filtering:** Work Assigned job existence check now correctly filters by current user (won't collide with other users' jobs).

### Thermal Readings Edit/Delete
- **Edit button** on each historical reading entry (author only) - prompts to change individual values
- **Delete button** on each historical reading entry (author only) - with confirmation
- Records who edited and when (editedBy/editedAt stored in Firebase)

### Thermal -> Work Assigned (Fixed)
- Saving thermal readings **always** auto-adds the route to Work Assigned (not just when notes are entered)
- If route is already in Work Assigned, it won't duplicate

### Thermal Anomaly Detection
- After recording thermal readings, compares each value to the **historical average** (last 10 readings)
- If current reading is **>7 degrees C above average** -> warning toast + prompt to add a note
- Only triggers when there's enough history (3+ readings)
- High baselines don't trigger false positives (alerts based on deviation from THAT point's normal)

### Work Assigned Overhaul
- **Job numbering:** Jobs show as Job 1, Job 2, etc. - numbers update when reordered
- **Drag to reorder:** Drag jobs up/down to plan work sequence
- **Mark as Done:** Green highlight, strikethrough - stays in queue for reference
- **Overdue (12h+):** Red highlight + "OVERDUE" badge (not auto-deleted anymore)
- **Delete individual jobs** + **Clear All** button to empty the queue
- **Notes on jobs:** "Add Note" under each job. Prompts to save to Route/Position permanently
- **Job counter in tab:** "Work Assigned (4)" shows active job count
- **"Add to my Work Assigned" button** on every detail page
- **Permanent work log:** Completed and overdue jobs logged for future team dashboard

### Mobile Optimization
- Flex header (title hidden on mobile, compact buttons)
- Scrollable tables, larger touch targets, iOS zoom prevention
- Logout/Switch User dropdown from header username

---

## v1.01 — Bug Fix + Tab Rename
**Date:** 2026-07-03  
**Status:** Deployed

### Fixes
- **Critical:** Removed stray `async` keyword (line 1171) that caused `ReferenceError: async is not defined` — broke all JS execution (login, tabs, search, everything). Bug originated in the build session, not a deployment error.
- **Tab renamed:** "Add Equipment" → "Manage Equipment"

### Additions
- **Logout / Switch User:** Tap the username in the header → dropdown with Switch User and Logout options
- **Mobile optimization:** Flex header (title hidden on mobile), compact tabs, larger touch targets, iOS zoom prevention, scrollable tables

---

## v1.00 — Initial Full Release

**Date:** 2026-07-03**Status:** Deployed**File:** `index.html` (163 KB)

### Overview

Complete rebuild of the Reliability App from the original proof-of-concept into a production-ready tool with 6 tabs, login system, work assignment tracking, and comprehensive CBM route management.

---

### 🔐 Login System

- Login prompt on first open (Login ID, no password)
- Remembers user on device (localStorage)
- "Continue as [user]?" on returning visits
- **Switch User / Logout** via clickable username in header
- Auto-fills `recordedBy` for thermal readings and work entries

### 🔍 Search

- **Fuzzy word matching** — splits query into words, matches if ALL appear anywhere in record
- Searches across: conveyor number, type, APN, store location, part type, route number, start/end points, waypoints, notes/description
- **Keyword shortcuts:** type "Ultrasound" / "Thermal" / "CBM" to see all matching routes
- **Clickable results** — each result is a hyperlink opening a detail page
- **Detail page** shows: all record fields, related parts, related routes, images
- **Edit / Delete buttons** on detail page
- **"Record New Readings"** link on thermal route details → pre-fills Thermal tab

### 🔧 Add Equipment

- Equipment types: Nitta, Transnorm, Transport conveyor, V-Belt, Intralox, SL2, Manual Induct Belt 1/2-4/Finger Belts, ECC, SC3, Ambaflex Spiral, Gappers, Fives, SWD, Other
- **Non-standard equipment IDs supported:** CB.SRT.01.ID22.01, SLAM.314.DVT1, 180000A (SPRL with letter suffix)
- Validation: pure digits = 6 required; alphanumeric = accepted; SPRL = 6 digits + optional letter
- SCADA screenshot upload: **click, drag & drop, or paste (Ctrl+V)**
- "Have you used any parts?" popup after save → quick-add parts inline
- **Equipment Description** field (separate from Notes)
- **Notes** field: textarea, 500 char limit with counter
- Bulk import: CSV/Excel with drag & drop

### 🔩 Add Parts

- Part types: Drive roller, End roller, Tension roller, Snubber roller, Bearing, Motor gearbox, Pulley A-F, Belt, Chain, Sprocket, Guide rail, Sensor, Other
- Duplicate APN detection (prompts to edit existing)
- Conveyor linking

### 📋 Add CBM Routes

- Route types: Ultrasound, Thermal, **Other**
- **Equipment Description field** (mandatory, red asterisk) — auto-filled from APM checklist "Equipment Description" column on route header line (CBM.ULTS.XX)
- **End Point is optional** (only start point mandatory)
- Waypoints: add, remove, drag to reorder
- Route location screenshot: click, drag & drop, or paste
- **Bulk Import (APM Checklist):**- Drag & drop Excel file onto import zone
- Auto-detects CBM.ULTS.XX and CBM.THERM.XX route headers
- Extracts conveyor numbers (last 6 digits), sorted lowest→highest
- Equipment Description captured from route header row
- Preview table before importing (shows Route, Type, Equip. Desc., Start, End, Waypoints)
- Smart duplicate handling — prompts to update or skip existing routes
- Route diff: shows what changed if re-importing an existing route
- Auto-creates equipment entries for new conveyors (blank notes)
- No Excel files stored in Firebase — processed client-side only

### 🌡️ Thermal Readings

- Motor displayed **above** Gearbox (consistent order)
- Recording form only in Thermal tab (history moved to search detail page)
- Login ID auto-filled from app login
- Excel export & print blank sheet

### 📝 Work Assigned (NEW TAB)

- Record daily assigned work per technician
- Work types: **FWO, PM, CBM**
- **FWO:** equipment + parts (Part Type dropdown + APN). If part exists → auto-links. If new → asks for store location + saves automatically
- **PM:** equipment only. If not registered → offers to save + optional SCADA screenshot
- **CBM:** choose Ultrasound/Thermal → route number. If exists → hyperlinked. If not → offers to create
- Hyperlinked routes and equipment (click to open detail)
- Edit / Delete buttons on each job
- **Auto-cleanup:** entries deleted 12 hours after creation (client-side on app load)
- Per-user: each technician only sees their own jobs

### 📱 Mobile Optimization

- Flex header: logo + username left, buttons right, title hidden on mobile
- Compact tabs (11px), scrollable
- Full-width inputs at 16px (prevents iOS auto-zoom)
- Tables horizontally scrollable
- Reduced padding and margins
- Toast notifications centered at bottom
- Detail page responsive

### ⚙️ Infrastructure

- **Firebase offline persistence** enabled
- **Database cleanup on login:** routes with endPoint === startPoint → endPoint cleared
- **Edit bug fix:** changing route/equipment number during edit correctly deletes old key + creates new (no orphaned records)
- Images: auto-resized to max 1200px, JPEG 0.85 quality, stored as Base64 in Firebase RTDB
- Soft delete (7-day trash) for manual deletions
- Full database backup/export as JSON (⬇ Backup button)

### 🔄 Auto-Backup (PowerShell Script)

- **Script:** `ReliabilityAppBackup.ps1`
- Runs daily via Windows Task Scheduler (no browser needed)
- Pulls full Firebase DB via REST API
- Saves to `\\ant.amazon.com\dept-eu\LCY2\Support\RME\16 - Reliability\6. Reliability App Back-Up`
- Keeps only 3 most recent backups, deletes older
- Requires one always-on PC with network + internet access

---

## Development History (Pre-v1.00)

| Date | Change |
| --- | --- |
| May 2026 | Initial proof-of-concept (Search, Equipment, Parts, Routes, Thermal) |
| Jun 2026 | Firebase Realtime Database integration, GitHub Pages hosting |
| Jun 2026 | QR scanner, image upload, bulk import, thermal charts |
| Jun 2026 | APM checklist parser, duplicate detection, soft delete |
| Jun 9 | "Other" dropdown, mandatory notes, drag & drop, paste, fuzzy search |
| Jun 9 | Clickable search results → detail page, edit/delete from detail |
| Jun 9 | APM description capture into Equipment Description field |
| Jun 30 | Full rebuild: login system, Work Assigned tab, validation overhaul, thermal restructure |
| Jul 1 | Batched deployment (A → B → C → Final) |
| Jul 3 | Logout dropdown, mobile optimization → **v1.00 release** |

---

## FAQ

**Q: Where is the data stored?**A: Firebase Realtime Database (Google Cloud, europe-west1 region). Project: lcy2reliability-3dafa. No data is stored on any local device except the localStorage login ID.

**Q: Do I need internet to use the app?**A: Yes for first load. Firebase offline persistence caches data, so previously loaded data is viewable offline, but new saves require internet.

**Q: Can multiple people use it at the same time?**A: Yes. Firebase syncs in real-time. Two technicians can add equipment/parts simultaneously without conflict.

**Q: What happens if I import the same APM checklist twice?**A: The app detects existing routes and prompts you to update or skip. Equipment entries that already exist are not overwritten (unless notes are empty and the import has a description to add).

**Q: Are uploaded images stored in Firebase Storage?**A: No. Images are Base64-encoded and stored directly in the Realtime Database. This avoids needing a paid Firebase plan but means images should be kept small (auto-resized to 1200px max width).

**Q: What's the Equipment Description vs Notes?**A: Equipment Description is the formal identifier from APM (e.g. "CBM, ULTS.SHP.DISCHARGE & RCRC"). Notes is free-text for anything the technician wants to record (observations, reminders, etc.).

**Q: Why does Work Assigned clear after 12 hours?**A: It's designed for daily shift use — record your assigned work at the start of shift, reference it during the day, and it auto-clears so the next shift starts fresh.

**Q: Can I recover deleted data?**A: Manual deletions go to a 7-day trash (soft delete). Auto-cleared Work Assigned entries are permanently deleted. The daily auto-backup to the shared drive provides a safety net.

**Q: How do I add a non-standard equipment ID like CB.SRT.01.ID22.01?**A: Just type it in. The app auto-detects: if the ID contains letters or dots, it accepts any format. If it's digits only, it enforces exactly 6 digits.

**Q: What does the Backup button do?**A: Downloads the entire database as a JSON file to your device. This is a personal backup — separate from the automated daily backup to the shared drive.

**Q: Who can access the app?**A: Anyone with the URL (lcy2reliability-dev.github.io/reliability-app). There's no password protection — the login is just an identifier for tracking who recorded what.

---

## Known Limitations

- **No password authentication** — login is an honour system identifier only
- **Image size** — Base64 in RTDB means large images slow the database. Keep SCADA screenshots minimal
- **Firebase free tier (Spark)** — 1GB storage, 10GB/month downloads. Monitor usage if team grows
- **Auto-backup requires** one always-on PC — if the machine is off when the task fires, backup is missed (will run at next boot if Task Scheduler is configured with "run ASAP after missed")
- **Offline writes** — Firebase caches locally but queues writes. If the app is closed before reconnecting, writes may be lost

---

## Future Roadmap

- [ ] Team dashboard (who recorded what, completion stats)
- [ ] Photo annotation (draw on screenshots to mark locations)
- [ ] Barcode/QR generation for equipment labels

---

*Last updated: 2026-07-03*
