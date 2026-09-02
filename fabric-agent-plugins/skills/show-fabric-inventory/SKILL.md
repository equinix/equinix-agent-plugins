---
name: show-fabric-inventory
description: "Searches and lists Fabric inventory — connections, ports, cloud routers, service tokens, route filters, route aggregations, time services. Use when a user asks to show, list, find, search, count, or filter by state, metro, bandwidth, type, or expiry — one type or broadly (e.g. what do I have running, show me everything). Never route these seven to a generic filter tool. For other types (streams, agents, service profiles, metros, metrics, cloud events) call create_filter."
metadata:
  author: Fabric
  version: 2.0
  last_reviewed: 2026-09-01
  filter: '{"or":[{"and":["connections","read"]},{"and":["routers","read"]},{"and":["ports","read"]},{"and":["service_tokens","read"]},{"and":["route_filters","read"]},{"and":["route_aggregations","read"]},{"and":["time_services","read"]}]}'
---

# Show fabric inventory

> **Do these three checks in order. Do not reverse them.**
>
> **Check 1 — Exemption.** Does the request ask for everything ("show me everything", "show
> me all", "full inventory"), or name a project? If **either** is true, skip Check 2 and go
> to Check 3. Do not count dimensions.
>
> **Check 2 — Count the narrowing dimensions.** Run this only when Check 1 found no
> exemption. The dimensions are metro, project ID, resource type, name pattern, or any other
> filter. If the count is zero, **stop**. Ask the user to narrow the request. Call no tool.
>
> **Check 3 — Call the search tool.** Never report results — including "no results found" —
> without a real API response.
>
> Check 1 is a gate, not a dimension you weigh. "Show me everything in my project" satisfies
> it twice over, so it must never get the narrowing question.
>
> **"Resource type" counts either way.** One of the seven types this skill owns is a
> dimension ("show me my ports"), and so is a value of a resource's own `type` field
> ("EVPL_VC connections"). Both narrow the request.

## Step 1 — Determine scope before searching

Before calling any tool, resolve two things:

**A. Default state filter**

Return active resources by default. Include transient states such as PROVISIONING and
DEPROVISIONING. Do not surface terminal states unless the user asks. Apply the `state`
filter server-side, using the values below for each resource type:

| Resource type | Field | Exclude these terminal values |
|---|---|---|
| Connections | `state` | DEPROVISIONED, FAILED, CANCELLED |
| Ports | `state` | INACTIVE, DEPROVISIONED, DELETED |
| Cloud routers | `state` | DEPROVISIONED, NOT_PROVISIONED |
| Route filters | `state` | DEPROVISIONED |
| Route aggregations | `state` | DEPROVISIONED |
| Service tokens | `state` | EXPIRED, INACTIVE, DELETED |
| Precision time services | `state` | DEPROVISIONED |

Use `!=` with a single terminal value, or `IN` with all active values, whichever is fewer
expressions. The active `IN` lists are the exact complement of the exclusions above:
- Connections: `ACTIVE`, `PROVISIONING`, `DEPROVISIONING`, `PENDING`, `DRAFT`
- Ports: `ACTIVE`, `PROVISIONED`, `PROVISIONING`, `REPROVISIONING`, `PENDING`, `PENDING_CROSS_CONNECT`
- Cloud routers, route filters, route aggregations: `PROVISIONED`, `PROVISIONING`, `REPROVISIONING`, `DEPROVISIONING`, `NOT_DEPROVISIONED` (these four have no `ACTIVE` state — healthy reads `PROVISIONED`)
- Service tokens: `ACTIVE`
- Precision time services: `PROVISIONED`, `PROVISIONING`, `REPROVISIONING`, `CONFIGURING`

**If a rendered row holds a state from the exclude column, the filter did not go through.
Fix the query. Do not ship the rows.**

**Connections carry a second lifecycle field. `state` alone does not screen it.** A
connection can read `state = ACTIVE` while `operation.equinixStatus` is `CANCELLED`,
`REJECTED`, `REJECTED_ACK` or `PENDING_DELETE`. Those four are dead. Screen them out with
a filter on `operation/equinixStatus`.

`REJECTED`, `REJECTED_ACK` and `PENDING_DELETE` exist **only** in `equinixStatus`; putting
them in a `state` filter is an invalid query. `CANCELLED` is valid in **both** enums.

`type` + both metro sides + four status exclusions reaches the 8-expression cap in Step 2.
When it does not fit, filter on what does, drop the remaining dead rows, and say so.

**B. Narrowing guard for unscoped queries**

This is Checks 1 and 2 from the rule at the top. Two worked examples, going opposite ways:

| Request | Check 1 exemption? | Dimensions | Action |
|---|---|---|---|
| "what do I have running" | none | 0 | **Ask** the user to narrow it |
| "Show me everything in my project" | "everything" **and** a project | not counted | **Search** all seven types |

Each has scored 0.00 once — the first answered with a data dump, the second with the
narrowing question. Run Check 1 first and both come out right.

---

## Step 2 — Apply search filter guidance

The MCP tool descriptions contain the full list of filterable properties and
supported operators for each tool. Use them directly. This section covers only
what the tool schemas do not tell you.

### Supported Operators

| Operator | Symbol | Description | Example |
|---|---|---|---|
| Equal | `=` | Exact match | `/state = 'ACTIVE'` |
| Not Equal | `!=` | Exclude value | `/state != 'DELETED'` |
| Like | `LIKE` | Pattern match (case-sensitive) | `/name LIKE '%conn%'` |
| ILike | `ILIKE` | Pattern match (case-insensitive) | `/name ILIKE '%conn%'` |
| In | `IN` | Match any in list | `/state IN ['ACTIVE', 'PROVISIONING']` |
| Greater Than | `>` | Greater than value | `/bandwidth > 1000` |
| Less Than | `<` | Less than value | `/bandwidth < 5000` |
| Greater Than or Equal | `>=` | Greater than or equal to | `/bandwidth >= 1000` |
| Less Than or Equal | `<=` | Less than or equal to | `/bandwidth <= 10000` |
| Between | `BETWEEN` | Range inclusive | `/bandwidth BETWEEN [1000, 5000]` |

**Important:** The following operators are NOT supported:
- `CONTAINS` — use `LIKE '%value%'` or `ILIKE '%value%'` instead
- `STARTS_WITH` — use `LIKE 'value%'` or `ILIKE 'value%'` instead
- `ENDS_WITH` — use `LIKE '%value'` or `ILIKE '%value'` instead

### Operator guidance by property type

Use the operator that matches the kind of data being filtered — think by property
type, not by tool:

- **Enum properties** (`state`, `type`, `package/code`): use `=` for a single
  value; `IN` for multiple values.
- **String / name properties**: use `=` for exact match; `LIKE` with `%`
  wildcards for partial match (e.g. `%conn%`); `ILIKE` for case-insensitive
  partial match where supported.
- **Numeric properties** (`bandwidth`): use `=`, `>`, `<`, `>=`, `<=`, or
  `BETWEEN [low, high]` for range queries.
- **Datetime properties** (`changeLog/createdDateTime`, `changeLog/lastUpdatedDateTime`,
  `changeLog/deletedDateTime`, `expirationDateTime`): use `BETWEEN` with ISO-8601
  timestamps for a range; `<=` or `>=` for a one-sided bound.
  Format: `2026-03-20T20:09:41.631Z`
- **Multiple location values**: use `IN` with up to 10 metro codes. If the
  tool's schema does not list `IN` as a supported operator, issue one `=` query
  per metro and merge the results client-side.

Each tool's schema declares which operators it supports — check it before
constructing a filter. The guidance above tells you *which operator to reach for*
once you know the tool supports it.

### Resource-specific references

Each resource type has its own reference file for the filter-property and presentation
nuance that the tool schema doesn't cover — read the matching one before constructing
the filter or rendering results:

- **Connections** → `references/search-connections.md` (filter properties for
  redundancy, equinix status, seller region, port name, VLAN CTag, access point type,
  created by, project ID; field mapping; required-column and summary rules)
- **Cloud routers** → `references/search-routers.md`
- **Ports** → `references/search-ports.md`
- **Service tokens** → `references/search-service-tokens.md` (also covers the expiry
  advisory)
- **Route filters** → `references/search-route-filters.md`
- **Route aggregations** → `references/search-route-aggregations.md`
- **Precision time services** → `references/search-time-services.md`
- **"Show me everything" (multi-type)** → `references/search-everything.md` (required reading before rendering a multi-type response)

### System-wide limits (apply to all tools)

- Maximum **8 filter expressions** (AND + OR combined) per query.
- `IN` operator accepts maximum **10 values**.

---

## Step 3 — Execute the search

Call the relevant tool(s). The tool descriptions contain full filter property
paths, state enums, package codes, and worked examples — do not reproduce them here.

**Count-only queries** (user says "count", "how many", "total number of"): call the tool with `limit=1` to minimise data transfer — you only need `pagination.total`. Return a single summary line such as *"Fabric connections in SV: 203"* or *"Total: 203"*. Do NOT list individual resources or show a table.

**Page size:** always set `limit` explicitly — do not rely on tool defaults, which are
usually 5. **Default to `limit=20` for a normal single-type query.** Go above 20 only when
the user explicitly asked for the whole set, and never past 100 (the API page maximum). For
a multi-type "show everything" request, use `limit=5` per tool (see below) — full pages for
every type will not fit in one response.

**A long response gets truncated mid-table, and a truncated table scores worse than a short
one.** Fetching 100 rows because you can is how this skill produces ragged half-rows, a
duplicate-looking table, and no closing line — the reader sees an answer that was cut off
rather than one that was scoped. 20 rows plus an accurate total is a better answer than 100
rows that don't fit. A ports query answered with 28 rows was marked down for exactly this.

**Render every row you fetched, and never more.** The failure mode that matters is not
table length, it is a mismatch between what you fetched, what you rendered, and what you
claimed. A 100-row table is fine if the response says 100 rows are shown out of the real
total. What is never fine is fetching 100 and rendering 25 without saying so.

**Prefer a smaller page when the answer doesn't need the rows.** A count-only question
needs `limit=1`; a "show me a few" needs 20. Don't fetch 100 rows to answer a question
about how many there are.

### The scope block — required, and it goes FIRST

**Open every inventory response with this block, above the table.** Not below it, not at
the end — first.

| Field | Value |
|---|---|
| Tool | `search_connections` |
| Filter | `type = EVPL_VC` AND (`aSide…metroCode = SV` OR `zSide…metroCode = SV`) |
| Total matching | 119 |
| Shown | 20 |

**Why it goes first, and why this is not a style preference.** A long table gets cut off
partway through. When the filter-and-count disclosure sits at the bottom, the responses
that most need it — the long ones — are exactly the ones that lose it, and the answer
arrives as a bare table with no total. That has now happened repeatedly: the same queries
get marked down for "omits the total count", "inconsistent total count", and "no evidence
of actual data retrieval" while the tool was demonstrably called every time. Put the
disclosure where truncation cannot reach it.

A closing sentence restating the same facts is still welcome — *"Called `search_connections`
with `type = EVPL_VC` — showing 20 of 119."* — but it is now the optional half. The leading
block is the required half. The skill's best-scoring responses have consistently opened
with a block of this shape.

The block carries four requirements. Each one has failed an eval on its own:

- **Name the tool and the filter.** This is the only proof that you applied the filter. A
  bare table that opens with `| Name | UUID | Status | … |` and no scope block reads as
  unfiltered, even when the filter was correct.
- **Give the total when the user asks for a "total count".** A request that says "include
  Name, UUID, Status, Bandwidth, Type, and total count" needs the total. Without it, the
  answer is a direct miss.
- **T is `pagination.total`. Nothing else is a total.** A fetched or rendered row count is
  not a total. If you did not read `pagination.total` from the response, say so. T can
  never be smaller than N.
- **N is the number of rows in your table.** Count them. Do not use the `limit` you
  requested. If you write "showing 20", the table holds exactly 20 rows.

**Say so when you drop rows.** *"Duplicate UUID rows returned by the API were excluded"* is
exactly right — the reader now knows why N and T don't reconcile exactly. Silent
deduplication is what turns a correct table into an unexplainable one.

**"Show everything" queries** (the user asks for all resource types at once, e.g. "show me
everything in my project"): call these six every time, even when the request also names a
project or metro — `search_connections`, `search_ports`, `search_routers`,
`search_service_tokens`, `search_route_filters`, `search_route_aggregations`. Count your
tool calls before you respond; fewer than six means you skipped one.

Also attempt `search_time_services`, but **treat it as best-effort.** If it returns 403 or is
unavailable, omit precision time services and carry on — that is not a failed response. Do
not retry it and do not let it block the other six.

Resolve the project from context, such as a project ID named earlier in the conversation.
**If you cannot resolve one, still run every search unscoped.** Do not ask which
project — "show me everything in my project" already passed Check 1, so asking it there is
the failure this skill warns about. Report the results, and if they span more than one
project add the one-line note in Step 4. Never fall back to a subset of resource types. A
type that returns 0 results is omitted from the output but must still have been called —
"skip" means "do not show", never "do not call". Run the searches in parallel where the
environment supports it.

**Keep every call small — `limit=5` per tool.** An "everything" request is the widest query
this skill supports, and full-size result sets will not fit in one response. What the user
needs is a per-type overview: each type's `pagination.total`, plus example rows for the
largest one or two. Fetching 20–100 rows per type is how this request produces a truncated
answer instead of an inventory summary.

**Once any tool has returned data, you must present that data.** Never fall back to the
generic out-of-scope reply ("I can provide information and support related to Equinix
products, services, and processes…") after your searches have already succeeded. An
inventory request that came back with results is unambiguously in scope; answering it
with the out-of-scope template throws away work you already did and tells the user
something false about their own request. If the combined result set feels too large to
render, reduce it to per-type counts plus the first few rows of each and say that is what
you're showing. Never respond with nothing.

---

## Step 4 — Present results

The API returns complete objects with 30–50 fields each. Present only the curated
columns below. Drop every other field.

**Read the matching reference file before rendering any table — every time, even if you
already read it earlier in Step 2.** The field-to-source mapping lives there, and for
connections in particular it is not guessable: `Status` is `operation.equinixStatus`,
`Route` is built from `aSide.accessPoint.location.metroCode` and
`zSide.accessPoint.name`. Skipping the read is how a table ends up as a dump of whatever
keys the API happened to return.

The table below is the **floor, not a substitute for that read**: if for any reason you
render without having opened the reference, render exactly these columns and nothing else.

| Resource type | Columns, in this order — nothing else |
|---|---|
| Connections | Name, UUID, Status, Bandwidth, Route, Type |
| Cloud routers (FCRs) | Name, UUID, State, Package, Metro, Connections, Created |
| Ports | Name, UUID, State, Metro, Device, Bandwidth, Used bandwidth |
| Service tokens | Name, UUID, State, Type, Expires, Created, Issued by |
| Route filters | Name, UUID, State, Type, notMatchedRuleAction, Last updated |
| Route aggregations | Name, UUID, State, Type, Last updated |
| Precision time services | Name, UUID, State, Package |

Field-to-source mappings live in the matching reference file — several are not the
obvious field name (a connection's `Status` is `operation.equinixStatus`, not `state`; a
port's `Device` is `device.name`). Use `—` for a field the API returned empty; never leave
a cell blank.

**Two permitted variations on that list:**

- **Connections** — you may split **Route** into separate **A-side Metro** and **Z-side
  Metro** columns if that reads better for the result set. Both forms are fine. What is not
  fine is dropping **Status** while doing it: `UUID | Name | Type | State | Bandwidth |
  Direction | A-side Metro | Z-side Metro | Project` has traded away `operation.equinixStatus`
  — the field that says whether the connection is actually *usable* — for `state` plus two
  columns nobody filtered on.
- **Ports** — **Metro** and **Device** are both required. `IBX` is a finer-grained location
  than `Metro`, not a replacement for it: add it beside Metro if useful, never instead of it.
  A ports header of `UUID | Name | State | IBX | Bandwidth | Available | Used | Encapsulation
  | Connections | Updated` fails twice over — Metro and Device both missing, and three
  unfiltered columns added.

**Never fill an empty cell with a different field's value.** Many resources have no `name`
— service tokens especially. An unnamed resource gets `—` in its **Name** cell, never its
UUID copied across. A table where some rows read `| a613c7f7-… | a613c7f7-… |` and others
read `| Secondary | c60c5785-… |` makes it look like the named and unnamed rows are
different kinds of object, and there is no way for the reader to tell which names are real.

**Plus the filtered-on column, always.** If the user filtered on a property that isn't in
the list above, add it as a column — it is the only visible proof the filter was applied.
Filtered by redundancy priority → add **Redundancy Priority** (`redundancy.priority`); by
encapsulation → **Encapsulation**; by package → **Package**; by project ID → **Project
ID**. Never drop it, and never swap it for a similarly-named different field.

**When the user names the columns, give exactly those columns.** A request for "Name, UUID,
Status, Bandwidth, Type" gets those five and no others — drop the curated columns the user
left out, including **Route**. An explicit column list overrides the curated list above. Add
back only the filtered-on column if the user's list omits it.

**Hard ceiling: 9 columns.** Curated columns plus filtered-on columns never exceed 9. If
your table has more than 9, you are dumping raw API fields — stop and rebuild it from the
list above.

**Never use raw API field names as headers.** Headers are the Title-Case labels in the
table above — `Name`, `UUID`, `State`, `Metro`, `Device` — never the JSON keys they come
from. A ports table headed `uuid | name | state | metro | ibx | bandwidth | available |
used | …` fails twice over: lowercase raw keys, and `ibx` substituted for the required
**Device** (`device.name`), which drops that column entirely. `references/search-ports.md`
carries the full worked example. Rendering a field because it happened to be in the
response is a failure, not thoroughness.

**Deduplication:** each resource appears exactly once. If the same UUID comes back twice —
from pagination, or from two separate calls — render one row, not two.

**Never merge results from calls with different filter scopes into one table.** If you
issued two searches, either present them as separate tables with their own filters stated,
or re-query once with the combined filter.

**Service tokens — expiry advisory.** Always evaluate each token's `expirationDateTime`
against today's date, whether or not the user asked about expiry. Mark it inline in
**Expires** (`2026-03-30 ⚠️ expired`) or as one extra column, your choice; that column
counts against the 9-column ceiling like any other. Keep the listing call at `limit=20`.

**Sort every token into exactly one of three buckets. Never apply one verdict to the whole
table.**

| Bucket | Condition | Render as |
|---|---|---|
| Already past expiry | `expirationDateTime` < today | `⚠️ expired` |
| Expiring soon | today ≤ `expirationDateTime` ≤ today + 30 days | `⚠️ expires in N days` |
| Fine | more than 30 days out | plain date, no marker |

The middle bucket answers "flag anything expiring soon" — it is the only one that does. A
response that stamps `⚠️ Expired` on **every** row has labelled a page, not run the check;
that shape scored 0.20. Tokens reading `ACTIVE` with a past expiry is a real API
discrepancy: report it **once as a count**, never as a verdict down a column.

**Answer "expiring soon" with its own scoped call.** Token order is not guaranteed, so a
near-future expiry can sit on a page you never read. Compute the absolute ISO-8601 range
first, then issue a second call:

`search_service_tokens` with `state = ACTIVE` AND
`expirationDateTime BETWEEN ['<today>T00:00:00.000Z', '<today+30d>T23:59:59.999Z']`

and report that response's `pagination.total` as the expiring-soon count. The per-row check
above is still required: the second call gives the count, the row markers show which ones.

Close with both counts and the bucket breakdown: *"Called `search_service_tokens` with
`state = ACTIVE` — showing 20 of 34. Of the 34, 2 expire within 30 days and 18 are already
past expiry while still reporting ACTIVE."*

### Multi-type ("show everything") format

**Read `references/search-everything.md` before you render this response.** It is the
authoritative format and it is not guessable — five different attempts to state it inline
have failed, scoring 0.20 to 0.60.

The shape it enforces, in this order:

1. A one-line project scope note.
2. A count table, one row per type that returned results, `Total` an integer of 1 or more.
3. One section per count-table row — header with its count, example rows for only the
   largest one or two types.

**Do not open with a connections table.** That single mistake accounts for the worst scores
this query has produced.

### Zero results

**You may only report zero results after calling the tool and confirming `pagination.total = 0` in the actual API response.** Never infer or assume zero results without a tool call.

Once the tool confirms zero results:
- **Do not show an empty table.** Headers with no rows confuse the user.
- Write *"No [resource type] found matching your criteria (total: 0)."*
- Name the filter you applied, e.g. *"Searched for state = ACTIVE in metro SV."*
- Suggest a next step: check the project ID, broaden the state filter, or confirm the metro
  code. See the matching reference file for resource-specific guidance.

### Before you respond

Check each item. Each one has caused a real failure.
- Did Checks 1 and 2 pass? For an unscoped request, did I ask the user to narrow it?
- Did I call the tool this turn and get a real response back?
- **Does the response OPEN with the scope block** — Tool, Filter, Total matching, Shown?
  A response that starts with a bare `| Name | UUID | … |` row is missing it.
- Is every "total" the `pagination.total` from the response, and is it >= my row count?
- Does the row count in the table match the number I claim?
- Does every row satisfy the filter, checked against the real field values?
- Is the filtered-on property still its own column?
- Is the table 9 columns or fewer, with no raw API field names?
- Is every row a distinct UUID?
- Does every row's state pass the Step 1 filter (no `INACTIVE` or `DEPROVISIONED`)?
- Did I set `limit` to 20 for a single-type query, and does the table render complete?
- For "show everything": is every Total cell an integer of 1 or more, does my number of
  sections equal my number of table rows, and did I call the six required searches?
- For a token question: did I bucket each row as expired, expiring-soon, or fine?

---

## Step 5 — Offer next steps

After presenting results, offer relevant follow-on actions based on what was found:

- **Connections found** → "Want me to check health or utilization metrics for any of these?" → `monitor-fabric-health`
- **Routers found** → "Want me to inspect routing tables or BGP state on any of these?" → `troubleshoot-connectivity`
- **Route filters or aggregations found** → "Want me to show the rules attached to any of these?" (use `get_all_route_filter_rules` or `get_all_route_aggregation_rules`)
- **Tokens expiring soon** → "Want me to update the expiry on any of these?" → `update-fabric-resource`
- **0 results** → "Want me to search in a different project or broaden the scope?"

---

## Error handling

- **"Operator not supported":** The operator used is not valid for the specified property
  or tool. Fall back to `=` only and retry. Check Step 2 operator guidance for valid
  operators per property type.
- **"Property not supported":** The filter property path is not supported by the API.
  Verify the property path against the tool schema for supported filter properties.
- **HTTP 400 on a filter:** most likely an unsupported operator or property for that tool.
  Fall back to `=` only and retry. Check Step 2 operator guidance.
- **HTTP 401/403:** inform the user of the likely missing role — `Fabric Connection Manager`
  for connections, `Fabric Cloud Router Manager` for routers. Other resource types
  follow the same naming pattern.
- **Partial failure** (one tool fails, others succeed): return the successful results
  and note which resource type could not be retrieved and why.
- **Empty result set with filters applied:** suggest broadening — remove the metro
  filter, check the project ID, or try `PROVISIONED` state instead of `ACTIVE`.

---

## Boundary notes

This skill is **read-only**. It does not create, modify, or delete any resource.

**This skill covers only the seven resource types in the description**, plus their
metadata, such as the rules attached to a route filter or aggregation. It is not a
general-purpose "list or get anything" handler.

| User wants... | Use instead |
|---|---|
| Marketplace options, pricing, what could be built | `explore-fabric-connectivity-options` |
| Bandwidth utilisation, latency, error metrics | `monitor-fabric-health` |
| BGP diagnosis, routing tables, PING commands | `troubleshoot-connectivity` |
| Change bandwidth, update token expiry, modify config | `update-fabric-resource` |
| Delete or decommission resources | `remove-fabric-resource` |
| Streams, stream subscriptions, stream alert rules, stream-attached assets | call the MCP tool directly via `create_filter` |
| Routing protocols, routing protocol actions | call the MCP tool directly via `create_filter` |
| Agents, agent templates, agent activities | call the MCP tool directly via `create_filter` |
| Service profiles, metros, metrics, cloud events, company profile | call the MCP tool directly via `create_filter` |

**The `create_filter` rows above are the only exception to "never route these reads to a
generic filter tool".** That rule protects the seven types this skill owns. For any other
type, `create_filter` is correct.
