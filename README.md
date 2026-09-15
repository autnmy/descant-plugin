# Descant plugin

The customer half of Descant's agent plugin: task Skills, and the MCP
server configuration that points an agent at your account's API.

Built from `cli-v0.1.2` (source commit `029a383432f6d8de2817de61285d62fc30c2f0e2`). This repository
holds the BUILT artifact only — no source lives here.

## Install

```sh
descant plugin install
```

It reads `plugin.json` for the Skill list, copies `skills/` into
your agent's Skills directory, and merges `mcp.json` into your MCP
client configuration. Verify the download with:

```sh
sha256sum -c SHA256SUMS
```

## Credential

Every operation these Skills name is authenticated by your own account
key — the one `descant login` stores, or `$DESCANT_API_KEY`.
