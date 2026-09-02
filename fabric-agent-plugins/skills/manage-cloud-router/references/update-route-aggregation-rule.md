# Update a Route Aggregation Rule
Detailed steps for updating an existing route aggregation rule name, description or prefix.

## Objective
Update a single route aggregation rule attribute (`name`, `description`, `prefix`, or `prefixMatch`) using user-confirmed values.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Route aggregation rule UUID is known or can be gathered from the user.
- Route aggregation UUID is known or can be gathered from the user.

## Definition of Done
- `update_route_aggregation_rule` succeeds with no API errors.
- Route aggregation rule can be retrieved via `get_route_aggregation_rule` using route aggregation rule UUID and route aggregation UUID.
- Retrieved configuration reflects the user-confirmed updated field and value.

## Steps

### 1. Resolve Route Aggregation Rule UUID and Route Aggregation UUID
If route aggregation rule UUID is unknown, ask the user to provide one before continuing.
If route aggregation UUID is unknown, ask the user to provide one before continuing.

Call `get_route_aggregation_rule` with:
- Route aggregation rule UUID
- Route aggregation UUID

If no route aggregation rule is returned, ask the user to confirm route aggregation UUID and route aggregation rule UUID are correct. Do not proceed to Step 2.
If multiple route aggregation rule match (unexpected), ask the user to select one by UUID, then verify it is the intended target.

### 2. Choose Update Type and Value
Ask the user which single field to update:
- `name` — must be a non-empty string
- `description` — must be a non-empty string
- `prefix` — must be a non-empty string

Collect:
- **Update type** (one of the four above)
- **New value** (string)

If the user does not provide both, ask again until provided.

### 3. Validate the New Value
- If `update_type` is `name` or `description` or `prefix`: ensure the value is non-empty. If empty, ask the user to provide a non-empty string.
Do not proceed to Step 4 until both update type and new value are valid.

### 4. Confirm Before Updating
Present a review summary to the user:
- Route aggregation rule UUID
- Current route aggregation UUID (always show this for scope clarity)
- Update type (one of: `name`, `description`, `prefix`)
- Current value (old data before change)
- New value (what will be written)

Require the user to confirm before proceeding to Step 5 (yes/no). The confirmation prompt must explicitly include the selected route aggregation rule UUID and clearly state the action being performed.
- If the user confirms "yes", proceed to Step 5.
- If the user responds "no", do not proceed to Step 5. Inform the user that the operation has been canceled. End the workflow and wait for a new user request.

### 5. Update the Route Aggregation rule
Call `update_route_aggregation_rule` once with:
- `route_aggregation_rule_uuid` (confirmed in Step 1)
- `route_aggregation_uuid` (confirmed in Step 1)
- `update_type` (chosen in Step 2, validated in Step 3: one of `name`, `description`, `prefix`, `prefixMatch`)
- `value` (chosen in Step 2, validated in Step 3: non-empty string)

**On success:** Capture and display:
- Route aggregation rule UUID
- Updated field name (which field changed: name, description, prefix, or prefixMatch)
- New value (what the field was changed to)
- Update timestamp (when the change was recorded)

**On error:** Do not proceed to verification. Check the Error Handling table below and take the corresponding action.

### 6. Verify Update
Only proceed if `update_route_aggregation_rule` returned successfully with a route aggregation rule UUID.

Prepare the verification call:
- Use route aggregation rule UUID from the update response
- Route aggregation UUID

Call `get_route_aggregation_rule` with the above parameters.

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the route aggregation rule is returned and whether requested field changes are reflected.
3. If found and changes are reflected, treat verification as successful and display route aggregation rule UUID, name, description, prefix, prefixMatch, and update timestamp.
4. If not found after all retries, mark verification incomplete, report "update submitted but not yet visible", and ask the user to retry verification shortly.
5. If found but requested fields are not updated after all retries, report "update submitted but changes are not yet reflected", and ask the user to retry verification shortly.

## Error Handling
| Error                                                                          | Action                                                                                                                                                                    |
|--------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Route aggregation rule not found (Step 1)                                      | Do not proceed. Ask user to confirm route aggregation UUID and route aggregation rule UUID, then re-run `get_route_aggregation_rule`.                                     |
| Invalid update type                                                            | Explain valid update types: `name`, `description`, `prefix. Ask the user to select one and retry.                                                                         |
| Invalid value format (name/description)                                        | String must be non-empty. Ask the user to provide a non-empty string and retry.                                                                                           |
| Update failed                                                                  | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Route aggregation rule not found (Step 6 verification)                         | Likely indexing delay. Ask user to retry verification shortly; do not make another write call.                                                                            |
| Route aggregation rule found but field not yet reflected (Step 6 verification) | Likely indexing delay. Ask user to retry verification shortly; do not make another write call.                                                                            |
| Authorization error                                                            | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited                                                  | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)                                                          | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error                                                          | Return the tool error summary to the user, state that route aggregation update did not complete, and end the workflow.                                                    |