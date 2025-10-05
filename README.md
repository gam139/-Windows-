光猫拨号 + 路由器 + 多网卡连接方案（Windows节点场景）
在部署 Pi Node 节点时，如果你的电脑配备了三张网卡，而光猫负责拨号、路由器提供 LAN 接口，那么你可以通过策略配置实现服务隔离、流量分流和资源最大化利用。本文将介绍网络拓扑、出网逻辑、服务绑定建议与实践操作。

🧭 网络拓扑设想
代码
[光猫] ←PPPoE拨号→ [路由器]
                          ├── LAN1 ← 网卡A（eth0）
                          ├── LAN2 ← 网卡B（eth1）
                          └── LAN3 ← 网卡C（eth2）
光猫负责拨号，获取公网 IP

路由器通过 WAN 接口连接光猫，LAN 接口组成局域网（如 192.168.1.x）

节点电脑的三张网卡分别连接到路由器的三个 LAN 口

✅ 能否上网分析
Windows 默认只使用一张网卡出网（由 Interface Metric 决定）

其余网卡虽然连接了 LAN，但不会主动用于出网，除非你做策略绑定

三网卡连接同一网段不会自动负载均衡，需手动配置

🧠 服务绑定建议
网卡	功能分工	是否出网	建议绑定方式
eth0	控制面板 + VPN	✅ 是	设置最低 Metric，默认网关
eth1	Docker容器隔离	❌ 否	Hyper-V交换机或WSL路由
eth2	frpc公网穿透	❌ 否	frpc.ini 中绑定 local_ip
🛠️ 实践操作建议
1. 设置静态 IP（避免冲突）
powershell
New-NetIPAddress -InterfaceAlias "eth0" -IPAddress 192.168.1.100 -PrefixLength 24 -DefaultGateway 192.168.1.1
New-NetIPAddress -InterfaceAlias "eth1" -IPAddress 192.168.1.101 -PrefixLength 24
New-NetIPAddress -InterfaceAlias "eth2" -IPAddress 192.168.1.102 -PrefixLength 24
2. 设置跃点数（Interface Metric）
powershell
Set-NetIPInterface -InterfaceAlias "eth0" -InterfaceMetric 10
Set-NetIPInterface -InterfaceAlias "eth1" -InterfaceMetric 20
Set-NetIPInterface -InterfaceAlias "eth2" -InterfaceMetric 30
3. frpc.ini 中绑定 IP
ini
[pi-node]
type = tcp
local_ip = 192.168.1.102
local_port = 31400
remote_port = 31400
4. Docker 网络绑定建议
方法一：WSL2 路由控制
在 WSL2 中设置容器网段为 192.168.1.101

使用 iptables 或 route 控制容器流量走 eth1

方法二：Hyper-V 虚拟交换机绑定网卡
打开 Hyper-V 管理器 → 创建外部虚拟交换机

绑定到 eth1

在 Docker Desktop 设置中选择该交换机作为默认网络

5. Pi Node 控制面板出网控制
Pi Node 默认使用系统默认网关出网

设置 eth0 为最低 Metric 即可确保其走 eth0

📘 附录：命令速查
powershell
# 查看网卡信息
Get-NetIPInterface | Format-Table -AutoSize

# 设置静态 IP
New-NetIPAddress -InterfaceAlias "eth1" -IPAddress 192.168.1.101 -PrefixLength 24

# 设置跃点数
Set-NetIPInterface -InterfaceAlias "eth1" -InterfaceMetric 20
