# Design & Decision Record (DDR)
**GreenField Technologies | IoT Systems Design**

**Team Members:**
1. Juan Felipe Pachon Restrepo
2. ____________________

## 1. System Overview

*   **System Type:** [X] Component (Lab 1-2) | [X] System (Lab 3-6) | [ ] Environment (Lab 7-8)
*   **Description:** ESP32-C6-DevKitC-1 node operating IEEE 802.15.4 radio at 2.4 GHz (Channel 15, 0 dBm TX power) using OpenThread for empirical RF characterization.

## 2. Lab Log & Stakeholder Summaries

### Lab 1: RF Characterization
- **To Samuel (Architect):** Spectrum energy scan performed across channels 11–26. Selected Channel 15 due to an optimal noise floor (-103 dBm) located in the guard band between WiFi channels 1 and 6. This mitigates co-channel interference and establishes a baseline link budget before physical range deployment.
- **To Gustavo (Product / Performance Summary):**
  - Max reliable range: 10 m (PER < 6% when RSSI > -80 dBm; PER ~1–2% at 0 m when RSSI > -72 dBm).
  - Recommended spacing: 10 m (accounting for indoor obstacles and vegetation margin).
  - Best channel: Channel 15 (noise floor -103 dBm); avoid WiFi channels 11–14, 16–19, 21–24.
  - 10-hectare field estimate: ~100 nodes × $40 = $4,000.
  - Verdict: ✅ Proceed — ESP32-C6 platform meets baseline link requirements operating on Channel 15.
- **To Edwin (Ops / Field Troubleshooting Checklist):**
  - Won't join: Verify device channel is set to Channel 15 (`ot dataset channel 15`); ensure PCB antenna is not touching metal enclosures or wet ground; maintain clear direct line of sight free from heavy metal structures.
  - Intermittent loss: If RSSI < -70 dBm, move node 2–3 meters closer to neighboring node; run `ot scan energy 500` to detect new local WiFi interference; elevate node at least 50 cm above dense vegetation.
  - Measured range guidelines: Line-of-sight (unobstructed): 10–15 m; Light obstacles / Indoor: 10 m; Heavy walls: < 5 m (drastic signal drop).

### Lab 2: 6LoWPAN Mesh
- **To Samuel (Architect):** Latency cost per hop is ~14.7 ms (15.85 ms for 1 hop vs 30.55 ms for 2 hops). Firmware must exclusively use the Mesh-Local EID (ML-EID) for application traffic because RLOC addresses change dynamically during topology reconvergence, whereas ML-EIDs remain static and maintain reachability.
- **To Edwin (Ops):** The network healed in an average of 8.33 seconds after router failure, easily satisfying the < 2-minute requirement. If a field node goes dark, technicians should check physical distance and RF line-of-sight to neighboring nodes, and inspect `ot router table` and neighbor tables on adjacent nodes to verify if link costs recomputed properly.

### Lab 3: Thread & CoAP
- **To Daniela (Customer):** Battery life is extended to ~1.56 years using 2× AA batteries by keeping the valve's radio in sleep mode between 5-second polling cycles. Soil temperature dashboard updates are pushed automatically only when values change (> 0.5 °C) via Observe, eliminating stale periodic data while saving 90.7% of transmission energy.
- **To Edwin (Ops):**
Valve commands use confirmable messages (CON) where an immediate 2.04 Changed response proves the action was received and executed.
If a valve is offline, the system retransmits 4 times over 45 seconds before failing loudly with a ResponseTimeout error.
Field technicians should expect an average control response time of 2.5 seconds (maximum 5 seconds), matching the valve's 5-second sleep cycle.

### Lab 4: Sensors & Control
- **To Edwin (Ops):**
  Field technicians tell a dead sensor from a dead node by checking the CoAP response code, as a dead sensor returns an explicit 5.03 Service Unavailable error after 60 seconds of freshness expiration. Conversely, a dead node causes complete radio silence, resulting in a 42.4-second CoAP ResponseTimeout error or missing 45-second Observe heartbeats. In short, if the node explicitly answers 5.03, inspect the sensor wiring, but if the node remains silent or times out, check node power and mesh connectivity.

### Lab 5: Border Router
- **To Daniela (Customer):**
  You can now monitor soil temperature and control the valve directly from your phone through the web dashboard with a response time of 101 ms over Wi-Fi. If the new Border Router box fails or loses power, off-mesh access from your phone will drop, but the field nodes will automatically keep working locally, allowing anyone in the field to operate the valve using a local mesh controller.

### Lab 6: Security
*   **To Edward (Security):**

### Lab 7: Dashboard
*   **To Gustavo (Product):**

### Lab 8: Final Integration
*   **To All:**

## 3. Architecture Decision Records (ADRs)

**ADR-001: Selection of 802.15.4 Radio Channel**
- **Context:** The 2.4 GHz ISM band is shared between IEEE 802.15.4 and high-power IEEE 802.11 (WiFi) networks. Uncontrolled co-channel interference causes frame collisions, forcing MAC retransmissions and degrading battery longevity.
- **Decision:** Operating channel set to **Channel 15** (2425 MHz).
- **Rationale:** Empirical energy scan measured a quiet noise floor of -103 dBm on Channel 15. Geometrically, Channel 15 sits in the spectral gap between standard WiFi Channel 1 and WiFi Channel 6, isolating our low-power mesh traffic from farm router interference.
- **Status:** [X] Accepted

**ADR-002: Selection of Thread Mesh Topology over LoRaWAN Star Topology**
- **Context:** Irrigation actuator control requires low-latency bidirectionality and high network availability. A star topology (LoRaWAN) creates single points of failure and high latency for multi-hop coverage.
- **Decision:** Adopt an OpenThread 6LoWPAN mesh topology.
- **Rationale:** Empirical testing proved that Thread achieves ~15.85 ms 1-hop and ~30.55 ms 2-hop RTT latency, suitable for real-time valve actuation. In failure testing ("Tractor Test"), the mesh automatically healed in **8.33 seconds** (well below the 120 s limit) by routing around dead nodes without downtime.
- **Status:** [X] Accepted

**ADR-003: Selection of CoAP + CBOR over MQTT/JSON for Mesh Application Layer**
- **Context:** Constrained 802.15.4 mesh networks cannot afford the high packet overhead, header sizes, and TCP connection maintenance required by HTTP or MQTT.
- **Decision:** Adopt CoAP (RFC 7252) over UDP with CBOR (RFC 8949) binary payloads and RFC 7641 Observe.
- **Rationale:** CoAP reduces application headers to 4 bytes and total message payload to 13–19 bytes, fitting easily inside a single 127-byte 802.15.4 frame without 6LoWPAN fragmentation. MQTT was rejected for the radio mesh because it requires persistent TCP connections and periodic keepalives that force the radio receiver to stay active, destroying battery longevity. The Observe pattern reduced telemetry traffic by 90.7% (28 notifications vs 300 polls over 5 minutes) compared to 1 Hz polling.
- **Status:** [X] Accepted

**ADR-004: Selection of 5-Second Poll Period for Battery-Powered Irrigation Valves**
- **Context:** Battery-operated field valves must turn off their receivers to save power, but must remain responsive enough for agricultural irrigation commands.
- **Decision:** Set the Sleepy End Device (SED) polling interval (`pollperiod`) to 5000 ms (5 s).
- **Rationale:** Empirical tests confirmed an average downlink ping RTT of 2,443.6 ms (max 4,444 ms), which is completely imperceptible for irrigation control. Polling every 5 seconds reduces the radio-on duty cycle to 0.2%, extending 2× AA battery lifespan to 1.56 years based on active RX ($74\text{ mA}$) and sleep ($35\text{ }\mu\text{A}$) current profiles.
- **Status:** [X] Accepted

**ADR-005: Selection of a Single OpenThread Border Router (OTBR) for Pilot Deployment**
- **Context:** Connecting the isolated 802.15.4 Thread mesh to the farm's Wi-Fi network is required for remote mobile monitoring, but adding gateway hardware introduces cost, added latency, and potential single-point-of-failure risks.
- **Decision:** Deploy a single dual-chip OTBR (ESP32-S3 host + ESP32-C6 RCP) as an SCD-hosted IoT gateway.
- **Rationale:** Empirical testing showed that the OTBR introduces only 94 ms of routing latency (101 ms total median CoAP GET time), easily meeting the < 200 ms target. In failure testing (OTBR disconnected), in-mesh actuation between Node C and Node S maintained 100% functionality with zero downtime, proving that off-mesh dependency is isolated to remote access without threatening local field autonomy.
- **Status:** [X] Accepted

## 4. ISO/IEC 30141 Mapping

### Domain Mapping

| Component | ISO Domain | Justification |
|-----------|------------|---------------|
| ESP32-C6 SoC | SCD | Sensing/controlling device (§6.4–6.5) |
| 802.15.4 radio + antenna | SCD | Communication subsystem |
| Physical Sensor Module (DHT11) | SCD | Hardware transducer acquiring environmental readings |
| ESP32-S3 + C6 RCP (OTBR) | SCD | SCD-hosted IoT gateway (§A.4 / Figure A.5) |
| Air (RF medium) | PED | Physical entity — EM propagation |
| Air / Soil Temperature & Humidity | PED | Physical entity being observed — Ambient thermal and moisture state |

### Component Capabilities

| Capability Category | Subcategory | Component/Feature | Active/Latent | Lab Introduced |
|---------------------|-------------|-------------------|---------------|----------------|
| Transducer | Actuation | On-board LED | Active | Lab 1 |
| Data | Processing & Storage | RSSI filtering / NVS storage / 802.15.4 TX-RX | Active | Lab 1 |
| Interface | Network & Serial | 802.15.4 network / OpenThread CLI / Serial monitor | Active | Lab 1 |
| Supporting | Security & Time | Time sync / Hardware crypto accelerator | Latent | Lab 1 |
| Latent | Wireless & Debug | BLE radio / WiFi radio / USB (debug interface) | Latent | Lab 1 |
| Gateway | Protocol & Routing | OTBR dual-chip gateway (Wi-Fi ↔ Thread, NAT64) | Active | Lab 5 |

### Thread Address Classification

| Type | Pattern | Scope | Used for | Lab Introduced |
| :--- | :--- | :--- | :--- | :---: |
| Link-local | `fe80::/64` | 1 radio hop | Neighbor discovery & MLE signaling between adjacent nodes. | Lab 2 |
| RLOC | `<prefix>:0:ff:fe00:<RLOC16>` | Entire mesh | Dynamic packet routing based on current network location. | Lab 2 |
| ALOC | `<prefix>:0:ff:fe00:fcXX` | Entire mesh | Anycast role identification (`fc00` identifies the Leader). | Lab 2 |
| ML-EID | `<prefix>:<random IID>` | Entire mesh | Stable application-layer device identification. | Lab 2 |
| OMR | `fd02:8e98:55d3:1:<SLAAC IID>` | Entire mesh / Wi-Fi | Routable address advertised by OTBR for off-mesh reachability. | Lab 5 |

### Functional Viewpoint — Thread Roles & Power Profiles

| Thread Role | Functional Role | Power Profile | Impact on 3-Month Battery Target | Lab Introduced |
| :--- | :--- | :--- | :--- | :---: |
| Leader | Network Coordinator | Always-On (100% RX, ~30mA) | Cannot run on batteries long-term; requires main power. | Lab 2 |
| Router | Packet Forwarder | Always-On (100% RX, ~30mA) | Cannot run on batteries long-term; requires main power or solar. | Lab 2 |
| REED | Standby Forwarder | Always-On (100% RX, ~30mA) | Operates as leaf until promoted; continuous RX drains battery quickly. | Lab 2 |
| End Device (FED) | Leaf Node | Always-On (100% RX, ~30mA) | Keeps radio receiver active; cannot meet 3-month battery life. | Lab 2 |
| Sleepy End Device (SED)| Leaf Sensor Node | Duty-Cycled (~1% RX, 5µA sleep) | Only role that meets 3-month target by sleeping between polls. | Lab 2 |
| Border Router (OTBR) | Gateway / Leader | Always-On (Mains-Powered) | Bridges Wi-Fi and Thread; requires external power supply. | Lab 5 |

### Application & Service Domain (ASD) — API Contracts

#### `/env/temp` — Air Temperature Interface
| Parameter | Contract Specification |
|---|---|
| Methods | `GET`, `GET` + Observe (RFC 7641) |
| Content-Format | `60` (`application/cbor`) |
| Payload | `{"t": float16}`, °C, 6 bytes (`A1 61 74 F9 hh ll`) |
| Accuracy | $\pm 2.0^\circ\text{C}$ (DHT11 datasheet rating) |
| Max-Age | $60\text{ s} - \text{reading age}$ (counts down to 0) |
| Plausibility | Rejects $T < -40^\circ\text{C}$, $T \ge 85^\circ\text{C}$, or $\Delta T > 5.0^\circ\text{C}$ in $2\text{ s}$ |
| Error Response | `5.03 Service Unavailable` with `Max-Age: 10` when reading age $> 60\text{ s}$ |
| Observe Policy | `NON` push when $\Delta T > 0.5^\circ\text{C}$; `CON` heartbeat after 45 s without change; single `5.03` terminates observation on expiration |

#### `/act/valve` — Irrigation Valve Interface
| Parameter | Contract Specification |
|---|---|
| Methods | `PUT` (set state), `GET` (read state) |
| Request Payload | CBOR `{"v": 0 \| 1}` (Content-Format 60) or text `"0"` / `"1"` |
| Response Payload| CBOR `{"v": 0 \| 1}` (Content-Format 60) |
| Response Codes | `2.04 Changed` (PUT success), `2.05 Content` (GET), `4.00 Bad Request` |
| Idempotency | Yes: duplicate `PUT` requests preserve the same state |

```cddl
env-reading = { t: float16 }        ; °C
valve-state = { v: 0 / 1 }          ; 0 = closed, 1 = open
```
### ISO/IEC 30141 §A.4 — IoT Gateway Scorecard (Enterprise System Pattern, Table A.3)

> *"IoT gateways are devices which connect SCD with other domains. IoT gateways provide functions such as protocol conversion, address mapping, data processing, information fusion, certification, and equipment management."*

| Function | Provided by OTBR? | Empirical Evidence / Rationale |
|---|---|---|
| Protocol conversion | Yes | Converts 802.15.4 / 6LoWPAN mesh datagrams to Wi-Fi 802.11 IPv6/IPv4 packets. |
| Address mapping | Yes | Performs NAT64 translation (`fd02:8e98:55d3:2:0:0::/96` mapping to IPv4). |
| Data processing | No | Forwards CoAP/UDP packets transparently without altering application payloads. |
| Information fusion | No | Does not aggregate or fuse multiple sensor measurements into combined records. |
| Certification | No | Unauthenticated shell access over USB and open CoAP endpoints ($\rightarrow$ Lab 6). |
| Equipment management | Yes | Advertises OMR prefixes via Router Advertisements, manages Thread Network Data, and acts as Leader. |

### ISO/IEC 30141 §A.5 — Enterprise Networking Pattern (Table A.4 Four-Network Walk)

| Network | System Component in SoilSense | Concrete IP Address / Range Assigned |
|---|---|---|
| Proximity | Thread Mesh Network (Node S, Node C, OTBR Radio) | ML-EID `fdab:6a0d:311a:e59c::/64` & OMR `fd02:8e98:55d3:1::/64` |
| Access | OTBR Wi-Fi Interface & Mobile Hotspot ("Juan") | OTBR IPv4 `10.101.156.84` & Link-Local `fe80::4a27:e2ff:fe2d:9a3c` |
| Services | Flask CoAP Dashboard & Upstream IPv4 Targets | Laptop Dashboard `10.101.156.xx:5000` & NAT64 IPv4 Target `8.8.8.8` |
| User | Daniela's Mobile Phone Browser | IP assigned dynamically by hotspot DHCP |

## 5. First Principles Reflections

**Lab 1:**
1. **Why does RSSI drop with distance?**  
   As you move away from the transmitting antenna, the radio signal spreads over an increasingly larger area. Therefore, the receiving antenna captures less energy at greater distances.

2. **The receiver detects signals down to -100 dBm — why did you need > -70 dBm for < 1% packet loss?**  
   Because to decode the data, merely detecting the signal is not enough; it must overcome the ambient noise floor and maintain sufficient margin to compensate for reflections and obstacles without losing packets.

**Lab 2:**
1. **How does a 40-byte IPv6 header fit in a 127-byte frame?**  
   6LoWPAN uses IPHC header compression. The link-local IPv6 Interface Identifier (IID) is omitted from the transmission because the receiver reconstructs it directly from the 802.15.4 MAC address. ML-EID uses context-based compression, reducing the 40-byte header down to 2–6 bytes.

2. **Why did healing take seconds and not minutes?**  
   Healing occurred in ~8 seconds because active ICMPv6 ping traffic triggered immediate MAC ACK failures when Node B went down. The sending node detected link failure instantly without waiting for the 100-second MLE advertisement timeout used on idle links.

3. **Why does Part 4 need a *poor* A–C link instead of *no* link?**  
   A poor link ($LQI=1$) allows A and C to hear each other's fallback routing advertisements while giving the path through B ($1+1=2$) a lower cost. A total MAC deny-list would simulate nodes completely out of range, preventing the mesh from switching to the direct link when B dies. In Daniela's field, nodes must be spaced so alternative neighbors remain within weak RF range.

**Lab 3:**
1. **Why can CoAP use UDP where HTTP needs TCP? What does CON give back, and what does NON deliberately give up?**
   CoAP uses UDP to avoid TCP's high connection setup and keepalive costs. CON restores reliability by requiring an ACK and auto-retransmitting on loss. NON deliberately drops ACKs to minimize radio time on routine periodic readings.

2. **Why does Observe save battery compared with `GET` every 60 s? Who knows when the value changed, and who pays for the radio?**
   GET forces the client to wake up and poll even when data hasn't changed. Observe lets the server—which alone knows when data shifts—push updates only when temperature moves $>0.5^\circ\text{C}$ or every 45 s on heartbeat, cutting traffic by over 90%.

3. **Why did the SED receive the same PUT twice in Task 5.3, and why was that harmless? What would have happened with a "toggle valve" command?**
   CoAP retransmits after 2–3 s assuming loss, but the valve sleeps for 5 s. The parent buffers both requests and delivers them together when the valve wakes. It was harmless because `PUT` sets an explicit state (`OPEN`). A non-idempotent `toggle` command would have opened and immediately closed the valve.

4. **Why CBOR and not a hand-made binary struct?**
   Custom C structs break whenever fields change or memory alignment differs across platforms. CBOR retains JSON's flexible key-value model while shrinking payloads down to 6 bytes in a standardized, schema-driven format.

**Lab 4:**
1. **Why does `/env/temp` answer 5.03 instead of the last value it has?**  
   Serving an old reading as current misleads the application, which could leave irrigation valves closed during extreme heat. Returning `5.03 Service Unavailable` explicitly warns the client that fresh data is missing rather than giving stale numbers past their Max-Age.

2. **How does a client tell "sensor dead" from "node unreachable"?**  
   A dead sensor allows the node to stay online and actively reply with a `5.03` error code after 60 seconds. An unreachable node causes complete silence, leading to a CoAP `ResponseTimeout` (42.4 seconds) or missed heartbeats.

3. **Why can't a plausibility check stop a value that is wrong but believable?**  
   Local checks only verify if a reading falls inside physical range limits and maximum step sizes. A value that is wrong but plausible (like 25.5 °C instead of 21.0 °C) satisfies all local rules, requiring external cross-validation or cryptographic proof to detect.

4. **Why can a busy radio make a sensor read fail?**  
   The DHT11 uses a single-wire protocol where the CPU manually measures microsecond timing pulses. Incoming radio interrupts pause the CPU during the reading window, distorting the measured pulse widths and causing checksum errors.

**Lab 5:**
1. **Why does a prefix advertisement plus SLAAC make S reachable from outside, with no gateway "magic"?**  
   The OTBR advertises the OMR prefix (`fd02:8e98:55d3:1::/64`) to the Thread mesh via Network Data, allowing Node S to configure a globally routable IPv6 address using SLAAC (RFC 4862). The OTBR then acts purely as a standard IPv6 router without requiring custom application-layer proxying or translation.

2. **Why is NAT64 needed when Thread is IPv6?**  
   Thread nodes operate on IPv6 only, but external Internet services and servers frequently use legacy IPv4 addresses. NAT64 synthesizes an IPv6 destination by prepending a `/96` prefix to the 32-bit IPv4 address, enabling IPv6-only mesh devices to communicate with IPv4 servers.

3. **Why are the bytes of `/env/temp` the same on both sides of the OTBR?**  
   Because the OTBR functions at the network layer (Layer 3 IPv6 routing) rather than the application layer. It forwards UDP packets transparently without altering or re-encoding the inner CoAP/CBOR payload.

4. **Why should a mains-powered gateway turn Wi-Fi power save off, when a battery node would do the opposite?**  
   Enabling Wi-Fi power save forces the radio receiver to sleep periodically, introducing high latency spikes (up to 267 ms). A mains-powered gateway prioritizes low latency (15 ms) over power conservation, whereas a battery-powered sensor node must prioritize battery lifespan.

**Lab 6:**
...

## 6. Performance Baselines

| Metric | Target | Measured | Status |
|--------|--------|----------|--------|
| Lab 1: Max Range | > 20m | 10 m (RSSI > -80 dBm, PER < 6%) | [X] Pass |
| Lab 2: Healing Time | < 120s | 8.33 s | [X] Pass |
| Lab 3: CoAP 1-Hop GET RTT | < 200ms | 15.85 ms | [X] Pass |
| Lab 3: CON Max Retry Span | < 120s | 93.0 s (Error 28: ResponseTimeout) | [X] Pass |
| Lab 3: SED Latency (1s poll)| < 2s | 544.1 ms avg (min 148 ms / max 1,061 ms) | [X] Pass |
| Lab 3: SED Latency (5s poll)| < 6s | 2,443.6 ms avg (min 443 ms / max 4,444 ms) | [X] Pass |
| Lab 3: SED Latency (15s poll)| < 20s | 6,850.1 ms avg (min 479 ms / max 14,479 ms) | [X] Pass |
| Lab 4: Lost Sensor Detection | ≤ 60 s | 60.0 s (5.03 response) | [X] Pass |
| Lab 4: Busy Radio Failure Rate | < 5% | 19.4% (13/67 failed reads under ping flood) | [ ] Fail |
| Lab 5: OTBR added latency | < 100 ms | 94 ms ((Laptop → S) - (Laptop → OTBR)) | [X] Pass |
| Lab 5: NAT64 RTT | < 200 ms | Synthesized `fd02:8e98:55d3:2:0:0:808:808` via /96 | [X] Pass |
| Lab 6: DTLS Time | < 3s | ___ s | [ ] Pass |


## 7. Ethics & Sustainability Checklist

*   [X] **Lab 1:** Verified interference doesn't disrupt neighbors.
* [X] **Lab 3 Safety & Sustainability Design (Failsafe Mechanism):** 
  To prevent crop damage, soil erosion, and water wastage if the network dies after receiving an `OPEN` (`v: 1`) command, the valve firmware implements a local hardware safety timer (15-minute cutoff). If no network heartbeat or explicit `CLOSE` (`v: 0`) command is received within 15 minutes, the valve autonomously shuts down (`v: 0`) on its own without requiring network connectivity.
*   [X] **Lab 4 (Safety Design for Missing Data):** If an automated irrigation control rule receives a `5.03` error or no CoAP response for longer than 60 seconds, it must default to a safe state (`CLOSE` valve) after a 5-minute timeout to prevent crop flooding or water depletion.
*   [X] **Lab 5 (Local Autonomy & Sustainability):** Verified that if the OTBR dies or loses power, in-mesh CoAP control between field nodes (C → S) continues to operate locally without cloud or gateway dependency.
*   [ ] **Lab 5 Security Gap (Unauthenticated Shell):** Anyone with physical access to the OTBR's USB cable can issue `ot` shell commands without authentication.
*   [ ] **Lab 5 Privacy Gap (Open Wi-Fi Access):** Any device connected to the Wi-Fi hotspot can freely query sensor readings or trigger valve actuation without encryption or proof of identity ($\rightarrow$ Lab 6).
*   [ ] **Lab 6:** Encryption enabled (Privacy).
*   [ ] **Lab 8:** End-of-Life plan considered.

## 8. Viewpoint Analysis

| Viewpoint | Labs Addressed | Key Concerns Documented |
|-----------|----------------|-------------------------|
| Foundational |            |                         |
| Business |               |                         |
| Usage |                  |                         |
| Functional |             |                         |
| Trustworthiness | Lab 4 | Data freshness (Max-Age), sensor failure signatures, radio load impact, and self-healing limits. |
| Construction | Lab 5 | Dual-chip OTBR architecture (ESP32-S3 host + ESP32-C6 RCP) and ISO/IEC 30141 Annex A Enterprise System (A.3) and Networking (A.4) patterns. |

## 9. Trustworthiness Audit (Lab 6+)

| Characteristic | Addressed? | How | Gaps |
|----------------|------------|-----|------|
| Availability | Yes (Lab 4) | Max-Age freshness tracking; explicit `5.03 Service Unavailable` on stale data (> 60 s); `ResponseTimeout` (42.4 s) when node is unreachable. | Server RAM reboot or 5.03 error clears observer subscriptions, requiring client re-registration. |
| Confidentiality | No | → Lab 6 | Unencrypted UDP payloads accessible to eavesdroppers on the radio medium. |
| Integrity | Yes (Lab 4) | Local range (-40 to 85 °C) and step-check (5 °C max jump) filters catch hardware/sensor fault spikes. | Cannot reject plausible false readings (e.g., injected 25.5 °C vs 21.0 °C) without multi-sensor fusion or authentication (→ Lab 6). |
| Reliability | Yes (Lab 4) | Sensor reads sampled every 2 s with 0.0% failure rate under quiet radio conditions. | 19.4% failure rate under busy radio traffic (ping flood) exceeds < 5% target due to CPU single-wire timing interrupts. |
| Resilience | Yes (Lab 4) | Auto-heals from transient read errors and wire reconnection (`Sensor back`). | Does not auto-heal active CoAP observations after 5.03 or node reboot; sensor driver missing at boot requires manual reset. |
| Safety |                  |     |      |
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
| 0 m          |  -68           |     -72        |   1         |      2      |
| 10 m         |   -78          |    -80         |    4        |        6    |
| 20 m         |     -94        |     -99        |     30      |        37   |

> **Testing note:** Measurements were conducted indoors. It was verified that with a clear line of sight without obstacles, the signal remained strong; however, placing walls or obstacles between the boards caused a drastic drop in signal level (RSSI) and increased packet loss.

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
| MQTT/JSON reading (Lab 0) | — | ~40 B | 1 | Persistent TCP connection |
| CoAP GET `/env/temp` | 15 B | 17 B | 2 | 1 UDP req, 1 UDP resp |
| CoAP Observe notification | — | 19 B | 1 | Single push NON datagram |
| CoAP PUT `/act/valve` (CON) | 18 B | 13 B | 2 | Piggybacked ACK response |

### Lab 4: Real Sensor Characterization & Reliability Baseline Data

#### DHT11 Sensor Characterization
| Metric | Datasheet Rating | Measured Value |
|---|---|---|
| Resolution | 1.0 °C | 1.0 °C |
| Noise Spread (1 min static) | N/A | 0.0 °C |
| Accuracy | ±2.0 °C | ±2.0 °C |
| Read Execution Latency | N/A | 21,811 µs avg (18,115 µs min / 22,113 µs max) |

#### Failure Mode & Radio Impact Baseline

| Test Condition | Measured Value | Failure Signature |
|---|---|---|
| Sensor Unplugged Detection | 60.0 s | Explicit `5.03 Service Unavailable` (`no fresh reading`) |
| Node Off Network Detection | 42.40 s | `coap receive response error 28: ResponseTimeout` |
| Quiet Radio Failure Rate (2 min) | 0.0% (0 / 76 failed) | Clean timing slots] |
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
