# Nebulous Fleet Command Server Infrastructure Guide

## Project Overview

This is a **Nebulous Fleet Command (N:FC) dedicated server management system** running on Ubuntu with remote deployment from Windows using Ansible. The project orchestrates multiple game servers, automated updates, nightly restarts, and monitoring via Grafana/Prometheus/Promtail.

**Deployment Tool**: Ansible (with Jinja2 templates and Ansible Vault for secrets management)

**Status**: Migration from shell/PowerShell scripts → Ansible completed ✓

### Why Ansible?

| Aspect | Before (PowerShell/Bash) | After (Ansible) |
|--------|--------------------------|-----------------|
| **Secrets** | `secrets.txt` (plaintext, gitignored) | `vault.yml` (encrypted, committed safely) |
| **Inventory** | Hardcoded IPs in `run.ps1` | `hosts.yml` (YAML, easily extended) |
| **Config** | Manual placeholder substitution | Jinja2 templates (dynamic, no errors) |
| **Modularity** | Single script | Roles (nds_server, system_config, monitoring) |
| **Multi-server** | Edit script and restart | `deploy-one alpha|bravo` or `deploy-all` (declarative) |
| **Idempotency** | Scripts re-run unsafely | Ansible ensures desired state (safe re-runs) |

### Architecture at a Glance

```
Windows Command Machine (run.ps1)
    ↓ SCP/SSH
┌─────────────────────────────────────┐
│  Ubuntu Server 1 (5.75.190.44)      │
│  ├─ N:FC Server (systemd: nds)      │
│  ├─ Prometheus + Node Exporter      │
│  └─ Promtail (log aggregation)      │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│  Ubuntu Server 2 (37.27.181.203)    │
│  ├─ N:FC Server (systemd: nds)      │
│  ├─ Prometheus + Node Exporter      │
│  └─ Promtail (log aggregation)      │
└─────────────────────────────────────┘
    ↓ metrics/logs
Grafana Cloud (free-tier with dashboards)
```

## Critical Developer Workflows

### Initial Deployment: `run.ps1`

The **master deployment script** (52 lines) orchestrates the entire infrastructure. **Important**: This deploys **2 game servers** (alpha and bravo) that require **manual in-game verification** to confirm successful deployment.

1. **Copies config files** to both servers:
   - `nds-neb2.conf` / `nds-neb3.conf` - N:FC server configurations (XML format, 154 lines)
   - `config.yaml` - Promtail scrape config (log paths and labels)
   - `prometheus.yml` - Prometheus scrape config (metrics endpoints)
   - `crontab` - Scheduled tasks (checkUpdate every 30 min, nightlyRestart at 08:05 UTC)
   - `tryUpdate.conf` - Systemd override (runs `update-nds` before service start)

2. **Creates directories** on remote servers:
   - `/root/promtail/tmp` - Promtail position tracking
   - `/root/prometheus` - Prometheus config volume
   - `/usr/local/lib/systemd/system/nds.service.d/` - Systemd service overrides

3. **Deploys Docker containers** (4 services per server):
   - `nodeexporter` - System metrics collection
   - `prometheus` - Metrics scraping and aggregation
   - `promtail` - Log shipper to Grafana Cloud
   - `nds` service managed by systemd (not Docker, but orchestrated)

4. **Sets permissions**:
   - Makes scripts executable: `chmod +x`
   - Enables steam user to restart services without password via sudoers

### Update & Restart Logic: `checkUpdate.sh` and `nightlyRestart.sh`

Both scripts follow the same pattern:

1. **Check player count** from latest nds.log entry: `grep "players now: " | tail -1`
2. **If empty** (`playersNow -eq 0`): Restart immediately via `systemctl restart nds.service`
3. **If occupied**: Write `ServerCommand.xml` to gracefully schedule restart after current match

**Key insight**: Both scripts clear old command files first (`rm /srv/steam/ServerCommand.xml*`), preventing stale commands from persisting.

**Cron schedule**:
- `checkUpdate.sh` runs every 30 minutes (steam user)
- `nightlyRestart.sh` runs daily at 08:05 UTC (steam user)
- Update check precedes restart by 5 minutes to avoid double-restarts

### Monitoring Stack

**Promtail** (`config.yaml`) scrapes **three log sources**:
- `/var/log/*.log` - System logs (labeled `job: varlogs`)
- `/srv/steam/nds.log` - Game server logs (labeled `job: nebulous`)
- `/srv/steam/log.txt` - Admin/automation logs (labeled `job: adminLog`)

All logs ship to Grafana Cloud's Loki endpoint. **Prometheus** scrapes node exporter metrics and pushes to Grafana Cloud.

**Pre-configured dashboards** available in `grafana/` directory (JSON exports: `Nebulous-*.json`, `Maps played-*.json`).

## Configuration & Secrets Management

### Secret Substitution Pattern

Files use `__VARIABLE_NAME` placeholders:
- `secrets.txt` (gitignored) contains actual values
- `secrets.txt.example` shows required fields
- **Manual substitution needed** before deployment

**Required secrets**:
- `__SERVER_IP1`, `__SERVER_IP2` - Server IPs (hardcoded in run.ps1: 5.75.190.44, 37.27.181.203)
- `__SERVER_ADMIN_PASSWORD` - N:FC server password
- `__PROMETHEUS_URL`, `__PROMETHEUS_USERNAME`, `__PROMETHEUS_PASSWORD` - Grafana Cloud Prometheus endpoint
- `__PROMTAIL_URL` - Grafana Cloud Loki endpoint (already in config.yaml)

**Current state**: Passwords are hardcoded in `prometheus.yml` and `config.yaml` (tokens visible in file). For modifications, preserve token structure.

### Server Configuration Files

- `nds-neb-test.conf` - Test server config (154 lines, XML, with password placeholder)
- `nds-neb2.conf`, `nds-neb3.conf` - Production configs for server 1 & 2 (follow same structure)

Key XML fields:
- `<ServerName>` - Display name in server browser
- `<Password>` - Admin password (`__SERVER_ADMIN_PASSWORD`)
- `<GamePort>` - Must be open TCP in/out (typically 7777)

## Project-Specific Patterns

### File Permissions & Ownership

- Scripts deployed to steam user's home (`/srv/steam/`)
- Sudoers override (`nds_restart_by_steam`) allows steam to restart nds service without password
- Systemd service runs under steam user context

### Graceful Server Management

Instead of hard kills, the system uses:
- **ServerCommand.xml** protocol - N:FC server monitors this file for scheduled restarts
- Logs final player count before graceful shutdown
- Prevents mid-match disruptions by scheduling restarts post-match

### Log Aggregation Strategy

Three separate label streams enable filtering:
- System health (`job: varlogs`)
- Gameplay events (`job: nebulous`)
- Administrative actions (`job: adminLog`)

## Integration Points & External Dependencies

| Component | Version | Purpose | Config File |
|-----------|---------|---------|------------|
| Nebulous Fleet Command | Steam app 2353090 | Game server | nds-neb-*.conf |
| Ubuntu | 24.04 | OS | README.md |
| Docker | Latest | Container runtime | run.ps1 |
| Prometheus | Latest | Metrics | prometheus.yml |
| Node Exporter | Latest | System metrics | run.ps1 |
| Promtail | Latest (main) | Log aggregation | config.yaml |
| Grafana Cloud | Free-tier | Monitoring/dashboards | README.md |

### Cross-Server Communication

- SSH/SCP from Windows to Ubuntu (all commands in run.ps1)
- Systemd service management via SSH
- Docker daemon accessible via SSH
- Log/metrics push to external Grafana Cloud (pull-based via remote_write)

## Common Tasks for AI Agents

1. **Test deployment on bravo only**: Comment out alpha lines in `run.ps1` (lines 8, 10) to deploy to single server during testing
2. **Add a new server**: Update `run.ps1` array and create `nds-neb4.conf`
3. **Change restart schedule**: Edit crontab entries (minute/hour fields)
4. **Add log source**: Update `config.yaml` scrape_configs section
5. **Debug server**: Check `/srv/steam/nds.log` (pulled via SSH in logs)
6. **Modify game settings**: Edit corresponding `nds-neb-*.conf` XML file
7. **Scale monitoring**: Prometheus remote_write already configured for multiple instances
8. **Verify deployment**: After running `run.ps1`, join both game servers in-game to confirm they are responding to commands

## File Structure

```
. (root)
├── run.ps1                 # Master deployment script (Windows)
├── config.yaml             # Promtail config (deployed to /root/promtail)
├── prometheus.yml          # Prometheus config (deployed to /root/prometheus)
├── crontab                 # Systemd cron schedule (deployed to /etc/crontab)
├── checkUpdate.sh          # Hourly update check (deployed to /srv/steam)
├── nightlyRestart.sh       # Daily restart (deployed to /srv/steam)
├── tryUpdate.conf          # Systemd service override (in /usr/local/lib/...)
├── nds_restart_by_steam    # Sudoers file (deployed to /etc/sudoers.d)
├── nds-neb-test.conf       # Test server XML config
├── nds-neb2.conf           # Server 1 XML config
├── nds-neb3.conf           # Server 2 XML config
├── secrets.txt.example     # Template (gitignored secrets.txt with real values)
├── README.md               # Setup instructions
└── grafana/                # Pre-exported dashboard JSONs
    ├── Nebulous-*.json
    └── Maps played-*.json
```

## Key Assumptions for Modifications

1. **Credentials in vault**: `vault.yml` contains encrypted Grafana Cloud tokens. Treat as sensitive.
2. **Hard-coded IPs in inventory**: `inventory/hosts.yml` defines server addresses; modify both inventory AND crontab if servers change.
3. **N:FC server path fixed**: Scripts assume `/srv/steam/` (set by VoodooFan's install script).
4. **Steam user exists**: Cron and sudoers depend on `steam` user account (created by install script).
5. **Systemd service named `nds`**: All restarts target this service name.

---

## Ansible Migration Details

### What Changed from Original PowerShell Approach

**Before**: Single `run.ps1` with hardcoded IPs and manual secret substitution.
**Now**: Ansible playbook with inventory, roles, templates, and encrypted vault.

#### Secrets Management Evolution

1. **Original**: `secrets.txt` (plaintext, gitignored)
   ```
   __SERVER_ADMIN_PASSWORD=password123
   __PROMETHEUS_USERNAME=user
   ```

2. **Migration**: Moved to `inventory/group_vars/nds_servers/vault.yml` (encrypted with Ansible Vault)
   ```bash
   ansible-vault encrypt vault.yml  # Encrypted at rest in git
   ```

3. **Usage**: Retrieved by Ansible during playbook execution
   ```yaml
   vault_server_admin_password: <encrypted>
   vault_prometheus_url: <encrypted>
   vault_promtail_url: <encrypted>
   ```

#### Inventory-Based Configuration

**Before**: Hardcoded in `run.ps1`
```powershell
$ips = @("5.75.190.44", "37.27.181.203")
```

**Now**: YAML-based, easy to extend
```yaml
nds2:
  ansible_host: 5.75.190.44
  server_name: "[EU] Bizarre Adventure #1"
  game_port: 7777
  nds_conf_file: nds-neb2.conf
nds3:
  ansible_host: 37.27.181.203
  server_name: "[EU] Bizarre Adventure #2"
  game_port: 7777
  nds_conf_file: nds-neb3.conf
```

#### Jinja2 Templates Replace Manual Substitution

**Before**: PowerShell script manually replaced `__VARIABLE_NAME` placeholders
**Now**: Jinja2 templates dynamically generate config files

Example (`nds.conf.j2`):
```jinja2
<ServerName>{{ server_name }}</ServerName>
<Password>{{ vault_server_admin_password }}</Password>
<GamePort>{{ game_port }}</GamePort>
```

Benefits:
- No substitution errors
- Per-server customization (different names, ports, etc.)
- Easy to modify without breaking script logic

---

## Ansible Project Structure

### Directory Layout

```
ansible/
├── ansible.cfg                                 # Ansible configuration
├── playbook.yml                                # Main orchestration playbook
├── README.md                                   # Comprehensive Ansible documentation
├── inventory/
│   ├── hosts.yml                               # Server inventory + per-host variables
│   └── group_vars/
│       └── nds_servers/
│           ├── vars.yml                        # Shared variables (paths, ports, schedules)
│           ├── vault.yml                       # Encrypted secrets (generated from template)
│           └── vault.yml.template              # Template for initial vault setup
├── roles/
│   ├── nds_server/
│   │   ├── tasks/main.yml                      # Game server deployment
│   │   ├── handlers/main.yml                   # Service handlers
│   │   └── files/
│   │       ├── checkUpdate.sh                  # Hourly update check
│   │       ├── nightlyRestart.sh               # Daily restart
│   │       ├── nds-neb2.conf                   # Server 1 config backup
│   │       └── nds-neb3.conf                   # Server 2 config backup
│   ├── system_config/
│   │   ├── tasks/main.yml                      # Cron, sudoers, permissions setup
│   │   └── files/
│   │       ├── crontab                         # Cron schedule config
│   │       ├── nds_restart_by_steam            # Sudoers override
│   │       └── tryUpdate.conf                  # Systemd service override
│   └── monitoring/
│       ├── tasks/main.yml                      # Prometheus, Promtail, Node Exporter setup
│       └── files/
│           ├── config.yaml                     # Promtail config backup
│           └── prometheus.yml                  # Prometheus config backup
└── templates/
    ├── nds.conf.j2                             # Game server XML config (Jinja2)
    ├── prometheus.yml.j2                       # Prometheus metrics config
    ├── config.yaml.j2                          # Promtail log aggregation config
    └── crontab.j2                              # Cron schedule
```

### Role Descriptions

#### `nds_server` Role
Manages game server deployment and configuration.
- Deploys N:FC server config from `nds.conf.j2` template
- Conditionally includes server password (can be empty)
- Manages systemd service (nds)
- Handles service restarts via handlers

**Key features**:
- Per-server custom config (different names, ports, maps, mods)
- Graceful restart capability via ServerCommand.xml
- Log file monitoring for player count

#### `system_config` Role
Configures system-level settings (cron, permissions, sudoers).
- Deploys crontab with `checkUpdate.sh` (every 30 min) and `nightlyRestart.sh` (08:05 UTC)
- Sets sudoers rules allowing steam user to restart nds service without password
- Configures systemd service override (`tryUpdate.conf`)
- Creates necessary directories with proper permissions

**Key features**:
- Automated hourly update checks
- Daily nightly restarts at UTC 08:05
- Safe privilege escalation for steam user

#### `monitoring` Role
Deploys monitoring infrastructure (Prometheus, Promtail, Node Exporter).
- Docker containers: node-exporter, prometheus, promtail
- Prometheus scrapes node-exporter metrics and game server logs
- Promtail ships three log streams to Grafana Cloud Loki
- Remote write to Grafana Cloud Prometheus

**Key features**:
- Multi-stream log aggregation (system, game, admin)
- Metrics push to Grafana Cloud
- Pre-configured dashboards (Nebulous, Maps played)

### Shared Variables (`inventory/group_vars/nds_servers/vars.yml`)

```yaml
# Paths
nds_log_path: /srv/steam/nds.log
admin_log_path: /srv/steam/log.txt
game_log_path: /srv/steam/nds.log

# Ports and schedules
game_port: 7777
checkupdate_minute: "*/30"
nightly_restart_hour: "8"
nightly_restart_minute: "5"

# Docker images
docker_images:
  nodeexporter: quay.io/prometheus/node-exporter:latest
  prometheus: prom/prometheus:latest
  promtail: grafana/promtail:main
```

### Vault Variables (`inventory/group_vars/nds_servers/vault.yml` - encrypted)

```yaml
vault_server_admin_password: <encrypted_value>
vault_promtail_url: <encrypted_loki_endpoint>
vault_prometheus_url: <encrypted_prometheus_url>
vault_prometheus_username: <encrypted_user>
vault_prometheus_password: <encrypted_token>
```

---

## Deployment Wrapper: `gameserver.ps1`

PowerShell script that bridges Windows command line to Ansible running in WSL.

### Commands

```powershell
# List inventory and verify setup
.\gameserver.ps1 verify

# Test SSH connectivity to all servers
.\gameserver.ps1 check

# View encrypted vault (read-only)
.\gameserver.ps1 view-vault

# Edit encrypted vault (launches editor in WSL)
.\gameserver.ps1 edit-vault


# Deploy to all servers
.\gameserver.ps1 deploy-all

# Deploy to test server only (maps to bravo)
.\gameserver.ps1 deploy-test

# Deploy to single server (pass alpha/bravo)
.\gameserver.ps1 deploy-one alpha
.\gameserver.ps1 deploy-one bravo

# Dry-run (preview changes without applying)
.\gameserver.ps1 deploy-all -DryRun
```

### Implementation Details

`gameserver.ps1` uses WSL to execute Ansible with vault password from `.vault_pass.txt` (gitignored file in repo root). It handles:
- Vault password file creation in WSL `/tmp/` with proper permissions
- Ansible playbook execution with `--limit` for single-server deployments
- Dry-run mode via `--check` and `--diff` flags
- Error handling and output formatting

---

## Adding a New Server

1. **Update inventory** (`ansible/inventory/hosts.yml`):
   ```yaml
   nds4:
     ansible_host: x.x.x.x
     server_name: "[EU] Bizarre Adventure #3"
     game_port: 7777
     nds_conf_file: nds-neb4.conf
     server_id: neb4
   ```

2. **Create config file** (copy existing and modify):
   ```bash
   cp nds-neb3.conf nds-neb4.conf  # Copy existing config
   # Edit nds-neb4.conf with new server name, port, etc.
   ```

3. **Update shared variables** if needed:
   - New port ranges? Edit `inventory/group_vars/nds_servers/vars.yml`
   - New secrets? Add to `vault.yml`

4. **Add server IP to WSL SSH config** (`~/.ssh/config` inside WSL Ubuntu):
   ```bash
   # Append the new IP to the existing Host line so Ansible can authenticate
   # The config lives in WSL, not Windows - edit it via:
   wsl -d Ubuntu -- bash -c "sed -i 's/Host <existing IPs>/Host <existing IPs> <new IP>/' ~/.ssh/config"
   # Verify:
   wsl -d Ubuntu -- cat ~/.ssh/config
   ```
   Without this step, Ansible will fail with `Permission denied (publickey,password)` on the new host
   even though Windows SSH connects fine, because WSL uses its own `~/.ssh/config` with `hetzner_rsa`.

5. **Run VoodooFan's install script** on the new server before deploying (creates `steam` user, installs SteamCMD):
   ```powershell
   ssh root@<new-ip> "curl -s https://raw.githubusercontent.com/VoodooFan/nebulous/main/install-steam-and-nds.bash | bash"
   ```

6. **Deploy**:
   ```powershell
   .\gameserver.ps1 deploy-all --limit nds4
   ```

---

## Changing Restart Schedule

Edit `inventory/group_vars/nds_servers/vars.yml`:
```yaml
checkupdate_minute: "*/30"     # Every 30 minutes
nightly_restart_hour: "8"       # At 8 AM UTC
nightly_restart_minute: "5"     # At 08:05 UTC
```

Then redeploy:
```powershell
.\gameserver.ps1 deploy-all
```

---

## Modifying Game Server Settings

Edit `ansible/templates/nds.conf.j2` (or per-server config files in `ansible/roles/nds_server/files/`):

```xml
<ServerName>{{ server_name }}</ServerName>
<Password>{{ vault_server_admin_password }}</Password>
<GamePort>{{ game_port }}</GamePort>
<MaxPlayers>50</MaxPlayers>
```

Then redeploy:
```powershell
.\gameserver.ps1 deploy-all
```

---

## Monitoring & Observability

### Prometheus Metrics

Node Exporter metrics scraped every 15 seconds:
- CPU, memory, disk usage
- Network I/O
- System uptime

Pushed to Grafana Cloud Prometheus via remote_write.

### Promtail Logs

Three log streams sent to Grafana Cloud Loki:
1. **System logs** (`job: varlogs`): `/var/log/*.log`
2. **Game logs** (`job: nebulous`): `/srv/steam/nds.log`
3. **Admin logs** (`job: adminLog`): `/srv/steam/log.txt`

### Dashboards

Pre-configured JSON dashboards in `grafana/` directory:
- `Nebulous-*.json`: Server stats, player count, uptime
- `Maps played-*.json`: Game statistics and session history

Import via Grafana UI: Dashboards → Import → Upload JSON file

---

## Vault Password Management

### Interactive Mode (Default)

```powershell
.\gameserver.ps1 deploy-all
# Prompts in WSL: "Vault password:"
```

### Non-Interactive Mode

Create `.vault_pass.txt` in repo root (gitignored):

```powershell
# Windows PowerShell
'your-vault-password' | Out-File -Encoding UTF8 -FilePath .vault_pass.txt
```

Then run without prompts:
```powershell
.\gameserver.ps1 deploy-all
```

### Changing Vault Password

```bash
cd ansible
ansible-vault rekey inventory/group_vars/nds_servers/vault.yml
```

---

## Troubleshooting

### SSH Connection Issues
```powershell
.\gameserver.ps1 check
```
This tests SSH connectivity to all hosts. Check:
- Network connectivity to server IPs
- SSH keys deployed correctly (ansible_user: root expected)
- Firewall rules allow SSH (port 22)

### Vault Password Errors
```
fatal: [nds2]: FAILED! => {"msg": "Vault password not supplied, but vault.yml encrypted"}
```
Create `.vault_pass.txt` or ensure vault password is set in WSL.

### Config Template Errors
Check Jinja2 syntax in `ansible/templates/`:
```bash
wsl -d Ubuntu -- bash -c "cd /mnt/c/.../ansible && python3 -m jinja2.cli templates/nds.conf.j2 --data '{}'"
```

### Ansible Inventory Issues
```powershell
.\gameserver.ps1 verify
```
Lists all servers and variables. Verify `ansible_host`, `game_port`, etc.

---

## Key Assumptions & Design Decisions

1. **Ansible Vault encryption**: Vault passwords encrypted at rest in git via Ansible Vault
2. **Per-server customization**: Inventory allows per-host variables (server names, ports, maps, mods)
3. **Graceful restarts**: ServerCommand.xml protocol prevents mid-match disruptions
4. **Log aggregation**: Three separate streams (system, game, admin) for flexible filtering
5. **Idempotency**: Re-running playbooks safely (idempotent tasks)
6. **WSL execution**: Ansible runs in WSL Ubuntu, controlled from Windows PowerShell
7. **No hardcoded credentials**: All secrets in encrypted vault.yml
8. **Template-based config**: Jinja2 templates prevent manual substitution errors

---

## References

- **N:FC Game Server**: [Steam 2353090](https://store.steampowered.com/app/2353090/)
- **Installation Script**: [VoodooFan's GitHub](https://github.com/VoodooFan/nebulous/blob/main/install-steam-and-nds.bash)
- **Ansible Docs**: [ansible.com/docs](https://docs.ansible.com/)
- **Prometheus**: [prometheus.io](https://prometheus.io/)
- **Grafana Cloud**: [grafana.com/products/cloud](https://grafana.com/products/cloud/)
- **Promtail**: [grafana.com/docs/loki/latest/clients/promtail/](https://grafana.com/docs/loki/latest/clients/promtail/)
