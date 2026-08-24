# HP EliteBook Ultra G1q — firmware blob inventory

Resolves [JimmayVV/omarchy-iso#4](https://github.com/JimmayVV/omarchy-iso/issues/4)
(research ticket for map [#1](https://github.com/JimmayVV/omarchy-iso/issues/1)).
Researched 2026-08-24 against mainline `master`, linux-firmware `main`, Arch/ALARM
`linux-firmware 20260810-2`, and community sources. Everything below is from
primary sources unless marked *community report* or *unverified*.

## TL;DR

* The G1q DTS (`x1e80100-hp-elitebook-ultra-g1q.dts`) is a 30-line overlay on the
  OmniBook X 14 DTS. It overrides exactly five `firmware-name` strings, all under
  `qcom/x1e80100/hp/elitebook-ultra-g1q/`: the GPU zap shader, ADSP image + DTB,
  CDSP image + DTB. Plus the sound-card name, which selects the audio topology file.
* **linux-firmware has no `hp/elitebook-ultra-g1q/` directory at all**, and its
  `hp/omnibook-x14/` directory holds only the audio topology. Dell and Lenovo submitted
  their X1E signed blobs with explicit licences (`LICENSE.dell`, `LICENSE.qcom`); HP
  has not. So every HP-signed blob is category (c): **extract from the laptop's
  Windows partition**. No pending linux-firmware merge request changes this.
* Everything *not* HP-signed is category (a), redistributable, and already in the Arch
  and Arch Linux ARM `linux-firmware-qcom` / `linux-firmware-atheros` packages: GPU
  SQE/GMU microcode, WCN6855 and WCN7850 Wi-Fi, Bluetooth, and the OmniBook X 14 audio
  topology.
* There is no category (b) in the strict sense: nothing relevant is in linux-firmware
  under a non-redistributable licence. The only nuance is the generic
  `qcom/x1e80100/{adsp,cdsp,gen70500_zap}.mbn` set, which is redistributable but
  signed for Qualcomm reference boards and will not authenticate on an OEM-fused HP
  machine; the DT deliberately points at the HP paths instead.
* A public ISO can therefore show a display (DPU/eDP need no blobs), get Wi-Fi and
  Bluetooth on either Wi-Fi SKU, and run the TUI configurator, using only
  linux-firmware. GPU acceleration, audio, battery reporting, and the DSPs need the
  five HP blobs and must be pulled from Windows on the device at first boot.

## How the kernel resolves firmware on this board

```
x1e80100-hp-elitebook-ultra-g1q.dts        model/compatible + 5 firmware-name overrides + sound model
  #include x1e80100-hp-omnibook-x14.dts    X14 firmware-name overrides (superseded), sound model
    #include hamoa.dtsi                     SoC (x1e80100 was renamed "hamoa" in 6.18)
    #include hamoa-pmics.dtsi
    #include x1-hp-omnibook-x14.dtsi        shared baseboard: Wi-Fi/BT/PMU, codec, GPU enable, panel
```

G1q overrides (mainline `master`, commit `afc48c680438`, in v6.17):

```
&gpu_zap_shader  firmware-name = "qcom/x1e80100/hp/elitebook-ultra-g1q/qcdxkmsuc8380.mbn";
&remoteproc_adsp firmware-name = ".../qcadsp8380.mbn", ".../adsp_dtbs.elf";
&remoteproc_cdsp firmware-name = ".../qccdsp8380.mbn", ".../cdsp_dtbs.elf";
&sound           model = "X1E80100-HP-ELITEBOOK-ULTRA-G1Q";
```

Neither the X14 DTS nor the G1q DTS enables `&iris` (video) and `hamoa.dtsi` has no
CAMSS node, so the kernel requests nothing for video or camera on this board today.

## Per-subsystem table

Categories: **(a)** in linux-firmware, redistributable; **(b)** in linux-firmware but
restricted; **(c)** extract from Windows. "ISO" = can ship on a public ISO.

| Subsystem | Path the kernel requests | Source | Licence | ISO | Windows origin (if (c)) |
|---|---|---|---|---|---|
| GPU microcode (Adreno `43050c01`, "gen70500", a.k.a. X1-85/a741 class) | `qcom/gen70500_sqe.fw`, `qcom/gen70500_gmu.bin` (from `a6xx_catalog.c`) | (a) linux-firmware; Arch `linux-firmware-qcom` | `LICENSE.qcom` + `NOTICE.qcom`, redistributable | Y | n/a |
| GPU zap shader | `qcom/x1e80100/hp/elitebook-ultra-g1q/qcdxkmsuc8380.mbn` | (c) not in linux-firmware | HP/Qualcomm proprietary, no redistribution grant | **N** | `C:\Windows\System32\DriverStore\FileRepository\qcdx8380.inf_arm64_<hash>\qcdxkmsuc8380.mbn` |
| GPU zap shader, generic | `qcom/x1e80100/gen70500_zap.mbn` (exists, v0.15) | (a) linux-firmware | `LICENSE.qcom` | Y, but useless | Signed for Qualcomm CRD/QCP; not requested by the HP DT and will not pass TrustZone auth on HP hardware |
| ADSP remoteproc (audio, battery/`pmic-glink`, sensors) | `qcom/x1e80100/hp/elitebook-ultra-g1q/qcadsp8380.mbn` + `adsp_dtbs.elf` | (c) | proprietary | **N** | `...\FileRepository\qcsubsys_ext_adsp8380.inf_arm64_<hash>\{qcadsp8380.mbn, adsp_dtbs.elf}` |
| ADSP, generic | `qcom/x1e80100/adsp.mbn`, `adsp_dtb.mbn` | (a) linux-firmware | `LICENSE.qcom-2` | Y, but useless | Reference-board signature; PAS auth fails on OEM devices |
| CDSP remoteproc (compute/NPU, fastrpc) | `qcom/x1e80100/hp/elitebook-ultra-g1q/qccdsp8380.mbn` + `cdsp_dtbs.elf` | (c) | proprietary | **N** | `...\FileRepository\qcsubsys_ext_cdsp8380.inf_arm64_<hash>\{qccdsp8380.mbn, cdsp_dtbs.elf}` |
| CDSP, generic | `qcom/x1e80100/cdsp.mbn`, `cdsp_dtb.mbn` | (a) linux-firmware | `LICENSE.qcom` | Y, but useless | same caveat |
| PD-mapper service tables | `adspr.jsn`, `adsps.jsn`, `adspua.jsn`, `cdspr.jsn`, `battmgr.jsn` | **Not requested by mainline**: in-kernel `qcom_pd_mapper` has an `x1e80100_domains` table (matches `qcom,x1e80100`). Only the userspace `pd-mapper` daemon reads these. Generic copies exist in linux-firmware (`RawFile`, `LICENSE.qcom-2`). | n/a | Y (generic) | If ever needed: same `qcsubsys_ext_adsp8380` / `qcsubsys_ext_cdsp8380` packages (`battmgr.jsn` location *unverified*; extraction tools just `find` it) |
| Wi-Fi SKU A: WCN6855 / FastConnect 6900 (`ath11k`, PCI `17cb:1103`) | `ath11k/WCN6855/hw2.0/{amss.bin, m3.bin, board-2.bin, regdb.bin}` (hw2.1 symlinks to hw2.0) | (a) linux-firmware; Arch `linux-firmware-atheros` | `LICENCE.atheros_firmware` | Y | n/a. *Community report*: stock `board-2.bin` lacked the HP module's entry in 2024-12; a patched `board-2.bin` is attached to Launchpad bug 2084960 (attachment 5841336) and is what `setup_elite.sh` installs under `/lib/firmware/updates/`. Whether the 2025-2026 upstream `board-2.bin` updates absorbed it is unknown; verify on device |
| Wi-Fi SKU B: WCN7850 / FastConnect 7800 (`ath12k`, PCI `17cb:1107`) | `ath12k/WCN7850/hw2.0/{amss.bin, m3.bin, board-2.bin}` | (a) linux-firmware; Arch `linux-firmware-atheros` | `LICENCE.atheros_firmware` | Y | n/a. `board-2.bin` already has HP/Foxconn (`105b`) entries; *community report*: "worked out of the box" |
| Bluetooth, WCN6855 (`qcom,wcn6855-bt` on `uart14`) | `qca/wcnhpbtfw21.tlv` + `qca/wcnhpnv21.bin` (tried first), fallback `qca/hpbtfw21.tlv` + `qca/hpnv21.bin` / board-id variants (`btqca.c`) | (a) linux-firmware; Arch `linux-firmware-atheros` | `LICENSE.qcom` + `NOTICE.qca` | Y | n/a |
| Bluetooth, WCN7850 | `qca/hmtbtfw20.tlv` + `qca/hmtnv20.bin` | (a) | as above | Y | n/a |
| Audio: DSP | ADSP image above | (c) | | **N** | |
| Audio: topology | `qcom/x1e80100/X1E80100-HP-ELITEBOOK-ULTRA-G1Q-tplg.bin` (`audioreach_tplg_init`: `qcom/<driver_name>/<card name>-tplg.bin`) | **Missing**: linux-firmware ships only `qcom/x1e80100/X1E80100-HP-OMNIBOOK-X14-tplg.bin` (built from BSD-3 `linux-msm/audioreach-topology`). Same baseboard, so the X14 build should work under the G1q name; a G1q `.m4` exists only in the Codeberg fork (`nosuchthingascloud/audioreach-topology`, 2026-01-13), not upstream | `LICENCE.linaro` (X14 build), BSD-3 (source) | Y | n/a |
| Audio: codecs (`wcd9385`, `wsa8845` x2) | none | | | | |
| Video decode/encode (`iris`, `qcom,x1e80100-iris` -> `sm8550-iris`) | Nothing today: node `status = "disabled"` in `hamoa.dtsi` ("IRIS firmware is signed by vendors") and the HP DTSes do not enable it. Lenovo's pattern: `&iris { firmware-name = ".../qcvss8380.mbn"; status = "okay"; }` | (c) if enabled | proprietary | **N** | `...\FileRepository\qcdx8380.inf_arm64_<hash>\qcvss8380.mbn` (graphics driver package; also carries `qcav1e8380.mbn` for AV1) |
| Camera (CAMSS ISP) | Nothing: no CAMSS node for x1e80100 in mainline; no driver to request firmware | n/a | | | Blocked on kernel work, not firmware (community DT patch exists for the X14; untested here) |
| QUP serial engines | `qcom/x1e80100/qupv3fw.elf` exists in linux-firmware (`LICENSE.qcom`) but is loaded only when the DT sets `qcom,load-firmware`; HP DTS does not (UEFI pre-loads it) | (a) | | Y, unused | |

Reference: the ten files that `qcom-firmware-extract` (Debian/Ubuntu) pulls from
Windows are exactly `adsp_dtbs.elf adspr.jsn adsps.jsn adspua.jsn battmgr.jsn
cdsp_dtbs.elf cdspr.jsn qcadsp8380.mbn qccdsp8380.mbn qcdxkmsuc8380.mbn`. It does not
pull `qcvss8380.mbn`. It locates files with `find <mount>/Windows/System32/DriverStore/FileRepository
-name <file>` and takes the newest, so the exact `.inf_arm64_<hash>` directory names
above are informational; the `qcsubsys_ext_adsp8380` / `qcsubsys_ext_cdsp8380` /
`qcdx8380` package names come from Qualcomm's 8380 reference driver set.

## What the live ISO minimally needs

| Goal | Needs | Shippable? |
|---|---|---|
| Boot, framebuffer/DPU display on the eDP panel, keyboard, touchpad, NVMe, USB | no blobs (DPU, eDP PHY, PCIe, USB, HID need none) | Y |
| Wi-Fi | `linux-firmware-atheros` (both SKUs) — possibly the patched WCN6855 `board-2.bin` under `/lib/firmware/updates/` | Y (patched board file is a Qualcomm-licensed derivative; treat as an opt-in overlay) |
| Bluetooth | `linux-firmware-atheros` | Y |
| GPU acceleration (Hyprland) | `linux-firmware-qcom` **plus** `qcdxkmsuc8380.mbn` (c) | N without extraction |
| Audio, battery %, CDSP | ADSP/CDSP blobs (c) + topology (Y) | N without extraction |
| Video decode, camera | not enabled / no driver | out of scope |

So: with stock linux-firmware the live ISO can boot to the configurator with a
working display and network on both Wi-Fi SKUs. That is enough for the "live USB
before install" gate in the map. Whether the display stays up when the GPU probe fails
for lack of a zap shader is the one thing to confirm on hardware (the DPU is a
separate driver from the Adreno GPU within `msm`; expected to work, *unverified*).

## Recommendation

1. **Ship only category (a).** The ISO's offline mirror carries ALARM's
   `linux-firmware-qcom` and `linux-firmware-atheros` (20260810-2 already contains
   every relevant file). Add a tiny omarchy package (e.g.
   `omarchy-snapdragon-firmware-compat`) that installs
   `qcom/x1e80100/X1E80100-HP-ELITEBOOK-ULTRA-G1Q-tplg.bin` as a copy of the X14
   topology (licence `LICENCE.linaro`, redistributable) and, if the on-device check
   shows `ath11k` failing to match a board entry, the patched WCN6855 `board-2.bin`
   under `/lib/firmware/updates/ath11k/WCN6855/hw2.0/` with the hw2.1 symlink.
2. **Do not bake HP-signed blobs into a public ISO.** They carry no redistribution
   grant, HP has not contributed them to linux-firmware, and the one GitHub mirror that
   hosts them (`anonymix007/x1e-firmware`, has both `hp/omnibook-x14/` and
   `hp/elitebook-ultra-g1q/`) is unlicensed and stale. A personal, non-distributed
   build may include them; the public artifact must not.
3. **Extract on the device instead.** Port `qcom-firmware-extract` to Arch as a
   first-boot/installer step: find the Windows partition (BitLocker -> `dislocker`,
   else `ntfs-3g`), `find` the five kernel-requested files (plus the five `.jsn` for
   completeness), install into `/lib/firmware/qcom/x1e80100/<vendor>/<device>/`, add
   them to the initramfs (the community dracut config lists exactly the five files),
   reboot. Keep it model-agnostic by reading the target paths straight from the DTS
   `firmware-name` strings (`/proc/device-tree/.../firmware-name`) rather than a
   model table; that makes it work for any X1E/X2E laptop whose DT is upstream.
4. For the **live session** the same extraction can run before the GPU/remoteproc
   drivers probe (initramfs hook or build those drivers as modules loaded after the
   copy), which would give the live ISO GPU acceleration and audio without any
   redistributed blob. Defer that until the plain framebuffer path is proven.
5. Keep the GPU microcode at the linux-firmware version. *Community report*: a newer
   `gen70500_sqe.fw` from CodeLinaro crashed frequently on the G1q.
6. Video (`iris`) needs both a DT change and `qcvss8380.mbn`; treat as a separate
   Tier-2 ticket. Camera is blocked on a CAMSS driver, not firmware.

## Verify on the G1q (cheap, before any packaging)

* `lspci -nn -d 17cb:` — `1103` = WCN6855/ath11k SKU, `1107` = WCN7850/ath12k SKU
  (the DTS declares `pci17cb,1107` with `wcn6855` PMU/BT compatibles; HP ships both).
* `dmesg | grep -E 'ath1[12]k.*board'` — whether stock `board-2.bin` matches.
* `dmesg | grep -iE 'zap|adreno'` — display behaviour when the zap load fails.
* `ls .../FileRepository | grep -iE 'qcsubsys_ext_(a|c)dsp8380|qcdx8380'` from
  Windows or a mounted partition — confirm package directory names and where
  `battmgr.jsn` lives.

## Sources

Kernel (mainline `master`, 2026-08-24):
* https://raw.githubusercontent.com/torvalds/linux/master/arch/arm64/boot/dts/qcom/x1e80100-hp-elitebook-ultra-g1q.dts
* https://raw.githubusercontent.com/torvalds/linux/master/arch/arm64/boot/dts/qcom/x1e80100-hp-omnibook-x14.dts
* https://raw.githubusercontent.com/torvalds/linux/master/arch/arm64/boot/dts/qcom/x1-hp-omnibook-x14.dtsi
* https://raw.githubusercontent.com/torvalds/linux/master/arch/arm64/boot/dts/qcom/hamoa.dtsi (GPU `43050c01`, `iris` disabled, no CAMSS)
* https://raw.githubusercontent.com/torvalds/linux/master/arch/arm64/boot/dts/qcom/x1e78100-lenovo-thinkpad-t14s.dtsi (the `&iris` / `qcvss8380.mbn` pattern)
* `drivers/gpu/drm/msm/adreno/a6xx_catalog.c` (gen70500 SQE/GMU names), `drivers/bluetooth/btqca.c` (BT firmware names), `sound/soc/qcom/qdsp6/topology.c` (topology path), `drivers/soc/qcom/qcom_pd_mapper.c` (x1e80100 table), `drivers/media/platform/qcom/iris/iris_probe.c`
* G1q DT patch series: https://patchew.org/linux/20250416094236.312079-1-juerg.haefliger@canonical.com/
* X14 DT commit `6f18b8d4142c` (notes both WCN6855 and WCN7850 SKUs)

linux-firmware (`main`, 2026-08-24):
* https://gitlab.com/kernel-firmware/linux-firmware/-/tree/main/qcom/x1e80100 (vendor dirs: `ASUSTeK/`, `LENOVO/21N1`, `LENOVO/83ED`, `dell/*`, `hp/omnibook-x14` = topology only)
* https://gitlab.com/kernel-firmware/linux-firmware/-/raw/main/WHENCE (licence blocks: Adreno, Dell `LICENSE.dell`, Lenovo `LICENSE.qcom`, ath11k/ath12k `LICENCE.atheros_firmware`, qca BT, audioreach `LICENCE.linaro`, qupv3fw)
* Merge-request search for "elitebook", "omnibook", "x1e80100": no HP blob submission
* https://github.com/linux-msm/audioreach-topology (BSD-3 source of the `-tplg.bin` files; no G1q `.m4` upstream)

Distro packaging:
* https://archlinux.org/packages/core/any/linux-firmware-qcom/files/ and `linux-firmware-atheros` (file lists), http://mirror.archlinuxarm.org/aarch64/core/ (ALARM 20260810-2)
* https://launchpad.net/~ubuntu-concept/+archive/ubuntu/x1e (Ubuntu ships stock `linux-firmware` + `qcom-firmware-extract`; no HP blob package)
* https://salsa.debian.org/debian/qcom-firmware-extract/-/raw/main/qcom-firmware-extract and https://manpages.debian.org/unstable/qcom-firmware-extract/qcom-firmware-extract.8.en.html

Community (treated as reports, not ground truth):
* https://bugs.launchpad.net/ubuntu-concept/+bug/2093269 (G1q: two Wi-Fi SKUs, `setup_elite.sh`)
* https://bugs.launchpad.net/ubuntu-concept/+bug/2084960 (X14: patched WCN6855 `board-2.bin`, attachment 5841336)
* https://codeberg.org/nosuchthingascloud/x1e80100-hp-elitebook-ultra-g1q (`setup_elite.sh`, `qcom-firmware-hook`, dracut conf)
* https://codeberg.org/nosuchthingascloud/audioreach-topology (G1q topology `.m4`, 2026-01-13)
* https://github.com/anonymix007/x1e-firmware (unlicensed mirror of extracted HP blobs)
* https://github.com/alejandroqh/qcom-firmware-updater (which blobs live in Qualcomm's Windows graphics driver package: `qcdxkmsuc8380.mbn`, `qcvss8380.mbn`, `qcav1e8380.mbn`)
* https://github.com/Jeremiah-Hawley/Linux-on-Snapdragon and https://github.com/Jeremiah-Hawley/Lenovo-Yoga-Slim-7x-snapdragon-firmware (extraction workflow; all files from `C:\Windows\System32\DriverStore\FileRepository`)
* https://github.com/WOA-Project/Qualcomm-Reference-Drivers (8380 package names `qcsubsys_ext_adsp8380`, `qcsubsys_ext_cdsp8380`, `qcdx8380`)
