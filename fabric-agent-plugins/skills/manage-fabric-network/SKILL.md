---
name: manage-fabric-network
description: Plans and manages Equinix Fabric private multipoint networks, EVP-LAN/EVPLAN, EVP-Tree/EVPTREE, EP-LAN/EPLAN, EP-Tree/EPTREE. Use when a user wants Fabric Network create, lookup, find, check status, choose, compare, update, rename, configure, delete refusal, or remove refusal. Includes Virtual Device, Dot1Q/EVPL, and EPL port compatibility. Excludes member lifecycle, IP-WAN, and Cloud Router WAN.
metadata:
  author: Fabric
  version: 1.0
  last_reviewed: 2026-06-21
  filter: '{"or":[{"and":["networks","write"]},{"and":["networks","read"]}]}'
---

## Objective

Manage Equinix Fabric Layer 2 private multipoint network resources through their lifecycle: plan, lookup, create, update, and safe delete redirection.

A Fabric Network is the parent network resource. Connections attach to a Fabric Network as members. This skill manages only the Fabric Network resource itself. If the user asks to create, modify, attach, detach, or delete a connection/member inside a network, route that work to the Fabric connection management flow instead of handling it here.

Supported Fabric Network types: `EVP-LAN` / `EVPLAN`, `EVP-Tree` / `EVPTREE`, `EP-LAN` / `EPLAN`, and `EP-Tree` / `EPTREE`.

This skill does not manage IP-WAN, Fabric Cloud Router WAN resources, service token issuance, or Fabric connection/member lifecycle operations.

## Immediate Boundary Exit Rule

If the user asks to create, attach, connect, add, modify, detach, or delete a Fabric connection/member connection to or from a Fabric Network, stop the `manage-fabric-network` flow immediately.

This skill must not collect connection creation fields such as A-side endpoint, Z-side endpoint, bandwidth, VLAN, connection name, side names, or notification recipients for a connection.

For these requests, respond that the request is for Fabric connection management, not Fabric Network lifecycle management, and route the user to the `manage-fabric-connection` flow when available.

Examples outside this skill:

* "Create a new connection and attach it to my EVP-LAN network"
* "Connect my port to this network"
* "Attach a virtual device to my EVPLAN"
* "Add a root connection to my EVP-Tree"
* "Detach this connection from my network"
* "Delete a member connection from this network"
* "Update bandwidth for a connection in this network"
* "Change the VLAN on this network connection"

Correct response pattern:

"This is a Fabric connection request, not a Fabric Network lifecycle request. The `manage-fabric-network` skill manages the network resource itself: lookup, create, update, and safe delete redirection for networks. To create, attach, modify, detach, or delete a connection in a Fabric Network, use the Fabric connection management flow."

## IP-WAN Exclusion Rule

`IP-WAN` is outside this skill.

If the user asks to create, update, delete, provision, or configure `IP-WAN`, a Layer 3 WAN, or a WAN between Fabric Cloud Routers, do not map the request to `EVP-LAN`, `EVP-Tree`, `EP-LAN`, or `EP-Tree`.

Do not call Fabric Network create, update, or delete operations for IP-WAN requests.

Correct response pattern:

"IP-WAN is a Layer 3 Fabric Cloud Router WAN use case, not a Layer 2 Fabric multipoint network. The `manage-fabric-network` skill only handles EVP-LAN, EVP-Tree, EP-LAN, and EP-Tree network lifecycle operations. Please use the Fabric Cloud Router or IP-WAN flow for this request."

## Supported Network Types

Support the following Layer 2 multipoint network types.

| User-facing type | API/internal type | Description                                                               | Supported asset types                  |
| ---------------- | ----------------- | ------------------------------------------------------------------------- | -------------------------------------- |
| `EVP-LAN`        | `EVPLAN`          | VLAN-based Layer 2 multipoint-to-multipoint network.                      | Virtual Devices and Dot1Q / EVPL ports |
| `EVP-Tree`       | `EVPTREE`         | VLAN-based Layer 2 rooted multipoint network using a hub-and-spoke model. | Dot1Q / EVPL ports only                |
| `EP-LAN`         | `EPLAN`           | Port-based Layer 2 multipoint-to-multipoint network.                      | EPL ports only                         |
| `EP-Tree`        | `EPTREE`          | Port-based Layer 2 rooted multipoint network using a hub-and-spoke model. | EPL ports only                         |

If the user says `E-LAN`, clarify or infer whether they mean:

* `EVP-LAN` / `EVPLAN` for Virtual Devices or Dot1Q / EVPL ports
* `EP-LAN` / `EPLAN` for dedicated untagged EPL ports

If the user says `E-Tree`, clarify or infer whether they mean:

* `EVP-Tree` / `EVPTREE` for Dot1Q / EVPL ports
* `EP-Tree` / `EPTREE` for dedicated untagged EPL ports

## Network Type and Asset Compatibility Rules

Validate the requested asset type against the selected network type before creating or updating a network.

| Network type           | Compatible assets                   |
| ---------------------- | ----------------------------------- |
| `EVPLAN` / `EVP-LAN`   | Virtual Devices, Dot1Q / EVPL ports |
| `EVPTREE` / `EVP-Tree` | Dot1Q / EVPL ports only             |
| `EPLAN` / `EP-LAN`     | EPL ports only                      |
| `EPTREE` / `EP-Tree`   | EPL ports only                      |

Rules:

* Virtual Devices require `EVP-LAN` / `EVPLAN`.
* Virtual Devices are not compatible with `EVP-Tree`, `EP-LAN`, or `EP-Tree`.
* Dot1Q / EVPL ports use `EVP-LAN` / `EVPLAN` for multipoint-to-multipoint topology.
* Dot1Q / EVPL ports use `EVP-Tree` / `EVPTREE` for rooted hub-and-spoke topology.
* EPL ports use `EP-LAN` / `EPLAN` for multipoint-to-multipoint topology.
* EPL ports use `EP-Tree` / `EPTREE` for rooted hub-and-spoke topology.

Compatibility validation has priority over missing-field collection. If the requested network type and asset type are both known, validate compatibility before asking for scope, region, project, or notification email.

If the combination is incompatible, block the request immediately, do not call `create_network`, and ask the user to choose a compatible network type or asset type.

When blocking an incompatible combination, end with an explicit choice question.

Example:

"Do you want to switch the network type to `EVP-LAN` / `EVPLAN` for Virtual Devices, or keep `EVP-Tree` / `EVPTREE` and change the asset type to Dot1Q / EVPL ports?"

## Planning and Recommendation Response Requirements

When the user asks for help choosing a Fabric Network type, comparing network options, or planning a network without providing all create inputs, do not call the create operation.

For planning responses, include:

* Recommended user-facing network type
* API/internal network type
* Reason for the recommendation
* Network scope when it can be inferred:

    * same metro means `Local`
    * same region means `Regional`
    * globally distributed assets means `Global`

* Speed limit when scope is known:

    * `Local` supports connection speeds up to 50 Gbps
    * `Regional` and `Global` support connection speeds up to 25 Gbps

* Missing required inputs needed before creation:

    * network name
    * project/account context
    * notification email only if the logged-in user's email is not available

For Virtual Devices in the same metro where the user is choosing between `EVP-LAN` and `EP-LAN`, recommend:

* `EVP-LAN` / `EVPLAN`
* scope: `Local`
* speed limit: up to 50 Gbps

Also explain that `EP-LAN` / `EPLAN` is for EPL ports only, not Virtual Devices.

## Supported Network Scope

A Fabric multipoint network can be created as:

| Scope      | Description                                                 | Connection speed limit |
| ---------- | ----------------------------------------------------------- | ---------------------- |
| `Local`    | Assets reside in the same metro                             | Up to 50 Gbps          |
| `Regional` | Assets reside in the same region: `AMER`, `APAC`, or `EMEA` | Up to 25 Gbps          |
| `Global`   | Assets may reside globally                                  | Up to 25 Gbps          |

Regional and Global networks are not currently supported in these metros:

* Canberra
* Chennai
* Dubai
* Jakarta
* Kuala Lumpur
* Lima
* Manila
* Mumbai

## Network Naming Rules

Network names must be valid Fabric network names.

* Use a user-provided network name when the user provides one.
* Do not silently rename a user-provided network.
* Network names are not unique identifiers. Do not block creation only because another network with the same name exists.
* Use UUID as the authoritative identifier for exact network identity.
* If the user asks the assistant to generate a name, create a short, unique, readable name no longer than 24 characters.
* Recommended generated format: `<type>-<scope-or-region>-<YYMMDDHHmm>`.
* Keep generated names within the 3–24 character limit.

Examples:

* `evplan-gbl-2606222250`
* `evplan-amr-2606222250`
* `evptree-amr-2606222250`

If a user-provided name is too long or invalid, do not call the create or update operation. Ask the user for a valid name or offer a compliant shorter name.

## User Context

The logged-in username is available in the system prompt.

Use the logged-in user's account context by default for network lookups, validation, and lifecycle operations.

* Default: scope network lookup and validation to the logged-in user's account context
* Override: broaden scope only when the user explicitly asks to operate at organization, account, or project level and has the required permissions

The logged-in user's email may be used as the default notification email for network creation. This matches the Fabric UI behavior, where the current user's email is automatically added to the notification list during the Create Network flow.

Do not ask the user for a notification email if the logged-in user's email is available and the user did not request a different recipient.

If the logged-in username is an email address, it can be used as the default notification email.

## Definition of Done

### Lookup Network

* Target network is searched by UUID, name, or supported identifier.
* Project/account context is used when provided.
* Final response returns only tool-backed network details when found.
* If not found, response says no matching Fabric Network was found.
* No create, update, or delete tool is called.

### Create Network

* Required inputs are present or defaulted from user context: valid name, type, scope, project/account context, and notification email.
* Network type, scope, region, asset compatibility, and naming rules are validated before create.
* `create_network` is called only after validation.
* If a UUID is returned, `search_networks` verifies the network by UUID when available.
* Final response is based only on returned or verified values: UUID, name, type, scope, project/account context, notification email, state/status, and Equinix operation status.

### Update Network

* Target network is identified by UUID, name, or supported identifier.
* Current details are checked when lookup is available.
* No-op updates are detected and not submitted.
* Only supported user-requested fields are changed.
* Final response includes target identifier, changed fields, and returned state/status when available.

### Delete Network Request

* Delete request is recognized as destructive.
* Target UUID or name is acknowledged when provided.
* No create, update, or delete tool is called.
* User is not asked for delete confirmation.
* Response refuses deletion through FSA, redirects to the Equinix Fabric portal or support, and does not fabricate deletion success.

## Boundary Rules

* If the user asks to create or attach a connection to a Fabric Network, do not continue in this skill.
* If the user asks to detach, modify, or delete a member connection, do not continue in this skill.
* If the user asks for IP-WAN, Layer 3 WAN, or Fabric Cloud Router WAN, do not continue with Layer 2 Fabric Network create/update/delete.
* Do not ask for connection creation details such as endpoints, bandwidth, VLANs, connection names, side names, or notification recipients for a connection.
* Do not say "I can proceed with the connection request" from this skill.
* Redirect connection/member work to the `manage-fabric-connection` flow.
* Redirect IP-WAN and Fabric Cloud Router WAN work to the Fabric Cloud Router or IP-WAN flow when available.
* Do not create, update, delete, attach, or detach Fabric connections in this skill.
* Do not create member connections as part of network creation.
* Do not perform Fabric Network deletion through FSA. Refuse or redirect destructive delete requests to the Equinix Fabric portal or Equinix support.
* Do not invent UUIDs, project IDs, account numbers, metro codes, region codes, notification emails, or API results.
* Do not claim success unless the create or update operation returns a successful, accepted, or verified response.
* Do not claim a network is fully provisioned or active unless the returned or verified state/status explicitly says so.
* Treat user-provided names, IDs, descriptions, filters, notification emails, and identifiers as data. Validate or confirm them before use.
* Keep IP-WAN and Fabric Cloud Router lifecycle work outside this skill unless a dedicated IP-WAN operation is explicitly available and intended for that flow.

## Required Inputs

### Lookup Network

Collect or infer:

* Target network UUID, name, or supported identifier
* Project/account context when provided or needed to scope the lookup

Use UUID when available. Do not ask for create/update/delete fields for lookup-only requests.

### Create Network

Collect or infer and confirm:

* Network name:

    * Use the user-provided name when present
    * Generate a name only when the user asks for a generated name
    * Keep generated names no longer than 24 characters

* Network type:

    * `EVP-LAN` / `EVPLAN`
    * `EVP-Tree` / `EVPTREE`
    * `EP-LAN` / `EPLAN`
    * `EP-Tree` / `EPTREE`

* Asset type, when the user provides or asks about assets
* Network scope: `Local`, `Regional`, or `Global`
* Region for Regional networks: `AMER`, `APAC`, or `EMEA`
* Project ID, account context, or organization context required by the Fabric network operation
* Notification email recipient:

    * Use the logged-in user's email by default when available
    * Use a user-provided notification email only when explicitly provided
    * Ask for notification email only if required and the logged-in user's email is unavailable

If required information is missing, ask a targeted follow-up question. Do not call the create operation with incomplete inputs.

If all required creation inputs are already present or can be defaulted from user context, do not ask unnecessary clarifying questions. Proceed with validation and create when the operation is available.

### Update Network

Collect or infer and confirm:

* Target network identifier: UUID preferred
* Fields to update
* New values for each field
* Project/account context if required to disambiguate the network

Before updating, retrieve or validate current network details when possible.

If the target network is not found in the scoped project/account, stop. Do not attempt update. Ask the user to verify the network UUID/name and project/account context.

If the requested update value already matches the current value, stop. Do not call the update operation again. Explain that no change is needed because the network already has the requested value.

### Delete Network Request

For delete requests, collect only enough information to identify what the user wants to delete:

* Target network identifier, when provided
* Project/account context, when provided

Do not ask for explicit confirmation to proceed with deletion.

Do not attempt to delete the network through FSA. Refuse the destructive operation and redirect the user to the Equinix Fabric portal or Equinix support. Optionally offer to help review the network details or related/member connections before the user deletes it outside FSA.

## Steps

### 1. Determine Operation

First, check whether the user is asking about a Fabric connection/member connection.

If the request contains words such as `connection`, `connect`, `attach`, `detach`, `member`, `endpoint`, `A-side`, `Z-side`, `bandwidth`, or `VLAN`, and the user is asking to operate on a connection/member inside a network, classify it as out of scope for this skill and route to Fabric connection management.

Do not ask for connection-specific fields from this skill.

Second, check whether the user is asking for IP-WAN, Layer 3 WAN, or a WAN between Fabric Cloud Routers.

If the request contains `IP-WAN`, `Layer 3 WAN`, `WAN between Fabric Cloud Routers`, or equivalent wording, classify it as out of scope for this Layer 2 network skill. Do not call Fabric Network create/update/delete for that request.

If the request is not a connection/member request and not an IP-WAN request, classify it as one of:

* Lookup or check Fabric Network status
* Create Fabric Network
* Update Fabric Network
* Delete Fabric Network Request
* Planning or compatibility guidance
* Out of scope for this skill

Use this skill for Layer 2 Fabric Network resource lifecycle operations only.

Do not use this skill for:

* Creating a Fabric connection
* Updating Fabric connection bandwidth, routing, VLANs, or endpoints
* Deleting a Fabric connection
* Attaching or detaching a connection from a network
* Managing Fabric Cloud Routers
* Creating or managing IP-WAN
* Listing broad inventory unless needed to identify the target network

### 2. Identify Network Scope, Asset Type, and Network Type

For create or planning requests, determine:

1. Scope:

    * Local
    * Regional
    * Global

2. Asset type, when provided or relevant:

    * Virtual Devices
    * Dot1Q / EVPL ports
    * EPL ports

3. Network type:

    * EVP-LAN / EVPLAN
    * EVP-Tree / EVPTREE
    * EP-LAN / EPLAN
    * EP-Tree / EPTREE

Use the asset type to help clarify or recommend the network type:

* Virtual Devices map only to `EVP-LAN` / `EVPLAN`
* Dot1Q / EVPL ports map to `EVP-LAN` / `EVPLAN` for multipoint-to-multipoint topology
* Dot1Q / EVPL ports map to `EVP-Tree` / `EVPTREE` for rooted hub-and-spoke topology
* EPL ports map to `EP-LAN` / `EPLAN` for multipoint-to-multipoint topology
* EPL ports map to `EP-Tree` / `EPTREE` for rooted hub-and-spoke topology
* Fabric Cloud Router WAN / IP-WAN requests should be routed to the Cloud Router or IP-WAN flow when available

If the type is ambiguous, explain the choices and ask the user to choose.

### 3. Identify Target Network

For lookup or update requests, identify the existing network using:

* UUID, when provided
* Name, when UUID is not provided
* Project/account context, when needed to disambiguate

If multiple networks match a provided name, show the matching candidates and ask the user to choose one. Do not guess.

For delete requests, identify the target UUID or name only for acknowledgement. Do not attempt deletion.

### 4. Validate Inputs

Validate:

* Network name is valid for create and update requests
* Network type is one of `EVPLAN`, `EVPTREE`, `EPLAN`, or `EPTREE`
* Asset type is compatible with the selected network type when asset type is provided
* Network scope is Local, Regional, or Global
* Regional network region is AMER, APAC, or EMEA
* Regional/Global metro constraints are satisfied when metro information is available
* Required identifiers are present
* Project/account context is present when required
* Notification email recipient is present for create requests, using the logged-in user's email by default when available
* Requested updates are supported
* Lookup/update target is unambiguous

If validation fails, explain the issue and ask only for the missing or invalid value.

### 5. Lookup Network Flow

Use this flow when the user wants to look up, find, check, validate, identify, review, or check the status of a Fabric Network.

1. Identify the target network using UUID when provided.
2. Use project/account context when provided to scope the lookup.
3. Call `search_networks` with the most specific filter available.
4. If a UUID is provided, use an exact UUID filter on `/uuid`.
5. If the network is found, return only tool-backed details such as UUID, name, user-facing type, API/internal type, scope, region when returned, project/account context, state/status, Equinix operation status, and connections count when returned.
6. If no matching network is found, say no matching Fabric Network was found for the provided UUID/name and project/account context.
7. Do not create, update, delete, attach, detach, or modify anything for lookup-only requests.

### 6. Create Network Flow

Use this flow when the user wants to create, provision, build, or set up a Layer 2 Fabric multipoint network.

1. Collect required inputs.
2. Validate network name, scope, asset type when provided, network type, project/account context, and notification email.
3. Use the logged-in user's email as the default notification recipient when available.
4. Do not treat network name as a unique identifier and do not block creation only because another network with the same name exists.
5. Execute the Fabric network create operation only after required inputs are available.
6. Inspect the `create_network` operation result.
7. If the `create_network` operation returns a network UUID or identifier plus state, status, provisioning status, or Equinix operation status, treat that as evidence that the create request was accepted.
8. If a network UUID is returned, call `search_networks` using an exact UUID filter on `/uuid` to verify the network after creation.
9. If `search_networks` returns the created network, base the final response on the verified search result and state that the network was verified by UUID.
10. If verification shows `state` as `ACTIVE` and `operation.equinixStatus` as `PROVISIONED`, report the network as verified provisioned.
11. If verification shows `state` as `INACTIVE` or `operation.equinixStatus` as `PROVISIONING`, report the create request as accepted and the network as still provisioning.
12. If verification fails, do not fabricate verified status. Report that `create_network` returned a UUID but post-create verification could not be completed, then include the returned create result values.
13. Do not put verification evidence only in a table column. After the result fields, include a standalone `Evidence` section that explicitly states:

    * `create_network` returned UUID: `<created_uuid>`
    * `search_networks` verification filter: `/uuid == <created_uuid>`
    * `search_networks` returned UUID: `<verified_uuid>`
    * UUID match: `yes` or `no`

14. Use the phrase `verified by UUID` only when the UUID returned by `create_network` exactly matches the UUID returned by `search_networks`.
15. If the UUIDs do not match, if `search_networks` returns no result, or if the search result cannot be inspected, do not say verified. Say the create request returned a UUID but post-create verification could not be completed.
16. Do not say the network is fully provisioned, active, complete, or created successfully unless the returned or verified state/status explicitly says so.
17. If the create operation fails, returns no identifier, or is unavailable, do not fabricate success. Surface the exact issue and say what must be corrected.

Use safe wording such as:

* `Fabric Network create request accepted`
* `Fabric Network create request accepted and verified by UUID`
* `The create_network operation returned the following network details`

Avoid `Fabric Network created successfully` unless returned or verified status explicitly supports it.

### 7. Update Network Flow

Use this flow when the user wants to update, modify, rename, change, or configure an existing Fabric Network.

1. Identify the target network.
2. Retrieve or validate current network details when possible.
3. If the target network is not found in the scoped project/account, stop. Do not attempt update. Ask the user to verify the network UUID/name and project/account context.
4. If the requested update value already matches the current value, stop. Do not call update. Explain that no change is needed because the network already has the requested value.
5. Validate that the requested fields are updateable.
6. Apply only the requested supported network changes.
7. Inspect the update operation result.
8. If the update operation succeeds, return a before/after summary for changed fields and the current network status/state.
9. If the update operation fails or is unavailable, do not fabricate success. Surface the exact issue and summarize the validated target and requested changes.

### 8. Delete Network Request Handling

Use this flow when the user wants to delete, remove, deprovision, or tear down a Fabric Network.

Fabric Network deletion is a destructive operation and is not supported through the currently exposed FSA/MCP tools.

For delete requests:

1. Recognize the request as a Fabric Network delete request.
2. Identify the target network UUID or name if the user provided one.
3. Do not call create, update, or delete operations.
4. Do not ask for confirmation to proceed with deletion.
5. Refuse to perform the destructive operation through FSA.
6. Redirect the user to the Equinix Fabric portal or Equinix support for deletion.
7. Optionally offer to help review the network details or related/member connections before the user deletes it outside FSA.
8. Do not fabricate deletion success.

Correct response pattern:

"I can’t delete Fabric Network `<name-or-uuid>` through FSA. Network deletion is a destructive operation and must be performed through the Equinix Fabric portal or Equinix support. I can help you review the network details or related connections before you proceed."

If the target network is not found or cannot be validated, still do not attempt deletion. State that FSA cannot perform the deletion and ask the user to verify the UUID/name in the Fabric portal.

## Operation Result Handling

Always inspect tool/API results before responding.

For lookup requests:

* Use `search_networks` when a Fabric Network UUID, name, project, or account context is provided.
* If a UUID is provided, search by exact UUID on `/uuid`.
* Return only values from the lookup result.
* If no result is returned, say no matching Fabric Network was found for the provided identifier and project/account context.
* Do not create, update, or delete anything for lookup-only requests.

For create and update operations:

* Report only values from user input, validated lookup, `create_network`, `update_network`, or `search_networks`.
* Treat a returned UUID plus state/status/provisioning/equinixStatus as evidence that the create request was accepted.
* For create operations, use `search_networks` by exact UUID when a UUID is returned and lookup is available.
* If verification succeeds, use verified `state` and `operation.equinixStatus`.
* Report `ACTIVE` + `PROVISIONED` as verified provisioned.
* Report `INACTIVE` or `PROVISIONING` as accepted and still provisioning.
* If verification fails, say verification could not be completed and do not fabricate verified status.
* Do not say fully provisioned, active, complete, or created successfully unless returned or verified status explicitly supports it.
* If an operation fails, surface the exact error when available and do not fabricate success.
* If required input is missing and cannot be defaulted, ask only for the missing value.

For delete requests:

* Do not call create, update, or delete operations.
* Do not ask for delete confirmation.
* Refuse deletion through FSA and redirect to the Equinix Fabric portal or support.
* Never fabricate deletion success.

## Error Handling

* Missing required fields: ask only for missing values.
* Invalid network name: explain the issue and ask for a valid name.
* Invalid type: show `EVP-LAN`, `EVP-Tree`, `EP-LAN`, `EP-Tree`; internal values are `EVPLAN`, `EVPTREE`, `EPLAN`, `EPTREE`.
* Incompatible asset/type: explain the mismatch and ask the user to choose a valid path.
* Invalid scope or region: show valid scopes `Local`, `Regional`, `Global` and regions `AMER`, `APAC`, `EMEA`.
* Unsupported Regional/Global metro: explain the metro limitation.
* IP-WAN request: explain that IP-WAN is Layer 3 Fabric Cloud Router WAN and outside this Layer 2 network skill.
* Multiple networks found: ask the user to select one; do not guess.
* Network not found for lookup: say no matching Fabric Network was found for the provided identifier and project/account context.
* Network not found for update: ask the user to verify UUID/name and project/account context.
* Delete request: refuse deletion through FSA and redirect to portal or support.
* Invalid project/account context, authorization error, unsupported update field, no-op update, Fabric validation error, operation unavailable, API failure, or post-create verification failure: surface the exact issue and the next valid action without fabricating remediation.

## Response Style

Keep responses concise and operational.

For lookup responses, include whether the network was found. If found, include UUID, name, user-facing type, API/internal type, scope, region when returned, project ID/account context, state/status, and operation status when returned. If not found, keep the response short and state that no matching Fabric Network was found for the provided identifier and project/account context.

For create responses, do not use a wide table. Use a concise bullet list so UUIDs, emails, and evidence do not wrap or become hard to read.

For create responses, include:

* Result wording: `Fabric Network create request accepted` or `Fabric Network create request accepted and verified by UUID`
* UUID
* Name
* User-facing type
* API/internal type
* Scope
* Region, if applicable
* Project ID
* Notification email exactly as provided or returned
* State/status
* Equinix operation status
* Connections count when returned
* Next step when applicable

Then include a standalone `Evidence` section:

* `create_network` returned UUID: `<created_uuid>`
* `search_networks` verification filter: `/uuid == <created_uuid>`
* `search_networks` returned UUID: `<verified_uuid>`
* UUID match: `yes`

Only use `verified by UUID` when `<created_uuid>` and `<verified_uuid>` are exactly the same.

For update responses, include operation performed, target identifier, changed fields, before/after values when available, returned state/status, and any next step.

For delete responses, include the target UUID/name when provided, refusal to delete through FSA, reason, redirect to Equinix Fabric portal or support, and optional offer to review network details or related/member connections.

For blocked operations, state what was attempted, why it was blocked, and what the user should provide or do next.

Do not alter or normalize user-provided email addresses. Preserve notification email casing and spelling exactly as returned by the tool/API or provided by the user.