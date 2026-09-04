# Search Cloud Routers

Filter-property reference and presentation rules specific to the `search_routers`
tool. Read this whenever the user's request is about **cloud routers (FCRs)** —
before constructing the filter and before rendering the results table.

The shared rules in `SKILL.md` (state-filter defaults, operator syntax, page size,
pagination disclosure, the "Before you respond" checklist) still apply on top of
everything below — this file only covers what's specific to cloud routers.

## Cloud router-specific filter properties

| User asks for... | Filter property | Example |
|---|---|---|
| Router UUID | `uuid` | `uuid = 'c4d9350e-7741-741d-1ce0-306a5c00a600'` |
| Name | `name` | `name LIKE '%prod%'` |
| State | `state` | `state = 'PROVISIONED'` |
| Package | `package/code` | `package/code = 'ADVANCED'` |
| Metro / location | `location/metroCode` | `location/metroCode = 'SV'` |
| Project ID | `project/projectId` | `project/projectId = '111111000111111'` |
| Created | `changeLog/createdDateTime` | `changeLog/createdDateTime BETWEEN [...]` |

**Package codes** (lowest to highest capacity tier): `LAB`, `BASIC`, `STANDARD`,
`ADVANCED`. Each tier caps max connections, max IPv4/IPv6 routes, and max VC
bandwidth differently — if the user is comparing tiers rather than filtering
existing routers, point them to `explore-fabric-connectivity-options` instead of
guessing capacity numbers here.

**State values:** the full enum is `PROVISIONING`, `PROVISIONED`, `REPROVISIONING`,
`DEPROVISIONING`, `DEPROVISIONED`, `NOT_PROVISIONED`, `NOT_DEPROVISIONED`. The default state
filter (see `SKILL.md` Step 1) excludes `DEPROVISIONED` and `NOT_PROVISIONED` unless the user
explicitly asks for terminal routers. A healthy router reads `PROVISIONED`.

**Critical:** always apply these as server-side filters — never retrieve all
routers and filter client-side.

## Presenting cloud router results

| Field | Source |
|---|---|
| Name | `name` |
| UUID | `uuid` |
| State | `state` |
| Package | `package.code` |
| Metro | `location.metroCode` |
| Connections | `connectionsCount` |
| Created | `changeLog.createdDateTime` |

**When the user filtered by a specific property, include that property as a column
in the results table.** For example:
- Filtered by package → **Package** column (`package.code`) is already a default —
  confirm it's present and not dropped.
- Filtered by project ID → add **Project ID** column (`project.projectId`).
- Filtered by created date range → add **Created** column (`changeLog.createdDateTime`),
  already a default — confirm it's present.

**Deduplication:** each router appears exactly once. If pagination returns the same
UUID more than once, show it only once.

**Verify before rendering:** for any exact-match or range filter (package, state,
metro, created date, etc.), check that every row's actual field value in the API
response satisfies the filter condition before adding it to the table. Never
include a row that doesn't match, and never invent or adjust a value to make it
fit.

**After the table, a summary sentence is mandatory** — restating both the exact
filter(s) applied and the total count, e.g.:
- *"Found 4 cloud routers in SV with package ADVANCED (total: 4)."*
- *"Found 12 cloud routers in project 111111000111111 (total: 12)."*

**If `pagination.total` is greater than the number of rows shown, you must state
this explicitly** — never present a partial list without saying so: *"Showing 20 of
33 — ask me to show more or narrow your filter."*

## Zero cloud routers found

Once the tool confirms `pagination.total = 0`:
- **Do NOT show an empty table.**
- State clearly: *"No cloud routers found matching your criteria (total: 0)."*
- Explain what filter was applied: *"Searched for routers in metro SV with package
  ADVANCED."*
- Suggest next steps: check the metro code, try a different package, confirm the
  project ID, or try searching without the package filter.
