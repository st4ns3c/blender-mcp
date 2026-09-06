# Blender MCP + OpenCode on Linux

Drive Blender with natural language from OpenCode using the Model Context Protocol (MCP).

This repository documents a tested, repeatable setup for connecting **OpenCode** to **Blender** via **BlenderMCP** on **Kali Linux**. Use it to onboard teammates, clients, or students without debugging the same issues twice.

Tested stack:

| Component | Version |
|---|---|
| OS | Kali GNU/Linux Rolling 2026.3 |
| Blender | 5.2.1 LTS |
| Python | 3.14.6 (system) + venv |
| blender-mcp | 1.9.1 (`ahujasid/blender-mcp`) |
| OpenCode | 1.18.29 |
| Transport | stdio (OpenCode ↔ MCP) + TCP `127.0.0.1:9876` (MCP ↔ Blender) |

## What is MCP? What is Blender MCP?

**Model Context Protocol (MCP)** is an open standard for exposing tools to an LLM client over stdio or HTTP. OpenCode launches a local MCP server as a subprocess and calls its tools.

**BlenderMCP** (`ahujasid/blender-mcp`) has two parts:

1. **MCP server** (`blender-mcp` Python package) — speaks MCP/stdio to OpenCode, speaks JSON-over-TCP to Blender.
2. **Blender addon** (`blender_mcp.py`) — runs inside Blender, opens a socket server on `127.0.0.1:9876`, executes `bpy` commands, returns scene data.

Data flow:

```text
OpenCode  ⇄  stdio  ⇄  blender-mcp server  ⇄  TCP :9876  ⇄  Blender addon  ⇄  bpy
```

With it connected you can: inspect the scene, create/modify/delete objects, assign materials, run arbitrary `bpy` code, take viewport screenshots, and trigger renders — all from a prompt.

## Quickstart (Kali / Debian-based)

```bash
# 1. System deps
sudo apt update
sudo apt install -y python3-pip python3-venv git

# 2. Isolated env (required on Kali — PEP 668 blocks global pip)
mkdir -p ~/blender-ai && cd ~/blender-ai
python3 -m venv venv
source venv/bin/activate
pip install blender-mcp

# 3. Install the Blender addon into Blender's user scripts
./venv/bin/blender-mcp install-addon
./venv/bin/blender-mcp addon-paths
# expected: /home/<user>/.config/blender/5.2/scripts/addons

# 4. Register the MCP server with OpenCode
# edit ~/.config/opencode/opencode.jsonc (see Configuration below)

# 5. In Blender: restart, enable addon, connect
# Edit > Preferences > Add-ons > enable "Interface: MCP for Blender"
# Hover 3D Viewport > press N > BlenderMCP tab > Connect / Start MCP Server

# 6. Verify
~/.opencode/bin/opencode mcp list
ss -tln | grep 9876
```

Full walkthrough with outputs and troubleshooting: [`docs/INSTALL.md`](docs/INSTALL.md).

## Configuration

OpenCode config file: `~/.config/opencode/opencode.jsonc`

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

Notes:

- Use the **absolute path** to the venv binary. Replace `/home/kali` with the target user's home.
- No arguments. Older guides show `blender-mcp --port 9876` — that flag does **not** exist in v1.9.1 and will prevent startup. The port is owned by the Blender addon, not the CLI.
- `type: "local"` means OpenCode spawns the command and talks over stdio. Do not run `blender-mcp` manually in a terminal.
- After editing, restart OpenCode (or the agent session) so it respawns the MCP subprocess.

For other clients (Claude Desktop / Cursor), the equivalent is:

```json
{
  "mcpServers": {
    "blender": {
      "command": "/home/kali/blender-ai/venv/bin/blender-mcp"
    }
  }
}
```

## Verification

1. **OpenCode sees the server:**

   ```bash
   ~/.opencode/bin/opencode mcp list
   # expected: ✓ blender connected
   ```

2. **Blender addon socket is listening:**

   ```bash
   ss -tln | grep 9876
   # expected: LISTEN 0 5 127.0.0.1:9876 0.0.0.0:*
   ```

   If port 9876 is missing, the addon is installed but not started — go back to Blender and press Connect.

3. **End-to-end query (bypasses OpenCode, talks straight to Blender):**

   ```bash
   ~/blender-ai/venv/bin/python3 -c "
   import socket, json
   s = socket.socket(); s.settimeout(10)
   s.connect(('localhost', 9876))
   s.sendall(json.dumps({'type': 'get_scene_info'}).encode())
   print(s.recv(65536).decode()[:2000])
   s.close()
   "
   # expected: {"status": "success", "result": {"name": "Scene", ...}}
   ```

## Project layout

```text
.
├── README.md          # overview + quickstart (this file)
└── docs/
    └── INSTALL.md     # full step-by-step for Kali with outputs
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `pip install` errors about externally-managed environment | Kali enforces PEP 668 | Use `python3 -m venv venv` and install inside it |
| `blender-mcp --port 9876` errors / exits | Flag does not exist in v1.9.1 | Run with no args; let OpenCode spawn it |
| `opencode mcp list` shows blender but Blender commands fail | Addon socket not running | In Blender: N-panel → BlenderMCP → Connect; confirm with `ss -tln` |
| No `BlenderMCP` tab after pressing N | Mouse not over 3D Viewport, or addon disabled | Hover viewport, press N or View → Sidebar; re-enable addon; restart Blender |
| `addon-paths` reports `(missing)` | Addon never installed | Run `blender-mcp install-addon`, then restart Blender |
| Alt-drag moves the Linux window instead of orbiting | DE intercepts Alt | Change window-move modifier from Alt to Super in GNOME/KDE settings |

## Security notes

- `execute_blender_code` runs arbitrary Python inside Blender. Save your `.blend` before AI-driven sessions.
- The socket binds to localhost only. Do not expose port 9876 to a network.
- Poly Haven / Sketchfab integrations download remote assets. Disable if working offline or in a restricted environment.

## Credits

- BlenderMCP by Siddharth Ahuja (`ahujasid/blender-mcp`, MIT)
- Blender Foundation (Blender 5.x LTS)
- OpenCode MCP docs: <https://opencode.ai/docs/mcp-servers/>

## License

Docs in this repo are yours to reuse internally. Underlying tools keep their own licenses: BlenderMCP (MIT), Blender (GPL).
