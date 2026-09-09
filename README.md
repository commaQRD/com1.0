# COM 1.0

Architecture notes for the **Xiaomi dijun** (O1) factory download stack, as observed from FDT v6.2.0, factory XML, GPT layouts, and signed boot images.

This repository describes **what the pipeline is and why it is structured that way**. It does not describe how to operate a downloader, how to construct host traffic, or how to flash a device.

## Scope

| In scope | Out of scope |
|---|---|
| Boot stages and which payload belongs where | Host commands, scripts, or packet recipes |
| Why bootrom download uses image IDs instead of flash addresses | Procedures to send, recover, unlock, or bypass |
| How lock state, eFuse, and RPMB relate conceptually | Exploits, OEM tokens, or RPMB programming |

## Platform

Observed identity from `boot.img` and modem metadata:

- Product / device: **dijun**
- AP: **MT6980** (Dimensity 9300 class)
- Modem: MediaTek **T800 / MOLY NR16 MD800** (`XIAOMI_2102MP3_T800`)
- OS fingerprint: `Xiaomi/dijun/dijun:15/.../OS2.0.109.0.VODCNDM`
- Kernel: Android 15 GKI `6.6.30`, 4K pages, boot image header v4

The early boot *names* look Kirin-like (`xloader`, `xrse`, `uefi`). The SoC and modem are MediaTek. Treat this as a Xiaomi custom boot chain on MTK silicon, not a Hisilicon chip.

USB personality observed in two phases:

1. Factory download enumeration (Xiaomi VID `2717`)
2. After UEFI is up: standard fastboot (`18D1:D00D`)

## Two-phase model

```
BootROM
  │  phase A — early loader (factory download personality)
  ├─ image-id payloads in memory / controller RAM
  ├─ UEFI becomes the running environment
  │  phase B — fastboot (UEFI)
  └─ logical partitions (*_ab) written through GPT on UFS
```

**Phase A** is not Android and not fastboot. It exists so DRAM, the security island, BL2, and UEFI can exist at all. The host names files; the device maps an **image id** to an internal destination. There is no flash address in that contract.

**Phase B** is ordinary Android fastboot once `uefi` is running. Logical names such as `xloader_ab` and `boot_ab` are resolved by UEFI against GPT. This is the Qualcomm **ABL** analogue on this platform: `uefi`, not `boot.img`.

Factory XML therefore has two tables: a bootrom table (id + path) and a later fastboot table (partition name + path). They are not interchangeable. In particular, `sec_xloader_usb.img` is a download-stage loader; `sec_xloader.img` is the copy that belongs in the LU0/LU1 `xloader` slots.

## Bootrom payload set (observed)

Order is significant because each stage is a precondition for the next (DRAM before BL2, BL2 before UEFI, UEFI before GPT use).

| Order | Role | Image id | Typical artifact |
|---|---|---|---|
| 1 | USB-capable xloader | `0x3` | `sec_xloader_usb.img` |
| 2 | Runtime security island | `0x4` | `sec_xrse_fw.img` |
| 3 | Storage-prepare complete | misc `0x5` | *command, not a file* |
| 4 | DDR parameters | `0xa` | `sec_ddr_para.img` |
| 5 | BL2 | `0x5` | `sec_bl2.bin` |
| 6 | UEFI (fastboot app) | `0x6` | `sec_uefi.img` |
| 7 | CPU power / clock helper | `0x7` | `sec_xctrl_cpu.img` |
| 8 | DDR helper | `0xb` | `sec_xctrl_ddr.img` |
| 9 | Low-power helper | `0xc` | `sec_lpctrl.bin` |
| 10 | GPT bundle | `0x8` | `gpt/dijun/gpt.img` |

A commented non-secboot pair exists in factory XML (`*_xr` xloader/xrse). Same ids, different certificates. That is a policy fork, not a different protocol.

Cold boot from UFS does **not** use the USB xloader. It uses LU0/LU1 `xloader_a`/`xloader_b` plus LU2 `bl2` → `bl31` → `uefi` → `boot`.

## Principle of the early download channel

Kirin-style factory download often puts a **storage address** in the host frame. This stack does not.

- The host announces *which object* (`image_id`) and *how large it is*.
- The running BootROM / xloader already knows where that object belongs (SRAM, DRAM window, or a controller mailbox).
- Integrity is a session checksum over the frame, plus later **signed image headers** on the payloads themselves (`0x9ABCDEF0` / `0x12345678`, certificates, `NVCOUNTER`).
- A framed session exists (open → object info → data → close). An auth message type exists in the protocol enum; that is device-side factory authentication, not a cloud login.
- A misc namespace carries control codes (storage-prepare done; a commented test-mode code). Those are state transitions, not partitions.

DDR parameters are parsed specially because they are a configuration blob for the memory controller, not a generic code image. UFS geometry (`ufs_param`, optional FFU) is a *device descriptor* problem: LUN count and size are provisioned on the storage controller **before** GPT is meaningful. GPT cannot invent LUNs that the UFS descriptor did not create.

Once UEFI enumerates as fastboot, the early channel is finished. Subsequent writes use partition names, A/B slots, anti-rollback (`getvar anti` vs image `NVCOUNTER`), and optional CRC list channels. That is a different interpreter on the same USB cable.

## Storage geometry (why four LUNs)

UFS here is not one disk with one GPT. Factory layout uses **four LUNs**, 4096-byte sectors:

| LUN | Role |
|---|---|
| LU0 | A-slot early boot (`auth_cert`, `rom_ext`, `xloader`, `xrse`) |
| LU1 | B-slot copy of the same |
| LU2 | `bl2`, `bl31`, `uefi`, Android slots, `super`, `userdata` |
| LU3 | NV / persist / protect / modem-adjacent small volumes |

Partition offsets in GPT are only valid **inside** a LUN that already has that size. That is what “hard-coded LUN geometry” means: xloader looks for a few-megabyte LU0, not a single large user LUN.

## Trust layers (conceptual)

These are independent stores. Confusing them is the usual source of wrong conclusions.

1. **SoC eFuse / OTP** — lives in the AP. Secure-boot enable, public-key hash, some rollback. Survives a storage swap.
2. **Signed `sec_*` images** — header + cert + `NVCOUNTER`. BootROM/xloader/xrse/uefi refuse a body that does not match policy.
3. **UFS RPMB** — lock flags, some rollback indexes, TEE-bound secrets. Bound to *this* UFS after factory provision. A blank disk is unprovisioned, not “unlocked”.
4. **AVB vbmeta*** — Android verified boot descriptors. Checked when UEFI launches Android, not during BootROM USB.
5. **FRP / account flags** — `frp`, `nvmem`, `persist`. Not the bootloader lock bit.

Flashing lock is enforced by **UEFI** (and earlier by xrse on the loader chain). BootROM download does not consult the RPMB lock flag. That is why an engineering unit with an engineering certificate set can re-enter the early channel after a software lock, while a production unit fused to production keys will not accept engineering payloads. It is a key/policy difference, not a missing lock.

`sec_uefi.img` on this drop is a signed, high-entropy body. Fastboot verbs are not sitting in plaintext in that file. What factory scripts actually use on the wire after UEFI is up is the ordinary set: `getvar` (`product`, `anti`, `crc`, `flash-done`), `download`/`flash`/`erase` of logical names, `set_active`, `reboot`.

## Mapping to Qualcomm / typical MTK

| Qualcomm | Typical MTK | dijun / O1 |
|---|---|---|
| PBL | BootROM | BootROM |
| XBL | preloader / LK prelude | `xloader` + `xrse` |
| — | — | `bl2` / `bl31` (TF-A style) |
| **ABL** | LK or UEFI | **`uefi`** |
| `boot` | `boot` | `boot` (GKI only; ramdisk split) |

There is no `abl` or `lk` partition in this GPT.

## Documents

- [docs/bootrom-channel.md](docs/bootrom-channel.md) — early-channel principle in more detail
- [docs/platform.md](docs/platform.md) — images, LUNs, and what each stage is for

## Disclaimer

Notes for reading factory artifacts. Not a flashing guide, not a security bypass, not a license to operate on devices you do not own.
