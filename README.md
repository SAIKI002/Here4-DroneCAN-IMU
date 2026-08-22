# Here4 GPS — Firmware Extraction & DroneCAN/IMU Analysis

This project investigates the **Here4 GNSS/IMU module**, its **STM32F302K8 microcontroller**, and its communication with the **Cube Orange+ flight controller through DroneCAN**. The work combines low-level firmware debugging and extraction using **SEGGER J-Link/SWD** with analysis of the Here4 sensor-data path, particularly GPS, magnetometer, accelerometer, and gyroscope data. The aim was to establish low-level access to the Here4, inspect and preserve its firmware, understand its DroneCAN communication path, and investigate how its IMU data is published and received by the flight controller.


> **Ownership & Attribution**
>
> This work was performed during an internship at **CoE-CNDS, VJTI**. The associated hardware, firmware, proprietary materials, experimental data, and internship-related intellectual property remain the property of **CoE-CNDS / VJTI**.
>
> The raw firmware image is **not included** in this repository.

---

## 1. Firmware Extraction

### Target

| Parameter | Value |
|---|---|
| MCU | STM32F302K8 |
| CPU | Arm Cortex-M4 with FPU |
| Flash | 64 KiB |
| SRAM | 16 KiB |
| Flash Address | `0x08000000 – 0x0800FFFF` |
| SWDIO | PA13 |
| SWCLK | PA14 |
| Reset | NRST |

The **KORE hardware was used to provide physical access to the debugging interface**. It was not the firmware target.

### Debug Interface

The Here4 was accessed using **SEGGER J-Link over SWD**.

Successful debug-port discovery:

```text
DAP initialized successfully.
Found SW-DP with ID 0x2BA01477
AP[0]: AHB-AP
Core found
Found Cortex-M4 r0p1
Cortex-M4 identified.
```

### RDP Analysis

The STM32 option bytes were inspected:

```text
Mem32 0x1FFFF800, 4
1FFFF800 = 00FF55AA 00FF00FF 00FF00FF 00FF00FF
```

The RDP byte was identified as:

```text
RDP = 0xAA
RDP Level = Level 0
```

This indicated that readout protection was disabled for the observed target state.

### Firmware Backup

The complete 64 KiB Flash region was extracted using:

```text
SaveBin D:\flash_backup.bin, 0x08000000, 0x10000
```

| Property | Value |
|---|---|
| Size | 65,536 bytes |
| Start | `0x08000000` |
| End | `0x0800FFFF` |
| SHA-256 | `4fe305311a39fc12cdbbfa30867817561d9cc0e79b808f8fea94856225c08217` |

The actual `flash_backup.bin` is **not included** in this repository.

The Flash was subsequently erased and verified through memory readback:

```text
08000000 = FFFFFFFF FFFFFFFF FFFFFFFF FFFFFFFF
```

The complete extraction and recovery record is available in:

[`docs/firmware-extraction-report.md`](docs/firmware-extraction-report.md)

---

## 2. DroneCAN & IMU Analysis

A second part of the investigation focused on the communication between the **Here4 and Cube Orange+ over DroneCAN**, particularly the Here4 IMU data path.

### DroneCAN Node

The Here4 was detected as:

| Parameter | Observed Value |
|---|---|
| Node ID | `125` |
| Node Name | `org.ardupilot.Here4AP` |
| State | OPERATIONAL |
| Health | OK |

A **DroneCAN Node ID** is the identifier used to distinguish a device on the CAN network.

Therefore, `Node 125` identifies the Here4 as a participant on the observed DroneCAN network. It is not the MCU ID or firmware version.

### Communication Path

```text
Here4
  │
  │ DroneCAN
  ▼
Cube Orange+
  │
  ▼
ArduPilot
  │
  ▼
Mission Planner
```

### IMU Investigation

The Here4 AP_Periph firmware was examined for its IMU implementation.

The source contained an IMU update path using:

```text
imu.get_accel()
imu.get_gyro()
```

and a DroneCAN:

```text
uavcan_equipment_ahrs_RawIMU
```

publisher.

The RawIMU message was identified as:

```text
DTID = 1003
```

### Runtime Observation

DroneCAN traffic from Node 125 was successfully observed, including messages such as:

```text
N125 DTID=1001
N125 DTID=341
N125 DTID=1
N125 DTID=1063
N125 DTID=20003
```

However:

```text
N125 DTID=1003
```

was **not observed** during the retained CAN-traffic monitoring.

Therefore:

- RawIMU publishing mechanism → **identified in source**
- RawIMU DTID 1003 → **identified**
- DroneCAN communication → **confirmed**
- Here4 accelerometer reception by Cube → **not independently verified**
- Here4 gyroscope reception by Cube → **not independently verified**

### Magnetometer Validation

The magnetometer path was used to confirm that Here4 sensor data was reaching the Cube through DroneCAN.

Observed Mission Planner output included:

```text
MAG1 X=-0.030 Y=-0.324 Z=-0.160
```

This confirmed a working sensor-data path:

```text
Here4 Magnetometer
        ↓
     DroneCAN
        ↓
   Cube Orange+
        ↓
    ArduPilot
        ↓
 Mission Planner
```

### Key Finding

The investigation established that the examined Here4 firmware contains an **IMU acquisition and RawIMU publishing mechanism**, but the live CAN investigation did **not independently verify RawIMU DTID 1003 transmission/reception**.

The absence of DTID 1003 was therefore treated as an investigation result rather than an assumed root cause.

The detailed DroneCAN/IMU analysis is available in:

[`docs/here4-dronecan-imu-analysis.md`](docs/here4-dronecan-imu-analysis.md)

---

## 3. Evidence

| Investigation | Result |
|---|---|
| STM32F302K8 identification | Confirmed |
| SWD/J-Link access | Confirmed |
| RDP Level 0 | Confirmed |
| 64 KiB firmware extraction | Confirmed |
| Here4 DroneCAN Node 125 | Confirmed |
| DroneCAN communication | Confirmed |
| Here4 magnetometer reception | Confirmed |
| RawIMU publisher in examined source | Identified |
| RawIMU DTID | `1003` |
| RawIMU DTID 1003 observed on CAN | Not observed |
| Here4 accelerometer/gyro reception | Not independently verified |

---

## Documentation

- [`Firmware Extraction Report`](docs/firmware-extraction-report.md)
- [`DroneCAN & IMU Analysis`](docs/here4-dronecan-imu-analysis.md)
- [`Physical Evidence`](images/)

