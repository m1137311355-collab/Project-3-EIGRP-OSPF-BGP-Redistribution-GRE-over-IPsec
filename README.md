# Project 3: 企業級多協定路由重分發 (Route Redistribution) 與 GRE over IPsec 安全 VPN 整合實作

**平台：** Cisco IOS / GNS3  
**關鍵技術：** Multi-Routing Protocols (EIGRP, OSPF, BGP), Route Redistribution, GRE over IPsec VPN
---

## 專案簡介與架構

### 傳統問題
異構網路（多路由協定）難以互通，且跨公網傳輸缺乏安全加密與動態路由支援

### 解決方案與核心亮點
* **高可用性與擴充性 (Availability & Scalability)：** 企業包含多個自治系統與區域（EIGRP AS 100, OSPF Area 0, BGP AS 64512/64513），傳統點對點靜態設定擴充困難[cite: 1]。本專案透過邊界路由器雙向路由重分發 (Route Redistribution) 實現全網動態互通，並搭配 GRE 隧道架構封裝動態路由協定封包
* **安全性 (Security)：** 純 GRE 隧道缺乏加密能力（明文傳輸），為了保護敏感企業資料，疊加 IPsec (IKEv1/ESP) 進行機密性保護。同時關閉路由器與交換器上未使用的協定及遺留功能（如關閉未使用的介面）以增加安全性與防範網路攻擊。

---

## Topology (邏輯拓撲圖)
EIGRP-1] -- (10.1.0.0/30) -- [EIGRP-2] -- (20.1.0.0/30) -- [EIGRP-OSPF] -- (30.1.0.0/30) -- [OSPF-1] -- (40.1.0.0/30) -- [OSPF-BGP] -- (50.1.0.0/30) -- [BGP-1]
|                                                                                                                                              |
============================================= Tunnel253 (GRE over IPsec) =======================================================================

---

## IP Address Table

| Device | Interface | IP Address | Subnet Mask | Description |
| :--- | :--- | :--- | :--- | :--- |
| **EIGRP-1** | g0/0 | 10.1.0.1 | /30 | Connect to router EIGRP-2|
| | g0/1-7 | Unassigned | N/A | Shut down|
| | Tunnel253 | 192.168.100.1 | /30 | Tunnel to router BGP-1 |
| **EIGRP-2** | g0/0 | 10.1.0.2 | /30 | Connect to router EIGRP-1 |
| | g0/1 | 20.1.0.1 | /30 | Connect to router EIGRP-OSPF |
| | g0/2-3 | Unassigned | N/A | Shut down|
| **EIGRP-OSPF** | g0/0 | 30.1.0.1 | /30 | Connect to router OSPF-1 |
| | g0/1 | 20.1.0.2 | /30 | Connect to router EIGRP-2|
| | g0/2-3 | Unassigned | N/A | Shut down|
| **OSPF-1** | g0/0 | 30.1.0.2 | /30 | Connect to router EIGRP-OSPF |
| | g0/1 | 40.1.0.1 | /30 | Connect to router OSPF-BGP|
| | g0/2-3 | Unassigned | N/A | Shut down|
| **OSPF-BGP** | g0/0 | 40.1.0.2 | /30 | Connect to router OSPF-1 |
| | g0/1 | 50.1.0.1 | /30 | Connect to router BGP-1|
| | g0/2-3 | Unassigned | N/A | Shut down|
| **BGP-1** | g0/0 | Unassigned | N/A | Shut down |
| | g0/1 | 50.1.0.2 | /30 | Connect to router OSPF-BGP |
| | g0/2 | Unassigned | N/A | Shut down |
| | Tunnel253 | 192.168.100.2 | /30 | Tunnel to router EIGRP-1 |

---

## 驗證與輸出 (Verification & Output)

### 1. EIGRP Neighborship
```text
EIGRP-2#show ip eigrp neighbors
EIGRP-IPv4 VR (EIGRP_1) Address-Family Neighbors for AS (100)
H   Address         Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                    (sec)         (ms)       Cnt Num
1   20.1.0.2        Gi0/1           12   00:04:02    1   100   0   11
0   10.1.0.1        Gi0/0           14   00:19:24    1   100   0   11


2. OSPF Neighborship
OSPF-1#show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
50.1.0.1          1   FULL/BDR        00:00:33    40.1.0.2        GigabitEthernet0/1
30.1.0.1          1   FULL/DR         00:00:36    30.1.0.1        GigabitEthernet0/0

3. Routing Table & Traceroute (EIGRP-1 to BGP-1 End-to-End)
EIGRP-1#show ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 

Gateway of last resort is not set

      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.1.0.0/30 is directly connected, GigabitEthernet0/0
L        10.1.0.1/32 is directly connected, GigabitEthernet0/0
      20.0.0.0/16 is subnetted, 1 subnets
D        20.1.0.0 [90/15368] via 10.1.0.2, 02:06:55, GigabitEthernet0/0
      30.0.0.0/30 is subnetted, 1 subnets
D EX     30.1.0.0 [170/1034240] via 10.1.0.2, 01:51:33, GigabitEthernet0/0
      40.0.0.0/30 is subnetted, 1 subnets
D EX     40.1.0.0 [170/1034240] via 10.1.0.2, 01:50:27, GigabitEthernet0/0
      50.0.0.0/30 is subnetted, 1 subnets
D EX     50.1.0.0 [170/1034240] via 10.1.0.2, 01:38:07, GigabitEthernet0/0[cite: 1]
      192.168.100.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.100.0/30 is directly connected, Tunnel253[cite: 1]
L        192.168.100.1/32 is directly connected, Tunnel253[cite: 1]

EIGRP-1#traceroute 50.1.0.2
Type escape sequence to abort.
Tracing the route to 50.1.0.2
VRF info: (vrf in name/id, vrf out name/id)
  1 10.1.0.2 3 msec 2 msec 1 msec[cite: 1]
  2 20.1.0.2 4 msec 4 msec 4 msec[cite: 1]
  3 30.1.0.2 4 msec 4 msec 4 msec[cite: 1]
  4 40.1.0.2 5 msec 5 msec 4 msec[cite: 1]
  5 50.1.0.2 6 msec 5 msec 5 msec[cite: 1]
EIGRP-1#traceroute 50.1.0.2
Type escape sequence to abort.
Tracing the route to 50.1.0.2
VRF info: (vrf in name/id, vrf out name/id)
  1 10.1.0.2 3 msec 2 msec 1 msec[cite: 1]
  2 20.1.0.2 4 msec 4 msec 4 msec[cite: 1]
  3 30.1.0.2 4 msec 4 msec 4 msec[cite: 1]
  4 40.1.0.2 5 msec 5 msec 4 msec[cite: 1]
  5 50.1.0.2 6 msec 5 msec 5 msec[cite: 1]

4. Tunnel Ping (GRE over IPsec)
BGP-1#ping 192.168.100.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.100.1, timeout is 2 seconds:
Success rate is 100 percent (5/5), round-trip min/avg/max = 9/10/14 ms[cite: 1]

5. IPsec Phase 1 & Phase 2 Verification
BGP-1#show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
10.1.0.1        50.1.0.2        QM_IDLE           1001 ACTIVE[cite: 1]

EIGRP-1#show crypto ipsec sa
interface: Tunnel253
    Crypto map tag: Tunnel253-head-0, local addr 10.1.0.1[cite: 1]
   protected vrf: (none)
   local ident (addr/mask/prot/port): (10.1.0.1/255.255.255.255/47/0)[cite: 1]
   remote ident (addr/mask/prot/port): (50.1.0.2/255.255.255.255/47/0)[cite: 1]
   current_peer 50.1.0.2 port 500[cite: 1]
     PERMIT, flags={origin is acl,}[cite: 1]
    #pkts encaps: 10, #pkts encrypt: 10, #pkts digest: 10[cite: 1]
    #pkts decaps: 10, #pkts decrypt: 10, #pkts verify: 10[cite: 1]


安全性補強設定 (Security Hardening)
關閉未使用的連接埠 (Shut Down Non-Used Ports)
  EIGRP-1#show ip interface brief
  Interface                  IP-Address      OK? Method Status                  Protocol
  GigabitEthernet0/0         10.1.0.1        YES manual up                      up[cite: 1]
  GigabitEthernet0/1         unassigned      YES NVRAM  administratively down   down[cite: 1]
  GigabitEthernet0/2         unassigned      YES NVRAM  administratively down   down[cite: 1]
  GigabitEthernet0/3         unassigned      YES NVRAM  administratively down   down[cite: 1]
  GigabitEthernet0/4         unassigned      YES NVRAM  administratively down   down[cite: 1]
  GigabitEthernet0/5         unassigned      YES NVRAM  administratively down   down[cite: 1]
  GigabitEthernet0/6         unassigned      YES NVRAM  administratively down   down[cite: 1]
  GigabitEthernet0/7         unassigned      YES NVRAM  administratively down   down[cite: 1]
  NVIO                       10.1.0.1        YES unset  up                      up[cite: 1]
  Tunnel253                  192.168.100.1   YES manual up                      up[cite: 1]

遠端存取與密碼安全限制 (Only SSH & Encrypted Password)
僅允許 SSH 連線，並對密碼進行加密保護[cite: 1]：
  line vty 5 15
  password 7 047828283F
  login
  transport input ssh
