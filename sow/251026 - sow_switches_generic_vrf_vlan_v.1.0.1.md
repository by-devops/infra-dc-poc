
# Statement of Work (SoW) — Switches Only (Generic, Fabric VRFs)

**Document Type:** SoW (Scope / Deliverables / Acceptance & Tests)  
**Applies To:** Switch roles (Fabric, PTP/Fabric-Services, OOB). No firewall/pfSense configuration included.  
**Status:** Draft, parameterized (generic VRFs/VLANs/IPs/DSCP).

---

## 1) Scope
Design, implement, validate, and hand over a dual-fabric, switches-only IP production network based on SMPTE ST 2110 and ST 2059 principles with:

- Fabric isolation via **VRF-FABRIC-A** and **VRF-FABRIC-B** (e.g., “RED/BLUE”).  
- **VRF-MGMT** for management/control; **OOB** via Management1 (separate plane).  
- **SSM multicast** (no RP), **OSPF per VRF**, **PTP Boundary Clock**, **MTU ≥ 9000** on routed & 2110 SVIs.  
- **VRF-leak** only **VRF-MGMT ⇄ VRF-FABRIC-{A|B}** (routes), enforced access via **SVI ACLs** on fabrics (least-privilege inbound control).  
- DHCP provided externally (e.g., firewall) with gateway = SVI; optional DHCP relay on switches (flag-controlled).

> **Out of scope at this stage:** Any firewall/pfSense rules, WAN, office/WiFi/IoT/DMZ policies, endpoint configuration, ASM/Anycast-RP, inter-fabric routing, security accreditation.

---

## 2) Assumptions
- One device per role for POC (e.g., 7060 fabric, 7020 PTP/services, 7048 OOB).  
- VRF-aware switching; EOS or equivalent features: VRF, OSPFv2 per VRF, PIM-SSM, IGMPv3, PTP BC, QoS/CoPP.  
- Gateways follow: **.254 on SVIs** owned by switches; if a firewall is present in-VLAN, it keeps **.1**, DHCP hands out **.254** as default gateway.  
- OOB network (Management1) is physically separate and never used for fabric traffic.

---

## 3) Project Parameters (placeholders)
Configure these before rendering device configs.

```yaml
FABRICS: ["FABRIC-A", "FABRIC-B"]
VRFS:
  MGMT: "VRF-MGMT"
  FABRIC_A: "VRF-FABRIC-A"
  FABRIC_B: "VRF-FABRIC-B"

VLAN_IDS:
  OOB_IPMI: 600              # optional, OOB plane
  MGMT_CTRL: 601             # in-band management plane
  FABRIC_A_2110_VIDEO: 610
  FABRIC_A_2110_AUDIO: 620
  FABRIC_A_2110_ANC:   630
  FABRIC_B_2110_VIDEO: 710
  FABRIC_B_2110_AUDIO: 720
  FABRIC_B_2110_ANC:   730
  # optional /30 “leak” VLANs if a separate transit is desired:
  LEAK_A_MGMT: 650
  LEAK_B_MGMT: 750

IP_PLAN:
  SVI_GW_SUFFIX: 254         # x.x.x.254/24 on SVIs
  P2P_MASK: 31               # /31 on routed p2p links
  LOOPBACKS:
    MGMT:    "10.<site>.112.<id>/32"
    FABRIC_A:"10.<site>.110.<id>/32"
    FABRIC_B:"10.<site>.111.<id>/32"

SSM_RANGES:
  FABRIC_A: "232.0.0.0/8"
  FABRIC_B: "234.0.0.0/8"

PTP_PROFILE:
  DOMAIN: 77
  MODE: "boundary"
  DELAY: "e2e"
  ANNOUNCE_LOG: 0            # 1/s
  SYNC_LOG: -3               # 8/s
  DELAY_REQ_LOG: -3          # 8/s
  TTL: 1
  SOURCE_IP: "VLAN_PTP_GW"   # set per-fabric PTP SVI

OSPF:
  HELLO: 1
  DEAD: 3

QoS_DSCP:
  PTP_EVENT: 56      # CS7
  PTP_GENERAL: 48    # CS6
  ROUTING_CTRL: 48   # CS6
  AUDIO_2110_30: 46  # EF
  VIDEO_2110_20: 34  # AF41
  ANC_2110_40: 26    # AF31
  MGMT: 16           # CS2
  SCAVENGER: 8       # CS1
  BEST_EFFORT: 0

FLAGS:
  DHCP_PROVIDER: "PFSENSE"   # PFSENSE | RELAY_TO_PFSENSE | NONE
  ENABLE_VRF_LEAK: true
  SVI_MTU: 9170
  CTRL_SOURCES: ["10.<site>.1.10/32"]    # controllers/NMS in VRF-MGMT
```

---

## 4) Architecture (concise)
- **VRFs:** `VRF-FABRIC-A`, `VRF-FABRIC-B`, `VRF-MGMT` (+ OOB separate).  
- **SVIs per fabric:** 2110-20/30/40 VLANs, `ip igmp version 3`, `ip pim sparse-mode`, `mtu ${SVI_MTU}`, gateway `x.x.x.${SVI_GW_SUFFIX}`.  
- **Underlay:** /31 p2p between Fabric and PTP/Services switches, OSPF p2p, timers `${OSPF.HELLO}/${OSPF.DEAD}`.  
- **Multicast:** SSM-only; ranges per VRF (`SSM_RANGES`).  
- **PTP:** Boundary Clock, domain `${PTP_PROFILE.DOMAIN}`, E2E, intervals per profile, event DSCP CS7, general DSCP CS6.  
- **VRF-leak (routes only):** `VRF-MGMT ⇄ VRF-FABRIC-{A|B}` (controllers → SVIs; loopbacks → MGMT). No `FABRIC-A ⇄ FABRIC-B`.  
- **ACLs:** Inbound on fabric SVIs to permit only `${FLAGS.CTRL_SOURCES}` and required ports (e.g., 22/443/161); default deny with log.  
- **OOB:** Management1 to OOB switch; never participate in leaks.

---

## 5) Deliverables
1) **Design Pack (HLD/LLD):** finalized parameter table (VRFs, VLANs, IP ranges, SSM), PTP profile, QoS policy, OSPF areas/timers, VRF-leak policy.  
2) **Configuration Bundle (switches only):** templated EOS stanzas rendered from this SoW.  
3) **Test Pack:** acceptance test plan plus ready-to-run commands/scripts.  
4) **Operational Pack:** As-Built, Day-2 runbook (troubleshooting PTP/multicast/OSPF/QoS, rollback steps).

---

## 6) Acceptance Criteria
**A. Isolation**  
- No reachability between **VRF-FABRIC-A** and **VRF-FABRIC-B**.  
- Only permitted MGMT → fabric flows pass; SVI ACL counters confirm.

**B. Routing & MTU**  
- OSPF adjacencies **Full** on all /31 p2p links per VRF.  
- **DF 9k** pings succeed across SVIs and routed links; no fragmentation.

**C. PTP**  
- Boundary Clock active; profile intervals match; offsets within vendor limits; no frequent GM/port role flaps during single-link failover.

**D. Multicast (SSM)**  
- (S,G) state forms for sample 2110-20/30/40 flows per fabric; IGMPv3 join ≤150 ms, leave ≤1 s (fast-leave where configured).  
- No `(*,G)` and no RP present.

**E. VRF-leak (if enabled)**  
- Only approved prefixes visible MGMT⇄fabrics (controllers to SVIs; loopbacks back).  
- No routes present FABRIC-A ⇄ FABRIC-B.

**F. OOB/MGMT**  
- OOB SSH access reachable; NTP/SNMP/syslog OK from VRF-MGMT.  
- DHCP behavior per `FLAGS.DHCP_PROVIDER` (gateway = SVI).

---

## 7) Configuration Templates (generic extracts)

### VRFs & SSM
```eos
vrf instance VRF-FABRIC-A
vrf instance VRF-FABRIC-B
vrf instance VRF-MGMT

ip pim ssm range ${SSM_RANGES.FABRIC_A} vrf VRF-FABRIC-A
ip pim ssm range ${SSM_RANGES.FABRIC_B} vrf VRF-FABRIC-B
```

### SVIs (example, Fabric-A video)
```eos
interface Vlan${VLAN_IDS.FABRIC_A_2110_VIDEO}
  vrf VRF-FABRIC-A
  ip address 10.<site>.10.${IP_PLAN.SVI_GW_SUFFIX}/24
  mtu ${FLAGS.SVI_MTU}
  ip igmp version 3
  ip pim sparse-mode
  ip access-group ACL-CTRL-FAB-A-IN in
```

### Inbound Control ACL (Fabric-A)
```eos
ip access-list ACL-CTRL-FAB-A-IN
  10 permit tcp ${FLAGS.CTRL_SOURCES[0]} 10.<site>.10.0/24 eq 443
  20 permit udp ${FLAGS.CTRL_SOURCES[0]} 10.<site>.10.0/24 eq 161
  90 deny ip any any log
```

### OSPF (per VRF, /31 p2p)
```eos
router ospf 10 vrf VRF-FABRIC-A
  router-id 10.<site>.110.<id>

interface Ethernet1
  vrf VRF-FABRIC-A
  ip address 10.<site>.101.0/${IP_PLAN.P2P_MASK}
  ip ospf network point-to-point
  ip ospf hello-interval ${OSPF.HELLO}
  ip ospf dead-interval ${OSPF.DEAD}
  ip pim sparse-mode
```

### PTP (global + access ports)
```eos
ptp mode ${PTP_PROFILE.MODE}
ptp domain ${PTP_PROFILE.DOMAIN}
ptp transport ipv4
ptp message-type ${PTP_PROFILE.DELAY}
ptp ttl ${PTP_PROFILE.TTL}
ptp announce interval ${PTP_PROFILE.ANNOUNCE_LOG}
ptp sync interval ${PTP_PROFILE.SYNC_LOG}
ptp delay-req interval ${PTP_PROFILE.DELAY_REQ_LOG}
ptp priority1 128
ptp priority2 128
ptp source ip 10.<site>.20.${IP_PLAN.SVI_GW_SUFFIX}

interface range Ethernet5-20
  ptp enable
  ptp role master
```

### QoS (DSCP to queues + optional marking)
```eos
qos dscp-map ${QoS_DSCP.PTP_EVENT}   7
qos dscp-map ${QoS_DSCP.PTP_GENERAL} 6
qos dscp-map ${QoS_DSCP.AUDIO_2110_30} 5
qos dscp-map ${QoS_DSCP.VIDEO_2110_20} 4
qos dscp-map ${QoS_DSCP.ANC_2110_40}   3
qos dscp-map ${QoS_DSCP.MGMT}          2
qos dscp-map ${QoS_DSCP.SCAVENGER}     1
qos dscp-map ${QoS_DSCP.BEST_EFFORT}   0

interface range Ethernet1-48
  priority-queue out 7
  priority-queue out 6

! optional ingress marking if endpoints don't set DSCP
ip access-list ACL-PTP-EVENT
  10 permit udp any any eq 319
ip access-list ACL-PTP-GEN
  10 permit udp any any eq 320
class-map type qos match-any CM-PTP-EVENT
  match access-group name ACL-PTP-EVENT
class-map type qos match-any CM-PTP-GEN
  match access-group name ACL-PTP-GEN
policy-map type qos PM-MARK-INGRESS
  class CM-PTP-EVENT
    set dscp ${QoS_DSCP.PTP_EVENT}
  class CM-PTP-GEN
    set dscp ${QoS_DSCP.PTP_GENERAL}
```

### VRF-leak (scoped)
```eos
ip prefix-list PL-MGMT-CTRL seq 10 permit 10.<site>.1.0/24
ip prefix-list PL-FABA-LOOP seq 10 permit 10.<site>.110.0/24
ip prefix-list PL-FABB-LOOP seq 10 permit 10.<site>.111.0/24

route-map RM-MGMT-TO-FABA permit 10
  match ip address prefix-list PL-MGMT-CTRL
route-map RM-FABA-TO-MGMT permit 10
  match ip address prefix-list PL-FABA-LOOP
route-map RM-MGMT-TO-FABB permit 10
  match ip address prefix-list PL-MGMT-CTRL
route-map RM-FABB-TO-MGMT permit 10
  match ip address prefix-list PL-FABB-LOOP

router general
  leak routes vrf VRF-MGMT into vrf VRF-FABRIC-A route-map RM-MGMT-TO-FABA
  leak routes vrf VRF-FABRIC-A into vrf VRF-MGMT route-map RM-FABA-TO-MGMT
  leak routes vrf VRF-MGMT into vrf VRF-FABRIC-B route-map RM-MGMT-TO-FABB
  leak routes vrf VRF-FABRIC-B into vrf VRF-MGMT route-map RM-FABB-TO-MGMT
```

### DHCP relay (only if `FLAGS.DHCP_PROVIDER == RELAY_TO_PFSENSE`)
```eos
interface Vlan${VLAN_IDS.MGMT_CTRL}
  vrf VRF-MGMT
  ip address 10.<site>.1.${IP_PLAN.SVI_GW_SUFFIX}/24
  ip helper-address 10.<site>.1.2 vrf VRF-MGMT
```

---

## 8) Tests & Commands

### A. MTU & Routing
```bash
# From an admin host in VRF-MGMT or on the switches:
ping -c 5 -M do -s 8972 10.<site>.10.${IP_PLAN.SVI_GW_SUFFIX}   # to Fabric-A SVI
ping -c 5 -M do -s 8972 10.<site>.101.1                         # to p2p far end
```
```
show ip ospf neighbor vrf VRF-FABRIC-A
show interfaces counters errors | nz
```

### B. PTP
```
show ptp clock
show ptp port detail
show ptp time-properties
```

### C. Multicast (SSM)
- Source a test sender to `232.x.x.x:2001` (Fabric-A) / `234.x.x.x:2002` (Fabric-B).  
- Join on a receiver with IGMPv3 include(S,G).
```
show ip mroute vrf VRF-FABRIC-A
show ip igmp groups vrf VRF-FABRIC-A
show ip pim neighbors vrf VRF-FABRIC-A
```

### D. ACL effectiveness
```
show access-lists ACL-CTRL-FAB-A-IN
show ip access-lists counters | nz
```

### E. VRF-leak scope
```
show ip route vrf VRF-FABRIC-A | include 10.<site>.1.0/24
show ip route vrf VRF-MGMT      | include 10.<site>.110.0/24
```

### F. QoS sanity
```
show qos dscp-counters | nz
show interfaces counters queue | nz
```

---

## 9) Handover
- **As-Built:** final VRF/VLAN/IP/SSM/PTP/QoS/OSPF/leak/ACL tables.  
- **Runbook:** PTP & multicast troubleshooting flow, change checkpoints, rollback.  
- **Test Report:** captured outputs for all acceptance commands and pass/fail status.

---

## Appendix — Multicast Address & Port Convention (User Mapping)

This appendix captures **your multicast mapping** so the SoW remains executable with concrete values.  
It follows the convention: **third octet = essence (20/30/40)** and **UDP port last digit = fabric (1=A, 2=B)**.

### A) Canonical per-fabric mapping
| Essence | Fabric A (RED)                  | UDP Port | Fabric B (BLUE)                 | UDP Port |
|---------|----------------------------------|----------|----------------------------------|----------|
| 2110-20 Video | `232.<site>.20.x`              | **2001**  | `234.<site>.20.x`              | **2002**  |
| 2110-30 Audio | `232.<site>.30.x`              | **3001**  | `234.<site>.30.x`              | **3002**  |
| 2110-40 Anc   | `232.<site>.40.x`              | **4001**  | `234.<site>.40.x`              | **4002**  |

> Use IGMPv3 include(S,G). SSM-only (no RP).

### B) Concrete mapping (as provided)
**Site code = 101**

- **Fabric A (RED)**  
  - 2110-20 (Video): `232.101.23.1–24` on **:2001**  
  - 2110-30 (Audio): `232.101.33.1–96` on **:3001**  
  - 2110-40 (Anc):   `232.101.43.1–24` on **:4001**  

- **Fabric B (BLUE)**  
  - 2110-20 (Video): `234.101.23.1–24` on **:2002**  
  - 2110-30 (Audio): `234.101.33.1–96` on **:3002**  
  - 2110-40 (Anc):   `234.101.43.1–24` on **:4002**  

### C) EOS SSM ranges & (optional) fabric ACL hints
```eos
! SSM ranges per VRF
ip pim ssm range 232.0.0.0/8 vrf VRF-FABRIC-A
ip pim ssm range 234.0.0.0/8 vrf VRF-FABRIC-B

! (Optional) Tight SSM group ACLs — keep only the used pools
ip access-list ACL-SSM-FAB-A-GROUPS
  10 permit ip any 232.101.23.0/24
  20 permit ip any 232.101.33.0/24
  30 permit ip any 232.101.43.0/24

ip access-list ACL-SSM-FAB-B-GROUPS
  10 permit ip any 234.101.23.0/24
  20 permit ip any 234.101.33.0/24
  30 permit ip any 234.101.43.0/24
```

### D) Test cases (add to Test Pack)
- Receiver joins:  
  - Fabric A video: **include(S=10.A.B.C, G=232.101.23.7)** on **UDP 2001**  
  - Fabric B audio: **include(S=10.D.E.F, G=234.101.33.12)** on **UDP 3002**
- Verify on switches:  
```
show ip mroute vrf VRF-FABRIC-A
show ip igmp groups vrf VRF-FABRIC-A
show ip mroute vrf VRF-FABRIC-B
show ip igmp groups vrf VRF-FABRIC-B
```

### E) Attachment rule — **4 Audio streams per Video (and ANC)**
For each **Video index `v` in 1..24**, reserve **four Audio groups**:

- **Audio group indices** for video `v`: `a ∈ {4v-3, 4v-2, 4v-1, 4v}`  
- RED (Fabric A): `G_audio = 232.101.33.a` on **UDP 3001**  
- BLUE (Fabric B): `G_audio = 234.101.33.a` on **UDP 3002**

**Examples:**  
- Video **v=1** → Audio a = **1,2,3,4** → `232.101.33.{1..4}` / `234.101.33.{1..4}`  
- Video **v=2** → Audio a = **5,6,7,8** → `232.101.33.{5..8}` / `234.101.33.{5..8}`  
- …  
- Video **v=24** → Audio a = **93,94,95,96**

**ANC attachment (1:1):**  
- ANC group index == Video index `v` →  
  RED: `232.101.43.v` on **UDP 4001**; BLUE: `234.101.43.v` on **UDP 4002**

> This yields **24 Video**, **96 Audio** (4 per Video), **24 ANC** per fabric.
