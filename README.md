# CryptaHome-Q: ML-KEM Hardware Co-Processor for Quantum-Resistant Smart Home Gateway

> **ChipFoundry Application Challenge — Proposal Submission**
> **Author:** Muhammad Bilal
> **Date:** March 25, 2026
> **License:** Apache 2.0
> **Target Sector:** Edge-IoT / Commercial Smart Home

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement & Market Need](#2-problem-statement--market-need)
3. [Proposed Solution](#3-proposed-solution)
4. [System Architecture](#4-system-architecture)
5. [Silicon Design — ML-KEM NTT Co-Processor](#5-silicon-design--ml-kem-ntt-co-processor)
6. [PCBA Design](#6-pcba-design)
7. [Firmware Architecture](#7-firmware-architecture)
8. [Mechanical Enclosure](#8-mechanical-enclosure)
9. [Verification & Validation Strategy](#9-verification--validation-strategy)
10. [Bill of Materials & Cost Analysis](#10-bill-of-materials--cost-analysis)
11. [Project Timeline](#11-project-timeline)
12. [Risk Analysis & Mitigation](#12-risk-analysis--mitigation)
13. [References](#13-references)

---

## 1. Executive Summary

**CryptaHome-Q** is a production-ready, quantum-resistant smart home gateway that offloads NIST FIPS 203 ML-KEM (formerly CRYSTALS-Kyber) key encapsulation to a custom silicon co-processor implemented inside the Caravel SoC harness on the SKY130 130 nm process.

The system pairs an ESP32-WROOM-32E application processor with the Caravel ASIC over SPI. The ESP32 manages the home automation stack — capacitive touchscreen UI, sensor aggregation (temperature, humidity, air quality, PIR motion, door/window reed switches), actuator control (smart locks, relay modules for lighting, motorized blinds, HVAC thermostat valve), and BLE 5.0 communication to a Raspberry Pi 4B hub. All key exchange and session establishment between the gateway and the hub are protected by ML-KEM-768, with the computationally expensive Number Theoretic Transform (NTT) and polynomial arithmetic offloaded to dedicated hardware inside Caravel. The Raspberry Pi hub runs a software-only ML-KEM stack (liboqs) and bridges the home network to the cloud via Wi-Fi/Ethernet.

The design targets a BOM cost under $28 for the edge node (excluding Caravel NRE) and is intended as a reference architecture demonstrating that post-quantum security can be embedded at the silicon level in cost-sensitive IoT endpoints, well ahead of the "harvest now, decrypt later" threat window.

---

## 2. Problem Statement & Market Need

### 2.1 The Quantum Threat to IoT

Classical key exchange algorithms (ECDH, RSA) deployed in billions of IoT devices today are vulnerable to Shor's algorithm running on a sufficiently capable quantum computer. While large-scale fault-tolerant quantum machines are still in development, the "harvest now, decrypt later" (HNDL) attack model is already active: adversaries intercept and store encrypted traffic today with the intent to decrypt it once quantum capability matures. For smart home devices — which handle physical security (locks, alarms), occupancy patterns, and personal health data — the confidentiality window extends far beyond the device's operational lifetime.

NIST finalized ML-KEM (FIPS 203) in August 2024 as the primary post-quantum key encapsulation mechanism. However, ML-KEM's core operation — polynomial multiplication in the ring Z_3329[x]/(x^256 + 1) via the Number Theoretic Transform — is computationally demanding for microcontrollers. A software ML-KEM-768 KeyGen + Encaps + Decaps cycle on an ESP32 at 240 MHz takes approximately 45–70 ms, which is acceptable for infrequent key exchanges but problematic when the system must handle multiple concurrent sessions, OTA re-keying, or burst reconnections after a power outage (a common smart home scenario where 10–30 devices reconnect simultaneously).

### 2.2 Why Hardware Acceleration Matters at the Edge

Dedicated NTT hardware reduces ML-KEM latency by 10–50× compared to software, freeing the ESP32 CPU for real-time sensor polling, display rendering, and BLE stack management. More importantly, a hardened silicon implementation provides:

- **Constant-time execution:** Eliminates timing side-channels that plague software NTT implementations on general-purpose cores with variable-latency multipliers and data-dependent branch predictors.
- **Reduced energy per operation:** Critical for battery-backed or solar-powered edge nodes; a single NTT in hardware at 33 MHz draws ~0.8 mW versus ~12 mW for software on ESP32 at 240 MHz.
- **Silicon root-of-trust potential:** The co-processor can be extended with a TRNG and secure key storage, providing a hardware trust anchor that firmware alone cannot guarantee.

### 2.3 Target Market

- Residential and small-commercial smart home installations requiring NIST PQC compliance.
- Critical infrastructure IoT (building access, HVAC) in government and defense supply chains mandating PQC migration per NSA CNSA 2.0 timelines.
- Retrofit PQC modules for existing Zigbee/Z-Wave gateways lacking quantum-resistant key exchange.

---

## 3. Proposed Solution

### 3.1 High-Level Concept

The CryptaHome-Q system consists of two nodes:

| Node | Role | PQC Implementation |
|------|------|--------------------|
| **Edge Gateway** (this design) | Sensor/actuator hub with touchscreen UI | ML-KEM in **hardware** (Caravel ASIC) + ESP32 firmware |
| **Home Hub** | Cloud bridge, rule engine, voice assistant integration | ML-KEM in **software** (liboqs on Raspberry Pi 4B) |

Communication between the Edge Gateway and Home Hub uses BLE 5.0 with ML-KEM-768 key encapsulation bootstrapping an AES-256-GCM session. The Caravel co-processor handles all NTT forward/inverse transforms and polynomial basemul operations; the ESP32 firmware manages the higher-level KEM protocol (sampling, encoding, compression, CPA/CCA transforms).

### 3.2 Why This Partitioning

Implementing the *entire* ML-KEM algorithm in Caravel RTL would consume excessive area at 130 nm (Keccak/SHA-3 alone requires ~0.5 mm² in SKY130). Instead, we offload only the **arithmetic bottleneck** — the NTT, INTT, and coefficient-wise polynomial operations — which represent 60–70% of ML-KEM execution time. The remaining operations (SHA-3/SHAKE, CBD sampling, encode/decode, compress/decompress) run efficiently in ESP32 firmware. This partitioning maximizes silicon efficiency within the 10 mm² user area while delivering the majority of the performance benefit.

---

## 4. System Architecture

### 4.1 System Block Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    EDGE GATEWAY NODE                         │
│                                                              │
│  ┌──────────────┐    SPI (20 MHz)    ┌──────────────────┐   │
│  │  ESP32-WROOM  │◄────────────────►│  CARAVEL ASIC     │   │
│  │   -32E        │  MOSI/MISO/SCK/CS │  (QFN-64)        │   │
│  │               │  + IRQ (GPIO)     │                   │   │
│  │  ┌─────────┐  │                   │  ┌─────────────┐  │   │
│  │  │ BLE 5.0 │  │                   │  │ NTT Engine  │  │   │
│  │  │ Stack   │  │                   │  │ (256-pt)    │  │   │
│  │  └─────────┘  │                   │  │ Barrett Mod │  │   │
│  │  ┌─────────┐  │                   │  │ Basemul     │  │   │
│  │  │ Touch   │  │                   │  │ Coeff SRAM  │  │   │
│  │  │ UI/LCD  │  │                   │  │ SPI Slave   │  │   │
│  │  └─────────┘  │                   │  │ Wishbone IF │  │   │
│  │  ┌─────────┐  │                   │  └─────────────┘  │   │
│  │  │ Sensor  │  │                   │                   │   │
│  │  │ Drivers │  │                   └──────────────────┘   │
│  │  └─────────┘  │                                          │
│  │  ┌─────────┐  │                                          │
│  │  │Actuator │  │                                          │
│  │  │Drivers  │  │                                          │
│  │  └─────────┘  │                                          │
│  └──────────────┘                                           │
│       │ I2C / GPIO / UART                                    │
│       ▼                                                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Sensors: BME680 (T/H/AQ), PIR (AM312), Reed SW,    │   │
│  │           IR Receiver, Light Sensor (BH1750)         │   │
│  │  Actuators: Relay Module (4ch), Servo (Lock),        │   │
│  │             MOSFET (LED dimmer), Thermostat Valve     │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
         │
         │  BLE 5.0 (ML-KEM-768 + AES-256-GCM)
         ▼
┌─────────────────────────────────────────────────────────────┐
│                     HOME HUB NODE                            │
│  ┌──────────────┐                                           │
│  │ Raspberry Pi  │  liboqs ML-KEM-768 (software)            │
│  │ 4B (4 GB)    │  Node-RED rule engine                     │
│  │              │  MQTT broker (Mosquitto)                   │
│  │  BLE 5.0 ◄──┘  Wi-Fi/Ethernet to cloud                  │
│  │  (USB dongle)                                            │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Communication Flow — PQC Key Exchange

1. **Session Initiation:** ESP32 sends `KEM_INIT` over BLE to Raspberry Pi.
2. **KeyGen (Raspberry Pi):** Pi generates ML-KEM-768 keypair `(pk, sk)` in software (liboqs). Sends `pk` (1184 bytes) to ESP32 over BLE.
3. **Encapsulation (ESP32 + Caravel):**
   - ESP32 receives `pk`, parses the polynomial vectors `t_hat` (in NTT domain).
   - ESP32 samples noise polynomials `r`, `e1`, `e2` using SHAKE-256 + CBD (firmware).
   - ESP32 sends polynomials to Caravel for NTT forward transforms (7× NTT calls for `r` vector, 3 polynomials × matrix multiply).
   - Caravel computes `NTT(r_i)`, returns results to ESP32.
   - ESP32 sends pairs for pointwise multiplication; Caravel performs `basemul` and returns products.
   - ESP32 sends accumulated results for INTT; Caravel returns time-domain polynomials.
   - ESP32 compresses, encodes ciphertext `ct` (1088 bytes) and shared secret `ss` (32 bytes).
   - ESP32 sends `ct` to Raspberry Pi over BLE.
4. **Decapsulation (Raspberry Pi):** Pi decapsulates `ct` using `sk` in software to recover `ss`.
5. **Session Established:** Both sides derive AES-256-GCM session key from `ss` using HKDF-SHA-256. All subsequent sensor data and actuator commands are AES-encrypted.

### 4.3 GPIO Pin Allocation (Caravel QFN-64)

| Caravel Pin | Function | Direction | Connected To |
|-------------|----------|-----------|-------------|
| mprj_io[7] | SPI_SCK | Input | ESP32 GPIO18 (VSPI_CLK) |
| mprj_io[8] | SPI_CS_N | Input | ESP32 GPIO5 (VSPI_CS0) |
| mprj_io[9] | SPI_MOSI | Input | ESP32 GPIO23 (VSPI_MOSI) |
| mprj_io[10] | SPI_MISO | Output | ESP32 GPIO19 (VSPI_MISO) |
| mprj_io[11] | IRQ_N | Output (open-drain) | ESP32 GPIO4 (interrupt) |
| mprj_io[12] | STATUS[0] | Output | Status LED (busy) |
| mprj_io[13] | STATUS[1] | Output | Status LED (error) |
| mprj_io[14] | DEBUG_TX | Output | Test header (UART debug) |
| mprj_io[15] | DEBUG_RX | Input | Test header (UART debug) |
| mprj_io[16:37] | Reserved | — | Future expansion / test points |
| mprj_io[0] | JTAG | Input | Debug header |
| mprj_io[1:4] | Housekeeping SPI | Bidirectional | Flash programming |
| mprj_io[5:6] | UART (mgmt) | Bidirectional | Debug console |

---

## 5. Silicon Design — ML-KEM NTT Co-Processor

### 5.1 Architectural Overview

The co-processor implements a configurable polynomial arithmetic engine targeting ML-KEM (FIPS 203) parameters:

| Parameter | Value |
|-----------|-------|
| Ring | Z_3329[x] / (x^256 + 1) |
| Polynomial degree n | 256 |
| Modulus q | 3329 (12-bit) |
| NTT type | Incomplete (7 layers of CT butterflies, 128 degree-1 basemuls) |
| Security level | ML-KEM-768 (NIST Level 3) |

**Critical Design Note:** Unlike Kyber v1/v2 (q=7681), the standardized ML-KEM uses q=3329, which is **not NWC-NTT friendly** (2n=512 does not divide q−1=3328, since 3328 = 2^8 × 13 and 512 = 2^9 > 2^8). Therefore, a complete 256-point negacyclic NTT cannot be performed directly. The standard implementation uses an **incomplete NTT** with 7 layers of Cooley-Tukey butterflies (reducing 256 coefficients to 128 pairs), followed by degree-1 polynomial basemul for pointwise multiplication, and 7 layers of Gentleman-Sande butterflies for INTT. This matches the NIST reference C implementation structure.

### 5.2 RTL Microarchitecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    USER PROJECT WRAPPER                           │
│                                                                   │
│  ┌─────────────┐    ┌──────────────────────────────────────┐     │
│  │  SPI Slave   │    │         NTT PROCESSING ENGINE        │     │
│  │  Interface   │    │                                      │     │
│  │  (Mode 0,    │───►│  ┌──────────┐    ┌──────────────┐   │     │
│  │   8/16-bit)  │    │  │ Butterfly │    │ Twiddle ROM  │   │     │
│  │             │◄───│  │ Unit (BFU) │◄──│ (128 × 12b)  │   │     │
│  └──────┬──────┘    │  │ CT + GS    │    └──────────────┘   │     │
│         │           │  │ configurable│                       │     │
│  ┌──────┴──────┐    │  └──────┬─────┘    ┌──────────────┐   │     │
│  │  Wishbone   │    │         │          │ Barrett      │   │     │
│  │  Bridge     │    │         ▼          │ Reduction    │   │     │
│  │  (32-bit    │    │  ┌──────────┐     │ Unit         │   │     │
│  │   slave)    │    │  │ Coeff    │     │ (mod 3329)   │   │     │
│  └─────────────┘    │  │ SRAM     │     └──────────────┘   │     │
│                     │  │ 2×256×12b│                         │     │
│  ┌─────────────┐    │  │ (dual-   │     ┌──────────────┐   │     │
│  │  Control    │    │  │  port)   │     │ Basemul Unit │   │     │
│  │  FSM &      │───►│  └──────────┘     │ (degree-1    │   │     │
│  │  Sequencer  │    │                    │  poly mult)  │   │     │
│  └─────────────┘    │                    └──────────────┘   │     │
│                     └──────────────────────────────────────┘     │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Register File (Wishbone-mapped)                             │ │
│  │  0x3000_0000: CTRL   — start, mode (NTT/INTT/basemul), rst  │ │
│  │  0x3000_0004: STATUS — busy, done, error flags               │ │
│  │  0x3000_0008: DATA_IN  — coefficient write port              │ │
│  │  0x3000_000C: DATA_OUT — coefficient read port               │ │
│  │  0x3000_0010: ADDR   — SRAM address for DMA-style access     │ │
│  │  0x3000_0014: CONFIG — butterfly count, layer select          │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### 5.3 Key Functional Blocks

#### 5.3.1 Butterfly Unit (BFU)

The BFU implements both Cooley-Tukey (NTT) and Gentleman-Sande (INTT) butterfly operations, selected by a mode bit:

**CT butterfly (NTT):**
```
a' = a + W·b mod q
b' = a - W·b mod q
```

**GS butterfly (INTT):**
```
a' = a + b mod q
b' = (a - b)·W^{-1} mod q
```

Where `W` is the twiddle factor fetched from the precomputed ROM. The multiplier is a 12×12 → 24-bit unsigned multiplier followed by Barrett reduction mod 3329. At 33 MHz (Caravel's external clock), one butterfly completes in 3 clock cycles (1 multiply + 1 reduction + 1 writeback).

#### 5.3.2 Barrett Reduction Unit

For modulus q=3329, Barrett reduction avoids expensive division:
```
t = (a * m) >> k
r = a - t * q
if r >= q: r = r - q
```
Where `m = floor(2^k / q)` is precomputed. For k=24 and q=3329, m=5039. The reduction requires one 24-bit multiply and one conditional subtraction — no iterative division.

#### 5.3.3 Twiddle Factor ROM

The incomplete NTT for q=3329 requires 128 twiddle factors (the 128th roots of unity in Z_3329, which *do* exist since 128 divides 3328). These are precomputed and stored in a 128×12-bit ROM synthesized from standard cells (~1,536 bits, negligible area). A separate 128-entry ROM stores inverse twiddle factors for INTT.

#### 5.3.4 Coefficient SRAM

Two banks of 256×12-bit dual-port SRAM (synthesized from flip-flops, ~6,144 bits total) enable ping-pong operation: the SPI interface fills Bank A while the NTT engine processes Bank B, minimizing idle time during batch operations.

#### 5.3.5 Basemul Unit

Handles the degree-1 polynomial multiplication required at the base of the incomplete NTT:
```
(a0 + a1·x) × (b0 + b1·x) mod (x² - ζ)
= (a0·b0 + a1·b1·ζ) + (a0·b1 + a1·b0)·x
```
Where `ζ` is the appropriate root from the twiddle factor table. This requires 4 multiplications and 2 additions mod q per pair, completing in 6 cycles.

### 5.4 Performance Estimates

| Operation | Cycles | Time @ 33 MHz |
|-----------|--------|---------------|
| 256-pt NTT (7 layers × 128 butterflies) | 2,688 | 81.5 μs |
| 256-pt INTT (7 layers × 128 butterflies) | 2,688 | 81.5 μs |
| 128 basemuls | 768 | 23.3 μs |
| SPI transfer (256 coefficients in, 16-bit) | 4,096 | 124.1 μs* |
| **Single polynomial multiply (NTT+basemul+INTT)** | **~6,144** | **~186 μs** |

*SPI transfer time at 20 MHz SPI clock.

A full ML-KEM-768 Encaps (including 9 NTTs, 9 INTTs, 9 basemul sets, and 3 NTT-domain additions) completes the hardware-accelerated arithmetic in approximately **3.4 ms**, compared to ~50 ms in pure ESP32 software — a **~15× speedup**.

### 5.5 Area Estimate

| Block | Estimated Area (SKY130) |
|-------|------------------------|
| Butterfly Unit (12-bit multiplier + adder/sub) | ~0.15 mm² |
| Barrett Reduction | ~0.08 mm² |
| Twiddle ROM (256 entries × 12 bits) | ~0.05 mm² |
| Coefficient SRAM (2 × 256 × 12 bits, synthesized) | ~0.4 mm² |
| Basemul Unit | ~0.12 mm² |
| SPI Slave + Wishbone Bridge | ~0.1 mm² |
| Control FSM + Sequencer | ~0.05 mm² |
| **Total estimated** | **~0.95 mm²** |

Well within the 10 mm² (2.92 mm × 3.52 mm) user project area, leaving ample room for routing, decoupling cells, and future extensions (e.g., TRNG, Keccak core).

### 5.6 Clock & Power Domains

The NTT engine runs on `vccd1` (1.8 V digital domain) clocked by the Caravel `wb_clk_i` at 33 MHz. The SPI slave interface is clocked by the external SPI_SCK from the ESP32, with a clock-domain crossing (CDC) synchronizer (2-flip-flop) between the SPI and wishbone clock domains. Estimated power consumption is ~5 mW at full throughput.

---

## 6. PCBA Design

### 6.1 Design Tool

All PCB design is done in **KiCad 8.x** with open-source symbol and footprint libraries.

### 6.2 PCB Specifications

| Parameter | Value |
|-----------|-------|
| Layer count | 4-layer (SIG-GND-PWR-SIG) |
| Dimensions | 100 mm × 80 mm |
| Finish | ENIG (lead-free) |
| Min trace/space | 6/6 mil |
| Impedance control | 50 Ω single-ended (SPI traces) |

### 6.3 Major Components

| Component | Part Number | Function |
|-----------|-------------|----------|
| ESP32-WROOM-32E | Espressif | Main application processor, BLE, Wi-Fi |
| Caravel ASIC | chipIgnite QFN-64 | ML-KEM NTT co-processor |
| ILI9341 2.8" TFT + Touch | Generic SPI | User interface display |
| BME680 | Bosch | Temperature, humidity, pressure, VOC gas |
| BH1750 | Rohm | Ambient light sensor (I2C) |
| AM312 | Generic | Miniature PIR motion sensor |
| Reed switch (×2) | Generic | Door/window open-close |
| 4-channel relay module | SRD-05VDC | Lighting, fan, appliance control |
| SG90 servo | TowerPro | Smart lock mechanism |
| AMS1117-3.3 | AMS | 3.3 V LDO for ESP32 + peripherals |
| AP2112K-1.8 | Diodes Inc. | 1.8 V LDO for Caravel core |
| W25Q128JVS | Winbond | 128 Mbit SPI Flash (Caravel firmware) |
| 33 MHz crystal + caps | Generic | Caravel clock source |
| IR receiver (TSOP38238) | Vishay | Remote control input |
| USB-C connector | Generic | 5 V power input + ESP32 programming |
| MOSFET (IRLZ44N) + gate driver | Various | LED strip dimmer (PWM) |
| Thermostat valve driver (H-bridge) | DRV8871 | Motorized valve for HVAC |

### 6.4 Power Architecture

```
USB-C 5V ──► AMS1117-3.3 ──► 3.3V rail (ESP32, sensors, relays, Caravel I/O)
                    │
                    └──► AP2112K-1.8 ──► 1.8V rail (Caravel vccd, vccd1, vccd2)
                    │
                    └──► Direct 5V ──► Relay coils, servo, MOSFET gate driver
```

Caravel analog domains (vdda1, vdda2) are connected to 3.3 V via the 3.3 V rail since no analog functionality is used in this digital-only design. All power domains have bulk and local decoupling (10 μF + 100 nF) per Caravel datasheet recommendations.

### 6.5 PCB Layout Considerations

- Caravel ASIC placed centrally with exposed pad (ground) soldered to a via-stitched ground plane for thermal dissipation.
- SPI traces between ESP32 and Caravel kept under 30 mm, length-matched within 2 mm for SCK/MOSI/MISO, with ground guard traces.
- Analog section (BME680, BH1750) isolated from relay/servo switching noise with a split ground plane and ferrite bead at the boundary.
- 3D antenna keep-out zone around ESP32 module (per Espressif layout guide).

---

## 7. Firmware Architecture

### 7.1 ESP32 Firmware Stack

```
┌────────────────────────────────────────────┐
│           Application Layer                 │
│   Touch UI  │  Sensor Mgr  │  Actuator Mgr │
├────────────────────────────────────────────┤
│           ML-KEM Protocol Layer             │
│   KeyGen helper  │  Encaps  │  Decaps       │
│   SHAKE-256/SHA3 │  CBD     │  Encode/Decode│
├────────────────────────────────────────────┤
│           Caravel HAL (Hardware Abstraction) │
│   SPI driver  │  NTT offload  │  IRQ handler│
├────────────────────────────────────────────┤
│           BLE Stack (NimBLE)                │
│   GATT server │  L2CAP CoC │  PQC profile   │
├────────────────────────────────────────────┤
│       ESP-IDF v5.x (FreeRTOS)              │
└────────────────────────────────────────────┘
```

### 7.2 Caravel HAL — NTT Offload API

```c
// Initialize Caravel SPI interface
void caravel_init(spi_host_device_t host, int cs_pin, int irq_pin);

// Load 256 coefficients into Caravel SRAM bank
void caravel_load_poly(const int16_t poly[256], uint8_t bank);

// Trigger NTT forward transform on specified bank
void caravel_ntt_forward(uint8_t bank);

// Trigger INTT (inverse NTT) on specified bank
void caravel_ntt_inverse(uint8_t bank);

// Trigger basemul between two banks, result in bank_out
void caravel_basemul(uint8_t bank_a, uint8_t bank_b, uint8_t bank_out);

// Read 256 result coefficients from Caravel SRAM bank
void caravel_read_poly(int16_t poly[256], uint8_t bank);

// Block until Caravel signals completion via IRQ or poll STATUS register
void caravel_wait_done(void);
```

### 7.3 BLE Communication Protocol

A custom GATT service (`0xPQC0`) with characteristics for:
- `KEM_PK` (1184 bytes, write): Receives ML-KEM-768 public key from Hub.
- `KEM_CT` (1088 bytes, read/notify): Ciphertext output from Encaps.
- `SESSION_STATUS` (1 byte, notify): Key exchange state machine status.
- `DATA_TX` (up to 244 bytes, notify): AES-256-GCM encrypted sensor data.
- `CMD_RX` (up to 244 bytes, write): AES-256-GCM encrypted actuator commands.

BLE 5.0 with 2M PHY and Data Length Extension (DLE) achieves ~1.4 Mbps application throughput, allowing the 1184-byte public key to transfer in ~7 ms.

### 7.4 Raspberry Pi Hub Software

- **OS:** Raspberry Pi OS Lite (64-bit)
- **ML-KEM library:** liboqs (Open Quantum Safe project), compiled with AArch64 NEON optimizations.
- **BLE stack:** BlueZ 5.x with `bluetoothctl` scripting or Python `bleak` library.
- **MQTT broker:** Mosquitto, local only, for internal device coordination.
- **Cloud bridge:** Python service forwarding selected events to AWS IoT Core / Home Assistant over TLS 1.3 (classical, since cloud-to-hub link is separately secured).
- **Rule engine:** Node-RED for user-defined automation rules (e.g., "if motion detected AND light < 50 lux, turn on living room lights").

---

## 8. Mechanical Enclosure

### 8.1 Design Tool

**FreeCAD 0.21** for parametric modeling, exported as STL for 3D printing (prototyping) and STEP for CNC milling (production).

### 8.2 Enclosure Specifications

| Parameter | Value |
|-----------|-------|
| External dimensions | 130 mm × 100 mm × 35 mm |
| Material (prototype) | PLA (FDM 3D print) |
| Material (production) | ABS injection molded |
| Color | White (body), translucent (status LED light pipe) |
| Mounting | DIN-rail clip (backplate) + wall-mount screw tabs |
| IP rating | IP20 (indoor use) |

### 8.3 Design Features

- Front panel cutout for 2.8" TFT display with capacitive touch overlay.
- Ventilation slots on sides for BME680 air sampling (with dust filter mesh).
- PIR sensor dome (Fresnel lens) flush-mounted on top face.
- LED status indicators visible through translucent light pipes (power, BLE link, PQC active, error).
- Rear cable gland entries (3× PG7) for relay wiring, sensor cables, and USB-C power.
- PCB standoffs with M3 brass inserts for vibration resistance.
- Snap-fit lid with single screw retention for tamper indication.

---

## 9. Verification & Validation Strategy

### 9.1 RTL Verification

| Test Level | Tool | Coverage Target |
|-----------|------|-----------------|
| Unit tests (BFU, Barrett, basemul) | Icarus Verilog + cocotb | 100% functional, all corner cases |
| NTT/INTT correctness | Icarus Verilog + Python golden model | Bit-exact match against reference NTT for all 128 twiddle factors |
| SPI slave protocol | cocotb SPI driver | Mode 0, single/burst transfers, CS timing |
| Wishbone bus integration | cocotb Wishbone driver | Read/write all registers, stall/ack handshake |
| Full ML-KEM flow | Verilator cycle-accurate + C testbench | End-to-end KeyGen/Encaps/Decaps producing correct shared secret |
| Gate-level simulation (GLS) | Icarus Verilog + SKY130 timing libs | Post-synthesis netlist with SDF annotation |
| STA | OpenSTA (via OpenLane) | Setup/hold at 33 MHz, all corners (ss/tt/ff, -40/25/100°C) |

### 9.2 Python Golden Model

A standalone Python reference implementing the incomplete NTT for q=3329 with the exact twiddle factor table used in RTL, enabling bit-exact comparison for every intermediate butterfly output across all 7 layers. The golden model is committed alongside the RTL as `verify/golden_ntt.py`.

### 9.3 System-Level Validation

- ESP32 ↔ Caravel SPI communication verified on Caravel development board before ASIC tapeout.
- ML-KEM-768 known-answer tests (KATs) from NIST run through the full hardware-accelerated path.
- BLE key exchange end-to-end test: ESP32 gateway ↔ Raspberry Pi hub, verifying shared secret match and AES-GCM session establishment.

### 9.4 Precheck & Tapeout

- OpenLane flow with `user_project_wrapper` integration.
- Caravel Precheck and Tapeout checks run on the ChipFoundry platform.
- DRC/LVS clean in Magic VLSI.

---

## 10. Bill of Materials & Cost Analysis

### 10.1 Edge Gateway BOM (per unit, 1000 qty)

| Component | Unit Cost (USD) |
|-----------|----------------|
| ESP32-WROOM-32E | $2.80 |
| Caravel ASIC (QFN-64) | Sponsored |
| 2.8" ILI9341 TFT + Touch | $4.50 |
| BME680 | $3.20 |
| BH1750 | $0.60 |
| AM312 PIR | $0.80 |
| Reed switches (×2) | $0.40 |
| 4-ch relay module | $2.50 |
| SG90 servo | $1.50 |
| W25Q128JVS Flash | $0.45 |
| Voltage regulators (×2) | $0.50 |
| Crystal, passives, connectors | $2.00 |
| 4-layer PCB (100×80 mm) | $3.50 |
| DRV8871 + MOSFET | $1.80 |
| IR receiver + misc | $0.60 |
| Enclosure (injection molded) | $2.50 |
| **Total** | **~$27.65** |

### 10.2 Home Hub BOM

| Component | Unit Cost (USD) |
|-----------|----------------|
| Raspberry Pi 4B (4 GB) | $55.00 |
| USB BLE 5.0 dongle | $8.00 |
| 32 GB microSD card | $5.00 |
| Case + power supply | $12.00 |
| **Total** | **~$80.00** |

### 10.3 Total System Cost

Edge Gateway (~$28) + Home Hub (~$80) = **~$108** for a complete quantum-resistant smart home system with 50+ sensor/actuator capacity per gateway.

---

## 11. Project Timeline

| Milestone | Date |
|-----------|------|
| Proposal submission | March 25, 2026 |
| RTL complete (NTT engine + SPI + Wishbone) | April 5, 2026 |
| Functional verification (unit + integration) | April 12, 2026 |
| OpenLane synthesis + place-and-route | April 18, 2026 |
| GLS + STA signoff | April 22, 2026 |
| KiCad PCB schematic + layout | April 25, 2026 |
| FreeCAD enclosure design | April 27, 2026 |
| Precheck + Tapeout submission | April 30, 2026 |
| Silicon return | October 2026 |
| PCBA assembly + bring-up | November 2026 |
| System integration + demo video | November 2026 |

---

## 12. Risk Analysis & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| NTT engine exceeds 10 mm² area | Low | High | Conservative area estimate includes 40% routing overhead; single-BFU design is compact |
| SPI timing violations at 20 MHz | Medium | Medium | SPI slave uses dedicated I/O registers with double-sync CDC; fallback to 10 MHz SPI |
| Barrett reduction off-by-one for edge cases | Low | High | Exhaustive test of all 3329² input combinations in simulation (~11M vectors) |
| BLE throughput insufficient for 1184-byte pk transfer | Low | Low | BLE 5.0 DLE supports 251-byte PDU; 5 packets at ~7 ms total is well within tolerance |
| Caravel management SoC firmware conflicts with user SPI | Low | Medium | User project SPI uses dedicated mprj_io pins, independent of housekeeping SPI on mprj_io[1:4] |
| SKY130 standard cell library lacks needed macros | Low | Medium | All SRAM is synthesized from flip-flops; no hard macros required |

---

## 13. References

1. NIST FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism Standard (ML-KEM), August 2024.
2. Avanzi, R., et al., "CRYSTALS-Kyber Algorithm Specifications v3.0," NIST PQC Standardization, 2021.
3. Satriawan, A., Mareta, R., Lee, H., "A Complete Beginner Guide to the Number Theoretic Transform (NTT)," Inha University, 2023.
4. Open Quantum Safe Project (liboqs): https://openquantumsafe.org/
5. Efabless Caravel SoC Datasheet, CARAVEL-001.
6. Espressif ESP32-WROOM-32E Datasheet, v1.3.
7. SKY130 PDK Documentation: https://skywater-pdk.readthedocs.io/
8. Barrett, P., "Implementing the Rivest Shamir and Adleman Public Key Encryption Algorithm on a Standard Digital Signal Processor," Advances in Cryptology, CRYPTO 1986.
9. NSA CNSA 2.0 — Commercial National Security Algorithm Suite 2.0, September 2022.

---

## License

```
Copyright 2026 Muhammad Bilal

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
