# FTC parts loan tracking app planning

## user - 05 Sep 2026, 16:06:10

can you start with the FTC Events api integration

## AI - 05 Sep 2026, 16:06:10

I'll start the FTC Events API integration. First, let me check the current API documentation to get the endpoint details right.

*Time to respond: 1m 29s*

## AI - 05 Sep 2026, 16:11:12

FTC Events API integration is built. Here's what I added:

**New files**
- `data/model/FtcTeam.kt` — `FtcTeamsResponse` + `FtcTeam` models (with `toTeam()` mapping to the app's `Team`)
- `data/FtcApiConfig.kt` — base URL, credentials, season, country
- `data/FtcEventsApi.kt` — OkHttp client with Basic Auth + pagination
- Updated `data/CatalogRepository.kt` — teams now load with this order: fresh cache (≤24h) → FTC API → stale cache → bundled `teams.csv`

**Dependency**: added OkHttp 4.12.0.

## What you need to do to activate it

1. **Register for API access** at `https://ftc-events.firstinspires.org/services/API/register` — you'll get a **username** and an **authorization token** (the token is *not* your login password).

2. **Fill in `data/FtcApiConfig.kt`**:
   - `USERNAME` = your API username
   - `TOKEN` = your API authorization token
   - `SEASON` = the year the season ends (I defaulted to `"2027"` for the 2026-27 season — verify against the docs)
   - `COUNTRY` = `"South Africa"` (leave as-is)

Until you fill in the credentials, the app keeps using the bundled `teams.csv` (graceful fallback), so nothing breaks.

## Things to verify (and flag if wrong)

- **Country filter param**: I used `?country=South Africa`. Confirm the exact query param name/value in the live API docs after registering (it may be `country` with the full name — that's my best understanding).
- **Attribution**: FIRST requires a link back to `https://ftc-events.firstinspires.org/services/API` on apps using their data. I haven't added an About screen yet — want me to add one (it also doubles as the place for your team authorship line)?

Want me to also add a **manual "refresh" button** on the Teams screen (so you can pull fresh data without waiting 24h or restarting), or an About/attribution screen next?

*Time to respond: 11s*

## user - 05 Sep 2026, 16:18:50

go ahead and add an about screen and manual refresh button

## AI - 05 Sep 2026, 16:20:44

Both done.

**About screen** (`AboutScreen.kt`)
- App name + "By FTC Team #25310 Pinelands Stingers"
- A short description
- "Data source" section with a clickable **FTC Events API** link (satisfies FIRST's attribution requirement)
- Accessible via a new **info icon** in the Loans screen top bar

**Manual refresh** (Teams screen)
- Added a **refresh icon** in the Teams screen top bar
- `CatalogViewModel.refreshTeams()` + `CatalogRepository.refreshTeams()` force a fresh fetch from the FTC API (bypassing the 24h cache) and update the cached file
- Shows a progress bar while refreshing and a red error message on failure (e.g. "FTC API credentials are not configured")

New/changed files: `AboutScreen.kt`, `TeamListScreen.kt`, `CatalogViewModel.kt`, `CatalogRepository.kt`, `FtcNavHost.kt`, `LoanListScreen.kt`.

Note: the refresh button will show an error until you fill in your FTC API username/token in `FtcApiConfig.kt` — that's expected, and it confirms the button works.

*Time to respond: 8s*

## user - 05 Sep 2026, 16:33:01

When i refresh, it just says "No teams returned from the FTC API", what is causing this and can you fix it

## AI - 05 Sep 2026, 16:33:01

The "No teams returned" means the API returned a successful (200) response with an empty team list — so the request works but the filter matches nothing. Two likely causes in my code:

1. **Season**: I defaulted `SEASON` to `"2027"` (the 2026-27 season), but kickoff is Sept 12, so that season's roster is still empty. Last season's data is `"2026"`.
2. **URL encoding**: I used `URLEncoder` which turns the space in "South Africa" into `+`, but OkHttp treats `+` as a literal plus in query strings — so the country filter likely matched nothing.

Let me fix both.

*Time to respond: 2m 30s*

## AI - 05 Sep 2026, 16:35:45

Fixed. Two issues resolved:

1. **Season** — was `"2027"` (the 2026-27 season, which is still empty since kickoff is Sept 12). Changed to `"2026"` — last season's complete roster.
2. **URL encoding** — `URLEncoder` turned the space in "South Africa" into `+`, but OkHttp treats `+` as a literal plus in query strings. Now uses `HttpUrl.Builder`, which correctly encodes it as `%20`.

The request now goes to:
`https://ftc-api.firstinspires.org/v2.0/2026/teams?country=South%20Africa&page=1`

I also made the error message include the season + country so future failures are easier to diagnose.

**Rebuild and refresh** — you should now see last season's South African teams.

Two notes:
- **After kickoff** (Sept 12) and once teams register for the new season, change `SEASON` back to `"2027"` in `FtcApiConfig.kt` to pull the current season's roster.
- **Security:** your API token is now sitting in `FtcApiConfig.kt`. It's fine for now (the API is non-commercial and low-sensitivity), but if you push to a public GitHub repo, anyone can extract it. Consider adding `FtcApiConfig.kt` to `.gitignore` (or moving credentials into a local, non-committed file). Want me to set that up?

If it *still* says no teams returned after this, the `country` parameter name is likely wrong in the API — tell me and I'll switch it to client-side filtering (fetch all teams, filter by country locally), which avoids the filter param entirely.

*Time to respond: 18s*

## user - 05 Sep 2026, 16:47:37

when i reloaded i got this error: "Unexpected JSON token at offset 84371: Expected string literal but 'null' literal was found at path: $.teams[237].nameFull
 
Use 'coercelnputValues = true' in 'Json {} builder to coerce nulls if property has a default value.
JSON
input: .....eamNumber":"24071","nameFull":null,"nameShort":"BERAPI
T LION....."

## AI - 05 Sep 2026, 16:47:37

The API returns `null` for `nameFull` on some teams, but my model declared it non-nullable. I'll make the string fields nullable and handle nulls. I'll also add client-side country filtering (the team names in your error, e.g. "BERAPI T LION", look non-South-African, so the `country` server filter may not be applying — filtering client-side guarantees you only get SA teams).

*Time to respond: 3m 53s*

## AI - 05 Sep 2026, 16:52:14

Fixed the crash and made the country filter robust.

**What was wrong**
- The API returns `null` for `nameFull` on some teams, but my model declared it non-nullable — so parsing threw. I made `nameFull`, `nameShort`, `city`, and `country` nullable (handling `null` → empty in the mapping), and added `coerceInputValues = true` as a safety net for any other unexpected nulls.
- Your error also revealed a second problem: team **24071 "BERAPI T LION"** looks like a non-South-African team in the results, which means the server-side `?country=` filter isn't actually filtering. So I added **client-side filtering** — the code now keeps only teams whose `country == "South Africa"`.

**Result**: the app should now fetch successfully and show only South African teams.

Two things to watch:
- **Speed**: if the server isn't honoring the country filter, the app downloads *all* teams before filtering, so the first refresh may take a while. It's cached for 24h after that, so subsequent loads are fast.
- If the SA list still looks wrong/empty, tell me the exact team count shown and I'll dig into the correct filter parameter name for the API.

*Time to respond: 26s*
