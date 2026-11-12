## hostmand - Hostname Manager 

A simple Bash CLI tool + daemon to save, manage, and connect to all your server hostnames.

Built for Bastion / Jump Server workflows and Red Hat-style labs, where you access private servers through a single bastion host and you don't want to memorize IP addresses anymore.

**`hostmand` survives server reboot.** The binary lives in `/usr/local/bin`, data lives in a permanent CSV file, and a systemd service re-generates your SSH config automatically on every boot.

**v2 adds:** automatic backup, bulk add from text file, custom SSH port per host (hidden from list), and sync across bastion / workstation.

Instead of typing this every time:


#### Features

- Save hostnames with IP, SSH user, description, bastion info, and **custom SSH port**
- List hostnames as `IP ADDRESS` -> `HOSTNAME` only (port hidden for clean reading)
- Search hostnames by keyword (shows port)
- Connect via SSH with automatic `ProxyJump` through bastion (with bastion port support)
- Ping by hostname instead of IP
- Export all hosts to `~/.ssh/config` (includes `Port` line)
- Import from `/etc/hosts`
- **Automatic backup** before every change, keep last 20 backups, easy restore
- **Bulk add** many hosts from a text file: `hostmand add-bulk hosts.txt`
- **Sync** `hosts.csv` across bastion and workstation: `hostmand sync workstation`
- **Daemon mode** via systemd service `hostmand.service`, auto-run on boot
- **Persistent after reboot**
- No dependencies, pure Bash
- Data stored in a simple CSV file, easy to backup and edit

#### Requirements

- Bash
- `ssh`, `ssh-copy-id`, `ping`, `scp`
- `systemd` (for daemon / auto-start after reboot, AlmaLinux / Rocky / RHEL / Ubuntu)
- Linux or macOS (systemd part is Linux only)
- No Python, no Docker, no database needed

#### Installation

If you just want the CLI without daemon:

```bash
chmod +x hostmand.sh
sudo cp hostmand.sh /usr/local/bin/hostmand
hostmand
```

Integration to daemon:

```bash
sudo cp hostmand.service /etc/systemd/system/hostmand.service
sudo systemctl daemon-reload
sudo systemctl enable hostmand
sudo systemctl start hostmand
```

#### Verify installation

```bash
hostmand
systemctl status hostmand
```

## Quick Start

#### Add your first hosts (with custom port)

```bash
hostmand add bastion localhost student "Lab entry point" - 2222
hostmand add workstation 10.0.0.10 student "Exam workstation" bastion 22
hostmand add node1 10.0.0.11 student "Node 1 RHCSA" bastion 22
hostmand add node2 10.0.0.12 student "Node 2 RHCSA" bastion 22
hostmand add web1 192.168.1.50 ubuntu "Web server" - 2222
```

Format:

```
hostmand add <name> <ip> [user] [description] [bastion] [port]
```

Interactive mode is also supported. Just run:

```bash
hostmand add
```

And answer the prompts one by one, including port.

#### List hosts - IP and Hostname

```bash
hostmand list
```

Output:

```
Saved Hostnames List:
IP ADDRESS           HOSTNAME
----------------------------------------
localhost            bastion
10.0.0.10            workstation
10.0.0.11            node1
10.0.0.12            node2
10.0.0.20            db-server
----------------------------------------
Total: 5 host
```

The list shows **IP ADDRESS first, then HOSTNAME only**. Port is hidden from list for clean reading, but still saved and used for `connect` / `export-ssh`. To see port, use `hostmand search <name>` or check `~/.ssh/config` after export.

#### Connect

```bash
hostmand connect node1
hostmand ssh node1
hostmand go node1
```

#### Export to SSH config

Generate `~/.ssh/config` from your database, so you can use native `ssh` directly:

```bash
hostmand export-ssh
```

After that:

```bash
ssh node1
ssh web1
```

Example generated config with custom port:

```
Host web1
  HostName 192.168.1.50
  User ubuntu
  Port 2222

Host node1
  HostName 10.0.0.11
  User student
  Port 22
  ProxyJump student@localhost:2222
```

Your old `~/.ssh/config` is backed up to `~/.ssh/config.bak`.

---

## All Commands

| Command | Description |
| --- | --- |
| `hostmand add <name> <ip> [user] [description] [bastion] [port]` | Add a new host with custom port |
| `hostmand list` | List all hosts (IP ADDRESS, HOSTNAME only) |
| `hostmand search <keyword>` | Search hosts by keyword (shows port) |
| `hostmand connect <name>` | SSH to host, auto via bastion + port |
| `hostmand ping <name>` | Ping host by name |
| `hostmand del <name>` | Delete a host (auto-backup first) |
| `hostmand edit` | Edit database CSV in nano |
| `hostmand export-ssh` | Generate `~/.ssh/config` from database |
| `hostmand import-hosts` | Import hosts from `/etc/hosts` |
| `hostmand backup` | Create backup now |
| `hostmand backup list` | List all backups |
| `hostmand backup restore <file>` | Restore from backup |
| `hostmand add-bulk <file>` | Add many hosts from text file |
| `hostmand sync <target>` | Push hosts.csv to target |
| `hostmand sync --pull <target>` | Pull hosts.csv from target |
| `hostmand sync --check <target>` | Compare local vs remote |
| `hostmand daemon` | Run daemon manually (check + regenerate SSH config) |
| `hostmand clear` | Delete ALL hosts (auto-backup first) |
| `hostmand help` | Show help menu |

---

#### Run in Daemon

`hostmand` has 2 ways to run as daemon:

#### Via systemd (recommended, auto-run on boot)

```bash
# Install + enable daemon
sudo bash install-hostmand.sh

# Or manual:
sudo cp hostmand.sh /usr/local/bin/hostmand
chmod +x /usr/local/bin/hostmand
sudo cp hostmand.service /etc/systemd/system/hostmand.service
sudo systemctl daemon-reload
sudo systemctl enable hostmand
sudo systemctl start hostmand
```

#### Run daemon manually (without reboot)

```bash
hostmand daemon
```

Output:

```
hostmand daemon: data is persistent in /home/student/.hostname_manager/hosts.csv
Checking hosts...
Saved Hostnames List:
IP ADDRESS           HOSTNAME
----------------------------------------
localhost            bastion
10.0.0.10            workstation
10.0.0.11            node1
10.0.0.12            node2
----------------------------------------
Total: 4 host

Regenerating SSH config...
Generating ~/.ssh/config from database...
Success! Now you can directly use: ssh node1
hostmand is ready after reboot.
```

#### Check daemon status

```bash
systemctl status hostmand
systemctl is-enabled hostmand
journalctl -u hostmand -f
cat /var/log/hostmand.log
```

Expected status:

```
● hostmand.service - hostmand - Hostname Manager Daemon
     Loaded: loaded (/etc/systemd/system/hostmand.service; enabled)
     Active: active (exited)
```

`active (exited)` is normal. Daemon is `Type=oneshot`, it runs once on boot, regenerates `~/.ssh/config`, then stays standby. It is not an error.

#### Restart daemon

```bash
sudo systemctl restart hostmand
sudo systemctl status hostmand
```

#### Stop / disable daemon

```bash
sudo systemctl stop hostmand
sudo systemctl disable hostmand
```

CLI `hostmand` will still work, but SSH config will not auto-regenerate on boot. You will need to run `hostmand export-ssh` manually.

#### What daemon does on every boot

1. Checks that `hosts.csv` exists
2. Runs `hostmand list` (IP + Hostname only)
3. Runs `hostmand export-ssh` to regenerate `~/.ssh/config` with correct ports
4. Writes log to `/var/log/hostmand.log` and `journalctl`

So after reboot, `ssh node1` works immediately without manual export.

---

## Data Format

All data is stored in:

```
~/.hostname_manager/hosts.csv
```

Format in v2 (6 columns):

```csv
name,ip,user,description,bastion,port
bastion,localhost,student,Lab entry point,-,2222
workstation,10.0.0.10,student,Exam workstation,bastion,22
node1,10.0.0.11,student,Node 1 RHCSA,bastion,22
node2,10.0.0.12,student,Node 2 RHCSA,bastion,22
db-server,10.0.0.20,root,Database server,bastion,2222
web1,192.168.1.50,ubuntu,Web server,-,2222
```

If you have an old v1 file with 5 columns (`name,ip,user,description,bastion`), `hostmand` auto-migrates it on first run by adding `,22` to each line. No manual migration needed.

---

#### Automatic Backup

Every mutating command auto-creates a backup before changing data: `add`, `del`, `clear`, `add-bulk`, `sync`, `import-hosts`, `restore`.

Backups are stored in:

```
~/.hostname_manager/backups/
  hosts-20261113-143000-before-add-node1.csv
  hosts-20261113-143500-before-del-node2.csv
  hosts-20261113-144000-manual.csv
```

Only the last 20 backups are kept, older ones are auto-deleted.

#### Manual backup

```bash
hostmand backup
```

#### List backups

```bash
hostmand backup list
```

#### Restore

```bash
hostmand backup restore hosts-20261113-143000-before-del-node1.csv
```

This will backup current file first, copy backup to `hosts.csv`, show `hostmand list`.

---

#### Bulk Add

Create `hosts.txt`:

```txt
# Format: name,ip,user,description,bastion,port
node1,10.0.0.11,student,RHCSA Lab,bastion,22
node2,10.0.0.12,student,RHCSA Lab,bastion,22
db1,10.0.0.20,root,Database server,bastion,2222
web1,192.168.1.50,ubuntu,Web server,-,2222
```

Run:

```bash
hostmand add-bulk hosts.txt
```

Duplicates are skipped, auto-backup created before bulk add.

---

#### Custom SSH Port Per Host

Each host can have its own SSH port.

```bash
hostmand add bastion localhost student "Lab entry point" - 2222
hostmand add db1 10.0.0.20 root "DB server" bastion 2222
```

- `hostmand list` hides port for clean reading
- `hostmand search db1` shows port: `db1 -> 10.0.0.20:2222`
- `hostmand connect db1` runs: `ssh -p 2222 -J student@localhost:2222 root@10.0.0.20`
- `hostmand export-ssh` generates `Port 2222` in `~/.ssh/config`

If you omit port, it defaults to `22`.

---

#### Sync

```bash
hostmand sync workstation
hostmand sync --pull workstation
hostmand sync --check workstation
```

Flow for push:
1. Backup local + backup remote
2. `scp` local `hosts.csv` to target via ProxyJump + port
3. On target, run `hostmand daemon` to regenerate its SSH config

---

## How Persistence Works After Reboot

#### Binary in `/usr/local/bin`

```bash
/usr/local/bin/hostmand
```

Not cleared on reboot.

#### Data in permanent file

```
~/.hostname_manager/hosts.csv
~/.hostname_manager/backups/
```

#### systemd service `hostmand.service` (daemon)

```ini
[Unit]
Description=hostmand - Hostname Manager Daemon
After=network.target network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/hostmand daemon
ExecStartPost=/bin/bash -c 'echo "[hostmand] started at $(date)" >> /var/log/hostmand.log'

[Install]
WantedBy=multi-user.target
```

---

#### Reboot Test

```bash
# Before reboot
hostmand add node1 10.0.0.11 student "Node 1 RHCSA" bastion 22
hostmand backup
hostmand list

# Reboot
sudo reboot

# After reboot
which hostmand
systemctl status hostmand
hostmand list
cat ~/.ssh/config
ssh node1
hostmand daemon
```

Expected after reboot:

```
Saved Hostnames List:
IP ADDRESS           HOSTNAME
----------------------------------------
10.0.0.11            node1
----------------------------------------
Total: 1 host
```

---

#### Bastion Workflow Example

```
Your Laptop
    |
    | ssh -p 2222 to localhost
    v
[ BASTION ]  <- only public entry point, daemon runs here
    |
    | ssh to 10.0.0.x
    v
[ NODE1 ] [ NODE2 ] [ WORKSTATION ]  <- private servers
```

```bash
hostmand add bastion localhost student "Lab entry point" - 2222
hostmand add node1 10.0.0.11 student "Node 1" bastion 22
hostmand connect node1
hostmand sync workstation
hostmand daemon
systemctl status hostmand
```

---

## Troubleshooting

#### Data lost after reboot

```bash
ls -l /usr/local/bin/hostmand
ls -l ~/.hostname_manager/hosts.csv
hostmand backup list
hostmand backup restore <file>
```

#### Uninstall

```bash
sudo systemctl disable hostmand
sudo systemctl stop hostmand
sudo rm /etc/systemd/system/hostmand.service
sudo systemctl daemon-reload
sudo rm /usr/local/bin/hostmand
rm -rf ~/.hostname_manager
```

#### Support

* [:octocat: Follow me on GitHub](https://github.com/anggrdwjy)
* [🔔 Subscribe me on Youtube](https://www.youtube.com/@anggarda.wijaya)

## License

This project is licensed under the **MIT License**—it is free to use, modify, and distribute, including for commercial purposes, as long as you include the copyright notice and this license.

See the full file at [`LICENSE`](LICENSE).
