# Configure Routing Protocol
Detailed steps for creating a new routing protocol for a connection.
Follow these steps when the user's intent is to create, attach, or configure a new routing protocol on a connection.

## Objective
Create a routing protocol on a user-selected connection using user-confirmed protocol type (`DIRECT` or `BGP`) and type-specific settings.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Connection UUID is known or can be gathered from the user.

## Definition of Done
- Routing protocol UUID is returned from the creation call.
- Routing protocol can be retrieved via `list_routing_protocols` using connection UUID.
- Retrieved routing protocol state is `PROVISIONING` or `PROVISIONED`.
- Retrieved configuration matches all user-confirmed fields for the selected type.

## Steps

### 1. Check Existing Connection
If connection UUID is unknown, ask the user to choose one before continuing.
Call `search_connections` with connection UUID and default pagination (`offset: 0, limit: 10`) to resolve the target connection.

If no connection is returned, do not call `create_routing_protocol`. Ask the user to confirm connection UUID.
If multiple connections match, ask the user to select one by UUID before continuing.

### 2. Gather Requirements
Collect and confirm protocol `type` (`DIRECT` or `BGP`).
Gather all required fields for the selected type per tool schema.
Do not continue until all required fields are present and user-confirmed.
Do not mix `DIRECT` and `BGP` sections in one request.

### 3. Confirm Before Creating
Present a review summary of the exact values gathered in Step 2 (the fields that will be sent in the request body), including:
- Connection UUID
- Protocol type (`DIRECT` or `BGP`)
- All type-specific fields collected for the selected type

Require the user to confirm before proceeding to Step 4 (yes/no). The confirmation prompt must clearly state that a new route protocol will be configured.
- If the user confirms "yes", proceed to Step 4.
- If the user responds "no", do not proceed to Step 4. Inform the user that the operation has been canceled, and end the workflow. Wait for a new user request.

### 4. Create the Routing Protocol
Call `create_routing_protocol` once with the Step 3-confirmed values.

Capture from response: routing protocol UUID, connection UUID, state, and creation timestamp.
On error, do not proceed to verification. Follow the Error Handling table.

### 5. Verify Provisioning
Only proceed if `create_routing_protocol` returned successfully with a routing protocol UUID.
Call `list_routing_protocols` with connection UUID using pagination (`offset: 0, limit: 10`), then identify the entry matching the routing protocol UUID returned by `create_routing_protocol`.

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the matched routing protocol is returned and whether state is `PROVISIONING` or `PROVISIONED`.
3. If the returned routing protocol UUID is found with state `PROVISIONING` or `PROVISIONED`, treat verification as successful and display routing protocol UUID, connection UUID, type, state, creation timestamp, and related IP fields.
4. If not found after all retries, mark verification incomplete, report "creation submitted but not yet visible", and ask the user to retry verification shortly.
5. If found but state is not `PROVISIONING` or `PROVISIONED`, report "creation submitted but state not yet transitioned", and ask the user to retry verification shortly.

## Error Handling

| Error                                   | Action                                                                                                           |
|-----------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Connection not found                    | Do not create; ask user to confirm connection UUID and re-run `search_connections`                               |
| Invalid protocol fields                 | Explain validation error, request corrected values, then retry create                                            |
| Routing protocol not yet visible        | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.  |
| Routing protocol state not transitioned | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.  |
| Creation failed                         | Surface the error message with remediation steps                                                                 |
| Authorization error                     | Inform user they need `Fabric Cloud Router Manager` or `Fabric Manager` role                                     |
| Quota exceeded / rate limited           | Do not retry immediately, explain limit and ask user to retry later or adjust package/scope                      |
| API unavailable (5xx)                   | Retry with backoff, if still failing, report transient service issue and suggest retry                           |