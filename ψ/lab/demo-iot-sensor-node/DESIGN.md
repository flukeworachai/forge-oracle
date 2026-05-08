# IoT Environmental Sensor Node — Design Document

> Forge 🔥 Demo — Cross-domain design: Hardware + Firmware + Cloud

## Overview

Battery-powered environmental sensor node ส่งข้อมูล temp/humidity/air quality ผ่าน BLE → gateway → MQTT → dashboard

```
┌─────────────────────────────────────────────────────┐
│                    SYSTEM VIEW                       │
│                                                      │
│  ┌──────────┐    BLE     ┌──────────┐    WiFi/MQTT  │
│  │  Sensor  │───────────▶│ Gateway  │──────────────▶│
│  │   Node   │            │(ESP32-S3)│               │
│  │(nRF52840)│            └──────────┘               │
│  └──────────┘                              ┌────────┤
│       │                                    │ Cloud  │
│   ┌───┴───┐                                │        │
│   │Battery│                         ┌──────┴──────┐ │
│   │CR2032 │                         │  Dashboard  │ │
│   └───────┘                         │  (Grafana)  │ │
│                                     └─────────────┘ │
└─────────────────────────────────────────────────────┘
```

---

## Hardware Design

### Component Selection

| Component | Choice | Why | Alternative | Trade-off |
|-----------|--------|-----|-------------|-----------|
| **MCU** | nRF52840 | BLE 5.0 + ultra-low-power (3μA sleep) | ESP32-C3 | ESP32 = WiFi built-in แต่ 10x power |
| **Temp/Humidity** | SHT40 | ±0.2°C accuracy, I2C, 1.08μA sleep | BME280 | BME280 มี pressure ด้วย แต่แพงกว่า |
| **Air Quality** | SGP40 | VOC index, I2C, same bus as SHT40 | CCS811 | CCS811 ถูกกว่าแต่ accuracy ต่ำกว่า |
| **Battery** | CR2032 (225mAh) | Coin cell, easy replace | LiPo 100mAh | LiPo = rechargeable แต่ต้องมี charging circuit |
| **Antenna** | Chip antenna (2.4GHz) | Small footprint, PCB-mount | PCB trace | Trace antenna = free แต่ต้อง tune + กิน PCB space |

### Power Budget

```
Mode            Current     Duration    Duty Cycle
─────────────────────────────────────────────────────
Deep Sleep      3 μA        59.9 sec    99.83%
Sensor Read     5 mA        50 ms       0.08%
BLE Advertise   8 mA        5 ms        0.008%
BLE TX          12 mA       50 ms       0.08%
─────────────────────────────────────────────────────
Average:        ~15 μA

Battery Life:   225mAh / 0.015mA = ~15,000 hours = ~625 days
```

### Schematic — Key Connections

```
                    nRF52840
                 ┌────────────┐
     3.3V ──────┤VDD      P0.3├──── SHT40 SDA ──┐
     GND  ──────┤GND      P0.4├──── SHT40 SCL ──┤── I2C Bus
                │         P0.5├──── SGP40 SDA ──┘    (4.7kΩ pullups)
     CR2032 ────┤VDDH    P0.6├──── SGP40 SCL ──┘
                │             │
                │        P0.13├──── LED (status)
                │        P0.14├──── Button (wake/pair)
                │             │
                │     ANT_OUT├──── Chip Antenna
                │             │       │
                │             │    Matching
                │             │    Network
                └────────────┘    (π-filter)

    Decoupling: 100nF + 4.7μF on VDD
    Reset: 10kΩ pullup + 100nF to GND
```

### PCB Considerations

- **Size**: 25mm × 30mm (2-layer, fits CR2032 holder)
- **Layer Stack**: Top = components + signals, Bottom = GND plane
- **Antenna**: Keep clear zone (no copper pour within 5mm of antenna)
- **I2C**: Short traces, star topology from MCU
- **Power**: Wide traces for battery path (0.5mm min)

---

## Firmware Architecture

```
┌─────────────────────────────────────────┐
│              Application                 │
│  ┌───────────┐  ┌──────────┐  ┌───────┐│
│  │  Sensor   │  │   BLE    │  │ Power ││
│  │  Manager  │  │  Service │  │ Mgmt  ││
│  └─────┬─────┘  └────┬─────┘  └───┬───┘│
│        │              │             │    │
│  ┌─────┴──────────────┴─────────────┴───┐│
│  │           Zephyr RTOS                ││
│  │  ┌────────┐ ┌──────┐ ┌───────────┐  ││
│  │  │I2C Drv │ │BLE   │ │Power Mgmt │  ││
│  │  │        │ │Stack │ │(PM subsys) │  ││
│  │  └────────┘ └──────┘ └───────────┘  ││
│  └──────────────────────────────────────┘│
│              nRF52840 HAL                │
└─────────────────────────────────────────┘
```

### State Machine

```
         ┌──────────┐
    ┌───▶│  SLEEP   │◀──── Timer: 60sec ────┐
    │    └────┬─────┘                        │
    │         │ RTC wakeup                   │
    │    ┌────▼─────┐                        │
    │    │  SENSE   │ Read SHT40 + SGP40     │
    │    └────┬─────┘                        │
    │         │ data ready                   │
    │    ┌────▼─────┐                        │
    │    │ADVERTISE │ BLE advertisement      │
    │    └────┬─────┘                        │
    │         │ gateway connected            │
    │    ┌────▼─────┐                        │
    │    │ TRANSMIT │ Send via GATT          │
    │    └────┬─────┘                        │
    │         │ ACK received                 │
    └─────────┘
```

### Key Code Structure

```
firmware/
├── CMakeLists.txt
├── prj.conf                 # Zephyr config
├── boards/
│   └── sensor_node.overlay  # DTS overlay
├── src/
│   ├── main.c               # Init + state machine
│   ├── sensor.c             # SHT40 + SGP40 driver
│   ├── sensor.h
│   ├── ble_service.c        # Custom GATT service
│   ├── ble_service.h
│   └── power.c              # Sleep + wakeup logic
└── dts/
    └── bindings/
        └── sensirion,sht4x.yaml
```

---

## Gateway Design (ESP32-S3)

```
┌──────────────────────────────────┐
│         ESP32-S3 Gateway         │
│                                  │
│  BLE Scanner ──▶ Ring Buffer     │
│                      │           │
│               Batch Timer (30s)  │
│                      │           │
│              MQTT Publisher      │
│                      │           │
│              WiFi ───┴──▶ Cloud  │
└──────────────────────────────────┘
```

### MQTT Topic Structure

```
env/{gateway_id}/{node_id}/temperature   → 25.3
env/{gateway_id}/{node_id}/humidity      → 62.1
env/{gateway_id}/{node_id}/voc_index     → 142
env/{gateway_id}/{node_id}/battery       → 2.89
env/{gateway_id}/{node_id}/rssi          → -67
```

---

## Cloud Architecture

```
┌────────────┐     ┌─────────────┐     ┌──────────┐
│   MQTT     │────▶│  TimescaleDB│────▶│ Grafana  │
│  Broker    │     │  (time-     │     │Dashboard │
│ (Mosquitto)│     │   series)   │     │          │
└────────────┘     └─────────────┘     └──────────┘
      │                                      │
      │            ┌─────────────┐           │
      └───────────▶│  Alert Mgr  │──────────▶│
                   │ (threshold) │   Webhook  │
                   └─────────────┘           │
                                       ┌─────▼────┐
                                       │  LINE /  │
                                       │  Discord │
                                       └──────────┘
```

### Alert Rules

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| Temperature | > 35°C | > 40°C | LINE notify |
| Humidity | > 80% | > 90% | LINE notify |
| VOC Index | > 200 | > 300 | LINE + Discord |
| Battery | < 2.5V | < 2.2V | LINE (replace battery) |
| RSSI | < -85 dBm | < -90 dBm | Check node placement |

---

## BOM Estimate (1 unit)

| Part | Qty | Unit Cost | Total |
|------|-----|-----------|-------|
| nRF52840 module (Fanstel BT840) | 1 | $8.50 | $8.50 |
| SHT40 | 1 | $2.20 | $2.20 |
| SGP40 | 1 | $5.80 | $5.80 |
| CR2032 holder | 1 | $0.30 | $0.30 |
| Chip antenna + matching | 1 | $0.80 | $0.80 |
| Passives (caps, resistors) | ~15 | $0.02 | $0.30 |
| PCB (JLCPCB, 5pcs) | 1 | $1.00 | $1.00 |
| **Total** | | | **$18.90** |

---

## Trade-off Decisions

| Decision | Option A | Option B | Chose | Why |
|----------|----------|----------|-------|-----|
| MCU | nRF52840 | ESP32-C3 | nRF52840 | 10x lower sleep current → 2 year battery |
| Protocol | BLE | LoRa | BLE | Indoor use, gateway within 20m |
| RTOS | Zephyr | FreeRTOS | Zephyr | Native nRF support, better BLE stack |
| Cloud DB | TimescaleDB | InfluxDB | TimescaleDB | SQL compatible, better retention policies |
| Alert | Email | LINE | LINE | User ใช้ LINE ทุกวัน, response time ดีกว่า |

---

*Designed by Forge 🔥 — 2026-05-09*
> "ความคิดที่ดี ต้องผ่านไฟ — ถ้ามันรอด มันจะแกร่ง"
