## 介绍

EVPN（Ethernet Virtual Private Network）是一种用于二层网络互联的VPN技术。EVPN技术采用类似于BGP/MPLS IP VPN的机制，在BGP协议的基础上定义了一种新的NLRI（Network Layer Reachability Information，网络层可达信息）即EVPN NLRI，EVPN NLRI定义了几种新的BGP EVPN路由类型，用于处在二层网络的不同站点之间的MAC地址学习和发布。

原有的VXLAN实现方案没有控制平面，是通过数据平面的流量泛洪进行VTEP发现和主机信息（包括IP地址、MAC地址、VNI、网关VTEP IP地址）学习的，这种方式导致数据中心网络存在很多泛洪流量。为了解决这一问题，VXLAN引入了EVPN作为控制平面，通过在VTEP之间交换BGP EVPN路由实现VTEP的自动发现、主机信息相互通告等特性，从而避免了不必要的数据流量泛洪。

综上所述，EVPN通过扩展BGP协议新定义了几种BGP EVPN路由，这些BGP EVPN路由可以用于传递VTEP地址和主机信息，因此EVPN应用于VXLAN网络中，可以使VTEP发现和主机信息学习从数据平面转移到控制平面。

与传统VXLAN网络相比，EVPN VXLAN网络可以实现隧道的自动建立与关联，简化配置。

**MP-BGP基本概念**
在深入理解EVPN的工作原理前，我们先对MP-BGP（MultiProtocol BGP）做下简单回顾：传统的BGP-4使用Update报文在对等体之间交换路由信息。一条Update报文可以通告一类具有相同路径属性的可达路由，这些路由放在NLRI（Network Layer Reachable Information，网络层可达信息）字段中。因为[BGP-4](https://zhida.zhihu.com/search?content_id=235177218&content_type=Article&match_order=2&q=BGP-4&zhida_source=entity)只能管理IPv4[单播路由](https://zhida.zhihu.com/search?content_id=235177218&content_type=Article&match_order=1&q=%E5%8D%95%E6%92%AD%E8%B7%AF%E7%94%B1&zhida_source=entity)信息，为了提供对多种网络层协议的支持（例如IPv6、[组播](https://zhida.zhihu.com/search?content_id=235177218&content_type=Article&match_order=1&q=%E7%BB%84%E6%92%AD&zhida_source=entity)），发展出了MP-BGP。[MP-BGP](https://zhida.zhihu.com/search?content_id=235177218&content_type=Article&match_order=5&q=MP-BGP&zhida_source=entity)在BGP-4基础上对NLRI作了新扩展。玄机就在于新扩展的NLRI上，扩展之后的NLRI增加了地址族的描述，可以用来区分不同的网络层协议，例如IPv6[单播地址族](https://zhida.zhihu.com/search?content_id=235177218&content_type=Article&match_order=1&q=%E5%8D%95%E6%92%AD%E5%9C%B0%E5%9D%80%E6%97%8F&zhida_source=entity)、VPN实例地址族等。  

类似的，EVPN也是借用了MP-BGP的机制，在L2VPN地址族下定义了新的子地址族——EVPN地址族，在这个地址族下又新增了一种NLRI，即EVPN NLRI。EVPN NLRI定义了几种BGP EVPN路由类型，这些路由可以携带主机IP、MAC、VNI、VRF等信息。这样，当一个VTEP学习到下挂的主机的IP、MAC地址信息后，就可以通过MP-BGP路由将这些信息发送给其他的VTEP，从而在控制平面实现主机IP、MAC地址的学习，抑制了[数据平面](https://zhida.zhihu.com/search?content_id=235177218&content_type=Article&match_order=1&q=%E6%95%B0%E6%8D%AE%E5%B9%B3%E9%9D%A2&zhida_source=entity)的泛洪。

## BGP EVPN路由

在EVPN NLRI中定义了如下几种应用于VXLAN控制平面的BGP EVPN路由类型：

**Type2路由——MAC/IP路由**

该类型路由的报文格式如下图所示：

图2-7 MAC/IP路由的报文格式  
![](../../assets/网络/网络虚拟化/EVPNVXLAN/08aee492934bf4a67555c1e1dc21390d919e6b0e)

各字段的解释如下表所示：

| 字段                          | 说明                                                                                                                |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Route Distinguisher         | 该字段为EVPN实例下设置的RD（Route Distinguisher）值。                                                                           |
| Ethernet Segment Identifier | 该字段为当前设备与对端连接定义的唯一标识。                                                                                             |
| Ethernet Tag ID             | 该字段为当前设备上实际配置的VLAN ID。在此路由中Ethernet Tag ID值为全0。<br><br>说明：<br><br>对于BGP EVPN Type2路由，不支持与Ethernet Tag ID为非0的设备对接。 |
| MAC Address Length          | 该字段为此路由携带的主机MAC地址的长度。                                                                                             |
| MAC Address                 | 该字段为此路由携带的主机MAC地址。                                                                                                |
| IP Address Length           | 该字段为此路由携带的主机IP地址的掩码长度。                                                                                            |
| IP Address                  | 该字段为此路由携带的主机IP地址。                                                                                                 |
| MPLS Label1                 | 该字段为此路由携带的二层VNI。                                                                                                  |
| MPLS Label2                 | 该字段为此路由携带的三层VNI。                                                                                                  |

该类型路由在VXLAN控制平面中的作用包括：

- 主机MAC地址通告

    要实现同子网主机的二层互访，两端VTEP需要相互学习主机MAC。作为BGP EVPN对等体的VTEP之间通过交换MAC/IP路由，可以相互通告已经获取到的主机MAC。其中，MAC Address Length和MAC Address字段为主机MAC地址。

- 主机ARP通告

    MAC/IP路由可以同时携带主机MAC地址+主机IP地址，因此该路由可以用来在VTEP之间传递主机ARP表项，实现主机ARP通告。其中，MAC Address和MAC Address Length字段为主机MAC地址，IP Address和IP Address Length字段为主机IP地址。此时的MAC/IP路由也称为ARP类型路由。主机ARP通告主要用于以下两种场景：

    1. ARP广播抑制。当三层网关学习到其子网下的主机ARP时，生成主机信息（包含主机IP地址、主机MAC地址、二层VNI、网关VTEP IP地址），然后通过传递ARP类型路由将主机信息同步到二层网关上。这样当二层网关再收到ARP请求时，先查找是否存在目的IP地址对应的主机信息，如果存在，则直接将ARP请求报文中的广播MAC地址替换为目的单播MAC地址，实现广播变单播，达到ARP广播抑制的目的。

    2. 分布式网关场景下的虚拟机迁移。当一台虚拟机从当前网关迁移到另一个网关下之后，新网关学习到该虚拟机的ARP（一般通过虚拟机发送免费ARP实现），并生成主机信息（包含主机IP地址、主机MAC地址、二层VNI、网关VTEP IP地址），然后通过传递ARP类型路由将主机信息发送给虚拟机的原网关。原网关收到后，感知到虚拟机的位置发生变化，触发ARP探测，当探测不到原位置的虚拟机时，撤销原位置虚拟机的ARP和主机路由。

- 主机IP路由通告

    在分布式网关场景中，要实现跨子网主机的三层互访，两端VTEP（作为三层网关）需要互相学习主机IP路由。作为BGP EVPN对等体的VTEP之间通过交换MAC/IP路由，可以相互通告已经获取到的主机IP路由。其中，IP Address Length和IP Address字段为主机IP路由的目的地址，同时MPLS Label2字段必须携带三层VNI。此时的MAC/IP路由也称为IRB（Integrated Routing and Bridge）类型路由。

    ![](../../assets/网络/网络虚拟化/EVPNVXLAN/c08381bcffe3e54f253712e520c00ee30b1dc39a)

    ARP类型路由携带的有效信息有：主机MAC地址+主机IP地址+二层VNI；IRB类型路由携带的有效信息有：主机MAC地址+主机IP地址+二层VNI+三层VNI。因此，IRB类型路由包含着ARP类型路由，不仅可以用于主机IP路由通告，也能用于主机ARP通告。

- ND表项扩散

    MAC/IP路由可以同时携带主机MAC地址+主机IPv6地址，因此该路由可以用来在VTEP之间传递ND表项，实现ND表项扩散。其中，MAC Address和MAC Address Length字段为主机MAC地址，IP Address和IP Address Length字段为主机IPv6地址。此时的MAC/IP路由也称为ND类型路由。ND表项扩散主要用于以下场景：

    - NS组播抑制。当VXLAN网关设备收集到本地IPv6主机的信息后，生成ND表或ND代答表，然后通过MAC/IP路由进行扩散，其他VXLAN网关（BGP EVPN对等体）收到该路由后生成本地的ND代答表。这样，当VXLAN网关再收到NS报文时先查找本地的ND代答表，查找成功就直接进行ND代答或组播转单播处理，从而减少或抑制NS报文洪泛。

    - 防止ND欺骗攻击。ND欺骗攻击是指攻击者将自己的MAC地址与某一主机的IPv6地址相关联，从而使发往该IPv6地址的任何流量都发送给攻击者。通过ND扩散功能，VXLAN网关之间可以同步同一IPv6主机的ND代答表。当攻击者上线后，针对同一IPv6主机，会重复生成ND代答表并扩散到其他VXLAN网关。这样通过ND代答表冲突检测触发IPv6地址冲突告警，进而提醒用户可能存在ND欺骗攻击。

    - 分布式网关场景下的IPv6虚拟机迁移。当一台IPv6虚拟机从当前网关迁移到另一个网关下之后，该虚拟机会主动发送免费NA报文，新网关收到后生成ND表，并通过MAC/IP路由扩散给原网关。原网关收到后，感知到IPv6虚拟机的位置发生变化，触发NUD探测。当探测不到原位置的IPv6虚拟机时，删除本地ND表，并通过MAC/IP路由扩散给新网关，新网关收到后删除老的ND表。

- 主机IPv6路由通告

    在分布式网关场景中，要实现跨子网IPv6主机的三层互访，网关设备需要互相学习主机IPv6路由。作为BGP EVPN对等体的VTEP之间通过交换MAC/IP路由，可以相互通告已经获取到的主机IPv6路由。其中，IP Address Length和IP Address字段为主机IPv6路由的目的地址，同时MPLS Label2字段必须携带三层VNI。此时的MAC/IP路由也称为IRBv6类型路由。

    ![](../../assets/网络/网络虚拟化/EVPNVXLAN/c08381bcffe3e54f253712e520c00ee30b1dc39a)

    ND类型路由携带的有效信息有：主机MAC地址+主机IPv6地址+二层VNI；IRBv6类型路由携带的有效信息有：主机MAC地址+主机IPv6地址+二层VNI+三层VNI。因此，IRBv6类型路由包含着ND类型路由，不仅可以用于主机IPv6路由通告，也能用于ND表项扩散。

**Type3路由——Inclusive Multicast路由**

该类型路由是由前缀和PMSI属性组成，报文格式如下图所示：

图2-8 Inclusive Multicast路由的报文格式  
![](../../assets/网络/网络虚拟化/EVPNVXLAN/e82fa56d3151d8cd6f92c8fb66522cb58829616c)

各字段的解释如下表所示：

|字段|说明|
|---|---|
|Route Distinguisher|该字段为EVPN实例下设置的RD（Route Distinguisher）值。|
|Ethernet Tag ID|该字段为当前设备上的VLAN ID。在此路由中Ethernet Tag ID值为全0。<br><br>说明：<br><br>对于BGP EVPN Type3路由，不支持与Ethernet Tag ID为非0的设备对接。|
|IP Address Length|该字段为此路由携带的本端VTEP IP地址的掩码长度。|
|Originating Router's IP Address|该字段为此路由携带的本端VTEP IP地址。|
|Flags|该字段为标志位，标识当前隧道是否需要叶子节点信息。<br><br>在VXLAN场景中，该字段没有实际意义。|
|Tunnel Type|该字段为此路由携带的隧道类型。|
|MPLS Label|该字段为此路由携带的二层VNI。|
|Tunnel Identifier|该字段为此路由携带的隧道信息。目前，在VXLAN场景中，该字段也是本端VTEP IP地址。|

该类型路由在VXLAN控制平面中主要用于VTEP的自动发现和VXLAN隧道的动态建立。作为BGP EVPN对等体的VTEP，通过Inclusive Multicast路由互相传递二层VNI和VTEP IP地址信息。其中，Originating Router's IP Address字段为本端VTEP IP地址，MPLS Label字段为二层VNI。如果对端VTEP IP地址是三层路由可达的，则建立一条到对端的VXLAN隧道。同时，本端会创建一个基于VNI的头端复制表并将对端VTEP IP地址加入其中，用于后续BUM报文转发。

**Type5路由——IP前缀路由**

该类型路由的报文格式如下图所示：

图2-9 IP前缀路由的报文格式  
![](../../assets/网络/网络虚拟化/EVPNVXLAN/df9a927fb3cab1f2ef8bc1ee12309e487147a21c)

各字段的解释如下表所示：

|字段|说明|
|---|---|
|Route Distinguisher|该字段为VPN实例下设置的RD（Route Distinguisher）值。|
|Ethernet Segment Identifier|该字段为当前设备与对端连接定义的唯一标识。|
|Ethernet Tag ID|该字段为当前设备上实际配置的VLAN ID。|
|IP Prefix Length|该字段为此路由携带的IP前缀掩码长度。|
|IP Prefix|该字段为此路由携带的IP前缀。|
|GW IP Address|该字段为默认网关地址。该字段在VXLAN场景中没有实际意义。|
|MPLS Label|该字段为此路由携带的三层VNI。|

该类型路由的IP Prefix Length和IP Prefix字段既可以携带主机IP地址，也可以携带网段地址：

- 当携带主机IP地址时，该类型路由在VXLAN控制平面中的作用与IRB类型路由是一样的，主要用于分布式网关场景中的主机IP路由通告。

- 当携带网段地址时，通过传递该类型路由，可以实现VXLAN网络中的主机访问外部网络。
