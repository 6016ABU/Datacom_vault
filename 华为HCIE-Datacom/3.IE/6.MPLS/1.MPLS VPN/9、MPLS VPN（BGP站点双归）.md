# BGP站点双归 
**本站点路由次优的问题：**  
AR1存在环回口路由100.1.1.1/32，在OSPF中通告  
**AR2将OSPF路由引入到BGP中**，产生MED=1的BGP路由100.1.1.1/32，通告给EBGP邻居AR4 和 IBGP邻居AR1  
AR1将学习到的路由传递给EBGP邻居时 会清空MED值（变为0）
![500](assets/9、MPLS%20VPN（BGP站点双归）/file-20260519155411026.png)
==med属性值只能传递到邻居的AS==，不能跨越AS传递，所以说，AR1因为是AR2的IBGP邻居，收到带有med=1的值，会清空med值，
导致AR1通告给AR3的路由，med=0  
AR3收到100.1.1.1/32路由的MED=0  
AR4收到100.1.1.1/32路由的MED=1  
AR3和AR4通过MP-IBGP邻居交互100.1.1.1/32的路由信息  
AR4就会优选AR3传递的路由条目（med越小越优先）  
AR4就通过次优访问100.1.1.1/32  
（路由通过更少的组网环境（路由协议）更优先）
### **如何解决该问题：** 
==在AR2引入OSPF路由时，通过配置将MED值设置为0==
值得注意的是：AR与AR3之间：  
AR3是在VPN实例下与AR1建立EBGP邻居，所以AR3需要在ipv4-family vpn-instance 下建立邻居
![700](assets/9、MPLS%20VPN（BGP站点双归）/file-20260519155411034.png)
```D
bgp 100
 peer 1.1.1.1 as-number 10
 peer 4.4.4.4 as-number 100
 peer 4.4.4.4 connect-interface LoopBack0
 #
 ipv4-family unicast
  undo synchronization
  undo peer 1.1.1.1 enable
  undo peer 4.4.4.4 enable
 #
 ipv4-family vpnv4
  policy vpn-target
  peer 4.4.4.4 enable
 #
 ipv4-family vpn-instance A
  peer 10.0.13.1 as-number 10
```
### 特殊场景下BGP的防环： 
![700](assets/9、MPLS%20VPN（BGP站点双归）/file-20260519155411034.png)
1.当站点AR1存在一条路由信息，引入进BGP，通过EBGP邻居传递给AR3、通过IBGP邻居传递给AR2。
2.AR3收到路由后通过MP-IBGP邻居传递给AR4，AR2收到路由后通过EBGP邻居传递给AR4。
3.AR4收到一条路由信息，但是存在两个下一跳，通过EBGP优于IBGP的选路规则，优选了AR2的路由信息。
4.将AR2传递给AR4的路由med值该大，变为100。
![](assets/9、MPLS%20VPN（BGP站点双归）/file-20260519155411005.png)
![485](assets/9、MPLS%20VPN（BGP站点双归）/file-20260519155411016.png)
5.AR4会优选AR3的路由信息，且会将优选的路由继续传递给AR2设备  
6.AR2不会接收AR4传递的路由信息，因为AS-path防环
**7.AR4对AR2执行```
```
[AR4-bgp-A]peer 10.1.24.2 substitute-as 
该命令将路由信息的as号全部变为自身所在的AS编号**  
```

8.AR2可以正常接收AR4传递的路由信息
![700](assets/9、MPLS%20VPN（BGP站点双归）/file-20260519155410997.png)
 ![700](assets/9、MPLS%20VPN（BGP站点双归）/file-20260519155410986.png)
综上所述，站点的内部的路由通过MPLS VPN网络有传回站点内，造成了路由的环路
### 通过BGP的Soo属性来防止该问题的产生  
[AR3-bgp-A]peer 10.1.13.1 soo 123:123 表示对于站点打上扩展团体属性的值 123:123
![700](assets/9、MPLS%20VPN（BGP站点双归）/file-20260519155410978.png)
[AR4-bgp-A]peer 10.1.24.2 soo 123:123 表示对于站点打上扩展团体属性的值 123:123  
如果收到的路由携带soo属性，且属性值和自身指定站点的soo属性值相同，则认为该路由信息就是站点内的  
不会在将路由信息传递给站点内
![700](assets/9、MPLS%20VPN（BGP站点双归）/file-20260519155410967.png)
![](assets/9、MPLS%20VPN（BGP站点双归）/file-20260519155410960.png)


### **BGP 的**Soo 属性（Site of Origin，起源站点属性）是 BGP 协议的一种扩展属性，  
主要用于 MPLS VPN（多协议标签交换虚拟专用网）场景，核心作用是**防止 VPN 路由在不同站点间传递时形成环路**。  
**一、基本定义**  
Soo 是 BGP 的**可选非过渡（Optional Non-Transitive）属性**，属性类型码为 130。它通过标记路由的 “起源站点” 信息，使路由在返回原站点时被识别并丢弃，从而避免环路。  
**二、核心作用**  
在 MPLS VPN 中，企业可能存在多个站点（如分支机构），且站点间可能通过两种方式交换路由。
- 站点间直接连接（如 CE-CE 直连，运行 IGP 或静态路由）；
- 通过服务提供商的 PE 设备（Provider Edge），利用 MP-BGP（多协议 BGP）传递 VPNv4 路由。
这种 “双路径” 可能导致路由环路（例如，一条路由从 Site A 出发，经 CE-CE 直连到 Site B，再经 PE 设备传回 Site A）。  
Soo 的作用是：**为从某站点起源的路由打上 “站点标记”**，当路由返回原站点时，接收设备（CE 或 PE）识别到标记与本地站点一致，会直接丢弃该路由，避免环路。  
**三、格式与特征**  
Soo 的格式与 RD（Route Distinguisher，路由区分符）类似，为 32 位数值，通常表示为两种形式：
- AS号:编号（如100:1，AS 号为 2 字节，编号为 2 字节）；
- IP地址:编号（如192.168.1.1:1，IP 地址为 4 字节，编号为 2 字节，实际取 IP 地址的低 2 字节）。
但需注意：
- Soo 与 RD 作用完全不同：RD 用于区分不同 VPN 的相同前缀（确保 VPNv4 路由唯一性）；Soo 用于标记路由的==起源站点（防环）。==
- Soo 是本地意义的属性，仅在同一 VPN 的站点间生效，不会被传递到其他 VPN。
