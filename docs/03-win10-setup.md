# 10.10 Windows RDP 服务器配置

## 1. RDP UDP 策略设置

### 方法 A：组策略（推荐）
运行 `gpedit.msc` → 计算机配置 → 管理模板 → Windows 组件 → 远程桌面服务 → 远程桌面会话主机 → 连接 → **选择 RDP 传输协议** → 已启用 → 传输类型：**同时使用 UDP 和 TCP**

### 方法 B：注册表
```powershell
# 启用 UDP + TCP 传输
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" /v SelectTransport /t REG_DWORD /d 0 /f
```

## 2. 启用 RDP 8.0 扩展

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v fEnableWin8Extensions /t REG_DWORD /d 1 /f
```

## 3. 生成并绑定自签名证书

生成包含公网域名的证书：

```powershell
# 生成证书（替换 vt-x.com 为你的域名）
$cert = New-SelfSignedCertificate -DnsName "vt-x.com", "116.62.11.86", "<计算机名>", "<内网 IP>" -CertStoreLocation "Cert:\LocalMachine\My" -NotAfter (Get-Date).AddYears(5)

# 记录 Thumbprint
$cert.Thumbprint
```

绑定到 RDP：

```powershell
# 通过 WMI 绑定证书（重启后持久化）
$r = Get-WmiObject -Namespace root\cimv2\terminalservices -Class Win32_TSGeneralSetting -Filter "TerminalName='RDP-tcp'"
$r.SSLCertificateSHA1Hash = "<Thumbprint>"
$r.Put()
```

## 4. 防火墙规则

恢复默认 RDP 防火墙规则：

```powershell
netsh advfirewall firewall add rule name="Remote Desktop (TCP-In)" dir=in protocol=tcp localport=3389 action=allow
netsh advfirewall firewall add rule name="Remote Desktop (UDP-In)" dir=in protocol=udp localport=3389 action=allow
```

## 5. 重启 RDP 服务

```powershell
net stop TermService /y; net start TermService
```

## 6. 验证配置

```powershell
# 检查证书绑定
$r = Get-WmiObject -Namespace root\cimv2\terminalservices -Class Win32_TSGeneralSetting -Filter "TerminalName='RDP-tcp'"
Write-Host "Certificate:" $r.SSLCertificateSHA1Hash

# 检查 UDP 监听
netstat -ano | findstr ":3389" | findstr UDP

# 检查 Policy
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" /v SelectTransport
reg query "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v fEnableWin8Extensions
```
