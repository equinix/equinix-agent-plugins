# Search Connections

Filter-property reference and presentation rules specific to the `search_connections`
tool. Read this whenever the user's request is about **connections** — before
constructing the filter and before rendering the results table.

The shared rules in `SKILL.md` (state-filter defaults, operator syntax, page size,
pagination disclosure, the "Before you respond" checklist) still apply on top of
everything below — this file only covers what's specific to connections.

## Connection-specific filter properties

| User asks for... | Filter property | Example |
|---|---|---|
| Redundancy priority | `redundancy/priority` | `redundancy/priority = 'PRIMARY'` |
| Equinix status | `operation/equinixStatus` | `operation/equinixStatus = 'PENDING_BGP_PEERING'` |
| Seller region (Z-side) | `zSide/accessPoint/sellerRegion` | `zSide/accessPoint/sellerRegion = 'AMER'` |
| Port name (A-side) | `aSide/accessPoint/port/name` | `aSide/accessPoint/port/name ILIKE '%myport%'` |
| Port name (Z-side) | `zSide/accessPoint/port/name` | `zSide/accessPoint/port/name ILIKE '%myport%'` |
| VLAN CTag range | `aSide/accessPoint/linkProtocol/vlanCTag` | `aSide/accessPoint/linkProtocol/vlanCTag BETWEEN [100, 200]` |
| Access point type | `aSide/accessPoint/type` or `zSide/accessPoint/type` | `aSide/accessPoint/type = 'COLO'` |
| Created by (user) | `changeLog/createdBy` | `changeLog/createdBy = 'eqxnfvuser1'` |
| Project ID | `project/projectId` | `project/projectId = '111111000111111'` |
| Metro / location | `aSide/accessPoint/location/metroCode` OR `zSide/accessPoint/location/metroCode` | `aSide/accessPoint/location/metroCode = 'SV' OR zSide/accessPoint/location/metroCode = 'SV'` |
| "Recent" (no explicit time range given) | sort by `changeLog/createdDateTime` descending, small `limit` (e.g. 10) | combine with any other filter given in the same request |

**A single metro filter must check both sides.** A connection whose Z-side (not A-side) is
in the requested metro is still a match — filtering only `aSide` silently drops half the
real results. Always express a metro filter as an OR across both sides unless the user
explicitly says "A-side" or "Z-side."

**Combining a metro filter with another filter (e.g. type):** wrap the metro OR-expression
in parentheses and AND it with the other condition — never drop the OR just because
another filter is also present:
`type = 'EVPL_VC' AND (aSide/accessPoint/location/metroCode = 'SV' OR
zSide/accessPoint/location/metroCode = 'SV')`
A query that ANDs `type` with only one side's metro (or omits the metro OR entirely) will
silently return connections that don't actually match the requested metro on either side.

**Don't confuse `redundancy.priority` with A-side/Z-side labels.** `redundancy.priority`
is a top-level field on the connection object itself — it says whether *this connection*
is the primary or secondary member of a redundant pair. It has nothing to do with any
PRIMARY/SECONDARY text that shows up inside `aSide`/`zSide` access point details (e.g.
embedded in a port name like `...-PRI-NK-506` or `...-SEC-NK-157`, or an access point
role field). Those per-side labels describe the port/access-point's own role and will
vary row to row even after you've correctly filtered on `redundancy.priority` — seeing
"SECONDARY" inside an A-side or Z-side cell is expected and is not a sign the filter
failed. Filter and verify strictly against the top-level `redundancy.priority` field;
never use A-side/Z-side text as a stand-in for it.

**Critical:** always apply these as server-side filters — never retrieve all connections
and filter client-side.

## Presenting connections results

| Field | Source |
|---|---|
| Name | `name` |
| UUID | `uuid` |
| Status | `operation.equinixStatus` |
| Bandwidth | `bandwidth` in Mbps |
| Route | `aSide.accessPoint.location.metroCode` → the Z-side's **name** (see below) |
| Type | `type` |

**The Z-side half of Route is a name, never an access-point type.** Where the name lives
depends on what the Z-side is:

| Z-side `accessPoint.type` | Use this for the Route's right-hand side |
|---|---|
| `COLO` | `zSide.accessPoint.port.name` (e.g. `201257-DC11-CX-PRI-01`) |
| `SP` | `zSide.accessPoint.profile.name` (e.g. `AWS Direct Connect`) |
| `NETWORK` | `zSide.accessPoint.network.name` (e.g. `EVPLAN-GLOBAL-02`) |
| `CLOUD_ROUTER` | `zSide.accessPoint.router.name` |
| `VD` | `zSide.accessPoint.virtualDevice.name` |

**Never print the type literal itself.** A Route column reading `DC → COLO`, `SV → COLO`,
`DA → COLO` on row after row is this bug: `COLO` is the *kind* of endpoint, not the
endpoint, so the column stops answering "where does this connection go?" — which is the
only reason it exists. If the relevant name field is genuinely absent, fall back to the
Z-side metro code (`DC → SV`), which is at least a location. If neither is present, use `—`.

**Filter paths use `changeLog/...`; the response body returns the key lowercased as
`changelog`.** Read `changelog.createdDateTime`, filter on `changeLog/createdDateTime`.

`Created` (`changelog.createdDateTime`) and `Created by` (`changelog.createdBy`) are
**not** default columns — add them only when the user filtered or sorted on them (e.g. a
"recent connections" or `createdBy` request). Adding them unprompted pushes the table
wider without making the applied filter any more verifiable.

**Use these six curated headers — never render raw API field names, and never flatten the
`aSide`/`zSide` objects into columns.** This is the worst observed failure for connections:

    | uuid | name | type | state | equinixStatus | providerStatus | bandwidth | direction |
    isRemote | geoScope | redundancyGroup | redundancyPriority | accountNumber | orgId |
    createdDateTime | updatedDateTime | aSideType | aSideMetroCode | aSideMetroName |
    aSidePortUuid | aSidePortName | aSideRouterUuid | aSideRouterName | aSideVlanTag |
    aSideVlanSTag | aSideVlanCTag | zSideType | zSideMetroCode | … | projectId |

41 columns, most of them empty, `Status` and `Route` both missing as such — the reader
cannot answer "where does this connection go?" from it, which is the one thing the **Route**
column exists to convey. Collapse the A-side/Z-side detail into the single **Route** column
(`aSide.accessPoint.location.metroCode` → `zSide.accessPoint.name`); surface an individual
A-side or Z-side field as its own column only when the user filtered on it.

**When the user filtered by a specific property, include that property as a column in
the results table — this is required, not optional.** For example:
- Filtered by redundancy → add a **Redundancy Priority** column sourced from the
  top-level `redundancy.priority` field specifically. A **Redundancy Group** column
  (`redundancy.group`) is a different field and does not satisfy this requirement — it
  does not show PRIMARY/SECONDARY at all, so swapping it in instead of Redundancy
  Priority leaves the filter unverifiable to the user. If you're also showing
  A-side/Z-side detail columns, that's fine, but Redundancy Priority must be present
  alongside them, not replaced by them.
- Filtered by equinixStatus → add **Equinix Status** column (`operation.equinixStatus`)
- Filtered by sellerRegion → add **Seller Region** column (`zSide.accessPoint.sellerRegion`)
- Filtered by createdBy → add **Created by** column (`changeLog.createdBy`) and
  **Created** date column (`changeLog.createdDateTime`)
- Filtered by VLAN CTag → add **A-side VLAN CTag** column
  (`aSide.accessPoint.linkProtocol.vlanCTag`)
- Filtered by type → add **Type** column (`type`)
- Filtered by project ID → add **Project ID** column (`project.projectId`)

**Never drop the filtered-on column.** This is the single most important column in the
table — it's the only visible proof the filter was actually applied. If you choose to
add extra columns beyond the defaults (e.g. Direction, Equinix Status, Provider Status,
A-side/Z-side details), the filtered property must still be present alongside them.
Swapping it out for unrelated "additional acceptable" columns is a failure, not a
stylistic choice — verify the filtered column is in your final table before presenting
it.

**Concrete failure to avoid:** for a redundancy-priority filter, a table with columns
UUID, Name, State, Bandwidth, A-side type, A-side metro, Z-side type, Z-side metro,
Project — but no Redundancy Priority column — fails this requirement even if the table
title or summary sentence says "Primary redundancy connections." A-side/Z-side detail
columns are not a substitute; the reader needs a Redundancy Priority column to verify
each row per-row, not just a title asserting it.

**Deduplication:** each connection appears exactly once. If pagination returns the same
UUID more than once, show it only once.

**Verify before rendering:** for any range or exact-match filter (VLAN CTag, bandwidth,
createdBy, state, project ID, etc.), check that every row's actual field value in the
API response satisfies the filter condition before adding it to the table. Never
include a row that doesn't match, and never invent or adjust a value to make it fit —
if the API returned something unexpected, report it as-is and flag the discrepancy
rather than silently dropping or fabricating.

**Never merge results from calls with different filter scopes into one table.** If
answering the request took more than one `search_connections` call (e.g. a follow-up
call, a broader lookup, a retry with adjusted parameters), each row you render must be
checked against the *current* request's filter — not just concatenated from whichever
calls happened to run. A table where most rows show the requested `project.projectId`
but a handful show a different project ID is proof that a row from a differently-scoped
call leaked into the table unchecked. If you're not certain a row matches the active
filter, re-verify it against the raw API response before including it, or drop it.

**After the table, a summary sentence is mandatory** — restating both the exact
filter(s) applied and the total count, e.g.:
- *"Found 5 connections created by eqxnfvuser1 (total: 5)."*
- *"Found 3 connections with A-side VLAN CTag between 100 and 200 (total: 3)."*
- *"Found 12 connections with redundancy priority PRIMARY (total: 12)."*

**If `pagination.total` is greater than the number of rows shown, you must state this
explicitly** — never present a partial list without saying so: *"Showing 20 of 47 — ask
me to show more or narrow your filter."* This applies every time totals and shown-row
counts differ, not only when convenient. See `SKILL.md`'s pagination-disclosure rule —
the total must never be stated in isolation from the shown count.

## Zero connections found

Once the tool confirms `pagination.total = 0`:
- **Do NOT show an empty table.**
- State clearly: *"No connections found matching your criteria (total: 0)."*
- Explain what filter was applied: *"Searched for A-side port name containing 'aws'."*
- Suggest next steps: check the port name spelling, try a broader pattern (e.g. `%a%`),
  confirm the port exists, or try searching without the port name filter.
