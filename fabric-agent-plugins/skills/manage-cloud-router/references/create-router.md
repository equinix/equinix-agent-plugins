# Create a Fabric Cloud Router
Detailed steps for provisioning a new Equinix Fabric Cloud Router (FCR).
Follow these steps when the user's intent is to create, provision, or deploy a new FCR.

## Objective
Create a new Equinix FCR with user-confirmed metro, package, name, project ID, and account number.

## Prerequisites
- User has IAM role `Fabric Cloud Router Manager` or `Fabric Manager`.
- Project ID and account number are known or can be gathered from the user.
- New billing accounts may require up to 24 hours before they are usable.

## Definition of Done
- Router UUID is returned from the creation call.
- Router can be retrieved via `search_routers` using router UUID + project ID.
- Retrieved router state is `PROVISIONING` or `PROVISIONED`.
- Returned configuration matches user-confirmed metro, package, name, and project.

## Steps

### 1. Check Existing Routers
If project ID is unknown, ask the user to choose one before continuing.
Call `search_routers` with project ID and default pagination (`offset: 0, limit: 10`) to understand:
- Current naming conventions
- Active metros

### 2. Gather Requirements
Collect and confirm from the user:
- **Metro** — e.g. `SV`, `NY`, `AM`
- **Package** — `LAB`, `BASIC`, `STANDARD`, or `ADVANCED` (validate in step 3)
- **Router name** — optional but recommended, suggest one based on naming patterns from step 1
- **Project ID** — required
- **Account number** — required; infer from step 1 results if not provided
- **Notification emails** — **required**; collect one or more email addresses for provisioning and operational alerts. If not provided in the request, ask the user to supply at least one email address before proceeding.

### 3. Validate Metro and Package
1. Call `list_metro` to confirm the metro exists.
2. If the user has not specified a package, or wants to compare options, call `list_router_packages` to fetch details — bandwidth, routing capabilities, and any limits — then present a comparison to help the user choose.
3. If the user already specified a package, call `list_router_packages` to list valid router packages, verify that the requested package exists, and confirm its details before proceeding.
4. Ask the user to confirm their final package choice before proceeding.

### 4. Confirm Before Creating
Present the full configuration to the user and require explicit confirmation before calling `create_router`:
- Router name
- Metro code
- Package code
- Account number
- Project ID (explicitly confirm this is the intended project/account scope)
- Notification emails (required — confirm at least one address is specified)
- Estimated capabilities based on the chosen package

Require the user to confirm before proceeding to Step 5 (yes/no). The confirmation prompt must clearly state the action being performed.
- If the user confirms "yes", proceed to Step 5.
- If the user responds "no", do not proceed to Step 5. Inform the user that the operation has been canceled, and end the workflow. Wait for a new user request.

### 5. Create the Router
Call `create_router` once with the Step 4-confirmed metro, package, name, project ID, account number, and notification emails. Notification emails are mandatory; do not call `create_router` if no email addresses have been collected.

Capture from response: router UUID, project ID, state, and creation timestamp.
On error, do not proceed to verification. Follow the Error Handling table.

### 6. Verify Provisioning
Only proceed if `create_router` returned successfully with a router UUID.
Call `search_routers` with returned router UUID and project ID using pagination (`offset: 0, limit: 10`).
If the response omits project ID, use the Step 4-confirmed project ID.

Because indexing/state propagation may be delayed, retry verification before failing:
1. Attempt up to 3 times using backoff delays of 5s, 10s and 15s. After attempt 3, stop retrying and report the final verification outcome.
2. On each attempt, check whether the router is returned and whether state is `PROVISIONING` or `PROVISIONED`.
3. If found with `PROVISIONING` or `PROVISIONED`, treat verification as successful and display router UUID, name, metro, package, state, and creation timestamp.
4. If not found after all retries, mark verification incomplete, report "creation submitted but not yet visible", and ask the user to retry verification shortly.

## Error Handling

| Error                         | Action                                                                                                                                                                    |
|-------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Metro not found               | Call `list_metro` and present valid options                                                                                                                               |
| Package code invalid          | Show the four valid codes: LAB, BASIC, STANDARD, ADVANCED                                                                                                                 |
| Creation failed               | Return the error message, provide remediation steps, and end the workflow. Do not proceed to verification or make additional tool calls.                                  |
| Authorization error           | Inform the user they need `Fabric Cloud Router Manager` or `Fabric Manager` role, stop the workflow, and wait for a new user request.                                     |
| Quota exceeded / rate limited | Do not retry in this workflow run. Explain the limit, suggest retrying later or adjusting package/scope, then end the workflow.                                           |
| API unavailable (5xx)         | Retry up to 3 times with backoff delays of 5s, 10s, and 15s. If all retries fail, report a transient service issue, advise the user to retry later, and end the workflow. |
| Unexpected tool error         | Return the tool error summary to the user, state that router creation did not complete, and end the workflow.                                                             |