# 排错指南

## 客户端连不上 vt-x.com:53386

### 1. 检查端口可达

```bash
timeout 4 nc -zv 116.62.11.86 53386
```

### 2. 检查 vt-x.com iptables

```bash
iptables -t nat -L PREROUTING -n -v | grep 53386
iptables -t nat -L POSTROUTING -n -v | grep 3389
```

确保有 TCP 和 UDP 的 DNAT/SNAT 规则。

### 3. 检查 WG 隧道

```bash
# vt-x.com
ping -c 3 192.168.10.200

# OpenWrt
cat /proc/net/nf_conntrack | grep 3389
```

### 4. 检查路由

```bash
# vt-x.com
ip route show | grep 192.168.10
```

应显示：`192.168.10.0/24 dev wg0 scope link`

如果丢失，重新添加：

```bash
ip route add 192.168.10.0/24 dev wg0
```

## 连接错误 0x904

错误码 `0x904` + 扩展码 `0x7` 表示服务器拒绝连接。

### 常见原因与解决

| 原因 | 解决方法 |
|------|----------|
| SecurityLayer=2 但证书无效 | 设置证书绑定或 SecurityLayer=1 |
| fEnableWin8Extensions 与自签名证书冲突 | 删除 fEnableWin8Extensions，或使用受信任证书 |
| 证书私钥不可访问 | 重新生成证书 |
| 防火墙规则缺失 | 添加 "Remote Desktop (TCP-In)" 和 "RDP UDP In" |

## UDP 不工作

### 网络层排查

1. 确认 vt-x.com UDP DNAT 规则存在
2. 确认 OpenWrt nft FORWARD 放行 wg0
3. 确认 10.10 防火墙放行 UDP 3389

### 服务端配置

1. `SelectTransport = 0`（允许 UDP+TCP）
2. `fEnableWin8Extensions = 1`（RDP 8.0 扩展）
3. RDP 证书已绑定且私钥可访问
4. RDP 服务已重启

### 客户端配置

在 Windows 客户端检查：

```cmd
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services\Client" /v fClientDisableUDP
```

如果返回 `0x1`，启用它：

```cmd
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services\Client" /v fClientDisableUDP /t REG_DWORD /d 0 /f
```

## 隧道断开

### WG 握手过期

```bash
wg show | grep handshake
```

如果握手时间超过 2 分钟，检查：

- OpenWrt 网络是否可达（4G/5G 信号）
- vt-x.com 的安全组是否放行 UDP 51820 端口
- 两端密钥是否匹配

### 恢复连接

```bash
# OpenWrt
wg set wg0 peer <vt-x.com publickey> endpoint <vt-x.com IP>:51820
```

## 10.10 局域网 UDP 丢失

如果修改配置后局域网 RDP 的 UDP 也失效：

```powershell
# 1. 恢复默认证书绑定
$r = Get-WmiObject -Namespace root\cimv2\terminalservices -Class Win32_TSGeneralSetting -Filter "TerminalName='RDP-tcp'"
$r.SSLCertificateSHA1Hash = $null
$r.Put()

# 2. 删除可能冲突的策略
reg delete "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" /v SecurityLayer /f 2>nul
reg delete "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" /v SelectTransport /f 2>nul

# 3. 添加默认防火墙规则
netsh advfirewall firewall add rule name="Remote Desktop (TCP-In)" dir=in protocol=tcp localport=3389 action=allow
netsh advfirewall firewall add rule name="Remote Desktop (UDP-In)" dir=in protocol=udp localport=3389 action=allow

# 4. 重启 RDP
net stop TermService /y; net start TermService
```
