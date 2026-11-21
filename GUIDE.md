# Guide For R019 and R020




```
strings getshell.ko | grep vermagic    # must equal the device string
device hi_sysctl.ko : vermagic=3.10.53-HULK2 SMP mod_unload modversions ARMv7
your getshell.ko    : vermagic=3.10.53-HULK2 SMP mod_unload modversions ARMv7 or R020
```
### Prechecks

```
cat  /mnt/jffs2/hw_boardinfo
# this value must be 4
obj.id = "0x00000001" ; obj.value = "4";
```

## 0. Put the module on the ONT


```sh
ftpget {ip} getshell_3.10.53_R019.ko pub/getshell_3.10.53_R019.ko
# e.g. via tftp / usb / scp → /mnt/jffs2/app/
ls -l /mnt/jffs2/app/getshell_3.10.53_R019.ko
```

## 1. Pre-checks
```sh
df -h /mnt/jffs2
ls /lib/modules/hisi_sdk
ls -lh /lib/modules/hisi_sdk      # sum must fit in jffs2 free
```

## 2. Manual test (catches symbol/CRC issues before reboot)
```sh
mkdir -p /mnt/jffs2/hw
printf '#!/bin/sh\ndate > /tmp/getshell_ran\n' > /mnt/jffs2/hw/getshell.sh
chmod +x /mnt/jffs2/hw/getshell.sh

insmod /mnt/jffs2/app/getshell_3.10.53_R019.ko
dmesg | tail
lsmod | grep getshell
cat /tmp/getshell_ran
```
- `getshell script run successfully !.` + timestamp → module + CRCs are good, proceed.
- `Invalid module format` → vermagic problem (shouldn't happen now).
- `Unknown symbol` / `disagrees about version of symbol` → CRC mismatch; you'd need the exact `Module.symvers`/config. (Unlikely given the exact vermagic match.)

## 3. Build the TranStar injection
```sh
mkdir -p /mnt/jffs2/TranStar
cp -rf /lib/modules/hisi_sdk/* /mnt/jffs2/TranStar/
cp /mnt/jffs2/app/getshell_3.10.53_R019.ko /mnt/jffs2/TranStar/hi_epon.ko
df -h /mnt/jffs2
```

## 4. Reboot & verify
```sh
reboot
# after:
lsmod | grep -E "getshell|hi_gpon|hi_epon"   # getshell present, hi_gpon present, hi_epon GONE
cat /tmp/getshell_ran                         # timestamp == boot (pre-NTP, 1981-ish is fine)
```
Then confirm PON registers and internet works.

## Instant revert
```sh
rm -rf /mnt/jffs2/TranStar
reboot
```

