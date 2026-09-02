# Search Precision Time Services

Filter-property reference and presentation rules specific to the
`search_time_services` tool. Read this whenever the user's request is about
**precision time services** (NTP/PTP) — before constructing the filter and before
rendering the results table.

The shared rules in `SKILL.md` (state-filter defaults, operator syntax, page size,
pagination disclosure, the "Before you respond" checklist) still apply on top of
everything below — this file only covers what's specific to precision time
services.

## Precision time service-specific filter properties

| User asks for... | Filter property | Example |
|---|---|---|
| Service UUID | `uuid` | `uuid = '...'` |
| Name | `name` | `name LIKE '%ept%'` |
| State | `state` | `state = 'PROVISIONED'` |
| Type | `type` | `type = 'PTP'` |
| Package | `package/code` | `package/code = 'PTP_ENTERPRISE'` |
| Metro / location | `location/metroCode` | `location/metroCode = 'SV'` |
| Project ID | `project/projectId` | `project/projectId = '111111000111111'` |
| Created | `changeLog/createdDateTime` | `changeLog/createdDateTime BETWEEN [...]` |

**Type values:** `NTP`, `PTP`.

**Package code values:** `NTP_STANDARD`, `NTP_ENTERPRISE`, `PTP_STANDARD`,
`PTP_ENTERPRISE`. The package code must match the service type (e.g.
`NTP_STANDARD` never pairs with a `PTP` service).

**State values:** the default state filter (see `SKILL.md` Step 1) excludes
`DEPROVISIONED` unless the user explicitly asks for terminal/inactive time
services. Time services typically transition through `PROVISIONING` →
`PROVISIONED`.

**Critical:** always apply these as server-side filters — never retrieve all
time services and filter client-side.

## Presenting precision time service results

| Field | Source |
|---|---|
| Name | `name` |
| UUID | `uuid` |
| State | `state` |
| Package | `package.code` (NTP or PTP type) |

**When the user filtered by a specific property, include that property as a
column in the results table.** For example:
- Filtered by type → add a **Type** column (`type`) alongside Package, since
  Package alone (e.g. `PTP_ENTERPRISE`) already implies type but the user's
  filtered-on field should still be visible directly.
- Filtered by metro → add **Metro** column (`location.metroCode`).
- Filtered by project ID → add **Project ID** column (`project.projectId`).

**Deduplication:** each time service appears exactly once. If pagination returns
the same UUID more than once, show it only once.

**Verify before rendering:** for any exact-match filter (type, package, state,
etc.), check that every row's actual field value in the API response satisfies the
filter condition before adding it to the table.

**After the table, a summary sentence is mandatory** — restating both the exact
filter(s) applied and the total count, e.g.:
- *"Found 2 precision time services of type PTP with package PTP_ENTERPRISE
  (total: 2)."*

**If `pagination.total` is greater than the number of rows shown, you must state
this explicitly** — never present a partial list without saying so: *"Showing 20 of
24 — ask me to show more or narrow your filter."*

## Zero precision time services found

Once the tool confirms `pagination.total = 0`:
- **Do NOT show an empty table.**
- State clearly: *"No precision time services found matching your criteria (total:
  0)."*
- Explain what filter was applied: *"Searched for PTP services in metro SV."*
- Suggest next steps: check the metro code, try the other service type, confirm
  the project ID, or try searching without the package filter.
