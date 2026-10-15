# infra-ansible

Ansible playbooks for bootstrapping and configuring uye servers.

## What's included

| Role | Purpose |
|---|---|
| `common` | Base packages, timezone, hostname |
| `users` | User accounts, SSH keys, sudo |
| `ufw` | Firewall rules (default deny-in) |
| `docker` | Docker CE + Compose plugin |
| `git` | Clone and keep in sync git repositories (public HTTPS + private via GitHub App) |
| `caddy` | Caddy web server (reverse proxy, HTTPS via ACME) |
| `edge` | Prepare signed NGINX edge, Cloudflare DNS certificate and explicit Caddy cutover |
| `wireguard` | WireGuard VPN server |
| `filebrowser` | [FileBrowser](https://filebrowser.xyz) web UI |
| `storage` | Mount block devices, persist in `/etc/fstab` |
| `runner` | Self-hosted GitHub Actions runner (ghrunner user, AppArmor, iptables egress policy) |

## Quick start

```bash
# 1. Install required Ansible collections
ansible-galaxy collection install -r requirements.yml

# 2. Edit inventory
cp inventory/hosts.yml inventory/hosts.yml  # update IPs and users

# 3. Set variables
# group_vars/all.yml      — global defaults
# group_vars/<group>.yml  — per-group overrides
# host_vars/<host>.yml    — per-host overrides

# 4. Dry-run first
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --check --diff

# 5. Apply
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml
```

## Requirements

- Ansible >= 2.14
- Python 3.9+
- Collections: `ansible.posix`, `community.general` (installed via `requirements.yml`)

## Secrets

Secrets are never committed. Two patterns are used:

- **WireGuard keys** — generated locally by `playbooks/keygen.yml` and stored in `secrets/wireguard.yml` (gitignored). Back this file up externally; losing it requires regenerating all keys and re-importing configs on every device.
- **Everything else** — pulled at runtime via a lookup plugin pointed at your secrets manager, or from the local `.env` file. Each role's `defaults/main.yml` marks secret variables with the expected lookup pattern.

---

## Bootstrapping a fresh Ubuntu server

A complete walkthrough from zero to a hardened, SSH-accessible server.

### 1. Prerequisites

On your **control machine** (laptop / workstation):

- Ansible ≥ 2.14: `pip install ansible`
- Python 3.9+
- This repository cloned locally

### 2. Set up SSH access

**a. Generate a key pair** (skip if you already have one)

```bash
ssh-keygen -t ed25519 -C "you@example.com"
# Creates ~/.ssh/id_ed25519 (private) and ~/.ssh/id_ed25519.pub (public)
```

**b. Install the public key on the server**

*Option A — cloud provider console (recommended):* When creating the server on DigitalOcean / Hetzner / AWS / etc., paste the contents of `~/.ssh/id_ed25519.pub` into the SSH key field. The provider writes it into `root`'s `authorized_keys` automatically.

*Option B — password-based SSH:* If the server already has a root password you can reach:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<server-ip>
```

**c. Verify the connection before touching Ansible**

```bash
ssh -i ~/.ssh/id_ed25519 root@<server-ip>
# Accept the host fingerprint when prompted, then exit
```

**d. Grant passwordless sudo** (skip if connecting as `root`)

Ansible uses `become: true` but cannot supply a sudo password interactively over SSH — it will time out. Do this once via an interactive session before running any playbook:

```bash
ssh -i ~/.ssh/id_ed25519 <user>@<server-ip>
echo "<user> ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/<user>
sudo chmod 440 /etc/sudoers.d/<user>
exit
```

The `users` role will maintain this going forward.

### 3. Install Ansible collections

Run once from the repo root:

```bash
ansible-galaxy collection install -r requirements.yml
```

### 4. Update the inventory

Edit `inventory/hosts.yml` and replace the placeholder values:

```yaml
all:
  children:
    servers:
      hosts:
        server1:
          ansible_host: <server-ip>          # your server's IP or hostname
          ansible_user: root                  # or ubuntu / debian depending on image
          ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```

### 5. Configure variables

Open `group_vars/all.yml` and set at minimum:

```yaml
timezone: Europe/Berlin   # or UTC, America/New_York, etc.

users:
  - name: igor
    groups: [sudo]
    sudo: true
    ssh_public_keys:
      - "ssh-ed25519 AAAA..."   # contents of ~/.ssh/id_ed25519.pub
```

The `users` role creates this account, installs the SSH key, and grants passwordless sudo. After `bootstrap.yml` runs you can SSH in as this user instead of root.

To open additional firewall ports, override `ufw_rules` (default is SSH only):

```yaml
ufw_rules:
  - port: "22"
    comment: SSH
  - port: "443"
    proto: tcp
    comment: HTTPS
```

### 6. Verify Ansible connectivity

```bash
ansible -i inventory/hosts.yml servers -m ansible.builtin.ping
```

You should see `"ping": "pong"` for each host.

### 7. Dry-run first

```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --check --diff
```

Review the diff output — no changes are applied in check mode.

### 8. Apply

```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml
```

**What the playbook does:**

| Role | Effect |
|---|---|
| `common` | apt update, installs base packages (curl, git, htop, unzip, vim), sets timezone and hostname |
| `users` | Creates user accounts, installs SSH authorized keys, grants NOPASSWD sudo |
| `ufw` | Installs UFW, sets default-deny inbound / allow outbound, applies your rule list, enables the firewall |

### 9. Next steps

Once the server is bootstrapped, use the role-specific playbooks as needed:

```bash
# Install Docker CE + Compose plugin
ansible-playbook -i inventory/hosts.yml playbooks/docker.yml

# Clone / pull git repositories (configure git_repos in host_vars first)
ansible-playbook -i inventory/hosts.yml playbooks/git.yml

# Deploy Caddy reverse proxy (configure caddy_sites_config in host_vars first)
ansible-playbook -i inventory/hosts.yml playbooks/caddy.yml

# Set up WireGuard VPN (run keygen.yml first — see "Setting up WireGuard VPN" below)
ansible-playbook -i inventory/hosts.yml playbooks/wireguard.yml

# Deploy FileBrowser web UI (requires filebrowser_admin_password secret)
ansible-playbook -i inventory/hosts.yml playbooks/filebrowser.yml

# Mount block devices (configure storage_mounts in host_vars first)
ansible-playbook -i inventory/hosts.yml playbooks/storage.yml

# Prepare the server for the self-hosted GitHub Actions runner (set RUNNER_GITHUB_TOKEN first)
ansible-playbook -i inventory/hosts.yml playbooks/runner.yml

# Run everything at once (requires secrets/wireguard.yml)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

### Setting up the GitHub App for private repos

GitHub Apps generate short-lived tokens per run — no separate GitHub account, fine-grained permissions, and clean audit logs.

**1. Install Python dependencies on your control machine**

```bash
pip install PyJWT cryptography
```

> **macOS only:** Python on macOS doesn't trust the system certificate store, which causes `SSL: CERTIFICATE_VERIFY_FAILED` when the lookup calls the GitHub API. Fix it once by running the certificate installer that ships with Python:
> ```bash
> open "/Applications/Python 3.x/Install Certificates.command"
> ```
> Replace `3.x` with your Python version (`python3 --version`). If you're using Homebrew Python, use `certifi` instead:
> ```bash
> pip install --upgrade certifi
> export SSL_CERT_FILE=$(python3 -c "import certifi; print(certifi.where())")
> ```

**2. Create a GitHub App**

Go to your org on GitHub → Settings → Developer settings → GitHub Apps → New GitHub App.

Fill in:
- **Name**: `uye-deploy` (or similar)
- **Homepage URL**: anything (e.g. your org URL)
- **Webhook**: uncheck "Active" — not needed
- **Permissions**: Contents → Read-only (under Repository permissions)

Click "Create GitHub App". Note the **App ID** shown on the settings page.

**3. Generate a private key**

On the App settings page → "Private keys" → Generate a private key. A `.pem` file downloads automatically. Move it to your control machine:

```bash
mv ~/Downloads/uye-deploy.*.private-key.pem ~/.ssh/github-app-uye.pem
chmod 600 ~/.ssh/github-app-uye.pem
```

**4. Install the App on your repos**

On the App settings page → "Install App" → install on your org → select the private repos that need access.

**5. Find the Installation ID**

After installing, go to your org → Settings → Installed GitHub Apps → click "Configure" next to your App. The URL will look like:

```
https://github.com/organizations/<org>/settings/installations/12345678
```

The number at the end is the **Installation ID**.

**6. Configure `host_vars/<host>.yml`**

```yaml
git_github_app:
  app_id: "123456"           # from step 2
  installation_id: "12345678"  # from step 5
  key_path: ~/.ssh/github-app-uye.pem  # from step 3

git_repos:
  # Public repo — no token needed
  - repo: https://github.com/org/public-app.git
    dest: /opt/public-app
    version: main

  # Private repo — token injected at runtime
  - repo: "https://x-access-token:{{ git_app_token }}@github.com/org/private-app.git"
    dest: /opt/private-app
    version: main
    user: rwrotson
```

Ansible generates a fresh token each run from your PEM key — nothing is stored on the server.

---

## Setting up Caddy (reverse proxy + HTTPS)

Caddy is installed from its official apt repository and configured via a `Caddyfile`. It automatically obtains and renews TLS certificates from Let's Encrypt.

### 1. Configure variables in `host_vars/<host>.yml`

```yaml
caddy_acme_email: you@example.com   # required — used for Let's Encrypt account

caddy_sites_config: |
  myapp.example.com {
    reverse_proxy localhost:8080
  }

  static.example.com {
    root * /var/www/html
    file_server
  }
```

`caddy_sites_config` is pasted verbatim into the Caddyfile after the global block. Write it as standard Caddyfile syntax.

### 2. Deploy

```bash
ansible-playbook -i inventory/hosts.yml playbooks/caddy.yml
```

This will:

1. Add the Caddy apt repository (signed with the Cloudsmith GPG key)
2. Install `caddy`
3. Write `/etc/caddy/Caddyfile` from the template
4. Open ports 80 and 443 in UFW
5. Enable and start the `caddy` systemd service

Caddy reloads its config gracefully on change — running connections are not dropped.

### Prepared migration to infra-edge

The live Caddy routes are prepared in `infra-edge` for `navidrome.uye.rocks`
(public) and `filebrowser.uye.rocks`, `vault.uye.rocks`, `portainer.uye.rocks`
(VPN `10.0.0.0/24` only). `navidromed.uye.rocks` is retired. Production NGINX
uses host networking so the three loopback backends remain reachable and the
VPN source address is preserved. The unfinished UYE Go Gateway site stays
disabled in production.

These playbooks do nothing until explicitly run. First publish and sign the
`infra-edge` image in GitHub Actions, make its GHCR package public, and ensure
the prepared code is on `infra-edge/main`. Then run:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/edge-prepare.yml
```

This clones `infra-edge`, writes a private Certbot credential using the same
Cloudflare DNS token as Caddy, issues the four-domain certificate via DNS-01,
and schedules renewals. It leaves Caddy and the runner deployer untouched.
The edge `.env` uses `/workspace-edge` Compose paths for the deployer container;
the playbook passes host paths explicitly for certificate issuance. The file is
`ghrunner:root` mode `0640`, readable by the deployer without granting access
to other host users.

At the agreed maintenance window, run:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/edge-cutover.yml
```

The cutover stops Caddy, appends the edge plugin to the runner's existing
`COMPOSE_FILE`, and waits for deployer to verify and start the signed image.
Failure automatically restores Caddy. After a successful cutover,
`/etc/infra-edge-active` prevents `site.yml` and `caddy.yml` from restarting
Caddy. To roll back later:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/edge-rollback.yml
```

Verify HTTPS routing from VPN and a non-VPN client after cutover; the nginx
container healthcheck alone does not prove that every backend answers.

### Key defaults (`roles/caddy/defaults/main.yml`)

| Variable | Default | Description |
|---|---|---|
| `caddy_acme_email` | `""` | Let's Encrypt account email (**required**) |
| `caddy_config_dir` | `/etc/caddy` | Directory for Caddyfile |
| `caddy_log_dir` | `/var/log/caddy` | Log directory |
| `caddy_admin_api` | `localhost:2019` | Caddy admin API endpoint |
| `caddy_open_firewall` | `true` | Open ports 80 and 443 in UFW |

---

## Setting up storage mounts

The `storage` role mounts block devices and persists the entries in `/etc/fstab` using `ansible.posix.mount`.

### Configure in `host_vars/<host>.yml`

```yaml
storage_mounts:
  - src: "UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"   # prefer UUID over /dev/sdX
    path: /data
    fstype: ext4
    opts: defaults
    state: mounted
```

Find the UUID of a block device on the server:

```bash
lsblk -o NAME,UUID,FSTYPE,SIZE,MOUNTPOINT
```

### Deploy

```bash
ansible-playbook -i inventory/hosts.yml playbooks/storage.yml
```

The mount directory is created if it doesn't exist. `state: mounted` both mounts immediately and adds the fstab entry. Use `state: present` to only write fstab without mounting.

---

## Setting up WireGuard VPN

Road-warrior setup: connect a MacBook or Android phone to the home network from anywhere. The server at `homeserver.uye.rocks` (`192.168.1.15`) acts as the VPN gateway. Clients get a tunnel IP in `10.0.0.0/24` and can reach the entire home LAN (`192.168.1.0/24`). DNS is handled by `dnsmasq` running on the server, forwarding to the home router at `192.168.1.1`.

### Architecture

```
[MacBook / Android]  ──── WireGuard tunnel ────►  [uye-home : 10.0.0.1]
      10.0.0.2/3                                         │
                                                    iptables NAT
                                                         │
                                              [Home LAN 192.168.1.0/24]
```

- **Server** `10.0.0.1` — listens on UDP 51820, runs dnsmasq for VPN clients
- **MacBook** `10.0.0.2` — split tunnel: only `10.0.0.0/24` + `192.168.1.0/24` goes through VPN
- **Android** `10.0.0.3` — same split tunnel

### Prerequisites

On your **control machine** (MacBook), install WireGuard tools:

```bash
brew install wireguard-tools
```

### Step 1 — Forward the port on your router

WireGuard needs UDP 51820 to reach the server from the internet. Log in to your router admin panel and add a port forwarding rule:

| Setting | Value |
|---|---|
| Protocol | UDP |
| External port | 51820 |
| Internal IP | 192.168.1.15 |
| Internal port | 51820 |
| Remote Host / Source IP | _(leave blank)_ |

> **Important:** Leave any "Remote Host", "Source IP", or "Source Address" field **blank**. Filling it in restricts forwarding to a single IP, which will silently block connections from mobile networks (where the carrier IP changes constantly).

Save and apply. This only needs to be done once.

### Step 2 — Generate keypairs

Run the keygen playbook on your local machine. It generates server and client keypairs, saves them to `secrets/wireguard.yml` (gitignored), and writes ready-to-import client config files:

```bash
# Dry-run first — validates wg is installed, shows what would be created
ansible-playbook playbooks/keygen.yml --check

# Generate keys for real
ansible-playbook playbooks/keygen.yml
```

> **Note**: `--check` mode skips the key generation shell tasks (they have side effects) but validates that the `wg` binary is present. If it fails with "wg not found", install the tools first: `brew install wireguard-tools`.

Expected output:

```
TASK [Summary] *****
ok: [localhost] => {
    "msg": [
        "Keys written to secrets/wireguard.yml",
        "Client configs written to secrets/clients/",
        "",
        "Next steps:",
        "  1. Forward UDP 51820 on your router to 192.168.1.15",
        "  2. ansible-playbook -i inventory/hosts.yml playbooks/wireguard.yml",
        "  3. Import secrets/clients/macbook.conf into WireGuard app",
        "  4. For Android: qrencode -t ansiutf8 < secrets/clients/android.conf"
    ]
}
```

The `secrets/` directory is gitignored. **Back it up** — if you lose `secrets/wireguard.yml` you will need to regenerate all keys and re-import configs on every device.

> **Idempotent**: re-running `keygen.yml` will NOT overwrite existing keys if `secrets/wireguard.yml` already exists. To rotate all keys, delete `secrets/wireguard.yml` first.

### Step 3 — Deploy to the server

```bash
ansible-playbook -i inventory/hosts.yml playbooks/wireguard.yml
```

This will:

1. Load keys from `secrets/wireguard.yml`
2. Open UDP 51820 in UFW (WireGuard port)
3. Open port 53 (UDP+TCP) in UFW for VPN clients (`10.0.0.0/24`) — required for dnsmasq to be reachable through the tunnel
4. Set `DEFAULT_FORWARD_POLICY="ACCEPT"` in `/etc/default/ufw` so the kernel passes packets between interfaces
5. Install `wireguard` and `dnsmasq` packages
6. Enable IPv4/IPv6 forwarding via sysctl
7. Apply iptables FORWARD (accept from `wg0`) and MASQUERADE (NAT outbound on LAN interface) rules explicitly — these survive UFW resets independently of PostUp
8. Write `/etc/wireguard/wg0.conf` with your peers and NAT rules
9. Clear any previous service failure state, then enable and start `wg-quick@wg0`
10. Configure `dnsmasq` to listen on `10.0.0.1:53`, forwarding DNS to `192.168.1.1`

Verify on the server after the playbook finishes:

```bash
ssh uye-home
sudo wg show
```

You should see the interface up with two peer entries (macbook and android), zero traffic yet.

### Step 4 — Connect from MacBook

**a. Install WireGuard**

Install the GUI app from the [Mac App Store](https://apps.apple.com/app/wireguard/id1451685025). For CLI-only use, `wireguard-tools` is already installed if you followed the Prerequisites step.

**b. Import the config**

```bash
# GUI (App Store app) — double-click or:
open secrets/clients/macbook.conf

# CLI — copy config then bring up the interface
sudo cp secrets/clients/macbook.conf /etc/wireguard/macbook.conf
sudo wg-quick up macbook
```

**c. Activate the tunnel**

GUI: click the tunnel name in the WireGuard menu bar app → Activate.

CLI:

```bash
sudo wg-quick up macbook
```

**d. Verify**

```bash
# Ping the VPN server
ping 10.0.0.1

# Reach a home LAN device
ping 192.168.1.1

# Confirm DNS resolves through the tunnel
dig +short homeserver.uye.rocks @10.0.0.1
```

### Step 5 — Connect from Android

**a. Install WireGuard**

Install [WireGuard from Google Play](https://play.google.com/store/apps/details?id=com.wireguard.android).

**b. Generate a QR code on your Mac**

Install `qrencode` if not present:

```bash
brew install qrencode
```

Print the QR code in the terminal:

```bash
qrencode -t ansiutf8 < secrets/clients/android.conf
```

**c. Scan from the app**

In the WireGuard app on Android: tap **+** → **Scan from QR code** → point the camera at the terminal output.

**d. Activate and verify**

Toggle the tunnel on. Open a browser and navigate to `http://192.168.1.1` (your router admin panel) — if it loads, you're tunneled into the home network.

**e. SSH from Android (Termius)**

To SSH into the server over the VPN from Termius, you need an Android SSH key authorized on the server.

Generate a key in Termux:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_android -C "android"
cat ~/.ssh/id_android.pub
```

Copy the public key output and add it to your `.env` file on the control machine:

```
ANDROID_SSH_PUBLIC_KEY=ssh-ed25519 AAAA...
```

Then re-run the users role to authorize the key on the server:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --tags users
```

In Termius: add a new host with address `10.0.0.1`, port `22`, username `rwrotson`, and import `~/.ssh/id_android` from the Termux keystore. Connect while the WireGuard tunnel is active.

### Adding a new peer

1. Generate a keypair for the new device on your Mac:

   ```bash
   wg genkey | tee /tmp/new_priv | wg pubkey > /tmp/new_pub && wg genpsk > /tmp/new_psk
   cat /tmp/new_priv /tmp/new_pub /tmp/new_psk
   ```

2. Add the keys to `secrets/wireguard.yml` under `wg_peers_keys`:

   ```yaml
   wg_peers_keys:
     macbook: { ... }
     android: { ... }
     ipad:                               # new device
       private_key: "<output of new_priv>"
       public_key:  "<output of new_pub>"
       psk:         "<output of new_psk>"
   ```

3. Add the peer to `inventory/host_vars/uye-home.yml`:

   ```yaml
   wireguard_peers:
     - name: macbook
       ...
     - name: android
       ...
     - name: ipad
       public_key:    "{{ wg_peers_keys.ipad.public_key }}"
       preshared_key: "{{ wg_peers_keys.ipad.psk }}"
       allowed_ips:   "10.0.0.4/32"
       persistent_keepalive: 25
   ```

4. Add the peer entry to `wg_peers` in `playbooks/keygen.yml`:

   ```yaml
   wg_peers:
     - { name: macbook, address: "10.0.0.2/32" }
     - { name: android, address: "10.0.0.3/32" }
     - { name: ipad,    address: "10.0.0.4/32" }   # add this
   ```

5. Regenerate client configs (keys already exist, only the config file is written):

   ```bash
   ansible-playbook playbooks/keygen.yml
   ```

6. Re-deploy the server config:

   ```bash
   ansible-playbook -i inventory/hosts.yml playbooks/wireguard.yml
   ```

   > **Note:** The wireguard role handler restarts `wg-quick@wg0` (not `wg syncconf`), so existing sessions for other peers will briefly drop and reconnect. This ensures PostUp iptables rules are always re-applied.

7. Import `secrets/clients/ipad.conf` on the new device.

### Rotating keys

To replace all keys (e.g. a device was lost):

```bash
rm secrets/wireguard.yml secrets/clients/*.conf
ansible-playbook playbooks/keygen.yml
ansible-playbook -i inventory/hosts.yml playbooks/wireguard.yml
# Re-import client configs on all devices
```

### Troubleshooting

**Handshake never completes (no traffic after activating tunnel)**

- Confirm UDP 51820 is port-forwarded on your router to `192.168.1.15`.
- Check the firewall allows it: `sudo ufw status` — you should see `51820/udp ALLOW`.
- Check WireGuard is listening: `sudo ss -ulnp | grep 51820` on the server.
- On the client: `sudo wg show` — look for `latest handshake` under the peer; if it never appears, the UDP packets are not reaching the server.
- **Mobile networks only:** Check that the router port-forward rule has no "Remote Host" / "Source IP" restriction. A single IP in that field will silently block all connections from other addresses (e.g. mobile carrier IPs). Clear the field so any source IP is allowed.

**Connected but can't reach LAN (192.168.x.x unreachable)**

- Confirm NAT is active: `sudo iptables -t nat -L POSTROUTING -n -v` — should show a MASQUERADE rule for the LAN interface (`eth0` / `ens3`).
- Confirm `FORWARD` rule is present: `sudo iptables -L FORWARD -n` — should show `ACCEPT` for input from `wg0`.
- Confirm `AllowedIPs` in the client config includes `192.168.1.0/24`.
- If the MASQUERADE/FORWARD rules are missing (e.g. after a reboot with non-persistent iptables), re-run `wireguard.yml` — it applies these rules explicitly via `ansible.builtin.iptables` tasks.

**DNS not working through the tunnel (names don't resolve)**

- Confirm dnsmasq is running: `systemctl status dnsmasq` on the server.
- Test directly: `dig @10.0.0.1 google.com` from the client while the tunnel is up.
- Check dnsmasq is bound to the right address: `sudo ss -ulnp | grep 53` — should show `10.0.0.1:53`.
- Confirm UFW allows DNS from the VPN subnet: `sudo ufw status | grep 53` — should show port 53 allowed from `10.0.0.0/24`. If missing, re-run `wireguard.yml`.
- If `systemd-resolved` conflicts, check for port 53 conflicts: `sudo lsof -i UDP:53`.

**Can't SSH from Android / Termius (permission denied)**

The server only authorizes keys listed in `ssh_public_keys` in `group_vars/all.yml`. Android needs its own key:

1. Generate in Termux: `ssh-keygen -t ed25519 -f ~/.ssh/id_android -C "android"`
2. Add `ANDROID_SSH_PUBLIC_KEY=<contents of id_android.pub>` to `.env` on your control machine
3. Run: `ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --tags users`

Then import `~/.ssh/id_android` into Termius and connect to `10.0.0.1:22` (VPN must be active).

**"Connection refused" on 10.0.0.1:22 from Android (SSH / Termius)**

This is usually a transient tunnel state — the WireGuard session is mid-rekey or just reestablishing after a network change (e.g. switching from WiFi to mobile data). Android's kernel temporarily has no route to `10.0.0.1` and returns "Connection refused" rather than a timeout. Wait a few seconds and retry; it recovers on its own.

To confirm the tunnel is actually routing traffic, run on the server while attempting to connect from Android:

```bash
sudo tcpdump -i wg0 -n 'host 10.0.0.3' -c 20
```

If ICMP and TCP packets appear — the tunnel is fine and the connection failure was transient. If nothing appears — the tunnel is down; toggle WireGuard off and on in the Android app to force a new handshake.

Also check that `sudo wg show` shows a recent handshake for the android peer (within the last ~2 minutes).

**`nslookup` in Termux shows `8.8.8.8` even when VPN is active**

Termux has its own static `/etc/resolv.conf` that doesn't follow the Android system DNS set by WireGuard. This means `nslookup` in Termux uses `8.8.8.8` regardless of the VPN state, while `ssh` and other tools use the system resolver (`10.0.0.1` when the tunnel is active). This discrepancy is normal — use `nslookup <host> 10.0.0.1` to explicitly test VPN DNS.

**`keygen.yml` fails with "wg not found"**

```bash
brew install wireguard-tools
```

**Re-running `wireguard.yml` shows no changes but dnsmasq isn't forwarding**

Check upstream DNS reachability from the server: `dig @192.168.1.1 google.com`. If the home router isn't responding to DNS, adjust `wireguard_upstream_dns` in `inventory/host_vars/uye-home.yml`.

**`wireguard.yml` fails with "Could not find the requested service wg-quick@wg0"**

This happens when the `wireguard-tools` package was just installed and systemd hasn't been reloaded yet, or when a previous failed run left the service in an unrecoverable state. The role handles this automatically via `systemctl reset-failed` and `daemon_reload: true`, so re-running the playbook is usually enough. If it persists, reset manually:

```bash
ssh uye-home
sudo systemctl reset-failed wg-quick@wg0
sudo systemctl daemon-reload
```

Then re-run the playbook.

---

## Setting up the self-hosted GitHub Actions runner

The `runner` role prepares a server to host [uye/infra-runner](https://github.com/uye-ltd/infra-runner) — a hardened, ephemeral GitHub Actions runner pool. Each CI job runs in a fresh container that is destroyed on completion. The deployer health endpoint is exposed only on the WireGuard VPN (`10.0.0.0/24`); no runner ports are public.

### What the playbook does

| Step | What happens |
|---|---|
| User | Creates the `ghrunner` service account and adds it to the `docker` group |
| Docker | Preserves the installed Kata runtime registration in daemon.json; runner job networks request explicit subnets |
| AppArmor | Installs and loads the `infra-runner` MAC profile from the cloned repo |
| Egress | Enables `infra-runner-egress.service`, which reapplies the dedicated runner chains after Docker starts/restarts without replacing UFW |
| Repository | Clones `https://github.com/uye-ltd/infra-runner.git` to `/home/ghrunner/infra-runner` |
| `.env` | Templates GitHub App credentials and runner pool config as owner `ghrunner`, group `root`, mode `0640`; the controller generates and auto-refreshes its own installation tokens at runtime |
| Compose override | Templates `docker-compose.override.yml` mounting the GitHub App private key read-only into the controller container |
| UFW | Opens the deployer health port (`8080/tcp`) from the VPN subnet only |

### Prerequisites

The GitHub App (`git_github_app`) must be configured in `host_vars/<host>.yml` with the following **organization-level** permissions:

| Permission | Level |
|---|---|
| Self-hosted runners | Read & write |
| Members | Read |

No PAT is required. The controller generates and refreshes its own short-lived installation tokens using the App credentials.

> **GHCR packages must be public.** GitHub App installation tokens cannot authenticate to
> GHCR regardless of the `Packages: Read` permission. The controller and deployer pull
> images anonymously. If packages are private, use a classic PAT with `read:packages` as
> `GITHUB_TOKEN` in `.env` instead.

### Step 1 — Review host_vars

`inventory/host_vars/uye-home.yml` already has the runner section pre-configured. Verify `git_github_app` is set (it is shared with the `git` role):

```yaml
git_github_app:
  app_id: "123456"
  installation_id: "78901234"
  key_path: /path/to/github-app.pem
```

Override any pool defaults in the same file if needed (see `roles/runner/defaults/main.yml` for the full list):

```yaml
runner_desired_idle: "2"
runner_memory: "6g"
runner_kaniko_size: "2g"
runner_runtime: "kata"  # set only after the server probe passes
runner_cpus: "2"
```

### Step 2 — Run the playbook

```bash
# Dry-run first
ansible-playbook -i inventory/hosts.yml playbooks/runner.yml --check --diff

# Apply
ansible-playbook -i inventory/hosts.yml playbooks/runner.yml
```

The playbook runs in two plays. Play 1 configures Docker; play 2 configures the runner. On an existing production host, review `docker compose config` before applying; the role updates its managed `.env` keys while preserving plugin-specific variables. Changes to `daemon.json` cause the Docker handler to restart Docker; use a maintenance window for those changes. The egress systemd unit itself can be updated without restarting Docker. For a targeted Kata setup and full probe on a server with the verified runner image already present, run `ansible-playbook -i inventory/hosts.yml playbooks/runner-kata.yml`; this targeted playbook leaves Docker and `.env` untouched.

### Step 3 — Start the stack

The playbook prepares the server but does not start the runner stack. Do this manually once as `ghrunner`:

```bash
ssh uye-home
sudo -iu ghrunner
cd ~/infra-runner
docker compose build controller deployer   # build images locally on first deploy
docker compose up -d
```

`controller` and `deployer` are built from source on the first deploy because no images exist in GHCR yet. After `docker compose up -d` the stack is running — stop here. The deployer takes over from this point: every push to `main` rebuilds and signs images, pushes them to GHCR, and the deployer applies updates automatically.

Do **not** run `docker compose pull` manually after the first deploy — the images don't exist in GHCR yet and it will return `403 Forbidden`. Once CI has pushed at least one build, `docker compose pull` can be used to force an immediate update outside the normal poll cycle (packages must be public for this to work with GitHub App auth).

### Step 4 — Verify

```bash
# From a machine on the WireGuard VPN:
curl -s http://10.0.0.1:8080/health | jq .
# → {"status":"ok","ts":"..."}

# Controller logs (structured JSON):
sudo -iu ghrunner
docker compose -f ~/infra-runner/docker-compose.yml logs -f controller | jq -R '. as $line | try fromjson catch $line'
```

| Response | Meaning |
|---|---|
| `200 {"status":"ok"}` | Last poll cycle completed cleanly |
| `503 {"status":"unhealthy"}` | Signature verification failed or `docker compose up` errored |

### Required GitHub org setting

**Settings → Actions → General → Fork pull request workflows →
"Require approval for all outside collaborators"**

This prevents PR workflows from unknown contributors from queuing at all.

### Key defaults (`roles/runner/defaults/main.yml`)

| Variable | Default | Description |
|---|---|---|
| `runner_user` | `ghrunner` | Dedicated service account |
| `runner_repo_dest` | `/home/ghrunner/infra-runner` | Where the runner repo is cloned |
| `runner_subnet` | `10.89.0.0/16` | Explicit job network range used by the egress policy |
| `runner_vpn_subnet` | `10.0.0.0/24` | WireGuard VPN subnet; health port restricted to this |
| `runner_health_port` | `8080` | Deployer health endpoint port |
| `runner_desired_idle` | `2` | Warm idle runners kept ready |
| `runner_memory` | `6g` | Memory limit per runner container |
| `runner_cpus` | `2` | CPU limit per runner container |
| `runner_apparmor_profile` | `infra-runner` | AppArmor profile name applied to every job container |
