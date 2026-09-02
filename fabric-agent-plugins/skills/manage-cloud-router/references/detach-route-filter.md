# Detach a Route Filter
Detailed steps for detaching a route filter from a connection.
Follow these steps when the user's intent is to detach a route filter from a connection.

## Objective
Detach a route filter from a connection with user-confirmed route filter UUID and connection UUID.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Router UUID is known or can be gathered from the user.

## Definition of Done
- Route filter UUID and attachment status are returned from the call.
- Returned attachment status is `DETACHING` or `DETACHED`.
- Response includes direction and route filter UUID matching user-confirmed values.

## Steps

### 1. Check Existing Attached Route Filter
If router UUID is missing, ask the user to provide it before continuing.

Call `search_route_filter_attachments_for_fcr` with router UUID, attachment status of `ATTACHED`, and pagination (`offset: 0, limit: 10`) to resolve the target attached route filter.

If no attached route filter is returned, do not proceed to Step 2. Ask the user to confirm router UUID. If the confirmed router UUID still returns no attached route filters, inform the user there is nothing to detach on this router and end the workflow.
If multiple attached route filters are returned, present the list to the user and ask them to select one specific entry by providing both the connection UUID and route filter UUID from that list. Validate that the selected pair matches the same item in the returned `href` values before continuing.
If exactly one attached route filter is returned, present it and ask user to confirm this is the target before proceeding to Step 2.

Once a single item is selected, capture its full details from the response (including connection UUID, route filter UUID, type, direction, attachment status, and timestamps), then proceed to Step 2.

### 2. Confirm Before Detachment
Present the full configuration of the selected item to the user:
- Connection UUID (explicitly confirm this is the intended connection)
- Route filter UUID
- Type
- Direction
- Attachment status
- Timestamp details from the selected item (if available)

Warn the user that this action detaches the route filter and may impact associated resources.

Require user confirmation before proceeding to Step 3. Present the selected connection UUID and route filter UUID in the confirmation prompt to leave no ambiguity about which attachment will be detached.

### 3. Detach the Route Filter
Call `detach_route_filter` once with connection UUID, and route filter UUID.

Capture from response: route filter UUID, type, attachment status, direction, and href.
On error, do not proceed to verification. Follow the Error Handling table.

### 4. Verify Detachment
Only proceed if `detach_route_filter` returned successfully with a route filter UUID.

Verify the response by comparing it against user-confirmed and selected values:
- Route filter UUID must match
- The returned href must contain the selected connection UUID and selected route filter UUID.
- Type should match the selected item from Step 1.
- Direction should match the selected item from Step 1.
- Attachment status must be `DETACHING` or `DETACHED`

If all fields match:
- Treat verification as successful and display route filter UUID, connection UUID, direction, and attachment status to the user.

If any field does not match or attachment status is neither `DETACHING` nor `DETACHED`:
- Report "detachment response received but verification criteria not met", and instruct user to retry verification by searching cloud router's route filter attachments again in 1-2 minutes.

## Error Handling

| Error                                        | Action                                                                                                                                                                    |
|----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Detachment state not yet reflected           | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Route filter not attached to this connection | Inform user the selected route filter is not attached to the selected connection; do not call detach again in this run.                                                   |
| Connection not found                         | Ask user to confirm connection UUID and router scope, then retry once corrected.                                                                                          |
| Detachment failed                            | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Authorization error                          | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited                | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)                        | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error                        | Return the tool error summary to the user, state that route filter detachment did not complete, and end the workflow.                                                     |