# 🔐 Encrypted MPLS L3VPN — Two Tenants, GET-VPN, BGP-Free Core

!\[Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?style=flat\&logo=cisco\&logoColor=white)
!\[MPLS](https://img.shields.io/badge/MPLS-L3VPN%20%C2%B7%20MP--BGP-blue?style=flat)
!\[GET-VPN](https://img.shields.io/badge/GET--VPN-GDOI-orange?style=flat)
!\[Status](https://img.shields.io/badge/Status-Phased%20Build-yellow?style=flat)

A service-provider style **MPLS L3VPN** carrying **two separate customers** across one shared core — each isolated in its own VRF, each **encrypted end-to-end with its own GET-VPN group**. The provider core is **BGP-free** (the P router label-switches only).

* **Customer A:** New York ↔ Köln — isolated in VRF CUST-A, encrypted by GET-VPN group A
* **Customer B:** Abu Dhabi ↔ Barcelona — isolated in VRF CUST-B, encrypted by GET-VPN group B

The two customers share the same physical PEs and core, yet **cannot reach each other** (VRF isolation), and each customer's traffic is **encrypted** across the provider (GET-VPN). This is exactly the multi-tenant, compliance-driven design a carrier or large enterprise runs: private *and* isolated *and* encrypted.

\---

## Why This Design

|Feature|Delivered by|Proven by|
|-|-|-|
|Private any-to-any WAN|MPLS L3VPN (MP-BGP VPNv4)|customer sites reach each other over the core|
|**Tenant isolation**|**VRF per customer**|**A cannot reach B — even on the same PEs**|
|Scalable core|**BGP-free P**|P has no BGP state, only labels + IGP|
|Encryption|GET-VPN per customer|each customer's traffic encrypted, header preserved|

> The isolation is the whole point of VRF: Customer A and Customer B ride the \*same\* routers but live in \*separate\* routing tables. Route-targets decide who imports whose routes — and A's RT never matches B's, so they never see each other. GET-VPN then encrypts each tenant independently.

\---

## Topology

```
  Customer A                                          Customer A
  New York (CE1/GM1)                                  Köln (CE3/GM3) + KS-A
  LAN 10.1.1.0/24                                     LAN 10.1.3.0/24 · KS-A 10.1.30.2 (p2p to CE3)
        │ OSPF 172.16.11.0/30              172.16.23.0/30 OSPF │
        │                                                      │
   ┌────┴──────────────┐                        ┌──────────────┴────┐
   │       PE1         │                        │       PE2         │
   │ VRF CUST-A        │      BGP-FREE CORE      │ VRF CUST-A        │
   │ VRF CUST-B        │  PE1 ═ P ═ PE2 (labels) │ VRF CUST-B        │
   │ Lo0 1.1.1.1       │  MP-BGP VPNv4 PE1<->PE2 │ Lo0 3.3.3.3       │
   └────┬──────────────┘   P Lo0 2.2.2.2 (no BGP)└──────────────┬────┘
        │ OSPF 172.16.12.0/30              172.16.24.0/30 OSPF │
  LAN 10.2.2.0/24                                     LAN 10.2.4.0/24 · KS-B 10.2.40.2 (p2p to CE4)
  Abu Dhabi (CE2/GM2)                                 Barcelona (CE4/GM4) + KS-B
  Customer B                                          Customer B

  Core:   OSPF area 0 + LDP    |   P = BGP-free (label switching only)
  VPN:    MP-BGP VPNv4 (PE1<->PE2)   |   VRF CUST-A (RD/RT 65000:1), VRF CUST-B (RD/RT 65000:2)
  PE-CE:  OSPF per VRF, redistributed into MP-BGP
  Crypto: GET-VPN group A (NY+Köln), GET-VPN group B (AbuDhabi+Barcelona)
```

**Roles**

|Device|Role|
|-|-|
|PE1 / PE2|Provider Edge — **both VRFs** (CUST-A + CUST-B), MP-BGP VPNv4, OSPF to CEs|
|P|Provider core — OSPF + LDP only, **BGP-free**|
|CE1 New York|Customer A edge + **GM1**|
|CE3 Köln|Customer A edge + **GM3**, hosts **KS-A**|
|CE2 Abu Dhabi|Customer B edge + **GM2**|
|CE4 Barcelona|Customer B edge + **GM4**, hosts **KS-B**|
|KS-A / KS-B|GET-VPN Key Server per customer|

\---

## Addressing

**Loopbacks**

|Device|Lo0|
|-|-|
|PE1|1.1.1.1/32|
|P|2.2.2.2/32|
|PE2|3.3.3.3/32|

**Core links (OSPF area 0 + LDP)**

|Link|Subnet|
|-|-|
|PE1 ↔ P|10.0.12.0/30 (PE1 .1, P .2)|
|P ↔ PE2|10.0.23.0/30 (P .1, PE2 .2)|

**PE-CE links (OSPF, per VRF)**

|Link|Customer|Subnet|
|-|-|-|
|PE1 ↔ CE1 (New York)|A|172.16.11.0/30|
|PE1 ↔ CE2 (Abu Dhabi)|B|172.16.12.0/30|
|PE2 ↔ CE3 (Köln)|A|172.16.23.0/30|
|PE2 ↔ CE4 (Barcelona)|B|172.16.24.0/30|

**Customer LANs + Key Servers**

|Customer|Site|LAN|KS|
|-|-|-|-|
|A|New York (CE1)|10.1.1.0/24|—|
|A|Köln (CE3)|10.1.3.0/24|KS-A = 10.1.4.2 (p2p 10.1.30.0/30 to CE3)|
|B|Abu Dhabi (CE2)|10.2.2.0/24|—|
|B|Barcelona (CE4)|10.2.4.0/24|KS-B = 10.2.5.2 (p2p 10.2.40.0/30 to CE4)|

**VRFs**

|VRF|RD|RT import/export|
|-|-|-|
|CUST-A|65000:1|65000:1|
|CUST-B|65000:2|65000:2|

**GET-VPN groups**

|Group|Identity|Encrypts|KS|
|-|-|-|-|
|CUST-A-GETVPN|999|10.1.0.0/16 ↔ 10.1.0.0/16|KS-A (10.1.30.2)|
|CUST-B-GETVPN|888|10.2.0.0/16 ↔ 10.2.0.0/16|KS-B (10.2.40.2)|

\---

## Build Phases

|Phase|What|Status|
|-|-|-|
|1|Core IGP + MPLS/LDP (PE1–P–PE2)|⬜|
|2|MP-BGP VPNv4 (PE1 ↔ PE2), P BGP-free|⬜|
|3|VRF CUST-A + CUST-B, PE-CE OSPF, redistribution|⬜|
|4|**Verify PLAIN + ISOLATION:** A↔A works, B↔B works, A↔B blocked|⬜|
|5|Key Servers KS-A + KS-B|⬜|
|6|GET-VPN on the CEs (both groups)|⬜|
|7|**Verify ENCRYPTED:** each customer's traffic encrypted|⬜|

> ⚠️ Finish Phases 1–4 (both VRFs routing, isolation proven, PLAIN) before any GET-VPN.

\---

## Phase 1 — Core IGP + MPLS/LDP

```cisco
! ===== PE1 =====
hostname PE1
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
interface Ethernet0/0
 description CORE-TO-P
 ip address 10.0.12.1 255.255.255.252
 mpls ip
 no shutdown
router ospf 1
 network 1.1.1.1 0.0.0.0 area 0
 network 10.0.12.0 0.0.0.3 area 0
mpls ldp router-id Loopback0 force
```

```cisco
! ===== P (BGP-FREE core) =====
hostname P
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
interface Ethernet0/0
 description TO-PE1
 ip address 10.0.12.2 255.255.255.252
 mpls ip
 no shutdown
interface Ethernet0/1
 description TO-PE2
 ip address 10.0.23.1 255.255.255.252
 mpls ip
 no shutdown
router ospf 1
 network 2.2.2.2 0.0.0.0 area 0
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
mpls ldp router-id Loopback0 force
! No BGP, no VRF — P only switches labels.
```

```cisco
! ===== PE2 =====
hostname PE2
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
interface Ethernet0/0
 description CORE-TO-P
 ip address 10.0.23.2 255.255.255.252
 mpls ip
 no shutdown
router ospf 1
 network 3.3.3.3 0.0.0.0 area 0
 network 10.0.23.0 0.0.0.3 area 0
mpls ldp router-id Loopback0 force
```

**Verify Phase 1**

```cisco
show mpls ldp neighbor          ! LDP up PE1-P and P-PE2
ping 3.3.3.3 source 1.1.1.1     ! PE1 loopback reaches PE2 loopback via P
```

\---

## Phase 2 — MP-BGP VPNv4 (PE1 ↔ PE2)

```cisco
! ===== PE1 =====
router bgp 65000
 neighbor 3.3.3.3 remote-as 65000
 neighbor 3.3.3.3 update-source Loopback0
 address-family vpnv4
  neighbor 3.3.3.3 activate
  neighbor 3.3.3.3 send-community extended
 exit-address-family
```

```cisco
! ===== PE2 =====
router bgp 65000
 neighbor 1.1.1.1 remote-as 65000
 neighbor 1.1.1.1 update-source Loopback0
 address-family vpnv4
  neighbor 1.1.1.1 activate
  neighbor 1.1.1.1 send-community extended
 exit-address-family
```

**Verify Phase 2**

```cisco
show bgp vpnv4 unicast all summary     ! PE1<->PE2 VPNv4 Established (P has no BGP)
```

\---

## Phase 3 — Two VRFs + PE-CE OSPF

Each PE holds **both** VRFs. Customer A and Customer B get separate tables, separate RTs — that is the isolation.

```cisco
! ===== PE1 — both VRFs =====
vrf definition CUST-A
 rd 65000:1
 address-family ipv4
  route-target export 65000:1
  route-target import 65000:1
 exit-address-family
vrf definition CUST-B
 rd 65000:2
 address-family ipv4
  route-target export 65000:2
  route-target import 65000:2
 exit-address-family

interface Ethernet0/1
 description TO-CE1-NEWYORK (Cust A)
 vrf forwarding CUST-A
 ip address 172.16.11.1 255.255.255.252
 no shutdown
interface Ethernet0/2
 description TO-CE2-ABUDHABI (Cust B)
 vrf forwarding CUST-B
 ip address 172.16.12.1 255.255.255.252
 no shutdown

router ospf 2 vrf CUST-A
 network 172.16.11.0 0.0.0.3 area 0
 redistribute bgp 65000 subnets
router ospf 3 vrf CUST-B
 network 172.16.12.0 0.0.0.3 area 0
 redistribute bgp 65000 subnets

router bgp 65000
 address-family ipv4 vrf CUST-A
  redistribute ospf 2
 exit-address-family
 address-family ipv4 vrf CUST-B
  redistribute ospf 3
 exit-address-family
```

```cisco
! ===== PE2 — both VRFs =====
vrf definition CUST-A
 rd 65000:1
 address-family ipv4
  route-target export 65000:1
  route-target import 65000:1
 exit-address-family
vrf definition CUST-B
 rd 65000:2
 address-family ipv4
  route-target export 65000:2
  route-target import 65000:2
 exit-address-family

interface Ethernet0/1
 description TO-CE3-KOLN (Cust A)
 vrf forwarding CUST-A
 ip address 172.16.23.1 255.255.255.252
 no shutdown
interface Ethernet0/2
 description TO-CE4-BARCELONA (Cust B)
 vrf forwarding CUST-B
 ip address 172.16.24.1 255.255.255.252
 no shutdown

router ospf 2 vrf CUST-A
 network 172.16.23.0 0.0.0.3 area 0
 redistribute bgp 65000 subnets
router ospf 3 vrf CUST-B
 network 172.16.24.0 0.0.0.3 area 0
 redistribute bgp 65000 subnets

router bgp 65000
 address-family ipv4 vrf CUST-A
  redistribute ospf 2
 exit-address-family
 address-family ipv4 vrf CUST-B
  redistribute ospf 3
 exit-address-family
```

```cisco
! ===== CE1 New York (Customer A) =====
hostname CE1
interface Ethernet0/0
 ip address 172.16.11.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 description NEWYORK-LAN
 ip address 10.1.1.1 255.255.255.0
 no shutdown
router ospf 1
 network 172.16.11.0 0.0.0.3 area 0
 network 10.1.1.0 0.0.0.255 area 0
```

```cisco
! ===== CE3 Köln (Customer A) — connects to KS-A on a dedicated link =====
hostname CE3
interface Ethernet0/0
 description TO-PE2
 ip address 172.16.23.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 description KOLN-LAN
 ip address 10.1.3.1 255.255.255.0
 no shutdown
interface Ethernet0/2
 description TO-KS-A (point-to-point)
 ip address 10.1.30.1 255.255.255.252
 no shutdown
router ospf 1
 network 172.16.23.0 0.0.0.3 area 0
 network 10.1.3.0 0.0.0.255 area 0
 network 10.1.30.0 0.0.0.3 area 0        ! advertise the KS link so GMs can reach KS-A
```

```cisco
! ===== CE2 Abu Dhabi (Customer B) =====
hostname CE2
interface Ethernet0/0
 ip address 172.16.12.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 description ABUDHABI-LAN
 ip address 10.2.2.1 255.255.255.0
 no shutdown
router ospf 1
 network 172.16.12.0 0.0.0.3 area 0
 network 10.2.2.0 0.0.0.255 area 0
```

```cisco
! ===== CE4 Barcelona (Customer B) — connects to KS-B on a dedicated link =====
hostname CE4
interface Ethernet0/0
 description TO-PE2
 ip address 172.16.24.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 description BARCELONA-LAN
 ip address 10.2.4.1 255.255.255.0
 no shutdown
interface Ethernet0/2
 description TO-KS-B (point-to-point)
 ip address 10.2.40.1 255.255.255.252
 no shutdown
router ospf 1
 network 172.16.24.0 0.0.0.3 area 0
 network 10.2.4.0 0.0.0.255 area 0
 network 10.2.40.0 0.0.0.3 area 0        ! advertise the KS link so GMs can reach KS-B
```

\---

## Phase 4 — Verify PLAIN + ISOLATION (the important checkpoint)

```cisco
! Customer A reachability (should WORK):
CE1(NewYork)# ping 10.1.3.1 source 10.1.1.1     ! NY → Köln  ✅

! Customer B reachability (should WORK):
CE2(AbuDhabi)# ping 10.2.4.1 source 10.2.2.1    ! AbuDhabi → Barcelona  ✅

! ISOLATION — Customer A must NOT reach Customer B:
CE1(NewYork)# ping 10.2.2.1 source 10.1.1.1     ! NY → AbuDhabi  ✗ FAILS = correct ❌
```

```
On the PEs, prove the tables are separate:
  show ip route vrf CUST-A     ! only 10.1.x prefixes
  show ip route vrf CUST-B     ! only 10.2.x prefixes
  → A's table has no B routes and vice versa = ISOLATION ✅
```

> 🧠 \*\*This is the payoff of VRF.\*\* Same PEs, same core, but Customer A's routes live only in CUST-A and Customer B's only in CUST-B. The route-targets never overlap (65000:1 vs 65000:2), so neither imports the other's routes. New York literally has no route to Abu Dhabi — isolation enforced by the control plane, not an ACL.

\---

## Phase 5 — Key Servers (KS-A, KS-B)

One Key Server per customer group. KS-A sits on Köln's LAN, KS-B on Barcelona's — each reachable by its own customer's GMs over that customer's VPN.

```cisco
! ===== KS-A (Customer A) — on the point-to-point link to CE3 =====
hostname KS-A
!
interface Ethernet0/0
 description TO-CE3
 ip address 10.1.30.2 255.255.255.252
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.1.30.1       ! default via CE3 → reaches all GMs over the VPN
!
ip domain name lab.local
crypto key generate rsa modulus 2048 label GETVPN-A

crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
crypto isakmp key GETKEY-A address 0.0.0.0

crypto ipsec transform-set GET-TS esp-aes 256 esp-sha-hmac
 mode tunnel
crypto ipsec profile GET-PROFILE
 set transform-set GET-TS

ip access-list extended GET-A-POLICY
 deny ip 10.1.4.0 0.0.0.3 any            ! do NOT encrypt KS registration/control traffic
 deny ip any 10.1.4.0 0.0.0.3
 permit ip 10.1.0.0 0.0.255.255 10.1.0.0 0.0.255.255

crypto gdoi group CUST-A-GETVPN
 identity number 999
 server local
  rekey authentication mypubkey rsa GETVPN-A
  rekey transport unicast
  sa ipsec 1
   profile GET-PROFILE
   match address ipv4 GET-A-POLICY
   replay counter window-size 64
  address ipv4 10.1.4.2
```

```cisco
! ===== KS-B (Customer B) — on the point-to-point link to CE4 (mirror, group 888) =====
hostname KS-B
!
interface Ethernet0/0
 description TO-CE4
 ip address 10.2.40.2 255.255.255.252
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.2.40.1       ! default via CE4
!
ip domain name lab.local
crypto key generate rsa modulus 2048 label GETVPN-B
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
crypto isakmp key GETKEY-B address 0.0.0.0
crypto ipsec transform-set GET-TS esp-aes 256 esp-sha-hmac
 mode tunnel
crypto ipsec profile GET-PROFILE
 set transform-set GET-TS
ip access-list extended GET-B-POLICY
 deny ip 10.2.40.0 0.0.0.3 any            ! do NOT encrypt KS registration/control traffic
 deny ip any 10.2.5.0 0.0.0.3
 permit ip 10.2.0.0 0.0.255.255 10.2.0.0 0.0.255.255
crypto gdoi group CUST-B-GETVPN
 identity number 888
 server local
  rekey authentication mypubkey rsa GETVPN-B
  rekey transport unicast
  sa ipsec 1
   profile GET-PROFILE
   match address ipv4 GET-B-POLICY
   replay counter window-size 64
  address ipv4 10.2.5.2
```

\---

## Phase 6 — GET-VPN on the CEs (both groups)

```cisco
! ===== Customer A CEs (CE1 New York, CE3 Köln) =====
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
crypto isakmp key GETKEY-A address 10.1.4.2
crypto gdoi group CUST-A-GETVPN
 identity number 999
 server address ipv4 10.1.4.2
crypto map GET-MAP 10 gdoi
 set group CUST-A-GETVPN
interface Ethernet0/0
 crypto map GET-MAP
```

```cisco
! ===== Customer B CEs (CE2 Abu Dhabi, CE4 Barcelona) =====
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
crypto isakmp key GETKEY-B address 10.2.5.2
crypto gdoi group CUST-B-GETVPN
 identity number 888
 server address ipv4 10.2.5.2
crypto map GET-MAP 10 gdoi
 set group CUST-B-GETVPN
interface Ethernet0/0
 crypto map GET-MAP
```

**Verify Phase 6**

```cisco
CE1# show crypto gdoi                 ! Registered to KS-A
CE2# show crypto gdoi                 ! Registered to KS-B
KS-A# show crypto gdoi ks members     ! CE1, CE3 registered
KS-B# show crypto gdoi ks members     ! CE2, CE4 registered
```

\---

## Phase 7 — Verify ENCRYPTED (per customer)

```cisco
! Customer A — same NY↔Köln, now encrypted:
CE1# ping 10.1.3.1 source 10.1.1.1
CE1# show crypto ipsec sa             ! encaps/decaps climbing ✅

! Customer B — AbuDhabi↔Barcelona, encrypted:
CE2# ping 10.2.4.1 source 10.2.2.1
CE2# show crypto ipsec sa             ! encaps/decaps climbing ✅

! Isolation STILL holds (A can't reach B, encrypted or not):
CE1# ping 10.2.2.1 source 10.1.1.1    ! still fails = correct ❌
```

```
Result:
  Customer A: NY ↔ Köln — isolated (VRF) + encrypted (GET-VPN group A) ✅
  Customer B: AbuDhabi ↔ Barcelona — isolated (VRF) + encrypted (group B) ✅
  A ↔ B: blocked (VRF isolation) ✅
Two tenants, one core, each private and encrypted — with separate keys.
```

> 🧠 \*\*Everything stacks cleanly:\*\* the VRF isolates the tenants at the routing layer, MP-BGP/MPLS carries each tenant across the BGP-free core, and each customer's GET-VPN group encrypts only that customer's traffic (10.1.x for A, 10.2.x for B) with its own key from its own Key Server. Header preservation means the PE still routes each encrypted packet in the correct VRF.

\---

## Troubleshooting Log

*Document each real issue and its fix here as you build.*

Likely ones:

* **LDP not up** → `mpls ip` missing on a core interface, or LDP router-id.
* **VPNv4 session down** → peer on loopbacks, `update-source Loopback0`, loopbacks in OSPF.
* **A route appears in B (or vice versa)** → wrong RT import/export; A must be 65000:1 only, B 65000:2 only.
* **CE can't reach the KS** → the KS LAN must be advertised in that customer's VRF so its GMs can register.
* **Registered but not encrypting** → crypto map must be on the PE-facing interface.

\---

## Key Concepts Reference

* **VRF isolation** — each customer has its own routing table; route-targets decide imports, so tenants never see each other even on shared PEs.
* **RD vs RT** — RD makes a customer's prefixes unique in MP-BGP; RT controls which VRFs import/export them (the actual isolation lever).
* **BGP-free core** — the P router runs only IGP + LDP, switching labels with no BGP state.
* **MP-BGP VPNv4** — carries customer (VPN) routes between PEs across the core.
* **GET-VPN / GDOI** — tunnelless group encryption; one group key per customer from that customer's Key Server.
* **Header preservation** — keeps customer IPs so the PE/VRF still routes the encrypted packet (why GET-VPN works over MPLS L3VPN).

\---

## Disclaimer

Personal home/lab environment for learning. Keys, secrets, RDs/RTs, and addressing are lab-only values — replace them before any real use.

