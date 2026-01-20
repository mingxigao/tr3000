# OpenWrt CI (Cudy TR3000 v1)

本仓库用于 GitHub Actions 自动编译 TR3000 v1 OpenWrt 固件，并内置：
- USB 网络共享/上网卡常用驱动（RNDIS / iPheth / QMI / MBIM / ECM/NCM 等）
- Samba4（默认强制 SMB3 加密）
- WireGuard（方案 B：只打包，不自动写任何 VPN 配置）

> 安全建议：不要把 SMB(445) 端口直接暴露到公网。推荐用 WireGuard 先把外网设备接入到 LAN，再访问 Samba。

## WireGuard 配置样例（UCI）

下面给出两种典型用法：
- 路由器作为 **WireGuard 服务端**（手机/电脑从外网连回家）
- 路由器作为 **WireGuard 客户端**（路由器连到你已有的外部 WG 服务器）

所有示例都假设：
- 路由器 LAN 网段为 `192.168.1.0/24`（按需改）
- WireGuard 虚拟网段为 `10.6.0.0/24`（按需改）
- WireGuard 接口名为 `wg0`

### 0) 生成密钥（在路由器上）

```sh
umask 077
wg genkey | tee /etc/wireguard/wg0.key | wg pubkey > /etc/wireguard/wg0.pub
cat /etc/wireguard/wg0.pub
```

- `wg0.key` 是私钥（不要泄露）
- `wg0.pub` 是公钥（可发给对端）

---

## A. 路由器作为 WireGuard 服务端（推荐远程访问家庭 Samba）

适用：你希望手机/电脑在外网连回家，访问 LAN 里设备（包括 Samba）。

### A1) 网络接口（wg0）

把下面的占位符替换：
- `<SERVER_PRIVKEY>`：路由器私钥
- `<PEER1_PUBKEY>`：手机/电脑公钥

```sh
uci -q delete network.wg0
uci set network.wg0='interface'
uci set network.wg0.proto='wireguard'
uci set network.wg0.private_key='<SERVER_PRIVKEY>'
uci add_list network.wg0.addresses='10.6.0.1/24'

# peer1（手机/电脑）
uci -q delete network.wgpeer1
uci set network.wgpeer1='wireguard_wg0'
uci set network.wgpeer1.public_key='<PEER1_PUBKEY>'
uci add_list network.wgpeer1.allowed_ips='10.6.0.2/32'
uci set network.wgpeer1.persistent_keepalive='25'

uci commit network
/etc/init.d/network restart
```

### A2) 防火墙（允许 VPN 进入 & 访问 LAN）

```sh
# 允许外网访问 WireGuard 监听端口（默认 51820/udp）
uci -q delete firewall.wg_in
uci set firewall.wg_in='rule'
uci set firewall.wg_in.name='Allow-WireGuard-Inbound'
uci set firewall.wg_in.src='wan'
uci set firewall.wg_in.proto='udp'
uci set firewall.wg_in.dest_port='51820'
uci set firewall.wg_in.target='ACCEPT'

# 新建 vpn zone 并允许 vpn -> lan 转发
uci -q delete firewall.vpn
uci set firewall.vpn='zone'
uci set firewall.vpn.name='vpn'
uci add_list firewall.vpn.network='wg0'
uci set firewall.vpn.input='ACCEPT'
uci set firewall.vpn.output='ACCEPT'
uci set firewall.vpn.forward='ACCEPT'

uci -q delete firewall.vpn_lan
uci set firewall.vpn_lan='forwarding'
uci set firewall.vpn_lan.src='vpn'
uci set firewall.vpn_lan.dest='lan'

uci commit firewall
/etc/init.d/firewall restart
```

> 如果路由器没有公网 IP（在运营商 NAT 后面），你需要：
> - 在上级路由器做 UDP 51820 端口转发到此路由器；或
> - 使用有公网 IP 的 VPS 做中转（路由器改为客户端模式）。

### A3) 手机/电脑端配置要点

手机/电脑端（peer1）一般配置为：
- Address：`10.6.0.2/32`
- Endpoint：`<你的公网IP或域名>:51820`
- AllowedIPs：建议至少包含 `192.168.1.0/24`（访问家里 LAN）和 `10.6.0.0/24`
- Persistent Keepalive：`25`

### A4) 验证

```sh
wg show
ip addr show wg0
logread | grep -i wireguard || true
```

VPN 连上后：
- 直接访问 LAN 内的 Samba（不需要暴露 445 到公网）

---

## B. 路由器作为 WireGuard 客户端（连接到外部 WG 服务器）

适用：你已经有一个 WireGuard 服务器（例如 VPS/另一台路由器），TR3000 作为客户端连过去。

把下面的占位符替换：
- `<CLIENT_PRIVKEY>`：本路由器私钥
- `<SERVER_PUBKEY>`：对端服务器公钥
- `<SERVER_ENDPOINT>`：对端服务器公网地址，例如 `example.com:51820`

### B1) 网络接口（wg0）

```sh
uci -q delete network.wg0
uci set network.wg0='interface'
uci set network.wg0.proto='wireguard'
uci set network.wg0.private_key='<CLIENT_PRIVKEY>'
uci add_list network.wg0.addresses='10.6.0.2/32'

uci -q delete network.wgpeer_server
uci set network.wgpeer_server='wireguard_wg0'
uci set network.wgpeer_server.public_key='<SERVER_PUBKEY>'
uci set network.wgpeer_server.endpoint_host='<SERVER_ENDPOINT>'
uci set network.wgpeer_server.persistent_keepalive='25'

# 只把“需要走 VPN 的网段”写进 allowed_ips
# - 如果只是远程访问家里 LAN：写对端让你访问的网段
# - 如果要全局代理（不建议默认这样做）：才写 0.0.0.0/0
uci add_list network.wgpeer_server.allowed_ips='10.6.0.0/24'
uci add_list network.wgpeer_server.allowed_ips='192.168.1.0/24'

uci commit network
/etc/init.d/network restart
```

### B2) 防火墙（通常不需要开放 WAN 入站）

客户端模式一般不需要在本路由器 WAN 上开放 51820 入站，因为连接是由本机主动发起的。

如果你希望 VPN 上的设备反向访问本路由器 LAN，需要根据你的拓扑决定是否增加 zone/forwarding。

---

## Samba 安全说明（已内置）

本固件通过 `uci-defaults` 写入 Samba4 安全默认值（SMB3 加密为 required）。
- 好处：传输默认加密、更安全
- 代价：非常老旧不支持 SMB3 加密的客户端会连接失败

如果你需要兼容旧客户端（不推荐在公网场景），可以把策略从 `required` 调整为 `desired`，或只对某个 share 单独要求加密。
