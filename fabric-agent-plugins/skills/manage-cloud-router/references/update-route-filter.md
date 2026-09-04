# Update a Route Filter
Detailed steps for updating an existing route filter name, description, or project ID.

## Objective
Update a single route filter attribute (`name`, `description`, or `project`) using user-confirmed values.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Route filter UUID is known or can be gathered from the user.
- Project ID is known or can be gathered from the user.

## Definition of Done
- `update_route_filter` succeeds with no API errors.
- Route filter can be retrieved via `search_route_filters` using route filter UUID and updated project ID (if project was changed).
- Retrieved configuration reflects the user-confirmed updated field and value.

## Steps

### 1. Resolve Route Filter UUID and Project
If route filter UUID is unknown, ask the user to provide one before continuing.
If project ID is unknown, ask the user to provide one before continuing.

Call `search_route_filters` with:
- Route filter UUID
- Project ID
- Pagination: `offset: 0, limit: 10`

If no route filter is returned, ask the user to confirm project ID and route filter UUID are correct. Do not proceed to Step 2.
If multiple route filters match (unexpected), ask the user to select one by UUID, then verify it is the intended target.

### 2. Choose Update Type and Value
Ask the user which single field to update:
- `name` — must be a non-empty string
- `description` — must be a non-empty string
- `project` — must be a valid project ID string

Collect:
- **Update type** (one of the three above)
- **New value** (string)

If the user does not provide both, ask again until provided.

### 3. Validate the New Value
- If `update_type` is `name` or `description`: ensure the value is non-empty. If empty, ask the user to provide a non-empty string.
- If `update_type` is `project`: ensure the value is a valid project ID format. If format is invalid, ask the user to provide a valid project ID.
Do not proceed to Step 4 until both update type and new value are valid.

### 4. Confirm Before Updating
Present a review summary to the user:
- Route filter UUID
- Current project ID (always show this for scope clarity)
- Update type (one of: `name`, `description`, `project`)
- Current value (old data before change)
- New value (what will be written)
- **If `update_type` is `project`:** explicitly state "Project ID will change from [old] to [new]"

Require the user to confirm before proceeding to Step 5 (yes/no). The confirmation prompt must explicitly include the selected route filter UUID and clearly state the action being performed.
- If the user confirms "yes", proceed to Step 5.
- If the user responds "no", do not proceed to Step 5. Inform the user that the operation has been canceled. End the workflow and wait for a new user request.

### 5. Update the Route Filter
Call `update_route_filter` once with:
- `route_filter_uuid` (confirmed in Step 1)
- `update_type` (chosen in Step 2, validated in Step 3: one of `name`, `description`, `project`)
- `value` (chosen in Step 2, validated in Step 3: non-empty string or valid project ID)

**On success:** Capture and display:
- Route filter UUID
- Updated field name (which field changed: name, description, or project)
- New value (what the field was changed to)
- Update timestamp (when the change was recorded)

**On error:** Do not proceed to verification. Check the Error Handling table below and take the corresponding action.

### 6. Verify Update
Only proceed if `update_route_filter` returned successfully with a route filter UUID.

Prepare the verification call:
- Use route filter UUID from the update response
- Project ID: use the new project ID if `update_type` was `project`; otherwise use the original project ID
- Pagination: `offset: 0, limit: 1`

Call `search_route_filters` with the above parameters.

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the route filter is returned and whether requested field changes are reflected.
3. If found and changes are reflected, treat verification as successful and display route filter UUID, name, description, project, state, and update timestamp.
4. If not found after all retries, mark verification incomplete, report "update submitted but not yet visible", and ask the user to retry verification shortly.
5. If found but requested fields are not updated after all retries, report "update submitted but changes are not yet reflected", and ask the user to retry verification shortly.


## Error Handling
| Error                                                                | Action                                                                                                                                                                    |
|----------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Route filter not found (Step 1)                                      | Do not proceed. Ask user to confirm project ID and route filter UUID, then re-run `search_route_filters`.                                                                 |
| Invalid update type                                                  | Explain valid update types: `name`, `description`, `project`. Ask the user to select one and retry.                                                                       |
| Invalid value format (name/description)                              | String must be non-empty. Ask the user to provide a non-empty string and retry.                                                                                           |
| Invalid project ID format                                            | Provide an example project ID format. Ask user to provide a valid project ID string and retry.                                                                            |
| Update failed                                                        | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Route filter not found (Step 6 verification)                         | Likely indexing delay. Ask user to retry verification shortly; do not make another write call.                                                                            |
| Route filter found but field not yet reflected (Step 6 verification) | Likely indexing delay. Ask user to retry verification shortly; do not make another write call.                                                                            |
| Authorization error                                                  | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited                                        | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)                                                | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error                                                | Return the tool error summary to the user, state that route filter update did not complete, and end the workflow.                                                         |