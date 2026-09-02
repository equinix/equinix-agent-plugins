# Create a Route Filter Rule
Detailed steps for creating a new route filter rule.
Follow these steps when the user's intent is to create a new route filter rule.

## Objective
Create a new route filter rule with user-confirmed name, prefix, prefix match, and route filter UUID.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Route filter UUID must be confirmed before any tool call is made.

## Definition of Done
- Route filter rule UUID is returned from the creation call.
- Route filter rule can be retrieved via `get_route_filter_rule` using route filter UUID and route filter rule UUID.
- Retrieved route filter rule state is `PROVISIONING` or `PROVISIONED`.
- Returned configuration matches user-confirmed name, prefix, prefix match and route filter UUID (plus description if provided).

## Steps

### 1. Check Existing Route Filter Rule
If route filter UUID is missing, stop and request it from the user. Do not call any write operation until route filter UUID is confirmed.
Call `get_all_route_filter_rules` with route filter UUID to identify:
- Existing naming patterns in this route filter
- Existing route filter rules that may conflict with the requested name

If name is missing, propose a name based on existing naming patterns.

### 2. Gather Requirements
Collect, validate, and explicitly confirm from the user:
- **Prefix** — required
- **Prefix match** — must be exactly `exact` or `orlonger`; reject any other value and request corrected input
- **Name** — required
- **Description** — optional
- **Route filter UUID** — required

If required fields are missing or invalid, ask for corrected input and do not proceed to Step 3.

### 3. Confirm Before Creating
Present the full configuration to the user and require explicit confirmation before calling `create_route_filter_rule`:
- Name
- Description (if provided)
- Prefix
- Prefix match
- Route filter UUID (explicitly confirm this is the intended route filter)

Require the user to confirm before proceeding to Step 4 (yes/no). The confirmation prompt must clearly state the action being performed.
- If the user confirms "yes", proceed to Step 4.
- If the user responds "no", do not proceed to Step 4. Inform the user that the operation has been canceled, and end the workflow. Wait for a new user request.

### 4. Create the Route Filter Rule
Call `create_route_filter_rule` once with the Step 3-confirmed name, prefix, prefix match, and route filter UUID. Include description if provided.

Capture from response: route filter rule UUID, name, state, and creation timestamp.
On error, do not proceed to verification. Follow the Error Handling table.

### 5. Verify Provisioning
Only proceed if `create_route_filter_rule` returned successfully with a route filter rule UUID.
Call `get_route_filter_rule` filtered by the returned route filter rule UUID and route filter ID.

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the route filter rule is returned and whether state is `PROVISIONING` or `PROVISIONED`.
3. If the returned route filter rule UUID is found with state `PROVISIONING` or `PROVISIONED`, treat verification as successful and display route filter rule UUID, route filter UUID, name, state, and creation timestamp (plus description if present).
4. If not found after all retries, mark verification incomplete, report "creation submitted but not yet visible", and instruct the user to retry verification in 1-2 minutes.
5. If found but state is not `PROVISIONING` or `PROVISIONED`, report "creation submitted but state not yet transitioned", and instruct the user to retry verification in 1-2 minutes.

## Error Handling

| Error                                    | Action                                                                                                                                                                    |
|------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Invalid prefix match                     | Show valid types: `exact`, `orlonger`                                                                                                                                     |
| Duplicate prefix (`EQ-3044202`)          | Inform the user that a route filter rule with this prefix already exists on the route filter. Do not retry or create another rule. Ask the user whether they want to update the existing rule instead. |
| Route filter rule not yet visible        | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Route filter rule state not transitioned | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Creation failed                          | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Search request failed (400)              | Correct filter using supported properties and `=` operator, then retry once; do not proceed with write calls                                                              |
| Authorization error                      | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited            | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)                    | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error                    | Return the tool error summary to the user, state that route filter rule creation did not complete, and end the workflow.                                                  |
