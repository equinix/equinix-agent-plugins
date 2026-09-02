# equinix-agent-plugins

Agent plugins for Equinix products. This repository is a **plugin marketplace**: it packages
Equinix skills plus the Equinix remote MCP server so agentic coding tools (Claude Code, Codex,
and anything else that speaks MCP) can operate Equinix Fabric directly.

## What's in here

| Plugin | Skills | MCP server |
|---|---|---|
| `equinix-fabric` | `show-fabric-inventory`, `manage-cloud-router` | `https://mcp.equinix.com/fabric` |

- **`show-fabric-inventory`** — search and list connections, ports, cloud routers, service
  tokens, route filters, route aggregations, and time services.
- **`manage-cloud-router`** — create/update Fabric Cloud Routers, configure BGP (IPv4/IPv6),
  and manage route filters, route filter rules, route aggregations, and aggregation rules.

### Layout

```
.claude-plugin/marketplace.json      # marketplace manifest (what Claude Code adds)
mcp.json                             # portable MCP server definition for other agents
codex/config.toml.example            # Codex MCP snippet
fabric-agent-plugins/                # the equinix-fabric plugin
├── .claude-plugin/plugin.json       # plugin manifest
├── .mcp.json                        # remote MCP server bundled with the plugin
└── skills/
    ├── show-fabric-inventory/SKILL.md
    └── manage-cloud-router/SKILL.md
```

## Install in Claude Code

Add the marketplace, then install the plugin:

```bash
claude plugin marketplace add equinix/equinix-agent-plugins
```

```bash
claude plugin install equinix-fabric@equinix-agent-plugins
```

Or from inside an interactive Claude Code session, use `/plugin` and pick
`equinix-fabric` from the `equinix-agent-plugins` marketplace.

The plugin ships the remote MCP server, so no separate MCP setup is needed. Authorize it on
first use:

```bash
claude mcp list
```

If `equinix-fabric` shows as needing auth, run `/mcp` in an interactive session and complete
the browser sign-in.

Verify the skills loaded:

```bash
claude plugin list
```

Then try: *"show me all my Fabric connections in SV"* or *"create a Fabric Cloud Router in DA"*.

To update or remove:

```bash
claude plugin update equinix-fabric@equinix-agent-plugins
```

```bash
claude plugin uninstall equinix-fabric@equinix-agent-plugins
```

### Install in Claude Desktop (UI)

If you use the Claude desktop app instead of the terminal, add the marketplace from Settings:

1. Open **Claude Desktop** and go to **Settings → Customize**.
2. Select **Plugins**.
3. Click **Add marketplace**.
4. Paste the repository link and confirm:

   ```
   https://github.com/equinix/equinix-agent-plugins
   ```

   The `equinix/equinix-agent-plugins` shorthand works too.
5. Wait for the marketplace to **sync** — Claude fetches
   `.claude-plugin/marketplace.json` from the repo and lists the plugins it declares. Use the
   **Sync** / refresh control on the marketplace entry any time you want to pull newly
   published plugins or versions.
6. Find **equinix-fabric** in the marketplace listing and click **Install**, then make sure
   its toggle is **enabled**.
7. Authorize the bundled MCP server: open **Settings → Connectors** (or the `/mcp` prompt in a
   session), find **equinix-fabric**, and complete the browser sign-in. The skills work only
   once the server is connected.
8. Restart or start a new session, then try *"show me all my Fabric connections in SV"*.

To update, hit **Sync** on the marketplace and then **Update** on the plugin. To remove, use
**Uninstall** on the plugin — or **Remove marketplace** to drop the whole source.

### Team-wide install

Commit this to your project's `.claude/settings.json` so every teammate gets the plugin on
clone:

```json
{
  "extraKnownMarketplaces": {
    "equinix-agent-plugins": {
      "source": {
        "source": "github",
        "repo": "equinix/equinix-agent-plugins"
      }
    }
  },
  "enabledPlugins": {
    "equinix-fabric@equinix-agent-plugins": true
  }
}
```

## Install in Codex

Codex has no marketplace, so wire up the MCP server and the skills separately.

**1. Add the MCP server.** Either run:

```bash
codex mcp add equinix-fabric --url https://mcp.equinix.com/fabric
```

…or merge [`codex/config.toml.example`](codex/config.toml.example) into `~/.codex/config.toml`.
Remote streamable-HTTP servers need `experimental_use_rmcp_client = true` at the top level of
that file.

Confirm it registered:

```bash
codex mcp list
```

**2. Add the skills.** Clone this repo and link the skills into your Codex skills directory:

```bash
git clone https://github.com/equinix/equinix-agent-plugins.git ~/src/equinix-agent-plugins
```

```bash
mkdir -p ~/.codex/skills && ln -sfn ~/src/equinix-agent-plugins/fabric-agent-plugins/skills/show-fabric-inventory ~/.codex/skills/show-fabric-inventory && ln -sfn ~/src/equinix-agent-plugins/fabric-agent-plugins/skills/manage-cloud-router ~/.codex/skills/manage-cloud-router
```

Use `.codex/skills/` inside a repo instead of `~/.codex/skills/` to scope the skills to one
project. Pull the repo to update; remove the symlinks to uninstall.

## Install in other MCP clients

Any MCP client that supports remote HTTP servers can use the server definition in
[`mcp.json`](mcp.json):

```json
{
  "mcpServers": {
    "equinix-fabric": {
      "type": "http",
      "url": "https://mcp.equinix.com/fabric"
    }
  }
}
```

The skills are plain Markdown — point your agent's skill/instruction loader at
`fabric-agent-plugins/skills/`, or paste a `SKILL.md` into its system instructions.

## Contributing a plugin

1. Create `<my-plugin>/.claude-plugin/plugin.json` with `name`, `version`, and `description`.
2. Put skills under `<my-plugin>/skills/<skill-name>/SKILL.md`, with any supporting docs in a
   sibling `references/` directory.
3. Add remote MCP servers to `<my-plugin>/.mcp.json`.
4. Register the plugin in the `plugins` array of
   [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json).

Validate before opening a PR:

```bash
claude plugin validate .
```

Test against your working copy without publishing:

```bash
claude plugin marketplace add /path/to/equinix-agent-plugins
```
