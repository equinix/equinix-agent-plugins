# Search Route Aggregations

Filter-property reference and presentation rules specific to the
`search_route_aggregations` tool. Read this whenever the user's request is about
**route aggregations** — before constructing the filter and before rendering the
results table.

The shared rules in `SKILL.md` (state-filter defaults, operator syntax, page size,
pagination disclosure, the "Before you respond" checklist) still apply on top of
everything below — this file only covers what's specific to route aggregations.

## Route aggregation-specific filter properties

| User asks for... | Filter property | Example |
|---|---|---|
| Route aggregation UUID | `uuid` | `uuid = '...'` |
| Name | `name` | `name LIKE '%summary%'` |
| State | `state` | `state = 'PROVISIONED'` |
| Type | `type` | `type = 'BGP_IPv4_PREFIX_AGGREGATION'` |
| Project ID | `project/projectId` | `project/projectId = '111111000111111'` |
| Last updated | `changeLog/updatedDateTime` | `changeLog/updatedDateTime BETWEEN [...]` |

**Type values:** `BGP_IPv4_PREFIX_AGGREGATION`, `BGP_IPv6_PREFIX_AGGREGATION`.

**State values:** the default state filter (see `SKILL.md` Step 1) excludes
`DEPROVISIONED` unless the user explicitly asks for terminal/inactive route
aggregations. Route aggregations typically transition through `PROVISIONING` →
`PROVISIONED`.

**Critical:** always apply these as server-side filters — never retrieve all
route aggregations and filter client-side.

## Presenting route aggregation results

| Field | Source |
|---|---|
| Name | `name` |
| UUID | `uuid` |
| State | `state` |
| Type | `type` (IPv4 or IPv6 prefix aggregation) |
| Last updated | `changeLog.updatedDateTime` |

**When the user filtered by a specific property, include that property as a
column in the results table.** For example:
- Filtered by type → **Type** column is already a default — confirm it's present.
- Filtered by project ID → add **Project ID** column (`project.projectId`).

**Deduplication:** each route aggregation appears exactly once. If pagination
returns the same UUID more than once, show it only once.

**Verify before rendering:** for any exact-match filter (type, state, etc.), check
that every row's actual field value in the API response satisfies the filter
condition before adding it to the table.

**After the table, a summary sentence is mandatory** — restating both the exact
filter(s) applied and the total count, e.g.:
- *"Found 2 route aggregations of type BGP_IPv4_PREFIX_AGGREGATION (total: 2)."*

**If `pagination.total` is greater than the number of rows shown, you must state
this explicitly** — never present a partial list without saying so: *"Showing 20 of
22 — ask me to show more or narrow your filter."*

If the user wants to see the rules attached to a specific route aggregation rather
than the aggregations themselves, use `get_all_route_aggregation_rules` instead —
that's a separate lookup, not covered by this file.

## Zero route aggregations found

Once the tool confirms `pagination.total = 0`:
- **Do NOT show an empty table.**
- State clearly: *"No route aggregations found matching your criteria (total: 0)."*
- Explain what filter was applied: *"Searched for route aggregations of type
  BGP_IPv6_PREFIX_AGGREGATION."*
- Suggest next steps: check the project ID, try the other IP-version type, or try
  searching without the type filter.
