# Cloudflare Tunnel Setup on Ubuntu
## Expose Web and SSH Services Without Open Ports

This guide walks through exposing services on an Ubuntu server to the internet using Cloudflare Tunnel — no port forwarding, no VPN, no exposed firewall rules required.

---

## Prerequisites

- Ubuntu server (20.04+)
- A domain managed by Cloudflare (nameservers pointing to Cloudflare)
- A Cloudflare account with Zero Trust enabled (free tier works)

---

## 1. Install cloudflared on the Server

```bash
# Add Cloudflare's package repo
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update && sudo apt install cloudflared -y
```

---

## 2. Authenticate with Cloudflare

```bash
cloudflared tunnel login
```

This opens a browser window. Select your domain to authorize. A credentials certificate is saved to `~/.cloudflared/cert.pem`.

---

## 3. Create a Tunnel

```bash
cloudflared tunnel create <tunnel-name>
```

Note the **Tunnel ID** from the output — you'll need it in the config. Credentials are saved to:

```
~/.cloudflared/<TUNNEL_ID>.json
```

---

## 4. Create the Config File

```bash
sudo mkdir -p /etc/cloudflared
sudo nano /etc/cloudflared/config.yml
```

```yaml
tunnel: <TUNNEL_ID>
credentials-file: /home/<your-user>/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: web.yourdomain.com
    service: http://localhost:3000        # adjust port to your web service
  - hostname: ssh.yourdomain.com
    service: ssh://localhost:22
  - service: http_status:404             # catch-all, must be last
```

Replace `<TUNNEL_ID>`, `<your-user>`, and hostnames accordingly.

---

## 5. Create DNS Records

Point your subdomains to the tunnel. Run once per hostname:

```bash
cloudflared tunnel route dns <tunnel-name> web.yourdomain.com
cloudflared tunnel route dns <tunnel-name> ssh.yourdomain.com
```

This creates `CNAME` records in Cloudflare DNS automatically.

---

## 6. Configure SSH Access in Cloudflare Zero Trust

SSH tunneling requires an application entry in Zero Trust:

1. Go to [Cloudflare Zero Trust](https://one.dash.cloudflare.com) → **Access → Applications**
2. Click **Add an application → Self-hosted**
3. Set:
   - **Application name**: anything descriptive
   - **Application domain**: `ssh.yourdomain.com`
   - **Application type**: SSH
4. Configure an access policy (e.g. allow your email address)
5. Save

Without this step, SSH connections through the tunnel will be rejected.

---

## 7. Run as a System Service

Install and enable the tunnel as a systemd service:

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

Check status:

```bash
sudo systemctl status cloudflared
```

Useful commands:

```bash
sudo systemctl restart cloudflared
sudo journalctl -u cloudflared -f    # live logs
```

---

## 8. Client Setup (Your Local Machine)

### Install cloudflared locally

**macOS:**
```bash
brew install cloudflared
```

**Linux:**
```bash
# Same apt install steps as server (see Step 1)
```

**Windows:** Download from [https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/)

### Configure SSH client

Add to `~/.ssh/config` on your local machine:

```
Host myserver
    HostName ssh.yourdomain.com
    User <your-server-username>
    ProxyCommand /usr/local/bin/cloudflared access ssh --hostname %h
```

> **Note:** Use the full path to `cloudflared` (find it with `which cloudflared`). On Apple Silicon Macs it's typically `/opt/homebrew/bin/cloudflared`. Using a bare `cloudflared` command can fail because `ssh` spawns the ProxyCommand in a minimal environment without your full `$PATH`.

Connect with:

```bash
ssh myserver
```

---

## Troubleshooting

**ProxyCommand not picked up:**
```bash
# Verify config is applied
ssh -G ssh.yourdomain.com | grep proxycommand

# Check file permissions (must not be group/world writable)
chmod 600 ~/.ssh/config
```

**`cloudflared: command not found` during SSH:**
Use the absolute path in `ProxyCommand` (see client setup above).

**Tunnel not routing SSH:**
Confirm your `config.yml` has a `ssh://localhost:22` ingress rule and restart the service:
```bash
sudo systemctl restart cloudflared
```

**Test tunnel connectivity directly:**
```bash
cloudflared access ssh --hostname ssh.yourdomain.com
# Should print the server's SSH banner, e.g.: SSH-2.0-OpenSSH_9.6p1
```

---

## Security Notes

- No ports need to be open on the server firewall — the tunnel is outbound-only
- Access policies in Cloudflare Zero Trust control who can reach each service
- SSH traffic is end-to-end encrypted; Cloudflare sees only authenticated tunnel metadata
- Regularly rotate your tunnel credentials if the server is compromised:
  ```bash
  cloudflared tunnel delete <tunnel-name>
  # Then recreate from Step 3
  ```
