# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page Arabic (RTL) registration form for a speed-racing event ("SPEED") run by the
Palestinian Motorsport and Motorcycle Federation (PMMF), plus its Google Apps Script backend.
There is no build step, no package manager, and no framework — it's one big self-contained HTML
file with inline `<style>` and `<script>`, backed by a Google Sheet.

This started as a fork of a drift-racing registration form (`aladadweh/pmmf-drift-registration`)
that's now a fully separate, independent product: its own repo, its own Google Sheet, its own Apps
Script deployment, its own Drive folders (`PMMF_SPEED_Registrations` / `الاشتراكات_سبيد`). No data
or deployment is shared with the drift form — do not point `CONFIG.API_URL` back at the drift
deployment.

Live site: https://aladadweh.github.io/pmmf-speed-registration/ (GitHub Pages, repo
`aladadweh/pmmf-speed-registration`, branch `master`).

## Files

- `pmmf_speed_registration.html` — the entire frontend (~1MB, mostly because the hero photo
  and both logos are inlined as base64 `data:` URIs). Deployed as-is to GitHub Pages.
- `pmmf_backend_apps_script.gs` — Google Apps Script backend. **Gitignored** — never committed,
  because it holds `DEFAULT_ADMIN_PASSCODE` in plaintext, which guards registrants' PII (names,
  national IDs, phone numbers) in the Google Sheet/Drive. Exists only on disk locally and in the
  live Apps Script project.
- `mock_server.js` — **Gitignored**. A local Node server that mirrors the `.gs` backend's action
  handlers in memory, for testing the form without touching the real Google Sheet.
- `index.html` — meta-refresh redirect to `pmmf_speed_registration.html`, because GitHub Pages
  serves `index.html` at the root and the real file isn't named that.
- `.nojekyll` — required so GitHub Pages serves the files as-is (Jekyll's build otherwise errors
  on the file, since it isn't valid Jekyll input).

## Running locally

```
node mock_server.js
```

Serves the form at `http://localhost:8787/`, patching `CONFIG.API_URL` (via regex replace on the
served HTML) to point at its own `/api` endpoint instead of the real Apps Script URL. Registration
data lives only in memory and resets on restart. Default admin passcode: `PMMF2026`.

There is no test suite, linter, or build command in this repo — verify changes by running the
mock server and driving the page in a browser (e.g. via a browser automation tool), since this is
a form whose correctness is mostly about UI flow and validation, not unit-testable logic.

After editing `pmmf_speed_registration.html`, **restart** `mock_server.js` only if you changed
`mock_server.js` itself — it re-reads the HTML file fresh on every request, so HTML edits are
picked up without a restart.

## Architecture

### Frontend (`pmmf_speed_registration.html`)

Single IIFE in a `<script>` tag. No dependencies except JSZip (loaded from a CDN `<script src>`
tag, used only to let a submitter re-download their uploaded documents as a zip after success).

- `CONFIG.API_URL` (near the top of the script) is the one line that points the form at a backend.
  Empty string → `boot()` shows a setup banner instead of the form.
- `state` — a single mutable object holding wizard step, uploaded files (as data URLs), and the
  last-fetched `status` (count/cap/daysLeft/closed) from the backend.
- `api(action, extra)` — the only way the frontend talks to the backend: `fetch(CONFIG.API_URL,
  {method:'POST', body: JSON.stringify({action, ...extra})})`. Every backend call goes through
  this one function.
- The registration wizard is 4 steps, each a `stepN()` function returning an HTML string:
  1. `step1()` — competitor info (name, ID, DOB, contact, category, emergency contact)
  2. `step2()` — assistant/co-driver info (name, ID, age — rejected client- and server-side if
     under 19 — phone, city)
  3. `step3()` — car info
  4. `step4()` — document uploads (photos/PDFs compressed client-side via canvas, see
     `compressImage()`) + waiver checkboxes
  Each step has a matching `validateStepN()` and the step count is hardcoded in a few places if
  you ever add/remove a step: `stepBar()`'s `labels` array and width divisor, `goStep()`'s width
  divisor, `wizardScreen()`'s step list, and the button IDs wired in `wireWizard()`
  (`toStep2`/`toStep1`/`toStep3`/`toStep2b`/`toStep4`/`toStep3b` — the naming pattern is
  `to<destination>` for forward buttons and `to<destination>b` for a step's "back" button since
  two different steps both need a button that returns to step 2, etc.).
- `renderAdminPanel()` / `openAdminGate()` — a password-gated admin overlay (triggered from the
  footer) that lists registrants and can delete one (`action: 'delete'`, keyed by national ID) or
  toggle registration open/closed (`action: 'toggleClose'`).
- **Registration-PDF generation happens client-side** (`pdfDocumentHtml()` +
  `generateRegistrationPdf()`, via html2pdf.js from a CDN): `onSubmit()` refreshes `status`,
  predicts the race number as `count + 1`, renders the form copy to a PDF data URL in the
  browser (full CSS fidelity — this exists because Google's HTML→Docs converter strips modern
  CSS), and sends it in the register payload as `pdfFile` + `pdfRegNumber`. The backend only
  uses it if `pdfRegNumber` matches the race number it computes; otherwise (or if the client
  PDF is missing/failed) it regenerates server-side via `buildRegistrationPdfBlob()`. The PDF
  template uses literal hex colors, not oklch/color-mix, and no letter-spacing — html2canvas
  doesn't support them (letter-spacing breaks Arabic ligatures).
- All user-facing strings are Arabic; keep new UI text Arabic and RTL-consistent.

### Backend (`pmmf_backend_apps_script.gs`)

Deployed manually through the Apps Script web editor (there is no CLI deploy for this file) as a
Web App bound to a Google Sheet. `doPost(e)` dispatches on `data.action` to one handler each:
`status`, `register`, `admin`, `toggleClose`, `delete` — this switch and the frontend's `api()`
calls must stay in sync.

- Registrant count/cap/closed state (`getStatus()`) is derived live from `sheet.getLastRow()`,
  not stored separately — so deleting a row automatically frees up a seat, no extra bookkeeping.
- `CAP` and `WINDOW_DAYS` constants gate registration (max participants / days since the sheet's
  `Meta` tab `OpenedAt` timestamp). `OpenedAt` also gates the *start* of registration: if it's in
  the future, `getStatus()` reports `notYetOpen: true` (frontend shows "opens on `Meta` sequence
  `OpenedAt`" instead of the generic closed reason). A brand-new `Meta` sheet seeds `OpenedAt` from
  `SCHEDULED_OPEN_AT` instead of "now" — set that constant when scheduling a future registration
  window before deploying. The admin panel's "تمديد التسجيل" button (`handleExtend`) resets
  `OpenedAt` to the current moment and un-closes, for ad-hoc reopening/extension.
- Sheet columns are positional (`getRange(2, col, ...)` / array indices in `handleAdmin`) — if you
  add/reorder a field in `getSheet()`'s header row or `handleRegister()`'s `appendRow()` call, you
  must update both together, plus any hardcoded column indices in `handleAdmin()`.
- On registration, creates a per-registrant Drive folder (named `fullName - nationalId`) containing
  a text summary (`buildInfoText()`), the uploaded files (decoded from the data URLs the
  frontend sends), and the registration-form PDF (client-generated `pdfFile` when its
  `pdfRegNumber` matches, else server-generated — see frontend section). Server-side PDF
  generation requires the **Drive API** and **Google Docs API** advanced services to be enabled
  in the Apps Script editor; failures are surfaced as a technical note in the federation email
  via the `diagnostics` array, not swallowed.
- Admin actions (`admin`, `toggleClose`, `delete`) all check `data.pass` against
  `getAdminPasscode()` (stored in Script Properties, seeded by manually running
  `setAdminPasscode()` once from the Apps Script editor — never rely on `DEFAULT_ADMIN_PASSCODE`
  in production).

### Keeping frontend, backend, and mock in sync

These three files independently implement the same action contract (`status`/`register`/`admin`/
`toggleClose`/`delete`) and the same registration schema (competitor + assistant + car fields).
When changing one — e.g. adding a field, changing `CAP`/`WINDOW_DAYS`, adding an action — update:
1. The relevant `stepN()` HTML + `validateStepN()` in the frontend
2. `pmmf_backend_apps_script.gs` (sheet header row, `appendRow()` order, and handler logic)
3. `mock_server.js` (in-memory equivalent, so local testing stays representative)

## Deploying

- **Frontend**: push to `master` — GitHub Pages rebuilds automatically. Check build status with
  `gh api repos/aladadweh/pmmf-speed-registration/pages/builds/latest -q .status`.
- **Backend**: `pmmf_backend_apps_script.gs` is gitignored and has no automated deploy. After
  editing it, changes must be manually copy-pasted into the live Apps Script project (via the
  Apps Script web editor) and redeployed as a **new version of the existing deployment** (Deploy →
  Manage deployments → edit → New version) so the `/exec` URL in `CONFIG.API_URL` stays valid.
  Forgetting this step means the frontend silently calls stale backend logic (e.g. missing actions
  return `{error: 'unknown_action'}`).
