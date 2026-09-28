# 计算机基础笔记

覆盖操作系统、网络、容器、数据库、Java 与 Go 等主题的学习笔记。本仓库内容来自不同阶段的整理，建议以目录和主题间链接为入口；示例命令须在实验或受控环境中验证后再用于生产。

## 网络学习路径

1. [网络分层概述](网络/物理+链路/概述.md) → [数据链路层](网络/物理+链路/数据链路层.md) → [VLAN](网络/物理+链路/VLAN.md) → [二三层转发](网络/物理+链路/二三层转发.md)。
2. [IPv4](网络/网络层/IPv4.md) / [IPv6](网络/网络层/IPv6.md) → [传输层](网络/传输层/Transport.md) → [HTTP](网络/应用层/http.md) / [DNS](网络/应用层/DNS.md)。
3. 数据中心方向可继续阅读 [三层架构与 Spine-Leaf](网络/数据中心/三层架构与spine-leaf架构.md)、[VXLAN](网络/网络虚拟化/VXLAN.md) 和 [EVPN + VXLAN](网络/网络虚拟化/EVPNVXLAN.md)。
4. 排障时可从 [ping](网络/工具与诊断/ping.md)、[traceroute](网络/工具与诊断/traceroute.md)、[TCPdump](网络/工具与诊断/TCPdump.md) 与 [iperf3](网络/工具与诊断/iperf3.md) 开始。
