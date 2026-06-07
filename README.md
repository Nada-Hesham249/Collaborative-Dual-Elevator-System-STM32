# 🏢 Dual-Elevator Control System
> Collaborative embedded system running on two STM32 Cortex-M4 boards, coordinating two elevators across 4 floors over a real-time SPI link.

<img width="1013" height="696" alt="image" src="https://github.com/user-attachments/assets/33e08e29-ad6f-4654-9202-a00423b9329b" />

---

##  What does it do?

Two physical STM32 boards talk to each other over SPI to run a smart elevator system:

- 🧠 **Board A (Master)** — acts as the building brain. It reads all hallway call buttons, decides which elevator to send, and runs Elevator A
- 🤖 **Board B (Slave)** — runs Elevator B and constantly streams its status to the Master so it can make smarter decisions
- 📡 Every **50ms**, the two boards exchange a full status packet — floor, direction, speed, door state — so nothing goes out of sync
- 🚨 If comms die, both boards switch to **independent emergency mode** automatically

---

## ⚙️ How it works

```
  HALLWAY BUTTON PRESSED
         │
         ▼
  Master reads call ──► Runs dispatch algorithm ──► Picks best elevator
         │
         ├─► Elevator A? ── Master drives it directly
         │
         └─► Elevator B? ── Master sends command over SPI ──► Slave executes
```

Each elevator runs as a **Finite State Machine**:

```
IDLE ──► MOVING_UP / MOVING_DOWN ──► DOORS_OPEN ──► IDLE
                    │
              EMERGENCY (overrides everything, any time)
```

Motor speed is simulated with a **PWM LED**:
-  0% duty = stopped
-  20% duty = slow
-  100% duty = full speed

---

## 📦 SPI Packet (8 bytes)

Every exchange between the two boards uses this fixed frame:

| Byte | Field | Value |
|------|-------|-------|
| 0 | Header | Always `0xA5` |
| 1 | FSM State | IDLE / MOVING / DOORS_OPEN / EMERGENCY |
| 2 | Current Floor | 1 – 4 |
| 3 | Target Floor | 1 – 4 |
| 4 | Direction | NONE / UP / DOWN |
| 5 | Door Status | OPEN / CLOSED |
| 6 | Reserved | `0x00` |
| 7 | Checksum | XOR of bytes 0–6 |

> The Slave **pre-loads** its TX register before the Master even asks — so the Master always gets fresh data with zero wait.

---

## 🧠 Dispatch Algorithm

When a hallway button is pressed, the Master picks which elevator to send using this priority order:

| Priority | Condition | Action |
|----------|-----------|--------|
|  0 | SPI comm fault | Master handles everything alone |
|  1 | Elevator already at that floor and idle | Assign immediately |
|  2 | Elevator moving toward floor, same direction | Assign — it'll stop on the way |
|  3 | Elevator passed the floor, same direction | Assign only if nothing better |
|  4 | Elevator moving away | Skip — don't assign |
|  5 | Both idle | Assign to nearest by floor distance |

---

## 🗂️ File Structure

```
├── 📁 Master/
│   ├── main.c                 # Init + 50ms scheduler tick
│   ├── fsm_elevator_a.c       # Elevator A state machine
│   ├── dispatcher.c           # The dispatch algorithm
│   ├── spi_master.c           # Non-blocking SPI master driver
│   ├── uart_telemetry.c       # Status print every 500ms
│   └── exti_handlers.c        # All button/sensor ISRs
│
├── 📁 Slave/
│   ├── main.c                 # Init + FSM loop
│   ├── fsm_elevator_b.c       # Elevator B state machine
│   ├── spi_slave.c            # Non-blocking slave (pre-loads TX)
│   ├── uart_telemetry.c       # Status print every 500ms
│   └── exti_handlers.c        # Cabin buttons + floor sensors
│
└── 📁 Shared/
    └── spi_protocol.h         # Frame definition + checksum — used by both boards
```

