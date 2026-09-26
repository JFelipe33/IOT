# Design & Decision Record (DDR)
**GreenField Technologies | IoT Systems Design**

**Team Members:**
1. ____________________
2. ____________________

---

## 1. System Overview

*   **System Type:** [ ] Component (Lab 1-2) | [ ] System (Lab 3-6) | [ ] Environment (Lab 7-8)
*   **Description:**

---

## 2. Lab Log & Stakeholder Summaries

### Lab 1: RF Characterization
* **To Samuel (Architect):** Spectrum energy scan performed across channels 11–26. Selected Channel 15 due to an optimal noise floor (-103 dBm) located in the guard band between WiFi channels 1 and 6. This mitigates co-channel interference and establishes a baseline link budget before physical range deployment.
* **To Edwin (Ops):** Configured node network to Channel 15. If packet loss occurs during field operations, check if a new local WiFi router has been installed on WiFi Channel 1 or Channel 6.

### Lab 2: 6LoWPAN
*   **To Samuel:**

### Lab 3: Thread & CoAP
*   **To Daniela (Customer):**

### Lab 4: Sensors & Control
*   **To Edwin:**

### Lab 5: Border Router
*   **To Daniela:**

### Lab 6: Security
*   **To Edward (Security):**

### Lab 7: Dashboard
*   **To Gustavo (Product):**

### Lab 8: Final Integration
*   **To All:**

---

## 3. Architecture Decision Records (ADRs)

**ADR-001: Selection of 802.15.4 Radio Channel**
* **Context:** The 2.4 GHz ISM band is shared between IEEE 802.15.4 and high-power IEEE 802.11 (WiFi) networks. Uncontrolled co-channel interference causes frame collisions, forcing MAC retransmissions and degrading battery longevity.
* **Decision:** Operating channel set to **Channel 15** (2425 MHz).
* **Rationale:** Empirical energy scan measured a quiet noise floor of -103 dBm on Channel 15. Geometrically, Channel 15 sits in the spectral gap between standard WiFi Channel 1 and WiFi Channel 6, isolating our low-power mesh traffic from farm router interference.
* **Status:** [X] Accepted

---

## 4. ISO/IEC 30141 Mapping

### Domain Mapping

| Component | ISO Domain | Justification |
|-----------|------------|---------------|
| ESP32-C6 SoC | SCD | Sensing/controlling device (§6.4–6.5) |
| 802.15.4 radio + antenna | SCD | Communication subsystem |
| Air (RF medium) | PED | Physical entity — EM propagation |

### Component Capabilities

| Capability Category | Subcategory | Component/Feature | Active/Latent | Lab Introduced |
|---------------------|-------------|-------------------|---------------|----------------|
| Transducer | Actuation | On-board LED | Active | Lab 1 |
| Data | Processing & Storage | RSSI filtering / NVS storage / 802.15.4 TX-RX | Active | Lab 1 |
| Interface | Network & Serial | 802.15.4 network / OpenThread CLI / Serial monitor | Active | Lab 1 |
| Supporting | Security & Time | Time sync / Hardware crypto accelerator | Latent | Lab 1 |
| Latent | Wireless & Debug | BLE radio / WiFi radio / USB (debug interface) | Latent | Lab 1 |

---

## 5. First Principles Reflections

**Lab 1:**
1.
2.

**Lab 2:**
1.

...

---

## 6. Performance Baselines

| Metric | Target | Measured | Status |
|--------|--------|----------|--------|
| Lab 1: Max Range | > 20m | ___ m | [ ] Pass |
| Lab 2: Healing Time | < 120s | ___ s | [ ] Pass |
| Lab 3: CoAP Latency | < 200ms| ___ ms | [ ] Pass |
| Lab 4: Poll Latency | < 5s | ___ s | [ ] Pass |
| Lab 6: DTLS Time | < 3s | ___ s | [ ] Pass |

---

## 7. Ethics & Sustainability Checklist

*   [ ] **Lab 1:** Verified interference doesn't disrupt neighbors.
*   [ ] **Lab 4:** Data collection minimized (Privacy).
*   [ ] **Lab 5:** System works locally without cloud (Sustainability).
*   [ ] **Lab 6:** Encryption enabled (Privacy).
*   [ ] **Lab 8:** End-of-Life plan considered.

---

## 8. Viewpoint Analysis

| Viewpoint | Labs Addressed | Key Concerns Documented |
|-----------|----------------|-------------------------|
| Foundational |            |                         |
| Business |               |                         |
| Usage |                  |                         |
| Functional |             |                         |
| Trustworthiness |        |                         |
| Construction |           |                         |

---

## 9. Trustworthiness Audit (Lab 6+)

| Characteristic | Addressed? | How | Gaps |
|----------------|------------|-----|------|
| Availability |            |     |      |
| Confidentiality |         |     |      |
| Integrity |               |     |      |
| Reliability |             |     |      |
| Resilience |              |     |      |
| Safety |                  |     |      |
| Compliance |              |     |      |

---

## 10. Construction Viewpoint - IoT System Pattern (Lab 8)

### Lab 1: Range and Performance Baseline Data
| Distance (m) | RSSI A→B (dBm) | RSSI B→A (dBm) | PER A→B (%) | PER B→A (%) |
|:------------:|:--------------:|:--------------:|:-----------:|:-----------:|
| 1 m          |                |                |             |             |
| 10 m         |                |                |             |             |
| 30 m         |                |                |             |             |

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
