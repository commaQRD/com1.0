# Platform notes (dijun / O1)

## Images that belong to early boot vs Android

| Artifact | Stage | Role |
|---|---|---|
| `sec_xloader_usb.img` | bootrom channel | USB-resident first loader |
| `sec_xloader.img` | UFS LU0/LU1 | Storage first loader |
| `sec_xrse_fw.img` | both (id `0x4` then slots) | Security island |
| `sec_ddr_para.img` | bootrom | Memory controller config |
| `sec_bl2.bin` / `bl31` | bootrom then LU2 | TF-A style BL2 / EL3 |
| `sec_uefi.img` | bootrom then LU2 `uefi_a/b` | Fastboot + Android launch (ABL analogue) |
| `sec_xctrl_*.img`, `sec_lpctrl.bin` | bootrom then LU2 | Power / DDR / LP helpers |
| `gpt.img` + `LU*_layout` | bootrom id `0x8` | Four GPT copies |
| `sec_tee.bin`, `xhee`, `xspm` | fastboot | TEE stack |
| `boot.img` | fastboot `boot_ab` | GKI 6.6.30, ramdisk empty (v4) |
| `boot_logo.img` | LU2 | UEFI firmware volume (`_FVH`), 6 MiB body |
| modem / MDDB | not bootrom | Catcher DB for MT6980 T800; not a bootrom object |

`boot.img` fingerprint: `Xiaomi/dijun/dijun:15/AP3A.240905.015.A2/OS2.0.109.0.VODCNDM:user/release-keys`. Security patch string in the same image: `2025-03-01`.

## UFS LUN map (factory table)

Sectors are 4096 bytes.

- **LU0 / LU1** (~3 MiB each): `auth_cert`, `rom_ext`, `xloader`, `xrse` for slots A and B.
- **LU2** (~23 GiB in the engineering table): `bl2`, `bl31`, `uefi`, vbmeta*, boot/init_boot/vendor_boot, `super`, `userdata` (~10 GiB in that table — engineering size, not a claim about retail capacity), `frp`, `misc`.
- **LU3**: `nvmem`, `nvcfg`, `persist`, `protect_*`, nvram-class volumes.

A replacement UFS does not magically have this cut. Provisioning LUNs is a controller-descriptor step; GPT is a later step that only fills LUNs that already exist. RPMB is a third store on the same package and is not a GPT partition.

## Signed container

Early `sec_*` files share a header magic pair `0x9ABCDEF0` + `0x12345678`, a section table, DER certificates (example timestamp `240511...Z` on `sec_uefi.img`), and `NVCOUNTER` (value 10 on that UEFI). The body of `sec_uefi.img` is high entropy after the certs — not a plaintext EFI FV you can `strings` for fastboot verbs.

Factory fastboot scripts compare `getvar product` to `dijun` and `getvar anti` to that counter before writing.

## Boot graph

```
              +-----------------+
              |     BootROM     |
              +--------+--------+
                       |
          +------------+------------+
          | storage xloader (LU0)   |     USB xloader (download personality)
          +------------+------------+     +------------+
                       |                  | xrse, ddr  |
                       |                  | bl2, uefi  |
                       v                  +------+-----+
              +--------+--------+                |
              |  bl2  -> bl31   |<---------------+
              +--------+--------+
                       |
              +--------v--------+
              |      uefi       |  ← fastboot lives here
              +--------+--------+
                       |
         tee / xhee / xspm / xctrl
                       |
              boot + vendor_boot + init_boot
                       |
                    super
```
