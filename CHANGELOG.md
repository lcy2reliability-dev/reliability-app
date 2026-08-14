# Reliability App — Changelog

**Repository:** GitHub → GitHub Pages (lcy2reliability-dev.github.io/reliability-app)
**Platform:** Single-page HTML app with Firebase Realtime Database
**Site:** LCY2, Tilbury
**Maintained by:** Tittus Streuti (strtt)
**Note:** From v1.04 onward, changes are made via Aki, working from a copy in `Reliability App - AKI/`. The Quick-built file is kept untouched as a fallback. Every app change is logged here in parallel — version bumps reflect feature changes; bug fixes are noted under the version they were fixed in without a version bump unless bundled with a feature change.


## v1.17 — 2026-08-13

**Status:** Deployed and login switched ON 2026-08-13. Login experience refined 2026-08-14 (email-confirmation gate; see the dedicated section below). Firebase rules not yet published (database still open).

**Highlights:**
- **Security build, login stays off until switched on.** All the server-side pieces needed to lock the database down are now in the app and ready. Nothing changes for anyone today: the app still opens with no login. Turning the lock on is a deliberate, separate step done in the Firebase console plus a one-line switch (`AUTH_ENABLED`), documented in `SECURITY_SETUP.md`.
- **New people are look-only until an Admin approves them.** Once login is on, the first time someone signs in they can view everything but cannot edit. They see a "waiting for admin approval" banner; an Admin approves them in Manage Users (the existing "Role review required" flag), after which they can edit normally. This stops a stranger who finds the public URL from changing data.
- **Complete database rules.** The rules now cover every part of the app, including the photo stores, the associated-parts catalogue and the Activity Log that were added after the first rules were written. The Activity Log is append-only (technicians can add entries but not edit or delete them; only Admins prune) and indexed on time.
- **More complete backup.** "Download Backup" now also captures route and equipment photos, the associated-parts catalogue and the Activity Log.

### Login experience & email-confirmation gate (2026-08-14)
Refines how people get in and how editing is unlocked. No version bump (still v1.17).

- **Set up your login** now collects the person's **name** and **Amazon email** and asks them to **confirm their PIN**. The Login ID is the part of the email before the @ (e.g. `strtt`). The login credential is standardised on `<alias>@amazon.com`, which is deliverable for every Amazon employee (confirmed by test), so sign-in stays simple (Login ID + PIN) and works regardless of whether someone normally uses `@amazon.co.uk` etc.
- **Approval by an Admin is replaced by email confirmation.** A new person can view immediately; the app emails them a confirmation link (Firebase built-in verification). When they click it and return to the app, editing unlocks automatically to technician level - Admins never need to review anyone. `canWrite()` for a technician now means "email confirmed" (or an Admin override). Admins can still bump anyone to Admin in Manage Users, and the "Mark reviewed" button remains as a manual override for stuck cases.
- **Forgot PIN** link on the sign-in screen sends a reset link to the Amazon inbox (Firebase password reset).
- **Change PIN** replaces "Switch User" in the menu (Logout stays below it). It re-checks the current PIN, then sets the new one.
- **Dashboard greeting** now uses the person's name ("Good morning, Tittus"); everything else (comments, Activity Log, attribution) still uses the alias (`strtt`).
- Sign-in no longer silently creates an account on an unknown Login ID; it points new people to "Set up your login".
- The pending banner now explains "confirm your email to unlock editing" and carries a **Resend link** button. The app re-checks verification when you return to the tab (and on refresh).
- **Fixed:** the guided **Tour** now reliably auto-starts for a brand-new user after their first login. Previously a background timer ran the tour check while the login screen was still up, which quietly used up its one-time trigger before login finished; the check now waits until you are actually in the app. (Existing users who already completed the tour still won't see it again unless they clear the app data / use a fresh browser.)
- **For the eventual locked-down rules:** technician writes should also be gated server-side on `auth.token.email_verified` (noted for the SECURITY_SETUP runbook).


**Detail:**

### Approval gate (look-only until reviewed)
- New self-registrations are created as `technician` with `reviewed:false`. Until an Admin approves them, `canWrite()` returns false, so every edit is blocked client-side (with a clear "waiting for approval" message) and server-side by the rules. A persistent amber banner explains the state and the body carries a `role-pending` class. Admins and already-approved technicians are unaffected.
- Approval reuses the existing "Role review required (N)" flag and the "Mark reviewed" button in Manage Users — no new screen.

### Firebase rules (see SECURITY_SETUP.md / firebase_rules_v117.json)
- Rewritten for the two-role model and expanded to every node the app now uses: `equipment`, `parts`, `routes` (create/edit for approved technicians, delete Admin-only), `readings`, `workAssigned`, `notes`, `routeImages`, `equipmentImages`, `associatedParts` (approved technician + Admin), `trash` and `users` (Admin-managed; self-registration limited to your own reviewed:false technician record), and `auditLog` (Admin read/prune, append-only for approved users, `.indexOn:["ts"]`).
- Self-registration cannot self-approve or self-promote: the rule forces `role=technician`, `reviewed=false` on the first write.

### Housekeeping
- `AUTH_ENABLED` stays `false` in this build — the security code is dormant until the console rollout. Download Backup node list corrected (added the photo/catalogue/audit nodes, dropped the unused `workLog`). Version bumped to 1.17 and the What's New banner refreshed. `SECURITY_SETUP.md` rewritten with the two-role model, the approval flow and an ordered switch-on runbook.


## v1.16 - 2026-08-13

**Status:** Not deployed yet — deploys together with v1.17.

**Highlights:**
- **Two access levels instead of three.** The old view-only "viewer" role has been retired. Everyone is now either a **Technician** (record readings, link and unlink parts, manage the Job list, add notes) or an **Admin** (everything, plus user management, imports, deletes and the new Activity Log). Any existing viewer accounts are treated as Technicians.
- **New Activity Log (Admins only).** A running, append-only record of who changed what and when: adds, edits, links and unlinks, deletes, readings, notes, jobs, role changes and imports. Open it from the Menu, filter by user, action or item. Entries older than 12 months are removed automatically.
- **Links now remember who created them.** Every part-to-position link records who made it. Once login is enforced, a link can be removed only by the person who made it or by an Admin, so you cannot accidentally undo someone else's work. Legacy links with no recorded owner are Admin-only to remove.
- **The guided tour is now role-aware.** Technicians get the core tour; Admins get extra steps covering the admin tools and the Activity Log. If you have already seen the tour and later become an Admin, you are shown just the new admin steps. "Take the Tour" in the Menu always replays the full tour for your role.

**Detail:**

### Roles collapsed to two (Technician / Admin)
- `normalizeRole()` maps anything that is not `admin` to `technician`, so legacy `viewer` records read as `technician`. All role defaults, the sign-up path, the Manage Users dropdown and the role-change validation now use `['technician','admin']`. View-only wording was removed from the users screen and the login hints.

### Audit log
- A `logAudit(action, entityType, entityId, details)` helper pushes an entry to `auditLog/` on every user-facing mutation: `{ ts, user, role, action, entityType, entityId, details }` with a Firebase server timestamp. It is fire-and-forget and never blocks or fails the underlying action.
- 31 call sites wired in across positions, parts, routes, part links, thermal readings, notes, job-list actions, screenshots and role changes. Bulk equipment import is logged as a single summary entry (imported / skipped counts) rather than one row per record.
- The recorded user is the typed name (no-auth mode) or the login ID (once auth is on). `pruneAuditLog()` deletes entries older than 12 months; it runs on login and whenever an Admin opens the Activity Log.

### Activity Log viewer (Admins)
- `openActivityLog()` shows the most recent 500 entries, newest first, in a filterable table (When / User / Action / Item / Details), reached from a new Menu item. Free-text filters for user, action and item combine.

### Per-user unlink
- Links now store `linkedBy` and `linkedAt` in `parts/<key>/conveyorNotes/<position>`. Unlink is blocked unless you are the link owner or an Admin, on both the position-edit and part-edit screens. Editing a link's Qty/Observations preserves the original `linkedBy`.
- Note: with login not yet enforced, everyone is effectively an Admin, so per-user unlink only starts to bite once the security work lands. Until then it quietly records ownership on every new link.

### Housekeeping
- Version bumped to 1.16 and the What's New banner refreshed. No data migration required: everything above is app behaviour plus new `auditLog/` writes. Existing data is untouched.
- Security follow-up (next project): publish Firebase rules (including `.indexOn: ["ts"]` on `auditLog`), enable backups, then flip `AUTH_ENABLED` to true. Until then the audit log is record-keeping rather than tamper-proof, and per-user unlink is dormant.


## v1.15 — 2026-08-11

**Status:** Deployed 2026-08-13.

**Highlights:**
- **Positions now show their full APM name everywhere**, with the short 6-digit number kept right alongside for scanning. Search results, the position page, route point lists, the Record Readings form, Thermal History, the picker and the printed report all now read the descriptive name (e.g. `AFE2.0.DIS.LN.01.703010`) instead of just `703010` — nothing was renamed in the database, this is a display change and no data was lost.
- **Tap any position, part or route anywhere in the app to open its full detail page.** That includes the APN entries under a position's “Parts Associated” list — they are now links: known parts open the part page, and APNs not yet in the parts database open a small info card built from the imported APM catalogue reference.
- **Decommissioned equipment has been cleared out of the live routes and moved to Inactive Items**, and the affected routes had their end-points corrected.
- **Your Job list order now syncs across devices.** The order you set with the arrows (or drag) on your laptop now shows in the same order on your phone — the plan is saved to the database instead of only to the device you arranged it on.
- **Search now understands a `*` wildcard:** e.g. `CB.SRT.01.*.01` lists all ten induction `.01` positions at once, and `CBM.ULTS.*` lists every Ultrasound route.
- **Filter the Linked Parts and Parts Associated lists on a position.** Each of those two tables now has a small filter box under every column heading (APN, Class, Bin, Qty, Description, Criticality, Observations for Linked Parts; APN, Class, Qty, Description for Parts Associated). Type in one or more boxes to narrow a long list instantly — filters combine, so you can e.g. find a part class with a matching description in one step.

**Detail:**

### Full position names (display only, no data change)
- A `positionName` field (the full APM equipment name) was written to 3,497 position records. The short conveyor/position number is unchanged and stays the scannable ID.
- A shared `posDisp()` helper renders “full name + grey short number” consistently across: position search results and the predictive dropdown, the position detail header, the route detail Route Points list, the Record Readings point titles, the position picker, the Linked Positions on a part page, Thermal History headings, and the printed route report's Route Points list.
- After a follow-up audit, another 173 positions were matched 1:1 to a unique APM name and assigned (143 live, 30 decommissioned). The remaining 169 keep showing their number only: 89 are ambiguous (a short tag like `01` or `12` matches many assets) and 80 are absent from the APM export entirely. See `Position name audit v1.15.xlsx`.

### Cross-linking everywhere
- Every place that names a position, part or route is now a link into that asset's detail view: routes-containing-this-position (Start/End cells), the burger-menu Alerts and Due lists, Linked Parts and Parts Associated tables, and existing search/job links.
- “Parts Associated” APNs: 2,985 of 3,464 associated APNs matched a part record and link straight to the part page; the remaining ones open an info popup (“not in the parts database yet”) from the imported reference so nothing is a dead end.

### Decommissioned points moved to Inactive
- Using a status-aware reconciliation against the APM export (`equipment_status` I = installed, D = decommissioned), 27 reading-points across 5 active routes that mapped only to decommissioned equipment were removed from those routes; none had recorded readings, so no history was lost.
  - `CBM.THERMO.562` and `CBM.THERMO.862`: dropped `180070`, new end-point `180170`.
  - `CBM.THERMO.903`: whole route was decommissioned — set inactive.
  - `CBM.ULTS.46`: dropped `270220/230/240/250`, new end-point `270210`.
  - `CBM.ULTS.114`: dropped 12 dead points, new end-point `182210`.
- 17 decommissioned equipment records were set inactive (they now live under “Manage Inactive Items” and can be restored). End-point rule used: the largest 6-digit number becomes the new end-point.
- All changes were backed up to `Database update\decomm_backup_20260811_141039.json` before applying.

### Job list order now syncs across devices
- Previously the order you arranged your jobs in was saved only to that device's browser (`localStorage`), so a plan built on the laptop never reached the phone.
- The order is now stored in Firebase as a `sortIndex` on each job and read back when the list renders, so any device signed in as you shows the same order in real time. Re-ordering writes are debounced (rapid arrow clicks = one save). Completed jobs still sort to the bottom.
- Note: the very first time you open the list after this update your existing order resets to newest-first — just re-arrange once and it will stick everywhere.

### Route point ordering (unchanged, confirmed)
- Route detail and the Record Readings form stay numerical ascending; Thermal History and the printed / Excel report stay in APM walk order (as decided in v1.13).

### Wildcard search
- Search now accepts a `*` wildcard so you can pin the parts of an ID you know and let the rest vary. `CB.SRT.01.*.01` returns all ten induction `.01` positions (ID11.01–ID25.01); `CBM.ULTS.*` lists every Ultrasound route; `*.ID23.*` returns the five ID23 sub-positions.
- Rules: dots stay literal, each `*` matches any run of characters, and the pattern is anchored at both ends (so `CB.SRT.01.*.01` excludes `.02`–`.05`). It applies to the ID field of positions, parts and routes, in both the full search and the live dropdown. A query with no `*` behaves exactly as before — nothing existing changes.

_Note: v1.15 ships together with the still-undeployed v1.12 hotfix, v1.13 and v1.14 in one deploy._

## v1.14 — 2026-08-06

**Status:** Not deployed yet.

**Highlights:**
- **Redesigned home screen** — the landing page now leads with a scan shortcut and three at-a-glance counts: Jobs assigned, Jobs overdue (shown in red) and Open thermal alerts. Below that, a “What’s left” panel breaks your open jobs down by type (e.g. 3 thermal routes, 1 ultrasound route, 1 PM). On phones it uses a field-first layout with a large “Scan a QR code” button; on laptops it stays a compact console. The old “Due for reading” tile was removed.

**Detail:**

### New landing dashboard (mobile field-first / desktop console)
- `renderDashboard` was rewritten to read live from `workAssigned` (filtered to the signed-in user) instead of counting rendered cards. Counts:
  - **Jobs assigned** = your jobs not yet marked done.
  - **Jobs overdue** = open jobs carried past the 4 a.m. work-day rollover, using the exact same rule as the Assigned Jobs tab (`createdAt` before the most recent 04:00). The tile turns red when the count is above zero.
  - **What’s left** = your open jobs grouped by work type; CBM jobs are split into Thermal / Ultrasound / Other by their CBM type. PM and FWO show as single lines for now (a fuller PM/FWO breakdown is planned for a later release).
  - **Open alerts** = the existing thermal anomaly scan (unchanged).
- The “Due for reading” tile was removed from the dashboard per field feedback. The 4 a.m. sweep and overdue logic themselves are unchanged — the dashboard only surfaces them.
- Responsive from one set of markup: at ≤ 640 px (phones) it shows the large cyan “Scan a QR code” hero and stacks the KPIs; above that (laptops) it shows a three-across KPI console, with scanning available from the existing camera button next to the search box.
- The scan shortcut opens the existing QR scanner (positions and parts).

### Home tab refinements (mobile field feedback)
- The **Search tab is renamed to Home**, since it now carries both the dashboard and the search bar.
- **Mobile search results now wrap correctly.** Long part descriptions were running off the right edge on iPhone (an iOS Safari flexbox quirk with the responsive result cards), which pushed the whole page sideways. The result cards were switched to a wrap-safe layout, so descriptions wrap onto multiple lines and the page no longer scrolls horizontally.
- **The camera button next to the search box is hidden on phones** (it duplicated the big “Scan a QR code” button on the home screen). It stays on laptops/desktop, where there is no scan hero.
- **The dashboard is now pinned to the Home page.** Returning to Home from a detail view used to leave the page blank until you tapped the RME logo; now the dashboard re-appears automatically whenever there is no active search.
- **“Assigned Jobs” is renamed to “Job list”** (the tab, its page heading and all the related messages), and the job buttons on search/detail views now read **“+ Add to Job list”** / **“− Remove from Job list”**. The dashboard’s “Jobs assigned” count keeps its name (it is a number, not the list itself).

### Link a position from the part screen (two-way linking)
- The part detail’s **Linked Positions** card now has a **“+ Link Position”** button. Until now you could only link a part *from* a position (the “+ Link Part” button on a position); you can now do it from either side.
- Tapping it opens a small form where you **scan or type a conveyor / position number** (with type-ahead suggestions), then enter the **quantity** and any **observations** — the same details as the existing part-linking flow. Confirming links the part to that position, records the quantity/observations, and (as before) removes it from that position’s APM “associated parts” suggestions.

### Route Points shown in numerical order on the route screen
- On a route’s detail screen the **Route Points** list is now sorted in numerical ascending order (matching the Record Readings form). It previously showed the APM walk order. The **Start / Waypoint / End** tags follow that same order to reflect the physical walk — the lowest-numbered point is Start, the highest is End, everything in between is a Waypoint. Thermal History (and the printed report / Excel export) is unchanged — it stays in APM walk order.

## v1.13 — 2026-08-06

**Status:** Not deployed yet.

**Highlights:**
- **Thermal History is now locked to APM order, for good** — the thermal history on a route's detail screen always lists reading points in APM walk order, and stays that way even if someone re-orders the route's points by editing the route. Each route now carries its own authoritative APM sequence (imported directly from APM), and the history reads from that instead of from the editable points list. The Record Readings form and the Route Points list stay in numerical order as before.

**Detail:**

### APM order decoupled from the editable route
- Previously the thermal history followed the order of the route's `motors` (reading-points) list. That list is rebuilt whenever the route is edited, and gets rebuilt in numerical order, which would silently undo any APM ordering. With 100+ people able to edit routes, that ordering was fragile.
- Each route now stores an authoritative `apmOrder` field (the list of position tags in APM walk sequence, imported from the APM export). Two new helpers — `apmOrderKeys` (for the history's point headings) and `apmOrderComponents` (for the print/PDF report and the Excel export) — sort by this field, falling back to numerical order when a route has no `apmOrder` recorded.
- `apmOrder` was imported for 639 routes (every route whose points all resolve against the APM export, including all 51 routes that currently have readings). The 15 routes that don't cleanly resolve keep the previous numerical-order behaviour via the fallback.
- The field is additive: it does not touch the `motors` list, so editing a route (which rebuilds `motors`) no longer affects the history order. Readings are keyed by component name, not by position, so re-ordering never detaches a reading.
- No stored readings are changed. This complements the v1.12 display-time ordering by giving it a stable source of truth that route edits cannot disturb.

## v1.12 — 2026-08-03

**Status:** Not deployed yet.

**Highlights:**
- **Screenshot on the job card** — the Assigned Jobs list now shows the route (or position/SCADA) screenshot right on each job card, in the space beside the job title. Tap it to open fullscreen and pinch to zoom, the same as anywhere else in the app. Cards grow a little to fit the image while keeping its aspect ratio.
- **Readings appear in the order that makes sense** — the Record Readings form now lists points in tag/number order (703245 before 703250, ID12.05 before ID13.01), so entering readings follows the physical walk. Reviewing a route's thermal history now always shows past readings in APM order (the order the components appear in the route), no matter what order they were recorded or edited in.
- **Edit a linked part's quantity and observations** — when you edit a position, each linked part now shows its current quantity and observations and has an "Edit Qty/Obs" button next to Unlink. You can add or correct these at any time, not only when you first link the part.
- **Parts Associated protected from accidental removal** — the remove (✕) button in the Parts Associated list is now hidden until you tap "Edit list" in that section. The "Link" button is always available; you can only remove an imported reference after deliberately entering edit mode.
- **New app logo** — the header now shows the proper Amazon "amazon rme" wordmark with the real Amazon smile, in white, replacing the old hand-drawn arrow. It reads cleanly on the dark header.
- **Security and quality hardening** (bundled with this release, no separate version bump) — text you type (descriptions, notes, waypoints, part/supplier fields) is now fully escaped everywhere it is shown, so it can never be treated as page code; the six external libraries the app loads (Firebase, Chart.js, XLSX, the barcode scanner) are pinned with integrity checks so a tampered copy is rejected; keyboard focus outlines and image alt text were added for accessibility; and an unused block of leftover styling was removed.

**Detail:**

### Assigned Jobs screenshot thumbnail
- Each job card renders a thumbnail of the linked route screenshot (CBM route jobs) or position SCADA screenshot (equipment jobs), sized to fit within roughly 210×110 px on desktop and 120×78 px on mobile, aspect ratio preserved.
- The image loads on demand using the same lazy-load path added in v1.11 (`routeImages` / `equipmentImages`), and is cached per session so re-rendering the list (e.g. ticking a job done) does not re-download it.
- Tapping the thumbnail opens the existing fullscreen viewer with pinch-to-zoom / pan / double-tap, and does not expand the card.
- Jobs with no screenshot simply show no thumbnail; the card layout is unchanged for those.

### Reading order
- The **Record Readings** input form now sorts reading points by tag using a natural sort, so `703245` comes before `703250` and `CB.SRT.01.ID12.05` before `CB.SRT.01.ID13.01` (and 9 before 10, not "10 before 9"). This matches the order you physically walk the route. Step-by-step mode follows the same order.
- The **thermal history** on the route detail screen now shows each reading's components in APM order (the order they appear in the route's list), regardless of the order the reading was recorded or later edited in. This is a display-time ordering only: no stored data is changed and nothing is migrated. A component that is no longer in the route's current list is shown at the end rather than dropped, so nothing disappears from history.

### Linked part quantity & observations
- A part's quantity and observations for a position (stored as `conveyorNotes`) could previously only be entered at the moment you linked it. If you skipped them, or needed to correct them later, there was no way back in.
- The position edit screen's "Linked Parts" list now shows each part's current Qty and observations inline, with an "Edit Qty/Obs" button that opens a small dialog pre-filled with the current values. Saving writes the updated note (`parts/<part>/conveyorNotes/<position>`); no other data is touched, and the list refreshes in place.

### Parts Associated — remove hidden behind Edit mode
- The Parts Associated list (the imported catalogue reference) showed a red ✕ to remove an entry, which was easy to hit by accident. That ✕ is now hidden by default and only appears after you tap "Edit list" in the section header (admin only). Tapping it again ("Done") hides the remove buttons. Linking a part is unaffected and always available.

### New app logo
- The header logo was replaced with the Amazon "amazon rme" wordmark using the correct Amazon smile. "amazon" is shown in white (reversed for the dark navy `#232F3E` header) with "rme" and the smile in the app cyan (`#00A8E1`). It is embedded inline as a PNG at 28 px height, in place of the previous hand-drawn SVG arrow, whose shape never looked quite right.

### Security and quality hardening (bundled, no version bump)
- **Cross-site-scripting cleanup.** Every place that displays text entered by users — equipment/route/part descriptions, notes, waypoints, APN, bin location, manufacturer and supplier fields, and thermal component names/values — now passes through the standard HTML-escaping helper, closing several XSS gaps that remained in the edit forms and thermal history. The note-edit button (which lives inside an `onclick` handler) uses a dedicated helper that also neutralises single quotes and backslashes so a note can't break out of the handler. The old partial `.replace(/"/g,'&quot;')` pattern (which missed `< > &`) was replaced throughout.
- **Subresource Integrity (SRI).** The six external scripts loaded from CDNs — three Firebase 9.23.0 modules, Chart.js 4.4.0, XLSX 0.18.5, and the ZXing 0.19.1 barcode library — now carry SHA-384 `integrity` hashes and `crossorigin="anonymous"`, so the browser refuses to run them if the delivered file doesn't match the expected content.
- **Keyboard focus visibility.** Added a `:focus-visible` outline for links, buttons, and form controls so keyboard users can see where they are (previously only text inputs had a focus style).
- **Image alt text.** Added `alt` descriptions to the four SCADA/route detail-screen images.
- **Dead CSS removed.** Deleted an orphaned `.work-job-card` style block (about 16 rules); the Assigned Jobs cards actually use `.work-card`, so this styling was never applied.

## v1.11 — 2026-07-31

**Status:** Deployed 2026-07-31; migration complete (cache 6.67 MB → 1.52 MB).

**Highlights:**
- **Faster startup, especially on mobile data** — route and position photos are no longer bundled into the main data the app downloads on open. They now load on demand, only when you open that specific route or position. The core data pulled on startup drops from about 6.7 MB to 1.6 MB, so searching and browsing feel quicker on a phone in the building.

**Detail:**

### Image split (startup performance)
- Route screenshots and position (SCADA) screenshots were previously stored inline inside the `routes` and `equipment` records. Because the app loads those records in bulk for search and matching, every screenshot was downloaded on startup even though a screenshot is only ever shown when you open one route or position. 89 route images (4.3 MB) and 14 position images (0.8 MB), about 5.1 MB in total, rode along on every load.
- Images now live in separate `routeImages` / `equipmentImages` stores and are fetched individually when a detail or edit screen opens (each is roughly 50 KB, so it appears instantly). The `routes` data the app caches shrank from 4.9 MB to 0.6 MB.
- No visible change to how you add, view, or replace a screenshot: same buttons, same screens. Existing images were migrated automatically, so there is nothing to re-upload.


## v1.10 — 2026-07-29
**Status:** Deployed 2026-07-29; field-tested 2026-07-31. This upload folded every unreleased change into a single drag-drop: the v1.10 batch below plus the v1.09 items (iOS scanning, smart scan routing, search-box upgrades, results summary, code-audit fixes) and the older v1.07/v1.08 items still pending (Parts Associated, route audit, overnight-work fix, dead-code sweep, DB cleanup).

**Highlights:**
- **Home dashboard** — the Search tab now opens on a small dashboard: a greeting, today's date, a big Scan button, and three tap-through tiles (Pending jobs, Open alerts, Due for reading). If any routes need attention, the top three appear as red cards you can tap straight to the route.
- **Open Alerts view** — a new burger-menu entry lists every thermal route whose most recent reading is more than 10°C from the component's own baseline (ISO 13373). A red badge on the burger button shows the count. The same screen has a "Due for a Reading" section for thermal routes recorded before but not in the last 7 days.
- **Record form redesigned** — opening a thermal route to record readings now shows a per-component baseline hint (last value, historical average, sample count), and each input tints green / amber / red as you type based on the deviation from that component's baseline. A sticky save bar shows the running "N of M entered" counter.
- **Step-by-step recording mode** — optional mode that walks you through one reading at a time with a large on-screen number pad (great for gloves / cold hands). Inputs become read-only so a stray tap can't overwrite them.
- **Drafts survive interruptions** — partly-typed readings are saved to the phone as you go, so if you lock the phone, get called away, or the app closes mid-route, the numbers are still there when you reopen it.
- **Offline reading buffer** — if you take readings without signal, they save locally and a small "waiting to sync" banner appears at the top. As soon as the phone reconnects, they upload automatically and the banner clears.
- **Grouped anomaly summary** — when a reading trips more than one alert, they now appear in one summary modal with a single shared note, instead of one modal per component.
- **Search results become tap-cards on mobile** — the wide routes and parts tables now stack into readable cards on narrow screens; on desktop they stay as tables.
- **Reorderable work cards** — drag work cards on desktop or tap the ▲ / ▼ arrows on mobile to change job order. Your order is remembered locally per user. Times on cards show "just now / Nm ago / HH:MM / Yesterday HH:MM".
- **Trend sparklines** — route search results and thermal work cards show a tiny 12-reading line chart, colored green / amber / red by the last vs average deviation.
- **Print / Save PDF route report** — every route detail has a Print button that opens a clean printable report (route metadata, points, last 6 thermal readings per component with average, latest anomaly flags, and route notes). No new library; works from the browser's standard print dialog (including "Save as PDF" and iOS Safari's Share → Print).
- **Higher-contrast tabs** — the inactive Search / Work Assigned tab is lighter and heavier, and the active tab has a bright cyan accent so it's obvious which section you're looking at.

**Detail:**

### Home dashboard
- Fills the empty state of the Search results area on load and whenever the search box is cleared. When you run a real search it's replaced by results; clearing the search brings it back.
- Pending jobs count comes from the live Work Assigned list, so it's always in sync with the Work tab.
- Alerts and Due tiles read from a shared thermal-scan (2-minute cache) so opening the dashboard, the burger badge and the Open Alerts view all reuse one set of Firebase reads.
- Big Scan button uses the same universal scanner as the Search box (native `BarcodeDetector` on Android/desktop, ZXing fallback on iPhone).

### Open Alerts + Due for a Reading
- The scan iterates only active thermal routes with defined motors/gearboxes and pulls each route's last 15 readings in parallel. Routes with fewer than four total readings are skipped for alert scoring (need a latest reading plus at least three prior for a baseline).
- Each component's baseline is computed from prior readings only (the reading being scored is excluded), matching the semantics used at record-time. Threshold is 10°C in either direction.
- Due list: thermal routes that have at least one reading on record but whose most recent one is older than 7 days. Never-recorded routes are deliberately excluded so the list doesn't flood with un-initialised routes.
- Alerts sort by biggest deviation first; Due sorts by oldest first.

### Record form
- Layout groups each reading point (Motor + Gearbox) as one card. Motor first, then Gearbox, per the memory convention.
- Baseline hint reads the route's last 10 readings and per component computes: last value, average, and sample count. Fewer than 3 samples shows "no history yet" or "(3+ for alerts)" so it's clear when tinting will kick in.
- Live tint (`recTint`): green under 7°C deviation, amber 7–10°C, red at 10°C or more. Uses each component's own average.
- Sticky bar (`.rec-savebar`) at the bottom shows a running "N of M readings entered" counter and holds the Save / Cancel buttons within thumb reach on mobile.
- Step-by-step mode (`enterStepMode`) hides all but the active point, sets every input `readonly`, and reveals a 3x4 number pad plus Prev / Show all / Next / Save controls. The bottom progress bar fills as you go. Focusing a specific point jumps the step index.
- Drafts (`saveDraft` / `loadDraft` / `clearDraft`): keyed per route in localStorage under `rme_draft_<routeKey>`, storing every non-empty input by its DOM id (plus the note text under `__note`). Fires on every keystroke. Restored on open with a "Draft restored" toast that names the count. Cleared on successful save (online or offline).

### Offline / sync
- `.info/connected` and `navigator.onLine` are watched; if either drops, an offline banner shows at the top of the app.
- Saving a reading while offline stores the payload in localStorage (`rme_pending_readings`) with the route key, route number, readings, note, timestamp, and recorded-by. A "sync status" strip shows the pending count.
- When the app reconnects, `flushPendingReadings` pushes everything to `readings/` and any notes to `notes/routes/`, then clears the buffer and shows a synced confirmation. The push order matches the order captured, so the earliest offline reading gets the earliest server timestamp.

### Anomaly summary
- `_detectAnomaly(component, currentValue, sharedHistory, displayLabel)` is now pure and returns a structured alert (or null). `saveRecordedReadings` runs it in a loop and collects a list before deciding to prompt.
- If any component alerts, one `showAnomalySummary` modal appears with a per-component card (over/under, reading vs average, deviation) and a single note textarea. Save writes one note per component to `notes/routes/<key>` with the prefix `[THERMAL ALERT]`, sharing the user note text at the end.

### Responsive search tables
- The Routes and Parts result tables carry `class="rtable"` with `data-label` on every `<td>`. Under 640 px they collapse to tap-cards using CSS `td::before { content:attr(data-label); }`, with the header row hidden.

### Work reorder + relative times
- Non-done cards carry `draggable="true"` on desktop; the existing drag handlers are now wired up.
- Every card also has a top-right ▲ / ▼ pair that swaps the card with its non-done sibling in the given direction.
- Order is persisted in `localStorage` under `work_order_<login>` on every reorder and on every Done toggle. `loadWorkAssigned`'s sort applies: done last, then saved-order index, then createdAt descending, so a new job lands at the top the first time it appears.
- Added-time and note-times use `formatWorkTime` (just now / N minutes ago / HH:MM / Yesterday HH:MM / date HH:MM) instead of the previous raw locale time.

### Tab contrast
- `.tab` inactive color moves from `#d5dbdb` to `#eaeeef` and weight 500 → 600; `.tab.active` gets a bright `#4cc3ef` foreground, weight 700, and a subtle `rgba(255,255,255,0.06)` background so the active section is obvious.

### Sparklines
- `sparklineSvg(values, w, h)` hand-draws a `<polyline>` plus a last-point dot inside a fixed-size SVG (green / amber / red by last-vs-average deviation, thresholds 7 and 10). No external library.
- `_getSparkValues(rk)` reads the last 12 readings for the route (with a 90-second per-route cache), averages each entry's numeric readings into one trend point, and returns the array.
- Placeholders (`<span class="rspark" data-rk="<key>">`) are injected on thermal-route search rows and thermal CBM work cards; `renderSparklines(container)` fills them asynchronously after render.

### Print / PDF route report
- Every route detail page has a **Print / Save PDF** button (`printRouteReport(routeKey)`). It builds a print-only report into a hidden `#print-root` container, adds `body.printing`, calls `window.print()`, and cleans up on `afterprint`.
- Print CSS hides every direct body child except `#print-root`, sets `@page { margin: 14 mm }`, and avoids page-breaks inside sections and table rows.
- Report content: header (title, generated timestamp, user); metadata table (route number, type); description; route points (Start / waypoints / End); thermal readings (last 6 per component with an average column); anomaly flags on the latest reading; the last 30 route notes.
- No new library, no CSP change. Works from every browser's standard print dialog (including "Save as PDF" and iOS Safari's Share → Print sheet).

### Version stamps + caches
- `APP_VERSION` → `1.10`; HTML top comment → v1.10; What's New banner → v1.10 with a new bullet list. New readings invalidate both the alerts scan cache and the affected route's sparkline cache so the dashboard, badge and lists refresh on the next open.


## v1.09 — 2026-07-29
**Status:** Deploying in OPEN mode. This one upload bundles everything pending: the v1.09 items here, the v1.08 items (barcode scan in Search, search-Enter fix), and the v1.07 items (Parts Associated, route audit, overnight-work fix, dead-code sweep, DB cleanup).

**Highlights:**
- **Barcode / QR scanning now works on iPhone** — the scanner previously relied on a browser feature (`BarcodeDetector`) that iOS Safari doesn't have. It now falls back to an in-page scanning library, so iPhones scan too. Android and desktop Chrome/Edge keep using the fast built-in detector.
- **Scan straight to what you need** — scanning a route number jumps directly into recording a reading for that route; scanning a position or part opens its detail page; anything else drops into the search box and runs the search.
- **Search box upgrades** — arrow keys move through the suggestions and Enter opens the highlighted one; a clear (x) button empties the box in one tap; recent items you've opened appear when the box is empty.
- **Torch and buzz while scanning** — a flashlight button appears when the device supports it (for low light), and the phone gives a short vibrate on a successful scan. Both are no-ops on iPhone (the browser doesn't allow them there) and harmless.
- **Results summary + remembered filters** — the results page shows a short count (e.g. "Found 3 positions and 1 route"), and your Positions / Parts / Routes filter choices are remembered between searches.

**Detail:**

### iOS / universal scanning
- When the browser has a native `BarcodeDetector` (Android, desktop Chrome/Edge) the scanner uses it directly — fast, nothing to download.
- Otherwise (notably iOS Safari) it lazy-loads the ZXing library from the existing allowed CDN and decodes from the live camera stream. This is the change that makes iPhones work.
- The rear camera is preferred (`facingMode:environment`); the stream and library are always torn down on close, so there's no lingering camera light.

### Smart scan routing
- On a successful read the scanned value is matched against the data: an exact route number goes to record-a-reading; an exact position or part opens its detail page; otherwise the value fills the search box and runs a normal search.
- A double-fire guard marks the scanner inactive before the first async step so a single scan can't trigger two actions.

### Search box
- Suggestions are keyboard-navigable: ArrowDown / ArrowUp highlight an item, Enter opens the highlighted suggestion (or runs a full search if none is highlighted), Escape closes the list.
- A clear (x) button appears inside the box when it has text and empties it in one tap.
- Recent items (the last few positions, parts, or routes you opened) appear when the box is focused and empty. They're stored locally on the device.

### Torch, vibrate, and results
- The torch/flashlight toggle appears in the scanner only if the camera reports the capability; it toggles the camera track's torch constraint.
- A short `navigator.vibrate` fires on a successful scan where supported.
- The results page prints a count-summary line, and the Positions / Parts / Routes section filters are saved to and restored from local storage.

### Code audit fixes
A static-analysis pass over the whole file (function references, handlers, DOM IDs, Firebase paths, duplicate declarations) found and fixed the following. All are folded into this same v1.09 upload; none change how the app looks or behaves for normal use.
- **Faster, lighter startup** — on load the app was silently subscribing to the entire equipment, parts, and routes data (the routes data alone is several MB of embedded images) to fill three on-screen lists that no longer exist, then discarding it. Those three subscriptions are removed, so the app opens with less network traffic and memory use.
- **"Add it now" for an unregistered route now works** — when adding a CBM job for a route that isn't registered yet, choosing "add it now" used to target an old, permanently-hidden form and appeared to do nothing. It now opens the proper Add Route form with the route number pre-filled.
- **Side-menu no longer pops open unexpectedly** — opening an add-form from outside the side menu (e.g. the "Add New Part" button shown when a part search finds nothing) could accidentally slide the menu open. The add-form now only closes the menu if it was already open.
- **Thermal export filename** — the "Export to Excel" button on a thermal route now names the file with the route number instead of leaving it blank (e.g. thermal_readings_route_CBM.THRM.01.xlsx).
- **Housekeeping** — removed an old unused thermal-save function (superseded by the current save path) plus the three dead list functions above, and corrected a stale code comment. About 110 lines / 8 KB trimmed, no behaviour change.


## v1.08 — 2026-07-29
**Status:** Superseded by v1.09 (never shipped on its own) — its items are included in the single v1.09 deploy. Bundles the pending v1.07 work below (dead-code sweep, overnight-work fix) plus the two new items here.

**Highlights:**
- **Scan a barcode / QR code from Search** — a camera button next to the search box opens a live scanner; a successful scan drops the value into the search box and runs the search automatically. (Rebuilds the old removed `startQRScan` stub properly, wired to a real button.)
- **Fix: search now responds immediately on Enter** — pressing Enter closes the predictive-suggestions dropdown and shows the results straight away; no more clicking elsewhere to dismiss the dropdown first.

**Detail:**

### Barcode / QR scanning in Search
- New camera button (📷) sits to the right of the search input.
- Uses the browser's built-in `BarcodeDetector` with a rear-facing camera (`facingMode:'environment'`). Detects QR plus common 1-D/2-D barcodes (Code 128/39/93, EAN-13/8, UPC-A/E, Codabar, ITF, Data Matrix, Aztec, PDF417) — the exact set is filtered to whatever the device supports.
- Full-screen scanner overlay with a live video preview, a green aiming line, and a Cancel button. On a successful read the overlay closes, the code is written to the search box, a toast confirms it, and `searchAll()` runs.
- Graceful fallbacks: if `BarcodeDetector` isn't available (e.g. iOS Safari) it explains to use the phone's Camera app and paste; if the camera permission is denied or unavailable it shows a clear message and closes. The camera stream is always stopped on close (no lingering camera light).

### Fix: predictive search closing on Enter
- Previously, pressing Enter ran the search but the suggestions dropdown stayed open (a debounced suggest timer re-opened it ~300 ms later), so you had to click elsewhere to clear it.
- `searchAll()` now cancels any pending suggest timer and hides the dropdown as its first action; the Enter handler also blurs the input (closing the on-screen keyboard on mobile). Clicking the Search button behaves the same way.


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


## Cleanup — dead-code sweep (2026-07-29)
**Status:** In working file, deploys with v1.07. No behaviour change — only unreachable code removed.

Removed 19 functions that were never called anywhere in the app (leftovers from earlier Quick iterations, since superseded by the current wired equivalents), plus two orphaned globals and some dead CSS:

- **Removed functions:** addRouteToWork, addWorkNote, burgerBulkImport, clearAllJobs, clearForm, deleteEntry, deleteJob, detailDelete, filterTable, loadThermalRoute, printBlankSheet, relinkRoute, removePicture, replacePicture, saveNewRoute, startQRScan, toggleRouteActive, stopQRScan, and the already-noted markJobDone/checkOverdueJobs.
- **Removed globals:** `qrStream` (only used by the dead QR-scan stub).
- Live functionality is unaffected — deletes go through `deleteWork`/`confirmDetailDelete`, notes through `saveWorkJobNote`, thermal recording through `goRecordThermal`, route creation through the import flow, etc. Every remaining on-click handler was verified to resolve to a defined function.

**Net:** ~300 lines removed, file ~365 KB (was ~385 KB). Syntax verified, braces/parens balanced.

**Note — QR/barcode scanning:** the removed `startQRScan` was a stub for scanning a position's QR/barcode to auto-fill the number. It referenced UI elements that no longer exist and was never reachable. If we want QR scanning as a real feature later, it should be rebuilt and wired to a button properly.


## Fix — uncompleted work no longer deleted overnight; overdue now shows in red (2026-07-29)
**Status:** In working file, deploys with v1.07

**The bug you hit:** thermal routes left assigned but not marked complete were being *deleted* at the 4am day-boundary. `cleanupOldWork()` was removing every job older than the cutoff regardless of status.

**Fixed / added:**
1. **Uncompleted jobs carry over.** The overnight cleanup now only removes jobs that were already marked **done** before the cutoff. Anything still pending stays in your queue the next day.
2. **Overdue jobs show in red.** A pending job created before today's work-day boundary now renders as a red card with an **OVERDUE** badge. (The old `checkOverdueJobs()` never worked — it read a `createdAt` attribute the card never set, styled a class name that didn't match the card, and only ran once so live updates wiped it. Overdue state is now computed inline on every render, so it's always correct.)
3. **Completing an overdue job clears it instantly.** Toggling an overdue carry-over to done removes it from the queue immediately (rather than leaving it greyed until the next 4am sweep). Same-day jobs still grey out and clear on the normal overnight cycle.

**Dead code removed:** the broken `checkOverdueJobs()` function, the unused `markJobDone()` function, and two orphaned `.work-job-card.overdue` CSS rules. (Noted: the wider `.work-job-card` CSS block, ~14 rules, is entirely unused — safe to remove in a later tidy-up.)


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
