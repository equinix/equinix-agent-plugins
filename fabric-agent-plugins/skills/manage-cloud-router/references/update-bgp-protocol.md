# Update Routing Protocol
Detailed steps for updating an existing routing protocol on a connection.
Follow these steps when the user's intent is to enable or disable BGP IPv4/IPv6 on an existing connection.

## Objective
Update an existing BGP routing protocol by toggling IPv4 and/or IPv6 BGP enabled flags.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Connection UUID is known or can be gathered from the user.
- Routing protocol UUID is known or can be resolved from the selected connection.

## Definition of Done
- `update_routing_protocol` succeeds with no API errors.
- Routing protocol can be retrieved via `list_routing_protocols` using connection UUID.
- Retrieved routing protocol reflects user-confirmed values for `bgpIpv4.enabled` and/or `bgpIpv6.enabled`.

## Constraints
- Update API supports JSON Patch operations only.
- Operation must use JSON Patch with `op: "replace"` for enabled-flag toggles.
- Allowed paths are only:
  - `/bgpIpv4/enabled`
  - `/bgpIpv6/enabled`
- Value must be boolean (`true` or `false`).
- Do not attempt to patch any other field (for example ASN, MED, auth key, BFD, timers, or peer IPs).

## Steps

### 1. Check Existing Connection
If connection UUID is unknown, ask the user to provide one before continuing.
Call `search_connections` with connection UUID and pagination (`offset: 0, limit: 10`) to resolve the target connection.

If no connection is returned, do not call `update_routing_protocol`. Ask the user to confirm connection UUID.
If multiple connections match, ask the user to select one by UUID before continuing.

### 2. Resolve Target Routing Protocol
If routing protocol UUID is unknown, call `list_routing_protocols` with connection UUID and pagination (`offset: 0, limit: 10`), then present candidates and ask the user to choose one by UUID.
If routing protocol UUID is provided, call `list_routing_protocols` and verify it exists under the selected connection.

If no matching routing protocol is found, do not call `update_routing_protocol`.

### 3. Gather Patch Intent
Ask the user which flag(s) to change:
- `bgpIpv4.enabled` (`true` or `false`)
- `bgpIpv6.enabled` (`true` or `false`)

Build JSON Patch operations using only allowed paths:
- `{"op":"replace","path":"/bgpIpv4/enabled","value":<boolean>}`
- `{"op":"replace","path":"/bgpIpv6/enabled","value":<boolean>}`

If the user asks to change unsupported fields, explain this tool only updates BGP enabled flags.
If no valid flag changes are selected, do not call `update_routing_protocol`.

### 4. Confirm Before Updating
Present a review summary of:
- Connection UUID
- Routing protocol UUID
- JSON Patch operations to be sent

Require the user to confirm before proceeding to Step 5 (yes/no). The confirmation prompt must explicitly include the selected connection UUID and routing protocol UUID, and clearly state the action being performed.
- If the user confirms "yes", proceed to Step 5.
- If the user responds "no", do not proceed to Step 5. Inform the user that the operation has been canceled. End the workflow and wait for a new user request.

### 5. Update the Routing Protocol
Call `update_routing_protocol` once with:
- `connection_uuid`
- `routing_protocol_uuid`
- `operations` (JSON Patch array)

Capture from response: routing protocol UUID, connection UUID, state, and updated `bgpIpv4.enabled` / `bgpIpv6.enabled` values.
On error, do not proceed to verification. Follow the Error Handling table.

### 6. Verify Update
Only proceed if `update_routing_protocol` returned successfully.
Call `list_routing_protocols` with connection UUID and pagination (`offset: 0, limit: 10`), then identify the entry matching the target routing protocol UUID.

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the matched routing protocol is returned and whether requested enabled-flag changes are reflected.
3. If found and reflected, treat verification as successful and display routing protocol UUID, connection UUID, and resulting enabled flags.
4. If not found after all retries, mark verification incomplete, report "update submitted but not yet visible", and ask the user to retry verification shortly.
5. If found but requested enabled flags are not updated after all retries, report "update submitted but changes are not yet reflected", and ask the user to retry verification shortly.

## Error Handling

| Error                                  | Action                                                                                                                                                                    |
|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Connection not found                   | Do not update; ask user to confirm connection UUID, then re-run `search_connections`                                                                                      |
| Routing protocol not found             | Do not update; ask user to confirm routing protocol UUID/connection UUID, then re-run lookup                                                                              |
| Invalid patch operation/path/value     | Explain valid paths (`/bgpIpv4/enabled`, `/bgpIpv6/enabled`) and require boolean values                                                                                   |
| Unsupported update request             | Inform user only BGP enabled flags can be updated via this API                                                                                                            |
| Routing protocol not yet visible       | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Routing protocol changes not reflected | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Update failed                          | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Authorization error                    | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited          | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)                  | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error                  | Return the tool error summary to the user, state that update did not complete, and end the workflow.                                                                      |