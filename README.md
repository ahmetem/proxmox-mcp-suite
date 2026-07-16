# Proxmox MCP Suite — Claude Code plugins for Proxmox VE + PBS

Two token-efficient [MCP](https://modelcontextprotocol.io/) servers that let
Claude Code (or any MCP client) run a **Proxmox VE** host and its **Proxmox
Backup Server** through their REST APIs — installable as Claude Code plugins in
a couple of commands.

| Plugin | What it does |
|---|---|
| **`proxmox-ve`** | 52 action-based tools for Proxmox VE: guest lifecycle (create / clone / power / resize), snapshots, backups, storage, disks / LVM / ZFS, guest / host / LXC exec, task forensics. |
| **`proxmox-backup`** | Proxmox Backup Server: datastores, snapshots, garbage collection, verify, prune — read-only by default. |

Both authenticate with **API tokens** (never a root password), gate every
state-changing action behind `confirm=true` (and destructive ones behind
`i_understand_data_loss=true`), and keep output compact so they sip context
rather than flooding it.

## Why these

Compared to other Proxmox MCP servers, this pair optimizes for **running inside
an LLM's context**:

| | This suite | Typical Proxmox MCP |
|---|---|---|
| Output | Compact + length-capped JSON, one-call `health_overview`, list filters, `fields` projection | Verbose JSON dumps |
| Task launches | `wait_seconds` polls the task and returns the result inline | Separate follow-up status call |
| Safety | Two-tier `confirm` + `i_understand_data_loss`; `dry_run` previews on high-consequence mutations | Often single-flag or none |
| Audit | Host/guest SSH exec appended to a **hash-chained, tamper-evident** log (`proxmox_audit_verify`) | Plain or no audit |
| PVE depth | ZFS/LVM/disk-prepare/storage-provisioning/task-forensics/self-heal | VM/CT lifecycle only |

> The two servers are intentionally separate plugins: install only Proxmox VE,
> only PBS, or both.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- [`uv`](https://docs.astral.sh/uv/) (provides `uvx`) — the plugins run each MCP
  server with `uvx`, which fetches and isolates it automatically. Install once:
  ```bash
  # macOS / Linux
  curl -LsSf https://astral.sh/uv/install.sh | sh
  # Windows (PowerShell)
  powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
  ```
- A Proxmox VE host (and/or PBS) reachable over HTTPS, with an API token.

## Install

```
/plugin marketplace add ahmetem/proxmox-mcp-suite
/plugin install proxmox-ve@proxmox-mcp-suite
/plugin install proxmox-backup@proxmox-mcp-suite
```

Install just one if you only need one. Then set the environment variables (see
below) and restart Claude Code.

## Configure

The plugins pass configuration to the MCP servers via environment variables, so
export them in the shell that launches Claude Code (no secret is stored in the
plugin).

### Proxmox VE (`proxmox-ve`)

Required:

| Variable | Example | Notes |
|---|---|---|
| `PROXMOX_HOST` | `pve.example.com` | IP or hostname of the PVE host |
| `PROXMOX_USER` | `root@pam` | User owning the API token |
| `PROXMOX_TOKEN_NAME` | `mcp-server` | Token ID |
| `PROXMOX_TOKEN_VALUE` | `xxxxxxxx-…` | Token secret (UUID) |

Optional (export only to override defaults): `PROXMOX_PORT` (8006),
`PROXMOX_VERIFY_SSL` (false), `PROXMOX_TIMEOUT` (30), and the SSH-backed
tools' `PROXMOX_SSH_HOST` / `PROXMOX_SSH_USER` / `PROXMOX_SSH_KEY_PATH` /
`PROXMOX_SSH_KNOWN_HOSTS`. See the
[server repo](https://github.com/ahmetem/homelab-proxmox-mcp) for the full list.

### Proxmox Backup Server (`proxmox-backup`)

Required:

| Variable | Example | Notes |
|---|---|---|
| `PBS_HOST` | `https://pbs.example.com:8007` | Full PBS REST URL |
| `PBS_TOKEN_ID` | `root@pam!mcp` | API token id |
| `PBS_TOKEN_SECRET` | `…` | Token secret (UUID) |
| `PBS_NODE` | `pbs` | Node name as it appears in task UPIDs |

Optional: `PBS_VERIFY_TLS` (false), `PBS_DEFAULT_DATASTORE`,
`PBS_HTTP_TIMEOUT` (30), and **`PBS_ALLOW_WRITE`** (false — the write-side
tools like `run_gc`, `prune`, `forget_snapshot` refuse to run until this is
`true`). See the [PBS server repo](https://github.com/ahmetem/homelab-pbs-mcp).

## Notes

- **Source:** the plugins run the servers straight from their Git repos via
  `uvx --from git+https://…`. If you'd rather use PyPI, publish the packages
  (`proxmox-mcp`, `pbs-mcp`) and change each plugin's `args` to just the
  package name (e.g. `["proxmox-mcp"]`).
- **Verify a server appears:** run `/mcp` after installing.
- **Security:** keep the Proxmox / PBS APIs on a trusted LAN or behind a VPN;
  privilege-separate the API tokens.

## License

The MCP servers are GPL-3.0-or-later (see each server repo). This marketplace
metadata is provided under the same terms.
