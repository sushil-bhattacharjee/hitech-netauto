# hiTech Automation AI — Operation Guide

Day-to-day usage of the five transports, request collections, the response
viewer, Netmiko Interactive sessions, and the AI assistant.

> URL: **http://<host-ip>:7071/**  · Restart: `systemctl --user restart hitech_automation_ai` · Hard-refresh: **Ctrl+Shift+R**

---

## 1. Layout

- **Left sidebar** — transport navigation (NETCONF · RESTCONF · CLI · XPath · Python)
  and, below it, the **collections tree** for the active transport. Drag the right
  edge to resize the sidebar; **≡** collapses it.
- **Main pane** — the request builder / editor for the active transport.
- **Top-right** — Theme toggle and **Chat with AI**.

Each transport keeps its **own** collection tree, saved under
`~/.hitech_automation_ai/<transport>_tree.json`.

---

## 2. Collections (all transports)

Bruno/Postman-style nested folders of saved requests.

- **+ Folder** — create a folder (right-click a folder for sub-folders).
- **💾 Save →** — opens a graphical picker: click a destination folder, name the
  request, Save. (Right-click a folder → *Save current here* targets it directly.)
- **Click a saved item** to load it. Behavior by transport:
  - **RESTCONF** — loads into the builder (https / curl / python sub-modes; curl &
    python auto-run on click).
  - **NETCONF / CLI** — **load only** (review, then Validate/Send) — device writes
    never auto-run from a click.
  - **XPath** — auto-runs the query (read-only).
  - **Python** — loads the full multi-file project (then press Run).
- **Right-click** any node — rename, delete, new folder. **Drag-and-drop** to move.
- **Saved filters** — RESTCONF saves its jq filter; XPath saves its XPath-filter and
  Extract-fields config, and re-applies them on open.

---

## 3. Transports

### NETCONF
Jinja2 (vars + template) **or** raw XML payload → ncclient (port 830). The
**✔ Validate** button renders + validates against the device without pushing.

### RESTCONF — three sub-modes
- **https-restconf** — build a request (method, URL, params, headers, vars, auth,
  payload). **⧉ Pretty-print** formats the JSON body (template `{{vars}}` preserved).
  **Environment** selector substitutes `{{VAR}}` from the active environment.
- **curl-restconf** — paste a curl command; runs through the host shell (pipes,
  `$VARS`, `jq`, `$(...)` all work).
- **python-restconf** — run a Python snippet (uses `requests`, `os.environ`, etc.).

> Tip: APIC/ACI uses the ACI REST API (`/api/aaaLogin.json`), not YANG. Don't add a
> YANG `Accept` header for it. The login token is a cookie that expires (~10 min) —
> prefer a single login→query script (curl `Session`/`requests.Session`).

### CLI — three sub-modes
- **Jinja2 template** / **Raw CLI** → Netmiko (port 22).
- **Netmiko Interactive** — a live session that can answer device prompts. See §5.

### XPath
Run an on-device XPath query, with client-side **XPath filter** and
**Extract fields → table** post-processing.

> On Cisco IOS-XE, XPath is **unprefixed** (e.g. `/native/ip/access-list`).
> Namespace prefixes like `Cisco-IOS-XE-native:` cause an RPCError.

### Python
A multi-file Python runner with a CodeMirror editor. Save the whole project as one
collection node.

---

## 4. The response viewer (all panes)

Whenever a response is JSON or XML you get a **VS Code-style view**:

- **Line numbers + indent guides**.
- **Folding** — click the caret on any object/array/element; **⊟ collapse-all** /
  **⊞ expand-all** in the toolbar.
- **Bracket/tag-pair hover** — hover a `{` / `[` or an XML tag to highlight its match.
- **🔎 Find** — case toggle + regex/wildcard toggle; reveals matches inside folded
  nodes. **📋 Copy** copies the raw text.
- Big responses auto-collapse below depth 2 to stay fast.

---

## 5. Netmiko Interactive sessions

For commands that prompt for confirmation — `reload`, `no username X`,
`copy run start`, `delete flash:`, etc.

1. **CLI → Netmiko Interactive**.
2. Pick a device (**must be `read_only: false`**) → **▶ Open session**.
3. Type a command, Enter. The terminal shows whatever the device prints —
   including `(y/n)?` / `[confirm]` prompts.
4. When it's waiting on a prompt, type your answer (`y` / Enter) and Send.
5. A confirmed `reload` drops the SSH session — the pane reports it cleanly.

> Blocked on read-only devices by design. Sessions idle-timeout after 5 minutes.

---

## 6. AI assistant (Chat with AI)

Modes combine freely:

- 📚 **RAG** — grounds answers in your local corpus (`rag-corpus/`).
- ☁️ **Cloud Claude** (Haiku/Sonnet/Opus) **or** local **Ollama** per message.
- ⚖️ **Compare** — same prompt to two models, side-by-side.
- 🤖 **Agentic** — the model calls read-only tools (`run_show_command`,
  `search_corpus`, `describe_device`, …) to investigate before answering; every
  tool call is shown in the trace.

Cost + token usage is reported on each response. All agent activity is appended to
`~/.hitech_automation_ai/audit.log`.

---

## 7. Devices panel

Lists inventory with host, ports, type, **R/W** (read-only vs writable), and whether
a password is configured. Edit `~/.hitech_automation_ai/devices.yaml` then click
**↻ Reload** — no restart needed for inventory changes.

---

## 8. Common commands

```bash
# restart after deploying a new build
systemctl --user restart hitech_automation_ai

# watch logs live
journalctl --user -u hitech_automation_ai -f

# rebuild the RAG index after changing rag-corpus/
cd build && python rag_builder.py --rebuild && systemctl --user restart hitech_automation_ai
```
