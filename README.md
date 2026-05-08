# Ubuntu 服务器用户级代理配置指南

> 🌐 Language / 语言选择：**[中文](./README.md)** | [English](./README_EN.md)

本指南适用于通过 SSH 连接的 Ubuntu 服务器（包括自建主机），实现**仅当前用户账号走代理，不影响其他用户**的翻墙方案。

---

## 目录

- [方案说明](#方案说明)
- [前置准备](#前置准备)
- [方案一：Clash 格式订阅](#方案一clash-格式订阅)
  - [1.1 下载 mihomo](#11-下载-mihomo)
  - [1.2 编写配置文件](#12-编写配置文件)
  - [1.3 配置 systemd 用户服务](#13-配置-systemd-用户服务)
  - [1.4 配置环境变量](#14-配置环境变量)
  - [1.5 验证代理是否成功](#15-验证代理是否成功)
- [方案二：V2Ray 格式订阅（需 subconverter 转换）](#方案二v2ray-格式订阅需-subconverter-转换)
  - [2.1 下载 mihomo 和 subconverter](#21-下载-mihomo-和-subconverter)
  - [2.2 对订阅链接进行 URL 编码](#22-对订阅链接进行-url-编码)
  - [2.3 编写配置文件](#23-编写配置文件)
  - [2.4 配置 systemd 用户服务](#24-配置-systemd-用户服务)
  - [2.5 配置环境变量](#25-配置环境变量)
  - [2.6 手动触发订阅更新](#26-手动触发订阅更新)
  - [2.7 验证代理是否成功](#27-验证代理是否成功)
- [常用管理命令](#常用管理命令)
- [注意事项](#注意事项)
- [常见问题排查](#常见问题排查)
- [Web可视化界面](#Web可视化界面)
- [目录结构参考](#目录结构参考)

---

## 方案说明

| 对比项 | 方案一（Clash 订阅） | 方案二（V2Ray 订阅） |
|--------|-----------------|-----------------|
| 适用订阅格式 | Clash / mihomo YAML 格式 | V2Ray base64 / URI 格式 |
| 是否需要额外工具 | ❌ 只需 mihomo | ✅ 需要 mihomo + subconverter |
| 节点自动更新 | ✅ 支持 | ✅ 支持（经 subconverter 实时转换） |
| 复杂程度 | 简单 | 稍复杂 |

**如何判断自己的订阅格式：**

```bash
curl -L "你的订阅链接" | head -5
```

| 输出内容 | 格式 | 选用方案 |
|---------|------|---------|
| 开头是 `proxies:` | Clash/mihomo YAML | 方案一 |
| 开头是乱码（base64） | V2Ray | 方案二 |
| 开头是 `ss://` `vmess://` | URI 格式 | 方案二 |

---

## 前置准备

### 确认服务器架构

```bash
uname -m
```

| 输出 | 对应架构 | 下载文件名包含 |
|------|---------|-------------|
| `x86_64` | amd64 | `amd64` |
| `aarch64` | arm64 | `arm64` |
| `armv7l` | armv7 | `armv7` |

### 创建工作目录

```bash
mkdir -p ~/proxy/config/providers
```

> **说明**：所有代理相关文件存放在家目录下，不需要 root 权限，不影响其他用户。

---

## 方案一：Clash 格式订阅

### 1.1 下载 mihomo

> **mihomo** 是 Clash.Meta 内核，Clash for Windows 底层使用的同款内核，支持多种代理协议。

```bash
cd ~/proxy

# 方法一：使用 GitHub 镜像（国内服务器推荐）
wget "https://gh-proxy.com/https://github.com/MetaCubeX/mihomo/releases/download/v1.19.10/mihomo-linux-amd64-v1.19.10.gz"

# 方法二：如果本地已有代理，SSH 动态转发后直接下载
# 本地执行：ssh -D 1080 -N user@your-server
# 服务器执行：
# export https_proxy="socks5://127.0.0.1:1080"
# wget "https://github.com/MetaCubeX/mihomo/releases/download/v1.19.10/mihomo-linux-amd64-v1.19.10.gz"

# 解压并重命名
gunzip mihomo-linux-amd64-*.gz
mv mihomo-linux-amd64-* mihomo
chmod +x mihomo
```

**✅ 验证下载成功：**

```bash
~/proxy/mihomo -v
```

**成功结果示例：**
```
Mihomo Meta v1.19.10 linux amd64
```

> 看到版本号输出即表示下载和解压成功。

---

### 1.2 编写配置文件

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
    url: "这里填写你的 Clash 格式订阅链接"
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

**填入订阅链接：**

```bash
nano ~/proxy/config/config.yaml
# 找到 url: "这里填写你的 Clash 格式订阅链接"
# 替换为你的实际订阅链接
# Ctrl+O 保存，Enter 确认，Ctrl+X 退出
```

**关键配置说明：**

| 配置项 | 值 | 作用 |
|-------|-----|------|
| `mixed-port: 7890` | 7890 | 代理监听端口，HTTP 和 SOCKS5 二合一 |
| `bind-address: 127.0.0.1` | 127.0.0.1 | 只监听本地回环，其他用户无法访问 |
| `allow-lan: false` | false | 禁止局域网访问，保证只有当前用户可用 |
| `interval: 86400` | 86400秒 | 每 24 小时自动拉取最新节点 |
| `url: https://www.gstatic.com/generate_204` | Google 204 地址 | 用于测速的地址，在国内被墙，能访问说明节点有效 |
| `type: url-test` | url-test | 自动选择延迟最低的节点 |
| `GEOIP,CN,DIRECT` | — | 国内 IP 直连，不走代理 |

**保护配置文件权限（含有订阅链接等敏感信息）：**

```bash
chmod 600 ~/proxy/config/config.yaml
```

**✅ 验证配置文件正确：**

```bash
head -5 ~/proxy/config/config.yaml
```

**成功结果：**
```
mixed-port: 7890
bind-address: 127.0.0.1
allow-lan: false
mode: rule
log-level: warning
```

> 看到正常的 YAML 结构，且 `bind-address` 为 `127.0.0.1`、`allow-lan` 为 `false` 即为正确。

**前台测试运行（验证配置无误）：**

```bash
~/proxy/mihomo -d ~/proxy/config/
```

**成功结果：**
```
INFO Start initial compatible provider my-sub
INFO Proxy 7890 activated
```

> 看到 `Proxy 7890 activated` 且没有 `ERROR` 字样即表示配置正确，按 `Ctrl+C` 退出后进行下一步。

---

### 1.3 配置 systemd 用户服务

> **systemd 用户服务**的作用：让 mihomo 在后台持续运行，SSH 断开后不停止，服务器重启后自动启动，进程崩溃后自动重启。

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

**启动服务：**

```bash
systemctl --user daemon-reload
systemctl --user enable mihomo   # 设置开机自启
systemctl --user start mihomo    # 立即启动
```

**设置登出后继续运行（需要管理员执行一次）：**

```bash
# $USER 会自动替换为当前用户名，无需手动修改
sudo loginctl enable-linger $USER
```

> **说明**：默认情况下用户服务在登出后停止。`enable-linger` 让服务脱离登录会话，即使没有 SSH 连接也能持续运行。

**✅ 验证服务状态：**

```bash
systemctl --user status mihomo
```

**成功结果：**
```
● mihomo.service - Mihomo Proxy
     Loaded: loaded (...; enabled; ...)
     Active: active (running) since ...
   Main PID: 12345 (mihomo)
```

> 关键字段：`Active: active (running)` 表示服务正常运行，`enabled` 表示已设置开机自启。

---

### 1.4 配置环境变量

> 环境变量只在你的 Shell 会话中生效，对其他用户完全无影响。

```bash
cat >> ~/.bashrc << 'EOF'

# 代理设置
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export ALL_PROXY="socks5://127.0.0.1:7890"
export no_proxy="localhost,127.0.0.1"
EOF

source ~/.bashrc
```

**✅ 验证环境变量生效：**

```bash
echo $http_proxy
```

**成功结果：**
```
http://127.0.0.1:7890
```

---

### 1.5 验证代理是否成功

**第一步：手动触发订阅更新**

```bash
curl -X PUT http://127.0.0.1:9090/providers/proxies/my-sub
```

**第二步：对比真实 IP 和代理 IP**

```bash
# 查看服务器真实 IP（强制 IPv4）
curl -4 ifconfig.me

# 查看通过代理后的出口 IP
curl -x socks5://127.0.0.1:7890 ifconfig.me
```

**✅ 成功结果：**
```
# 命令1（真实 IP）：
111.111.111.111

# 命令2（代理 IP）：
222.222.222.222   ← 与真实 IP 不同，为代理节点的 IP
```

**第三步：测试访问被墙网站**

```bash
curl -x socks5://127.0.0.1:7890 https://www.google.com -I
```

**✅ 成功结果：**
```
HTTP/2 200
```

> 两个 IP 不同，且能访问 Google 返回 `200`，代理配置完全成功。

---

## 方案二：V2Ray 格式订阅（需 subconverter 转换）

> **整体流程**：
> ```
> mihomo（每24小时）→ 请求本地 subconverter → subconverter 拉取最新 V2Ray 订阅 → 实时转换为 Clash 格式 → 返回给 mihomo
> ```

### 2.1 下载 mihomo 和 subconverter

```bash
mkdir -p ~/proxy/config/providers
mkdir -p ~/subconverter

# 下载 mihomo
cd ~/proxy
wget "https://gh-proxy.com/https://github.com/MetaCubeX/mihomo/releases/download/v1.19.10/mihomo-linux-amd64-v1.19.10.gz"
gunzip mihomo-linux-amd64-*.gz
mv mihomo-linux-amd64-* mihomo
chmod +x mihomo

# 下载 subconverter
cd ~/subconverter
wget "https://gh-proxy.com/https://github.com/tindy2013/subconverter/releases/latest/download/subconverter_linux64.tar.gz"
tar xzf subconverter_linux64.tar.gz

# 修复目录结构：tar 解压后会多一层目录，需要将文件移到正确位置
# 解压后结构：~/subconverter/subconverter/（目录）
# 需要的结构：~/subconverter/subconverter（可执行文件）
mv ~/subconverter/subconverter/* ~/subconverter/
rmdir ~/subconverter/subconverter
chmod +x ~/subconverter/subconverter
```

**✅ 验证 mihomo 下载成功：**

```bash
~/proxy/mihomo -v
```

**成功结果：**
```
Mihomo Meta v1.19.10 linux amd64
```

**✅ 验证 subconverter 文件正确（是可执行文件而非目录）：**

```bash
file ~/subconverter/subconverter
```

**成功结果：**
```
/home/yourname/subconverter/subconverter: ELF 64-bit LSB executable ...
```

> 必须显示 `ELF 64-bit LSB executable`，如果显示 `directory` 说明目录结构有问题，需重新执行上面的 `mv` 步骤。

---

### 2.2 对订阅链接进行 URL 编码

> **为什么需要 URL 编码**：订阅链接作为参数拼接在 subconverter URL 中，链接中的 `/`、`:`、`?` 等字符会被误解析为 URL 的路径或参数分隔符，必须编码后才能正确传递。

```bash
python3 -c "import urllib.parse; print(urllib.parse.quote('你的V2Ray订阅链接', safe=''))"
```

> **注意**：必须加 `safe=''`，否则 `/` 不会被编码，导致 subconverter 解析出错。

**示例：**
```bash
# 输入
python3 -c "import urllib.parse; print(urllib.parse.quote('https://airport.com/subscribe?token=abc123', safe=''))"

# 输出（编码后的链接）
https%3A%2F%2Fairport.com%2Fsubscribe%3Ftoken%3Dabc123
```

**记录编码后的链接，下一步使用。**

---

### 2.3 编写配置文件

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
    url: "http://127.0.0.1:25500/sub?target=clash&url=这里填写URL编码后的订阅链接"
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

**填入编码后的订阅链接：**

```bash
nano ~/proxy/config/config.yaml
# 将 url 字段中的占位文字替换为上一步编码后的链接
# 完整 url 示例：
# url: "http://127.0.0.1:25500/sub?target=clash&url=https%3A%2F%2Fairport.com%2Fsubscribe%3Ftoken%3Dabc123"
```

**保护配置文件：**

```bash
chmod 600 ~/proxy/config/config.yaml
```

---

### 2.4 配置 systemd 用户服务

> 需要配置两个服务：**subconverter**（订阅转换）和 **mihomo**（代理核心）。mihomo 依赖 subconverter，所以 subconverter 必须先启动。

**subconverter 服务：**

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

**mihomo 服务：**

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

**启动服务：**

```bash
systemctl --user daemon-reload

# 先启动 subconverter
systemctl --user enable subconverter
systemctl --user start subconverter

# 等待 subconverter 就绪后再启动 mihomo
sleep 2
systemctl --user enable mihomo
systemctl --user start mihomo

# 设置登出后继续运行
sudo loginctl enable-linger $USER
```

**✅ 验证 subconverter 服务状态：**

```bash
systemctl --user status subconverter
```

**成功结果：**
```
● subconverter.service - Subconverter
     Active: active (running) since ...
   Main PID: 12345 (subconverter)
```

> `Active: active (running)` 为正常，若显示 `failed` 查看 [常见问题排查](#常见问题排查)。

**✅ 验证 mihomo 服务状态：**

```bash
systemctl --user status mihomo
```

**成功结果：**
```
● mihomo.service - Mihomo Proxy
     Active: active (running) since ...
   Main PID: 12346 (mihomo)
```

**✅ 验证 subconverter 能成功转换订阅：**

```bash
curl "http://127.0.0.1:25500/sub?target=clash&url=你的URL编码订阅链接" | head -10
```

**成功结果：**
```yaml
proxies:
  - name: "香港-01"
    type: vmess
    server: ...
```

> 看到 `proxies:` 开头的 YAML 格式节点列表即为成功。

---

### 2.5 配置环境变量

```bash
cat >> ~/.bashrc << 'EOF'

# 代理设置
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export ALL_PROXY="socks5://127.0.0.1:7890"
export no_proxy="localhost,127.0.0.1"
EOF

source ~/.bashrc
```

---

### 2.6 手动触发订阅更新

> mihomo 启动时不会立即拉取 proxy-providers 订阅，需要手动触发一次，后续按 `interval` 自动更新。

```bash
curl -X PUT http://127.0.0.1:9090/providers/proxies/my-sub
```

---

### 2.7 验证代理是否成功

**对比真实 IP 和代理 IP：**

```bash
# 服务器真实 IP（强制 IPv4，避免 IPv6 干扰）
curl -4 ifconfig.me

# 通过代理后的出口 IP
curl -x socks5://127.0.0.1:7890 ifconfig.me
```

**✅ 成功结果：**
```
# 真实 IP：
111.111.111.111

# 代理 IP：
222.222.222.222   ← 与真实 IP 不同
```

**测试访问被墙网站：**

```bash
curl -x socks5://127.0.0.1:7890 https://www.google.com -I
```

**✅ 成功结果：**
```
HTTP/2 200
```

---

## 常用管理命令

```bash
# 查看服务状态
systemctl --user status mihomo
systemctl --user status subconverter   # 仅方案二

# 启动 / 停止 / 重启
systemctl --user start mihomo
systemctl --user stop mihomo
systemctl --user restart mihomo

# 查看实时日志
journalctl --user -u mihomo -f
journalctl --user -u subconverter -f   # 仅方案二

# 手动触发节点更新（不需要重启服务）
curl -X PUT http://127.0.0.1:9090/providers/proxies/my-sub

# 临时关闭代理（当前会话）
unset http_proxy https_proxy ALL_PROXY

# 临时开启代理（当前会话）
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export ALL_PROXY="socks5://127.0.0.1:7890"

# 在服务器上 SSH 连接其他机器时跳过代理
ssh -o ProxyCommand=none user@other-server
```

---

## 注意事项

### 安全

- `bind-address` 必须设为 `127.0.0.1`，**绝对不能用 `0.0.0.0`**，否则其他用户甚至公网可访问你的代理
- `allow-lan` 保持 `false`
- 配置文件含有订阅链接等敏感信息，权限设为 `600`：`chmod 600 ~/proxy/config/config.yaml`

### SSH 连接

- 从本地连入服务器的 SSH 完全不受代理影响（代理只影响从服务器发出的流量）
- 在服务器上用 `ssh` 连接其他机器时会走代理，如需跳过：`ssh -o ProxyCommand=none user@other-server`

### 动态 IP（自建主机适用）

家庭宽带 IP 会动态变化，建议配置 DDNS：

```bash
# 推荐免费服务：Duck DNS、Cloudflare DDNS
# 配置后用域名连接，不受 IP 变化影响
ssh user@your-ddns-domain.duckdns.org
```

### 端口冲突

```bash
# 检查 7890 端口是否被占用
ss -tlnp | grep 7890
# 如被占用，在 config.yaml 中修改 mixed-port 为其他端口
```

---

## 常见问题排查

### subconverter 启动失败：`Is a directory`

**错误信息：**
```
Failed to locate executable: Is a directory
```

**原因**：解压后目录结构多了一层，`subconverter` 是目录而非可执行文件。

**修复：**
```bash
systemctl --user stop subconverter
mv ~/subconverter/subconverter/* ~/subconverter/
rmdir ~/subconverter/subconverter
systemctl --user start subconverter
```

### 代理 IP 和真实 IP 相同

**可能原因：**
1. 未手动触发订阅更新 → 执行 `curl -X PUT http://127.0.0.1:9090/providers/proxies/my-sub`
2. `curl ifconfig.me` 走了 IPv6 → 使用 `curl -4 ifconfig.me` 强制 IPv4 对比
3. 节点全部失效 → 查看节点延迟 `curl http://127.0.0.1:9090/providers/proxies/my-sub`

### curl 访问 HTTPS 报 SSL 错误

**错误信息：**
```
curl: (35) error:0A000126:SSL routines::unexpected eof while reading
```

**排查步骤：**
```bash
# 1. 确认节点已加载
curl http://127.0.0.1:9090/providers/proxies/my-sub | head -50

# 2. 测试节点连通性
nc -zv 节点服务器IP 节点端口

# 3. 查看 mihomo 实时日志
journalctl --user -u mihomo -f
# 同时在另一窗口执行 curl 测试，观察日志输出
```

### 查看详细错误日志

```bash
journalctl --user -u mihomo -n 50
journalctl --user -u subconverter -n 50
```

---

## Web可视化界面

### 1.本地建立SSH端口转发
```bash
ssh -p 22 -L 9090:127.0.0.1:9090 <你的服务器账号名>@<你的服务器IP>
```

### 2.打开可视化面板
在本地浏览器中打开：https://metacubex.github.io/metacubexd/

---

## 目录结构参考

**方案一：**
```
~
└── proxy/
    ├── mihomo                  # 主程序
    └── config/
        ├── config.yaml         # 配置文件
        └── providers/
            └── sub.yaml        # 订阅缓存
```

**方案二：**
```
~
├── proxy/
│   ├── mihomo                  # 主程序
│   └── config/
│       ├── config.yaml         # 配置文件
│       └── providers/
│           └── sub.yaml        # 转换后的节点缓存
└── subconverter/
    ├── subconverter            # 主程序
    └── pref.toml               # subconverter 配置
```
