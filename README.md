# HS8545M-ONT — Huawei ONT Root Shell, `getshell` Kernel Modules & Boot-Time Autostart

**Persistent root shell + boot-time app autostart on Huawei / HiSilicon Dopra Linux GPON & EPON ONTs (光猫).**
Prebuilt `getshell` kernel modules, allShell (补全shell) patches, and the classic enable/config tools — with a fully documented, **reversible** injection technique that needs **no firmware downgrade** and **no boardinfo edit**.
> ⚠️ **Use at your own risk.** Modifying your ONT can brick it or void your warranty. Back up your config and have a recovery method ready before you proceed.
> Repo: <https://github.com/mrxehmad/HS8545M-ONT>

---

## Verified working

| Model | Carrier | Firmware | Kernel | Module |
|---|---|---|---|---|
| **Huawei HS8545M5** | China Mobile (CMCC) | `V5R020C00S200` | `4.4.197` (musl) | `getshell_4.4.197_R020.ko` |
| **Huawei HG8546M** | EchoLife | `V5R019C10S125` | `3.10.53-HULK2` | `getshell_3.10.53_R019.ko` |

Both achieve **reboot-persistent root execution** via the **TranStar** module-directory hook in `/etc/rc.d/rc.start/1.sdk_init.sh`, sacrificing only the *unused* PON module (`hi_epon.ko` on a GPON line). Full write-up in **[ARTICLE.md](ARTICLE.md)**; step-by-step in **[GUIDE.md](GUIDE.md)**.

---

## 📦 What's inside

```
kernel-modules/
  getshell_3.10.53_R019.ko   # vermagic: 3.10.53-HULK2 SMP mod_unload modversions ARMv7
  getshell_4.4.197_R020.ko   # vermagic: 4.4.197 SMP mod_unload ARMv7
allshell/
  r017.bin / r019.bin / r020.bin   # shell-completion (补全shell) patches
  tools/
    ONT组播配置工具V3-V5 2.0.exe    # multicast / enable tool (使能/组播)
    huaweiXML_CFG.exe              # hw_ctree.xml encrypt/decrypt
    HW Dollar2.exe                 # $2... password decrypt
    HWFW_GUI.exe                   # firmware pack/unpack editor
ARTICLE.md   # full technical documentation
GUIDE.md     # quick guide (in progress)
```

The `getshell` module's only job: on `insmod`, run `/mnt/jffs2/hw/getshell.sh` **as root**. Wire it into the stock boot via TranStar and you get a clean autostart primitive for sing-box (WireGuard), AdGuard Home, ttyd, Caddy, frp, shadowsocks, etc.

---

## 🧩 Compatibility & supported models

**Important:** the two prebuilt `.ko` files are **kernel-version-specific** (vermagic must match exactly). The **TranStar technique** and the **allShell patches**, however, apply across the whole **HiSilicon SD511x / Dopra Linux** ONT family. For any other model, build `getshell.ko` against its exact kernel (see ARTICLE.md) or use the matching allShell patch.

### Same SDK family — technique applicable (build a matching module / use allShell)
EchoLife **HG** series:
`HG8010` `HG8010H` `HG8012H` `HG8120C` `HG8120H` `HG8240` `HG8240H` `HG8240H5` `HG8245` `HG8245A` `HG8245C` `HG8245H` `HG8245H5` `HG8245Q` `HG8245Q2` `HG8245W5` `HG8245W5-6T` `HG8247` `HG8247H` `HG8247W5` `HG8310M` `HG8321R` `HG8540M` `HG8546M`

China-carrier **HS** series (CMCC / Telecom / Unicom):
`HS8145C` `HS8145V` `HS8145C5` `HS8145V5` `HS8145V6` `HS8346V` `HS8545M` `HS8545M5` `HS8546V` `HS8546V5`

**EG** enterprise series:
`EG8040H` `EG8141A5` `EG8145V5` `EG8245H` `EG8245H5` `EG8247W5`

**HN** series:
`HN8145V` `HN8245Q` `HN8245W5` `HN8346V` `HS8346X6` `HN8346X6`

SFP ONT sticks:
`MA5671A` `MA5675M`

Edge / home-gateway ONTs:
`K662C` `K662D` `K662E`

### ⚠️ Different architecture — NOT this exploit
Newer **OptiXstar** units (`V173` `V271` `V273` `HG8245X6` `HN8145XR` `HN8145X6` `HN8145X6N` `HN8546X6`, firmware **R021/R022/R023**, kernel **5.10**, `whwh` signed images) use a different plugin framework (`eaiapp` / `cplugin`) and signed firmware. See the credits for the projects that cover those.

### Chipsets / platforms covered
`HiSilicon SD5113` `SD5115` `SD5116` `SD5117` `SD5117P` `SD5118` · `HULK2` · ARMv7 Cortex-A9 · Dopra Linux · kernel `3.10.53` / `4.4.197` · firmware `V3R017` / `V5R019` / `V5R020` · GPON / EPON / XPON.

---

## 🚀 Quick start (TL;DR)

```sh
# 1. Confirm the TranStar hook + which PON module is unused
grep -n "TranStar" /etc/rc.d/rc.start/1.sdk_init.sh
cat /mnt/jffs2/xpon_mode ; lsmod | grep -E "hi_epon|hi_gpon"

# 2. Match vermagic to YOUR kernel
strings /lib/modules/hisi_sdk/hi_sysctl.ko | grep vermagic

# 3. Test the module manually (root shell required)
insmod getshell_<ver>.ko ; dmesg | tail ; cat /tmp/getshell_ran

# 4. Inject via TranStar (replace the UNUSED PON module only)
mkdir -p /mnt/jffs2/TranStar
cp -rf /lib/modules/hisi_sdk/* /mnt/jffs2/TranStar/
cp getshell_<ver>.ko /mnt/jffs2/TranStar/hi_epon.ko   # GPON line → hi_epon
reboot

# Revert anytime:
rm -rf /mnt/jffs2/TranStar ; reboot
```

Full details, payloads, firewall (`INPUT_ACL`), and app examples: **[ARTICLE.md](ARTICLE.md)**.

---

## 🔎 Keywords / 关键词

Huawei ONT root, Huawei ONU shell, getshell.ko, TranStar exploit, allShell, 补全shell, 光猫破解, 光猫超级密码, 开启telnet, 使能工具, ONT组播工具, hw_ctree.xml, aescrypt2, HW Dollar2, telecomadmin, CMCCAdmin, adminHW, Dopra Linux, HiSilicon SD5116, SD5115, HULK2, GPON, EPON, XPON, V5R019, V5R020, V3R017, kernel module autostart, boot persistence, HS8545M5, HG8546M, HS8145V5, HS8546V5, HG8245H, HG8321R, EG8145V5, HN8145V, MA5671A, K662C, WireGuard on ONT, AdGuard Home on router, sing-box, ttyd, frp, shadowsocks-libev, 华为光猫, 移动光猫, 电信光猫, 联通光猫.

---

## ⚠️ Safety & legal

- For **your own** equipment / authorized interoperability & right-to-repair research only. Do **not** bypass ISP authentication or steal service.
- Modifying carrier CPE may violate your contract and **can brick the device or drop your line**. Keep backups (`/mnt/jffs2`, `hw_ctree.xml`, `hw_boardinfo`) and a recovery path (enable tool / UART).
- Replace **only the unused PON module** (`hi_epon.ko` on GPON, `hi_gpon.ko` on EPON). Never swap the active one.
- Never create `/mnt/jffs2/stop`, `Equip.sh`, or `debug_start.sh` (one-shot debug trap that exits SDK init).
- Disable TR-069 if you want the mod to survive; keep logs in `/tmp` (jffs2 is sync-mounted UBIFS).
- Prebuilt modules derive from GPL kernel work; source/build instructions are in ARTICLE.md. Windows tools in `allshell/tools/` are third-party freeware from the community, mirrored for convenience — all rights belong to their authors.

---

## 🙏 Credits & acknowledgements

This repo stands on the work of many people and communities:

- **0neday (COOLSTAR)** — original [`Exploit-HS8545M-ONT`](https://github.com/0neday/Exploit-HS8545M-ONT): the TranStar `getshell` technique, `getshell.c`, allShell builds.
- **bt-worm** — [`Exploit-HS8545M-ONT` fork](https://github.com/bt-worm/Exploit-HS8545M-ONT): `bridge.ko` + app-launcher payload (`shadowsocks-libev`, `dnsmasq-notfull`, `sshd`).
- **2879597772 / wcyuns** — [`ONT`](https://github.com/2879597772/ONT): enable tools, firmware packs, allShell collection.
- **QingNetwork / j33h22** — [`FuckingHuaweiHG8321R`](https://github.com/QingNetwork/FuckingHuaweiHG8321R): free enable/config tooling & guides.
- **achaoge** — discovery of the hidden OpenWrt/SAF container (chinadsl & right.com.cn).
- **nullException** — running frp / v2ray / shadowsocks on Huawei ONTs (chinadsl).
- **zebraMada** — [`sing-box-for-V173`](https://github.com/zebraMada/sing-box-for-V173): userspace WireGuard/proxy on OptiXstar.
- **tabris17** — [`Crack-Huawei-ONT`](https://github.com/tabris17/Crack-Huawei-ONT): `load pack by tftp` shell-patch method.
- **coolrecep (Recep Baltaş)** — [`Huawei-HG8245X6-Root` / HekzOntRace](https://github.com/coolrecep/Huawei-HG8245X6-Root); **foulab** (TOCTOU) and **Ferdi Burak** (hidden config page).
- **palmerc** — [`AESCrypt2`](https://github.com/palmerc/AESCrypt2); **staaldraad** — Huawei password-decrypt gist; **Jakiboy** — [`Hwdecode`](https://github.com/Jakiboy/Hwdecode); **back1ply** — [`huawei-dn8245v-decrypt`](https://github.com/back1ply/huawei-dn8245v-decrypt).
- **ValdikSS** — Huawei update-server firmware index gist.
- **Anime4000 / logon84** — `Hacking_Huawei_HG8240H5_ONT`, `Hacking_Huawei_HG8012H_ONT`.
- **hack-gpon project** (simonebortolin & contributors) — <https://hack-gpon.org>.
- **csersoft / feyacxf** — jffs2 permanent-shell method (chinadsl); **chinadslxiaobai** — HS8545M5 R021/R022 threads; **海信** — HS8545M5 R22 downgrade guide (right.com.cn); **GGpengqiang** — HN8346X6 LinkHome SDK plugin; **geyee** — HS8145V6 command reference.
- **free-time-team / leexiaolan** — HG8120C `Equip.sh` / `hw_ldsp_cfg` autostart analysis.
- **xdevgms** — SourceForge *EchoLife ONT Tools Huawei*.
- The **恩山无线论坛 (right.com.cn)** and **宽带技术网 (chinadsl.net)** communities, and the **HiSilicon / Huawei GPL open-source** releases.

If your work is referenced here and you'd like a correction or removal, open an issue.

---

## 📄 License

Documentation: CC-BY-SA-4.0. Kernel modules are derivative of GPL-2.0 Linux kernel work (source/build steps in ARTICLE.md). Third-party tools retain their original licenses/authors.
