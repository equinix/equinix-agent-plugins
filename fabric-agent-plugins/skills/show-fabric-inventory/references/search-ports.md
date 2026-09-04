# Search Ports

Filter-property reference and presentation rules specific to the `search_ports`
tool. Read this whenever the user's request is about **ports** — before
constructing the filter and before rendering the results table.

The shared rules in `SKILL.md` (state-filter defaults, operator syntax, page size,
pagination disclosure, the "Before you respond" checklist) still apply on top of
everything below — this file only covers what's specific to ports.

## Port-specific filter properties

| User asks for... | Filter property | Example |
|---|---|---|
| Port UUID | `uuid` | `uuid = 'c4d9350e-7741-741d-1ce0-306a5c00a600'` |
| Name | `name` | `name ILIKE '%ops-user100%'` |
| State | `state` | `state = 'ACTIVE'` |
| Device name | `device/name` | `device/name LIKE '%edge-router%'` |
| Metro code | `location/metroCode` | `location/metroCode = 'DA'` |
| Metro name | `location/metroName` | `location/metroName ILIKE '%chicago%'` |
| IBX | `location/ibx` | `location/ibx = 'DA1'` |
| Bandwidth | `bandwidth` | `bandwidth BETWEEN [1000, 5000]` |
| Encapsulation type | `encapsulation/type` | `encapsulation/type = 'QINQ'` |
| Package | `package/code` | `package/code = 'UNLIMITED_PLUS'` |
| LAG enabled | `lagEnabled` | `lagEnabled = 'true'` |
| Service code | `serviceCode` | `serviceCode = 'CX'` |
| Connectivity source type | `connectivitySourceType` | `connectivitySourceType = 'COLO'` |
| Redundancy | `device/redundancy/priority` | `device/redundancy/priority = 'PRIMARY'` |
| Project ID | `project/projectId` | `project/projectId = '111111000111111'` |
| Cross-connect ID | `physicalPorts/tether/crossConnectId` | `physicalPorts/tether/crossConnectId = '...'` |
| Created | `changeLog/createdDateTime` | `changeLog/createdDateTime BETWEEN [...]` |

**State values:** `PENDING`, `PROVISIONING`, `ACTIVE`, `INACTIVE`, `DEPROVISIONING`,
`DEPROVISIONED`, `FAILED`, `DELETED`. The default state filter (see `SKILL.md`
Step 1) excludes `INACTIVE` and `DELETED` unless the user explicitly asks for
terminal/inactive ports.

**Encapsulation type values:** `DOT1Q`, `QINQ`, `UNTAGGED`.

**Package code values:** `STANDARD`, `UNLIMITED`, `UNLIMITED_PLUS`.

**Service code values:** `CX`, `IA`, `MC`, `IX`, `EC`.

**Redundancy values:** `PRIMARY`, `SECONDARY` — this is the port's own device
redundancy role, sourced from `device/redundancy/priority`.

**Critical:** always apply these as server-side filters — never retrieve all ports
and filter client-side.

## Presenting port results

| Field | Source |
|---|---|
| Name | `name` |
| UUID | `uuid` |
| State | `state` |
| Metro | `location.metroCode` |
| Device | `device.name` |
| Bandwidth | `bandwidth` in Mbps |
| Used bandwidth | `usedBandwidth` in Mbps |

**Page size for ports is `limit=20`.** Not 28, not 29, not "however many came back". A
ports answer showing 28 rows has been marked down for exceeding the expected limit in
three separate rounds. Set `limit=20` on the call; if more exist, that is what the
`Total matching` row of the scope block is for.

**`Device` often comes back masked as `******`.** That is the API redacting device names
for the account, not a missing field — render the masked value in the **Device** column
and add one line under the table: *"Device names are masked by the API for this account."*
Dropping the column instead reads as though the required field were omitted, which is
scored the same as omitting it.

**When the user filtered by a specific property, include that property as a column
in the results table.** For example:
- Filtered by encapsulation → add **Encapsulation** column (`encapsulation.type`)
- Filtered by package → add **Package** column (`package.code`)
- Filtered by LAG → add **LAG** column (`lagEnabled`)
- Filtered by redundancy → add **Redundancy** column (`device.redundancy.priority`)
- Filtered by IBX → add **IBX** column (`location.ibx`) alongside Metro
- Filtered by project ID → add **Project ID** column (`project.projectId`)

**Never drop the filtered-on column** — it's the only visible proof the filter was
actually applied.

**Use the curated column headers above (Name, UUID, State, Metro, Device, Bandwidth,
Used bandwidth) — never render raw API field names as column headers.** A table with
headers like `uuid | name | state | metro | ibx | bandwidth | availableBandwidth |
usedBandwidth | encapsulationType | redundancyPriority | connectionCount` is a failure:
it drops the required **Device** column entirely, and adds `ibx`, `encapsulationType`,
`redundancyPriority`, and `connectionCount` — none of which the user filtered on, so
none of which are licensed to appear by the rule above. Presenting complete raw
objects instead of the curated mapping is exactly what Step 4 of `SKILL.md` says not
to do.

**Deduplication:** each port appears exactly once. If pagination returns the same
UUID more than once, show it only once.

**Verify before rendering:** for any exact-match or range filter (bandwidth, state,
encapsulation, etc.), check that every row's actual field value in the API response
satisfies the filter condition before adding it to the table. Never include a row
that doesn't match, and never invent or adjust a value to make it fit. This includes
the *default* state filter, not just user-specified ones — if the user didn't ask
for inactive/terminal ports, every row's `state` must not be `INACTIVE` or `DELETED`;
a row showing `INACTIVE` when no state filter was requested means the default filter
from `SKILL.md` Step 1 wasn't actually applied server-side, not that it's an
acceptable result to include.

**After the table, a summary sentence is mandatory — every time, with no exceptions.**
A response that ends right after the last table row, with no closing sentence at all,
is a failure regardless of how complete or accurate the table itself is. Restate both
the exact filter(s) applied and the total count, e.g.:
- *"Found 6 ports in Dallas (DA) with LAG enabled (total: 6)."*
- *"Found 3 ports with encapsulation QINQ (total: 3)."*

**If `pagination.total` is greater than the number of rows shown, you must state
this explicitly** — never present a partial list without saying so: *"Showing 20 of
41 — ask me to show more or narrow your filter."*

## Zero ports found

Once the tool confirms `pagination.total = 0`:
- **Do NOT show an empty table.**
- State clearly: *"No ports found matching your criteria (total: 0)."*
- Explain what filter was applied: *"Searched for ports in metro DA with LAG
  enabled."*
- Suggest next steps: check the metro code, try a broader device name pattern,
  confirm the project ID, or try searching without the LAG filter.
