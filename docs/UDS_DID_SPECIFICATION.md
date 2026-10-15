# ISO 14229-1 UDS Data Identifier (DID) & Security Specification

This document outlines the standard Data Identifiers (DIDs) supported by the **`stm32-automotive-uds-isotp-node`** diagnostic server, including session constraints, access permissions, and Security Access (Service `0x27`) parameters.

---

## 1. Supported Diagnostic Sessions (Service `0x10`)

| Sub-Function | Session Type | Description | Permitted Services |
|:---:|:---|:---|:---|
| `0x01` | **Default Session** | Normal ECU operating mode on key-on | Read DIDs (`0x22`), TesterPresent (`0x3E`) |
| `0x02` | **Programming Session** | Bootloader flash erase & download mode | RequestDownload (`0x34`), TransferData (`0x36`) |
| `0x03` | **Extended Diagnostic Session** | Engineering & calibration mode | Write DIDs (`0x2E`), RoutineControl (`0x31`), SecurityAccess (`0x27`) |

---

## 2. Supported Data Identifiers (Service `0x22` / `0x2E`)

| DID | Length | Data Field | Access Mode | Required Session | Security Level |
|:---:|:---:|:---|:---:|:---:|:---:|
| `0xF189` | 4 Bytes | **ECU Software Version** (e.g. `1.0.4.0`) | Read Only | Default, Extended | None |
| `0xF190` | 17 Bytes | **Vehicle Identification Number (VIN)** | Read / Write | Read: Any / Write: Extended | Level 1 (0x27 01/02) |
| `0xF199` | 4 Bytes | **ECU Manufacturing / Flash Date** | Read Only | Any | None |
| `0x0100` | 8 Bytes | **EV Battery Telemetry Snapshot** (Voltage, Current, Temp) | Read Only | Any | None |
| `0x0101` | 4 Bytes | **Motor Inverter Speed & Torque Demand** | Read Only | Extended | None |
| `0x0200` | 2 Bytes | **Odometer Accumulated Mileage (km)** | Read Only | Any | None |
| `0x0300` | 6 Bytes | **ECU Serial Number** | Read / Write | Extended | Level 1 |

---

## 3. Security Access Algorithm (Service `0x27`)

Security Access prevents unauthorized firmware modification or VIN overwriting.

```
Client (Tester)                                 Server (ECU)
      │                                               │
      │── Request Seed: 0x27 0x01 ───────────────────►│
      │                                               │ Generates 32-bit PRNG Seed
      │◄── Response: 0x67 0x01 [Seed_0..3] ───────────│ Starts 10-second expiry timer
      │                                               │
      │ Computes Key = F(Seed, Secret_Mask)           │
      │                                               │
      │── Send Key: 0x27 0x02 [Key_0..3] ────────────►│ Validates Key == Expected_Key
      │                                               │ Unlocks Security Level 1
      │◄── Positive Response: 0x67 0x02 ──────────────│
```

### Negative Response Codes (NRC)
* `0x35` — **Invalid Key (`NRC_INVALID_KEY`):** Increments failed attempt counter (locks out for 10 seconds after 3 consecutive failures).
* `0x36` — **Exceeded Number of Attempts (`NRC_EXCEEDED_ATTEMPTS`):** Returns when tester exceeds maximum authentication attempts.
* `0x37` — **Required Time Delay Not Expired (`NRC_REQUIRED_TIME_DELAY_NOT_EXPIRED`):** Returns during penalty lockout window.
