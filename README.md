# CAN-Bus-Black-Box

A flight recorder for robots and vehicles. An embedded Linux logger sits on a CAN bus, continuously records traffic, and automatically captures the moments before and after a fault so failures can be analyzed and replayed afterward.

**Status:** In design. V1 (logging) is next. See [Roadmap](#roadmap).


## Why

When a vehicle or robot stops unexpectedly, the question is always "what happened in the seconds before?" Live debugging rarely catches it. This project records the bus continuously and preserves the context around any fault, the same idea as an aircraft black box.

## Architecture

```
  STM32 (simulated vehicle nodes)
   ├── Motor RPM
   ├── Battery voltage
   ├── Temperature
   └── IMU angle
          │
     CAN transceiver
          │
       CAN BUS
          │
     CAN interface (MCP2515 over SPI, or USB-CAN)
          │
   Raspberry Pi (Linux + SocketCAN, can0)
          │
   CAN Receiver → Parser → Ring Buffer → Fault Detector
                                │              │
                                │        on fault: save
                                ▼        pre- and post-fault window
                             SQLite  ◄──────────┘
                                │
                        Analysis / Replay tool
```

## Hardware

| Part | Role |
|------|------|
| STM32 dev board | Simulates vehicle ECUs and transmits CAN frames |
| CAN transceiver (e.g. SN65HVD230 / TJA1050) | Physical-layer interface for the MCU |
| Raspberry Pi 4/5 | Logger running embedded Linux |
| MCP2515 CAN module (or USB-CAN adapter) | CAN interface for the Pi, exposed as `can0` via SocketCAN |
| 120 Ω termination resistors | One at each end of the bus |

## Planned CAN message set

Draft IDs, subject to change as the design firms up.

| ID | Signal | Rate | Payload |
|----|--------|------|---------|
| 0x101 | Motor RPM | 100 Hz | uint16 |
| 0x205 | Battery voltage | 10 Hz | uint16 (mV) |
| 0x301 | IMU angle | 100 Hz | int16 (0.1°) |
| 0x401 | Temperature | 1 Hz | int16 (0.1 °C) |
| 0x7FF | Fault / heartbeat | 1 Hz | flags |

## Key design decisions

- **SocketCAN** so the bus appears as a standard Linux network interface and standard tools (`candump`, `cansend`) work for debugging.
- **Ring buffer in RAM** holding the last 30 s of traffic, so normal operation avoids constant disk writes.
- **Batched SQLite writes** in transactions instead of per-message commits, to avoid I/O bottlenecks at high bus load.
- **Fault capture window:** on a fault, persist the previous 30 s plus the next 10 s into its own timestamped database file.
- **Fault tolerance:** logger runs as a `systemd` service with automatic restart, so a crash doesn't end the recording.

## Roadmap

- [ ] **V1:** Pi receives CAN frames from the STM32 and saves them
- [ ] **V2:** SQLite storage and a basic analysis script (plots per signal)
- [ ] **V3:** Fault detection (thresholds, missing heartbeat, bus errors)
- [ ] **V4:** Rolling buffer with automatic pre/post-fault capture
- [ ] **V5:** Replay mode and dashboard

## Skills this project exercises

- **Embedded:** STM32 firmware in C, CAN peripheral, sensor data framing
- **Linux:** SocketCAN, systemd services, process management, logging
- **Software engineering:** multithreading, database design, error handling, testing, Git

## Build and run

Setup instructions will be added as each version lands.

```bash
# Bring up the CAN interface on the Pi (500 kbit/s)
sudo ip link set can0 up type can bitrate 500000

# Watch raw traffic
candump can0
```
