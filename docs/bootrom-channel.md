# Early download channel (principle)

This note explains **why** the bootrom-stage transfer looks the way it does. It does not specify host-side operations, frame construction, or tooling.

## What problem this channel solves

At reset, DRAM may be off, GPT is not trusted yet, and Android fastboot does not exist. Something still has to:

1. Bring up a USB (or UART) personality the factory host can see.
2. Accept a short, ordered list of **named objects**.
3. Hand off to UEFI, which then speaks fastboot against UFS.

The object list is small on purpose: only the pieces required to reach a policy-enforcing environment (`uefi`) plus a GPT so later writes have names.

## Object identity vs storage address

Two factory styles:

- **Address-bearing** (classic Kirin download): the host frame includes a destination address. The loader is a dumb DMA into that window.
- **Id-bearing** (this stack): the host frame names an `image_id`. The loader has a built-in table: id → region, maximum size, and which parser to run (code vs DDR blob vs GPT vs misc command).

Ids observed on dijun:

- code/config images: `0x3` xloader (USB variant), `0x4` xrse, `0xa` ddr_param, `0x5` bl2, `0x6` uefi, `0x7`/`0xb`/`0xc` helpers, `0x8` GPT
- misc: `0x5` as a *command* meaning storage-prepare complete (same numeric value as bl2’s id, different opcode class)

Because the destination is not in the host frame, a host that “sends the same bytes to a different address” is a category error. Changing where a blob lands is a firmware-table change, not a protocol-field change.

## Session shape

Conceptually the session is:

```
open connection
  for each object in the XML bootrom table:
      announce object (id + length)
      stream body
  optional control (misc)
close
```

Framing exists so the stream can be delimited and integrity-checked at the transport layer. Payloads additionally carry their own signed headers. Transport checksums catch cable noise; image signatures catch substitution. They answer different threats.

An authentication message type exists in the protocol enum. That is a factory handshake with the device, not an account login in the CLI that was inspected.

UART appears in factory XML (`bootrom tool=uart`, 115200) as the *logical* channel name for this stage. Enumeration on the wire is still a USB device in the observed setup; treat “uart” as the FDT stage name, not a requirement that the host use a 16550.

## Why the USB xloader is a different file

`sec_xloader_usb.img` is built to *be* the download-stage resident. `sec_xloader.img` is built to live in LU0/LU1 and start from storage. Same family, different I/O path. Putting the storage image in the early channel (or the USB image into the GPT slot) breaks the id/slot contract even if both parse as `0x9ABCDEF0` containers.

## What this channel is not

- Not fastboot. No `flash:`, no sparse, no `*_ab` mapping.
- Not a UFS programmer by itself. LUN provision (`ufs_param` / FFU) is a separate descriptor protocol on the storage side. GPT (`id 0x8`) only makes sense after LUNs exist.
- Not an unlock mechanism. BootROM does not read the RPMB lock flag. Lock policy starts when xrse/uefi run.
- Not checkm30. checkm30 is a 2021 HiSilicon BootROM USB issue. This product is MT6980 with a Xiaomi-shaped loader list. Similar *layer* (pre-UEFI USB), different ROM.

## Handoff

After UEFI is resident and helpers/GPT are in place, the device drops the download personality and enumerates as fastboot. From that point, partition names and A/B policy apply. The early channel has nothing left to say.
