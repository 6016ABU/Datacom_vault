---
title: "BGP技术学习 | BGP 安全"
source: "https://mp.weixin.qq.com/s/Rx8hbQxDQEyHDiTvypiX6Q"
author:
  - "[[运维搬砖]]"
published:
created: 2026-05-06
description: "BGP 是在互联网早期\x26quot;信任网络\x26quot;的理念下设计的，缺乏内建的安全机制。任何 BGP 邻居通告的路由信息都会被默认接受和传播，这使得 BGP 容易受到各种攻击和误操作的影响。随着互联网基础设施的重要性日益增加，BGP 安全已成为网络工程领域的核心议题。"
tags:
  - "clippings"
---
运维搬砖 *2026年3月26日 07:49*

## 第六章 BGP 安全

---

## 概述

BGP 是在互联网早期"信任网络"的理念下设计的，缺乏内建的安全机制。任何 BGP 邻居通告的路由信息都会被默认接受和传播，这使得 BGP 容易受到各种攻击和误操作的影响。随着互联网基础设施的重要性日益增加，BGP 安全已成为网络工程领域的核心议题。

---

## 6.1 基本安全措施

### 6.1.1 MD5 认证（RFC 2385）

TCP MD5 签名是最基本的 BGP 邻居认证机制。它在 TCP 层为每个报文计算 MD5 散列值，防止 BGP 会话被劫持或注入伪造消息。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/2sHfY8zAtFz7HVLNLSDGbdXBiaUMl8icLjia1RsicI5o7tHOIJPCibn14pHwYiaAVaaDhzCtuuUgZD8D0BdO15KbNMfvgZJwT2cHPNiaYRbmWibXTO4/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=0)

### 6.1.2 TTL 安全（GTSM，RFC 5082）

**GTSM（Generalized TTL Security Mechanism）** 通过检查接收报文的 TTL 值来确保 BGP 消息来自直连邻居，有效防御远程伪造攻击。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**GTSM vs ebgp-multihop 对比** ：

| 特性 | GTSM | ebgp-multihop |
| --- | --- | --- |
| 目的 | 安全加固 | 允许非直连邻居 |
| TTL 行为 | 发送 TTL=255，检查最小 TTL | 设置发送的 TTL 值 |
| 安全性 | 高（过滤远程攻击） | 低（扩大了攻击面） |
| 互斥 | 是 | 是 |

### 6.1.3 最大前缀数限制（Maximum-prefix）

限制从邻居接收的路由前缀数量，防止路由泄漏导致的路由表爆炸。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 6.1.4 路由过滤（Bogon 过滤）

**Bogon** 是指不应该出现在公共互联网路由表中的地址空间。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> **最佳实践** ：同时过滤过长前缀（如 IPv4 /25 及以上、IPv6 /49 及以上），因为这些通常是配置错误或攻击。

---

## 6.2 高级安全机制

### 6.2.1 RPKI（Resource Public Key Infrastructure）

RPKI 是目前最重要的 BGP 安全增强机制，它通过加密方式验证 **路由起源的合法性** ——即"谁有权通告某个 IP 前缀"。

**核心概念** ：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**ROA（Route Origin Authorization）** ：

ROA 是 RPKI 的核心数据对象，它声明了"哪个 AS 有权通告哪个 IP 前缀"。

| ROA 字段 | 说明 |
| --- | --- |
| 前缀 | IP 地址前缀（如 203.0.113.0/24） |
| 起源 AS | 授权通告此前缀的 AS 号 |
| 最大前缀长度 | 允许的最长前缀掩码 |
| 有效期 | ROA 的起止时间 |

**路由起源验证（Route Origin Validation, ROV）** ：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**RPKI 验证流程** ：

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**RPKI 配置示例（Cisco IOS-XR）** ：

```
! 配置 RPKI 缓存服务器
router bgp 65001
 rpki server 10.0.0.100
  transport tcp port 8282
  refresh-time 600
 !
 address-family ipv4 unicast
  ! 对 Invalid 路由降低 LOCAL_PREF
  route-policy RPKI-POLICY
   if validation-state is invalid then
    drop
   elseif validation-state is valid then
    set local-preference 200
   else
    pass
   endif
  end-policy
```

**华为设备RPKI配置示例（VRP系统）**

```
# 配置RPKI缓存服务器
rpki server 10.0.0.100 port 8282 refresh 600

# 进入BGP视图
bgp 65001
 peer 192.168.1.2 as-number 65002  # 示例：配置对等体，实际按需添加

 # 启用RPKI验证功能
 rpki enable

 # 进入IPv4单播地址族
 address-family ipv4 unicast
  # 应用路由策略控制RPKI验证结果
  route-policy RPKI-POLICY permit node 10
   if-match rpki-state invalid
   apply drop
  route-policy RPKI-POLICY permit node 20
   if-match rpki-state valid
   apply local-preference 200
  route-policy RPKI-POLICY permit node 30
   # 默认允许其他状态（unknown）通过，不修改属性

  # 将策略应用到入方向路由
  peer 192.168.1.2 route-policy RPKI-POLICY import
  peer 192.168.1.2 rpki enable
```

**RPKI 部署现状** ：

- • 全球约 40-50% 的路由已有 ROA 覆盖
- • 主要运营商（如 AT&T、NTT、Cloudflare 等）已启用 ROV
- • 部署趋势持续增长，各国政府也在推动

### 6.2.2 BGPsec（RFC 8205）

BGPsec 在 RPKI 的基础上更进一步，不仅验证路由起源，还验证 **整个 AS\_PATH 的完整性** 。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

> **现实情况** ：BGPsec 的计算开销大、部署复杂，目前几乎没有实际大规模部署。业界更倾向于通过 RPKI + ASPA 的组合来实现路径安全。

### 6.2.3 ASPA（Autonomous System Provider Authorization）

ASPA 是一种新兴的安全机制，旨在防止 **路由泄漏（Route Leak）** 问题。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 6.2.4 IRR（Internet Routing Registry）

IRR 是一种基于数据库的路由注册系统，运营商可以查询 IRR 来验证路由通告的合法性。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

---

## 6.3 常见 BGP 安全事件

### 6.3.1 BGP 劫持（Hijacking）

BGP 劫持是指一个 AS 错误地（或恶意地）通告了不属于自己的 IP 前缀，导致流量被引导到错误的目的地。

```
正常情况：
  AS65001 合法拥有 203.0.113.0/24
  AS65001 通告 203.0.113.0/24 → 全球路由正确

劫持场景：
  AS66666 (恶意) 也通告 203.0.113.0/24
  由于 BGP 没有验证机制，部分网络可能选择 AS66666 的路由
  → 流量被劫持到 AS66666

更精确的劫持：
  AS66666 通告 203.0.113.0/25 (更长前缀)
  最长前缀匹配原则 → 几乎所有网络都会选择 /25
  → 流量被完全劫持
```

**著名案例** ：

**2008 年 Pakistan/YouTube 事件** ：

```
事件经过：
1. 巴基斯坦政府要求封锁 YouTube (208.65.153.0/24)
2. 巴基斯坦电信 (AS17557) 内部配置了黑洞路由
3. 错误地将 208.65.153.0/24 通过 BGP 通告给了上游 PCCW (AS3491)
4. PCCW 未做过滤，将路由传播到全球
5. 全球用户约 2 小时无法访问 YouTube

根因分析：
  - 巴基斯坦电信的配置错误（应该只在内部生效）
  - PCCW 没有对客户路由做前缀过滤
  - YouTube 当时没有部署 RPKI

教训：
  ① 上游运营商必须对客户路由做严格过滤
  ② RPKI ROA 可以防止此类事件
  ③ 网络边界安全至关重要
```

### 6.3.2 路由泄漏（Route Leak）

路由泄漏是指一个 AS 将不应该传播的路由发送到了错误的方向。这通常不是恶意行为，而是配置错误。

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

**著名案例** ：

**2019 年 Verizon/Cloudflare 事件** ：

```
事件经过：
1. 一个小型 ISP (AS396531) 将从 Cloudflare 学到的路由
   泄漏给了上游 Allegheny Technologies
2. Allegheny Technologies 又传给了 Verizon (AS701)
3. Verizon 没有对客户路由做充分过滤
4. 导致大量 Cloudflare 流量经过这个小型 ISP → 网络拥塞
5. 数百个网站受到影响，持续约 3 小时
```

### 6.3.3 防御措施总结

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### MANRS（Mutually Agreed Norms for Routing Security）

MANRS 是由 ISOC 发起的路由安全协作框架，参与者承诺执行以下安全措施：

1. 1\. **防止错误路由传播** ：实施路由过滤
2. 2\. **防止非法源地址流量** ：实施 BCP 38/uRPF
3. 3\. **促进全球协作** ：维护联系信息、参与社区协作
4. 4\. **推动 RPKI 部署** ：发布 ROA、执行 ROV

---

## 本章小结

1. 1\. **基础安全措施** （MD5 认证、GTSM、Maximum-prefix、Bogon 过滤）是所有 BGP 部署的最低安全要求。
2. 2\. **RPKI** 是当前最重要的 BGP 安全增强机制，通过 ROA 验证路由起源的合法性。
3. 3\. **BGP 劫持** 和 **路由泄漏** 是最常见的 BGP 安全事件，历史上多次造成大规模互联网中断。
4. 4\. BGP 安全需要 **多层防御** ：从基础过滤到加密验证，再到监控检测，形成完整的安全体系。
5. 5\. **RPKI 部署势在必行** ——这已不是"是否需要"的问题，而是"何时完成"的问题。

---

> **思考题**
> 
> 1. 1\. 如果一个攻击者通告了 203.0.113.0/25 来劫持 203.0.113.0/24，RPKI ROA 能否检测到？如何配置 ROA 来防御？
> 2. 2\. 为什么路由泄漏比路由劫持更难防御？ASPA 如何帮助解决这个问题？
> 3. 3\. 作为一个小型企业用户，你能采取哪些措施来保护自己的 BGP 路由安全？
> 4. 4\. GTSM (TTL Security) 为什么不能与 ebgp-multihop 同时使用？

---

*全文完，觉得不错的话就* ***点个赞*** *或者* ***关注*** *吧*

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

![图片](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

运维搬砖大学习 · 目录

继续滑动看下一个

运维搬砖

向上滑动看下一个