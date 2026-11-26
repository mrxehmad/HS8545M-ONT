# ⚠️ HS8145C5 — Partial boot failure (POWER blinking, no Web UI / no WiFi)

**Status: NOT verified on HS8145C5 — real failure report, read before trying.**
Reported by the repo owner (Nov 2026) after applying the TranStar `getshell`
injection from this repo to a **Huawei HS8145C5**.

| | |
|---|---|
| Model | Huawei HS8145C5 (GPON, HiSilicon SD511x / Dopra Linux) |
| Firmware | `V5R019CxxSxxx` |
| Kernel | `3.10.53` |
| Module used | `getshell_3.10.53_R019.ko` |

## Symptoms after reboot

- **POWER LED blinks forever** — never goes solid (boot never completes).
- **Wired networking still works** — kernel, LAN switch and **telnet are alive**.
- **Web UI dead** — nothing on `192.168.100.1`.
- **WiFi dead** — SSIDs never come up.

This is a **soft brick, not a hard brick**. SDK init (`1.sdk_init.sh`) fails at
the injected-module stage; web/WiFi services start *after* that stage, so they
never launch. Telnet starts independently — **it is your lifeline.**

## How it was recovered (no reflash, no UART)

1. Connect a PC to a LAN port **by Ethernet cable** (WiFi is dead).
2. If DHCP is dead, set a static IP on the PC (e.g. `192.168.100.2/24`).
3. `telnet 192.168.100.1` and log in.
4. Undo the injection — restore the pre-exploit state:
   ```sh
   rm -rf /mnt/jffs2/TranStar
   # if you also edited hw_boardinfo, restore your backup:
   # cp /mnt/jffs2/hw_boardinfo.bak /mnt/jffs2/hw_boardinfo
   reboot
   ```
5. After reboot: POWER solid, Web UI and WiFi back, config untouched.

## Rules before trying this on an HS8145C5

- **vermagic must match your exact kernel.** The prebuilt `.ko` files are for
  HS8545M5 (R020 / 4.4.197) and HG8546M (R019 / 3.10.53-HULK2). HS8145C5 builds
  differ — a failed module load can abort SDK init exactly as above.
- Replace **only the unused PON module** (`hi_epon.ko` on a GPON line).
  Check first: `cat /mnt/jffs2/xpon_mode ; lsmod`
- **Enable and test telnet over cable BEFORE rebooting** with the injection in
  place. Back up `/mnt/jffs2` (`hw_boardinfo`, `hw_ctree.xml`) first.
- Test the module with `insmod` manually before making it persistent.
- POWER blinking forever **but telnet answers** → don't reflash, don't panic:
  undo via telnet as above.

*Recovery confirmed on the unit tested. Your firmware may behave differently —
keep the enable tool / UART recovery path ready.*
