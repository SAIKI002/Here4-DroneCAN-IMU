# Here4 STM32F302K8 — Firmware Extraction, Flash Backup & Recovery

## 1. Purpose and Scope

The task was to obtain low-level debug access to the Here4 GPS module, identify its MCU, establish ARM Serial Wire Debug (SWD) communication using a SEGGER J-Link, inspect Readout Protection (RDP), read MCU Flash, create a raw firmware backup, and document a controlled erase/recovery workflow.

The firmware target was the **STM32F302K8 inside the Here4**. The KORE hardware was used for physical access to the debugging port; the KORE board itself was not the firmware target.

## 2. Hardware Architecture

```text
Here4 GPS module
   ↓
STM32F302K8 target MCU
   ↓
Accessible debug/test points through the KORE interface arrangement
   ↓
SWD: SWDIO + SWCLK + NRST + VTref + GND
   ↓
SEGGER J-Link
   ↓ USB
PC / J-Link Commander
```

## 3. Target MCU

- MCU: STM32F302K8
- CPU: Arm Cortex-M4 with FPU
- Maximum CPU frequency: 72 MHz
- Main Flash: 64 KiB
- SRAM: 16 KiB
- Main Flash address: `0x08000000–0x0800FFFF`
- SRAM address: `0x20000000–0x20003FFF`
- SWDIO: PA13
- SWCLK: PA14
- Reset: NRST, active-low

## 4. SWD and J-Link Pin Configuration

| Signal | STM32F302K8 | J-Link ARM 20-pin | Function |
|---|---|---:|---|
| VTref | Target VDD/reference | 1 | Target voltage reference |
| SWDIO | PA13 | 7 | Bidirectional SWD data |
| SWCLK | PA14 | 9 | SWD clock |
| NRST | NRST | 15 | Active-low reset |
| GND | VSS/GND | 4,6,8,10,12,14,16,18,20 | Common ground |
| SWO | Not required | 13 | Optional trace output |

The exact KORE-to-Here4 pad/connector numbering was not preserved clearly enough to state reliably and is therefore not included.

## 5. Initial Connection Failure

Initial attempts produced:

```text
Failed to initialize DAP.
Can not attach to CPU. Trying connect under reset.
Connecting to CPU via connect under reset failed.
```

The investigation checked:

- physical SWD continuity
- target reference voltage
- common ground
- SWDIO
- SWCLK
- NRST
- target selection
- connection conditions

The important milestone was obtaining a valid DAP/SW-DP connection.

## 6. Successful SWD Connection

```text
DAP initialized successfully.
Found SW-DP with ID 0x2BA01477
DPIDR: 0x2BA01477
AP[0]: AHB-AP
Core found
Found Cortex-M4 r0p1
Cortex-M4 identified.
```

This directly established communication with the STM32 debug infrastructure.

## 7. CPU Register Inspection

Observed project values:

| Register | Value | Interpretation |
|---|---|---|
| PC | `0x08006AFE` | Inside main Flash |
| SP/MSP | `0x20000FF8` | Inside SRAM |
| LR | `0x0800333B` | Flash-region return address with Thumb bit |

Together with the vector-table read, these values showed that firmware was present before the erase.

## 8. RDP / Readout Protection

Command:

```text
Mem32 0x1FFFF800, 4
```

Recorded output:

```text
1FFFF800 = 00FF55AA 00FF00FF 00FF00FF 00FF00FF
```

For STM32F302x6/x8, the first byte at the option-byte base is the RDP byte. The displayed word:

```text
0x00FF55AA
```

corresponds to increasing memory bytes:

```text
AA 55 FF 00
```

Therefore:

```text
RDP = 0xAA
RDP Level = Level 0
```

The RDP byte interpretation follows the STM32F302 option-byte layout described in ST reference documentation.

## 9. Firmware Presence / Vector Table

Command:

```text
Mem32 0x08000000, 16
```

Recorded beginning:

```text
08000000 = 20003EF8 0800131D ...
```

`0x20003EF8` is consistent with an initial stack pointer inside the 16 KiB SRAM region.

`0x0800131D` lies in the main Flash region and has the Thumb-state bit set, consistent with a reset-handler address.

## 10. Firmware Extraction

Command:

```text
SaveBin D:\flash_backup.bin, 0x08000000, 0x10000
```

The extraction covered:

```text
Start = 0x08000000
Size  = 0x10000
       = 65,536 bytes
       = 64 KiB
```

This matches the full main Flash capacity documented for the STM32F302K8.

## 11. Backup Identity

| Property | Recorded value |
|---|---|
| Filename | `flash_backup.bin` |
| Size | 65,536 bytes |
| SHA-256 | `4fe305311a39fc12cdbbfa30867817561d9cc0e79b808f8fea94856225c08217` |
| Initial stack pointer | `0x20003EF8` |
| Reset handler | `0x0800131D` |
| Third vector entry | `0x080011E5` |
| Fourth vector entry | `0x080011E7` |

First 32 bytes recorded for artifact identification:

```text
f8 3e 00 20 1d 13 00 08 e5 11 00 08
e7 11 00 08 e7 11 00 08 e7 11 00 08
e7 11 00 08 00 00 00 00
```

The raw firmware binary is intentionally excluded from this repository because the associated firmware is internship/company-owned material.

## 12. Flash Erase

Command:

```text
erase
```

Recorded output:

```text
Erasing device...
Erasing done.
```

Beginning of Flash after erase:

```text
Mem32 0x08000000, 8

08000000 = FFFFFFFF FFFFFFFF FFFFFFFF FFFFFFFF
08000010 = FFFFFFFF FFFFFFFF FFFFFFFF FFFFFFFF
```

The inspected region returned `0xFFFFFFFF`, consistent with erased Flash.

## 13. Firmware Restoration

The retained project history records the following intended recovery sequence:

```text
loadbin D:\flash_backup.bin, 0x08000000
verifybin D:\flash_backup.bin, 0x08000000
mem32 0x08000000, 8
r
g
```

The intended order is:

```text
Program backup
      ↓
Verify programmed memory
      ↓
Inspect vector table
      ↓
Reset
      ↓
Run
```

The historical transcript does not contain a successful `LoadBin`/`VerifyBin` terminal output. Therefore the restoration is documented as project-history evidence, while a successful post-restore verification is not independently claimed.

## 14. Verification Nuance

An earlier `VerifyBin` attempt reported a mismatch at `0x08000000`. After erase, the same region returned `0xFF`, which is expected for erased Flash.

Therefore the earlier mismatch cannot, from the retained transcript alone, establish that the backup was corrupt.

The controlled workflow is:

```text
BACKUP → ERASE → LOADBIN → VERIFYBIN → RESET/RUN
```

## 15. Evidence Matrix

| Item | Evidence | Status |
|---|---|---|
| STM32F302K8 target | J-Link session + project record | Direct |
| SWD works | DAP/SW-DP/Cortex-M4 discovery | Direct |
| Firmware present | PC/vector table + registers | Direct |
| RDP Level 0 | `0x1FFF F800` first byte = `0xAA` | Direct |
| 64 KiB backup | `SaveBin` output + retained artifact metadata | Direct |
| Backup matches target | Matching vector-table values | Strong direct correlation |
| Erase succeeded | Erase output + `0xFFFFFFFF` readback | Direct |
| Firmware restored | Project history | Historical evidence |
| Verify after restore passed | No retained success log | Not independently proven |
| Exact KORE-side pin numbers | Not retained/legible | Not specified |

## 16. Separation from Later Cube/KORE STM32H753VI Work

This investigation concerns:

- **Target:** Here4 GPS
- **MCU:** STM32F302K8
- **Core:** Cortex-M4
- **Purpose:** Here4 firmware extraction/recovery

It is separate from the later Cube Orange+ / STM32H753VI investigation.

The later KORE J20 connector mapping must not be copied into this Here4 task without independent evidence.

## 17. References

1. STMicroelectronics — STM32F302K8
2. STMicroelectronics — STM32F302x6/x8 datasheet
3. STMicroelectronics — RM0365 STM32F302 reference manual
4. SEGGER — J-Link interface description
5. CubePilot — KORE Carrier Board

The source documentation contains the corresponding official URLs.

## 18. Technical Conclusion

The investigation established SWD access to the Here4's STM32F302K8, detected its Cortex-M4 debug infrastructure, inspected CPU state, and determined RDP Level 0.

The existing firmware was read from `0x08000000` and saved as a 64 KiB raw binary. The retained artifact metadata identifies the backup with SHA-256:

```text
4fe305311a39fc12cdbbfa30867817561d9cc0e79b808f8fea94856225c08217
```

The target Flash was subsequently erased, and the inspected region returned `0xFFFFFFFF`. The saved backup was later used for recovery according to project history. The repository deliberately distinguishes retained terminal evidence from activities recorded only in project history.
