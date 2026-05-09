# 配置详解
1. 底层使用IGP互联互通（OSPF，ISIS）
2. 配置LDP协议，PE，P，ASBR之间都需要配置
3. PE与CE之间通过IGP或者EBGP传递路由
4. PE与ASBR之间建立MP-BGP邻居关系（VPNv4），无RR场景
5. ASBR1与ASBR2之间建立MP-EBGP邻居关系（VPNv4），取消RT值校验
6. ASBR1与ASBR2互联接口使能mpls

![](assets/13、Option-B/file-20251211000824757.png)

## 具体关键配置：
**ASBR1:  **
```
interface GigabitEthernet0/0/1  
 ip address 10.0.45.4 255.255.255.0   
 mpls  
#  
bgp 10  
 peer 2.2.2.2 as-number 10   
 peer 2.2.2.2 connect-interface LoopBack0  
 peer 10.0.45.5 as-number 100   
 #  
 ipv4-family unicast  
  undo synchronization  
  undo peer 2.2.2.2 enable  
  undo peer 10.0.45.5 enable  
 #   
 ipv4-family vpnv4  
  undo policy vpn-target  
  peer 2.2.2.2 enable  
  peer 10.0.45.5 enable

```
**AS 100内 ：
ASBR2:
```
#  
interface GigabitEthernet0/0/1  
 ip address 10.0.45.5 255.255.255.0   
 mpls  
#  
bgp 100  
 peer 7.7.7.7 as-number 100   
 peer 7.7.7.7 connect-interface LoopBack0  
 peer 10.0.45.4 as-number 10   
 #  
 ipv4-family unicast  
  undo synchronization  
  undo peer 7.7.7.7 enable  
  peer 10.0.45.4 enable  
 #   
 ipv4-family vpnv4  
  undo policy vpn-target  
  peer 7.7.7.7 enable  
  peer 10.0.45.4 enable  
#
```

## **与Option-A的区别：
1.ASBR之间建立VPNv4邻居  
2.ASBR之间的接口不再绑定实例，  
通过在接口上使能mpls，  
以及在BGP下的ipv4-family vpnv4 undo policy vpn-target关闭掉 RT值的检测
 
==优点：减轻了ASBR的压力，不需要配置VPN-Instance来标记路由==
 
控制平面：
区别：在ASBR2将192.168.2.0/24的路由通告给ASBR2时  
继续沿用PE2给192.168.2.0/24的私网标签  
即：单向路径上：私网标签一直存在，且不会改变
 
转发（数据）平面：
区别：只是在ASBR之间是通过MPLS标签交换路由


## Option B上的LSP
核心标识！说明这是 Option B 场景下 ASBR 专属的中转 LSP，**不绑定任何用户业务 VRF**，仅用于跨 AS 的 VPN 流量标签转发，==区别于普通 PE 上绑定用户 VRF 的 L3VPN LSP==
```
<ASBR1>dis mpls lsp
-------------------------------------------------------------------------------
                 LSP Information: L3VPN  LSP
-------------------------------------------------------------------------------
FEC                In/Out Label  In/Out IF                      Vrf Name       
192.168.2.0/24     1026/1026     -/-                            ASBR LSP       
192.168.1.0/24     1027/1026     -/-                            ASBR LSP       
-------------------------------------------------------------------------------
                 LSP Information: LDP LSP
-------------------------------------------------------------------------------
FEC                In/Out Label  In/Out IF                      Vrf Name       
3.3.3.3/32         NULL/3        -/GE0/0/0                                     
3.3.3.3/32         1024/3        -/GE0/0/0                                     
4.4.4.4/32         3/NULL        -/-                                           
2.2.2.2/32         NULL/1025     -/GE0/0/0                                     
2.2.2.2/32         1025/1025     -/GE0/0/0                                     
```


## 在ASBR上的路由通告
```
<ASBR1>dis bgp vpnv4 all routing-table 192.168.1.0


 BGP local router ID : 10.0.34.4
 Local AS number : 10

 Total routes of Route Distinguisher(100:1): 1
 BGP routing table entry information of 192.168.1.0/24:
 Label information (Received/Applied): 1026/1027
 From: 2.2.2.2 (10.0.23.2)
 Route Duration: 00h07m50s  
 Relay IP Nexthop: 10.0.34.3
 Relay IP Out-Interface: GigabitEthernet0/0/0
 Relay Tunnel Out-Interface: GigabitEthernet0/0/0
 Relay token: 0x4
 Original nexthop: 2.2.2.2
 Qos information : 0x0
 Ext-Community:RT <1 : 1>, OSPF DOMAIN ID <0.0.0.0 : 0>, 
               OSPF RT <0.0.0.0 : 1 : 0>, OSPF ROUTER ID <2.2.2.2 : 0>
 AS-path Nil, origin igp, MED 3, localpref 100, pref-val 0, valid, internal, best, select, pre 255, IGP cost 2
 Advertised to such 1 peers:
    10.0.45.5
```

```
<ASBR1>dis bgp vpnv4 all routing-table 192.168.2.0


 BGP local router ID : 10.0.34.4
 Local AS number : 10

 Total routes of Route Distinguisher(200:1): 1
 BGP routing table entry information of 192.168.2.0/24:
 Label information (Received/Applied): 1026/1026
 From: 10.0.45.5 (10.0.56.5)
 Route Duration: 00h08m26s  
 Relay Tunnel Out-Interface: GigabitEthernet0/0/1
 Relay token: 0x1
 Original nexthop: 10.0.45.5
 Qos information : 0x0
 Ext-Community:RT <1 : 1>, OSPF DOMAIN ID <0.0.0.0 : 0>, 
               OSPF RT <0.0.0.0 : 1 : 0>, OSPF ROUTER ID <2.7.7.7 : 0>
 AS-path 100, origin igp, pref-val 0, valid, external, best, select, pre 255
 Advertised to such 1 peers:
    2.2.2.2
```