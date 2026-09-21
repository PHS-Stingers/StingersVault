# FTC parts loan tracking app planning

## user - 21 Sep 2026, 19:19:18

Let's connect the list of parts into the app now. Create and explain the steps you are going to take. Ask me questions if needed.

## AI - 21 Sep 2026, 19:19:18

Let me investigate the current parts setup before proposing a plan.

*Time to respond: 16s*

## AI - 21 Sep 2026, 19:19:35

A parts scraper and curated catalog already exist in `tools/catalog/`. Let me examine them.

*Time to respond: 5s*

## AI - 21 Sep 2026, 19:19:40

A scraper has already produced a large catalog (`parts_all.json` — 8,081 products from AndyMark, goBILDA, REV). Let me check its size and schema.

*Time to respond: 10s*

## AI - 21 Sep 2026, 19:19:51

Here's what I found and my proposed plan.

## What's already there

A scraper (`tools/catalog/scrape_catalog.py`) has already produced a full sellable catalog:

| File | Contents |
|---|---|
| `parts_all.json` (3.58 MB) | **8,081 products** — AndyMark 5,374 · goBILDA 1,991 · REV 716 |
| `parts_all.csv` (1.9 MB) | Same data as CSV |
| `meta.json` | Counts + tag vocabulary (motor, gear, wheel, fastener, …) |
| `parts_all.json` record shape | `sku, name, vendor, url, nonRobot, categories[], tags[]` |

Meanwhile the app's `assets/parts.json` is still the **14-part starter list** with a different shape: `id, sku, name, vendor(enum), category, ftcLegal, aliases`.

## The gaps I need to bridge

1. **Schema mismatch** — scraped uses `tags[]`/`categories[]`/`url`/`nonRobot`; the app's `Part` uses a single `category`, `ftcLegal`, `aliases`.
2. **Vendor mismatch** — app enum is `GOBILDA / REV / STUDICA / CUSTOM`; the catalog has **AndyMark (the majority, 5,374)** plus goBILDA/REV. Studica is in the app but wasn't scraped.
3. **Scale** — the part picker in `NewLoanScreen` is a plain dropdown. 8,000 items in it would choke; it needs search.
4. **Size** — a 3.6 MB asset in the APK (can be slimmed by dropping the verbose `categories[]`).

## Proposed steps

1. **Add a generator** `tools/catalog/build_app_catalog.py` that turns `parts_all.json` → `app/src/main/assets/parts.json` (app schema): drop `nonRobot` merch, drop verbose `categories[]`, map vendor → enum name, derive `category` from the primary tag, set `ftcLegal=true`, keep `url`.
2. **Update the app model** — add `ANDYMARK` to the `Vendor` enum; optionally add `tags`/`url` to `Part` so search can match on tags.
3. **Add search-as-you-type** to the part picker (replace the dropdown) and the Parts screen, so thousands of parts are usable.
4. **Keep custom parts separate** from the bundled catalog (they're user-made), and decide whether to persist them (and per-team aliases) in Firestore — that was the scoping doc's plan.

Before I build it, a few decisions:

*Time to respond: 13m 17s*

## AI - 21 Sep 2026, 19:33:08

Locked in: full robot catalog, AndyMark included, **Firestore-backed**, custom parts/aliases later. That changes the design meaningfully, so here's the concrete plan.

## What Firestore changes

Putting ~7,950 parts in Firestore raises one real issue: **Firestore can't do substring search, and reading thousands of docs per screen-open would blow through the free tier** (50k reads/day). So the design is: store the catalog in Firestore, sync it to the device once (and only again when it changes), and search **locally**.

## Proposed architecture

**1. Firestore data model**
- `parts/{partId}` — one doc per part: `sku, name, nameLower, vendor, tags[], url`
- `catalogMeta/current` — `{ version, count, updatedAt }`
- Rules: `parts` readable by authenticated + verified users; **no client writes** (seeded out-of-band)

**2. Seeder** — a script that reads `parts_all.json`, drops the 128 non-robot items, maps to the doc shape, batch-writes (~16 batches of 500) and bumps `catalogMeta/version`.

**3. App repository** — `PartsRepository`:
- Read `catalogMeta/current` (1 read).
- If the version is newer than the locally cached one, page through `parts` and cache locally (JSON file in app storage).
- Otherwise, use the cached copy.
- Search runs locally over the cached list (instant, works offline).

**4. App model** — add `ANDYMARK` to the `Vendor` enum; add `tags` + `url` to `Part`.

**5. UI**
- Replace the parts dropdown in `NewLoanScreen` with a search-as-you-type picker.
- Add a search bar to the Parts screen.

**6. Rules & index** — update Firestore rules for `parts` access.

## What I'll need from you
To seed Firestore outside the app, I need a **Firebase service account key** (Firebase console → Project settings → Service accounts → Generate new private key). That's a secret — I'll add it to `.gitignore` and never commit it.

Two final decisions:

*Time to respond: 1m 0s*

## AI - 21 Sep 2026, 19:34:08

Both decisions are in. Here is the complete, final plan.

## Firestore data model
- **`parts/{partId}`** — `{ sku, name, nameLower, vendor, tags[], url, ftcLegal }`
  - `partId` = stable slug like `gobilda-5203-2402-0019` / `andymark-am-1761`
- **`catalogMeta/current`** — `{ version: Int, count: Int, updatedAt: Long }`
- **Rules** (console): `parts` + `catalogMeta` readable by **verified** users; **no client writes**.

## Step 1 — Seeder script
`tools/catalog/seed_firestore.py` (+ `requirements.txt` with `firebase-admin`):
1. Loads `parts_all.json`, drops `nonRobot == true` (~128).
2. Maps vendor: `andymark→ANDYMARK`, `gobilda→GOBILDA`, `rev→REV`.
3. Builds docs with `nameLower` (for future use), `tags`, `url`, `ftcLegal=true`.
4. Batch-writes to `parts` (500/batch ≈ 16 batches) and bumps `catalogMeta.version`.
5. Prints a summary (seeded count, version).

You run it once (and again whenever you re-scrape). It needs the service-account key; I'll add `*.serviceAccount.json` / key patterns to `.gitignore`.

## Step 2 — App model
`data/model/Models.kt`:
- Add `ANDYMARK` to `Vendor` (and its `from()` handling).
- Add `tags: List<String> = emptyList()` and `url: String = ""` to `Part`.

## Step 3 — Repository + local cache
New `data/PartsRepository.kt`:
- Read `catalogMeta/current` (1 read) → compare with locally cached version.
- **Changed** → page through `parts` (`limit(500)` + `startAfter`), cache the list to `filesDir/parts_cache.json`, store the version.
- **Unchanged** → load the cache.
- **Offline / failure** → fall back to cache, then to the small bundled `assets/parts.json`.

## Step 4 — ViewModel
`ui/CatalogViewModel.kt`: source parts from `PartsRepository`; expose `syncing`, `searchQuery`, and `filteredParts` (case-insensitive match on name / sku / tags).

## Step 5 — UI search
- `NewLoanScreen.kt`: replace the parts `SelectField` with a **search-as-you-type picker** (text field + result list, capped at ~50 rows).
- `PartListScreen.kt`: add a search bar above the list.

## Step 6 — Rules
You paste the updated rules (I'll provide them), adding `parts`/`catalogMeta` read for verified users and denying writes.

## Verification
1. Run the seeder → confirm ~7,950 docs in the Firebase console.
2. App: first launch syncs (progress shown), search works, subsequent launches use the cache.
3. Airplane-mode test → catalog still searchable from cache.

## Trade-offs to be aware of
- One-time ~7,950 reads per device on first sync (then ~1 read/day) — comfortably inside the free tier at your scale.
- Search is **local substring**, so it's fast and offline, but the whole catalog is downloaded to the device.
- Re-scraping requires re-running the seeder (it overwrites docs and bumps the version so devices re-sync).

I have everything I need. Shall I implement it? If so, I'll start with the seeder + model changes, and I'll need you to generate the service-account key and drop it somewhere in the repo (I'll tell you the exact path and gitignore it).

*Time to respond: 14s*

## user - 21 Sep 2026, 19:35:19

Go ahead and implement it. Make no mistakes.

## AI - 21 Sep 2026, 19:35:19

I'll implement this carefully. Let me set up tasks and read the exact current state of every file I'll touch.

*Time to respond: 31s*

## AI - 21 Sep 2026, 19:35:54

Now implementing. Starting with the model, repository, and cache:

*Time to respond: 25s*

## AI - 21 Sep 2026, 19:36:55

Now the seeder script, requirements, and `.gitignore`:

*Time to respond: 13s*

## AI - 21 Sep 2026, 19:37:09

Let me verify there are no leftover references and everything is wired correctly.

*Time to respond: 10s*

## AI - 21 Sep 2026, 19:37:32

Let me verify the seeder's logic end-to-end with a dry run (no credentials needed):

*Time to respond: 10s*

## AI - 21 Sep 2026, 19:37:48

Done. The catalog is wired in and the seeder is verified (dry run: 8,081 records → **7,953 robot parts**; 128 merch excluded — matches the scraper's `nonRobot` count).

## Code changes
- **`Models.kt`** — added `ANDYMARK` to `Vendor`; added `tags` + `url` to `Part`.
- **`PartsRepository.kt`** (new) — reads `catalogMeta/current`; if the version changed, pages through `parts` and caches to `filesDir/parts_cache.json`; falls back to the bundled asset when offline/unseeded.
- **`CatalogViewModel.kt`** — parts now come from `PartsRepository`; added `partsLoading`.
- **`Components.kt`** — added `Part.matchesQuery` and a searchable `PartPickerField` dialog (capped at 50 results).
- **`NewLoanScreen.kt`** — parts dropdown replaced with the searchable picker.
- **`PartListScreen.kt`** — search bar (name/SKU/tag) + loading bar + tags on each card.
- **`.gitignore`** — service-account key patterns.
- **`tools/catalog/seed_firestore.py`** + **`requirements.txt`** (new).

## You need to do three things

**1. Generate a service account key**
Firebase console → gear → **Project settings → Service accounts → Generate new private key**. Save it as `tools/catalog/service-account.json` (the `.gitignore` now excludes `*service-account*.json`).

**2. Seed Firestore**
```
pip install -r tools/catalog/requirements.txt
python tools/catalog/seed_firestore.py --key tools/catalog/service-account.json
```
(Add `--prune` to delete parts removed from a re-scrape; add `--dry-run` to preview.)

**3. Update Firestore rules** — paste this (adds `parts` + `catalogMeta` read-only for verified users):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function signedIn() { return request.auth != null; }
    function verified() { return request.auth.token.email_verified == true; }
    function userData() { return get(/databases/$(database)/documents/users/$(request.auth.uid)).data; }
    function userTeam() { return userData().teamId; }
    function userRole() { return userData().role; }
    function isAuthority() { return userRole() == 'COACH' || userRole() == 'CAPTAIN'; }

    match /users/{uid} {
      allow read: if signedIn() && verified() && request.auth.uid == uid;
      allow create: if signedIn() && verified()
        && request.auth.uid == uid
        && request.resource.data.teamId is string
        && request.resource.data.role in ['COACH', 'CAPTAIN', 'MEMBER']
        && request.resource.data.inviteCode == get(/databases/$(database)/documents/teamInvites/$(request.resource.data.teamId)).data.inviteCode;
      allow update: if signedIn() && verified()
        && request.auth.uid == uid
        && request.resource.data.teamId == resource.data.teamId
        && request.resource.data.role == resource.data.role;
      allow delete: if false;
    }

    match /teamInvites/{teamId} {
      allow read, write: if false;
    }

    match /loans/{loan} {
      allow read: if signedIn() && verified() && resource.data.teamIds.hasAny([userTeam()]);
      allow create: if signedIn() && verified() && request.resource.data.teamIds.hasAny([userTeam()]);
      allow update: if signedIn() && verified()
        && isAuthority()
        && resource.data.teamIds.hasAny([userTeam()])
        && request.resource.data.teamIds == resource.data.teamIds
        && request.resource.data.loanerTeamNumber == resource.data.loanerTeamNumber
        && request.resource.data.borrowerTeamNumber == resource.data.borrowerTeamNumber;
      allow delete: if false;
    }

    match /parts/{partId} {
      allow read: if signedIn() && verified();
      allow write: if false;
    }

    match /catalogMeta/{docId} {
      allow read: if signedIn() && verified();
      allow write: if false;
    }
  }
}
```

Then rebuild the app. On first launch it detects the new catalog version, syncs (~7,950 reads, once), caches locally, and search works offline afterward. Before seeding, the app keeps using the small bundled fallback — so it still runs.

One note: I couldn't compile the Kotlin here (no Android SDK on this machine), so please build in Android Studio. If anything flags, paste it and I'll fix it immediately.

*Time to respond: 6s*

## user - 21 Sep 2026, 19:59:27

Lets create the per-team part aliases

## AI - 21 Sep 2026, 19:59:27

Now let me implement per-team aliases. Setting up tasks:

*Time to respond: 37s*

## AI - 21 Sep 2026, 20:00:40

Per-team aliases are implemented.

## Code changes
- **`AliasesRepository.kt`** (new) — team-scoped alias storage in a `partAliases` collection. Docs: `{ teamId, partId, aliases[], updatedAt }`, keyed `{teamId}_{partId}`. `loadAliases(teamId)` returns a `partId → aliases` map; `saveAliases` upserts (empty list deletes).
- **`CatalogViewModel.kt`** — loads aliases when the team is set, merges them onto the parts (so every screen sees each part's team aliases), and exposes `setAliases(partId, aliases)` with an optimistic local update.
- **`Components.kt`** — `matchesQuery` now also matches **aliases**, so typing an alias in the picker or the catalog search finds the part.
- **`AliasEditor.kt`** (new) — a dialog to add/remove a part's aliases for the team.
- **`PartListScreen.kt`** — each part card has an **edit icon** that opens the alias editor; the search hint now mentions aliases.
- **`LoanListScreen.kt`** — sets the current team on `CatalogViewModel` (alongside the loans one), which triggers alias loading.

## How it works in the app
Open **Parts** → tap the pencil on any part → add aliases like "Motor", "YJ" → Save. Those aliases are saved to Firestore for your team only, show under the part, and are searchable everywhere (including the "Log a share" part picker).

## One thing to add: Firestore rules

Append this block inside your `match /databases/{database}/documents` (alongside `parts`/`catalogMeta`):

```
    match /partAliases/{aliasId} {
      allow read: if signedIn() && verified() && resource.data.teamId == userTeam();
      allow create: if signedIn() && verified() && request.resource.data.teamId == userTeam();
      allow update: if signedIn() && verified()
        && resource.data.teamId == userTeam()
        && request.resource.data.teamId == userTeam();
      allow delete: if signedIn() && verified() && resource.data.teamId == userTeam();
    }
```

So teams can only read/write their own aliases.

Rebuild and test: log in, open Parts, alias a part, then search that alias in both the catalog and the new-loan part picker. (No new index is needed — it's a single-field equality query.)

*Time to respond: 5s*

## user - 21 Sep 2026, 20:04:36

In the search menu when selecting a part to be loaned, there should be a way to filter between goBilda, Rev, etc. Add that.

## AI - 21 Sep 2026, 20:04:36

I'll add vendor filter chips to the part picker dialog in `Components.kt`.

*Time to respond: 15s*

## AI - 21 Sep 2026, 20:04:52

Let me verify the result:

*Time to respond: 3s*

## AI - 21 Sep 2026, 20:04:56

The vendor filter is added. Let me confirm the imports landed correctly:

*Time to respond: 5s*

## AI - 21 Sep 2026, 20:05:02

Done. The part picker dialog now has a horizontally scrollable row of vendor filter chips — **All**, then AndyMark, goBILDA, REV, Studica, and Custom (driven by `Vendor.entries`, so it stays in sync automatically).

Results are now filtered by **vendor + search query** together, still capped at 50 rows. Rebuild and open "Log a share" → Select part to see the chips.

*Time to respond: 4s*

## user - 21 Sep 2026, 20:09:34

Remove the "custom" chip. At the bottom, say "+32" or however many more items there are that aren't being displayed, so that people don't think that the catalogue is missing items.

## AI - 21 Sep 2026, 20:09:34

Two changes to the picker: drop the Custom chip and add a "+N more" footer.

*Time to respond: 18s*

## AI - 21 Sep 2026, 20:09:57

Both changes are in place:

1. **Custom chip removed** — the vendor row now iterates `Vendor.entries.filter { it != Vendor.CUSTOM }`, so it shows All, AndyMark, goBILDA, REV, Studica.
2. **"+N more" footer** — the full match list is computed, only the first 50 are shown, and when there are more, a `+N more (refine your search)` line appears below the list (always visible, outside the scroll area).

The list's max height was trimmed from 360dp to 320dp to make room for the footer. Rebuild to see it.

*Time to respond: 4s*

## user - 21 Sep 2026, 20:14:37

In the "Parts Catalogue" menu, remove the "Custom" chip for filtering

## AI - 21 Sep 2026, 20:14:41

Done. The Parts Catalog filter row now iterates `Vendor.entries.filter { it != Vendor.CUSTOM }`, so it shows **All, AndyMark, goBILDA, REV, Studica** — matching the part picker.

*Time to respond: 3s*

## user - 21 Sep 2026, 20:17:43

In the "About" menu, add some text that says that this app is not affiliated with goBilda, Rev, etc

## AI - 21 Sep 2026, 20:17:57

Added a **Disclaimer** section to the About screen:

> FTC Equipment Tracker is an independent project built by FTC Team #25310 and is not affiliated with, endorsed by, or sponsored by goBILDA, REV Robotics, AndyMark, Studica, FIRST, or any other manufacturer or organization mentioned in the app. Product names, brands, and trademarks belong to their respective owners.

It sits below the Data source section, separated by a divider. Rebuild to see it.

*Time to respond: 4s*
