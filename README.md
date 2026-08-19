# Here4 GPS — Firmware Extraction & Recovery

Technical documentation of a low-level firmware-debugging investigation performed on a **Here4 GPS module** during internship research at the **Center of Excellence (CoE) in Complex & Nonlinear Dynamical Systems (CNDS), VJTI**.

The work focused on establishing **ARM Serial Wire Debug (SWD)** access to the Here4's **STM32F302K8**, determining its readout-protection state, inspecting Flash/vector-table contents, creating a controlled 64 KiB firmware backup, and documenting the erase/recovery workflow.

> **Ownership & attribution**
>
> This repository is a technical record of work performed during the internship at **CoE-CNDS, VJTI**. The associated hardware, firmware, proprietary materials, experimental data, and project intellectual property remain the property of **CoE-CNDS / VJTI**, unless otherwise stated. This repository should not be interpreted as a transfer or claim of ownership by the author.
>
> The raw firmware image is **not included** in this public repository. Its identity is documented by metadata and a SHA-256 hash only.

## Scope

- Here4 GPS hardware investigation
- STM32F302K8 target identification
- SWD communication using SEGGER J-Link
- Debug Port / SW-DP discovery
- CPU register and vector-table inspection
- STM32 readout-protection (RDP) analysis
- Raw Flash extraction and backup validation
- Controlled Flash erase
- Firmware restoration workflow
- Evidence-based documentation of limitations and verification status

## Hardware Architecture

```text
Here4 GPS Module
       │
       ▼
STM32F302K8
       │
       ▼
Accessible debug/test points
       │
       │  SWDIO / SWCLK / NRST / VTref / GND
       ▼
SEGGER J-Link
       │
       │ USB
       ▼
PC / J-Link Commander
```

The **KORE hardware was used to provide physical access to the debugging interface** in the setup. It was not the firmware target for this Here4 extraction.

## Target MCU

| Parameter | Value |
|---|---|
| MCU | STM32F302K8 |
| CPU | Arm Cortex-M4 with FPU |
| Maximum CPU frequency | 72 MHz |
| Main Flash | 64 KiB |
| SRAM | 16 KiB |
| Main Flash | `0x08000000 – 0x0800FFFF` |
| SWDIO | PA13 |
| SWCLK | PA14 |
| Reset | NRST, active-low |

## SWD / J-Link Interface

| Signal | STM32F302K8 | J-Link ARM 20-pin |
|---|---|---:|
| VTref | Target VDD/reference | 1 |
| SWDIO | PA13 | 7 |
| SWCLK | PA14 | 9 |
| NRST | NRST | 15 |
| GND | VSS/GND | 4, 6, 8, 10, 12, 14, 16, 18, 20 |

**Note:** The exact KORE-to-Here4 debug-port pad/connector numbering was not retained clearly enough to state reliably, so unsupported KORE pin numbers are intentionally omitted.

## Physical Evidence

Only a small subset of the available photographs is included to keep the repository focused:

- Complete J-Link / microscope debugging setup
- PCB-level probing
- J-Link connector and wiring
- Microscope-based board inspection
- MCU/component close-up

See [`images/`](images/).

## Debug Connection

The initial connection attempts produced errors such as:

```text
Failed to initialize DAP.
Can not attach to CPU.
Trying connect under reset.
Connecting to CPU via connect under reset failed.
```

The investigation checked the physical SWD path, target reference voltage, common ground, SWDIO, SWCLK, reset, target selection, and connection conditions.

A successful debug connection was subsequently established.

### Successful SW-DP discovery

```text
DAP initialized successfully.
Found SW-DP with ID 0x2BA01477
DPIDR: 0x2BA01477
AP[0]: AHB-AP
Core found
Found Cortex-M4 r0p1
Cortex-M4 identified.
```

This established communication with the STM32 debug infrastructure.

## RDP / Readout Protection

The option-byte region was inspected using:

```text
Mem32 0x1FFFF800, 4
1FFFF800 = 00FF55AA 00FF00FF 00FF00FF 00FF00FF
```

For the STM32F302x6/x8 option-byte layout, the first byte at `0x1FFF F800` is the RDP byte. Because the displayed 32-bit value is represented in little-endian memory order:

```text
0x00FF55AA
     ↓
AA 55 FF 00
```

Therefore:

```text
RDP = 0xAA
RDP Level = Level 0
```

This means readout protection was disabled for the observed target state.

## Firmware Presence

The beginning of Flash was inspected:

```text
Mem32 0x08000000, 16
08000000 = 20003EF8 0800131D ...
```

The first value is consistent with an initial stack pointer inside the 16 KiB SRAM region. The second is a Flash-region address with the Thumb-state bit set, consistent with a reset-handler entry.

## Firmware Extraction

The recorded extraction command was:

```text
SaveBin D:\flash_backup.bin, 0x08000000, 0x10000
```

The resulting backup was:

- **Filename:** `flash_backup.bin`
- **Size:** 65,536 bytes (64 KiB)
- **Start address:** `0x08000000`
- **End address:** `0x0800FFFF`
- **SHA-256:** `4fe305311a39fc12cdbbfa30867817561d9cc0e79b808f8fea94856225c08217`

The raw binary itself is intentionally **not published in this repository**.

## Flash Erase & Recovery

The documented sequence was:

```text
erase

loadbin D:\flash_backup.bin, 0x08000000
verifybin D:\flash_backup.bin, 0x08000000

mem32 0x08000000, 8
r
g
```

After the erase, the beginning of Flash returned:

```text
08000000 = FFFFFFFF FFFFFFFF FFFFFFFF FFFFFFFF
08000010 = FFFFFFFF FFFFFFFF FFFFFFFF FFFFFFFF
```

This confirms the inspected Flash region was in the erased state.

The retained project history records subsequent use of the backup for restoration. A successful post-restore `VerifyBin` terminal transcript was not retained, so this repository does **not** claim an independently proven verification pass.

## Verification Boundary

An earlier verification attempt reported a mismatch at `0x08000000`. Because later inspection showed the address as `0xFF` after erase, that earlier result cannot by itself be used to judge backup integrity.

The technically correct recovery sequence is:

```text
BACKUP
  ↓
ERASE
  ↓
LOADBIN
  ↓
VERIFYBIN
  ↓
RESET / RUN
```

The documentation intentionally distinguishes directly retained terminal evidence from project-history/user-confirmed activity.

## Evidence Summary

| Finding | Evidence | Status |
|---|---|---|
| STM32F302K8 target | Debug session / project record | Direct |
| SWD communication | SW-DP + Cortex-M4 discovery | Direct |
| Firmware present | Registers + vector table | Direct |
| RDP Level 0 | Option-byte read | Direct |
| 64 KiB backup | `SaveBin` output + retained artifact metadata | Direct |
| Backup/vector correlation | Matching vector-table values | Strong correlation |
| Flash erase | Erase output + `0xFFFFFFFF` readback | Direct |
| Firmware restoration | Project history | Historical evidence |
| Successful post-restore VerifyBin | No retained success log | Not independently proven |
| Exact KORE-side pin numbers | Not sufficiently preserved | Not specified |

## Separation from Cube Orange+ Work

This repository concerns the **Here4 / STM32F302K8** investigation.

It must not be mixed with the later **Cube Orange+ / STM32H753VI** firmware investigation. In particular, KORE J20 pin mappings from the Cube FMU investigation are not presented as Here4 debug-port mappings.

## Reproducible Workflow

1. Identify the target MCU as STM32F302K8.
2. Establish physical access to the debug interface.
3. Connect VTref, SWDIO, SWCLK, NRST and GND.
4. Configure J-Link for STM32F302K8 and SWD.
5. Establish the debug connection.
6. Halt and inspect CPU/debug state.
7. Inspect the RDP option byte.
8. Inspect the Flash vector table.
9. Create a complete 64 KiB backup.
10. Preserve the backup before any destructive operation.
11. Erase the target Flash when required.
12. Read back the erased region.
13. Restore the retained backup.
14. Run `VerifyBin`.
15. Reset and run the target.

## References

- STMicroelectronics — STM32F302K8
- STMicroelectronics — STM32F302x6/x8 datasheet
- STMicroelectronics — RM0365 STM32F302 reference manual
- SEGGER — J-Link interface description
- CubePilot — KORE Carrier Board

Detailed references and the full technical record are in [`docs/firmware-extraction-report.md`](docs/firmware-extraction-report.md).

---

**Internship context:** CoE-CNDS, VJTI  
**Focus:** Embedded firmware analysis • SWD • JTAG/SWD debugging • STM32 • Flash extraction & recovery
