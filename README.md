# 基于 Vultr 的 VPS 部署学习笔记

声明：仅用于学习用途，其他与我无关

> 本仓库记录 VPS 初始化、Shadowsocks-libev 部署、Windows 客户端连接、Clash/Mihomo 分享、PAC 模式排错，以及独立朋友节点的共享流量限额实验。  
> **隐私说明：** 本文中的服务器 IP、客户端 IP、密码、用户名和敏感路径已用占位符或示例值代替。不要把真实密钥、完整配置、连接日志或带 IP 的截图提交到仓库。

> 实测跟着gpt来确实可以部署（实在不会把我融了问gpt）

## 目录

- [一、项目概览](#一项目概览)
- [二、创建 Vultr VPS](#二创建-vultr-vps)
- [三、初始化 Ubuntu](#三初始化-ubuntu)
- [四、安装 Shadowsocks-libev](#四安装-shadowsocks-libev)
- [五、专用服务账户与 systemd](#五专用服务账户与-systemd)
- [六、UFW 和云防火墙](#六ufw-和云防火墙)
- [七、Windows 客户端连接与验证](#七windows-客户端连接与验证)
- [八、分享节点给 Clash/Mihomo](#八分享节点给-clashmihomo)
- [九、PAC 模式排错](#九pac-模式排错)
- [十、独立朋友节点与共享 500 GB 限额实验](#十独立朋友节点与共享-500-gb-限额实验)
- [十一、常用排错命令](#十一常用排错命令)
- [十二、已知限制和上线前检查](#十二已知限制和上线前检查)
- [推荐链接、参考资料与致谢](#推荐链接参考资料与致谢)

---

## 一、项目概览

这份笔记合并了两部分实践记录。它们是**不同 VPS 实例上的阶段性实验**，具体系统版本、端口与配置不能不加检查地混用。

### 实例 A：基础 VPS 与单节点部署

| 项目 | 本次记录 |
|---|---|
| 云服务商 | Vultr |
| 实例类型 | Shared CPU |
| 套餐 | `vc2-1c-1gb`（当时选择） |
| CPU / 内存 | 1 vCPU / 1 GB RAM |
| 机房 | ？？？？？？？？？？？ |
| 系统 | Ubuntu 22.04.5 LTS x64 |
| 示例服务端口 | `15336` TCP + UDP |
| 服务单元 | `shadowsocks-libev-server@ss-server.service` |
| 客户端 | Windows Shadowsocks；Clash/Mihomo 配置示例 |
| 费用记录 | 当时控制台预估约 US$6/月（含自动备份选项；价格可能变化，以 Vultr 账单为准） |

Vultr 套餐价格、区域、备份和网络选项可能变化，下单时应重新核对。
加密方式选择aes-256-gcm(shadowsocks使用的加密方式)

### 实例 B：朋友专用节点与共享流量限额

另一次实验中，使用单独的 Shadowsocks 朋友服务端口（示例为 `15337`），并通过 nftables 命名配额尝试限制上传和下载合计的月度网络流量。该实验的系统版本、服务配置和防火墙状态与实例 A 不完全相同。

- 个人节点和朋友节点使用不同端口、密码和服务配置。
- 朋友的多台设备共用同一个命名配额。
- UFW 控制端口访问；nftables 的自定义表处理朋友节点配额。
- systemd timer 按日本时区每月重置额度。
- 规则的超额行为尚未完成端到端压力测试，nftables 表也尚未建立开机自动加载流程，因此暂时不能当作经过完整验证的生产级限流系统。

---

## 二、创建 Vultr VPS

Vultr的购买与配置仅供参考（选择jp的节点 high performence 的vps 效果可能更好一些）

本人的推广链接https://www.vultr.com/?ref=9929461

大家可以支持一下，新用户可以充10刀得310刀

<img width="1906" height="956" alt="屏幕截图 2026-10-18 132800" src="https://github.com/user-attachments/assets/d53fda8f-d51e-4e41-90ed-9249198adc66" />

Vultr的节点选择可以参考pflhyc61大佬的https://github.com/pflhyc61/vultr-server-location-guide

接下来的流程：

准备启动实例

进入 Vultr 控制台，按控制台当前界面选择创建实例（例如 `Products → Compute → Create Instance`）。

当时选择的配置如下：

| 配置项 | 记录值 |
|---|---|
| Instance Type | Shared CPU |
| Plan | `vc2-1c-1gb` |
| CPU | 1 vCPU |
| Memory | 1 GB |
| Location | jp的快一点 |
| Operating System | Ubuntu 22.04 LTS x64 |
| IPv4 | Enabled |
| VPC Network | Off |
| Automatic Backups | On |
| DDoS Protection | Off |
| Limited User Login | Off |
| Cloud-Init | Off |

完成配置后部署实例，等待控制台显示实例可用。套餐名称、价格及选项属于历史记录，实际购买时请以当前控制台内容为准。

### 连接服务器

可以使用 Vultr 的 Web Console：
<img width="613" height="140" alt="image" src="https://github.com/user-attachments/assets/32f034ef-8652-4398-998e-28d416e6560c" />

也可以从本地终端通过 SSH 登录（如使用putty）
---

## 三、初始化 Ubuntu

### 1. 检查系统、网络与防火墙状态

可以使用这个剪贴板
<img width="678" height="292" alt="屏幕截图 2026-10-18 134418" src="https://github.com/user-attachments/assets/cc5dd08b-e6e9-45ed-9d24-5a066f8bcc43" />


```bash
whoami
cat /etc/os-release
ip -4 addr
ip route
ufw status verbose
```

本次实例 A 的记录结果为 root 用户、Ubuntu 22.04.5 LTS、默认路由正常、UFW 已启用，SSH 的 `22/tcp` 已允许。这里的结果只对应当时那台实例，重新部署时应重新检查。

### 2. 更新系统

```bash
apt update
apt upgrade -y
```

更新过程中可能遇到配置文件冲突或服务重启提示。

原实践中，SSH 配置冲突时选择保留当前配置，

PAM 的 `common-*` 覆盖提示选择了 No，并保留默认服务重启选项。

注：这些是当时基于已有配置作出的选择，**不应当作所有机器都通用的固定答案**。在生产环境中，应先理解配置差异，确保 SSH 访问方式不会被意外改动，并保留可回滚途径。



### 3. 检查时间（非必要，可跳过）

```bash
timedatectl status
```

如果月度任务需要按日本时间执行，可以在相应实例上设置：

```bash
timedatectl set-timezone Asia/Tokyo
timedatectl status
```

请确认时区设置符合目标用途；这不是所有 VPS 都必须使用的设置。

---



## 四、安装 Shadowsocks-libev

### 1. 安装与检查程序

Ubuntu/Debian 系可先查询当前软件源，再安装：

```bash
apt install -y shadowsocks-libev openssl
ss-server -h
```

看到 `ss-server` 帮助信息，可以确认程序可执行；这本身并不代表服务已正常启动。

本次实践使用 `shadowsocks-libev`，没有继续沿用旧版 Python 安装方案。软件包与服务模板名称会随发行版及版本有所不同。



### 2. 生成随机密码并存入本机文件

以下代码应在服务器终端连续执行，以确保 shell 变量仍存在：

```bash
SS_PASS="$(openssl rand -hex 24)"
printf '%s\\n' "$SS_PASS" > /root/ss-password.txt
chmod 600 /root/ss-password.txt
```

- `openssl rand -hex 24` 生成随机十六进制密码。
- `chmod 600` 使该文件只有所有者可读写。
- 密码存放于服务器本机；不要把密码文件或其内容提交到 GitHub。

需要在本机查看密码时可执行 `cat /root/ss-password.txt`，但不要把结果发到聊天、Issue 或截图中。



### 3. 创建配置文件

本次实例 A 使用配置路径：

`/etc/shadowsocks-libev/ss-server.json`

创建目录并写入示例配置：

```bash
install -d -m 700 /etc/shadowsocks-libev

cat > /etc/shadowsocks-libev/ss-server.json <<EOF
{
    "server": "0.0.0.0",
    "server_port": 15336,
    "password": "$SS_PASS",
    "timeout": 300,
    "method": "aes-256-gcm",
    "mode": "tcp_and_udp"
}
EOF

chmod 600 /etc/shadowsocks-libev/ss-server.json
python3 -c 'import json; json.load(open("/etc/shadowsocks-libev/ss-server.json")); print("yes")'
```

字段说明：

| 字段 | 值 | 用途 |
|---|---|---|
| `server` | `0.0.0.0` | 监听所有 IPv4 网卡 |
| `server_port` | `15336` | 服务端口示例 |
| `password` | 随机生成 | 客户端认证密码 |
| `timeout` | `300` | 超时参数，单位为秒 |
| `method` | `aes-256-gcm` | 加密方法 |
| `mode` | `tcp_and_udp` | TCP/UDP 转发模式 |

客户端和服务端的地址、端口、密码、加密方法必须匹配。不同版本对配置项的支持可能不同，发生启动错误时请对照本机 `ss-server` 文档和服务日志。

---



## 五、专用服务账户与 systemd

这一部分记录了本次排查服务启动问题时采用的做法，不能假定所有系统都需要相同的覆盖配置。

### 1. 检查 systemd 模板

```bash
ls -l /lib/systemd/system/shadowsocks-libev-server@.service
systemctl list-unit-files '*shadowsocks*'
grep -E '^(CONFFILE|DAEMON_ARGS)=' /etc/default/shadowsocks-libev
```

本次出现过的核心问题是服务默认寻找：

`/etc/shadowsocks-libev/config.json`

而实际创建的配置为：

`/etc/shadowsocks-libev/ss-server.json`

因此，需要先确认 systemd 模板和发行版默认参数，不能假定服务会自动使用新配置。出现 `ERROR: Invalid config path.` 时，应检查路径、配置语法和服务账户对每一级父目录的访问权限。


### 2. 创建服务账户并授予最小必要权限

本次使用 `ssserver` 账户运行代理服务：

```bash
id ssserver >/dev/null 2>&1 || \
  adduser --system --no-create-home --group \
  --shell /usr/sbin/nologin ssserver

chown root:ssserver /etc/shadowsocks-libev
chmod 710 /etc/shadowsocks-libev

chown ssserver:ssserver /etc/shadowsocks-libev/ss-server.json
chmod 600 /etc/shadowsocks-libev/ss-server.json

runuser -u ssserver -- \
  test -r /etc/shadowsocks-libev/ss-server.json \
  && echo "配置文件可读取yes" \
  || echo "配置文件仍无法读取no"
```

目录的执行权限允许服务账户穿过目录访问其有权读取的文件；配置文件只对拥有者开放读取。这是一种与本次服务用户匹配的权限设置示例。修改权限之前要确认实际服务用户和组，避免套用后造成服务无法读取文件或秘密暴露。

### 3. 为实例 A 创建 systemd 覆盖配置

```bash
mkdir -p /etc/systemd/system/shadowsocks-libev-server@ss-server.service.d

cat > /etc/systemd/system/shadowsocks-libev-server@ss-server.service.d/override.conf <<'EOF'
[Service]
DynamicUser=false
User=ssserver
Group=ssserver
EOF

systemctl daemon-reload
systemctl enable --now shadowsocks-libev-server@ss-server.service
```

- `enable`：设置开机自动启动。
- `--now`：同时立即启动服务。

### 4. 检查状态和监听端口

```bash
systemctl --no-pager --full status shadowsocks-libev-server@ss-server.service
ss -lntup | grep -E ':15336\\b'
journalctl -u shadowsocks-libev-server@ss-server.service -n 50 --no-pager
```

本次验证时，目标服务显示 `active (running)`，日志中存在 TCP 和 UDP 在 `0.0.0.0:15336` 监听的信息。

另一个系统自带的服务单元 `shadowsocks-libev.service` 当时也处于运行状态，但只监听本机回环地址 `127.0.0.1:8388` 和 `::1:8388`。它与对外提供服务的 `shadowsocks-libev-server@ss-server.service` 不是同一个监听实例。排错时应明确自己检查的是哪个 unit、配置文件和端口。

---

## 六、UFW 和云防火墙

先查看现有规则：

```bash
ufw status numbered
ufw status verbose
```

确认 SSH 规则存在后，再开放示例代理端口：

```bash
ufw allow 15336/tcp
ufw allow 15336/udp
ufw status
```

应确认规则中存在：
`
22/tcp       ALLOW IN
15336/tcp    ALLOW IN
15336/udp    ALLOW IN
`
保留 SSH 的 `22/tcp` 规则，避免锁死远程登录.

Vultr 控制台如果配置了额外的 Firewall Group，也需要检查该云防火墙是否允许对应 TCP/UDP 端口。系统内 UFW 与云防火墙是两处不同的控制层，任何一层阻断都可能让外部客户端连接失败。

---

## 七、Windows 客户端连接与验证

下载Shadowsocks 客户端，链接https://github.com/shadowsocks/shadowsocks-windows

### 1. 从 Windows 测试 TCP 连通性

在 Windows PowerShell 执行：

```powershell
Test-NetConnection YOUR_SERVER_IP -Port 15336
```

把 `YOUR_SERVER_IP` 换成实际服务器 IP。返回 `TcpTestSucceeded : True` 代表这一端口的 TCP 连接测试成功；它不单独证明 UDP 可用，也不证明代理认证已经成功。

### 2. 设置 Windows Shadowsocks 客户端

在客户端新增服务器，填写：

| 客户端字段 | 应填写的值 |
|---|---|
| Server / Server IP | 自己的服务器 IP |
| Server Port | `15336` |
| Password | 服务器本机保存的服务密码 |
| Encryption Method | `aes-256-gcm` |
| Plugin | None / 留空（如服务端未配置插件） |
| Local Port | 保留客户端默认值，常见为 `1080` |

保存后将该服务器设为当前服务器并开启系统代理。开始排错时可以用 Global 模式进行可控的连接验证，确认工作后再排查 PAC 规则。

### 3. 验证代理链路

以下是 Windows 上的示例命令：

```powershell
# 直接请求（直连是否成功取决于本地网络）
curl.exe --max-time 15 https://api.ipify.org

# 指定本地 SOCKS5 代理并由代理端解析域名
curl.exe --proxy socks5h://127.0.0.1:1080 --max-time 15 https://api.ipify.org
```

在本次实例 A 的测试中，直连请求曾返回连接被重置，而通过 SOCKS5 代理的请求返回 VPS 的公网出口 IP。后者说明该次请求确实经过代理转发。为保护隐私，仓库不保存当时的真实 IP。

如果代理请求失败，依次检查客户端是否启动、服务是否监听、密码/加密方法是否一致、UFW 与云防火墙是否放行，以及 Shadowsocks 服务日志。（实在不会把我融了问gpt）

---

## 八、分享节点给 Clash/Mihomo（很奇怪，shandowsocks很稳定，clash时不时抽风）

若使用支持 Mihomo 配置格式的 Clash 客户端，可以将下列节点条目合并到已有配置文件的 `proxies` 列表中：

注意：
给朋友分享的话别直接给原密码！！！，可以建一个访客端口和密码，或者尝试更好的办法
给朋友分享的话别直接给原密码！！！，可以建一个访客端口和密码，或者尝试更好的办法
给朋友分享的话别直接给原密码！！！，可以建一个访客端口和密码，或者尝试更好的办法

```yaml
mixed-port: 7890
allow-lan: false
mode: rule
log-level: info

proxies:
  - name: "My-VPS"
    type: ss
    server: YOUR_SERVER_IP
    port: 15336
    cipher: aes-256-gcm
    password: "REPLACE_WITH_REAL_PASSWORD"
    udp: true

proxy-groups:
  - name: "PROXY"
    type: select
    proxies:
      - "My-VPS"
      - DIRECT

rules:
  - MATCH,PROXY
```

请把地址和密码替换为自己服务端的实际值。不要把含有真实密码的 YAML 上传到仓库，也不要用这个示例覆盖朋友现有配置文件的全部内容。

少量可信任用户临时共用节点较简单；但共用密码意味着无法单独撤销某个人的权限或单独统计其用量。需要分别管理时，应为不同使用者设置独立密码、端口与服务实例，或选用支持账户管理的方案。

---

## 九、PAC 模式排错

本次 Windows 客户端遇到过 Global 模式可用、PAC 模式不可用的情况。排查方向优先在本机客户端与系统代理设置，而不是立即重装服务器。

当时的修复流程：

1. 在系统托盘中右键点击 Shadowsocks 图标。
2. 在 PAC 菜单里选择 `Update Local PAC from GFWList` 更新本地 PAC。
3. 回到 `System Proxy Mode → PAC`。
4. 确认 `Enable System Proxy` 已启用。
5. 关闭并重新打开浏览器后再次测试。

之后确认 PAC 模式恢复工作。

PAC 会按照规则决定哪些网站直连、哪些经代理。因此，PAC 与 Global 的访问结果本来就可能不同。如果仍异常，可先查看 Windows 代理设置、客户端日志、本地 PAC 更新时间，再对比具体网站匹配的规则。不同客户端版本的菜单名称可能不同。

---

## 十、独立朋友节点与共享 500 GB 限额实验

此部分是另一台 VPS 上的单独实验。该实例使用了不同的系统版本与朋友服务端口；**不要把这里的路径、端口与实例 A 混为一谈**。当前配置的关键思路是个人节点使用 `15336`，朋友节点使用单独的 `15337`，两个服务实例使用不同密码。

### 1. 目标架构

- 朋友的多台设备共用一个总配额。
- 每月按日本时间 1 日重置。
- 超额后只阻断朋友端口，不应影响个人端口。
- 不使用 `nft flush ruleset` 之类会清空全局规则的命令。

### 2. nftables 示例规则

####第 1 步：先检查日本服务器

登录jp节点的 Vultr Console，在终端执行下面的只读检查命令：
```bash
id ssserver

stat -c '%A %U:%G %n' \
  /etc/shadowsocks-libev \
  /etc/shadowsocks-libev/ss-server.json

systemctl is-active shadowsocks-libev-server@ss-server.service

ss -lntup | grep -E ':(15336|15337)\b' || true

ufw status
```

输出看不懂的发给gpt即可

####第 2 步：检查服务器

登录服务器终端，执行下面这组命令：
```bash
id ssserver

systemctl is-active shadowsocks-libev-server@ss-server.service

ss -lntup | grep -E ':(15336|15337)\b' || true

ufw status
```

把输出发给gpt即可，不需要发送任何密码。

gpt会根据现有配置，继续带你创建朋友专用端口、密码和每月 500 GB 限额控制，再把最终节点配置给朋友

接下来先创建朋友节点，但暂时不开放 15337 的公网访问。

####第 3 步：创建朋友专用账户和密码

在服务器的 root@vultr:~# 终端中，完整复制执行下面这段命令：

```bash
# 创建朋友专用系统账户
id ssfriend >/dev/null 2>&1 || \
  adduser --system --no-create-home \
  --ingroup ssserver --shell /usr/sbin/nologin ssfriend

# 确保配置目录允许服务组进入
chown root:ssserver /etc/shadowsocks-libev
chmod 710 /etc/shadowsocks-libev

# 为朋友生成独立随机密码
FRIEND_PASS="$(openssl rand -hex 24)"

# 创建朋友专用配置文件
cat > /etc/shadowsocks-libev/ss-friend.json <<EOF
{
    "server": "0.0.0.0",
    "server_port": 15337,
    "password": "$FRIEND_PASS",
    "timeout": 300,
    "method": "aes-256-gcm",
    "mode": "tcp_and_udp"
}
EOF

# 单独保存朋友密码，只有 root 可读取
printf '%s\n' "$FRIEND_PASS" > /root/ss-friend-password.txt
chmod 600 /root/ss-friend-password.txt

# 保护朋友配置文件
chown ssfriend:ssserver /etc/shadowsocks-libev/ss-friend.json
chmod 600 /etc/shadowsocks-libev/ss-friend.json

# 检查 JSON 格式，不显示密码
python3 -c 'import json; json.load(open("/etc/shadowsocks-libev/ss-friend.json")); print("yes朋友节点配置格式正确")'

```
正常情况下，最后会显示：
`
yes
`

这里生成了全新的密码，与个人节点的密码不同。不要把 /root/ss-password.txt 中的个人密码复制给朋友

####第 4 步：创建朋友专用服务

继续执行：
```bash
mkdir -p /etc/systemd/system/shadowsocks-libev-server@ss-friend.service.d

cat > /etc/systemd/system/shadowsocks-libev-server@ss-friend.service.d/override.conf <<'EOF'
[Service]
DynamicUser=false
User=ssfriend
Group=ssserver
EOF

systemctl daemon-reload
systemctl enable --now shadowsocks-libev-server@ss-friend.service
```

这会启动新的服务实例 shadowsocks-libev-server@ss-friend.service，不会替换你的个人节点。



####第 5 步：检查两个节点是否互不影响

执行下面三条命令：
```bash
systemctl is-active shadowsocks-libev-server@ss-server.service


systemctl --no-pager --full status shadowsocks-libev-server@ss-friend.service


ss -lntup | grep -E ':(15336|15337)\b' || true
```

我们希望看到个人节点仍是 active，朋友节点也显示 active (running)，并且 TCP 和 UDP 分别监听 15336、15337

####第 6 步：检查配置文件权限和限额工具

在服务器的终端中，执行下面这组命令：
```bash
# 检查配置文件和密码文件的权限，不显示密码内容
stat -c '%A %U:%G %n' \
  /etc/shadowsocks-libev/ss-friend.json \
  /root/ss-friend-password.txt

# 检查 nftables 是否已安装
command -v nft
nft --version

# 检查系统版本
grep -E 'PRETTY_NAME|VERSION_ID' /etc/os-release

# 再次确认个人节点没有受到影响
systemctl is-active shadowsocks-libev-server@ss-server.service
```

特别要检查朋友的配置文件是否由 ssfriend 用户拥有或具有正确的读取权限。如果文件仍是 root:root 且权限为 600，就需要先修复，否则服务后续重启时可能无法读取配置。

####第 7 步：只检查防火墙，不修改配置

请在日本服务器终端执行下面三条命令：
```bash
iptables --version

nft list ruleset

systemctl is-active ufw
systemctl is-active nftables || true
```

这些命令只读取当前状态，不会开放或关闭端口，也不会重启服

####第 8 步：创建并检查限额规则

在日本服务器终端中，完整复制执行下面的命令。这一步只创建规则文件并检查语法，暂时不会启用限额规则，也不会影响正在运行的节点。
```bash
cat > /root/ssfriend-quota.nft <<'EOF'
table inet ssfriend_quota {
    quota friend_monthly {
        until 500000000000 bytes;
    }

    chain input {
        type filter hook input priority -10; policy accept;

        tcp dport 15337 quota name "friend_monthly" accept
        udp dport 15337 quota name "friend_monthly" accept

        tcp dport 15337 drop
        udp dport 15337 drop
    }

    chain output {
        type filter hook output priority -10; policy accept;

        tcp sport 15337 quota name "friend_monthly" accept
        udp sport 15337 quota name "friend_monthly" accept

        tcp sport 15337 drop
        udp sport 15337 drop
    }
}
EOF

nft -c -f /root/ssfriend-quota.nft
```

这段规则的设计是：
- 15337 的 TCP、UDP 入站和出站流量共用一个命名配额。
- 总量达到约定的 500,000,000,000 字节(500GB)后，后续匹配的流量会被丢弃。
- 个人节点 15336 不会匹配这些规则。
- 当前还没有设置每月自动重置，也没有处理重启后的额度恢复；这些必须在正式开放朋友节点前补齐

####第 9 步：创建每月流量重置脚本

先创建一个脚本。它会在每月 1 日重置朋友节点的流量计数器，个人节点 15336 不会被重置。
执行：
```bash
cat > /usr/local/sbin/ssfriend-quota-reset <<'EOF'
#!/bin/sh
set -eu

if nft list table inet ssfriend_quota >/dev/null 2>&1; then
    nft reset quota inet ssfriend_quota friend_monthly
else
    echo "朋友节点流量配额表尚未加载no" >&2
    exit 1
fi
EOF

chmod 700 /usr/local/sbin/ssfriend-quota-reset
```

然后检查脚本是否存在：
```bash
ls -l /usr/local/sbin/ssfriend-quota-reset
```

正常应显示类似：
`
-rwx------ 1 root root ... /usr/local/sbin/ssfriend-quota-reset
`

gpt核对了 nftables 的命令文档，nft reset quota inet ssfriend_quota friend_monthly 是重置命名配额的标准命令格式。

请先执行：

```bash
nft --version
nft --help | head -n 25
```

下一步我们先检查当前状态，不修改任何防火墙规则。请执行：
```bash
nft list table inet ssfriend_quota
ufw status numbered
```

第一条如果提示找不到 ssfriend_quota 表，属于预期情况，因为我们还没有正式加载规则。第二条用于确认现有 UFW 规则没有变化

### 如果考虑重启导致的配额重置问题请问gpt

## 不考虑重启情况下设置：

#### 第 1 步：加载朋友节点的限额规则

先加载自定义 nftables 表。这一步不会清空现有防火墙规则，也不会开放朋友端口。
在服务器执行：
```bash
nft -f /root/ssfriend-quota.nft
```

然后检查是否加载成功：
```bash
nft list table inet ssfriend_quota
```

接下来先检查两个命令是否正常。请执行：(本人使用tokyo节点)
```bash
nft list quota inet ssfriend_quota friend_monthly
systemd-analyze calendar '*-*-01 00:05:00 Asia/Tokyo'
```


#### 第 2 步：

创建每月自动重置任务
我们先创建一个专门重置朋友节点配额的 systemd 服务和定时器。它只执行 ssfriend-quota-reset，不会重启 Shadowsocks 服务，也不会重置个人节点的流量。
先执行下面整段命令：
```bash
cat > /etc/systemd/system/ssfriend-quota-reset.service <<'EOF'
[Unit]
Description=Reset monthly Shadowsocks friend quota

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/ssfriend-quota-reset
EOF

cat > /etc/systemd/system/ssfriend-quota-reset.timer <<'EOF'
[Unit]
Description=Reset Shadowsocks friend quota monthly

[Timer]
OnCalendar=*-*-01 00:05:00 Asia/Tokyo
Persistent=true
AccuracySec=1s
Unit=ssfriend-quota-reset.service

[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now ssfriend-quota-reset.timer
```

Persistent=true 会让 systemd 在错过计划执行时间时尝试补执行任务


然后检查定时器是否启用：
```bash
systemctl status ssfriend-quota-reset.timer --no-pager
systemctl list-timers ssfriend-quota-reset.timer --all
```

请在服务器上执行下面三条命令：
```bash
systemctl is-enabled ssfriend-quota-reset.timer
systemctl is-active ssfriend-quota-reset.timer
systemctl list-timers ssfriend-quota-reset.timer --all --no-pager
```

正常情况下，前两条分别应显示：
`
enabled
active
`

第三条还应显示定时器的下次执行时间

执行：
```bash
systemctl start ssfriend-quota-reset.service
systemctl --no-pager --full status ssfriend-quota-reset.service
```

正常情况下，服务状态应显示 
`
status=0/SUCCESS 或 code=exited, status=0/SUCCESS
`

#### 第 3 步：开放朋友专用端口

现在可以开放 15337 的 TCP 和 UDP 端口。执行：
```bash
ufw allow 15337/tcp
ufw allow 15337/udp
```

然后检查：
```bash
ufw status numbered
nft list table inet ssfriend_quota
systemctl is-active ssfriend-quota-reset.timer
```

把这三条命令的输出发给gpt(如果看不懂)
我们预期看到：
- UFW 已开放 15337/tcp 和 15337/udp。
- ssfriend_quota 表仍然存在，里面有朋友节点的限额规则。
- 定时器状态是 active。
确认后，就可以准备朋友使用的 Clash 配置了。朋友只需要获得朋友节点独立密码，不要给他个人节点的密码，也不要把密码贴到这里



#### 第 4 步：生成朋友的 Clash 配置

下面是适用于 Clash Meta / Mihomo 的基础配置。如果朋友已经有自己的配置文件，只需把 proxies 中的节点条目合并进去，不要覆盖原来的全部规则。
替换
`
YOUR-IP 和  替换为朋友专用密码
`

```yaml
mixed-port: 7890
allow-lan: false
mode: rule
log-level: info

proxies:
  - name: "JP-Friend-500G"
    type: ss
    server: YOUR-IP
    port: 15337
    cipher: aes-256-gcm
    password: "替换为朋友专用密码"
    udp: true

proxy-groups:
  - name: "PROXY"
    type: select
    proxies:
      - "JP-Friend-500G"
      - DIRECT

rules:
  - MATCH,PROXY
```


设置密码
在服务器上执行：
```bash
cat /root/ss-friend-password.txt
```

将显示的密码填入配置中的 password 字段，保存后导入 Clash。不要把密码发给gpt，也不要将个人节点15536端口的密码交给朋友。 朋友需要通过你的安全渠道获得这个独立密码。


#### 第 5 步：测试连接

先在服务器上执行：
```bash
systemctl is-active shadowsocks-libev-server@ss-friend.service
ss -lntup | grep ':15337'
```

确认服务是 active，并且 TCP、UDP 监听正常。然后让朋友导入配置，选择 JP-Friend-500G，尝试打开网页。
后续可以用下面的命令检查月度流量配额：
```bash
nft list quota inet ssfriend_quota friend_monthly
```

nftables 的命名配额由多条规则共享，因此入站和出站流量会累加到同一限额中。


请注意，统计的是经过代理端口的网络包流量，会包含协议开销，因此不等于应用实际下载文件的精确大小

 
朋友只需要拿到朋友节点独立密码，不应使用个人节点密码。若多台设备共用这一账号，它们也会共用同一个月度配额。



### 7. 当前实验状态与未完成事项

- 自定义 nftables 表曾成功加载。
- 查询命名配额时曾出现 `used` 字段，说明计数器已记录一定流量。
- 月度重置脚本曾以 `status=0/SUCCESS` 成功执行。
- UFW 曾为朋友端口添加 TCP、UDP 允许规则。
- **超额后真实连接是否被完整阻断尚未完成端到端测试。**
- **nftables 自定义表尚未配置开机自动加载。**
- **重启前已用额度未持久化保存。**
- **尚未实现按客户端 IP 分别统计历史流量。** `ss` 主要适合检查当下的 TCP 连接；UDP 监听输出通常不能列出所有客户端。公网 IP 也不一定对应单独一个人或一台设备。

因此，不要把这一部分当作已完成生产验证的硬限流方案；在补齐重启保护与超额测试之前，不应依赖它确保严格的月度账单上限。

---

## 十一、常用排错命令

### 实例 A：基础 Shadowsocks 服务

```bash
systemctl status shadowsocks-libev-server@ss-server.service --no-pager
systemctl restart shadowsocks-libev-server@ss-server.service
journalctl -u shadowsocks-libev-server@ss-server.service -n 50 --no-pager
ss -lntup | grep -E ':15336\\b'
ufw status verbose
```

### 实例 B：朋友节点及其配额实验

```bash
systemctl is-active shadowsocks-libev-server@ss-friend.service
ss -lntup | grep ':15337'
ss -tnp state established '( sport = :15337 )'
nft list table inet ssfriend_quota
nft list quota inet ssfriend_quota friend_monthly
systemctl is-active ssfriend-quota-reset.timer
systemctl list-timers ssfriend-quota-reset.timer --all --no-pager
journalctl -u shadowsocks-libev-server@ss-friend.service -n 80 --no-pager
journalctl -u ssfriend-quota-reset.service -n 40 --no-pager
```

Windows SOCKS5 测试示例：

```powershell
curl.exe --proxy socks5h://127.0.0.1:1080 --max-time 15 https://api.ipify.org
```

---

## 十二、已知限制和上线前检查

1. **确认服务器实例。** 本文同时记录基础部署和独立朋友节点实验，两者的系统版本、端口与服务配置不同。操作之前先确认当前 SSH 连接的是哪台实例。
2. **不要公开任何真实凭据。** 不要提交服务器 IP / 域名、客户端公网 IP、密码、SSH 私钥、API Token、含连接 IP 的日志或填入真实密码的客户端 YAML。
3. **保留 SSH 回退通道。** 修改 UFW、SSH 或 systemd 配置前先检查现有状态，确认不会锁死远程登录。
4. **检查多个防火墙层。** 外部访问可能同时受 UFW 与云防火墙规则影响。
5. **不要盲目选择软件包升级提示。** 配置冲突应根据差异判断，并保留恢复方案。
6. **区分端口监听和代理成功。** `ss` 或 TCP 探测只能证明特定层面的状态；用客户端请求和服务日志进一步验证代理链路。
7. **配额方案尚未完整验证。** 朋友节点流量配额在超额行为测试、开机加载、重启状态恢复和逐 IP 历史统计方面仍有缺口。
8. **推广链接请透明标注。** 推荐链接可能包含推荐标识，价格与优惠政策以服务商最新页面为准。

## 推荐链接、参考资料与致谢

- **项目仓库：** [VultrVPS_deploy_VPN](https://github.com/lucianosummer/VultrVPS-Deployment-Notes)
- **Vultr 分享 / 推荐链接：** [https://www.vultr.com/?ref=9929462-9J](https://www.vultr.com/?ref=9929462-9J)。这是推荐链接，请以 Vultr 当前条款、价格和优惠说明为准。
- **参考教程：** [sucong426/VPN — 快速搭建个人 VPN 教程](https://github.com/sucong426/VPN)。感谢项目作者整理与分享 VPS、Shadowsocks 入门资料，为本笔记提供学习参考。
- 本仓库是对个人部署与排错过程的独立整理，并非上述参考项目的官方版本或镜像。参考教程包含较早的软件与系统环境，具体命令应以本机版本和官方文档为准。
- 本文没有复制参考项目的完整脚本或截图。如果以后引用其具体代码、图片或文字，应遵循原作者声明的许可与署名要求；未提供明确许可证时，不应默认可以任意复制。
