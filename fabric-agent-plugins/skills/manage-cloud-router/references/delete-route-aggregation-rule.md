# Delete an existing Route Aggregation Rule
Detailed steps for deleting an existing route aggregation rule.
Follow these steps when the user's intent is to delete or remove an existing route aggregation rule.

## Objective
Soft-delete an existing route aggregation rule by requesting deprovisioning and verifying state transition.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Route aggregation rule UUID and route aggregation UUID are known or can be gathered from the user.

## Definition of Done
- No errors returned from the delete call.
- Route aggregation rule remains queryable via `get_route_aggregation_rule` using route aggregation rule UUID + route aggregation ID.
- Route aggregation rule state transitions to `DEPROVISIONING` or reaches terminal `DEPROVISIONED`.

## Steps

### 1. Check Route Aggregation Rule Exists
If route aggregation UUID is missing, ask the user to provide it before continuing.
If route aggregation rule UUID is missing, ask the user to provide one before continuing.

Call `get_route_aggregation_rule` with route aggregation rule UUID, route aggregation UUID to resolve the target route aggregation rule.

If no route aggregation rule is returned, do not call `delete_route_aggregation_rule`. Ask the user to confirm route aggregation UUID and route aggregation rule UUID.
If multiple route aggregation rules match, ask the user to select one by route aggregation rule UUID before continuing.

### 2. Confirm Before Delete
Using the resolved route aggregation rule from Step 1 (`get_route_aggregation_rule`), present the current route aggregation rule details to the user:
- Route aggregation rule UUID
- Route aggregation UUID (always show this for scope clarity)
- Name
- Description (if present)
- State (if present)

Warn the user that this action deprovisions the route aggregation rule and may impact associated resources.

Require the user to confirm before proceeding to Step 3 (yes/no). The confirmation prompt must explicitly include the selected route aggregation rule UUID and clearly state the action being performed.
- If the user confirms "yes", proceed to Step 3.
- If the user responds "no", do not proceed to Step 3. Inform the user that the operation has been canceled. End the workflow and wait for a new user request.

### 3. Delete the Route Aggregation rule
Call `delete_route_aggregation_rule` once with the resolved route aggregation rule UUID.
On error, do not proceed to verification. Follow the Error Handling table.

### 4. Verify Delete
Only proceed if `delete_route_aggregation_rule` returned successfully.
Call `get_route_aggregation_rule` using route aggregation rule UUID and route aggregation UUID.

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the route aggregation rule is returned and whether state is `DEPROVISIONING` or `DEPROVISIONED`.
3. If found with `DEPROVISIONING` or `DEPROVISIONED`, treat verification as successful and display route aggregation rule UUID, route aggregation UUID, name, description (if present), state, and timestamp.
4. If not found after all retries, report verification as inconsistent for a soft delete, surface "delete submitted but route aggregation rule not visible", and ask the user to retry verification shortly.
5. If found but state is not `DEPROVISIONING` or `DEPROVISIONED`, report "delete submitted but state not yet transitioned", and ask the user to retry verification shortly.

## Error Handling

| Error                                                   | Action                                                                                                                                                                    |
|---------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Route aggregation rule not found                        | Do not delete; ask user to confirm route aggregation UUID and route aggregation rule UUID, then re-run `get_route_aggregation_rule`                                       |
| Invalid UUID format                                     | Ask for a valid UUID format and retry                                                                                                                                     |
| Route aggregation rule in use                           | Tell user to remove or detach dependencies, then retry deletion.                                                                                                          |
| Route aggregation rule not visible after delete request | Report verification inconsistency for soft delete; ask user to retry verification shortly                                                                                 |
| Delete failed                                           | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Delete not yet reflected                                | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Authorization error                                     | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited                           | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)                                   | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error                                   | Return the tool error summary to the user, state that route aggregation rule deletion did not complete, and end the workflow.                                             |