# CosmoLink

### Wireless long‑range data tunnel – high‑performance tracking, 2 Mbps throughput, and more

## Project Overview
CosmoLink is a **Layer 1 and 2 communication solution** designed to transmit data between ground and air units using a dual‑band approach:
- **433 MHz LoRa** (Semtech LR2021) for low‑datarate telemetry and positioning data.
- **2.4 GHz high‑throughput band (MSK/FLRC)** for high‑bandwidth real time data streaming.

---
## Project Goals 
- **2 Mbps half‑duplex tunnel** (1 Mbps each direction or 2 Mbps in one direction)
- **long range** up to 50km
- **small size** suitable for air-born vehicles
- **antenna tracking** with high performance high frequency positioning

---

## Architecture

**basic node**
```
+-------------------------------+        +------------------------------+
|   Ground Unit    		| <----> |    Air Unit       		|
| (STM32)    			|        | (STM32)       		|
|  LR2021 #1 (LoRa)            	|        |  LR2021 #1 (LoRa) 		|
|  LR2021 #2 (MSK/FLRC)    	|        |  LR2021 #2 (MSK/FLRC) 	|
|  positioning unit		|	 |  positioning unit		|
+-------------------------------+        +------------------------------+
```
- **STM32** STM32H562 or H523 (to decide in RnD) handles peripheral control, packet framing, position calculations and timing.
- **LR2021 #1** – LoRa link (433 MHz) for low‑rate telemetry and positioning data. This band will support position data for an antenna‑tracker system.
- **LR2021 #2** – Configured in MSK/FLRC mode (2.4 GHz) for high‑bandwidth data transfer between nodes using directional antennas (on the ground unit with an antenna-tracker).

**positioning unit**
```
+-----------------------+
| BNO086 (IMU)		|
| M10Q (GNSS)		|
+-----------------------+
```
- **BNO086** supports 9dof sensor fusion with built in ARM Cortex M0+.
- **M10Q** generate accurate positioning data with 10Hz refresh rate, and accurate time data for synchronisation.

***positioning unit** is not a critical component and the device must stay operational on shorter ranges if antenna tracking is not needed*

---

## Communication Stack
| Radio | Band | Modulation | Data Rate | Direction |
|-------|------|------------|-----------|-----------|
| LR2021 | 433 MHz (UHF)| LoRa  | ≤ 50 kbps (typical) | Telemetry/position |
| LR2021 | 2.4 GHz (S-band) | MSK/FLRC | >2 Mbps (half‑duplex) | High‑bandwidth payload |

*It is important to keep data rate of the high-band data channel at over 2Mbps, for link essential overhead (ACK, SYN, ID, call sign).*

### Physical Layer Performance (FLRC / MSK)

**FLRC**

- **coding rate**: 1
- **Bandwidth**: 2.4 MHz
- **Tx Power**: >30dbm
- **Packet Size**: 255 bytes (The hardware FIFO limit of the Semtech chip)
- **Preamble & Sync**: 2-byte preamble and a 32-bit (4-byte) sync word
- **Time on Air (ToA)**: ~0.82 ms/packet.
- **Chip State Transition**: TX / RX ~40 - 60 µs.
- **Expected One-Way Latency**: ~1.1 - 1.3 ms/packet from MCU buffer to MCU buffer.
---
### clock synchronisation 
- PTP-lite 3 way handshakes
- State machine driven two stage lock
  - Jam-Sync (few ms precision)
  - PI controller (sub us lock)
- 64bit timer made of 2 32bit timers
- time sync over s-band

### software
- cubeMX
- VScode stm32 extantion
- bare bone C
- using hal drivers

### hardware 
- G-NiceRF LoRa2021F33-2G4 modules are planed for LR2021
- stm32
- w5500




### planned buffer operation modes
- constant latency
  - modes for selection the bufor size (for example 0ms (no retransmissions), 40ms, 100ms)
  - packets that won’t make it in time will be dropped
- variable latency
  - modes for selection max time (for example 0ms - 100ms)
  - packets that won’t make it in time selected in the upper range will be dropped, but in ideal conditions (LQ100) latency will be minimal

### planned data transfer operation modes
- standard version
  - data in/out over random channels (TCP, UART, SPI, CAN)
  - sending raw bytes
- layer 2 ethernet bypass
  - data in/out over ethernet
  - w5500 in MACRAW mode
  - working as an invisible wireless ethernet cable
  - data retransmission still can apply (ack, back)

### S-band packet 

Max size 255-byte

| header | length |Sync number | payload | CRC |
|-------|-----|--|------------|-----------|
|8 bit| 8 bit | 16 bit| the rest lmao | 16 bit |


## S-band TDD

**Time devision duplex super frame system**

```
+-----------------------------------------------------------------------------------------------------------+
|                                   			 TDD Superframe (20 ms)                               		|
+---------------------------------------+-------------------+-----------------------------+-----------------+
|    Ground TX Slot Window (N Packets)  |  Guard (250 µs)   | Air TX Slot Window (M Pkts) |Guard (250 µs)   |
+---------------------------------------+-------------------+-----------------------------+-----------------+
```

In every super frame there will fit 22-23 transmission windows that will be split between two nodes. Every node will have at lest 2 transmission windows for ACK/NACK/PTP/other types of frames.

### TDD slot allocation
