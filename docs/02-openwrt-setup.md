# OpenWrt 配置（WRT 路由器）

## 1. 安装 WireGuard

```bash
opkg update
opkg install wireguard-tools kmod-wireguard luci-proto-wireguard
```

## 2. 生成密钥对

```bash
wg genkey | tee /etc/wireguard/private.key | wg pubkey > /etc/wireguard/public.key
cat /etc/wireguard/public.key
```

记录公钥。

## 3. 配置 wg0 接口

```bash
# 创建 WG 接口
ip link add wg0 type wireguard
ip addr add 10.0.0.2/30 dev wg0
wg set wg0 private-key /etc/wireguard/private.key
wg set wg0 peer <vt-x.com publickey> \
  allowed-ips 10.0.0.0/30 \
  endpoint <vt-x.com public IP>:51820 \
  persistent-keepalive 25
ip link set mtu 1200 up dev wg0
```

防火墙放行 WG 端口：

```bash
uci add firewall rule
uci set firewall.@rule[-1].name="Allow-WireGuard"
uci set firewall.@rule[-1].src="wan"
uci set firewall.@rule[-1].dest_port="51820"
uci set firewall.@rule[-1].proto="udp"
uci set firewall.@rule[-1].target="ACCEPT"
uci commit firewall
/etc/init.d/firewall reload
```

## 4. nftables FORWARD 规则

```bash
# 放行 wg0 与其他接口之间的转发
nft insert rule inet fw4 forward oifname "wg0" accept
nft insert rule inet fw4 forward iifname "wg0" accept
```

持久化（OpenWrt 重启后 nft 规则会丢失。添加到 `/etc/firewall.user`）：

```bash
cat >> /etc/firewall.user << 'EOF'
# WireGuard forward rules
nft insert rule inet fw4 forward oifname "wg0" accept
nft insert rule inet fw4 forward iifname "wg0" accept
EOF
chmod +x /etc/firewall.user
```

## 5. 验证

```bash
# WG 握手状态
wg show

# 预期输出：
# interface: wg0
#   public key: <OpenWrt publickey>
#   listening port: 57956
# peer: <vt-x.com publickey>
#   endpoint: <vt-x.com IP>:51820
#   allowed ips: 10.0.0.0/30
#   latest handshake: X seconds ago
#   transfer: XXX received, XXX sent

# conntrack 检查
cat /proc/net/nf_conntrack | grep 3389
```
