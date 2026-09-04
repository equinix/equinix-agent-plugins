# Delete an existing route filter
Detailed steps for deleting an existing route filter.
Follow these steps when the user's intent is to delete or remove an existing route filter.

## Objective
Soft-delete an existing route filter by requesting deprovisioning and verifying state transition.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Route filter UUID and project ID are known or can be gathered from the user.

## Definition of Done
- No errors returned from the delete call.
- Route filter remains queryable via `search_route_filters` using route filter UUID + project ID.
- Route filter state transitions to `DEPROVISIONING` or reaches terminal `DEPROVISIONED`.

## Steps

### 1. Check Route Filter Exists
If project ID is missing, ask the user to provide it before continuing.
If route filter UUID is missing, ask the user to provide one before continuing.

Call `search_route_filters` with project ID, route filter UUID, and pagination (`offset: 0, limit: 10`) to resolve the target route filter.

If no route filter is returned, do not call `delete_route_filter`. Ask the user to confirm project ID and route filter UUID.
If multiple route filters match, ask the user to select one by route filter UUID before continuing.

### 2. Confirm Before Delete
Using the resolved route filter from Step 1 (`search_route_filters`), present the current route filter details to the user:
- Route filter UUID
- Current project ID (always show this for scope clarity)
- Name
- Description (if present)
- State (if present)

Warn the user that this action deprovisions the route filter and may impact associated resources.

Require the user to confirm before proceeding to Step 3 (yes/no). The confirmation prompt must explicitly include the selected route filter UUID and clearly state the action being performed.
- If the user confirms "yes", proceed to Step 3.
- If the user responds "no", do not proceed to Step 3. Inform the user that the operation has been canceled. End the workflow and wait for a new user request.

### 3. Delete the Route Filter
Call `delete_route_filter` once with the resolved route filter UUID.
On error, do not proceed to verification. Follow the Error Handling table.

### 4. Verify Delete
Only proceed if `delete_route_filter` returned successfully.
Call `search_route_filters` using route filter UUID and project ID with pagination (`offset: 0, limit: 1`).

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the route filter is returned and whether state is `DEPROVISIONING` or `DEPROVISIONED`.
3. If found with `DEPROVISIONING` or `DEPROVISIONED`, treat verification as successful and display route filter UUID, project ID, name, description (if present), state, and timestamp.
4. If not found after all retries, report verification as inconsistent for a soft delete, surface "delete submitted but route filter not visible", and ask the user to retry verification shortly.
5. If found but state is not `DEPROVISIONING` or `DEPROVISIONED`, report "delete submitted but state not yet transitioned", and ask the user to retry verification shortly.

## Error Handling

| Error                                         | Action                                                                                                                                                                    |
|-----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Route filter not found                        | Do not delete; ask user to confirm project ID and route filter UUID, then re-run `search_route_filters`                                                                   |
| Invalid UUID format                           | Ask for a valid UUID format and retry                                                                                                                                     |
| Route filter in use                           | Tell user to remove or detach dependencies, then retry deletion.                                                                                                          |
| Route filter not visible after delete request | Report verification inconsistency for soft delete; ask user to retry verification shortly                                                                                 |
| Delete failed                                 | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Delete not yet reflected                      | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Authorization error                           | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited                 | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)                         | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error                         | Return the tool error summary to the user, state that route filter deletion did not complete, and end the workflow.                                                       |