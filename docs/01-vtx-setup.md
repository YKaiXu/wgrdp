# vt-x.com 配置（阿里云 ECS）

## 1. 安装 WireGuard

```bash
apt update && apt install -y wireguard
```

## 2. 生成密钥对

```bash
mkdir -p /etc/wireguard
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey
```

记录公钥：`cat /etc/wireguard/publickey`

## 3. 配置 wg0 接口

创建 `/etc/wireguard/wg0.conf`：

```ini
[Interface]
PrivateKey = <vt-x.com privatekey>
Address = 10.0.0.1/30
ListenPort = 51820
MTU = 1200

# 隧道启动时添加路由
PostUp = ip route add 192.168.10.0/24 dev wg0

[Peer]
PublicKey = <OpenWrt publickey>
AllowedIPs = 10.0.0.2/32, 192.168.10.0/24
PersistentKeepalive = 25
```

启动 WG：

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

## 4. 启用 IP 转发

```bash
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p
```

## 5. 配置 iptables DNAT + SNAT

将公网端口 53386 的 TCP/UDP 流量转发到 10.10:3389：

```bash
# PREROUTING DNAT
iptables -t nat -A PREROUTING -p tcp --dport 53386 -j DNAT --to-destination 192.168.10.200:3389
iptables -t nat -A PREROUTING -p udp --dport 53386 -j DNAT --to-destination 192.168.10.200:3389
iptables -t nat -A POSTROUTING -p tcp --dport 3389 -d 192.168.10.200 -j SNAT --to-source 10.0.0.1
iptables -t nat -A POSTROUTING -p udp --dport 3389 -d 192.168.10.200 -j SNAT --to-source 10.0.0.1

# FORWARD 放行（可选，默认 ACCEPT 则不需要）
iptables -A FORWARD -p tcp --dport 3389 -j ACCEPT
iptables -A FORWARD -p udp --dport 3389 -j ACCEPT
```

持久化规则（可选）：

```bash
apt install -y iptables-persistent
iptables-save > /etc/iptables/rules.v4
```

## 6. 添加路由

```bash
ip route add 192.168.10.0/24 dev wg0
```

## 7. 验证

```bash
# 检查 WG 状态
wg show

# 测试到目标主机的连通性
ping -c 3 192.168.10.200

# 测试 RDP 端口
timeout 3 nc -zv 192.168.10.200 3389
```
