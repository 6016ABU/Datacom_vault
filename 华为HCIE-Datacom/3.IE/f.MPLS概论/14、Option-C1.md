
# 一、PE之间建立多跳MP-EBGP邻居

## 配置详解
1. 底层使用IGP互联互通（OSPF，ISIS）
2. 配置LDP协议，PE，P，ASBR之间都需要配置
3. PE与CE之间通过IGP或者EBGP传递路由
4. 同一个AS内部PE、P、ASBR之间建立RR-IBGP邻居关系
5. ASBR1与ASBR2之间建立EBGP邻居关系，关闭RT值校验
	1. network通告本端PE的Loopback地址
	2. 对来自本端PE的路由分配标签，使能标签路由能力，传递给对端ASBR
	3. 对来自对端ASBR的路由重新分配标签，使能标签路由能力，传递给本端PE
6. ASBR1与ASBR2互联接口使能mpls
7. PE1与PE2建立多跳MP-EBGP邻居关系（VPNv4）

## 拓扑
 ![](assets/14、Option-C1/file-20251211001209765.png)
### 配置
**PE1  **
```
ip vpn-instance A  
 ipv4-family  
  route-distinguisher 100:1  
  vpn-target 1:1 export-extcommunity  
  vpn-target 1:1 import-extcommunity  
#  
bgp 10  
 peer 4.4.4.4 as-number 10   
 peer 4.4.4.4 connect-interface LoopBack0  
 peer 7.7.7.7 as-number 100   
 peer 7.7.7.7 ebgp-max-hop 10   
 peer 7.7.7.7 connect-interface LoopBack0  
 #  
 ipv4-family unicast  
  undo synchronization  
  peer 4.4.4.4 enable  
  peer 4.4.4.4 label-route-capability  
  undo peer 7.7.7.7 enable  
 #   
 ipv4-family vpnv4  
  policy vpn-target  
  peer 7.7.7.7 enable  
 #  
 ipv4-family vpn-instance A   
  network 192.168.1.0 24  
#  
ospf 2 router-id 2.2.2.2 vpn-instance A  
 import-route bgp  
 area 0.0.0.0 
```
ASBR1：
```
bgp 10  
 peer 2.2.2.2 as-number 10   
 peer 2.2.2.2 connect-interface LoopBack0  
 peer 10.0.45.5 as-number 100   
 #  
 ipv4-family unicast  
  undo synchronization  
  network 2.2.2.2 255.255.255.255   
  peer 2.2.2.2 enable  
  peer 2.2.2.2 route-policy 1 export  
  peer 2.2.2.2 label-route-capability  
  peer 10.0.45.5 enable  
  peer 10.0.45.5 route-policy 2 export  
  peer 10.0.45.5 label-route-capability  
 #   
 ipv4-family vpnv4  
  undo policy vpn-target  
#  
Route-policy 1 permit node 10   
  if-match mpls-label  
  apply mpls-label   
Route-policy 2 permit node 10    
  apply mpls-label 
```
PE2  
```
ip vpn-instance C  
 ipv4-family  
  route-distinguisher 200:1  
  vpn-target 1:1 export-extcommunity  
  vpn-target 1:1 import-extcommunity  
#  
bgp 100  
 peer 2.2.2.2 as-number 10   
 peer 2.2.2.2 ebgp-max-hop 10   
 peer 2.2.2.2 connect-interface LoopBack0  
 peer 5.5.5.5 as-number 100   
 peer 5.5.5.5 connect-interface LoopBack0  
 #  
 ipv4-family unicast  
  undo synchronization  
  peer 2.2.2.2 enable  
  peer 5.5.5.5 enable  
  peer 5.5.5.5 label-route-capability  
 #   
 ipv4-family vpnv4  
  policy vpn-target  
  peer 2.2.2.2 enable  
 #  
 ipv4-family vpn-instance C   
  network 192.168.2.0   
#  
ospf 2 router-id 2.7.7.7 vpn-instance C  
 import-route bgp  
 area 0.0.0.0 
```
ASBR2：
```
bgp 100  
 peer 7.7.7.7 as-number 100   
 peer 7.7.7.7 connect-interface LoopBack0  
 peer 10.0.45.4 as-number 10   
 #  
 ipv4-family unicast  
  undo synchronization  
  network 7.7.7.7 255.255.255.255   
  peer 7.7.7.7 enable  
  peer 7.7.7.7 route-policy 1 export  
  peer 7.7.7.7 label-route-capability  
  peer 10.0.45.4 enable  
  peer 10.0.45.4 route-policy 2 export  
  peer 10.0.45.4 label-route-capability  
 #   
 ipv4-family vpnv4  
  undo policy vpn-target  
#  
Route-policy 1 permit node 10   
  if-match mpls-label  
  apply mpls-label   
Route-policy 2 permit node 10    
  apply mpls-label 
```

### 控制平面：  
PC1访问PC2：
1.PE2将192.168.2.0/24通告在 ipv4-family vpn-instance C 邻居当中，PE2为其分配一个私网标签1026；
2.涉及到跨AS传输，所以会封装一个介于私网标签和公网标签之间的一个标签：1027  
（该标签的作用是，在ASBR之间模拟公网标签进行数据包的传输）  
3.PE2的环回口，将自己的入标签1025 通告给：IBGP邻居ASBR2，到达ASBR2之前的一跳会将公网标签pop掉
4.ASBR2因为存在route-policy，会将介于公网标签和私网标签之间的标签重新分配
5.ASBR1会将PE2 的路由通告IBGP邻居通告给PE1，并且告知PE2，该路由跨AS，需要中间件标签
6.PE1通过RT值，将该路由传递给与该RT值相匹配的接口

### 数据平面：
1.当数据包到达PE1，查看VPN-Istance A 路由表，发现是从EBGP邻居学习来的，并且需要递归到7.7.7.7
/<PE1/>dis ip routing-table vpn-instance A 192.168.2.0
![](assets/14、Option-C1/file-20251211001407149.png)

2.查看VPNv4路由表，接收到1026的私网标签，需要封装1026私网标签
[PE1]display bgp vpnv4 all routing-table 192.168.2.0
![](assets/14、Option-C1/file-20251211001418297.png)

3.查看去往7.7.7.7是否需要通过MPLS 标签封装，TunnelID不为0，需要封装标签  
[PE1] dis fib
![](assets/14、Option-C1/file-20251211001425363.png)

4.查看封装的去往7.7.7.7封装的标签，该标签是在私网标签与公网标签之中
[PE1]display mpls lsp
![](assets/14、Option-C1/file-20251211001429701.png)  

5.问题来了：有去往7.7.7.7的标签，但是下一跳该走哪里？因为7.7.7.7路由时ASBR1在BGP引入到OSPF中，所以：要去查看公共路由表
[PE1]display ip routing-table 7.7.7.7
![](assets/14、Option-C1/file-20251211001437431.png)  

6.需要迭代到4.4.4.4，查看去往4.4.4.4是否需要MPLS封装，TunnelID非0，需要公网标签封装
![](assets/14、Option-C1/file-20251211001444571.png)

7.去往7.7.7.7迭代到4.4.4.4封装1024公网标签￼[PE1]display mpls lsp include 4.4.4.4 32
![](assets/14、Option-C1/file-20251211001448889.png)
![](assets/14、Option-C1/file-20251211001456211.png)  

8.当数据包到达ASBR1后，会将公网标签弹出，查找去往7.7.7.7 的路由，通过EBGP从10.0.45.5学习到7.7.7.7
![](assets/14、Option-C1/file-20251211001504490.png)   
![](assets/14、Option-C1/file-20251211001509293.png)
9.当数据包从ASBR1发出时，因为做了route-policy，重新分发了标签为1025
==为什么需要重新分布标签呢？  ==
因为：在AS10内使用的中间层标签可能，在AS100内可能以及被其他标签占用，如果不重新分发标签，回导致找不到路
![](assets/14、Option-C1/file-20251211001517392.png)
\<ASBR1\>dis mpls lsp
![](assets/14、Option-C1/file-20251211001522087.png)

10.数据包到达ASBR2上后，查看路由表：通过OSPF学习到7.7.7.7，需要通过mpls封装标签
![](assets/14、Option-C1/file-20251211001618852.png)
![](assets/14、Option-C1/file-20251211001622301.png)
![](assets/14、Option-C1/file-20251211001556895.png)   
  

11.数据包到达PE2时，pop公网标签，只剩下标签，然后PE2查表，1026标签时为VPN-Instance C标识，数据包从VPN-Instance绑定的接口发出
![](assets/14、Option-C1/file-20251211001629800.png) 
![](assets/14、Option-C1/file-20251211001635902.png)
最后数据到达CE2，CE2查表，到网关，再到PC2


## 输出详解
### LSP解释：
1. 2.2.2.2/32为本端通告network通告的路由，所有分配1025标签
2. 7.7.7.7/32为对端ASBR2通告给本端的路由，分配给本端的标签为1024，本端ASBR1再重新为改路由分配1027标签。
```
<ASBR1>dis mpls lsp
-------------------------------------------------------------------------------
                 LSP Information: BGP  LSP
-------------------------------------------------------------------------------
FEC                In/Out Label  In/Out IF                      Vrf Name       
7.7.7.7/32         NULL/1024     -/-                                           
2.2.2.2/32         1025/NULL     -/-                                           
7.7.7.7/32         1027/1024     -/-                                           
-------------------------------------------------------------------------------
                 LSP Information: LDP LSP
-------------------------------------------------------------------------------
FEC                In/Out Label  In/Out IF                      Vrf Name       
4.4.4.4/32         3/NULL        -/-                                           
3.3.3.3/32         NULL/3        -/GE0/0/0                                     
3.3.3.3/32         1024/3        -/GE0/0/0                                     
2.2.2.2/32         NULL/1025     -/GE0/0/0                                     
2.2.2.2/32         1026/1025     -/GE0/0/0 
```


### **三层标签如何而来**
1. 例如CE1访问CE2，CE1查看路由表将报文发送给PE1的VRF-A
2. PE1接收到报文，根据目的ip地址查看路由表，发现需要relay到7.7.7.7，并且TunnelID不为0，需要标签
```
<PE1>display ip routing-table vpn-instance A 192.168.2.0 verbose 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Table : A
Summary Count : 1

Destination: 192.168.2.0/24
     Protocol: EBGP             Process ID: 0
   Preference: 255                    Cost: 3
      NextHop: 7.7.7.7           Neighbour: 7.7.7.7
        State: Active Adv Relied       Age: 00h08m25s
          Tag: 0                  Priority: low
        Label: 1026                QoSInfo: 0x0
   IndirectID: 0x5              
 RelayNextHop: 4.4.4.4           Interface: GigabitEthernet0/0/1
     TunnelID: 0x5                   Flags: RD
```

3. 于是查看去往7.7.7.7的路由表，发现还要relay到4.4.4.4，并且TunnelID不为0，需要标签
```
<PE1>display ip routing-table 7.7.7.7 verbose 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Table : Public
Summary Count : 1

Destination: 7.7.7.7/32
     Protocol: IBGP             Process ID: 0
   Preference: 255                    Cost: 2
      NextHop: 4.4.4.4           Neighbour: 4.4.4.4
        State: Active Adv Relied       Age: 00h09m43s
          Tag: 0                  Priority: low
        Label: 1027                QoSInfo: 0x0
   IndirectID: 0x2              
 RelayNextHop: 10.0.23.3         Interface: GigabitEthernet0/0/1
     TunnelID: 0x3                   Flags: RD
```
4. 查看去往4.4.4.4的路由表，并且TunnelID不为0，需要标签，单不需要relay，说明下一跳能够之间到达
```
<PE1>display ip routing-table 4.4.4.4 verbose 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Table : Public
Summary Count : 1

Destination: 4.4.4.4/32
     Protocol: OSPF             Process ID: 1
   Preference: 10                     Cost: 2
      NextHop: 10.0.23.3         Neighbour: 0.0.0.0
        State: Active Adv              Age: 00h10m05s
          Tag: 0                  Priority: medium
        Label: NULL                QoSInfo: 0x0
   IndirectID: 0x0              
 RelayNextHop: 0.0.0.0           Interface: GigabitEthernet0/0/1
     TunnelID: 0x3                   Flags:  D
```
5. 查看私网标签分配，去往192.168.2.0 分配了 **1026**
```
<PE1>display bgp vpnv4 all routing-table 192.168.2.0 


 BGP local router ID : 10.0.23.2
 Local AS number : 10

 Total routes of Route Distinguisher(200:1): 1
 BGP routing table entry information of 192.168.2.0/24:
 Label information (Received/Applied): 1026/NULL
 From: 7.7.7.7 (10.0.67.7)
 Route Duration: 00h16m16s  
 Relay IP Nexthop: 10.0.23.3
 Relay IP Out-Interface: GigabitEthernet0/0/1
 Relay Tunnel Out-Interface: GigabitEthernet0/0/1
 Relay token: 0x5
 Original nexthop: 7.7.7.7
 Qos information : 0x0
 Ext-Community:RT <1 : 1>, OSPF DOMAIN ID <0.0.0.0 : 0>, 
               OSPF RT <0.0.0.0 : 1 : 0>, OSPF ROUTER ID <2.7.7.7 : 0>
 AS-path 100, origin igp, MED 3, pref-val 0, valid, external, best, select, pre 255, IGP cost 2
 Not advertised to any peer yet


 VPN-Instance A, Router ID 10.0.23.2:

 Total Number of Routes: 1
 BGP routing table entry information of 192.168.2.0/24:
 Label information (Received/Applied): 1026/NULL
 From: 7.7.7.7 (10.0.67.7)
 Route Duration: 00h16m16s  
 Relay Tunnel Out-Interface: GigabitEthernet0/0/1
 Relay token: 0x5
 Original nexthop: 7.7.7.7
 Qos information : 0x0
 Ext-Community:RT <1 : 1>, OSPF DOMAIN ID <0.0.0.0 : 0>, 
               OSPF RT <0.0.0.0 : 1 : 0>, OSPF ROUTER ID <2.7.7.7 : 0>
 AS-path 100, origin igp, MED 3, pref-val 0, valid, external, best, select, active, pre 255, IGP cost 2
 Not advertised to any peer yet
```
5. 查看MPLS LSP路径，公网，以及跨域LSP；公网分配**1024**，跨域分配**1027**
```
<PE1>display mpls lsp 
-------------------------------------------------------------------------------
                 LSP Information: BGP  LSP
-------------------------------------------------------------------------------
FEC                In/Out Label  In/Out IF                      Vrf Name       
7.7.7.7/32         NULL/1027     -/-                                           
192.168.1.0/24     1026/NULL     -/-                            A              
-------------------------------------------------------------------------------
                 LSP Information: LDP LSP
-------------------------------------------------------------------------------
FEC                In/Out Label  In/Out IF                      Vrf Name       
3.3.3.3/32         NULL/3        -/GE0/0/1                                     
3.3.3.3/32         1024/3        -/GE0/0/1                                     
4.4.4.4/32         NULL/1024     -/GE0/0/1                                     
4.4.4.4/32         1025/1024     -/GE0/0/1                                     
2.2.2.2/32         3/NULL        -/-
```
6. 安装迭代路径，标签路径如下：
```
1024 # 公网标签
1027 # 跨域标签
1026 # 私网标签
```
### tracert 路径标签显示
```
<CE1>tracert -v -a 192.168.1.254 192.168.2.254
 traceroute to  192.168.2.254(192.168.2.254), max hops: 30 ,packet length: 40,press CTRL_C to break 
 1 10.0.12.2 20 ms  10 ms  10 ms 
 2 10.0.23.3[MPLS Label=1024/1027/1026 Exp=0/0/0 S=0/0/1 TTL=1/1/1] 60 ms  50 ms  50 ms 
 3 10.0.34.4[MPLS Label=1027/1026 Exp=0/0 S=0/1 TTL=1/2] 60 ms  50 ms  60 ms 
 4 10.0.45.5[MPLS Label=1024/1026 Exp=0/0 S=0/1 TTL=1/3] 50 ms  50 ms  50 ms 
 5 10.0.56.6[MPLS Label=1024/1026 Exp=0/0 S=0/1 TTL=1/4] 50 ms  40 ms  40 ms 
 6 10.0.78.7 40 ms  50 ms  50 ms 
 7 10.0.78.8 50 ms  50 ms  40 ms 
```

## 一、VPNv4 路由与跨域 BGP-LU 标签路由

这两种路由是**跨域 MPLS VPN 的两大核心控制平面载体**，但定位完全不同：**VPNv4 是 "业务路由"，负责传递用户私网 VPN 路由；BGP-LU 是 "隧道路由"，负责为业务路由提供跨 AS 的公网标签转发通道**。
### 1.必选核心元素（VPNv4 独有）

| 元素                                | 长度               | 作用与特点                                                                                                                        |
| :-------------------------------- | :--------------- | :--------------------------------------------------------------------------------------------------------------------------- |
| **RD（Route Distinguisher，路由区分符）** | 8 字节（64 位）       | 核心作用：将重复的私网 IPv4 前缀转换为全局唯一的 VPNv4 前缀<br>本地意义：仅在发布该路由的 PE 上有效，对端 PE 仅用它区分不同路由，不用于转发决策<br>每个 VRF 实例必须配置唯一的 RD                  |
| **VPN-Target（Route Target，路由目标）** | 8 字节（BGP 扩展团体属性） | 核心作用：控制 VPNv4 路由的导入与导出，实现 VPN 之间的隔离与互通。<br>分为 Export RT（导出时打标签）和 Import RT（导入时匹配标签）<br>全局意义：必须全网统一规划，匹配成功的路由才能被导入目标 PE 的 VRF |
| **私网 MPLS 标签**                    | 32 位（仅 20 位有效）   | 由发布该路由的 PE 分配，用于标识目的 VPN 或具体私网路由。<br>分配方式：基于路由（每条路由一个标签）或基于 VRF（整个 VPN 共享一个标签）<br>全程不变：在整个骨干网转发过程中，私网标签仅由最后一跳 PE 弹出，中间设备不修改  |
| **下一跳地址**                         | 4 字节（IPv4）       | 永远是**发布该 VPNv4 路由的 PE 的 Loopback 地址**，无论经过多少个 BGP 邻居转发。<br>关键特性：下一跳不随 ASBR 改变（Option C 场景），仅在 Option B 场景下会被 ASBR 修改为自身地址    |
### 2. 可选标准 BGP 元素

VPNv4 路由同时携带所有标准 BGP 路由属性，用于路由选择和策略控制：

- AS_PATH、Origin、Local_Preference、MED
- 团体属性（Community）、扩展团体属性（除 VPN-Target 外）
- 路由衰减、权重等 BGP 私有属性
### 核心特点

1. **地址唯一性**：通过 RD 前缀，彻底解决不同 VPN 使用相同私网 IP 段的问题
2. **天然隔离性**：通过 VPN-Target 属性，实现 VPN 之间的逻辑隔离，默认不通
3. **PE 端到端**：仅 PE 设备理解和处理 VPNv4 路由，P 设备和 ASBR（Option C 场景）完全不感知
4. **标签固定性**：私网标签由源 PE 分配，全程不修改，仅由目的 PE 弹出
5. **地址族专属**：必须在 BGP 的`vpnv4`地址族下交换，不能在普通 IPv4 地址族下传递

## 二、跨域 BGP-LU 标签路由（RFC 3107）
BGP-LU（Labeled Unicast，带标签单播）是 BGP 的另一个扩展，核心作用是**为普通 IPv4 单播路由分配 MPLS 标签**，用于建立跨 AS 的公网标签隧道。它是 Option C1/C2 跨域方案的基础，解决了 LDP 无法跨 AS 分配标签的问题。

#### 1.必选核心元素

| 元素             | 长度             | 作用与特点                                                                                                                                                                                   |
| :------------- | :------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **IPv4 前缀**    | 4 字节           | 普通的公网 IPv4 前缀，在跨域 MPLS VPN 中，几乎都是**PE 的 Loopback /32 主机路由**（用于 PE 之间建立 MP-EBGP 对等体）                                                                                                     |
| **公网 MPLS 标签** | 32 位（仅 20 位有效） | 由发布该路由的 BGP-LU 邻居分配，用于标识去往该前缀的公网 LSP。<br>- 本地意义：每个 BGP-LU 邻居为同一个前缀分配的标签可以不同<br>- 逐跳替换：每经过一个 BGP-LU 节点，标签都会被替换为下一跳分配的标签<br>- 分配触发：必须通过`apply mpls-label`路由策略强制分配，BGP 默认不会自动为 IBGP 路由分配标签 |
| **下一跳地址**      | 4 字节（IPv4）     | 是**发布该 BGP-LU 路由的 BGP 邻居的直连地址**，除非通过路由策略修改。<br>- IBGP 场景：下一跳为发布者的 Loopback 地址（需配置`update-source LoopBack0`）<br>- EBGP 场景：默认下一跳为直连互联地址，必须通过路由策略修正为 ASBR 自身的 Loopback 地址，否则 PE 无法迭代路由     |

#### 2. 可选标准 BGP 元素

BGP-LU 路由同样携带所有标准 BGP 路由属性，用于公网路由选择和策略控制：

- AS_PATH、Origin、Local_Preference、MED
- 团体属性、扩展团体属性
- 路由衰减、权重等

### 核心特点

1. **公网属性**：属于公网路由，用于建立骨干网标签隧道，不携带任何私网信息
2. **标签动态性**：标签由 BGP 动态分配，每经过一个 BGP-LU 节点都会被替换
3. **ASBR 核心处理**：ASBR 是 BGP-LU 路由的核心处理节点，负责标签分配、下一跳修正和路由过滤
4. **地址族兼容**：在普通 IPv4 单播地址族下交换，只需为邻居使能`label-route-capability`即可
5. **隧道拼接**：与域内 LDP LSP 拼接，形成 PE 之间端到端的公网标签隧道
6. 
## 三、两种路由的核心区别对比表

|对比维度|VPNv4 路由|BGP-LU 标签路由|
|:--|:--|:--|
|**核心定位**|私网业务路由，传递用户 VPN 路由|公网隧道路由，建立跨 AS 标签转发通道|
|**标准定义**|RFC 4364|RFC 3107|
|**前缀格式**|`RD:IPv4前缀/掩码`（12 字节）|普通`IPv4前缀/掩码`（4 字节）|
|**必选独有属性**|RD、VPN-Target|无（标签是附加在普通 IPv4 路由上的）|
|**标签类型**|私网标签，全程不变|公网标签，逐跳替换|
|**下一跳特点**|永远是源 PE 的 Loopback 地址（Option C）|是发布该路由的 BGP 邻居的地址，可修改|
|**交换地址族**|BGP VPNv4 地址族|BGP IPv4 单播地址族（使能标签能力）|
|**处理设备**|仅 PE 设备处理|ASBR 和 PE 设备处理，P 设备不感知|
|**跨域方案中的作用**|Option A/B/C 中均用于传递私网路由|仅 Option C1/C2 中用于建立公网隧道|
|**扩展性瓶颈**|Option B 中 ASBR 需维护全量 VPNv4 路由|仅维护 PE Loopback 路由，扩展性极高|
