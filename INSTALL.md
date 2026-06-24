# hiTech Automation AI — Install & Setup

A local FastAPI web app for network automation across **five transports** —
NETCONF, RESTCONF, CLI, XPath, and Python — with Bruno/Postman-style request
collections, a VS Code-style foldable JSON/XML response viewer, live
**Netmiko Interactive** device sessions, and an integrated **agentic AI assistant**
(local Ollama models or the cloud Claude API) grounded by a local RAG corpus.

> Tested on Ubuntu 24.04. Single-user / localhost deployment.

---

## 1. Prerequisites

```bash
sudo apt update
sudo apt install -y python3 python3-venv git
```

- **Python 3.11+** (3.13 fine)
- **(Optional) Ollama** for local LLMs — https://ollama.com — running on this host
  or another reachable host (default `http://127.0.0.1:11434`).
- **(Optional) Anthropic API key** to use cloud Claude as the assistant brain.

---

## 2. Get the code

```bash
git clone git@github.com:sushil-bhattacharjee/hitech-automation-ai.git
cd hitech-automation-ai/software_ai
```

The application lives in `software_ai/build/`. Everything you run points there.

---

## 3. Python environment

This project uses a virtualenv kept **outside** `build/` so it survives upgrades.

```bash
# from software_ai/
python3 -m venv ../.venv          # creates software_ai/../.venv  (adjust to taste)
source ../.venv/bin/activate
pip install -r build/requirements.txt
```

> If you use `uv`, note that a `uv venv` has **no pip** by default — install with
> `uv pip install --python <venv>/bin/python -r build/requirements.txt`.

---

## 4. First run (manual, to verify)

```bash
cd build
uvicorn main:app --host 0.0.0.0 --port 7071
```

Open **http://<host-ip>:7071/**. You should see the **hiTech Automation AI** UI.
Stop with Ctrl-C once it loads.

---

## 5. Run as a service (systemd --user)

Create `~/.config/systemd/user/hitech_automation_ai.service`:

```ini
[Unit]
Description=hiTech Automation AI Web UI
After=network-online.target

[Service]
Type=simple
WorkingDirectory=%h/hitech-automation-ai/software_ai/build
ExecStart=%h/hitech-automation-ai/.venv/bin/uvicorn main:app --host 0.0.0.0 --port 7071
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

> Adjust `WorkingDirectory` / `ExecStart` to wherever you cloned the repo and
> created the venv. `%h` expands to your home directory.

Enable + start:

```bash
systemctl --user daemon-reload
systemctl --user enable hitech_automation_ai
systemctl --user start hitech_automation_ai
systemctl --user status hitech_automation_ai --no-pager
```

Daily upgrade cycle after that:

```bash
cd ~/hitech-automation-ai/software_ai
unzip -o hitech_automation_ai.<version>.zip      # extracts build/
systemctl --user restart hitech_automation_ai
```
Then hard-refresh the browser (**Ctrl+Shift+R**).

---

## 6. Private state directory

All your data lives **outside** `build/`, so upgrades never touch it:

```
~/.hitech_automation_ai/
├── devices.yaml            # device inventory
├── secrets.yaml            # optional: $secret:VAR values (chmod 600)
├── audit.log               # agent activity (JSON lines)
├── restconf_tree.json      # saved RESTCONF collections
├── netconf_tree.json       # saved NETCONF collections
├── cli_tree.json           # saved CLI collections
├── xpath_tree.json         # saved XPath collections
├── python_tree.json        # saved Python projects
└── vector_db/              # RAG embeddings (built from rag-corpus/)
```

> **Upgrading from the old `netconf-sender` build?** On first start the app
> auto-migrates `~/.netconf-sender/` → `~/.hitech_automation_ai/` (copies, leaves
> the original intact). No manual step required.

Override the location with `HITECH_STATE_DIR=/path` if needed.

---

## 7. Device inventory

Edit `~/.hitech_automation_ai/devices.yaml`, then click **↻ Reload** in the
Devices panel (no restart needed). Two formats are supported.

**Flat (simplest):**

```yaml
devices:
  - name: cat8Kv71
    host: 192.168.89.71
    port_netconf: 830
    port_ssh: 22
    username: expert
    password: $env:CAT8KV71_PASSWORD     # or a literal for labs
    device_type: cisco-iosxe
    read_only: false                     # false = allows config/interactive
    default: true
```

**pyATS testbed:**

```yaml
devices:
  cat8Kv71:
    os: iosxe
    credentials:
      default:
        username: "%ENV{CAT8KV71_USERNAME}"
        password: "%ENV{CAT8KV71_PASSWORD}"
    connections:
      cli:     {protocol: ssh,     ip: "%ENV{CAT8KV71}", port: 22}
      netconf: {protocol: netconf, ip: "%ENV{CAT8KV71}", port: 830}
    custom:
      read_only: false
      default: true
```

### Secret resolution (most → least secure)

| Reference in YAML | Resolved from | Notes |
|---|---|---|
| `%ENV{VAR}` / `$env:VAR` | process environment | set via systemd drop-in (below) |
| `$secret:VAR` | `~/.hitech_automation_ai/secrets.yaml` | whole-string only; **recommended** |
| literal value | the file itself | fine for throwaway labs only |

For env-var secrets, add a systemd drop-in
`~/.config/systemd/user/hitech_automation_ai.service.d/secrets.conf`:

```ini
[Service]
Environment="CAT8KV71=192.168.89.71"
Environment="CAT8KV71_USERNAME=expert"
Environment="CAT8KV71_PASSWORD=changeme"
```
Then `systemctl --user daemon-reload && systemctl --user restart hitech_automation_ai`.

> **Never commit `devices.yaml`, `secrets.yaml`, `secrets.conf`, or `.envrc`.**
> They are gitignored by default.

---

## 8. LLM assistant (optional)

Point at Ollama and/or Anthropic via the same drop-in (e.g. `ollama.conf`):

```ini
[Service]
Environment="OLLAMA_URL=http://127.0.0.1:11434"
Environment="OLLAMA_MODEL=qwen2.5-coder:7b"
Environment="ANTHROPIC_API_KEY=sk-ant-...your-key..."
```

Build the RAG index from the bundled corpus:

```bash
cd build
python rag_builder.py --rebuild
systemctl --user restart hitech_automation_ai
```

See `build/LLM_INTEGRATION.md` for the provider abstraction, RAG pipeline, and
hardware guidance.

---

## 9. Docker (optional)

A `Dockerfile` + `docker-compose.yml` ship in `build/`. Pass `OLLAMA_URL`,
`ANTHROPIC_API_KEY`, and device env vars via a compose `environment:` block or
an `.env` file (kept out of git).

---

## 10. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Page won't load / 500 on startup | `journalctl --user -u hitech_automation_ai -n 40 --no-pager` — read the traceback |
| `ModuleNotFoundError` after upgrade | venv missing a dep → `pip install -r build/requirements.txt` (or `uv pip install ...`) |
| Devices list empty | YAML parse error or env vars unset → check `journalctl`, and `systemctl --user show hitech_automation_ai -p Environment` |
| `host: ""` / `password_configured: false` | the referenced env var isn't set in the service environment |
| Interactive CLI blocked (403) | device is `read_only: true` → set `read_only: false` and ↻ Reload |
| Service not auto-starting on boot | run `systemctl --user enable hitech_automation_ai` |
