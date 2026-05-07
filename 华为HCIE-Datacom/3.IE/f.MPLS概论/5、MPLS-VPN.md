# MPLS-VPN
**VPN的基本概念：**  
**VPN：虚拟专用网络**
在一个公共网络中实现虚拟的专用网络
VPN本质就是隧道，隧道的本质就是数据的再封装

**设备类型：**
1.CE设备（用户的边缘设备）  
2.PE设备（运营商的边缘设备）  
3.P设备（运营商的骨干设备）

站点site：用户的网络统称  
站点的边缘设备是CE设备  
CE设备连接的一定是PE设备

站点内使用私网地址  
站点内如何实现相互通信？ 可以通过配置静态路由或动态IGP协议实现相互通信  
站点内如何实现外部访问？ 可以通过NAT实现访问外部网络  
站点与站点之间如何实现访问？ 物理专线 、 虚拟网络（VPN）

## 控制平面路由交互
站点之间相互访问的前提是获取对方的路由信息
站点之间如何获取对方的路由信息？
为什么站点的路由无法通过公网之间传递到对方站点？  
因为站点的路由都是私网路由，而私网地址是可以重复的，所以公网设备如何学习私网路由 则会发送冲突

**公网设备（PE）区分私网路由和公网路由：**
通过VRF技术，配置VPN实例 来隔离私网路由和公网路由
**什么是VRF？什么是VPN实例？**
VRF：虚拟路由转发，在物理设备上通过虚拟化构建多台逻辑设备  
VPN实例：是VRF实现的技术手段  
每一台设备都会存在一张公有路由表（public）  
```D
[Huawei]ip vpn-instance test 创建一台逻辑设备，逻辑设备的路由表项名称为test  
```
需要在接口和协议下绑定VPN实例，通过VPN实例的隔离实现路由表的隔离
 
公网设备之间进行路由信息的传递（不考虑如何传递），P设备可以接收该路由，如何区分路由是不同站点的？  
PE设备根据VPN实例区分不同的站点路由，而VPN实例是本地生效的，无法影响其他设备  
P设备如果收到相同网段的路由，则无法区分  
P设备如果收到不同网段的路由，则可以正常进行区分
 
**PE设备如何发送不同网段的路由信息？**  
PE设备发送私网路由时，根据规则发送的是VPNv4路由
 
**什么是VPNv4路由？**
VPNv4路由 是由 8Byte RD + 4Byte ipv4私网路由 组成
 
**什么是RD(Route Distinguisher)？**
RD：路由区分符 书写方式 AA:NN（32bit:32bit）  
通过在VPN实例下配置，将不同站点的私网路由 通过指定的RD值进行绑定，绑定后就形成了VPNv4路由  
```D
[Huawei-vpn-instance-test]route-distinguisher 100:1  
```
作用：用来区分不同站点相同的路由信息
 
**如果对端PE接收不同站点的路由后，可以通过RD值进行区分；但是如何将不同站点的路由接收进本地站点？**  
可以通过RT值来判断，是否接收VPNv4路由进入本地站点
 
**什么是RT？（十分重要，一般问题出在这里）**
RT：路由目标 （Route Target） 书写方式 AA:NN（32bit:32bit）  
VPN实例下配置RT值：
```D
[Huawei-vpn-instance-test-af-ipv4]vpn-target 1:1 both  
```
**RT值区分：**
1.IRT：入方向RT（==本地意义==，不会随着VPNv4路由发出；作用：用于匹配VPNv4路由携带的ERT值，如果匹配，则接收该路由）  
2.ERT：出方向RT（==全局意义==，随着VPNv4路由发出；作用：当作VPNv4的一个属性，等待IRT值的匹配）  
IRT和ERT在VPN实例下可以配置多个，一个VPNv4路由携带的多个ERT值 之间 是或的关系   无法使用RD值来替换RT，因为RD值只能配置一个 局限性太大  
无法使用RT值来替换RD，因为RT值可以存在多个 无法进行有效区分
 
## **VPNv4路由是如何传递的？**
通过MP-BGP建立VPNv4的BGP邻居关系，传递VPNv4路由  
**PE之间建立BGP邻居关系，并通过BGP进行路由传递。为什么采用BGP呢？**  
BGP使用TCP作为其传输层协议，提高了协议的可靠性。可以跨路由器的两个PE设备之间直接交换路由。  
BGP拓展性强，为PE间传播VPN路由提供了便利。  
PE之间需要传送的路由条目可能较大，BGP只发送更新的路由，提高传递路由数量的同时不占用过多链路带宽。
 
为了正确处理VPN路由，MPLS VPN使用RFC2858（Multiprotocol Extensions for BGP-4）中规定的MP-BGP，即BGP-4的多协议扩展。  
MP-BGP采用地址族来区分不同的网络层协议，既可以支持传统的IPv4地址族，又可以支持其它地址族（比如VPN-IPv4地址族、IPv6地址等）。  
**MP-BGP新增了两种路径属性：**
NLRI：Network Layer Reachability Information 网络层可达信息  
MP_REACH_NLRI：Multiprotocol Reachable NLRI，多协议可达NLRI。用于发布可达路由及下一跳信息。  
MP_UNREACH_NLRI：Multiprotocol Unreachable NLRI，多协议不可达NLRI。用于撤销不可达路由。
![](assets/5、MPLS-VPN/file-20251210154223560.png)

```
dis bgp vpnv4 all peer # 查看VPNv4邻居信息  
dis bgp vpnv4 all routing-table # 查看VPNv4路由信息
```
 ![1000](assets/5、MPLS-VPN/file-20251210154619411.png)  
![](assets/5、MPLS-VPN/file-20251210154653364.png)
### MPLS VPN配置思路：  
1.公网IGP与LDP邻居建立  
2.站点VPN参数规划  
绑定VPN实例信息  
3.构建MP-BGP邻居用于传递VPNv4路由
```
AR3：  
#  
bgp 100  
 peer 5.5.5.5 as-number 100   
 peer 5.5.5.5 connect-interface LoopBack0  
 #  
 ipv4-family unicast  
  undo synchronization  
  undo peer 5.5.5.5 enable  
 #   
 ipv4-family vpnv4  
  policy vpn-target  
  peer 5.5.5.5 enable  
 #  
 ipv4-family vpn-instance A   
  network 192.168.12.0 
```

```
AR5  
#  
bgp 100  
 peer 3.3.3.3 as-number 100   
 peer 3.3.3.3 connect-interface LoopBack0  
 #  
 ipv4-family unicast  
  undo synchronization  
  undo peer 3.3.3.3 enable  
 #   
 ipv4-family vpnv4  
  policy vpn-target  
  peer 3.3.3.3 enable  
 #  
 ipv4-family vpn-instance C   
  import-route ospf 2
```
![600](assets/5、MPLS-VPN/file-20251210154817767.png)
![600](assets/5、MPLS-VPN/file-20251210154835701.png)
![600](assets/5、MPLS-VPN/file-20251210154845785.png)
### **两张表项存在相互转换的能力：**  
1.如果路由是本端产生的，则下一跳为全0，放入到VPN实例表项中；自动的转换为VPNv4路由  
2.如果路由是本端学习的，则下一跳为非0，放入到VPNv4表项中，根据IRT匹配ERT的值将路由转换到VPN实例路由表中

## 转发平面数据转发流程
AR1（CE设备）将数据发送AR3（PE设备）  
AR3（PE设备）进行数据的处理（再封装）  
**1.先根据目标判断是否存在路由信息**
	1.如果不存在路由信息，则直接丢弃报文  
	2.如果存在路由信息，则判断FIB表项中tunnel id是否为0  
		1.如果为0，则按照IP执行转发  
		2.如果为非0，则按照标签封装转发  
**2.执行标签封装的顺序**
	1.根据MP-BGP协议为传递的私网路由分发的私网标签，执行封装  
		无法通过封装的私网标签直接转发数据  
		BGP需要进行路由的迭代（相当于要执行隧道的迭代）  
		BGP路由根据下一跳进行路由的迭代  
		在公网路由表中找到BGP路由对应下一跳的路由信息  
		继续判断下一跳的路由信息是否要执行隧道转发  
	2.根据LDP协议为公网路由分发的公网标签，执行封装

**动态标签分发：**
	1.LDP协议  
	2.MP-BGP协议  
	3.RSVP-TE协议
 
**在MPLS VPN网络中：**  
LDP协议一般用于公网IGP进行公网标签的分配  
MP-BGP协议一般用于私网站点进行私网标签的分配
 
**标签和路由的关系：**  
1.一定是先存在路由，才能存在标签  
2.如果存在标签，意味着一定存在路由  
3.如果不存在路由，则无法处理对应的标签值
 
==公网设备（P设备）无法在IP路由表存在私网路由信息，所以也无法处理私网标签信息==

```mermaid
flowchart TD
    %% 节点样式定义
    classDef startend fill:#E8F4F8,stroke:#2B7BBA,stroke-width:2px,rounded:10px;
    classDef action fill:#E8FBF2,stroke:#00B894,stroke-width:2px,rounded:8px;
    classDef table fill:#F5F0FF,stroke:#6C5CE7,stroke-width:2px,rounded:8px;
    classDef judge fill:#FFF9E6,stroke:#F39C12,stroke-width:2px,rounded:8px;
    classDef state fill:#FFE8F5,stroke:#D63384,stroke-width:2px,rounded:8px;

    %% 流程起始
    A0([MPLS VPN 控制平面流程开始]):::startend

    %% ======================
    %% 阶段1：骨干网公网LSP隧道预建立（IGP+LDP）
    %% ======================
    subgraph 阶段1：骨干网公网LSP隧道预建立
        direction TB
        %% 泳道：PE1、P、PE2 公网设备
        subgraph 公网设备PE1
            A1[公网接口使能MPLS、MPLS LDP\n发布Loopback0与公网网段路由到IGP]:::action
            A2[IGP收敛完成，生成全局RIB公网路由表]:::table
            A3[从全局RIB提取生成全局FIB公网转发表]:::table
            A4[与直连邻居建立LDP会话，会话进入Operational状态]:::state
            A5[为Loopback0 FEC分配本地入标签，接收邻居标签映射]:::action
            A6[生成LIB标签信息库]:::table
            A7[从LIB提取有效条目，生成LFIB标签转发表]:::table
        end

        subgraph 公网设备P
            B1[公网接口使能MPLS、MPLS LDP\n发布Loopback0与公网网段路由到IGP]:::action
            B2[IGP收敛完成，生成全局RIB公网路由表]:::table
            B3[从全局RIB提取生成全局FIB公网转发表]:::table
            B4[与直连邻居建立LDP会话，会话进入Operational状态]:::state
            B5[为Loopback0 FEC分配本地入标签，接收邻居标签映射]:::action
            B6[生成LIB标签信息库]:::table
            B7[从LIB提取有效条目，生成LFIB标签转发表]:::table
        end

        subgraph 公网设备PE2
            C1[公网接口使能MPLS、MPLS LDP\n发布Loopback0与公网网段路由到IGP]:::action
            C2[IGP收敛完成，生成全局RIB公网路由表]:::table
            C3[从全局RIB提取生成全局FIB公网转发表]:::table
            C4[与直连邻居建立LDP会话，会话进入Operational状态]:::state
            C5[为Loopback0 FEC分配本地入标签，接收邻居标签映射]:::action
            C6[生成LIB标签信息库]:::table
            C7[从LIB提取有效条目，生成LFIB标签转发表]:::table
            C8[与PE1基于Loopback0建立MP-BGP VPNv4对等体]:::state
        end

        %% 阶段1内部连线
        A0 --> A1 & B1 & C1
        A1 --> A2 --> A3 --> A4 --> A5 --> A6 --> A7
        B1 --> B2 --> B3 --> B4 --> B5 --> B6 --> B7
        C1 --> C2 --> C3 --> C4 --> C5 --> C6 --> C7 --> C8
        A7 -->|PE1与PE2完成公网端到端LSP打通| C8
    end

    %% ======================
    %% 阶段2：VPN私网路由端到端学习与发布
    %% ======================
    subgraph 阶段2：VPN私网路由端到端学习与发布
        direction TB
        %% 泳道：CE-A、PE1、PE2、CE-B
        subgraph 用户边缘CE-A（站点A）
            D1[将本地私网网段192.168.1.0/24发布给PE1]:::action
        end

        subgraph 运营商边缘PE1
            E1[入接口绑定VPN1实例，接收CE-A私网路由]:::action
            E2[将私网路由存入VPN1实例的VRF RIB路由表]:::table
            E3[为私网路由添加RD 100:1，转换为VPNv4路由\n分配内层VPN标签，绑定Export RT 100:100]:::action
            E4[生成BGP VPNv4路由表]:::table
            E5[通过MP-BGP将VPNv4路由发布给PE2]:::action
        end

        subgraph 运营商边缘PE2
            F1[通过MP-BGP接收PE1发布的VPNv4路由]:::action
            F2{校验路由Export RT与本地VPN1 Import RT是否匹配?}:::judge
            F3[剥离RD，还原为纯IPv4私网路由\n将内层VPN标签存入LIB]:::action
            F4[将私网路由导入VPN1实例的VRF RIB路由表]:::table
            F5[从VRF RIB提取生成VPN1的VRF FIB转发表]:::table
            F6[将私网路由发布给直连的CE-B]:::action
        end

        subgraph 用户边缘CE-B（站点B）
            G1[接收PE2发布的站点A私网路由，生成本地路由表]:::action
        end

        %% 阶段2内部连线
        D1 --> E1
        E1 --> E2 --> E3 --> E4 --> E5
        E5 --> F1
        F1 --> F2
        F2 -->|匹配通过| F3 --> F4 --> F5 --> F6
        F2 -->|匹配失败| F7[丢弃该VPNv4路由，不导入VRF]:::action
        F6 --> G1
    end

    %% 流程结束
    H0([控制平面流程结束\n公网LSP隧道打通，私网路由端到端学习完成\n所有转发所需表项全部生成]):::startend
    G1 --> H0

    %% 反向流程标注（站点B→站点A路由发布）
    H1[注：站点B→站点A的私网路由发布流程完全对称]
    H0 --> H1
```