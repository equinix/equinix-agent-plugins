# Updating an existing Fabric Cloud Router
Detailed steps for updating an existing Equinix Fabric Cloud Router (FCR).
Follow these steps when the user's intent is to update or modify an existing FCR.

## Objective
Update an existing Equinix FCR with user-confirmed name, package, and notification emails.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Router identifier (UUID or name) and project ID are known or can be gathered from the user.

## Definition of Done
- Router UUID is returned from the update call.
- Router can be retrieved via `search_routers` using router UUID + project ID.
- Returned configuration reflects the user-confirmed updated fields (package, name, and/or notification emails).

## Steps

### 1. Check Router Exists
If project ID is missing, ask the user to provide it before continuing.
If router identifier (UUID or name) is missing, ask the user to provide one before continuing.

Call `search_routers` with project ID, router identifier, and pagination (`offset: 0, limit: 10`) to resolve the target router.

If no router is returned, do not call `update_router`. Ask the user to confirm project ID and router identifier.
If multiple routers match, ask the user to select one by UUID before continuing.

### 2. Choose Update Fields
Ask the user which fields to update (one or more):
- **Package** — if the user wants to compare options, call `list_router_packages` and present differences.
- **Router name**
- **Notification emails**

If no fields are selected, do not call `update_router`. Ask the user what they want to change.

### 3. Validate Package
If the user specified a package, call `list_router_packages` to list valid router packages and verify that the requested package exists before proceeding.

### 4. Confirm Before Update
Present the update to the user using the following format and require explicit confirmation before calling `update_router`:

Action: [verb] router [UUID] in project [projectId]

┌──────────────┬─────────────────┬─────────────┐
│    Field     │  Current value  │  New value  │
├──────────────┼─────────────────┼─────────────┤
│ [field name] │ [current value] │ [new value] │
└──────────────┴─────────────────┴─────────────┘

Where:
- Action verb is one of: Rename, Upgrade, Update
- Include only the fields that are being changed in the table
- If package is changing, use current and new package codes as the values
- If notification emails are changing, list them as comma-separated values

Require the user to confirm before proceeding to Step 5 (yes/no). The confirmation prompt must explicitly include the selected router UUID and clearly state the action being performed.
- If the user confirms "yes", proceed to Step 5.
- If the user responds "no", do not proceed to Step 5. Inform the user that the operation has been canceled. End the workflow and wait for a new user request.

### 5. Update the Router
For each confirmed field change, call `update_router` sequentially — one call per field (package, name, emails). Do NOT call `update_router` in parallel for multiple fields; the router enters
a transient state after the first call and will reject concurrent updates. Wait for each call to return successfully before issuing the next.

Capture from response: router UUID, project ID, state, and update timestamp.

### 6. Verify Update
Only proceed if `update_router` returned successfully with a router UUID.
Call `search_routers` using the returned router UUID and project ID using pagination (`offset: 0, limit: 1`).
If the response omits project ID, use the Step 4-confirmed project ID.

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the router is returned and whether requested field changes are reflected.
3. If found and changes are reflected, treat verification as successful and display router UUID, name, metro, package, state, and update timestamp.
4. If not found after all retries, mark verification incomplete, report "update submitted but not yet visible", and ask the user to retry verification shortly.
5. If found but requested fields are not updated after all retries, report "update submitted but changes are not yet reflected", and ask the user to retry verification shortly.

## Error Handling

| Error                          | Action                                                                                                                                                                    |
|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Router not found               | Do not update; ask user to confirm project ID and router UUID/name, then re-run `search_routers`                                                                          |
| Package code invalid           | Show the four valid codes: LAB, BASIC, STANDARD, ADVANCED                                                                                                                 |
| Update failed                  | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Update not yet reflected       | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Authorization error            | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited  | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)          | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error          | Return the tool error summary to the user, state that router update did not complete, and end the workflow.                                                               |