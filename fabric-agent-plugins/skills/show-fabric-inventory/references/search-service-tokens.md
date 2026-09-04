# Search Service Tokens

Filter-property reference and presentation rules specific to the
`search_service_tokens` tool. Read this whenever the user's request is about
**service tokens** — before constructing the filter and before rendering the
results table.

The shared rules in `SKILL.md` (state-filter defaults, operator syntax, page size,
pagination disclosure, the "Before you respond" checklist) still apply on top of
everything below — this file only covers what's specific to service tokens.

## Service token-specific filter properties

| User asks for... | Filter property | Example |
|---|---|---|
| Token UUID | `uuid` | `uuid = '...'` |
| Name | `name` | `name LIKE '%partner%'` |
| State | `state` | `state = 'ACTIVE'` |
| Type | `type` | `type = '...'` — check the tool schema for the exact enum values it accepts |
| Project ID | `project/projectId` | `project/projectId = '111111000111111'` |
| Expiration | `expirationDateTime` | `expirationDateTime <= '2026-08-20T00:00:00.000Z'` |
| Created | `changeLog/createdDateTime` | `changeLog/createdDateTime BETWEEN [...]` |

**State values:** the full enum is `ACTIVE`, `INACTIVE`, `EXPIRED`, `DELETED`. The default
state filter (see `SKILL.md` Step 1) excludes `EXPIRED`, `INACTIVE` and `DELETED` unless the
user explicitly asks for terminal/inactive tokens.

**When the user says "active" explicitly (e.g. "show my active service tokens"),
that is a stronger, positive filter than the default.** Apply `state = 'ACTIVE'` as
an exact-match server-side filter — do not settle for the default exclusion-only
filter above. Excluding `EXPIRED`/`INACTIVE`/`DELETED` still lets other non-active states
(e.g. pending or provisioning states) through, which is not what "active" means to
the user. Verify every row's `state` field actually reads `ACTIVE` before rendering
it — if any row doesn't, the filter wasn't applied server-side and must be fixed
before you respond, not filtered out client-side after the fact.

**Type enum is not fixed here** — the tool's own schema is the source of truth for
valid `type` values; check it before constructing a `type` filter rather than
guessing.

**Critical:** always apply these as server-side filters — never retrieve all
tokens and filter client-side.

## Presenting service token results

| Field | Source |
|---|---|
| Name | `name` |
| UUID | `uuid` |
| State | `state` |
| Type | `type` |
| Expires | `expirationDateTime` formatted as a date |
| Created | `changeLog.createdDateTime` |
| Issued by | `changeLog.createdBy` |

Do **not** add a separate "⚠️ Expiry warning" column. Mark an at-risk token inline in its
**Expires** cell instead — `2026-09-14 ⚠️ 12 days` for one expiring within 30 days,
`2025-01-07 ⚠️ expired` for one already past its date. A dedicated column that repeats the
same warning text on nearly every row is noise, and it pushes the table past the 9-column
ceiling in `SKILL.md` Step 4.

**Token expiry advisory (always required, not conditional on the user asking):**
after retrieving tokens, always client-side check `expirationDateTime` against
today's date. Flag tokens expiring within 30 days even if the user did not ask
about expiry. `expirationDateTime` supports date operators server-side, but a
relative window like "within 30 days" must first be computed into an absolute
ISO-8601 range — the API cannot evaluate it as a relative expression. The
post-retrieval flag-and-warn step is still required for all tokens returned in the
page.

**When the user filtered by a specific property, include that property as a
column in the results table.** For example:
- Filtered by type → add **Type** column (`type`) — already a default, confirm
  it's present.
- Filtered by project ID → add **Project ID** column (`project.projectId`).
- Filtered by expiration window → the **Expires** column is already a default;
  confirm the inline ⚠️ marker is applied consistently to every at-risk row, not just
  the ones the user filtered on.

**Always render the per-token table — never substitute an aggregate metrics table.** A
two-column "Metric | Value" summary (e.g. "Active service tokens found: 34", "Expiring
within next 30 days: 0") is not sufficient, even when the user's phrasing focuses on
flagging expiry (e.g. "flag anything expiring soon"). The per-token table with Name,
UUID, State, Type, Expires, Created, and Issued by is mandatory; the expiry flag is an
additional per-row marker on that table, not a replacement for it.

**Deduplication:** each token appears exactly once. If pagination returns the same
UUID more than once, show it only once.

**Before rendering, verify all three of these against the actual API response — each
has caused a real failure:**
- Every row's `state` matches the filter you actually meant to apply (e.g. every row
  reads `ACTIVE` if the user said "active" — not just "not EXPIRED/INACTIVE/DELETED").
- Every row's `expirationDateTime` was checked against today's date, and the inline ⚠️
  marker is present in the **Expires** cell of every row within 30 days or already past
  — not just mentioned in the summary.
- The row count in your table does not exceed the `limit` you requested (default 20).
- Tokens returned as `ACTIVE` with an expiry date already in the past are a real API
  discrepancy worth surfacing — but report it **once**, in the closing sentence ("18 of
  the 20 shown are past their expiry date despite still reporting ACTIVE"), not as an
  identical warning repeated on 18 separate rows.

**After the table, a summary sentence is mandatory** — restating both the exact
filter(s) applied and the total count, plus a call-out of any tokens flagged for
upcoming expiry, e.g.:
- *"Found 8 service tokens (total: 8). 2 expire within 30 days — flagged above."*

**If `pagination.total` is greater than the number of rows shown, you must state
this explicitly** — never present a partial list without saying so: *"Showing 20 of
27 — ask me to show more or narrow your filter."*

## Zero service tokens found

Once the tool confirms `pagination.total = 0`:
- **Do NOT show an empty table.**
- State clearly: *"No service tokens found matching your criteria (total: 0)."*
- Explain what filter was applied: *"Searched for tokens expiring before
  2026-08-20."*
- Suggest next steps: check the project ID, broaden the expiration window, or try
  searching without the type filter.
