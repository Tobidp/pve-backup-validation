# Proxmox Backup Validation

Automated **test restores** of Proxmox Backup Server (PBS) backups into isolated
VMs. For each VM it restores the latest backup, boots it on an isolated bridge,
validates **boot → systemd services → TCP ports → HTTP endpoints**, sends a
Telegram report, and destroys the temporary VM.

Service checks (TCP/HTTP) run **inside the guest** via the QEMU guest agent, so
they work even on a fully isolated bridge with no route and no DHCP.

> Designed for clusters **without shared storage** (no Ceph): restores pull from
> PBS, so a single "test node" can validate VMs from any node in the cluster.

## Features

- 🔁 Latest-snapshot test restore per VM, then automatic teardown
- 🧪 Validates boot, systemd units, listening ports and HTTP endpoints
- 🔍 Service **auto-discovery** from a signature library (+ manual overrides)
- 🔒 Isolated bridge, firewall disabled on the test NIC, NIC config preserved
- 🛡️ Single-run lock (`flock`) and cleanup on interruption (`INT`/`TERM`)
- 📨 Telegram notifications (success / failure / summary)
- 🗂️ Per-run, per-VM log files

## Requirements

**On the PVE node (where the script runs):**

```bash
which qm qmrestore pvesh python3 flock curl
apt install -y curl    # if curl is missing (used for Telegram)
```

**On each Linux guest to be validated:**

```bash
apt install -y qemu-guest-agent
systemctl enable --now qemu-guest-agent
# For HTTP checks, the guest also needs curl or wget (optional)
```

On the PVE node, make sure the agent is enabled in the VM config:

```bash
qm set <VMID> --agent enabled=1
```

## 1. Create the isolated bridge

Edit `/etc/network/interfaces`:

```
auto vmbr99
iface vmbr99 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    # No gateway, no route — fully isolated
```

Apply and verify:

```bash
ifreload -a
ip link show vmbr99
```

> **DHCP is NOT required.** Since checks run inside the guest, the test VM does
> not need an IP on the isolated bridge. The guest IP, when available, is only
> shown in the report ("Test IP"). If you *want* an IP shown, you can optionally
> run dnsmasq on `vmbr99` — but it is purely cosmetic.

## 2. Install the script and config

The script reads everything from `/root` by default:

```bash
cp backup_validation.sh /root/
chmod +x /root/backup_validation.sh

cp backup_validation.conf.example /root/backup_validation.conf   # edit it
cp backup_validation.env          /root/backup_validation.env    # edit it
chmod 600 /root/backup_validation.env                            # contains secrets
```

> Save all files with **LF** (Unix) line endings. CRLF is tolerated for the
> `.conf`/`.env`, but the `.sh` must be LF or the shebang breaks.

## 3. Configure secrets (Telegram)

Secrets and per-node overrides live in `backup_validation.env` (sourced at
runtime), **not** in the script:

```bash
TELEGRAM_TOKEN="your_token_here"
TELEGRAM_CHAT_ID="your_chat_id_here"
```

Any default from the script can be overridden here (storage names, timeouts,
paths, bridge, etc.). Leave Telegram empty to disable notifications.

## 4. Configure the VM list

Edit `/root/backup_validation.conf`. One VM per line, `;`-separated:

```
# vmid ; mode   ; overrides
105    ; auto   ;
110    ; hybrid ; cloudflared:0
210    ; manual ; myapp:8080
```

## 5. Run

```bash
# Full cycle (all VMs in the config file)
/root/backup_validation.sh

# One-off test of a single VM
/root/backup_validation.sh --vmid 105
/root/backup_validation.sh --vmid 105 --mode hybrid --overrides "cloudflared:0"
/root/backup_validation.sh --vmid 105 --mode manual --overrides "myapp:8080"
```

Logs are written to `/var/log/backup_validation/<MM-DD-YY_HHhMM>/`:

```
/var/log/backup_validation/06-09-26_02h00/
  ├── _cycle.log        # cycle-level: pre-checks, summary
  ├── 105/105.log       # per-VM test log
  └── 110/110.log
```

## 6. Schedule via cron

```bash
# Weekly, Sundays at 02:00
cat > /etc/cron.d/backup-validation <<'EOF'
0 2 * * 0 root /root/backup_validation.sh
EOF
```

## Operating modes

| Mode | Use | Example line |
|------|-----|--------------|
| `auto` | Default services from the signature library | `100;auto;` |
| `hybrid` | Auto-discovery + custom services | `101;hybrid;api:8080` |
| `manual` | Custom services only | `102;manual;myapp:9000` |

Override format: `service:port,service:port`. Use `port 0` to skip the port/HTTP
check and validate only the systemd unit (e.g. `cloudflared:0`).

## Built-in signature library

| Service | Check |
|---------|-------|
| Apache (`apache2`/`httpd`) | HTTP/80 |
| Nginx | HTTP/80 |
| PostgreSQL | TCP/5432 |
| MariaDB/MySQL | TCP/3306 |
| Redis | TCP/6379 |
| MongoDB | TCP/27017 |
| ClickHouse | TCP/9000 |
| Docker | systemd only |
| Elasticsearch | HTTP/9200 |
| RabbitMQ | TCP/5672 |
| cloudflared | systemd + log scan |

## Troubleshooting

**Guest agent not responding**
- Check `systemctl status qemu-guest-agent` inside the VM
- Confirm `qm set <VMID> --agent enabled=1` on the original VM

**HTTP check fails but the service is up**
- The check runs against `127.0.0.1` inside the guest. If the service binds to a
  specific interface (not `0.0.0.0`/localhost), use a `tcp` override instead —
  the TCP check uses `ss -ltn` and matches any listen address.

**Temporary VMID busy**
- Look for leftovers: `qm list | grep '^ *900'`
- Clean up manually: `qm destroy <VMID> --purge 1`

**Restore fails**
- Check destination storage: `pvesm status`
- Check free space: `df -h`

**Another run in progress**
- A `flock` prevents overlapping runs. If a previous run is still going (long
  restore), the new one aborts. Lock file: `/var/lock/backup_validation.lock`.

## Author

**Tobias Pandolfo** ([@Tobidp](https://github.com/Tobidp)) — [LinkedIn](https://www.linkedin.com/in/tobiaspandolfo/)

## License

[MIT](LICENSE) © 2026 Tobias Pandolfo.
Free to use and modify, as long as the original copyright notice is kept.
