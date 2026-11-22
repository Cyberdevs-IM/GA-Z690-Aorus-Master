# Aquantia AQC113 sleep/wake fix (OpenCore, macOS 15)

Fixes a Hackintosh that won't stay asleep because the onboard Aquantia AQC113 10GbE NIC keeps waking it.

## Symptoms

- About 8–35 s after sleep the machine wakes on its own. The displays stay black (DarkWake), then it goes back into Maintenance Sleep, and this repeats roughly every 80 s.
- `pmset -g log` shows `due to /Network`, `/HID Activity` or an empty reason; `kern.wakereason` shows `PXSX` or nothing.
- With the Ethernet cable **unplugged**, the machine stays asleep.
- The kernel log shows:
  ```
  AppleEthernetAquantiaAqtion113 [en0] check wake success 1, reason 3 (Link)
  ```
  `ioreg` shows `LastWake Reason = 3`.

## Root cause

This isn't an ACPI problem. Before sleep, Apple's driver (`AppleEthernetAquantiaAqtion113::handleEnableNicProxy`) puts the NIC firmware into sleep-proxy mode. It writes a hardcoded wake mask (`0x44`) to the firmware register `0x12028`:

| bit | meaning (Aquantia `wake_on_lan_s`) |
|-----|-------------------------------------|
| 0 | magic packet |
| 1 | pattern |
| **2** | **link up** ← the cause |
| 3 | link down |
| 4 | ping |
| 5 | timer |

`0x44` = bit 6 + bit 2. Bit 2 (wake on **link up**) makes the firmware wake the system whenever the link renegotiates after sleep. Switches and other link partners nearly always renegotiate, so the machine wakes every time.

That's why none of the usual fixes work:

| Tried | Result |
|-------|--------|
| `_PRW`/GPRW patches, `RPxx` TPRW SSDTs | ACPI GPE path closed, still wakes |
| Clearing PCIe PME_En on NIC + root port in `_PTS` | still wakes |
| `PCIEXP_WAKE_DIS` in PM1_EN, disarming WAKE# GPIO | still wakes |
| `pmset womp 0`, BIOS ErP | still wakes |
| `ifconfig en0 down` before sleep | still wakes |

The wake is generated in the NIC firmware, which nothing at the OS or ACPI level can reach.

## Fix: one OpenCore kernel patch

Clear bit 2 from the immediate operand (`0x44` → `0x40`), so the firmware is no longer told to wake on link up. Magic packet and the other wake sources are unchanged.

`config.plist` → `Kernel` → `Patch`:

| Key | Value |
|-----|-------|
| Arch | `x86_64` |
| Comment | `Aquantia AQC113: disable wake on link-up` |
| Enabled | `true` |
| Identifier | `com.apple.driver.AppleEthernetAquantiaAqtion` |
| Find | `8D144183C24441891424` |
| Replace | `8D144183C24041891424` |
| Count | `1` |
| MinKernel | `24.0.0` |
| MaxKernel | `24.99.99` |
| Base / Mask / ReplaceMask | *(empty)* |

```
lea  edx, [rcx+rax*2]
add  edx, 0x44      ; 83 C2 44  →  83 C2 40
mov  [r12], edx
```

Verified on macOS 15.8.2 (Darwin 24). The pattern occurs only once in the kext. **After every macOS update, check the bytes are still there.** If OpenCore reports the patch as not applied, the code has changed and the patch has to be found again.

### Verify

```bash
pmset sleepnow
# wait several minutes, wake with keyboard/mouse
pmset -g log | grep -E " (Sleep|Wake) " | tail -4
ioreg -lw0 | grep -i '"LastWake'      # Reason should no longer be 3
```

The machine should stay asleep until you press a key or move the mouse.

## Related notes

- **Leave ErP disabled.** With ErP enabled, S3 powered the board fully off on this system.
- On this board the Aquantia `device-id`/`compatible` spoof in DeviceProperties (→ `94C0`) is required. Removing it makes the card disappear.
- A separate, intermittent resume failure (`Failure during sleep: 0x1F EFI/Bootrom`) on later cycles turned out not to be the NIC: it also happened with the NIC disabled. Here it was the **RAM overclock (XMP 6200)**. At 4800 MHz (XMP off), 12/12 sleep cycles passed. If repeated sleep cycles hang, test with XMP off.

## Tested hardware

- Intel i9-12900K, GA-Z690 Aorus Master motherboard with onboard AQC113C 10GbE
- AMD RX 6800 XT
- OpenCore, macOS Sequoia 15.8.2
