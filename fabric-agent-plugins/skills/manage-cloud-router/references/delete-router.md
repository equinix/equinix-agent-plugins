# Delete an existing Fabric Cloud Router
Detailed steps for deleting an existing Equinix Fabric Cloud Router (FCR).
Follow these steps when the user's intent is to delete or remove an existing FCR.

## Objective
Soft-delete an existing Equinix FCR by requesting deprovisioning and verifying state transition.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Router identifier (UUID or name) and project ID are known or can be gathered from the user.

## Definition of Done
- No errors returned from the delete call.
- Router remains queryable via `search_routers` using router UUID + project ID.
- Router state transitions to `DEPROVISIONING` or reaches terminal `DEPROVISIONED`.

## Steps

### 1. Check Router Exists
If project ID is missing, ask the user to provide it before continuing.
If router identifier (UUID or name) is missing, ask the user to provide one before continuing.

Call `search_routers` with project ID, router identifier, and pagination (`offset: 0, limit: 10`) to resolve the target router.

If no router is returned, do not call `delete_router`. Ask the user to confirm project ID and router identifier.
If multiple routers match, ask the user to select one by router UUID before continuing.

### 2. Confirm Before Delete
Using the resolved router from Step 1 (`search_routers`), present the current router details: UUID, name, metro, package, and state.
Warn the user that this action deprovisions the router and may impact associated resources.

Require the user to confirm before proceeding to Step 3 (yes/no). The confirmation prompt must explicitly include the selected router UUID and clearly state the action being performed.
- If the user confirms "yes", proceed to Step 3.
- If the user responds "no", do not proceed to Step 3. Inform the user that the operation has been canceled. End the workflow and wait for a new user request.

### 3. Delete the Router
Call `delete_router` once with the resolved router UUID.
On error, do not proceed to verification. Follow the Error Handling table.

### 4. Verify Delete
Only proceed if `delete_router` returned successfully.
Call `search_routers` using router UUID and project ID with pagination (`offset: 0, limit: 1`).

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the router is returned and whether state is `DEPROVISIONING` or `DEPROVISIONED`.
3. If found with `DEPROVISIONING` or `DEPROVISIONED`, treat verification as successful and display router UUID, name, metro, package, state, and timestamp.
4. If not found after all retries, report verification as inconsistent for a soft delete, surface "delete submitted but router not visible", and ask the user to retry verification shortly.
5. If found but state is not `DEPROVISIONING` or `DEPROVISIONED`, report "delete submitted but state not yet transitioned", and ask the user to retry verification shortly.

## Error Handling

| Error                                   | Action                                                                                                                                                                    |
|-----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Router not found                        | Do not delete; ask user to confirm project ID and router UUID/name, then re-run `search_routers`                                                                          |
| Router not visible after delete request | Report verification inconsistency for soft delete; ask user to retry verification shortly                                                                                 |
| Delete failed                           | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Delete not yet reflected                | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Authorization error                     | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited           | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)                   | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error                   | Return the tool error summary to the user, state that router deletion did not complete, and end the workflow.                                                             |