# Design & Decision Record (DDR)
**GreenField Technologies | IoT Systems Design**

**Team Members:**
1. Juan Felipe Pachon Restrepo
2. ____________________

## 1. System Overview

*   **System Type:** [X] Component (Lab 1-2) | [X] System (Lab 3-6) | [ ] Environment (Lab 7-8)
*   **Description:** SoilSense network built on ESP32-C6-DevKitC-1 nodes with an IEEE 802.15.4 radio at 2.4 GHz, organized as a Thread mesh (OpenThread). A field node S (DHT11 sensor and irrigation valve, CoAP resources `/env/temp` and `/act/valve`) and a client node C work inside the mesh, and a Border Router (ESP32-S3 host + ESP32-C6 RCP) connects the mesh to Wi-Fi for remote access.

## 2. Lab Log & Stakeholder Summaries

### Lab 1: RF Characterization
- **To Samuel (Architect):** An energy scan of channels 11–26 showed that Channel 15 has the lowest noise floor (−103 dBm). It lies in the gap between Wi-Fi channels 1 and 6, which reduces interference from farm routers. Packet loss first exceeds 1 % at about −72 dBm (B→A at 1 m: 2 %). At −80 dBm (10 m) PER reaches 6 %, and at −94/−99 dBm (20 m) the link loses 30–37 % of its packets. The maximum distance measured with PER ≤ 6 % is 10 m indoors, so the recommended spacing is no more than 10 m, with an extra margin for vegetation.
- **To Gustavo (Product / Performance Summary):**
  - Max reliable range: 10 m indoors (PER ≤ 6 % with RSSI ≥ −80 dBm); PER is 1–2 % at 1 m (RSSI −68/−72 dBm).
  - Recommended spacing: 10 m (indoor obstacles; vegetation margin still to be applied in the field).
  - Best channel: Channel 15 (noise floor −103 dBm). Avoid 802.15.4 channels 11–14, 16–19 and 21–24, which lie under Wi-Fi channels 1, 6 and 11.
  - 10-hectare field estimate: 10 ha = 100,000 m²; at 10 m spacing each node covers 100 m² → 100,000 / 100 = **1,000 nodes × $40 = $40,000**.
  - Verdict: ⚠️ Test more — the indoor range (10 m) does not reach the 20 m target, and at this spacing the field needs about 1,000 nodes. An outdoor line-of-sight test is needed before committing the budget.
- **To Edwin (Ops / Field Troubleshooting Checklist):**
  - Won't join: Load the full active dataset from the leader (`ot dataset set active <hex>`), which carries Channel 15, the PAN ID and the network key; on a board used before, run `ot factoryreset` first. Keep the PCB antenna away from metal enclosures and wet ground, with a clear line of sight free of heavy metal structures.
  - Intermittent loss: If RSSI < −70 dBm, move the node 2–3 meters closer to its neighbor; run `ot scan energy 500` to detect new local Wi-Fi interference; raise the node at least 50 cm above dense vegetation.
  - Measured range guidelines: Line-of-sight (unobstructed): 10–15 m; Light obstacles / Indoor: 10 m; Heavy walls: < 5 m (sharp signal drop).

### Lab 2: 6LoWPAN Mesh
- **To Samuel (Architect):** Each extra hop costs about 14.7 ms (15.85 ms for 1 hop vs 30.55 ms for 2 hops, 0 % loss). Application firmware must use the Mesh-Local EID (ML-EID) for all traffic. The RLOC encodes where a node is attached and changes when the topology changes, while the ML-EID is random and stable, so the node stays reachable at the same address.
- **To Edwin (Ops):** The network healed in 8.33 seconds on average (runs of 9, 8 and 8 s) after the relay router failed, well within the 2-minute requirement. If a field node goes dark, technicians should check the physical distance and line of sight to its neighbors, and read `ot router table` and `ot neighbor table` on nearby nodes to confirm that the routes were recalculated.

### Lab 3: Thread & CoAP
- **To Daniela (Customer):** Each reading now costs a single small radio message (15–19 bytes) instead of a full HTTP exchange. The valve keeps its radio asleep and checks for commands every 5 seconds, which gives an estimated battery life of about 1.56 years on 2× AA batteries. The soil temperature dashboard is updated automatically only when the value changes by more than 0.5 °C (Observe), so it shows current data while sending 90.7 % fewer messages than polling once per second.
- **To Edwin (Ops):**
Valve commands use confirmable messages (CON): an immediate 2.04 Changed response proves that the command was received and executed.
If a valve is offline, the client retransmits up to 4 times with doubling waits; the last retransmission leaves 30–45 seconds after the first attempt, and the command fails loudly with a ResponseTimeout error after 62–93 seconds (93.0 s measured).
Field technicians should expect an average control response time of 2.4 seconds (maximum 4.4 seconds), set by the valve's 5-second sleep cycle.

### Lab 4: Sensors & Control
- **To Edwin (Ops):**
  Field technicians can tell a dead sensor from a dead node by the CoAP response. A dead sensor leaves the node online: it keeps answering and, 60 seconds after the last good reading, returns an explicit 5.03 Service Unavailable error. A dead node causes complete radio silence: the 45-second Observe heartbeats stop and a request ends in a CoAP ResponseTimeout (42.4 s measured). In short, if the node answers 5.03, inspect the sensor wiring; if the node is silent or times out, check the node's power and mesh connectivity.

### Lab 5: Border Router
- **To Daniela (Customer):**
  You can now see the soil temperature and control the valve from your phone through the web dashboard, with a response time of about 101 ms over Wi-Fi. If the new Border Router box fails or loses power, remote access from your phone stops, but the field nodes keep working locally: the valve still obeys commands sent from inside the mesh.

### Lab 6: Security
*   **To Edward (Security):**

### Lab 7: Dashboard
*   **To Gustavo (Product):**

### Lab 8: Final Integration
*   **To All:**

## 3. Architecture Decision Records (ADRs)

**ADR-001: Selection of 802.15.4 Radio Channel**
- **Context:** The 2.4 GHz ISM band is shared by IEEE 802.15.4 and high-power IEEE 802.11 (Wi-Fi) networks. Co-channel interference causes frame collisions, forces MAC retransmissions and shortens battery life.
- **Decision:** Operating channel set to **Channel 15** (2425 MHz).
- **Rationale:** The energy scan measured the lowest noise floor on Channel 15 (−103 dBm). 802.15.4 channels 11–14, 16–19 and 21–24 lie under Wi-Fi channels 1, 6 and 11, while Channel 15 sits in the gap between Wi-Fi channels 1 and 6. This keeps the low-power mesh traffic away from the farm routers.
- **Status:** [X] Accepted

**ADR-002: Selection of Thread Mesh Topology over LoRaWAN Star Topology**
- **Context:** Irrigation valve control needs low-latency two-way communication and high network availability. In a star topology (LoRaWAN) every node depends on a direct link to a gateway and has no relay path, so a failed link isolates the node.
- **Decision:** Adopt an OpenThread 6LoWPAN mesh topology.
- **Rationale:** Measurements showed a round-trip time of 15.85 ms for 1 hop and 30.55 ms for 2 hops, which is suitable for valve control. In the failure test ("Tractor Test") the mesh routed around the dead router and healed in **8.33 seconds** on average, well below the 120 s limit.
- **Status:** [X] Accepted

**ADR-003: Selection of CoAP + CBOR over MQTT/JSON for Mesh Application Layer**
- **Context:** Constrained 802.15.4 mesh networks cannot afford the large headers, packet count and TCP connection upkeep required by HTTP or MQTT.
- **Decision:** Adopt CoAP (RFC 7252) over UDP with CBOR (RFC 8949) binary payloads and RFC 7641 Observe.
- **Rationale:** CoAP has a 4-byte fixed header, and the measured messages are 13–19 bytes (GET 15/17 B, Observe notification 19 B, PUT 18/13 B). Each one fits easily in a single 127-byte 802.15.4 frame without 6LoWPAN fragmentation. MQTT was rejected for the radio side because it needs a persistent TCP connection with periodic keepalives that keep the radio awake. The Observe pattern reduced telemetry traffic by 90.7 % (28 notifications vs 300 polls over 5 minutes) compared with 1 Hz polling.
- **Status:** [X] Accepted

**ADR-004: Selection of 5-Second Poll Period for Battery-Powered Irrigation Valves**
- **Context:** Battery-powered field valves must turn off their receivers to save power, but must remain responsive enough for irrigation commands.
- **Decision:** Set the Sleepy End Device (SED) polling interval (`pollperiod`) to 5000 ms (5 s).
- **Rationale:** Tests measured an average downlink ping RTT of 2,443.6 ms (maximum 4,444 ms), which is short compared with irrigation timescales. Polling every 5 seconds reduces the radio-on duty cycle to 0.2 %, which gives an estimated 2× AA battery life of 1.56 years with the datasheet active RX current ($74\text{ mA}$) and light-sleep current ($35\text{ }\mu\text{A}$).
- **Status:** [X] Accepted

**ADR-005: Selection of a Single OpenThread Border Router (OTBR) for Pilot Deployment**
- **Context:** Connecting the isolated 802.15.4 Thread mesh to the farm's Wi-Fi network is required for remote mobile monitoring, but gateway hardware adds cost, latency and a possible single point of failure.
- **Decision:** Deploy a single dual-chip OTBR (ESP32-S3 host + ESP32-C6 RCP) as an SCD-hosted IoT gateway.
- **Rationale:** The OTBR adds 94 ms of routing latency (101 ms median CoAP GET from the laptop), which meets the < 200 ms target. With the OTBR disconnected, in-mesh valve control from Node C to Node S kept working (2.04 Changed). The single point of failure therefore affects only remote access, not local field operation.
- **Status:** [X] Accepted

**ADR-006: PSK or Certificates**
- **Context:**
- **Decision:**
- **Rationale:**
- **Status:** [ ] Proposed | [ ] Accepted | [ ] Deprecated

**ADR-007: Telemetry and the Broker**
- **Context:**
- **Decision:**
- **Rationale:**
- **Status:** [ ] Proposed | [ ] Accepted | [ ] Deprecated

**ADR-008: Hardening the Golden Master**
- **Context:**
- **Decision:**
- **Rationale:**
- **Status:** [ ] Proposed | [ ] Accepted | [ ] Deprecated

## 4. ISO/IEC 30141 Mapping

### Domain Mapping

| Component | ISO Domain | Justification |
|-----------|------------|---------------|
| ESP32-C6 SoC | SCD | Sensing/controlling device (§6.4–6.5) |
| 802.15.4 radio + antenna | SCD | Communication subsystem; the antenna is the boundary with the PED |
| Thread mesh (routers, MLE) | SCD | Communication subsystem of the sensing domain (proximity network) |
| UDP / IPv6 transport and SED parent poll | SCD | Device communication, not application logic |
| Physical Sensor Module (DHT11) | SCD | Hardware transducer acquiring environmental readings |
| Plausibility checks (range and step) | SCD | Sensing-side validation of each reading |
| `/env/temp` and `/act/valve` CoAP resources (contracts, Observe, CON) | ASD | Services consumed by the rest of the system through stable ML-EID addresses |
| ESP32-S3 + C6 RCP (OTBR) | SCD | SCD-hosted IoT gateway (§A.4 / Figure A.5) |
| Air (RF medium) | PED | Physical entity — EM propagation |
| Wi-Fi interference | PED | Physical phenomenon that adds noise at the antenna |
| Air / Soil Temperature & Humidity | PED | Physical entity being observed — Ambient thermal and moisture state |

### Component Capabilities

| Capability Category | Subcategory | Component/Feature | Active/Latent | Lab Introduced |
|---------------------|-------------|-------------------|---------------|----------------|
| Transducer | Actuation | On-board LED | Active (shows the node state) | Lab 1 |
| Data | Processing & Storage | RSSI filtering / NVS storage / 802.15.4 TX-RX | Active (used to form the link and run every measurement) | Lab 1 |
| Interface | Network & Serial | 802.15.4 network / OpenThread CLI / Serial monitor | Active (used to control and read the boards) | Lab 1 |
| Supporting | Security & Time | Time sync / Hardware crypto accelerator | Latent (not used by the application firmware) | Lab 1 |
| Latent | Wireless & Debug | BLE radio / WiFi radio / USB (debug interface) | Latent (not started by the lab firmware) | Lab 1 |
| Transducer | Actuation | Irrigation valve (LED white while open) | Active (driven by `PUT /act/valve`) | Lab 3 |
| Interface | Application | CoAP server (`/env/temp`, `/act/valve`) | Active (serves the API contracts) | Lab 3 |
| Transducer | Sensing | DHT11 temperature and humidity sensor | Active (read every 2 s) | Lab 4 |
| Gateway | Protocol & Routing | OTBR dual-chip gateway (Wi-Fi ↔ Thread, NAT64) | Active | Lab 5 |

### Thread Address Classification

| Type | Pattern | Scope | Used for | Lab Introduced |
| :--- | :--- | :--- | :--- | :---: |
| Link-local | `fe80::/64` | 1 radio hop | Neighbor discovery & MLE signaling between adjacent nodes. | Lab 2 |
| RLOC | `<prefix>:0:ff:fe00:<RLOC16>` | Entire mesh | Packet routing based on the node's current attachment point. | Lab 2 |
| ALOC | `<prefix>:0:ff:fe00:fcXX` | Entire mesh | Anycast role identification (`fc00` identifies the Leader). | Lab 2 |
| ML-EID | `<prefix>:<random IID>` | Entire mesh | Stable application-layer device identification. | Lab 2 |
| OMR | `fd02:8e98:55d3:1:<SLAAC IID>` | Entire mesh / Wi-Fi | Routable address advertised by the OTBR for off-mesh reachability. | Lab 5 |

### Functional Viewpoint — Thread Roles & Power Profiles

| Thread Role | Functional Role | Power Profile | Impact on 3-Month Battery Target | Lab Introduced |
| :--- | :--- | :--- | :--- | :---: |
| Leader | Network Coordinator | Always-On (100% RX, ~74 mA) | Cannot run on batteries long-term; requires mains power. | Lab 2 |
| Router | Packet Forwarder | Always-On (100% RX, ~74 mA) | Cannot run on batteries long-term; requires mains or solar power. | Lab 2 |
| REED | Standby Forwarder | Always-On (100% RX, ~74 mA) | Works as a leaf until promoted; continuous RX drains the battery quickly. | Lab 2 |
| End Device (FED) | Leaf Node | Always-On (100% RX, ~74 mA) | Keeps the receiver active; cannot meet 3-month battery life. | Lab 2 |
| Sleepy End Device (SED)| Leaf Sensor Node | Duty-Cycled (0.2% RX at 5 s poll, 35 µA sleep) | Only role that meets the 3-month target by sleeping between polls. | Lab 2 |
| Border Router (OTBR) | Gateway / Leader | Always-On (Mains-Powered) | Bridges Wi-Fi and Thread; requires an external power supply. | Lab 5 |

### Application & Service Domain (ASD) — API Contracts

#### `/env/temp` — Air Temperature Interface
| Parameter | Contract Specification |
|---|---|
| Methods | `GET`, `GET` + Observe (RFC 7641) |
| Content-Format | `60` (`application/cbor`) |
| Payload | `{"t": float16}`, °C, 6 bytes (`A1 61 74 F9 hh ll`) |
| Accuracy | $\pm 2.0^\circ\text{C}$ (DHT11 datasheet rating) |
| Max-Age | $60\text{ s} - \text{reading age}$ (counts down to 0) |
| Plausibility | Rejects $T < -40^\circ\text{C}$, $T \ge 85^\circ\text{C}$, or $\Delta T > 5.0^\circ\text{C}$ from the last accepted reading when that reading is less than $10\text{ s}$ old |
| Error Response | `5.03 Service Unavailable` with `Max-Age: 10` when reading age $> 60\text{ s}$ |
| Observe Policy | `NON` push when $\Delta T > 0.5^\circ\text{C}$; `CON` heartbeat after 45 s without change; a single `5.03` ends the observation on expiration |

#### `/act/valve` — Irrigation Valve Interface
| Parameter | Contract Specification |
|---|---|
| Methods | `PUT` (set state), `GET` (read state) |
| Request Payload | CBOR `{"v": 0 \| 1}` (Content-Format 60) or text `"0"` / `"1"` |
| Response Payload| CBOR `{"v": 0 \| 1}` (Content-Format 60) |
| Response Codes | `2.04 Changed` (PUT success), `2.05 Content` (GET), `4.00 Bad Request` |
| Idempotency | Yes: duplicate `PUT` requests leave the same state |

```cddl
env-reading = { t: float16 }        ; °C
valve-state = { v: 0 / 1 }          ; 0 = closed, 1 = open
```
### ISO/IEC 30141 §A.4 — IoT Gateway Scorecard (Enterprise System Pattern, Table A.3)

> *"IoT gateways are devices which connect SCD with other domains. IoT gateways provide functions such as protocol conversion, address mapping, data processing, information fusion, certification, and equipment management."*

| Function | Provided by OTBR? | Empirical Evidence / Rationale |
|---|---|---|
| Protocol conversion | Yes | Converts 802.15.4 / 6LoWPAN mesh datagrams into Wi-Fi 802.11 IPv6/IPv4 packets. |
| Address mapping | Yes | Performs NAT64 translation (`fd02:8e98:55d3:2:0:0::/96` mapped to IPv4). |
| Data processing | No | Forwards CoAP/UDP packets without changing the application payload. |
| Information fusion | No | Does not combine several sensor measurements into one record. |
| Certification | No | Unauthenticated shell access over USB and open CoAP endpoints ($\rightarrow$ Lab 6). |
| Equipment management | Partial | Advertises the OMR prefix in Router Advertisements, manages the Thread Network Data and acts as Leader; it does not configure or update the nodes remotely. |

### ISO/IEC 30141 §A.5 — Enterprise Networking Pattern (Table A.4 Four-Network Walk)

| Network | System Component in SoilSense | Concrete IP Address / Range Assigned |
|---|---|---|
| Proximity | Thread Mesh Network (Node S, Node C, OTBR Radio) | ML-EID `fdab:6a0d:311a:e59c::/64` & OMR `fd02:8e98:55d3:1::/64` |
| Access | OTBR Wi-Fi Interface & Mobile Hotspot ("Juan") | OTBR IPv4 `10.101.156.84` & Link-Local `fe80::4a27:e2ff:fe2d:9a3c` |
| Services | Flask CoAP Dashboard & Upstream IPv4 Targets | Laptop dashboard on TCP port 5000 (hotspot network `10.101.156.x`) & NAT64 IPv4 target `8.8.8.8` |
| User | Daniela's Mobile Phone Browser | IP assigned dynamically by the hotspot DHCP |

## 5. First Principles Reflections

**Lab 1:**
1. **Why does RSSI drop with distance, and by how much per doubling?**  
   The antenna radiates energy in all directions, so the same power spreads over a sphere whose surface grows with the square of the distance. The receiving antenna has a fixed size, so it captures a smaller share of that power as the distance grows. In free space the received power falls as $1/d^2$: each time the distance doubles, the signal drops by 6 dB. In our indoor test the drop between 10 m and 20 m was 16 dB (A→B) and 19 dB (B→A), much more than 6 dB, because walls and obstacles add losses on top of the distance.

2. **The receiver decodes signals near −100 dBm. Why did you need a much stronger signal for PER < 1 %? Where do the missing decibels go?**  
   The −100 dBm sensitivity is the weakest signal the radio can decode under ideal test conditions; it does not mean that every packet arrives. Our measured noise floor on Channel 15 was −103 dBm, so a −100 dBm signal is only 3 dB above the noise. A real link also needs margin for fading caused by reflections (multipath), for people and objects in the path, and for Wi-Fi interference. These effects take the "missing" decibels: in our test PER already exceeded 1 % at −72 dBm and reached 30–37 % at −94/−99 dBm.

**Lab 2:**
1. **How does a 40-byte IPv6 header fit in a 127-byte frame? Which interface ID can the receiver rebuild from the MAC header, the RLOC's or the ML-EID's?**  
   6LoWPAN uses IPHC header compression, which removes every field the receiver can predict: the version, the traffic class and flow label, the payload length (taken from the frame), the next header (UDP, which is also compressed) and the hop limit. The address prefix is removed when it is link-local or the mesh-local prefix shared as context 0. The interface ID is removed when the receiver can rebuild it from the 802.15.4 header: the RLOC's IID (`0:ff:fe00:RLOC16`) comes from the 16-bit short address, and the link-local IID comes from the extended MAC address. The ML-EID's IID is random, so it cannot be rebuilt and is sent inline (8 bytes). In the best case the 40-byte header is reduced to about 2 bytes.

2. **Why did healing take seconds and not minutes? What told A that B was gone, and what would happen on a link that carried no traffic?**  
   A was sending one ping per second through B. When B was unplugged, those frames received no MAC acknowledgement. After 4 consecutive unacknowledged frames (each already retried by the MAC), A removed the link at once and switched to the direct A–C path, which took about 8 seconds. On a link with no traffic, only the MLE timer would detect the failure: a router that sends no advertisement for 100 seconds is removed.

3. **Why does Task E need a *poor* A–C link instead of *no* link? What does that mean for node spacing in Daniela's field?**  
   With the poor link ($LQ = 1$, cost 4), the path through B ($1 + 1 = 2$) is cheaper, so traffic goes through B while B works. Because A and C can still hear each other, the poor link stays in their tables and becomes the backup route when B dies. With no link at all, C would become unreachable. In Daniela's field, nodes must be placed so that each one stays at least within weak range of a second neighbor.

**Lab 3:**
1. **Why can CoAP use UDP where HTTP needs TCP? What does CON give back, and what does NON deliberately give up?**
   HTTP relies on TCP for reliability, which needs a connection setup, connection state and keepalives. CoAP uses UDP and adds reliability only to the messages that need it. CON gives back reliability: the receiver must send an ACK, and the sender retransmits with doubling waits until it gets one or gives up after 62–93 s. NON deliberately gives up the ACK and the retransmissions to save radio time; a lost NON message is not noticed, which is acceptable for routine readings that the next one replaces.

2. **Why does Observe save battery compared with a GET every 60 s? Who knows when the value changed, and who pays for the radio?**
   With a GET every 60 s, the client must wake up and ask, and the server must answer, even when the data has not changed; both pay radio time on every poll. Only the server knows when the value changed. With Observe, the client registers once and the server sends a notification only when the temperature changes by more than $0.5^\circ\text{C}$, or a heartbeat after 45 s without change. In our test this sent 28 messages instead of 300 polls, 90.7 % fewer.

3. **Why did the sleepy valve receive the same PUT twice, and why was that harmless? What would have happened with a "toggle valve" command?**
   CoAP retransmits a CON request after 2–3 s when no ACK arrives, but the valve sleeps for 5 s. The parent stores both copies and delivers them together when the valve wakes up. It was harmless because `PUT` sets an explicit state (`OPEN`), so applying it twice leaves the same result. A non-idempotent `toggle` command would have opened the valve and then closed it again.

4. **Why CBOR and not a hand-made binary struct?**
   A hand-made C struct breaks when fields are added or reordered, or when memory alignment and byte order differ between platforms. CBOR keeps JSON's self-describing key-value model in a standard binary format, so the reading fits in 6 bytes and both sides can validate it against the CDDL contract.

**Lab 4:**
1. **Why does `/env/temp` answer 5.03 instead of the last value it has? What would a stale value presented as current do to Daniela's decisions?**  
   A stale reading served as current misleads the application and the user. Daniela could see a normal temperature during a heat event, or an automatic rule could keep the valves closed when the crop needs water. Returning `5.03 Service Unavailable` tells the client clearly that no fresh data exists, instead of giving an old number after its Max-Age has expired.

2. **How does a client tell "sensor dead" from "node unreachable"? Which one needs an answer from the node, and which shows up only as silence?**  
   A dead sensor needs an answer from the node: the node stays online and replies with a `5.03` error 60 seconds after the last good reading. An unreachable node shows up only as silence: notifications and 45-second heartbeats stop, and a request ends in a CoAP `ResponseTimeout` (42.4 seconds measured).

3. **Why can't a plausibility check stop a value that is wrong but believable?**  
   Local checks only test whether a reading is inside the physical range and whether it jumps too fast from the previous reading. A value that is wrong but believable (25.5 °C when the real temperature is 21.0 °C) passes both rules. Detecting it requires information from outside the node, such as a second sensor or, if an attacker sends it, authentication of the source.

4. **Why can a busy radio make a sensor read fail? Compare a sensor whose protocol the CPU times bit by bit with one that has its own bus hardware.**  
   The DHT11 uses a single-wire protocol in which the CPU measures the length of each pulse in microseconds during a reading of about 22 ms. Radio interrupts pause the CPU inside that window, so pulse lengths are measured wrongly and the checksum fails (19.4 % failures under ping flood vs 0 % with a quiet radio). A sensor with its own bus hardware, such as the BMP280 on I²C, is timed by the I²C peripheral, so radio interrupts do not corrupt its data.

**Lab 5:**
1. **Why does a prefix advertisement plus SLAAC make S reachable from outside, with no gateway "magic"?**  
   The OTBR advertises the OMR prefix (`fd02:8e98:55d3:1::/64`) to the Thread mesh in the Network Data, and Node S builds its own address in that prefix with SLAAC (RFC 4862). On the Wi-Fi side, the OTBR's Router Advertisements announce a route to the same prefix, so the laptop learns where to send packets. The OTBR then works only as a standard IPv6 router, without any application-layer proxy or translation. Because the hotspot does not delegate a prefix, the OMR prefix is a ULA: it is routable within the site, not on the global Internet.

2. **Why is NAT64 needed when Thread is IPv6?**  
   Thread nodes use only IPv6, but many Internet services still use IPv4 addresses only. NAT64 writes the 32-bit IPv4 address at the end of a `/96` prefix (8.8.8.8 → `fd02:8e98:55d3:2:0:0:808:808`), and the OTBR translates between that IPv6 address and IPv4, so IPv6-only mesh nodes can reach IPv4 servers.

3. **Why are the bytes of `/env/temp` the same on both sides of the OTBR?**  
   The OTBR works at the network layer (Layer 3 IPv6 routing), not at the application layer. Only the network and link headers change (compressed IPv6 in 802.15.4 frames inside the mesh, full IPv6 in 802.11 frames on Wi-Fi); the UDP payload with the CoAP message and its CBOR bytes is forwarded unchanged.

4. **Why should a mains-powered gateway turn Wi-Fi power save off, when a battery node would do the opposite?**  
   With power save on, the Wi-Fi receiver sleeps between beacons, and packets wait until it wakes up; we measured 267 ms with power save on against 15 ms with it off. A mains-powered gateway does not need to save energy, so it should favor low latency. A battery-powered node must favor battery life, so it accepts the extra latency.

**Lab 6:**
1. **Why DTLS and not TLS for CoAP?**

2. **Why does one AEAD tag give both confidentiality and integrity?**

3. **Why is authentication, not encryption, what stops the forged OPEN?**

4. **Why does a wrong key produce silence instead of an error message?**

**Lab 7:**
1. **Why isn't telemetry business data? Audience, cadence, cost.**

2. **Why a broker instead of every phone reaching every node over IPv6?**

3. **Why is silence the hardest signal to read?**

**Lab 8:**
1. **Why does the valve fail closed?**

2. **Why is availability a business property?**

3. **Why is "no new features" a feature in the last week?**

## 6. Performance Baselines

| Metric | Target | Measured | Status |
|--------|--------|----------|--------|
| Lab 1: Max Range | > 20m | 10 m (RSSI ≥ −80 dBm, PER ≤ 6 %) | [ ] Fail |
| Lab 2: Healing Time | < 120s | 8.33 s | [X] Pass |
| Lab 3: CoAP 1-Hop GET RTT | < 200ms | 15.85 ms | [X] Pass |
| Lab 3: CON Max Retry Span | < 120s | 93.0 s (Error 28: ResponseTimeout) | [X] Pass |
| Lab 3: SED Latency (1s poll)| < 2s | 544.1 ms avg (min 148 ms / max 1,061 ms) | [X] Pass |
| Lab 3: SED Latency (5s poll)| < 5s | 2,443.6 ms avg (min 443 ms / max 4,444 ms) | [X] Pass |
| Lab 3: SED Latency (15s poll)| < 20s | 6,850.1 ms avg (min 479 ms / max 14,479 ms) | [X] Pass |
| Lab 4: Lost Sensor Detection | ≤ 60 s | 60.0 s (5.03 response) | [X] Pass |
| Lab 4: Busy Radio Failure Rate | < 5% | 19.4% (13/67 failed reads under ping flood) | [ ] Fail |
| Lab 5: OTBR added latency | < 100 ms | 94 ms ((Laptop → S) - (Laptop → OTBR)) | [X] Pass |
| Lab 5: NAT64 RTT | < 200 ms | ___ ms | [ ] Pass |
| Lab 6: DTLS Time | < 3s | ___ s | [ ] Pass |
| Lab 6: DTLS per-record overhead | — | ___ B | — |
| Lab 7: Silent node to red | < 3 min | ___ s | [ ] Pass |
| Lab 7: Report period | — | ___ s | — |
| Lab 8: Cold start (all nodes green) | < 120 s | ___ s | [ ] Pass |
| Lab 8: Blackout recovery | < 120 s | ___ s | [ ] Pass |
| Lab 8: e2e stress (GETs served / avg latency) | ≥ 80 % | ___ % / ___ ms | [ ] Pass |
| Lab 8: Soak (hours / unplanned reboots) | ≥ 12 h / 0 | ___ h / ___ | [ ] Pass |


## 7. Ethics & Sustainability Checklist

*   [X] **Lab 1:** Only passive energy scanning was used (`ot scan energy` listens and does not transmit), so the measurements did not disrupt neighboring networks.
*   [X] **Lab 2:** Own network only: the mesh dataset was created with `ot dataset init new`, which generates a random network key.
* [X] **Lab 3 Safety & Sustainability Design (Failsafe Mechanism):** 
  To prevent crop damage, soil erosion and water waste if the network fails after an `OPEN` (`v: 1`) command, the valve firmware must include a local safety timer (15-minute cutoff). If no network heartbeat or explicit `CLOSE` (`v: 0`) command arrives within 15 minutes, the valve closes on its own (`v: 0`) without needing network connectivity.
*   [X] **Lab 4 (Safety Design for Missing Data):** If an automatic irrigation rule receives a `5.03` error or no CoAP response, it must not act on the last known value. If fresh data does not return within a 5-minute timeout, the rule sets the valve to the safe state (`CLOSE`) to prevent flooding or water waste.
*   [X] **Lab 5 (Local Autonomy & Sustainability):** Verified that if the OTBR fails or loses power, in-mesh CoAP control between field nodes (C → S) keeps working locally without cloud or gateway dependency (`PUT /act/valve` answered `2.04 Changed`).
*   [ ] **Lab 5 Security Gap (Unauthenticated Shell):** Anyone with physical access to the OTBR's USB cable can issue `ot` shell commands without authentication.
*   [ ] **Lab 5 Privacy Gap (Open Wi-Fi Access):** Any device connected to the Wi-Fi hotspot can read sensor values or operate the valve without encryption or proof of identity ($\rightarrow$ Lab 6).
*   [ ] **Lab 6:** Encryption enabled (Privacy).
*   [ ] **Lab 7:** Data minimization and data destination documented.
*   [ ] **Lab 8:** End-of-Life plan considered.

## 8. Viewpoint Analysis

| Viewpoint | Labs Addressed | Key Concerns Documented |
|-----------|----------------|-------------------------|
| Foundational | Lab 1-2 | IoT component: ESP32-C6 with 802.15.4 radio and antenna (SCD); antenna as the PED/SCD boundary; active and latent component capabilities. |
| Business |               |                         |
| Usage |                  |                         |
| Functional | Lab 2-3 | Thread roles and power profiles; mesh as the SCD communication subsystem; CoAP API contracts in the ASD with UDP/IPv6/SED poll in the SCD. |
| Trustworthiness | Lab 4 | Data freshness (Max-Age), sensor failure signatures, radio load impact, and self-healing limits. |
| Construction | Lab 5 | Dual-chip OTBR architecture (ESP32-S3 host + ESP32-C6 RCP); ISO/IEC 30141 Annex A enterprise system pattern (Table A.3 / Figure A.5) and enterprise networking pattern (Table A.4 / Figure A.6). |

## 9. Trustworthiness Audit (Lab 6+)

| Characteristic | Addressed? | How | Gaps |
|----------------|------------|-----|------|
| Availability | Yes (Lab 4) | Max-Age freshness tracking; explicit `5.03 Service Unavailable` on stale data (> 60 s); `ResponseTimeout` (42.4 s) when the node is unreachable. | A server reboot or a 5.03 error clears observer subscriptions, so the client must register again. |
| Confidentiality | No | → Lab 6 | Unencrypted UDP payloads can be read by anyone on the access network. |
| Integrity | Yes (Lab 4) | Range (−40 to 85 °C) and step (5 °C maximum jump) checks reject sensor faults and impossible values. | Cannot reject false but believable readings (e.g., injected 25.5 °C vs 21.0 °C) without multi-sensor fusion or authentication (→ Lab 6). |
| Reliability | Yes (Lab 4) | Sensor read every 2 s with a 0.0 % failure rate under quiet radio conditions. | 19.4 % failure rate under busy radio traffic (ping flood) exceeds the < 5 % target because radio interrupts disturb the CPU-timed single-wire protocol. |
| Resilience | Yes (Lab 4) | Recovers by itself from transient read errors and from wire reconnection (`Sensor back`). | Does not restore Observe subscriptions after a 5.03 or a node reboot; after a reboot, readings and counters start again from zero. |
| Safety | Yes (Labs 3–4) | On a 5.03 or no response, the irrigation rule does not use the last value and sets the valve to `CLOSE` after a 5-minute timeout; the valve must close on its own 15 minutes after the last command or heartbeat. | The local valve timer is a design requirement and is not yet part of the lab firmware. |
| Compliance |              |     |      |

## 10. Construction Viewpoint - IoT System Pattern (Lab 8)

| Pattern Element | Category | Your System |
|-----------------|----------|-------------|
| IoT System |            |             |
| IoT Components |         |             |
| Digital Network |        |             |
| IoT Devices |            |             |
| Primary Capability (observation) | |   |
| Primary Capability (control) | |       |
| Secondary Capability (processing) | |  |
| Secondary Capability (transferring) | | |
| Secondary Capability (storage) | |     |
| Interface (network) |    |             |
| Interface (human UI) |   |             |
| Interface (application) | |            |
| Supplemental (security) | |            |
| Supplemental (orchestration) | |       |
| Supplemental (management) | |          |

## 11. Empirical Measurement Tables

### Lab 1: Range and Performance Baseline Data

| Distance (m) | RSSI A→B (dBm) | RSSI B→A (dBm) | PER A→B (%) | PER B→A (%) |
|:------------:|:--------------:|:--------------:|:-----------:|:-----------:|
| 1            |  -68           |     -72        |   1         |      2      |
| 5            |  ≈ -75         |     ≈ -78      |   ≈ 3       |      ≈ 5    |
| 10           |   -78          |    -80         |    4        |        6    |
| 20           |     -94        |     -99        |     30      |        37   |

> **Testing note:** Measurements were taken indoors at a fixed TX power of 0 dBm. With a clear line of sight the signal remained strong; walls or obstacles between the boards caused a sharp drop in RSSI and more packet loss. The 5 m values were calculated by logarithmic interpolation between the 1 m and 10 m measurements. The test ends at 20 m, where PER is already above 10 %.

### Lab 2: 6LoWPAN Mesh & Resilience Baseline Data

#### Multi-hop Latency Baseline (Task 3.2)

| Path | Hops | RTT Min (ms) | RTT Avg (ms) | RTT Max (ms) | Packet Loss (%) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| A → B | 1 | 15.00 | 15.85 | 31.00 | 0.0% |
| A → C (via B) | 2 | 30.00 | 30.55 | 37.00 | 0.0% |

#### Tractor Test Convergence Baseline 

| Run | Lost Pings | Healing Time (s) | Packet Loss (%) | Post-Healing Route |
| :---: | :---: | :---: | :---: | :--- |
| 1 | 9 | 9.0 | 4.5% | Direct A → C (LQI 1, Next Hop 63) |
| 2 | 8 | 8.0 | 4.0% | Direct A → C (LQI 1, Next Hop 63) |
| 3 | 8 | 8.0 | 4.0% | Direct A → C (LQI 1, Next Hop 63) |

### Lab 3: CoAP Application Layer & Energy Baselines

#### SED Latency vs Poll Period

| Poll Period | RTT Min / Avg / Max (ms) | Radio-on Fraction for Polling |
| :---: | :---: | :---: |
| 1 s | 148 / 544.1 / 1,061 | 1 % |
| 5 s | 443 / 2,443.6 / 4,444 | 0.2 % |
| 15 s | 479 / 6,850.1 / 14,479 | 0.067 % |

### Sleepy Valve Energy & Battery Life Estimate (5s Poll Period)

- **ESP32-C6 Datasheet Parameters (Table 5-9 & Table 5-11):**
  - Active 802.15.4 RX Current ($I_{rx}$): $74.0\text{ mA}$ (Table 5-9)
  - Light Sleep Current ($I_{sleep}$): $35.0\text{ }\mu\text{A}$ (Table 5-11)
  - Poll Active Duration ($t_{on}$): $10\text{ ms}$
  - Poll Period ($T_{poll}$): $5000\text{ ms}$

- **Duty Cycle & Average Current Calculation:**
  $$\text{Duty Cycle } (D) = \frac{t_{on}}{T_{poll}} = \frac{10\text{ ms}}{5000\text{ ms}} = 0.002 \quad (0.2\%)$$

  $$I_{avg} = (D \times I_{rx}) + ((1 - D) \times I_{sleep})$$
  $$I_{avg} = (0.002 \times 74.0\text{ mA}) + (0.998 \times 0.035\text{ mA}) = 0.148\text{ mA} + 0.0349\text{ mA} = 0.1829\text{ mA} \quad (182.9\text{ }\mu\text{A})$$

- **Battery Lifespan (Standard 2× AA Alkaline Batteries @ 2500 mAh):**
  $$\text{Battery Life (hours)} = \frac{2500\text{ mAh}}{0.1829\text{ mA}} \approx 13,668.6\text{ hours} \approx \mathbf{1.56\text{ years}}$$

#### Empirical Byte Budget Comparison

| Exchange | Request Payload | Response Payload | Total Packets | Notes |
|---|---|---|---|---|
| HTTP/JSON reading (Lab 0) | ~150 B + TCP | ~120 B + TCP | 7+ | Handshake, GET, 200 OK, teardown |
| MQTT/JSON reading (Lab 0) | — | ~40 B + TCP | 1 | Persistent TCP connection |
| CoAP GET `/env/temp` | 15 B | 17 B | 2 | 1 UDP request, 1 UDP response |
| CoAP Observe notification | — | 19 B | 1 | Single NON push datagram |
| CoAP PUT `/act/valve` (CON) | 18 B | 13 B | 2 | Piggybacked ACK response |

### Lab 4: Real Sensor Characterization & Reliability Baseline Data

#### DHT11 Sensor Characterization
| Metric | Datasheet Rating | Measured Value |
|---|---|---|
| Resolution | 1.0 °C | 1.0 °C |
| Noise Spread (1 min static) | N/A | 0.0 °C |
| Accuracy | ±2.0 °C | — |
| Read Execution Latency | N/A | 21,811 µs avg (18,115 µs min / 22,113 µs max) |

#### Failure Mode & Radio Impact Baseline

| Test Condition | Measured Value | Failure Signature |
|---|---|---|
| Sensor Unplugged Detection | 60.0 s | Explicit `5.03 Service Unavailable` (`no fresh reading`) |
| Node Off Network Detection | 42.40 s | `coap receive response error 28: ResponseTimeout` |
| Quiet Radio Failure Rate (2 min) | 0.0% (0 / 76 failed) | Clean timing slots |
| Busy Radio Failure Rate (10 pings/s, 2 min) | 19.4% (13 / 67 failed) | CPU interrupt bit-timing corruption (Checksum error) |

### Lab 5: Border Router Boundary & Latency Metrics

| Quantity | Prediction | Your value | Target |
|---|---|---|---|
| In-mesh RTT, C → S (`ot ping`) | 15 ms | 15.85 ms | your Lab 3 value |
| Wi-Fi hop, laptop → OTBR, power save on | 200 ms | 267 ms | — |
| Wi-Fi hop, laptop → OTBR, power save off | 20 ms | 15 ms | — |
| Laptop → S, `ping` | 100 ms | 109 ms | — |
| Laptop → S, CoAP GET `/env/temp` | 120 ms | 101 ms | < 200 ms |
| OTBR added latency: (laptop → S) − (laptop → OTBR) | 80 ms | 94 ms | < 100 ms |
| NAT64: S → 8.8.8.8 | 100 ms | Synthesized (`fd02:8e98:55d3:2:0:0:808:808`) | < 200 ms |
| OTBR off: C → S `PUT /act/valve` | Works | Works (`2.04 Changed`) | works |
