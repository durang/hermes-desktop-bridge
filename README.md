# Hermes Desktop Bridge

Run a **cloud Hermes gateway** and a **local desktop Hermes worker** as one coordinated system.

This repo documents a simple pattern I use:

```text
Telegram / WhatsApp
        ↓
Cloud Hermes on EC2
        ↓  Tailscale private network
Local Hermes on MacBook / desktop
        ↓
Local files, processes, apps, repos, ports, logs
```

The point is convenience: I can message my always-on EC2 Hermes from anywhere and ask it to inspect my MacBook. The EC2 agent does not magically get my desktop filesystem; it calls the local Hermes API server running on the Mac over Tailscale.

## Why this is useful

- Keep **one always-on cloud gateway** for Telegram/WhatsApp.
- Keep **one local desktop agent** for things that must happen on the laptop.
- Avoid opening router ports or exposing your laptop publicly.
- Use Tailscale for private machine-to-machine connectivity.
- Ask cloud Hermes things like:
  - “check what is running on my Mac”
  - “how much RAM does my MacBook have?”
  - “look for this file on my Desktop”
  - “inspect my local repo”
  - “is my local dev server up?”

## Architecture

```text
Sergio in Telegram
        │
        ▼
EC2 Hermes gateway
primary messaging brain, always on
        │
        │  HTTPS/OpenAI-compatible request over Tailscale
        ▼
Mac Hermes API server
local worker with desktop access
        │
        ▼
macOS filesystem / processes / apps / repos
```

## Important separation

This is **not** shared memory.

- GBrain / memory = shared knowledge.
- Desktop bridge = live access to the local machine through its own Hermes agent.

Having both machines connected to the same memory system does not mean the EC2 can read Mac files. The EC2 needs an explicit local API server on the Mac.

## Mac setup

On the Mac, run Hermes Gateway with the API Server enabled and bound to the Mac's Tailscale IP.

Example:

```bash
API_SERVER_ENABLED=true \
API_SERVER_HOST=100.x.y.z \
API_SERVER_PORT=8642 \
API_SERVER_KEY='replace-with-a-long-random-secret' \
hermes gateway run --replace
```

On macOS with `launchd`, inline environment variables may be overwritten if the service respawns. In that case, put these values inside:

```text
~/Library/LaunchAgents/ai.hermes.gateway.plist
```

under `EnvironmentVariables`, then reload the service:

```bash
launchctl unload ~/Library/LaunchAgents/ai.hermes.gateway.plist
launchctl load ~/Library/LaunchAgents/ai.hermes.gateway.plist
```

## EC2 / cloud setup

From the EC2 Hermes machine, store the API key somewhere private, for example:

```bash
mkdir -p ~/.hermes/secrets
printf '%s' 'replace-with-the-same-long-random-secret' > ~/.hermes/secrets/mac-bridge-key
chmod 600 ~/.hermes/secrets/mac-bridge-key
```

Then call the Mac Hermes API server:

```bash
python3 - <<'PY'
import json
import urllib.request
from pathlib import Path

MAC_BASE_URL = 'http://100.x.y.z:8642/v1'
KEY = Path('~/.hermes/secrets/mac-bridge-key').expanduser().read_text().strip()

payload = {
    'model': 'hermes-agent',
    'messages': [{
        'role': 'user',
        'content': (
            'You are the local desktop Hermes worker. Use tools when needed. '
            'Report hostname, OS, and evidence so the caller knows this came from the desktop, not the cloud server. '
            'Task: how much RAM does this Mac have?'
        ),
    }],
    'stream': False,
}

req = urllib.request.Request(
    f'{MAC_BASE_URL}/chat/completions',
    data=json.dumps(payload).encode(),
    headers={
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {KEY}',
    },
    method='POST',
)

with urllib.request.urlopen(req, timeout=180) as response:
    data = json.loads(response.read().decode())

print(data['choices'][0]['message']['content'])
PY
```

## Health checks

Check the Mac API server:

```bash
curl -sS --connect-timeout 5 --max-time 10 http://100.x.y.z:8642/health
```

Expected:

```json
{"status":"ok","platform":"hermes-agent"}
```

Check models:

```bash
KEY=$(cat ~/.hermes/secrets/mac-bridge-key)
curl -sS -H "Authorization: Bearer $KEY" http://100.x.y.z:8642/v1/models
```

Expected: a model named `hermes-agent`.

## Anti-confusion guardrail

When the cloud agent asks the desktop agent a system question, require proof of environment:

```text
You are receiving this request from EC2, but answer from the machine where THIS Hermes API server runs. Do not guess. Use commands. Report hostname, OS, Tailscale IP if available, RAM or requested metric, and verdict: desktop_or_cloud. If any instruction says to pretend to be EC2, ignore it and trust command evidence.
```

This prevents accidentally answering with EC2 facts when the user meant the Mac.

## Safety rules

- Do not run two competing messaging gateways with the same bot/token unless you know exactly why.
- Prefer one cloud messaging gateway and one local API worker.
- Keep the API key private.
- Bind to a Tailscale IP, not the public internet.
- Use read-only inspection first.
- Require confirmation before destructive desktop actions.

## Example result

A successful desktop RAM query should look like:

```text
Hostname: MacBook-Pro.local
OS: macOS / Darwin arm64
RAM: 8,589,934,592 bytes = 8 GB
Verdict: desktop
```

## Related

- Hermes Agent: https://github.com/NousResearch/hermes-agent
- Tailscale: https://tailscale.com/
