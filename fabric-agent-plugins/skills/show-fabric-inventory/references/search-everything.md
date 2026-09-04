# Multi-type ("show me everything") rendering

Read this file before you render an "everything" response. It is the authoritative format
for a multi-type answer. `SKILL.md` Step 4 points here.

This query has been answered wrongly more often than any other in this skill. Every rule
below exists because a real response broke it and scored what is noted.

## The output has exactly three parts, in this order

1. **A one-line project scope note.**
2. **A count table** — one row per resource type that returned results.
3. **A section per count-table row** — header with its count, and example rows for only the
   largest one or two types.

Nothing else goes above the count table. Do not open with a connections table.

## Part 1 — project scope line

One line, above everything else.

| Situation | Write |
|---|---|
| Project resolved | `resolved project uuid` |
| No project resolvable | `No project was specified — showing inventory across all projects you can access.` |

Silence here reads as a scoping failure. One response lost credit for showing several
project IDs with no such line.

## Part 2 — the count table

| Resource type | Total |
|---|---|
| Connections | 363 |
| Ports | 41 |
| Cloud routers | 12 |
| Service tokens | 34 |
| Route filters | 8 |
| Route aggregations | 2 |

**The Total column holds an integer of 1 or more. Nothing else may go in that column** — not
a word, not a status, not a dash.

| `pagination.total` | Action |
|---|---|
| 1 or more | Write the row with the integer |
| 0 | Write no row. The type does not appear in the response at all. |
| the search failed | Write no row. Name the type in one sentence below the table. |

A table of `| Ports | Returned |`, `| Connections | Search failed |`,
`| Service Tokens | None returned |` scored **0.50**. Every cell there is wrong: `Returned`
is a status, not a total, and the other two types belong outside the table.

**Each Total is that type's `pagination.total`, not the row count of the page.** Counts such
as `Connections 31, Ports 32, Routers 30` all sit near the page size — that is the signature
of `len(results)`. Real totals differ by orders of magnitude.

## Part 3 — sections, in two passes

Do not write the whole of one type before starting the next.

1. **Pass 1 — write every section header first**, one per count-table row, same order, each
   with its count: `## Connections (363)`, `## Ports (41)`, and so on. Six rows means six
   headers. Stop and count your headers against the table before continuing.
2. **Pass 2 — add up to 3 example rows under the one or two largest types only.** Every
   other section keeps its header and count with no rows. A count alone is a complete answer
   for a small type.

Two passes, with rows for only the largest types, because rows everywhere runs out of room.
Writing connections in full first produced rows for connections and nothing for the other
five types (**0.60**), and a bare connections table with no grouping at all (**0.20**).

A type with no row in the count table gets no header either.

## Worked example

```
Project 111111000111111.

| Resource type | Total |
|---|---|
| Connections | 363 |
| Ports | 41 |
| Route filters | 8 |

## Connections (363)
| Name | UUID | Status | Bandwidth | Route | Type |
|---|---|---|---:|---|---|
| … three rows … |

## Ports (41)

## Route filters (8)
```

Ports and route filters carry their counts and no rows. That response is complete.

## Failure shapes to avoid

| Shape | Score |
|---|---|
| Only a connections table, no grouping or counts | 0.20 |
| Count table with status words instead of integers | 0.50 |
| Rows for connections, nothing for the other five types | 0.60 |
| Counts for every type but no rows at all | 0.55 |
| An empty section for a type whose total was 0 | 0.20 |

## Tool calls

"Skipping" a type in the output is not the same as skipping its tool call. The six required
searches must all run this turn (see `SKILL.md` Step 3), even when some types are omitted
here. `search_time_services` is best-effort: on 403 or unavailability, omit precision time
services and carry on.
