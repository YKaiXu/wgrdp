# wgrdp - WireGuard RDP Tunnel with UDP Acceleration

通过 WireGuard 隧道 + DNAT 端口转发，实现 RDP 远程桌面连接，并启用 RDP UDP 多通道加速。

## 架构

```
客户端 (Windows)
    │ TCP+UDP :53386
    ▼
vt-x.com (阿里云 ECS)
    │ iptables DNAT :53386 → 10.10:3389
    │ SNAT → 10.0.0.1
    ▼ WireGuard 隧道
OpenWrt (WRT 路由器, 4G/5G)
    │ nft FORWARD wg0 → br-lan
    ▼
10.10 (Windows RDP 服务器)
    │ 证书 CN=vt-x.com 绑定
    │ fEnableWin8Extensions=1
    │ SelectTransport=0 (UDP+TCP)
```

## 文件结构

```
wgrdp/
├── README.md              # 本文档
├── docs/
│   ├── 01-vtx-setup.md    # vt-x.com 配置
│   ├── 02-openwrt-setup.md # OpenWrt 配置
│   ├── 03-win10-setup.md   # 10.10 RDP 服务器配置
│   ├── 04-udp-verify.md    # UDP 通道验证
│   └── 05-troubleshooting.md # 排错指南
├── scripts/
│   ├── vtx-setup.sh       # vt-x.com 一键配置
│   ├── openwrt-setup.sh   # OpenWrt 一键配置
│   └── win10-rdp-udp.ps1  # 10.10 一键配置 (PowerShell)
└── configs/
    ├── vtx-wg0.conf       # vt-x.com WG 配置示例
    ├── openwrt-wg0.conf   # OpenWrt WG 配置示例
    └── frpc.toml           # frp 配置参考（备选方案）
```
