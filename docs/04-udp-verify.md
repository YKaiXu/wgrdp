# UDP 通道验证

## 1. 检查 WG 隧道状态

```bash
# vt-x.com
wg show | grep -E 'handshake|transfer'

# OpenWrt
wg show | grep -E 'handshake|transfer'
```

握手时间应为"X seconds ago"（而不是 minutes）。

## 2. 检查三端 conntrack

### vt-x.com

```bash
cat /proc/net/nf_conntrack | grep 3389
```

期望结果（UDP 条目存在且 ASSURED）：

```
ipv4     2 udp      17 XX src=<客户端IP> dst=<vt-x公网IP> sport=XXXX dport=53386 src=192.168.10.200 dst=10.0.0.1 sport=3389 dport=XXXX [ASSURED]
ipv4     2 tcp      6 299 ESTABLISHED src=<客户端IP> dst=<vt-x公网IP> sport=XXXX dport=53386 src=192.168.10.200 dst=10.0.0.1 sport=3389 dport=XXXX [ASSURED]
```

### OpenWrt

```bash
cat /proc/net/nf_conntrack | grep 3389
```

期望结果：

```
ipv4     2 udp      17 XX src=10.0.0.1 dst=192.168.10.200 sport=XXXX dport=3389 packets=N bytes=N src=192.168.10.200 dst=10.0.0.1 sport=3389 dport=XXXX packets=M bytes=M [ASSURED]
```

`[ASSURED]` 表示双向流量已交换。

## 3. 检查 iptables 计数

```bash
# vt-x.com
iptables -t nat -L PREROUTING -n -v | grep 53386
iptables -t nat -L POSTROUTING -n -v | grep 3389
```

计数器应持续增加。

## 4. Windows RDP 连接信息

在 Windows RDP 连接栏点击左上角图标 → 连接信息：

- **传输协议：** 显示 TCP（主控制通道）
- UDP 作为图形加速通道在后台工作，包量可通过任务管理器观察

## 5. 持续监控

```bash
# 实时监控 conntrack（每 2 秒刷新）
watch -n 2 'cat /proc/net/nf_conntrack | grep 3389'
```

如果 UDP 包数（packets=N）持续增长，说明 UDP 通道正在传输数据。
