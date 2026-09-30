---
name: manage-fabric-connection
description: Manages Equinix Fabric connection create, validate-before-create, update, and retry operations for EVPL_VC, EPL_VC, IP_VC, IA_VC, EVPLAN_VC, EPLAN_VC, port-to-port, port-to-network, port-to-AWS, FCR-to-port IP_VC, and IA_PROFILE Internet Access. Use when a user wants connection create, update name/bandwidth/emails, retry failed creation, or FCR endpoint planning. Excludes lookup/search/status inventory, service tokens, Fabric Network lifecycle, routing config, and VD lifecycle.
metadata:
  author: Fabric
  version: 1.0
  last_reviewed: 2026-08-24
  filter: '{"or":[{"and":["connections","write"]},{"and":["connections","read"]}]}'
---

## Objective

Manage Equinix Fabric connection resources through create, validate-before-create, update, and retry workflows using the currently exposed MCP tools.

Supported core tools:

* `create_connection`
* `update_connection`
* `retry_connection`
* `check_connection`

Supporting discovery tools may be used when available:

* `search_ports`
* `search_service_profiles`
* `search_prices`

This skill manages Fabric connections only. It does not manage Fabric Network resources, Cloud Router routing configuration, service token issuance, Virtual Device lifecycle, or destructive connection deletion.

## Immediate Boundary Rules

Use this skill for Fabric connection lifecycle requests such as:

* "Connect my SV port to AWS Direct Connect at 1G"
* "Create a colo-to-colo connection between DC and SV"
* "Rename this connection"
* "Increase my connection bandwidth"
* "Update notification emails for this connection"
* "Retry my failed connection"

Do not use this skill for:

* Creating, updating, deleting, or configuring Fabric Networks
* Creating or managing service tokens
* Issuing buyer-side tokens
* Configuring Cloud Router routing, BGP, routes, prefixes, or route filters
* Creating or managing Virtual Devices
* Deleting, removing, deprovisioning, or tearing down a Fabric connection through FSA

Correct boundary response patterns:

* Fabric Network request: "This is a Fabric Network lifecycle request, not a Fabric connection request. Use the Fabric Network management flow."
* Service token issuance: "Token issuance is handled by the service token management flow."
* Cloud Router routing: "Routing configuration is handled by the Cloud Router management flow."
* Delete connection: "I can’t delete Fabric connections through FSA. Connection deletion is destructive and must go through the Equinix Fabric portal or Equinix support."

Connection lookup, search, list, inventory, and status-only requests are out of scope for this skill. Route those requests to the `show-fabric-inventory` skill.

Examples outside this skill:

* "Find my connection by UUID"
* "List my Fabric connections"
* "Show connection status"
* "Search for connections in this project"
* "Check whether this connection exists"

Correct response pattern:

"Connection lookup, search, inventory, and status-only requests are handled by the `show-fabric-inventory` skill. The `manage-fabric-connection` skill only handles create, validate-before-create, update, and retry workflows."

## Supported Connection Patterns

| User request pattern | Typical connection type | Endpoint pattern                                                |
|---|---|-----------------------------------------------------------------|
| Port-to-port / colo-to-colo | `EVPL_VC` or `EPL_VC` | A-side `COLO` to  Z-side `COLO`                                 |
| Port-to-service-provider, including AWS Direct Connect | `EVPL_VC` or provider-specific type | A-side `COLO` to  Z-side `SP`                                   |
| Port-to-Fabric-Network | `EVPLAN_VC` or `EPLAN_VC` | A-side `COLO` to  Z-side `NETWORK`                              |
| FCR-to-port / Fabric Cloud Router endpoint connection | `IP_VC` | A-side `CLOUD_ROUTER` endpoint to Z-side `COLO` port            |
| Internet Access / IA profile connection | `IA_VC` | A-side `COLO` or `VD` endpoint to Z-side `SP` with `IA_PROFILE` |
| Existing token-based connection | token-supported type | Depends on token and endpoint                                   |

For Fabric Cloud Router, FCR, router-to-port, or FCR-to-port connection requests:

* Treat the request as a Fabric connection planning request when the user wants to connect an FCR endpoint to a port or another Fabric endpoint.
* Use `IP_VC` as the connection type for FCR-to-port examples when the A-side is a Fabric Cloud Router.
* Do not treat `CLOUD_ROUTER` as a connection type. `CLOUD_ROUTER` is an `accessPoint.type`.
* For local FCR connectivity, the FCR and target endpoint must be in the same metro.
* Do not configure routing, BGP, routes, route filters, prefixes, or Cloud Router lifecycle settings.
* Do not create the connection unless the required endpoint payload fields are known and complete.

For Internet Access connection requests:

* Use `IA_VC` when connecting a port or virtual device to an Internet Access service profile.
* The Z-side is an `SP` access point with profile type `IA_PROFILE`.

## Required Inputs

### Create Connection

Collect or infer and validate:

* Connection type
* Connection name
* Bandwidth in Mbps
* Project ID or account context
* Notification email
* A-side endpoint
* Z-side endpoint
* Link protocol and VLAN tags when required
* Provider details when Z-side is a service provider

Minimum create payload fields are:

* `type`
* `name`
* `bandwidth`
* `notifications`
* `aSide`
* `zSide`

Include `project` when the user provides a project ID.

For `COLO` port endpoints, collect:

* port UUID or resolvable port name
* metro when needed
* link protocol such as `DOT1Q`, `QINQ`, or untagged/EPL behavior
* VLAN tag when required

For service-provider endpoints, collect:

* service profile UUID or resolvable service profile name
* seller metro or metro code
* seller region when required
* authentication key when required
* bandwidth
* A-side port and VLAN details
* project and notification email

For Fabric Network endpoints, collect:

* network UUID or resolvable network name
* network type when known
* compatible port or endpoint details

Do not create if required endpoint details are missing.

For VLAN-based port connections, a concrete VLAN tag is required when the endpoint uses `DOT1Q`.

Phrases such as `available VLAN`, `any VLAN`, or `use any available VLAN` are not concrete VLAN values. Do not call `create_connection` with a missing VLAN tag unless a tool has returned a specific available VLAN value. Ask the user for the exact VLAN tag or use a discovery tool if one is available and returns a concrete value.

### Update Connection

Supported `update_type` values:

* `name`
* `bandwidth`
* `emails`
* `aws_keys`
* `migration_aside_port`
* `migration_aside_virtual_device`

Collect or infer:

* Target connection UUID, preferred
* Project/account context when needed
* Requested update type
* New value

Before updating:

* Require a target connection UUID.
* Validate that the requested update type is supported.
* Validate the new value format.
* Do not attempt inventory lookup or status lookup from this skill.
* If the user asks to verify the existing value before update, route lookup/status verification to `show-fabric-inventory`.
* Do not update unsupported fields.

### Retry Connection

Use `retry_connection` only for supported retry actions.

For this first skill version:

* Retry failed or stuck connection creation when the user asks to retry a failed connection create.
* Do not retry destructive deletion workflows unless deletion retry is explicitly approved for this skill.

Collect:

* Connection UUID
* Retry intent, such as failed creation retry

### Delete Connection Request

Connection delete, remove, deprovision, and tear-down operations are out of scope.

For delete requests:

* Recognize the request as destructive.
* Do not call create, update, retry, or delete operations.
* Do not ask for confirmation to proceed.
* Refuse deletion through FSA.
* Redirect the user to the Equinix Fabric portal or Equinix support.
* For connection lookup or status review before deletion, route the user to `show-fabric-inventory`.

## Steps

### 1. Determine Operation

Classify the request as one of:

* Create Fabric connection
* Validate or plan Fabric connection create
* Update Fabric connection
* Retry failed connection creation
* Delete connection refusal
* Out of scope

If the request is lookup, search, list, inventory, or status-only, exit this skill and route to `show-fabric-inventory`.

If the request is about Fabric Network lifecycle, service token issuance, Cloud Router routing config, or Virtual Device lifecycle, exit this skill and redirect.

### 2. Planning or Incomplete Create Flow

Use this flow when the user asks to create or plan a Fabric connection but required create inputs are missing.

Always return a final response. Never leave the response blank for planning or incomplete create requests.

Do not call `create_connection` when required create inputs are missing.

For incomplete create requests, respond with:

* Current state: `not ready to create`
* Connection pattern identified
* Known values from the user request
* Missing required values
* Next step: ask only for the missing values

For port-to-service-provider requests such as AWS Direct Connect, missing values may include:

* connection name
* project ID
* notification email
* A-side VLAN tag for Dot1Q ports
* service profile UUID
* seller metro or metro code
* seller region
* authentication key

For FCR-to-port requests, missing values may include:

* connection name
* project ID
* notification email
* bandwidth
* connection type or supported create payload type
* FCR endpoint details required by the API
* port VLAN tag only when required by the selected connection type and payload

For FCR-to-port planning responses, always include:

* Whether the request is treated as Fabric connection planning
* Metro rule: local FCR connectivity requires the FCR and port to be in the same metro
* Whether metro compatibility can be inferred from user-provided details
* Known values from the request
* Missing required create inputs
* Clear statement that routing, BGP, routes, and prefixes are not configured by this skill
* Clear statement that `create_connection` is not called until all required payload fields are known

### 3. Create Connection Flow

Use this flow when the user asks to create, provision, build, connect, or set up a Fabric connection.

1. Identify connection pattern and type.
2. Collect required inputs.
3. Resolve endpoint names to UUIDs when discovery tools are available.
4. Validate bandwidth, endpoint types, VLAN requirements, project, and notification email.
5. For service-provider connections, call `check_connection` when service profile UUID, seller region, and authentication key are provided.
6. Do not call `create_connection` until required fields are complete.
7. For `DOT1Q` endpoints, verify that each required side has a concrete `vlanTag`. Do not treat "available VLAN" or "any VLAN" as a valid VLAN tag.
8. Call `create_connection`.
9. Inspect the tool result.
10. If a UUID or accepted operation result is returned, report only tool-backed details.
11. If the operation fails or returns validation errors, surface the exact error and do not fabricate success.
12. Do not create related Fabric Networks, service tokens, Cloud Router routing config, or Virtual Devices.

Safe success wording:

* `Fabric connection create request accepted`
* `Fabric connection submitted`
* `The create_connection operation returned the following details`

Avoid saying `created successfully`, `active`, or `provisioned` unless returned by the tool.

### 4. Update Connection Flow

Use this flow when the user asks to rename, change bandwidth, update emails, update AWS keys, or migrate supported A-side endpoint fields.

1. Require the target connection UUID.
2. Validate that the requested update type is supported.
3. Validate the new value.
4. Do not call `search_connections`.
5. Call `update_connection` with the correct `update_type`.
6. Inspect the result.
7. If the result contains returned fields, change status, updated name, updated bandwidth, updated notifications, state, or operation status, report those values explicitly.
8. For rename updates, compare the requested new name with the returned name when available.
9. If the returned name matches the requested name, state that the rename was accepted or completed based on the returned tool result.
10. If the returned name does not match the requested name, state that the update was submitted but the returned name did not confirm the requested value.
11. If the tool result is empty, null, malformed, or uninspectable, do not fabricate success. State that `update_connection` was called but did not return usable confirmation.
12. Report the tool-backed update result or exact API error.

Update mappings:

| User intent | `update_type` |
|---|---|
| Rename connection | `name` |
| Change bandwidth | `bandwidth` |
| Update notification emails | `emails` |
| Update AWS access keys | `aws_keys` |
| Migrate A-side port | `migration_aside_port` |
| Migrate A-side virtual device | `migration_aside_virtual_device` |

### 5. Retry Connection Flow

Use this flow when the user asks to retry a failed or stuck connection creation.

1. Identify the connection UUID.
2. Confirm the retry intent is for connection creation.
3. Do not call `search_connections`.
4. Call `retry_connection` only for supported non-delete retry actions.
5. Report the tool-backed retry result or exact API error.

Do not retry deletion/deprovision workflows unless deletion retry is explicitly added to this skill scope.

### 6. Delete Connection Refusal Flow

Use this flow when the user asks to delete, remove, deprovision, tear down, or terminate a Fabric connection.

1. Recognize the request as destructive.
2. Acknowledge the target UUID/name when provided.
3. Do not call create, update, retry, or delete operations.
4. Do not ask for delete confirmation.
5. Refuse deletion through FSA.
6. Redirect to the Equinix Fabric portal or Equinix support.
7. Optionally offer to look up connection details first.

Correct response pattern:

"I can’t delete Fabric connection `<uuid-or-name>` through FSA. Connection deletion is destructive and must be performed through the Equinix Fabric portal or Equinix support. For connection lookup or status review before deletion, route the user to `show-fabric-inventory`."

## Validation Rules

* Do not invent UUIDs, VLANs, metros, project IDs, service profiles, authentication keys, emails, statuses, or API results.
* Use UUID as the authoritative identifier when available.
* Connection names are not unique identifiers.
* Do not block create only because a name may already exist.
* Do not create with incomplete required inputs.
* Do not update unsupported fields.
* Do not claim success unless a tool result supports it.
* Do not claim active/provisioned unless returned state/status says so.
* Preserve user-provided email addresses exactly.
* Treat auth keys, AWS keys, tokens, and secrets as sensitive. Do not echo them back unless necessary, and do not include them in examples unless explicitly approved.

## Error Handling

* Missing required field: ask only for the missing value.
* Ambiguous connection name: require a connection UUID or route lookup to `show-fabric-inventory`.
* Connection lookup or status request: route to `show-fabric-inventory`.
* Invalid bandwidth: explain the allowed value expected by the API/tool and ask for a valid Mbps value.
* VLAN conflict or invalid VLAN: surface the exact validation error and ask for another VLAN if needed.
* Unsupported endpoint type: explain supported connection endpoint patterns.
* Service-provider validation failure: surface the `check_connection` error and ask for corrected service profile, seller region, or authentication key.
* Unsupported update type: list supported update types.
* No-op update: say the connection already has the requested value and do not call update.
* Delete request: refuse deletion through FSA and redirect to portal/support.
* Tool unavailable or API failure: state the exact issue and do not fabricate results.

## Response Style

For planning or incomplete create responses, start with one explicit sentence: `Fabric connection planning completed. Current state: not ready to create.`

Include:

* Connection pattern identified
* Known values from the request
* Missing required values
* Whether `create_connection` was called: `no`
* Next step: ask only for the missing values

For FCR-to-port planning responses, explicitly state whether both endpoints are in the same metro when the prompt provides both metros.

Keep responses concise and operational.

For create responses, use bullets instead of wide tables. Include:

* Result
* UUID, if returned
* Name
* Type
* Bandwidth
* Project ID
* Notification email
* A-side endpoint summary
* Z-side endpoint summary
* State/status
* Equinix operation status
* Next step, when useful

For create failures, the result must say `create failed`, not `update failed`.

For update responses, do not return only a table. Start with one explicit result sentence.

Include:

* Result: `update completed`, `update accepted`, `no change needed`, `update failed`, or `confirmation unavailable`
* Target connection UUID
* Update type
* Previous value when known
* Requested new value
* Returned value when available
* Whether returned value matches requested value: `yes`, `no`, or `unknown`
* Change status when returned
* State/status when returned
* Next step when useful

For retry responses, include:

* Target connection UUID/name
* Retry action attempted
* Tool-backed result or exact error

For delete refusal responses, include:

* Target UUID/name when provided
* Clear refusal to delete through FSA
* Reason: destructive deletion is not supported through this skill
* Redirect to Equinix Fabric portal or Equinix support