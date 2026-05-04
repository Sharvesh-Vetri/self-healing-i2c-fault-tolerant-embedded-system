# Self-Healing I2C Fault-Tolerant Embedded System (PIC16F887)

A self-healing embedded system on PIC16F887 that detects, classifies, and autonomously recovers from I2C bus failures while preserving fault history using dual-buffer EEPROM logging.

---

## Why This Matters

In real embedded systems, I2C bus failures can freeze the entire system and require manual reset.

This project demonstrates how to:
- Detect communication failures in real time
- Prevent system-wide hangs using timeout protection and watchdog recovery
- Recover a broken I2C bus using NXP-recommended techniques
- Preserve fault history even when the communication bus is non-functional

These techniques are directly applicable to industrial, automotive, and IoT systems where reliability is critical.

---

## Features

- Real-time temperature monitoring via LM75 sensor over I2C
- Automatic I2C bus fault detection and classification (SDA stuck, SCL stuck, NACK, Timeout)
- Hardware watchdog timer integration with automatic watchdog reset logging
- Two-stage fault logging — buffers to internal EEPROM first, flushes to external EEPROM when bus recovers
- RTC-based fault timestamping using DS1307 with cached last-known-good time fallback
- Persistent fault log across power cycles using 24LC256 external EEPROM (up to 256 records)
- Automatic I2C bus recovery using the standard 9-clock bit-bang sequence
- Interactive fault log browsing on LCD with single-press navigation and hold-to-slideshow
- First-run detection using magic byte sentinel to prevent corrupt counter restoration
- No sprintf, no interrupts, no dynamic memory — safe and deterministic on mid-range PIC

---

## Schematic Layout

<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/aa0d38a6-f33a-41e8-bb1e-0a4b51d2cee1" />

--

## Hardware

| Component        | Description                        | Interface              |
|------------------|------------------------------------|------------------------|
| PIC16F887        | Main MCU, 8MHz internal oscillator | —                      |
| LM75             | Temperature sensor                 | I2C (0x90 / 0x91)     |
| DS1307           | Real-time clock                    | I2C (0xD0 / 0xD1)     |
| 24LC256          | External EEPROM, fault log storage | I2C (0xA0 / 0xA1)     |
| HD44780 16x4 LCD | Status and fault display           | PORTD, RE0=RS, RE1=EN  |
| 4.7kΩ pull-ups   | I2C bus pull-ups                   | RC3 = SCL, RC4 = SDA   |
| RB0              | Watchdog inject test button        | GPIO input             |
| RB1              | Fault log browse button            | GPIO input             |

---

## Pin Map

```
PIC16F887
├── RC3 (SCL) ──┬── LM75 SCL
│               ├── DS1307 SCL
│               └── 24LC256 SCL
├── RC4 (SDA) ──┬── LM75 SDA
│               ├── DS1307 SDA
│               └── 24LC256 SDA
├── PORTD[7:0] ── LCD Data Bus (D0–D7)
├── RE0 ────────── LCD RS
├── RE1 ────────── LCD EN
├── RB0 ────────── Watchdog inject button (active low)
└── RB1 ────────── Fault log browse button (active low)
```

---

## LCD Layout

```
Line 1 (0x80): +032C               <- Temperature
Line 2 (0xC0): Status:SYSTEM OK    <- System status / fault type
Line 3 (0x90): Fault 001:SDA       <- Fault record (browsing mode)
Line 4 (0xD0): Time 10:31:34       <- Fault timestamp (browsing mode)
```

---

## Fault Types

| Code | Name           | Cause                                      |
|------|----------------|--------------------------------------------|
| 1    | SDA STUCK      | SDA line held low, bus cannot be released  |
| 2    | SCL STUCK      | SCL line held low, bus is frozen           |
| 3    | NACK           | Device did not acknowledge — missing/hung  |
| 4    | TIMEOUT        | I2C operation exceeded timeout counter     |
| 5    | WD RST         | System was reset by the watchdog timer     |

---

## External EEPROM Log Format

Each fault record occupies 4 bytes stored sequentially from address 0x0000:

```
Address 0x0000: [fault_type] [hour] [minute] [second]  <- Record 0
Address 0x0004: [fault_type] [hour] [minute] [second]  <- Record 1
Address 0x0008: [fault_type] [hour] [minute] [second]  <- Record 2
...
Address 0x03FC: [fault_type] [hour] [minute] [second]  <- Record 255
```

Maximum 256 records. write_index wraps to 0 after record 255.

---

## Internal EEPROM Map

| Address | Content                                      |
|---------|----------------------------------------------|
| 0       | Buffered fault type                          |
| 1       | Buffered hour                                |
| 2       | Buffered minute                              |
| 3       | Buffered second                              |
| 4       | Valid flag (1 = fault pending flush)         |
| 5       | write_index low byte                         |
| 6       | write_index high byte                        |
| 7       | log_count low byte                           |
| 8       | log_count high byte                          |
| 9       | Init magic byte (0xA5 = initialized)         |

---

## Two-Stage Fault Logging

The system uses a two-stage logging strategy to guarantee no fault is ever lost, even if the I2C bus itself is the component that failed.

```
Fault detected
      │
      ▼
Stage 1: Write fault record to internal EEPROM (no I2C needed)
      │
      ▼
Perform I2C bus recovery (9-clock sequence)
      │
      ▼
Display fault on LCD for 2 seconds
      │
      ▼
Return to main loop — retry temperature reads
      │
  Bus healthy again?
      │
      ▼
Stage 2: Flush buffered record to external EEPROM
      │
      ▼
Clear valid flag — logging complete
```

---

## Fault Log Browsing (RB1)

| Action             | Result                                              |
|--------------------|-----------------------------------------------------|
| Single press       | Enter browsing mode, display fault record 0         |
| Press while browsing | Advance to next record, wraps after last record   |
| Hold RB1           | Slideshow mode — auto-advances every 5 seconds      |
| Release RB1        | Exit browsing mode, return to normal display        |

Empty EEPROM slots (never written) display as:
```
Line 3: NO RECORD
Line 4: ----
```

---

## Fault Handling Flow

```
1.  Read temperature from LM75
2.  If read fails, retry up to 3 times
3.  If still failing:
      a. Classify fault (SDA / SCL / NACK / TIMEOUT)
      b. Read RTC timestamp
         (use cached last-known-good time if RTC unreachable)
      c. Buffer fault record to internal EEPROM
      d. Run I2C bus recovery (9-clock bit-bang sequence)
      e. Display fault type on LCD for 2 seconds
4.  On next successful temperature read:
      a. Flush buffered fault to external EEPROM
      b. Update last-known-good timestamp cache
      c. Display temperature and SYSTEM OK on LCD
```

---

## Build Environment

| Tool       | Details                          |
|------------|----------------------------------|
| Compiler   | Microchip XC8 v2.40              |
| IDE        | MPLAB X v6.05                    |
| Simulator  | Proteus 8                        |
| MCU        | PIC16F887                        |
| Standard   | C99 (with C90 libraries on XC8)  |
| Oscillator | Internal 8MHz (INTRC_NOCLKOUT)   |

---

## Configuration Bits

```c
#pragma config FOSC  = INTRC_NOCLKOUT  // Internal oscillator, no clock output
#pragma config WDTE  = ON              // Watchdog timer enabled
#pragma config PWRTE = ON              // Power-up timer enabled
#pragma config MCLRE = ON              // MCLR pin enabled
#pragma config CP    = OFF             // Code protection off
#pragma config LVP   = OFF             // Low voltage programming off
```

---

## Key Design Decisions

**No interrupts** — All I2C, timing, and GPIO operations are fully polled, keeping the firmware simple and deterministic with no race conditions.

**No dynamic memory** — All buffers are fixed-size arrays on the stack, safe for a mid-range PIC with 368 bytes of RAM.

**No sprintf** — All LCD strings are built manually digit by digit to avoid XC8 C90 stdio linkage failures on mid-range devices.

**Defensive I2C** — Every I2C primitive has its own independent timeout counter so no bus condition can permanently hang the firmware.

**Two-stage logging** — Guarantees fault records are never lost even when the I2C bus itself is the component that failed.

**Magic byte sentinel** — Detects first power-up and prevents write_index and log_count from being corrupted by uninitialized 0xFF values in blank internal EEPROM.

**Cached RTC timestamp** — If the RTC is unreachable at the moment a fault occurs, the last successfully read timestamp is used so the log entry always has a meaningful time value.

---

## Author

Sharvesh V  
Created: 8 February 2026
