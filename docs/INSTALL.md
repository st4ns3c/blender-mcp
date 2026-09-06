# Install guide — Blender MCP + OpenCode on Kali Linux

Step-by-step record of the working setup. Follow top to bottom on a fresh machine.

## 0. Prerequisites

- Kali Linux (tested on Rolling 2026.3) with sudo access
- Blender installed and launchable (tested on 5.2.1 LTS)
- OpenCode installed (tested on 1.18.29, binary at `~/.opencode/bin/opencode`)
- Internet access for `apt` and PyPI

Check versions:

```bash
blender --version
python3 --version
~/.opencode/bin/opencode --version
lsb_release -a
```

## 1. Install system dependencies

Kali blocks global `pip` installs (PEP 668), so we need `venv`:

```bash
sudo apt update
sudo apt install -y python3-pip python3-venv git
```

## 2. Create an isolated Python environment and install blender-mcp

```bash
mkdir -p ~/blender-ai && cd ~/blender-ai
python3 -m venv venv
source venv/bin/activate
pip install blender-mcp
```

Confirm:

```bash
./venv/bin/pip show blender-mcp
# Name: blender-mcp / Version: 1.9.1
./venv/bin/blender-mcp --help
# subcommands: install-addon, addon-paths
```

> Do not run `blender-mcp --port 9876`. That flag does not exist in v1.9.1.
> The MCP server is spawned by OpenCode over stdio. The TCP port belongs
> to the Blender addon (step 4).

## 3. Install the Blender addon

```bash
~/blender-ai/venv/bin/blender-mcp install-addon
~/blender-ai/venv/bin/blender-mcp addon-paths
```

Expected output:

```text
Installed MCP for Blender addon to /home/<user>/.config/blender/5.2/scripts/addons/blender_mcp.py.
In Blender: Preferences → Add-ons → disable then enable 'Interface: MCP for Blender',
or restart Blender, then click Start MCP Server.
```

Verify the file exists:

```bash
ls -la ~/.config/blender/5.2/scripts/addons/
# blender_mcp.py (~175 KB)
```

## 4. Enable and connect inside Blender

1. **Restart Blender.** It must start after step 3 so it picks up the new addon.
2. Go to `Edit > Preferences > Add-ons`, search `MCP`, enable **`Interface: MCP for Blender`**.
3. Back in the main window, hover the mouse over the central **3D Viewport** and press **`N`**.
   - This toggles the Sidebar on the right edge of the viewport.
   - Alternative: `View > Sidebar`.
4. In the Sidebar header, click the **`BlenderMCP`** tab (use the `>` arrow if tabs overflow).
5. Click **`Connect` / `Start MCP Server`** (label varies by version). Status should turn green / say connected.

How to know it worked — in a terminal:

```bash
ss -tln | grep 9876
# LISTEN 0 5 127.0.0.1:9876 0.0.0.0:*
```

No listener = addon installed but not started. Repeat step 5.

Linux window-manager note: if Alt-drag moves the whole Blender window instead of
orbiting, change your DE's window-move modifier from `Alt` to `Super`.

## 5. Register the server with OpenCode

Edit `~/.config/opencode/opencode.jsonc` (create it if missing):

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

Replace `/home/kali` with the actual home directory on the target machine.
Use an absolute path — `~/...` is not expanded here.

Restart OpenCode (or start a new session) so it launches the MCP subprocess.

Reference: <https://opencode.ai/docs/mcp-servers/> — local servers use
`type: "local"` + `command: [...]` and communicate over stdio.

## 6. Verify end to end

```bash
# A. OpenCode sees the server
~/.opencode/bin/opencode mcp list
# ✓ blender connected — /home/kali/blender-ai/venv/bin/blender-mcp

# B. Blender socket is up
ss -tln | grep 9876

# C. Direct Blender query (no OpenCode involved)
~/blender-ai/venv/bin/python3 -c "
import socket, json
s = socket.socket(); s.settimeout(10)
s.connect(('localhost', 9876))
s.sendall(json.dumps({'type': 'get_scene_info'}).encode())
print(s.recv(65536).decode()[:2000])
s.close()
"
```

Healthy response looks like:

```json
{"status": "success", "result": {"name": "Scene", "object_count": 3,
"objects": [{"name": "Cube", ...}, {"name": "Light", ...}, {"name": "Camera", ...}]}}
```

Default Blender startup scene is Cube + Light + Camera — that is the expected baseline.

## 7. First AI test

In OpenCode, with Blender open and connected:

```text
What objects are in the current Blender scene?
```

Then:

```text
Create a UV sphere at (0, 0, 2) and give it a red Principled BSDF material.
```

Confirm visually in Blender's viewport. Save the `.blend` before destructive prompts.

## 8. Reusing this on another machine

1. Copy this repo / follow steps 1–5 verbatim.
2. Only two values change per machine: the home-directory path in `opencode.jsonc`
   and the Blender version directory under `~/.config/blender/<version>/`.
3. Re-run the three verification commands in step 6 before demoing.

## Appendix: what each piece does

| Piece | Role | If it breaks |
|---|---|---|
| `~/blender-ai/venv` | Isolated Python env with `blender-mcp` | Re-create venv, `pip install blender-mcp` |
| `blender_mcp.py` addon | Socket server inside Blender on :9876 | Re-run `install-addon`, restart Blender |
| `opencode.jsonc` | Tells OpenCode how to spawn the MCP | Check JSON syntax, absolute path, restart OpenCode |
| Port 9876 listener | Proof Blender side is live | Click Connect in BlenderMCP tab |
| `opencode mcp list` | Proof OpenCode side is live | Check command path, `enabled: true` |
