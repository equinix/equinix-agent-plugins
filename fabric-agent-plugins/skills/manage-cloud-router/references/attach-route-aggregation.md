# Attach a Route Aggregation
Detailed steps for attaching a route aggregation to a connection.
Follow these steps when the user's intent is to attach a route aggregation to a connection.

## Objective
Attach a route aggregation to a connection with user-confirmed route aggregation UUID, connection UUID, and direction.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Connection UUID is known or can be gathered from the user.

## Definition of Done
- Route aggregation UUID and attachment status are returned from the call.
- Returned attachment status is `ATTACHING` or `ATTACHED`.
- Response includes direction and route aggregation UUID matching user-confirmed values.

## Steps

### 1. Validate required resource identifiers for the attachment operation
Verify whether the user has provided a valid connection UUID. 
- If connection UUID is missing or ambiguous, invoke the `manage-fabric-connection` skill to help the user search for and select a connection.
- From the connection response, capture the connection UUID. Explicitly confirm with the user that the selected connection is the intended target before proceeding.

Verify whether the user has provided a valid route aggregation UUID.
- If route aggregation UUID is missing or ambiguous, ask the user to provide one before continuing.
- If project ID is missing or ambiguous, ask the user to provide one before continuing.
- Call `search_route_aggregations` with: Route aggregation UUID, Project ID, Pagination: `offset: 0, limit: 10`
  - If no route aggregation is returned, ask the user to confirm project ID and route aggregation UUID are correct. Do not proceed to Step 2.
  - If multiple route aggregations match (unexpected), ask the user to select one by route aggregation UUID, then verify it is the intended target.

Once both connection UUID and route aggregation UUID are resolved, proceed to Step 2.

### 2. Confirm Before Attachment
Present the full configuration to the user and require explicit confirmation before calling `attach_route_aggregation`:
- Connection UUID (explicitly confirm this is the intended connection)
- Route aggregation UUID

Require the user to confirm before proceeding to Step 3 (yes/no). The confirmation prompt must clearly state the action being performed.
- If the user confirms "yes", proceed to Step 3.
- If the user responds "no", do not proceed to Step 3. Inform the user that the operation has been canceled, and end the workflow. Wait for a new user request.

### 3. Attach the Route Aggregation
Call `attach_route_aggregation` once with the Step 2-confirmed connection UUID and route aggregation UUID.

Capture from response: route aggregation UUID, attachment status, direction, and creation timestamp.
On error, do not proceed to verification. Follow the Error Handling table.

### 5. Verify Provisioning
Only proceed if `attach_route_aggregation` returned successfully with a route aggregation UUID.

Verify the response by comparing it against user-confirmed values:
- Route aggregation UUID must match
- Connection UUID must match
- Attachment status must be `ATTACHING` or `ATTACHED`

If all fields match:
- Treat verification as successful and display route aggregation UUID, connection UUID, direction, attachment status, and creation timestamp to the user.

If any field does not match or attachment status is neither `ATTACHING` nor `ATTACHED`:
- Report "attachment response received but verification criteria not met", and instruct user to retry verification in 1-2 minutes.

## Error Handling

| Error                              | Action                                                                                                                                                                    |
|------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Attachment state not yet reflected | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Attachment failed                  | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Authorization error                | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited      | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)              | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error              | Return the tool error summary to the user, state that route aggregation attachment did not complete, and end the workflow.                                                |