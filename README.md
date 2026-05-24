# Affinda Agent Plugin

This is the public runtime plugin for using Affinda with AI agents.
It bundles:

- the Affinda MCP server configuration;
- the Affinda Global plugin at `plugins/affinda`;
- the Affinda US plugin at `plugins/affinda-us1`;
- the Affinda skill bundled with each regional plugin;
- Claude, Codex, and Cursor plugin manifests.

Choose the plugin that matches your Affinda account region. The default
Affinda plugin uses the Global/AP1 MCP endpoint:

```text
https://mcp.affinda.com/mcp
```

The Affinda US plugin uses:

```text
https://mcp.us1.affinda.com/mcp
```

Install one regional plugin at a time. Installing both exposes duplicate
Affinda tool sets and can make region selection ambiguous.

This repository is generated from Affinda's private source repository.
The public repository is an installable distribution, not the source of
truth for changes.

The top-level repository is a marketplace root. The installable plugin
content lives under `plugins/affinda` and `plugins/affinda-us1`.
