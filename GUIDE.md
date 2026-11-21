# Guide — `getshell` TranStar Boot Autostart (Huawei ONT)

Persistent root execution at boot on **R019 (3.10.53‑HULK2)** and **R020 (4.4.197)** Huawei/HiSilicon Dopra ONTs.
**Prerequisite:** you already have a root shell (`WAP(Dopra Linux) #`). This guide adds *persistence/autorun*, not the initial unlock.

> ⚠️ **Use at your own risk.** Modifying your ONT can brick it or void your warranty. Back up your config and have a recovery method ready before you proceed.


| Firmware | Kernel | Module | vermagic |
|---|---|---|---|
| R019 (e.g. HG8546M) | 3.10.53‑HULK2 | `getshell_3.10.53_R019.ko` | `3.10.53-HULK2 SMP mod_unload modversions ARMv7` |
| R020 (e.g. HS8545M5) | 4.4.197 | `getshell_4.4.197_R020.ko` | `4.4.197 SMP mod_unload ARMv7` |

> Set this once per shell session (use your module's name):
> ```sh
> MOD=getshell_3.10.53_R019.ko      # or: MOD=getshell_4.4.197_R020.ko
> ```

---

## 1. Verify vermagic (must match EXACTLY)

On the **ONT** (ground truth from a stock module):
```sh
strings /lib/modules/hisi_sdk/hi_sysctl.ko | grep vermagic
```
On your **PC** (the module you'll load):
```sh
strings "$MOD" | grep vermagic
```
The two strings must be **identical**, including `-HULK2` and `modversions`.
- R019 has `modversions` → symbol CRCs are checked, so the manual test in §4 is mandatory.
- Mismatch → `Invalid module format`. Build a matching module (see ARTICLE.md).

## 2. Pre-checks on the ONT

```sh
# boardinfo: object 0x00000001 should be "4" (plain text on these firmwares; usually already 4)
cat /mnt/jffs2/hw_boardinfo | head -1
#   obj.id = "0x00000001" ; obj.value = "4";

# TranStar hook must exist
grep -n "TranStar" /etc/rc.d/rc.start/1.sdk_init.sh

# Which PON mode is active, and which module is UNUSED?
cat /mnt/jffs2/xpon_mode                 # 1 = GPON
lsmod | grep -E "hi_epon|hi_gpon"        # the inactive-mode module is your sacrificial target

# Space: the hisi_sdk set must fit in jffs2 free
df -h /mnt/jffs2
ls -lh /lib/modules/hisi_sdk
```

**Sacrificial-module rule:** replace the module for the **INACTIVE** PON mode only.
- GPON (`xpon_mode=1`) → replace `hi_epon.ko`, **keep `hi_gpon.ko`**.
- EPON → replace `hi_gpon.ko`, keep `hi_epon.ko`.

**Back up first:** `hw_boardinfo`, `hw_ctree.xml`, and `/mnt/jffs2` (tar to USB/PC).

## 3. Put the module on the ONT (`/mnt/jffs2/app/`)

BusyBox `ftpget` (HOST LOCAL REMOTE):
```sh
cd /mnt/jffs2/app
ftpget 192.168.1.100 "$MOD" "pub/$MOD"                 # anonymous FTP
# with login:  ftpget -u user -p pass 192.168.1.100 "$MOD" "pub/$MOD"
```
Or TFTP / USB / SCP:
```sh
cd /mnt/jffs2/app && tftp -g -r "$MOD" 192.168.1.100
# usb:  mount /dev/sda1 /mnt/usb && cp /mnt/usb/"$MOD" /mnt/jffs2/app/
ls -l /mnt/jffs2/app/"$MOD"
```

## 4. Manual test (catches symbol/CRC issues BEFORE reboot)

```sh
mkdir -p /mnt/jffs2/hw
printf '#!/bin/sh\ndate > /tmp/getshell_ran\n' > /mnt/jffs2/hw/getshell.sh
chmod +x /mnt/jffs2/hw/getshell.sh

insmod /mnt/jffs2/app/"$MOD"
dmesg | tail
lsmod | grep getshell
cat /tmp/getshell_ran
```
- `getshell script run successfully !.` + timestamp → good, proceed.
- `Invalid module format` → vermagic mismatch (§1).
- `Unknown symbol` / `disagrees about version of symbol` → CRC/config mismatch; needs exact `Module.symvers`.

## 5. Build the TranStar injection

```sh
mkdir -p /mnt/jffs2/TranStar
cp -rf /lib/modules/hisi_sdk/* /mnt/jffs2/TranStar/
# GPON line → swap the UNUSED hi_epon.ko (use hi_gpon.ko instead if you're on EPON):
cp /mnt/jffs2/app/"$MOD" /mnt/jffs2/TranStar/hi_epon.ko
df -h /mnt/jffs2
```

## 6. Reboot & verify

```sh
reboot
# after it returns:
lsmod | grep -E "getshell|hi_gpon|hi_epon"   # getshell present, hi_gpon present, hi_epon GONE
cat /tmp/getshell_ran                         # == this boot (pre-NTP "1981" is normal)
```
Then from `SU_WAP>`: `display onu info` (want O5/registered) and pass traffic from a LAN client.

## 7. Instant revert

```sh
rm -rf /mnt/jffs2/TranStar
reboot        # boot falls back to /lib/modules/hisi_sdk → fully stock
```

---

## Safety rules
- Swap **only the unused PON module**. Never the active one.
- **Never create** `/mnt/jffs2/stop`, `Equip.sh`, or `debug_start.sh` (one-shot debug trap that exits SDK init).
- Keep payload logs in `/tmp` (jffs2 is sync-mounted UBIFS → flash wear).
- Disable TR‑069 if you want the mod to survive ISP pushes.
- Next: add your app payload to `/mnt/jffs2/hw/getshell.sh` (sing-box / AdGuard / ttyd / frp) — see ARTICLE.md.
