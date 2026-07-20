# Ripple Pool Solutions Field App: Developer Guide

This document exists so that any competent developer, human or AI, can maintain this
app safely. Read it fully before changing anything. The single most important fact
about this codebase: **several functions in it compute chemical doses for real
swimming pools.** A bug here is not a rendering glitch; it is the wrong amount of
acid in someone's pool. The engineering culture of this project is built around that
fact.

## What this is

A field management app for a residential pool service company: client management,
visit logging, water chemistry analysis (LSI-based), chemical dosing recommendations,
PDF reports, guided daily route flow, two-device sync, truck load forecasting,
chemical inventory, and internal analytics. It replaces paid SaaS tools with a free,
custom-fit system.

## Architecture: one file, on purpose

The entire app is `index.html`: vanilla JavaScript, inline CSS with variables, no
framework, no build step, no bundler, no package.json. This is a deliberate
architecture, not technical debt. Do not propose splitting it. The reasons:

1. **Deploy is drag-and-drop.** The operator deploys by uploading one file to GitHub
   Pages. No toolchain can break, because there is no toolchain.
2. **No asset paths can 404.** Everything except two CDN libraries and Google Fonts is
   inline, including the app icons (base64).
3. **The app degrades gracefully.** App, then Drive backups, then pen and paper. It is
   an optimization layer over a business that can survive without it.
4. **localStorage is origin-bound.** One file at one stable address means the data
   never moves.

## Stack

- **Hosting:** GitHub Pages (this repo). Live at the repo's Pages URL.
- **Runtime:** Chrome on a Windows laptop; Safari standalone Home Screen web app on
  iPhone. Both must keep working; the phone is the primary field device.
- **CDN libraries:** SheetJS (Excel export), jsPDF (PDF reports). Pinned versions in
  the `<head>`.
- **Weather:** Open-Meteo forecast API (no key), cached about 12 hours.
- **Backend:** a Google Apps Script web app (not in this repo) handling merge sync,
  photo archiving to Drive, an email mailer, nightly snapshots, and analytics. The
  client stores the backend URL and a passcode in localStorage via the sync setup
  screen; neither is hardcoded in this file, so this repo contains no credentials.

## Navigation

The script section opens with a TABLE OF CONTENTS comment listing every section in
file order. Each section starts with a boxed banner comment:

    /* =========== SECTION NAME =========== */

(the real banners use the `═` box-drawing character; search for the section name).
Line numbers are never used for navigation because they go stale.

## Data model

All data lives in browser localStorage, never in this file. Deploys never touch data.

- `ripple_pool_v3`: the main DB, `{ clients:[], visits:[], tomb:[], nid }`.
- `ripple_draft_v1`: in-progress visit draft (includes epoch-ms times that visit
  records do NOT store; see the time-field note below).
- `ripple_summer_v1`: weather cache.
- `ripple_sync_v1`: sync state (backend URL, passcode, cursors, device id).
- `ripple_flow_v1`: day-scoped guided-route state; per-device by design.
- `ripple_epoch_v1`, `ripple_bookinit_v2`: one-time migration flags.
- `ripple_truck_v1`: truck capacities and chemical stock ledger (synced since v7.12;
  its `selected` field is per-device and unused since v7.13).
- `ripple_ppp_v1`: chemical price list (synced since v7.12).

Client and visit records use short field names; the DATABASE and CLIENTS sections
document them inline. Three facts that have bitten before:

1. **Book resolution:** always call `bookOf(c)`; never read `c.bk` directly. A missing
   `bk` means the default book.
2. **Time fields:** visit records store `tstart`, `tend`, `tm`, and a `dur` stamped at
   creation. They do NOT store `tstartMs`/`tendMs`; those live only in the draft and
   window state. Duration must be derived at read time.
3. **Skip records** (`skip:true`) are real records that ride sync but are excluded
   from every analytical computation (dosing history, forecasts, ledgers, reports).

## Sync

Merge-based, per-record newest-`mt`-wins, tombstones for deletions, server-side lock.
Every record mutation must bump `mt` or it will never push (this exact bug once made a
client vanish; the migration now stamps `mt` on everything). Deleting a client
cascades to all their visits everywhere, including the server master; the loud confirm
exists for that reason. Inactive status, not deletion, is how departed clients are
handled.

Since v7.12 the same sync round trip also carries an "admin" section (the
`ripple_ppp_v1` and `ripple_truck_v1` keys) merged by a client-side merge function
that has an exact server-side mirror. **The two merge implementations must stay
logically identical.** Any change to one requires the same change to the other plus a
rerun of the executable parity test (same inputs through both implementations, both
argument orders).

## The chemistry engine and the protected-function doctrine

The dosing engine is the load-bearing wall. The doctrine:

1. **Protected functions are never edited casually.** The current protected list, with
   the app version each one's byte-identity baseline comes from, is maintained in the
   operator's project records. The core members: `calcDose`, `doLSI`, `projAfter`,
   `buildNarrative`, the v7.2 shared report helpers, `generatePDF`,
   `buildEmailHTML`, the Pool Health Report scoring family (`phrPoints`, `phrScore`),
   and the sync and merge functions.
2. **Every build proves the protected list byte-identical** to its baseline, by
   extracting the functions from the delivered file and comparing. When an edit
   intentionally touches a protected function, the proof is that the diff is exactly
   the intended edit and nothing else.
3. **Carried values never dose.** The carry-forward layer (which fills untested
   parameters from history for display and status) feeds the LSI display and records
   only. `calcDose` reads fresh measurements. Do not "improve" this boundary.
4. **Pure-read layers stay pure.** The health reports, forecasts, and ledgers read the
   DB and write nothing. Builds prove this by whole-DB JSON comparison before and
   after a run. Do not add a write to a pure-read layer.
5. **No chemistry factor is ever invented.** Every constant in the engine traces to a
   published source, verified at build time. If no published factor exists for an
   effect, the app does not model the effect.
6. **Missing data refuses, never guesses.** No pool volume means no dose. Internal
   honesty over invented numbers, everywhere.

## Verification protocol (mandatory for every build)

1. Verify the base file is the live deployed version (pull the raw file from this
   repo; checksum it) before any edit.
2. Make edits via anchored, count-asserted operations that fail loudly; a failed
   anchor writes zero bytes.
3. Extract the script block and syntax-check it in Node.
4. Run simulation tests on functions extracted from the delivered file itself, never
   from a separately written reference.
5. Grep every expected feature marker; check `<div>` tag balance.
6. Prove the protected-function list byte-identical (or prove an intentional diff is
   exactly the intended edit).
7. Leak-sweep: internal-only values (anything from the analytics and ledger sections)
   must never reach the client-facing PDF or email builders.
8. Measure every number stated in prose with a tool first. Byte counts come from the
   file system (`wc -c`, `md5sum`), never from decoded string length; this file's
   multibyte characters make the two differ by thousands.
9. When a test fails, determine whether the TEST or the code is wrong before touching
   the app.

## Deploying

1. Upload the new `index.html` to this repo (commit via the GitHub interface or
   GitHub Desktop).
2. The only reliable success signal is the green check next to `github-pages` under
   DEPLOYMENTS on the repo Code page. Budget up to ten minutes for the CDN.
3. Verify with a cache-busting query string, then the browser console: `APP_VER`.
   `APP_VER` is an internal constant and is never rendered in the interface; each
   version also has a documented visual marker for on-screen verification.
4. Data is never touched by deploys: it lives in the browser, bound to the site
   address.
5. The iPhone standalone app has its own cache container. Try a plain open first;
   every update in this app's history has picked up without intervention. Never clear
   Safari website data on the phone (it deletes app data).

## Version conventions

- `APP_VER` at the top of the script section is the version of record.
- v7.5 is permanently reserved for a shelved plan and will never ship; do not reuse
  the number.
- Version history and the reasoning behind every design decision live in the
  operator's project records, which a maintainer should request access to. This
  README is the map; those records are the memory.

## What not to do

- Do not split the file, add a framework, or introduce a build step.
- Do not add paid services of any kind. The app must remain free to run.
- Do not modify a protected function without the full verification protocol.
- Do not make any analytical layer write to the DB.
- Do not invent chemistry factors or soften the engine's refusal-to-guess behavior.
- Do not hardcode weekdays anywhere; route days and close-out days are derived from
  the roster.
- Do not "fix" `renderPPPCard`'s self-no-op guard (its Home mount was removed
  deliberately) or resurrect the removed `tlfSelect` transport selector.
- Do not rename functions for style. The operator's project records reference exact
  function names throughout; greppability against that history is a safety feature.
