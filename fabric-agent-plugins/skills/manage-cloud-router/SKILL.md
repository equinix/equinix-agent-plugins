---
name: manage-cloud-router
  
description: >
  Manages Equinix Fabric Cloud Router (FCR) workflows:
  create, list/search, and update routers; 
  configure and update BGP on routers/connections (including IPv4/IPv6 enablement); 
  and create, attach, and update route filters, route filter rules, route aggregations, and route aggregation rules. 
  Use when users request FCR router or routing-policy operations; not for non-FCR resources.

metadata:
  author: Fabric
  version: 1.0
  last_reviewed: 2026-09-01
  filter: '{"or":[{"and":["routers","write"]},{"and":["routers","read"]},{"and":["route_filters","write"]},{"and":["route_filters","read"]},{"and":["route_filter_rules","write"]},{"and":["route_filter_rules","read"]},{"and":["route_aggregations","write"]},{"and":["route_aggregations","read"]},{"and":["route_aggregation_rules","write"]},{"and":["route_aggregation_rules","read"]},{"and":["routing_protocols","write"]},{"and":["routing_protocols","read"]},{"and":["prices","read"]},{"and":["connections","read"]}]}'
---

# Manage Cloud Router

## When to use this skill

**Trigger scenarios and phrases:**
- "Create / provision / deploy a BASIC Fabric Cloud Router or FCR in Silicon Valley"
- "Upgrade my FCR to STANDARD package"
- "Update / rename / modify / patch a cloud router"
- "Configure BGP on my FCR connection" → *Note: this skill manages the router resource, not the connections; for connection-level BGP configuration, use `manage-fabric-connection`*
- "Configure / set up BGP on my FCR connection or router"
- "Disable / enable IPv4 or IPv6 BGP on a connection"
- "Find / list / search / show / get my cloud routers"
- "Show all routers in my org / account"
- "How many routers do I have?", "What package is my router on?"
- "Check my router's status / state"
- "Create / add a route filter for my FCR"
- "Attach / associate a route filter to a connection"
- "Rename / update a route filter"
- "Add / create a route filter rule on a route filter"
- "Update / rename a route filter rule"
- "Create / provision a route aggregation"
- "Attach / associate a route aggregation to a connection"
- "Rename / update a route aggregation"
- "Add / create a route aggregation rule on a route aggregation"
- "Update / rename a route aggregation rule"

**Do NOT trigger this skill for:**
- Managing connections between routers → use `manage-fabric-connection`

## User Context
Primarily used by **network engineers** managing Equinix Fabric routing infrastructure.
Assume familiarity with BGP, prefix notation, and metro codes.
The logged-in username is available in the system prompt. **Always scope router lookups to that username by default** — when listing existing routers or checking project limits, filter by the user's own account.

- **Default**: filter existing routers by the logged-in user's account
- **Override**: broaden to org/account scope only when the user explicitly asks (e.g. "show all routers in the org", "check team's routers")

## Persona
**Network engineer** who manages cloud router lifecycle — provisioning FCRs, tuning packages, and deprovisioning routers.

## Prerequisites
Before starting any operation, confirm:
- The logged-in user's account is available — all lookups default to that user's scope
- IAM role: user must have `Fabric Cloud Router Manager` or `Fabric Manager`

## Operation routing
Detect the resource type AND operation from the user's message, then read the matching reference file:
- User wants to **create / provision / deploy** a router → read `references/create-router.md` and follow those steps
- User wants to **update / upgrade / rename / modify** a router → read `references/update-router.md` and follow those steps
- User wants to **create / attach / configure** BGP on a router or connection → read `references/bgp-protocol.md` and follow those steps
- User wants to **update / modify** BGP enabled status on an existing connection → read `references/update-bgp-protocol.md` and follow those steps
- User wants to **enable / disable** BGP IPv4 or IPv6 on an existing connection → read `references/update-bgp-protocol.md` and follow those steps
- User wants to **create** a route filter → read `references/create-route-filter.md` and follow those steps
- User wants to **attach** a route filter → read `references/attach-route-filter.md` and follow those steps
- User wants to **update** a route filter → read `references/update-route-filter.md` and follow those steps
- User wants to **create** a route filter rules → read `references/create-route-filter-rule.md` and follow those steps
- User wants to **update** a route filter rules → read `references/update-route-filter-rule.md` and follow those steps
- User wants to **create** a route aggregation → read `references/create-route-aggregation.md` and follow those steps
- User wants to **attach** a route aggregation → read `references/attach-route-aggregation.md` and follow those steps
- User wants to **update** a route aggregation → read `references/update-route-aggregation.md` and follow those steps
- User wants to **create** a route aggregation rules → read `references/create-route-aggregation-rule.md` and follow those steps
- User wants to **update** a route aggregation rules → read `references/update-route-aggregation-rule.md` and follow those steps
- **Intent unclear** → ask exactly one question before proceeding: *"Are you looking to create, search, or update a Fabric Cloud Router?"*

---

## Confirmation and error handling
Refer to the reference files listed under `Operation routing`.

## Boundary notes
This skill does NOT handle:
- **Connection management other than BGP configuration** → use `manage-fabric-connection`
- **Service profile** → use `manage-service-profile`
- **Service tokens, buyer access token lifecycles** → use `manage-service-token`
- **NTP/PTP time service lifecycles** → use `manage-time-service`
- **Port, Link Aggregation Groups (LAG) lifecycles** → use `manage-fabric-port`
- **Multipoint network lifecycle** → use `manage-fabric-network`
- **Marketplace company identity lifecycle** → use `manage-company-profile`
- **Billing or account setup** → direct the user to the Equinix Fabric portal