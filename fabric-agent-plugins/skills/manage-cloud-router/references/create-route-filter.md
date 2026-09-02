# Create a Route Filter
Detailed steps for creating a new route filter.
Follow these steps when the user's intent is to create a new route filter.

## Objective
Create a new route filter with user-confirmed type, name, and project ID.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Project ID is known or can be gathered from the user.

## Definition of Done
- Route filter UUID is returned from the creation call.
- Route filter can be retrieved via `search_route_filters` using route filter UUID + project ID.
- Retrieved route filter state is `PROVISIONING` or `PROVISIONED`.
- Returned configuration matches user-confirmed type, name, and project (plus description if provided).

## Steps

### 1. Check Existing Route Filters
If project ID is unknown, ask the user to provide one before continuing.
Call `search_route_filters` with project ID and default pagination (`offset: 0, limit: 10`) to understand:
- Current naming conventions
- Existing route filters in the selected project

Use results to suggest a name if the user has not specified one.

### 2. Gather Requirements
Collect and confirm from the user:
- **Type** — `BGP_IPv4_PREFIX_FILTER` or `BGP_IPv6_PREFIX_FILTER`
- **Name** — required
- **Description** — optional
- **Project ID** — required

### 3. Confirm Before Creating
Present the full configuration to the user and require explicit confirmation before calling `create_route_filter`:
- Type
- Name
- Description (if provided)
- Project ID (explicitly confirm this is the intended project/account scope)

Require the user to confirm before proceeding to Step 4 (yes/no). The confirmation prompt must clearly state the action being performed.
- If the user confirms "yes", proceed to Step 4.
- If the user responds "no", do not proceed to Step 4. Inform the user that the operation has been canceled, and end the workflow. Wait for a new user request.

### 4. Create the Route Filter
Call `create_route_filter` once with the Step 3-confirmed type, name, and project ID. Include description if provided.

Capture from response: route filter UUID, project ID, state, and creation timestamp.
On error, do not proceed to verification. Follow the Error Handling table.

### 5. Verify Provisioning
Only proceed if `create_route_filter` returned successfully with a route filter UUID.
Call `search_route_filters` with returned route filter UUID and project ID using pagination (`offset: 0, limit: 10`).
If the response omits project ID, use the Step 3-confirmed project ID.

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the route filter is returned and whether state is `PROVISIONING` or `PROVISIONED`.
3. If the returned route filter UUID is found with state `PROVISIONING` or `PROVISIONED`, treat verification as successful and display route filter UUID, name, state, and creation timestamp (plus description if present).
4. If not found after all retries, mark verification incomplete, report "creation submitted but not yet visible", and ask the user to retry verification shortly.
5. If found but state is not `PROVISIONING` or `PROVISIONED`, report "creation submitted but state not yet transitioned", and ask the user to retry verification shortly.

## Error Handling

| Error                               | Action                                                                                                                                                                    |
|-------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Invalid type                        | Show valid types: `BGP_IPv4_PREFIX_FILTER`, `BGP_IPv6_PREFIX_FILTER`                                                                                                      |
| Route filter not yet visible        | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Route filter state not transitioned | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Creation failed                     | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Authorization error                 | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited       | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)               | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error               | Return the tool error summary to the user, state that route filter creation did not complete, and end the workflow.                                                       |