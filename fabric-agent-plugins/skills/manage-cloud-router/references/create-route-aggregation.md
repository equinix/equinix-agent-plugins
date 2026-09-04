# Create a Route Aggregation
Detailed steps for creating a new route aggregation.
Follow these steps when the user's intent is to create a new route aggregation.

## Objective
Create a new route aggregation with user-confirmed type, name, and project ID.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Project ID must be confirmed before any tool call is made.

## Definition of Done
- Route aggregation UUID is returned from the creation call.
- Route aggregation can be retrieved via `search_route_aggregations` using route aggregation UUID + project ID.
- Retrieved route aggregation state is `PROVISIONING` or `PROVISIONED`.
- Returned configuration matches user-confirmed type, name, and project (plus description if provided).

## Steps

### 1. Check Existing Route Aggregations
If project ID is missing, stop and request it from the user. Do not call any write operation until project ID is confirmed.
Call `search_route_aggregations` with project ID and default pagination (`offset: 0, limit: 10`) to identify:
- Existing naming patterns in this project
- Existing route aggregations that may conflict with the requested name

If name is missing, propose a name based on existing naming patterns.

### 2. Gather Requirements
Collect, validate, and explicitly confirm from the user:
- **Type** — must be exactly `BGP_IPv4_PREFIX_AGGREGATION` or `BGP_IPv6_PREFIX_AGGREGATION`; reject any other value and request corrected input
- **Name** — required
- **Description** — optional
- **Project ID** — required

If required fields are missing or invalid, ask for corrected input and do not proceed to Step 3.

### 3. Confirm Before Creating
Present the full configuration to the user and require explicit confirmation before calling `create_route_aggregation`:
- Type
- Name
- Description (if provided)
- Project ID (explicitly confirm this is the intended project/account scope)

Require the user to confirm before proceeding to Step 4 (yes/no). The confirmation prompt must clearly state the action being performed.
- If the user confirms "yes", proceed to Step 4.
- If the user responds "no", do not proceed to Step 4. Inform the user that the operation has been canceled, and end the workflow. Wait for a new user request.

### 4. Create the Route Aggregation
Call `create_route_aggregation` once with the Step 3-confirmed type, name, and project ID. Include description if provided.

Capture from response: route aggregation UUID, project ID, state, and creation timestamp.
On error, do not proceed to verification. Follow the Error Handling table.

### 5. Verify Provisioning
Only proceed if `create_route_aggregation` returned successfully with a route aggregation UUID.
Call `search_route_aggregations` filtered by the returned route aggregation UUID and project ID, using pagination (`offset: 0, limit: 10`).

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the route aggregation is returned and whether state is `PROVISIONING` or `PROVISIONED`.
3. If the returned route aggregation UUID is found with state `PROVISIONING` or `PROVISIONED`, treat verification as successful and display route aggregation UUID, project ID, name, state, and creation timestamp (plus description if present).
4. If not found after all retries, mark verification incomplete, report "creation submitted but not yet visible", and instruct the user to retry verification in 1-2 minutes.
5. If found but state is not `PROVISIONING` or `PROVISIONED`, report "creation submitted but state not yet transitioned", and instruct the user to retry verification in 1-2 minutes.

## Error Handling

| Error                                    | Action                                                                                                                                                                    |
|------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Invalid type                             | Show valid types: `BGP_IPv4_PREFIX_AGGREGATION`, `BGP_IPv6_PREFIX_AGGREGATION`                                                                                            |
| Route aggregation not yet visible        | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Route aggregation state not transitioned | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Creation failed                          | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Search request failed (400)              | Correct filter using supported properties and `=` operator, then retry once; do not proceed with write calls                                                              |
| Authorization error                      | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited            | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)                    | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error                    | Return the tool error summary to the user, state that route aggregation creation did not complete, and end the workflow.                                                  |
