# Driver for `USB\VID_2717&PID_FF88&REV_0100`

The FDT package does not ship an INF. Windows will show an unknown device.

This PID on dijun is the **factory download** gadget, not Android RNDIS/ADB. Do not install a random “Xiaomi RNDIS” package.

FDT talks to this stage as a **serial port** (`-port`, `--wait-for-com`). Bind **usbser** first. Use WinUSB only if the tool still cannot open a COM port.

## Recommended: Zadig (signed)

1. Plug the phone so Device Manager shows `VID_2717` `PID_FF88`.
2. Download [Zadig](https://zadig.akeo.ie/).
3. Options → List All Devices.
4. Select the Xiaomi `2717:FF88` device.
5. Target driver: **USB Serial (CDC)**.
6. Replace Driver.
7. Device Manager → Ports (COM & LPT) should show a COMx.

If FDT still cannot open it, repeat Zadig and choose **WinUSB** instead (device moves under Universal Serial Bus devices).

## INF in this folder (unsigned)

| File | Binds to |
|---|---|
| `xiaomi_ff88_usbser.inf` | `usbser.sys` → COM port |
| `xiaomi_ff88_winusb.inf` | `winusb.sys` → raw USB |

Windows 10/11 will reject unsigned INF unless test-signing is on, or you install from Device Manager → Update driver → Browse → Let me pick → Have Disk → this folder (and accept the unsigned warning if shown).

Do not use these INFs on `VID_18D1&PID_D00D` (fastboot). That interface uses the Google/Xiaomi Android WinUSB driver.
