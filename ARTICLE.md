# Persistent Root & Boot‑Time App Autostart on the Huawei HS8545M5 (V5R020) — Without Downgrading

*Turning an ISP‑locked GPON ONT into a tiny, always‑on Linux server using a kernel‑module injection into the stock boot flow.*

---

## Abstract

The Huawei HS8545M5 is a mass‑deployed China Mobile GPON/EPON ONT. Like most carrier CPE, it ships with a locked‑down userspace, a remote‑management daemon (TR‑069) that can overwrite your changes, and no supported way to run your own software. This article documents a complete, **reboot‑persistent** method to execute arbitrary code as root at boot on firmware **V5R020C00S200** — *without* downgrading to R019, *without* editing the encrypted‑looking `hw_boardinfo`, and *without* replacing any module the active PON line depends on.

The technique combines two ideas:

1. A tiny kernel module (`getshell.ko`) whose only job is to run a shell script as root when it loads.
2. A writable‑module‑directory window in the stock `1.sdk_init.sh` boot script (the **TranStar** path) that lets you load that module *in place of an unused SDK module* during a normal boot.

The result is a clean, reversible autostart primitive. On top of it you can run a userspace WireGuard proxy (sing‑box), network‑wide ad blocking (AdGuard Home), a browser terminal (ttyd), a reverse proxy (Caddy), or anything else that fits in ~56 MB of flash and ~100 MB of RAM.

> **Scope & ethics.** This is interoperability / right‑to‑repair research on **your own** CPE. It does not bypass ISP authentication, steal service, or touch the OLT. Modifying carrier equipment may violate your contract and can brick the device or drop your line. Proceed at your own risk, keep backups, and keep a recovery path. Serials, MACs, LOIDs and credentials are redacted throughout.

---

## 1. The device and why it's interesting

### 1.1 Hardware / software at a glance

| Property | Value |
|---|---|
| Model | Huawei HS8545M5 (CMCC) |
| Hardware version | `147C.A` |
| Software version | `V5R020C00S200` |
| Kernel | `4.4.197 #1 SMP … armv7l` |
| SoC | HiSilicon SD5116 family, ARMv7 Cortex‑A9 |
| RAM | ~244 MB total, ~104 MB available |
| NAND | 256 MB class, UBI volumes |
| libc | **musl** (`/lib/ld-musl-arm.so.1`) |
| Main OS | Dopra Linux (BusyBox userspace) |
| Hidden container | LXC `jvm` running **Apache Felix 4.4.0 OSGi on J2SE‑1.5** |

Two facts drive everything that follows:

- **musl libc** → any *static* ARMv7 binary (especially CGO‑disabled Go builds) runs natively. You are not tied to Huawei's ancient toolchain.
- **Kernel 4.4.197** → modern enough for current Go binaries, but **no `/dev/net/tun`** and no kernel WireGuard. Userspace networking (gVisor netstack) is the way.

### 1.2 The firmware landscape

The community consensus is that **R019/R020 are the "open" V5 releases**, while **R021/R022 progressively lock things down** (telnet removed, shell neutered, downgrade paths closed). Most public exploits and "allShell" packs target R019. A huge amount of forum traffic is people on R021/R022 trying to *downgrade to R020*.

If you are already on **R020 with a working root shell, you are in the best possible state.** Do not upgrade. As we'll show, you also don't need to downgrade to R019 to use the famous `Exploit-HS8545M-ONT` — its core mechanism works on R020 with a correctly built module.

### 1.3 Two worlds inside one box

The ONT runs:

- The **main Dopra OS** — the real router: PON, forwarding, Wi‑Fi, web UI, TR‑069. This is where we want to live.
- An **LXC container named `jvm`** — a Java/OSGi (Felix) stack for carrier "smart gateway" plugins. The `cplugin` descriptors literally read `Name:java / Version:J2SE-1.5` and `Name:felix / Version:4.4.0`.

The OSGi world is a **red herring** for custom apps. Writing bundles for a Java‑5‑era Felix runtime is a dead end. Note also there is **no `/opt/upt` SAF OpenWrt container** on this unit (some Telecom‑branded models have one). So the "install OpenWrt ipks in the hidden container" trick floating around forums does **not** apply here. We work in the main OS.

---

## 2. Reconnaissance: reading the box

Before touching anything, dump the state. Everything below came from a root shell (`WAP(Dopra Linux) #`).

### 2.1 Memory and storage

```text
Filesystem          Size   Used  Avail  Mounted on
/dev/root          41.6M  41.6M     0   /            (squashfs, read-only)
/dev/ubi0_4         8.8M   1.5M   7.3M  /mnt/jffs2   (config, UBIFS)
/dev/ubi0_5       109.2M  48.1M  56.4M  /mnt/jffs2/app (apps, UBIFS)
tmpfs             512.0M     0  512.0M  /dev/shm
none              512.0M   1.9M 510.1M  /var
```

MTD layout:

```text
mtd2/3   allsystemA/B   ~55 MB each   (kernel + rootfs, A/B)
mtd4     ubilayer       ~143 MB       (UBI: config + app volumes)
mtd10    file_system    ~11.6 MB
mtd11    app_system     ~120 MB
```

Key takeaways:

- The directory is called `jffs2` but it is **actually UBIFS** (`mount -t ubifs -o sync …`). The `-o sync` matters: every write hits flash immediately. **Keep logs off it.**
- `/mnt/jffs2` (config) is small (~7 MB free) but `/mnt/jffs2/app` has **~56 MB free** — that's your app budget.
- UBIFS **compresses**. As we'll see, ~3.2 MB of `.ko` files consumed only ~1.5 MB on flash.

### 2.2 The boot script is the map

The single most important file is `/etc/rc.d/rc.start/1.sdk_init.sh`. Grepping it for `jffs2` and `insmod` reveals every hook and every module load. The relevant landmarks:

```text
253:  [ -f /mnt/jffs2/stop ] && exit
323:  [ -f /mnt/jffs2/Equip.sh ] && /bin/Equip.sh && exit
337:  if [ -f /mnt/jffs2/debug_start.sh ]; then
338:      /mnt/jffs2/debug_start.sh
339:      cd /mnt/jffs2
340:      mv -f debug_start.sh debug_startbak.sh
341:      exit
342:  fi
345:  if [ -f /mnt/jffs2/TranStar/hi_sysctl.ko ]; then
346:        cd /mnt/jffs2/TranStar/
...
469:      insmod hi_gmac.ko
470:      insmod hi_emac.ko
471:      insmod hi_gpon.ko
472:      insmod hi_epon.ko
```

This one excerpt contains both the **trap** and the **opportunity**.

---

## 3. The autostart problem

We need a hook that (a) runs as root, (b) on **every** boot, (c) **without** disrupting normal SDK initialization.

### 3.1 Pitfall: `debug_start.sh` is a one‑shot debug trap

It's tempting to see lines 337–339 and think "great, an autorun hook." **Read the whole block.** After running your script it does:

```sh
mv -f debug_start.sh debug_startbak.sh   # rename → never runs again
exit                                      # EXIT 1.sdk_init.sh
```

That `exit` terminates SDK init **before any `hi_*` module loads**. Create `debug_start.sh` and on the next boot your ONT runs your script once, renames it, and comes up with **no PON / no forwarding**. (The following boot is normal again because the file was renamed — so it's not a permanent brick, but it is absolutely *not* persistence.) Many forum guides miss this because they grep only for `jffs2` and never see lines 340–342.

### 3.2 Pitfall: `Equip.sh` and `stop`

- `/mnt/jffs2/stop` (line 253) halts the boot. Never create it.
- `/mnt/jffs2/Equip.sh` (line 323) diverts into **equip/manufacturing mode**, changing device behavior and exiting normal init. Not a clean autorun.

### 3.3 The opportunity: the TranStar module directory

Lines 345–350:

```sh
if [ -f /mnt/jffs2/TranStar/hi_sysctl.ko ]; then
      cd /mnt/jffs2/TranStar/          # writable UBIFS
else
      cd /lib/modules/hisi_sdk          # read-only squashfs
fi
```

Then dozens of **relative‑path** `insmod hi_*.ko` calls follow. If you populate `/mnt/jffs2/TranStar/` with the SDK modules, the boot loads them **from your writable directory** instead of read‑only flash. Replace *one unused* module with your own, and the stock boot happily `insmod`s it for you — every boot, no debug trap, no equip mode.

This is the core of the public `Exploit-HS8545M-ONT` (0neday, and the bt‑worm fork). The fork's README frames it for R019, but the **mechanism is present verbatim in R020** — we just have to adapt the details.

---

## 4. The `getshell` kernel module

### 4.1 What it does

`getshell.ko` is a minimal module: on `init` it executes `/mnt/jffs2/hw/getshell.sh` as root (via a usermode helper), prints `getshell script run successfully !.` to the kernel log, and returns 0. It provides no symbols and depends on nothing — `lsmod` shows it at ~1470 bytes with refcount 0. It is a **boot‑time root script executor**, nothing more.

### 4.2 Why a 4.4.197 build works on R020

The prebuilt module in the original repo is stamped "only support r019." The reason is **vermagic**, not magic. Our R020 kernel is `4.4.197`. A module built for that exact release reports:

```text
$ modinfo getshell_4.4.197.ko | grep vermagic
vermagic:  4.4.197 SMP mod_unload ARMv7
```

That matches the running kernel, so the version gate passes. (Symbol CRCs under `MODVERSIONS` and any signature enforcement are separate gates — best results come from building against Huawei's GPL kernel source for the exact firmware. In practice the module loaded cleanly here.)

### 4.3 Prove it manually first

Because we already have root shell, we can test the module **without any boot trickery**:

```sh
mkdir -p /mnt/jffs2/hw
printf '#!/bin/sh\ndate > /tmp/getshell_ran\n' > /mnt/jffs2/hw/getshell.sh
chmod +x /mnt/jffs2/hw/getshell.sh

insmod /mnt/jffs2/app/getshell_4.4.197.ko
dmesg | tail
lsmod | grep getshell
cat /tmp/getshell_ran
```

Observed:

```text
[1214144.030039] getshell script run successfully !.
getshell  1470  0
Fri Nov 20 07:37:48 UTC 2026
```

Module loads, payload runs as root. The primitive is confirmed on R020. A manual `insmod` is **not persistent** (and a bad module can oops → reboot, which is reversible), so this is a safe sandbox before wiring it into boot.

---

## 5. The TranStar injection

### 5.1 Boot flow, before and after

```text
STOCK                              INJECTED
─────                              ────────
cd /lib/modules/hisi_sdk           cd /mnt/jffs2/TranStar      (hi_sysctl.ko present)
insmod hi_sysctl.ko                insmod hi_sysctl.ko         (real copy)
insmod hi_crg.ko …                 insmod hi_crg.ko …          (real copies)
insmod hi_gpon.ko   ← ACTIVE       insmod hi_gpon.ko           (real copy, ACTIVE)
insmod hi_epon.ko   ← unused       insmod hi_epon.ko           (== getshell.ko!)
                                     └─ runs /mnt/jffs2/hw/getshell.sh as root
```

Only **relative‑path** `insmod`s are redirected. Absolute loads like `insmod /lib/modules/wap/hw_module_*.ko` are untouched, so the rest of the system initializes exactly as stock.

### 5.2 Choosing the sacrificial module

This is the step that decides whether you keep your internet. Read the refcounts:

```text
hi_epon   171063  0                                   ← loaded, UNUSED
hi_gpon   199452  2  hw_module_gmac,hw_ker_optic_chip_511x_gpon   ← ACTIVE GPON
```

The HS8545M5 is a GPON/EPON combo; the boot branch at lines 469–472 loads **both** `hi_gpon.ko` and `hi_epon.ko`. On a GPON line (`xpon_mode=1`) the EPON module is dead weight — refcount 0, nothing depends on it. **That is the module we replace.** Replacing `hi_gpon.ko` instead would kill registration and forwarding.

> Rule: sacrifice the module with **refcount 0** that corresponds to the *inactive* PON mode. Verify with `lsmod` before every change.

### 5.3 `hw_boardinfo`: nothing to do

The R019 README says to set `obj.id = "0x00000001" ; obj.value = "4"`. On this R020 unit the file is **plain text** and that object is **already `"4"`**:

```text
obj.id = "0x00000001" ; obj.value = "4";
```

So no edit, no decrypt/re‑encrypt dance. (The boot's `decrypt_boardinfo` call is effectively a no‑op for this format.) Always `cat` it before assuming.

---

## 6. Implementation, step by step

### 6.1 Pre‑checks

`du` is missing from this BusyBox, so size the module set with `ls -lh`:

```sh
df -h /mnt/jffs2
lsmod | grep hi_epon          # confirm refcount 0
ls -lh /lib/modules/hisi_sdk
```

The 23 `.ko` files sum to **~3.2 MB** — comfortably inside the 7.3 MB free on the config volume.

### 6.2 Build the injected module set

```sh
mkdir -p /mnt/jffs2/TranStar
cp -rf /lib/modules/hisi_sdk/* /mnt/jffs2/TranStar/
cp /mnt/jffs2/app/getshell_4.4.197.ko /mnt/jffs2/TranStar/hi_epon.ko
df -h /mnt/jffs2
```

Copy **all** modules, not just two — once `hi_sysctl.ko` is present the boot loads *everything* from TranStar. Missing modules mean a broken boot.

Result: flash usage went from 1.5 MB → 3.0 MB (**+1.5 MB**, thanks to UBIFS compression), leaving **5.8 MB free**.

### 6.3 Reboot and verify

Keep the trivial test payload for the first reboot to isolate risk:

```sh
reboot
```

After it returns:

```text
# lsmod | grep -E "getshell|hi_gpon|hi_epon"
getshell  1470  0
hi_gpon   199452  2  hw_module_gmac,hw_ker_optic_chip_511x_gpon
(hi_epon absent)

# cat /tmp/getshell_ran
Thu Jan  1 00:00:03 UTC 1981
```

Interpretation:

- `getshell` present → the injection fired at boot.
- `hi_gpon` refcount 2 → **GPON intact**. `hi_ponlp`/`hi_bridge` refcounts dropped by exactly 1, confirming nothing else needed `hi_epon`.
- The **1981** timestamp is *proof*, not a bug: the ONT's RTC starts at its epoch and only syncs via NTP/ISP later. `00:00:03` means the payload ran ~3 s into boot, during SDK init — exactly the intended injection point.

Finally, confirm service: from `SU_WAP>` run `display onu info` (want O5/registered) and `display optic`, then pass traffic from a LAN client.

### 6.4 Instant revert

```sh
rm -rf /mnt/jffs2/TranStar
reboot
```

The boot falls back to `/lib/modules/hisi_sdk` (the `else` branch). Originals were never modified, so this is a complete, clean undo.

---

## 7. The payload layer: turning root into apps

The module gives you root execution at boot; `getshell.sh` is where you use it. Design rules:

1. **Idempotent.** Guard with `pidof` so re‑runs don't spawn duplicates.
2. **Respawn.** Wrap services in `while true; do …; sleep 5; done &`.
3. **Log to tmpfs.** `/tmp`, never the sync‑mounted UBIFS.
4. **Background everything and `exit 0`** so boot is never blocked.
5. **Open the firewall.** Huawei's `INPUT_ACL` drops LAN→ONT traffic by default.

A robust orchestrator:

```sh
#!/bin/sh
A=/mnt/jffs2/app/myapps
LOG=/tmp/myapps.log
mkdir -p "$A"
export PATH=$A/bin:/sbin:/bin:/usr/sbin:/usr/bin:$PATH

for p in 2080 9090 7681 3000; do
  iptables -C INPUT_ACL -i br+ -p tcp --dport $p -j ACCEPT 2>/dev/null || \
  iptables -I INPUT_ACL -i br+ -p tcp --dport $p -j ACCEPT 2>/dev/null
done

if ! pidof sing-box >/dev/null; then
  ( while true; do "$A/sing-box" run -c "$A/config.json" -D "$A" >>"$LOG" 2>&1; sleep 5; done ) &
fi
if ! pidof ttyd >/dev/null; then
  ( while true; do "$A/ttyd" -p 7681 -c admin:STRONG_PASS /bin/sh >>"$LOG" 2>&1; sleep 5; done ) &
fi

date > /tmp/getshell_ran
exit 0
```

**The 1981 clock matters for TLS.** At boot the clock is wrong, so anything doing certificate validation (AdGuard DoH/DoT upstreams, Caddy ACME) fails on the first attempt. The `sleep 5` respawn loop transparently retries until NTP sets the time. WireGuard and frp don't care about the clock.

The bt‑worm fork ships a ready payload in this exact style — `getshell.sh` calling `hw/bin/shadowsocks-libev.sh`, `start-dnsmasq-notfull.sh`, `start-sshd.sh`, `set-variable-path.sh`. You can keep it, or swap in your own launchers. (Comment out its `hw_hardinfo_feature` restore: the R020 boot regenerates that file later anyway.)

---

## 8. Which apps actually fit

Constraints: **musl**, **ARMv7**, **no TUN**, ~56 MB flash, ~100 MB RAM. Static Go binaries are ideal.

| App | Role | Binary | RAM | Notes |
|---|---|---|---|---|
| **sing-box** | WireGuard + proxy + Clash API | ~30 MB | ~40 MB | WireGuard via **userspace gVisor** — no TUN/kernel module needed |
| **AdGuard Home** | DNS ad/tracker blocking | ~30 MB | ~60–100 MB | Port 53 conflicts with dnsmasq → redirect |
| **ttyd** | Browser terminal (management) | ~2 MB | ~5 MB | Static; **LAN‑only**, strong auth |
| **Caddy** | nginx‑style reverse proxy | ~30 MB | ~30 MB | Official static ARMv7; easier than cross‑compiling nginx |
| **frp (frpc)** | Outbound reverse tunnel to a VPS | ~20 MB | ~15 MB | Firewall‑friendly (dials out) |

You can't fit all of them; pick one "big" service plus ttyd.

### 8.1 WireGuard without TUN (sing-box)

There is no `/dev/net/tun` and no kernel WireGuard. sing-box implements WireGuard with a **userspace gVisor netstack by default**, so it works as an *endpoint* without any kernel support. Note the API change: the WireGuard **outbound was deprecated and removed in 1.13**; use the **endpoint** form on current releases:

```json
{
  "log": { "level": "info" },
  "dns": { "servers": [ { "tag": "cf", "address": "1.1.1.1" } ] },
  "inbounds": [
    { "type": "mixed", "tag": "proxy", "listen": "0.0.0.0", "listen_port": 2080 }
  ],
  "endpoints": [
    {
      "type": "wireguard", "tag": "wg", "mtu": 1420,
      "address": ["10.0.0.2/32"],
      "private_key": "YOUR_WG_PRIVATE_KEY",
      "peers": [ {
        "address": "WG_SERVER_IP", "port": 51820,
        "public_key": "SERVER_PUBLIC_KEY",
        "allowed_ips": ["0.0.0.0/0", "::/0"],
        "persistent_keepalive_interval": 25
      } ]
    }
  ],
  "route": { "final": "wg" },
  "experimental": { "clash_api": { "external_controller": "0.0.0.0:9090" } }
}
```

LAN clients point their SOCKS/HTTP proxy at `ONT-IP:2080`; the Clash dashboard lives at `:9090`. Generate keys with `sing-box generate wg-keypair`.

### 8.2 AdGuard Home around dnsmasq

The ONT already owns port 53. Run AGH's DNS on **5335** and redirect LAN DNS to it (add to the payload):

```sh
iptables -t nat -I PREROUTING -i br+ -p udp --dport 53 -j REDIRECT --to-ports 5335
iptables -t nat -I PREROUTING -i br+ -p tcp --dport 53 -j REDIRECT --to-ports 5335
```

Set AGH upstreams to public resolvers (not the ONT, or you loop). Keep query logs off flash.

---

## 9. Safety, recovery, and gotchas

- **Revert is one command:** `rm -rf /mnt/jffs2/TranStar && reboot`.
- **Back up first:** `tar czf /mnt/usb/jffs2-backup.tar.gz /mnt/jffs2`, plus `hw_ctree.xml`, `hw_boardinfo`, and your LOID/PPPoE/VLAN notes. Optionally dump `allsystemA` for a full rollback.
- **GPON/EPON caveat:** this stays safe only while you're on GPON. If you switch to EPON, `hi_epon` becomes required — revert TranStar or move the swap to the then‑unused `hi_gpon.ko`.
- **TR‑069:** the ISP can push firmware/config that wipes your work (or, worse, lands on top of a modified module set). Disable TR‑069 / block the ACS if you want the mod to stick.
- **Flash wear:** `-o sync` UBIFS means logs on flash are expensive. Always log to `/tmp`.
- **Never create** `stop`, `Equip.sh`, or `debug_start.sh` in jffs2.
- **Brick path:** a wrong module set or power loss during a *firmware flash* can brick; recovery is the enable/multicast tool, with UART as a last resort. The TranStar method itself touches no firmware partition, which is why it's comparatively low‑risk.

---

## 10. Why we didn't downgrade to R019

A common reflex is "the exploit is R019‑only, so downgrade." That's a solution to a problem we don't have:

- We **already had root shell** on R020 — the exploit's end goal.
- The **TranStar window exists verbatim in R020** (line 345), and a `4.4.197`‑vermagic module loads fine.
- Downgrading to a *stock* R019 image would **destroy the working shell**, force a re‑crack, risk registration mismatch, and invite CMCC to auto‑upgrade us straight back.

The only thing R019 buys you is a prebuilt module you don't need. Building (or obtaining) the module for the running kernel is strictly better.

---

## 11. Legal & ethical notes

This work is framed as **interoperability and right‑to‑repair on hardware you control**: running your own software on your own CPE, recovering management access, and improving security/utility (ad blocking, encrypted tunnels, remote homelab access). It does **not** circumvent ISP authentication or entitlements. Depending on jurisdiction and contract, modifying carrier‑supplied equipment may still be restricted; the safest posture is a device you own outright, kept on a lab line, with full backups and a recovery path. Don't expose management services (ttyd, dashboards) to the WAN.

---

## 12. Conclusion

On a locked HS8545M5 running V5R020, we achieved **reboot‑persistent root execution** with:

1. A `4.4.197`‑vermagic `getshell.ko` that runs a script as root at module load.
2. A **TranStar** writable module directory that the stock boot loads from.
3. A surgical swap of the **unused `hi_epon.ko`**, leaving active GPON untouched.
4. A idempotent, respawning, tmpfs‑logging payload that launches real services.

No downgrade, no boardinfo edit, no firmware partition writes, and a one‑command revert. The ONT keeps doing its day job while quietly becoming a ~100 MB‑RAM Linux server that survives every reboot.

**Next steps:** drop sing-box (WireGuard endpoint) + ttyd into `/mnt/jffs2/app/myapps/`, wire them into `getshell.sh`, and reboot‑verify. From there, AdGuard or Caddy are incremental additions within the same budget.

---

## Appendix: cheat sheet

```sh
# state
df -h; ls -lh /lib/modules/hisi_sdk; lsmod | grep -E "hi_epon|hi_gpon"
cat /mnt/jffs2/xpon_mode; cat /mnt/jffs2/hw_boardinfo | head

# inject
mkdir -p /mnt/jffs2/TranStar
cp -rf /lib/modules/hisi_sdk/* /mnt/jffs2/TranStar/
cp /mnt/jffs2/app/getshell_4.4.197.ko /mnt/jffs2/TranStar/hi_epon.ko

# payload
vi /mnt/jffs2/hw/getshell.sh ; chmod +x /mnt/jffs2/hw/getshell.sh

# test without reboot
rmmod getshell ; insmod /mnt/jffs2/app/getshell_4.4.197.ko ; cat /tmp/getshell_ran

# verify after reboot
lsmod | grep -E "getshell|hi_gpon|hi_epon" ; cat /tmp/getshell_ran

# revert
rm -rf /mnt/jffs2/TranStar ; reboot
```

*Module reference: `getshell_4.4.197.ko`, vermagic `4.4.197 SMP mod_unload ARMv7`, MD5 `98590bd8c6ddd301731a795fd5dc77f1`. Based on `0neday/Exploit-HS8545M-ONT` and the `bt-worm` fork.*
