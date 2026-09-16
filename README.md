# Descant plugin

The customer half of Descant's agent plugin: task Skills, and the MCP
server configuration that points an agent at your account's API.

Built from `cli-v0.1.5` (source commit `2d5cda46b33f7cdf55f41dd99a241a961d5048e5`). This repository
holds the BUILT artifact only — no source lives here.

## Install

```sh
descant plugin install \
  --skills-dir <where your client reads Skills> \
  --mcp-config <your client's MCP config file>
```

BOTH PATHS ARE REQUIRED, and that is deliberate. MCP clients disagree
about where configuration lives and those locations change, so this
command writes exactly where you point it rather than guessing — a
guess that misses writes your Skills somewhere nothing reads them,
which looks exactly like a successful install.

It verifies every file against `SHA256SUMS` before writing anything,
copies `skills/` to the directory you named, and merges one entry
into your MCP config without touching the rest of it. Run it twice and
nothing changes the second time. Verify the download yourself with:

```sh
sha256sum -c SHA256SUMS
```

## Credential

Every operation these Skills name is authenticated by your own account
key — the one `descant login` stores, or `$DESCANT_API_KEY`.
