# Ubuntu Server User-Level Proxy Setup Guide

> 🌐 Language / 语言选择：[中文](./README.md) | **[English](./README_EN.md)**

This guide is for Ubuntu servers accessed via SSH (including self-hosted machines). It sets up a proxy **only for your user account**, without affecting other users on the same server.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Option 1: Clash Format Subscription](#option-1-clash-format-subscription)
  - [1.1 Download mihomo](#11-download-mihomo)
  - [1.2 Write Configuration File](#12-write-configuration-file)
  - [1.3 Configure systemd User Service](#13-configure-systemd-user-service)
  - [1.4 Set Environment Variables](#14-set-environment-variables)
  - [1.5 Verify Proxy Works](#15-verify-proxy-works)
- [Option 2: V2Ray Format Subscription (requires subconverter)](#option-2-v2ray-format-subscription-requires-subconverter)
  - [2.1 Download mihomo and subconverter](#21-download-mihomo-and-subconverter)
  - [2.2 URL-Encode the Subscription Link](#22-url-encode-the-subscription-link)
  - [2.3 Write Configuration File](#23-write-configuration-file)
  - [2.4 Configure systemd User Services](#24-configure-systemd-user-services)
  - [2.5 Set Environment Variables](#25-set-environment-variables)
  - [2.6 Manually Trigger Subscription Update](#26-manually-trigger-subscription-update)
  - [2.7 Verify Proxy Works](#27-verify-proxy-works)
- [Management Commands](#management-commands)
- [Important Notes](#important-notes)
- [Troubleshooting](#troubleshooting)

---

## Overview

| | Option 1 (Clash Subscription) | Option 2 (V2Ray Subscription) |
|--|-------------------------------|-------------------------------|
| Subscription Format | Clash / mihomo YAML | V2Ray base64 / URI |
| Extra Tools Required | ❌ mihomo only | ✅ mihomo + subconverter |
| Auto Node Updates | ✅ Yes | ✅ Yes (via real-time subconverter conversion) |
| Complexity | Simple | Slightly more complex |

**How to identify your subscription format:**

```bash
curl -L "your-subscription-url" | head -5
```

| Output | Format | Use Option |
|--------|--------|------------|
| Starts with `proxies:` | Clash/mihomo YAML | Option 1 |
| Garbled text (base64) | V2Ray | Option 2 |
| Starts with `ss://` or `vmess://` | URI format | Option 2 |

---

## Prerequisites

### Check Server Architecture

```bash
uname -m
```

| Output | Architecture | Download filename contains |
|--------|-------------|--------------------------|
| `x86_64` | amd64 | `amd64` |
| `aarch64` | arm64 | `arm64` |
| `armv7l` | armv7 | `armv7` |

### Create Working Directory

```bash
mkdir -p ~/proxy/config/providers
```

> **Note**: All proxy files live under your home directory — no root required, no impact on other users.

---

## Option 1: Clash Format Subscription

### 1.1 Download mihomo

> **mihomo** is the Clash.Meta core — the same engine used by Clash for Windows under the hood.

```bash
cd ~/proxy

# Method 1: Use a GitHub mirror (recommended if GitHub is blocked)
wget "https://gh-proxy.com/https://github.com/MetaCubeX/mihomo/releases/download/v1.19.10/mihomo-linux-amd64-v1.19.10.gz"

# Method 2: SSH dynamic port forwarding (if you have a local proxy)
# Run locally: ssh -D 1080 -N user@your-server
# Then on server:
# export https_proxy="socks5://127.0.0.1:1080"
# wget "https://github.com/MetaCubeX/mihomo/releases/download/v1.19.10/mihomo-linux-amd64-v1.19.10.gz"

# Extract and rename
gunzip mihomo-linux-amd64-*.gz
mv mihomo-linux-amd64-* mihomo
chmod +x mihomo
```

**✅ Verify download succeeded:**

```bash
~/proxy/mihomo -v
```

**Expected output:**
```
Mihomo Meta v1.19.10 linux amd64
```

> Any version string output means the download and extraction were successful.

---

### 1.2 Write Configuration File

```bash
cat > ~/proxy/config/config.yaml << 'EOF'
mixed-port: 7890
bind-address: 127.0.0.1
allow-lan: false
mode: rule
log-level: warning
external-controller: 127.0.0.1:9090

proxy-providers:
  my-sub:
    type: http
    url: "paste-your-clash-subscription-url-here"
    interval: 86400
    path: ./providers/sub.yaml
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300

proxy-groups:
  - name: "PROXY"
    type: url-test
    use:
      - my-sub
    url: https://www.gstatic.com/generate_204
    interval: 300

rules:
  - GEOIP,CN,DIRECT
  - MATCH,PROXY
EOF
```

**Fill in your subscription URL:**

```bash
nano ~/proxy/config/config.yaml
# Find: url: "paste-your-clash-subscription-url-here"
# Replace with your actual Clash subscription URL
# Ctrl+O to save, Enter to confirm, Ctrl+X to exit
```

**Key configuration explained:**

| Field | Value | Purpose |
|-------|-------|---------|
| `mixed-port: 7890` | 7890 | Proxy listen port, handles both HTTP and SOCKS5 |
| `bind-address: 127.0.0.1` | 127.0.0.1 | Only binds to loopback — other users cannot access it |
| `allow-lan: false` | false | Disables LAN access, keeping the proxy to current user only |
| `interval: 86400` | 86400s | Pulls fresh node list every 24 hours automatically |
| `url: https://www.gstatic.com/generate_204` | Google 204 | Health-check URL — blocked in China, so reachability proves the node works |
| `type: url-test` | url-test | Automatically selects the lowest-latency node |
| `GEOIP,CN,DIRECT` | — | Chinese IPs connect directly without going through the proxy |

**Protect config file (contains sensitive subscription URL):**

```bash
chmod 600 ~/proxy/config/config.yaml
```

**Test run in foreground (verify config is valid):**

```bash
~/proxy/mihomo -d ~/proxy/config/
```

**✅ Expected output:**
```
INFO Start initial compatible provider my-sub
INFO Proxy 7890 activated
```

> If you see `Proxy 7890 activated` with no `ERROR` lines, the config is correct. Press `Ctrl+C` to exit and proceed.

---

### 1.3 Configure systemd User Service

> **Why systemd?** Without it, mihomo stops when you close the SSH session. A systemd user service keeps it running in the background, survives logouts, auto-starts on reboot, and restarts automatically if it crashes.

```bash
mkdir -p ~/.config/systemd/user/

cat > ~/.config/systemd/user/mihomo.service << 'EOF'
[Unit]
Description=Mihomo Proxy
After=network.target

[Service]
Type=simple
ExecStart=%h/proxy/mihomo -d %h/proxy/config
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF
```

**Enable and start the service:**

```bash
systemctl --user daemon-reload
systemctl --user enable mihomo   # auto-start on boot
systemctl --user start mihomo    # start now
```

**Allow service to run after logout (requires sudo, one-time setup):**

```bash
# $USER is automatically substituted with your username — no need to edit it
sudo loginctl enable-linger $USER
```

> **Why this matters**: By default, user services stop when you log out. `enable-linger` decouples the service from your login session so it keeps running even with no active SSH connections.

**✅ Verify service is running:**

```bash
systemctl --user status mihomo
```

**Expected output:**
```
● mihomo.service - Mihomo Proxy
     Loaded: loaded (...; enabled; ...)
     Active: active (running) since ...
   Main PID: 12345 (mihomo)
```

> Key fields: `Active: active (running)` = service is healthy; `enabled` = will auto-start on reboot.

---

### 1.4 Set Environment Variables

> These variables only apply to your shell sessions — completely invisible to other users.

```bash
cat >> ~/.bashrc << 'EOF'

# Proxy settings
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export ALL_PROXY="socks5://127.0.0.1:7890"
export no_proxy="localhost,127.0.0.1"
EOF

source ~/.bashrc
```

**✅ Verify environment variables are set:**

```bash
echo $http_proxy
```

**Expected output:**
```
http://127.0.0.1:7890
```

---

### 1.5 Verify Proxy Works

**Step 1: Trigger subscription update**

```bash
curl -X PUT http://127.0.0.1:9090/providers/proxies/my-sub
```

**Step 2: Compare real IP vs proxied IP**

```bash
# Server's real IP (force IPv4 to avoid IPv6 confusion)
curl -4 ifconfig.me

# IP as seen through the proxy
curl -x socks5://127.0.0.1:7890 ifconfig.me
```

**✅ Success looks like:**
```
# Real IP:
111.111.111.111

# Proxied IP:
222.222.222.222   ← Different from real IP — this is the proxy node's IP
```

**Step 3: Test access to a blocked site**

```bash
curl -x socks5://127.0.0.1:7890 https://www.google.com -I
```

**✅ Success looks like:**
```
HTTP/2 200
```

> Two different IPs + Google returning `200` = proxy is fully working.

---

## Option 2: V2Ray Format Subscription (requires subconverter)

> **How it works:**
> ```
> mihomo (every 24h) → requests local subconverter → subconverter fetches latest V2Ray subscription
>   → converts to Clash format in real-time → returned to mihomo
> ```

### 2.1 Download mihomo and subconverter

```bash
mkdir -p ~/proxy/config/providers
mkdir -p ~/subconverter

# Download mihomo
cd ~/proxy
wget "https://gh-proxy.com/https://github.com/MetaCubeX/mihomo/releases/download/v1.19.10/mihomo-linux-amd64-v1.19.10.gz"
gunzip mihomo-linux-amd64-*.gz
mv mihomo-linux-amd64-* mihomo
chmod +x mihomo

# Download subconverter
cd ~/subconverter
wget "https://gh-proxy.com/https://github.com/tindy2013/subconverter/releases/latest/download/subconverter_linux64.tar.gz"
tar xzf subconverter_linux64.tar.gz

# Fix directory structure: tar extraction creates an extra nested directory
# Before fix: ~/subconverter/subconverter/ (directory)
# After fix:  ~/subconverter/subconverter  (executable)
mv ~/subconverter/subconverter/* ~/subconverter/
rmdir ~/subconverter/subconverter
chmod +x ~/subconverter/subconverter
```

**✅ Verify mihomo:**

```bash
~/proxy/mihomo -v
```

**Expected:**
```
Mihomo Meta v1.19.10 linux amd64
```

**✅ Verify subconverter is an executable (not a directory):**

```bash
file ~/subconverter/subconverter
```

**Expected:**
```
/home/yourname/subconverter/subconverter: ELF 64-bit LSB executable ...
```

> Must show `ELF 64-bit LSB executable`. If it shows `directory`, the directory structure is wrong — redo the `mv` step above.

---

### 2.2 URL-Encode the Subscription Link

> **Why encode?** Your subscription URL is passed as a query parameter inside the subconverter URL. Characters like `/`, `:`, and `?` in your subscription URL would be misinterpreted as URL structure if not encoded.

```bash
python3 -c "import urllib.parse; print(urllib.parse.quote('your-v2ray-subscription-url', safe=''))"
```

> **Important**: The `safe=''` argument is required. Without it, `/` won't be encoded, causing subconverter to misparse the URL.

**Example:**
```bash
# Input
python3 -c "import urllib.parse; print(urllib.parse.quote('https://airport.com/subscribe?token=abc123', safe=''))"

# Output (encoded URL)
https%3A%2F%2Fairport.com%2Fsubscribe%3Ftoken%3Dabc123
```

**Save the encoded URL — you'll need it in the next step.**

---

### 2.3 Write Configuration File

```bash
cat > ~/proxy/config/config.yaml << 'EOF'
mixed-port: 7890
bind-address: 127.0.0.1
allow-lan: false
mode: rule
log-level: warning
external-controller: 127.0.0.1:9090

proxy-providers:
  my-sub:
    type: http
    url: "http://127.0.0.1:25500/sub?target=clash&url=paste-url-encoded-subscription-here"
    interval: 86400
    path: ./providers/sub.yaml
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300

proxy-groups:
  - name: "PROXY"
    type: url-test
    use:
      - my-sub
    url: https://www.gstatic.com/generate_204
    interval: 300

rules:
  - GEOIP,CN,DIRECT
  - MATCH,PROXY
EOF
```

**Fill in the encoded URL:**

```bash
nano ~/proxy/config/config.yaml
# Replace the placeholder with your encoded URL
# Full example:
# url: "http://127.0.0.1:25500/sub?target=clash&url=https%3A%2F%2Fairport.com%2Fsubscribe%3Ftoken%3Dabc123"
```

**Protect the config file:**

```bash
chmod 600 ~/proxy/config/config.yaml
```

---

### 2.4 Configure systemd User Services

> Two services are needed: **subconverter** (subscription converter) and **mihomo** (proxy core). subconverter must start first since mihomo depends on it.

**subconverter service:**

```bash
mkdir -p ~/.config/systemd/user/

cat > ~/.config/systemd/user/subconverter.service << 'EOF'
[Unit]
Description=Subconverter
After=network.target

[Service]
Type=simple
WorkingDirectory=%h/subconverter
ExecStart=%h/subconverter/subconverter
Restart=on-failure
RestartSec=3

[Install]
WantedBy=default.target
EOF
```

**mihomo service:**

```bash
cat > ~/.config/systemd/user/mihomo.service << 'EOF'
[Unit]
Description=Mihomo Proxy
After=network.target subconverter.service
Requires=subconverter.service

[Service]
Type=simple
ExecStart=%h/proxy/mihomo -d %h/proxy/config
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF
```

**Enable and start both services:**

```bash
systemctl --user daemon-reload

# Start subconverter first
systemctl --user enable subconverter
systemctl --user start subconverter

# Wait for subconverter to be ready, then start mihomo
sleep 2
systemctl --user enable mihomo
systemctl --user start mihomo

# Allow services to persist after logout
sudo loginctl enable-linger $USER
```

**✅ Verify subconverter is running:**

```bash
systemctl --user status subconverter
```

**Expected:**
```
● subconverter.service - Subconverter
     Active: active (running) since ...
   Main PID: 12345 (subconverter)
```

**✅ Verify mihomo is running:**

```bash
systemctl --user status mihomo
```

**Expected:**
```
● mihomo.service - Mihomo Proxy
     Active: active (running) since ...
   Main PID: 12346 (mihomo)
```

**✅ Verify subconverter can convert your subscription:**

```bash
curl "http://127.0.0.1:25500/sub?target=clash&url=your-url-encoded-subscription" | head -10
```

**Expected:**
```yaml
proxies:
  - name: "HK-01"
    type: vmess
    server: ...
```

> Seeing a `proxies:` YAML list means the conversion is working correctly.

---

### 2.5 Set Environment Variables

```bash
cat >> ~/.bashrc << 'EOF'

# Proxy settings
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export ALL_PROXY="socks5://127.0.0.1:7890"
export no_proxy="localhost,127.0.0.1"
EOF

source ~/.bashrc
```

---

### 2.6 Manually Trigger Subscription Update

> mihomo does not pull proxy-providers subscriptions immediately on startup. Trigger it manually once — subsequent updates happen automatically per `interval`.

```bash
curl -X PUT http://127.0.0.1:9090/providers/proxies/my-sub
```

---

### 2.7 Verify Proxy Works

**Compare real IP vs proxied IP:**

```bash
# Server's real IP (force IPv4 to avoid IPv6 confusion)
curl -4 ifconfig.me

# IP through the proxy
curl -x socks5://127.0.0.1:7890 ifconfig.me
```

**✅ Success:**
```
# Real IP:
111.111.111.111

# Proxied IP:
222.222.222.222   ← Different IP = proxy node's IP
```

**Test access to a blocked site:**

```bash
curl -x socks5://127.0.0.1:7890 https://www.google.com -I
```

**✅ Success:**
```
HTTP/2 200
```

---

## Management Commands

```bash
# Check service status
systemctl --user status mihomo
systemctl --user status subconverter   # Option 2 only

# Start / Stop / Restart
systemctl --user start mihomo
systemctl --user stop mihomo
systemctl --user restart mihomo

# View live logs
journalctl --user -u mihomo -f
journalctl --user -u subconverter -f   # Option 2 only

# Manually trigger node update (no restart needed)
curl -X PUT http://127.0.0.1:9090/providers/proxies/my-sub

# Temporarily disable proxy (current session only)
unset http_proxy https_proxy ALL_PROXY

# Re-enable proxy (current session only)
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export ALL_PROXY="socks5://127.0.0.1:7890"

# SSH to another server bypassing the proxy
ssh -o ProxyCommand=none user@other-server
```

---

## Important Notes

### Security

- `bind-address` **must** be `127.0.0.1` — never use `0.0.0.0`, which would expose your proxy to other users or the public internet
- Keep `allow-lan: false`
- Config file contains your subscription URL — protect it: `chmod 600 ~/proxy/config/config.yaml`

### SSH Connections

- Inbound SSH connections to your server are completely unaffected (the proxy only affects outbound traffic from the server)
- If you SSH from the server to another machine, that connection will go through the proxy. To bypass: `ssh -o ProxyCommand=none user@other-server`

### Dynamic IP (self-hosted machines)

Home ISP IPs change on router restart or reconnection. Set up DDNS to connect by hostname instead:

```bash
# Free services: Duck DNS, Cloudflare DDNS
# After setup, connect using your domain:
ssh user@your-ddns-domain.duckdns.org
```

### Port Conflicts

```bash
# Check if port 7890 is already in use
ss -tlnp | grep 7890
# If occupied, change mixed-port in config.yaml to another port
```

---

## Troubleshooting

### subconverter fails with "Is a directory"

**Error:**
```
Failed to locate executable: Is a directory
```

**Cause**: The tar archive extracted into a nested directory with the same name as the executable.

**Fix:**
```bash
systemctl --user stop subconverter
mv ~/subconverter/subconverter/* ~/subconverter/
rmdir ~/subconverter/subconverter
systemctl --user start subconverter
```

### Proxied IP is the same as real IP

**Possible causes:**

1. Subscription not yet loaded → run `curl -X PUT http://127.0.0.1:9090/providers/proxies/my-sub`
2. `curl ifconfig.me` returned an IPv6 address → use `curl -4 ifconfig.me` to force IPv4 comparison
3. All nodes are down → check node health: `curl http://127.0.0.1:9090/providers/proxies/my-sub`

### SSL error on HTTPS requests through proxy

**Error:**
```
curl: (35) error:0A000126:SSL routines::unexpected eof while reading
```

**Diagnose:**
```bash
# 1. Confirm nodes are loaded
curl http://127.0.0.1:9090/providers/proxies/my-sub | head -50

# 2. Test raw connectivity to a node
nc -zv <node-server-ip> <node-port>

# 3. Watch live mihomo logs while testing
journalctl --user -u mihomo -f
# Run curl test in another window simultaneously
```

### View detailed error logs

```bash
journalctl --user -u mihomo -n 50
journalctl --user -u subconverter -n 50
```

---

## Directory Structure Reference

**Option 1:**
```
~
└── proxy/
    ├── mihomo                  # main binary
    └── config/
        ├── config.yaml         # configuration file
        └── providers/
            └── sub.yaml        # cached subscription
```

**Option 2:**
```
~
├── proxy/
│   ├── mihomo                  # main binary
│   └── config/
│       ├── config.yaml         # configuration file
│       └── providers/
│           └── sub.yaml        # converted node cache
└── subconverter/
    ├── subconverter            # main binary
    └── pref.toml               # subconverter config
```
