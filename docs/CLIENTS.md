# Client setup — Blender MCP for OpenCode, Codex, Grok, Claude Code

Shared prerequisite (once per machine): `~/blender-ai/venv/bin/blender-mcp`
installed via `pip install blender-mcp`, addon installed via `blender-mcp install-addon`,
Blender restarted with the addon enabled and connected (port 9876 listening).

Binary path below uses `/home/kali` — replace with the real home directory.
Never append `--port`; v1.9.1 takes no arguments.

## OpenCode

File: `~/.config/opencode/opencode.jsonc`

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "blender": {
      "type": "local",
      "command": ["/home/kali/blender-ai/venv/bin/blender-mcp"],
      "enabled": true
    }
  }
}
```

Verify:

```bash
~/.opencode/bin/opencode mcp list
# ✓ blender connected
```

## Codex CLI

```bash
codex mcp add blender -- /home/kali/blender-ai/venv/bin/blender-mcp
codex mcp list   # blender ... enabled
codex mcp get blender
```

Config lands in `~/.codex/config.toml` as `[mcp_servers.blender]`.
Remove with `codex mcp remove blender`.

## Grok CLI

```bash
grok mcp add blender /home/kali/blender-ai/venv/bin/blender-mcp -s user
grok mcp list
timeout 90 grok mcp doctor blender
# ✓ command found, server started, handshake OK, ~28 tools
```

Config lands in `~/.grok/config.toml` (user scope, all projects).
Project scope alternative: `-s project` writes `./.grok/config.toml`.
Remove with `grok mcp remove blender`.

## Claude Code

```bash
claude mcp add --transport stdio blender -- /home/kali/blender-ai/venv/bin/blender-mcp
claude mcp list
claude mcp get blender
```

Notes:

- Flags (`--transport`, `--scope`, `--env`) must come **before** the server name.
- `--` separates the name from the server command (required).
- Scopes: `--scope local` (default, this project only), `project` (`.mcp.json`, shared),
  `user` (all projects).
- JSON alternative:
  `claude mcp add-json blender '{"type":"stdio","command":"/home/kali/blender-ai/venv/bin/blender-mcp"}'`
- Docs: <https://code.claude.com/docs/en/mcp>

## Shared Blender-side check (all clients)

```bash
ss -tln | grep 9876
# LISTEN 0 5 127.0.0.1:9876 0.0.0.0:*
```

No listener = addon installed but Connect not pressed.
In Blender: hover 3D Viewport → `N` → `BlenderMCP` tab → Connect.
