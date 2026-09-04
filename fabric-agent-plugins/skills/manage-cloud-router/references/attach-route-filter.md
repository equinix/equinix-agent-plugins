# Attach a Route Filter
Detailed steps for attaching a route filter to a connection.
Follow these steps when the user's intent is to attach a route filter to a connection.

## Objective
Attach a route filter to a connection with user-confirmed route filter UUID, connection UUID, and direction.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Connection UUID is known or can be gathered from the user.

## Definition of Done
- Route filter UUID and attachment status are returned from the call.
- Returned attachment status is `ATTACHING` or `ATTACHED`.
- Response includes direction and route filter UUID matching user-confirmed values.

## Steps

### 1. Validate required resource identifiers for the attachment operation
Verify whether the user has provided a valid connection UUID. 
- If connection UUID is missing or ambiguous, invoke the `manage-fabric-connection` skill to help the user search for and select a connection.
- From the connection response, capture the connection UUID. Explicitly confirm with the user that the selected connection is the intended target before proceeding.

Verify whether the user has provided a valid route filter UUID.
- If route filter UUID is missing or ambiguous, ask the user to provide one before continuing.
- If project ID is missing or ambiguous, ask the user to provide one before continuing.
- Call `search_route_filters` with: Route filter UUID, Project ID, Pagination: `offset: 0, limit: 10`
  - If no route filter is returned, ask the user to confirm project ID and route filter UUID are correct. Do not proceed to Step 2.
  - If multiple route filters match (unexpected), ask the user to select one by route filter UUID, then verify it is the intended target.

Once both connection UUID and route filter UUID are resolved, proceed to Step 2.

### 2. Gather Requirements
Collect and confirm from the user:
- **Type** — `INBOUND` or `OUTBOUND`

### 3. Confirm Before Attachment
Present the full configuration to the user and require explicit confirmation before calling `attach_route_filter`:
- Type
- Connection UUID (explicitly confirm this is the intended connection)
- Route filter UUID

Require the user to confirm before proceeding to Step 4 (yes/no). The confirmation prompt must clearly state the action being performed.
- If the user confirms "yes", proceed to Step 4.
- If the user responds "no", do not proceed to Step 4. Inform the user that the operation has been canceled, and end the workflow. Wait for a new user request.

### 4. Attach the Route Filter
Call `attach_route_filter` once with the Step 3-confirmed type, connection UUID, and route filter UUID.

Capture from response: route filter UUID, type, attachment status, direction, and creation timestamp.
On error, do not proceed to verification. Follow the Error Handling table.

### 5. Verify Provisioning
Only proceed if `attach_route_filter` returned successfully with a route filter UUID.

Verify the response by comparing it against user-confirmed values:
- Route filter UUID must match
- Connection UUID must match
- Direction must match the user-confirmed Type
- Attachment status must be `ATTACHING` or `ATTACHED`

If all fields match:
- Treat verification as successful and display route filter UUID, connection UUID, direction, attachment status, and creation timestamp to the user.

If any field does not match or attachment status is neither `ATTACHING` nor `ATTACHED`:
- Report "attachment response received but verification criteria not met", and instruct user to retry verification in 1-2 minutes.

## Error Handling

| Error                              | Action                                                                                                                                                                    |
|------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Invalid type                       | Show valid types: `INBOUND`, `OUTBOUND`                                                                                                                                   |
| Attachment state not yet reflected | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Attachment failed                  | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Authorization error                | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited      | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)              | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error              | Return the tool error summary to the user, state that route filter attachment did not complete, and end the workflow.                                                     |