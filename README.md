# Affinda Agent Plugin

This is the public runtime plugin for using Affinda with AI agents.
It bundles:

- the Affinda MCP server configuration;
- the Affinda plugin at `plugins/affinda`;
- the Affinda skill at `plugins/affinda/skills/affinda`;
- Claude, Codex, and Cursor plugin manifests.

The default MCP endpoint is:

```text
https://mcp.affinda.com/mcp
```

If your Affinda account is hosted in the US environment, use:

```text
https://mcp.us1.affinda.com/mcp
```

Claude users are prompted for the MCP endpoint during plugin setup. Codex
and Cursor users should update the MCP endpoint in their client settings
if they need the US environment.

This repository is generated from Affinda's private source repository.
The public repository is an installable distribution, not the source of
truth for changes.

The top-level repository is a marketplace root. The installable plugin
content lives under `plugins/affinda`.
