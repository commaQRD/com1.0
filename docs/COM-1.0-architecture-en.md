# COM 1.0 — Factory Download Architecture (English)

Xiaomi **dijun** (O1) · FDT v6.2.0. Principle only: what the pipeline is, not how to operate a downloader.

English counterpart of the earlier Chinese analysis (FDT, bootrom object list, two-phase model, boot chain, storage and trust layers).

## 1. Scope

| In scope | Out of scope |
|---|---|
| Boot stages and which payload belongs where | Host commands, scripts, or packet recipes |
| Why early download uses image IDs instead of flash addresses | Procedures to send, recover, unlock, or bypass |
| How lock state, eFuse, and RPMB relate conceptually | Exploits, OEM tokens, or RPMB programming |

## 2. Platform

| Field | Observed value |
|---|---|
| Product / device | dijun |
| AP | MT6980 (Dimensity 9300 class) |
| Modem | MediaTek T800 / MOLY NR16 MD800 (`XIAOMI_2102MP3_T800`) |
| OS fingerprint | `Xiaomi/dijun/dijun:15/…/OS2.0.109.0.VODCNDM` |
| Kernel | Android 15 GKI 6.6.30, 4K pages, boot header v4 |
| Download USB | VID `2717` PID `FF88` REV `0100` |
| Fastboot USB | `18D1:D00D` after UEFI is up |

Early-boot names look Kirin-like (`xloader`, `xrse`, `uefi`). The SoC and modem are MediaTek. Treat this as a Xiaomi custom boot chain on MTK silicon, not a Hisilicon chip.

## 3. Two-phase model

**Phase A** is BootROM / factory-download personality. DRAM, the security island, BL2, and UEFI do not exist yet. The host names files; the device maps an **image id** to an internal destination. There is no flash address in that contract.

**Phase B** is ordinary Android fastboot once `uefi` is running. Logical names such as `xloader_ab` and `boot_ab` are resolved by UEFI against GPT. This is the Qualcomm **ABL** analogue: `uefi`, not `boot.img`.

Factory XML therefore has two tables: a bootrom table (id + path) and a later fastboot table (partition name + path). They are not interchangeable. `sec_xloader_usb.img` is the download-stage loader; `sec_xloader.img` is the copy that belongs in the LU0/LU1 `xloader` slots.

## 4. Bootrom object list

Order is significant because each stage is a precondition for the next.

| Order | Role | id | Typical artifact |
|---|---|---|---|
| 1 | USB-capable xloader | `0x3` | `sec_xloader_usb.img` |
| 2 | Runtime security island | `0x4` | `sec_xrse_fw.img` |
| 3 | Storage-prepare complete | misc `0x5` | command, not a file |
| 4 | DDR parameters | `0xa` | `sec_ddr_para.img` (FDT splits into segments) |
| 5 | BL2 | `0x5` | `sec_bl2.bin` |
| 6 | UEFI / fastboot app | `0x6` | `sec_uefi.img` |
| 7 | CPU helper | `0x7` | `sec_xctrl_cpu.img` |
| 8 | DDR helper | `0xb` | `sec_xctrl_ddr.img` |
| 9 | Low-power helper | `0xc` | `sec_lpctrl.bin` |
| 10 | GPT bundle | `0x8` | `gpt/dijun/gpt.img` |

A commented non-secboot pair exists in factory XML (`*_xr` xloader/xrse). Same ids, different certificates.

Cold boot from UFS does **not** use the USB xloader. It uses LU0/LU1 `xloader_a`/`xloader_b` plus LU2 `bl2` → `bl31` → `uefi` → `boot`.

## 5. Early-channel principle

Kirin-style factory download often puts a **storage address** in the host frame. This stack does not. The host announces *which object* (`image_id`) and *how large it is*. The running BootROM / xloader already knows where that object belongs.

Integrity is a session checksum over the frame, plus later **signed image headers** on the payloads (`0x9ABCDEF0` / `0x12345678`, certificates, `NVCOUNTER`). Transport checksums catch cable noise; image signatures catch substitution.

A framed session exists (open → object info → data → close). An auth message type exists in the protocol enum; that is device-side factory authentication, not a cloud login.

FDT presents the early stage as `tool=uart`, baud 115200. Enumeration on the wire is still USB. Treat “uart” as the FDT stage name.

After DDR parameters are accepted, xloader relocates into DRAM and the USB PHY is rebuilt. The host COM handle often dies at that moment. Sessions that pin a fixed COM port then fail at the bl2 handshake (`send conn start fail`, `COM offline`). The next object is still bl2; the cable session is not.

## 6. Storage geometry

Four LUNs, 4096-byte sectors.

| LUN | Role |
|---|---|
| LU0 | A-slot early boot (`auth_cert`, `rom_ext`, `xloader`, `xrse`) |
| LU1 | B-slot copy of the same |
| LU2 | `bl2`, `bl31`, `uefi`, Android slots, `super`, `userdata` |
| LU3 | NV / persist / protect / modem-adjacent volumes |

Hard-coded LUN geometry means xloader looks for a few-megabyte LU0, not a single large user LUN. GPT cannot invent LUNs the UFS descriptor did not create. A replacement package is unprovisioned, not “unlocked”.

## 7. Trust layers

| Layer | Where | What it governs |
|---|---|---|
| SoC eFuse / OTP | AP die | Secure-boot enable, key hash, some rollback. Survives a storage swap. |
| Signed `sec_*` images | File header + cert | BootROM/xloader/xrse/uefi refuse a body that fails policy. `NVCOUNTER` on UEFI was 10 in the drop examined. |
| UFS RPMB | This UFS package | Lock flags, some rollback, TEE-bound secrets. Bound after factory provision. |
| AVB vbmeta* | LU2 | Checked when UEFI launches Android, not during BootROM USB. |
| FRP / account flags | `frp`, `nvmem`, `persist` | Not the bootloader lock bit. |

Software lock is enforced by **UEFI** (and earlier by xrse). BootROM download does not consult the RPMB lock flag. An engineering unit with an engineering certificate set can re-enter the early channel after a software lock; a production unit fused to production keys will not accept engineering payloads. That is a key/policy difference, not a missing lock.

## 8. Mapping to Qualcomm / typical MTK

| Qualcomm | Typical MTK | dijun / O1 |
|---|---|---|
| PBL | BootROM | BootROM |
| XBL | preloader / LK prelude | `xloader` + `xrse` |
| — | — | `bl2` / `bl31` (TF-A style) |
| **ABL** | LK or UEFI | **`uefi`** |
| `boot` | `boot` | `boot` (GKI; ramdisk split) |

No `abl` or `lk` partition in this GPT.

## 9. What this channel is not

- Not fastboot. No `flash:`, no sparse, no `*_ab` mapping during phase A.
- Not a UFS programmer by itself. LUN provision is a separate descriptor protocol. GPT (`id 0x8`) only makes sense after LUNs exist.
- Not Qualcomm 9008. That path is `05C6:9008` / Sahara / Firehose. This product is `2717:FF88` / FDT.
- Not checkm30. That was a 2021 HiSilicon BootROM USB issue. This product is MT6980 with a Xiaomi-shaped loader list.

## Disclaimer

Notes for reading factory artifacts on hardware the reader is authorized to work on. Not a flashing guide, not a security bypass.
