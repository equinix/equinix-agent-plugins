# Create a Route Aggregation Rule
Detailed steps for creating a new route aggregation rule.
Follow these steps when the user's intent is to create a new route aggregation rule.

## Objective
Create a new route aggregation rule with user-confirmed name, prefix, and route aggregation UUID.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Route aggregation UUID must be confirmed before any tool call is made.

## Definition of Done
- Route aggregation rule UUID is returned from the creation call.
- Route aggregation rule can be retrieved via `get_route_aggregation_rule` using route aggregation UUID and route aggregation rule UUID.
- Retrieved route aggregation rule state is `PROVISIONING` or `PROVISIONED`.
- Returned configuration matches user-confirmed name, prefix and route aggregation UUID (plus description if provided).

## Steps

### 1. Check Existing Route Aggregation Rule
If route aggregation UUID is missing, stop and request it from the user. Do not call any write operation until route aggregation UUID is confirmed.
Call `get_all_route_aggregation_rules` with route aggregation UUID to identify:
- Existing naming patterns in this route aggregation
- Existing route aggregation rules that may conflict with the requested name

If name is missing, propose a name based on existing naming patterns.

### 2. Gather Requirements
Collect, validate, and explicitly confirm from the user:
- **Prefix** — required
- **Name** — required
- **Description** — optional
- **Route aggregation UUID** — required

If required fields are missing or invalid, ask for corrected input and do not proceed to Step 3.

### 3. Confirm Before Creating
Present the full configuration to the user and require explicit confirmation before calling `create_route_aggregation_rule`:
- Name
- Description (if provided)
- Prefix
- Route aggregation UUID (explicitly confirm this is the intended route aggregation)

Require the user to confirm before proceeding to Step 4 (yes/no). The confirmation prompt must clearly state the action being performed.
- If the user confirms "yes", proceed to Step 4.
- If the user responds "no", do not proceed to Step 4. Inform the user that the operation has been canceled, and end the workflow. Wait for a new user request.

### 4. Create the Route Aggregation Rule
Call `create_route_aggregation_rule` once with the Step 3-confirmed name, prefix, and route aggregation UUID. Include description if provided.

Capture from response: route aggregation rule UUID, name, state, and creation timestamp.
On error, do not proceed to verification. Follow the Error Handling table.

### 5. Verify Provisioning
Only proceed if `create_route_aggregation_rule` returned successfully with a route aggregation rule UUID.
Call `get_route_aggregation_rule` filtered by the returned route aggregation rule UUID and route aggregation ID.

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the route aggregation rule is returned and whether state is `PROVISIONING` or `PROVISIONED`.
3. If the returned route aggregation rule UUID is found with state `PROVISIONING` or `PROVISIONED`, treat verification as successful and display route aggregation rule UUID, route aggregation UUID, name, state, and creation timestamp (plus description if present).
4. If not found after all retries, mark verification incomplete, report "creation submitted but not yet visible", and instruct the user to retry verification in 1-2 minutes.
5. If found but state is not `PROVISIONING` or `PROVISIONED`, report "creation submitted but state not yet transitioned", and instruct the user to retry verification in 1-2 minutes.

## Error Handling

| Error                                         | Action                                                                                                                                                                    |
|-----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Duplicate prefix (`EQ-3044404`)               | Inform the user that a route aggregation rule with this prefix already exists on the route aggregation. Do not retry or create another rule. Ask the user whether they want to update the existing rule instead. |
| Route aggregation rule not yet visible        | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Route aggregation rule state not transitioned | Instruct user to retry verification in 1-2 minutes; do not perform additional write calls in this workflow run.                                                           |
| Creation failed                               | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Search request failed (400)                   | Correct aggregation using supported properties and `=` operator, then retry once; do not proceed with write calls                                                         |
| Authorization error                           | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited                 | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)                         | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error                         | Return the tool error summary to the user, state that route aggregation rule creation did not complete, and end the workflow.                                             |
