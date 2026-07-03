# Reliability App — Changelog

**Repository:** GitHub → Netlify (reliabilityapp.netlify.app)**Platform:** Single-page HTML app with Firebase Realtime Database**Site:** LCY2, Tilbury**Maintained by:** Tittus Streuti (strtt)

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
| Jun 2026 | Firebase Realtime Database integration, Netlify hosting |
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

**Q: Who can access the app?**A: Anyone with the URL (reliabilityapp.netlify.app). There's no password protection — the login is just an identifier for tracking who recorded what.

---

## Known Limitations

- **No password authentication** — login is an honour system identifier only
- **Image size** — Base64 in RTDB means large images slow the database. Keep SCADA screenshots minimal
- **Firebase free tier (Spark)** — 1GB storage, 10GB/month downloads. Monitor usage if team grows
- **Auto-backup requires** one always-on PC — if the machine is off when the task fires, backup is missed (will run at next boot if Task Scheduler is configured with "run ASAP after missed")
- **Offline writes** — Firebase caches locally but queues writes. If the app is closed before reconnecting, writes may be lost

---

## Future Roadmap

- [x] Team dashboard (who recorded what, completion stats)
- [x] Export to APM format for upload back to HxGN EAM
- [x] Photo annotation (draw on screenshots to mark locations)
- [x] Barcode/QR generation for equipment labels

---

*Last updated: 2026-07-03*
