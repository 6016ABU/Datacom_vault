# 配置详解（无RR场景）
1. 底层使用IGP互联互通（OSPF，ISIS）
2. 配置LDP协议，PE，P，ASBR之间都需要配置
3. PE与CE之间通过IGP或者EBGP传递路由
4. 同一个AS内部PE、P、ASBR之间建立RR-IBGP邻居关系
5. ASBR1与ASBR2之间建立EBGP邻居关系，关闭RT值校验
	1. network通告本端PE的Loopback地址
	2. 对来自本端PE的路由替换标签，使能标签路由能力
	3. 对来自对端ASBR的路由分配标签，使能标签路由能力
6. ASBR1与ASBR2互联接口使能mpls
7. PE1与PE2建立MP-IBGP邻居关系（VPNv4）

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


# 输出详解
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