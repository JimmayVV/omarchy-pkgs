# ALARM `linux-aarch64` vs a `linux-x1e` kernel for Snapdragon X1E / X2

Research for [JimmayVV/omarchy-iso#2](https://github.com/JimmayVV/omarchy-iso/issues/2)
(map: [omarchy-iso#1](https://github.com/JimmayVV/omarchy-iso/issues/1)). Snapshot taken 2026-08-24.

## Question

Does Arch Linux ARM's stock `linux-aarch64` (7.2-2) enable the Snapdragon X1E80100 / X2 Elite
SoC drivers and ship `qcom/x1e80100-hp-elitebook-ultra-g1q.dtb` and the X2 DTBs? If not, what is
the minimal path — a config fragment on ALARM's PKGBUILD, or a `linux-x1e`-style PKGBUILD
tracking a community tree?

## Answer in one paragraph

**Yes for X1E, no for X2.** ALARM's `linux-aarch64` 7.2-2 is a mainline 7.2 tarball plus five
Rockchip/RPi patches, built with `CONFIG_ARCH_QCOM=y` and every X1E80100 driver the HP EliteBook
Ultra G1q's device tree needs (GPU, display, USB, PHYs, PMIC, RPMh, Wi-Fi, BT, audio, thermal,
UFS/NVMe, EFI stub). It builds all `qcom/*.dtb` and the shipped package contains
`boot/dtbs/qcom/x1e80100-hp-elitebook-ultra-g1q.dtb` (plus the `-el2` variant). Diffing Johan
Hovold's minimal `johan_defconfig` against ALARM's config finds **no missing X1E symbol**; the
differences are y-vs-m choices and generic options. For X2 Elite (SoC codename Glymur) the
picture is different: ALARM leaves `PINCTRL_GLYMUR`, `CLK_GLYMUR_*` and
`INTERCONNECT_QCOM_GLYMUR` unset, and mainline 7.2 has only the `glymur-crd` reference-board DTB
(the first retail X2 laptop DTB, ASUS Zenbook A16, is only in post-7.2 master). Recommendation:
**use ALARM's stock `linux-aarch64` for the X1E/G1q milestone — no kernel package at all — and
defer X2 to a small config-delta rebuild when an X2 DTB exists in the tarball ALARM ships.**
A `linux-x1e` package tracking a community tree is not justified by anything the G1q needs.

## Sources snapshot (what exactly was compared)

| Item | Value | Source |
| --- | --- | --- |
| ALARM package | `linux-aarch64` 7.2-2, built 2026-08-24 18:58 UTC, 64 MB | [archlinuxarm.org/packages/aarch64/linux-aarch64](https://archlinuxarm.org/packages/aarch64/linux-aarch64), mirror listing |
| ALARM PKGBUILD | `pkgver=7.2 pkgrel=2`, source `linux-7.2.tar.xz` + 5 patches (rk3399 pwm, pps x86 hack, rk3568 PCIe MSI revert, RP1 pl011, RPi5 UART) | [core/linux-aarch64/PKGBUILD](https://github.com/archlinuxarm/PKGBUILDs/blob/master/core/linux-aarch64/PKGBUILD) @ `11ee3096f` (2026-08-24) |
| ALARM config | "Linux/arm64 7.2.0 Kernel Configuration", 12 757 lines, `CONFIG_LOCALVERSION="-ARCH"` | [core/linux-aarch64/config](https://github.com/archlinuxarm/PKGBUILDs/blob/master/core/linux-aarch64/config) @ `11ee3096f` |
| DTB build/install | `make DTC_FLAGS="-@" dtbs` then `make INSTALL_DTBS_PATH=$pkgdir/boot/dtbs dtbs_install` — everything under `dtb-$(CONFIG_ARCH_QCOM)` | PKGBUILD `build()` / `_package()` |
| Mainline DTS list | `arch/arm64/boot/dts/qcom/Makefile` at tag `v7.2` and at `master` | [torvalds/linux v7.2](https://github.com/torvalds/linux/blob/v7.2/arch/arm64/boot/dts/qcom/Makefile) |
| Mainline defconfig | `arch/arm64/configs/defconfig` at `v7.2` | [torvalds/linux v7.2](https://github.com/torvalds/linux/blob/v7.2/arch/arm64/configs/defconfig) |
| jhovold minimal config | `arch/arm64/configs/johan_defconfig`, branch `wip/x1e80100-6.16` (429 lines) | [jhovold/linux wip/x1e80100-6.16](https://github.com/jhovold/linux/tree/wip/x1e80100-6.16) |
| G1q DT chain | `x1e80100-hp-elitebook-ultra-g1q.dts` → `x1e80100-hp-omnibook-x14.dts` → `x1-hp-omnibook-x14.dtsi` → `hamoa.dtsi` (`x1e80100.dtsi` was renamed `hamoa.dtsi` before 7.2) | torvalds/linux v7.2 |

None of ALARM's five patches touch Qualcomm code. ALARM's `linux-firmware` is 20260810 straight from
kernel-firmware ([PKGBUILD](https://github.com/archlinuxarm/PKGBUILDs/blob/master/core/linux-firmware/PKGBUILD)).

## CONFIG_ table: what the G1q needs vs ALARM 7.2-2

Column "need" is what the G1q DT chain and `johan_defconfig` require; "ALARM" is the literal
value in ALARM's `config`; "mainline defconfig" is v7.2 for reference. `m` is fine everywhere
because mkinitcpio (`kms`/`autodetect` hooks) puts modules in the initramfs.

### Boot / firmware interface

| Symbol | need | ALARM | mainline defconfig | note |
| --- | --- | --- | --- | --- |
| `CONFIG_ARCH_QCOM` | y | **y** | y | gates every `dtb-$(CONFIG_ARCH_QCOM)` |
| `CONFIG_EFI` / `EFI_STUB` / `EFI_GENERIC_STUB` | y | **y / y / y** | y | EFI boot |
| `CONFIG_EFI_ARMSTUB_DTB_LOADER` | – | not set | not set | `dtb=` cmdline loader is *off*; DTB must come from the bootloader (Limine `dtb_path`, see notes) |
| `CONFIG_EFI_ZBOOT` | – | not set | – | plain `Image`/`Image.gz` shipped |
| `CONFIG_ARM64_4K_PAGES` / `VA_BITS_48` | y | **y / y** | y | |
| `CONFIG_QCOM_SCM` | y | **y** | y | |
| `CONFIG_QCOM_TZMEM_MODE_GENERIC` / `_SHMBRIDGE` | either | **GENERIC** | SHMBRIDGE | defconfig flipped to SHM Bridge in 6.11 ("falls back to generic where unsupported", commit `f5a27053293f`); `johan_defconfig` does not set it. Not blocking. |
| `CONFIG_QCOM_QSEECOM` / `_UEFISECAPP` | y | **y / y** | y | EFI variables via TZ; `hp,elitebook-ultra-g1q` and `hp,omnibook-x14` are in `qcom_scm_qseecom_allowlist[]` at v7.2 |
| `CONFIG_ARM_SMMU` / `ARM_SMMU_QCOM` | y | **y / y** | y | |
| `CONFIG_QCOM_PDC` / `RESET_QCOM_PDC` | y | **y / y** | y | |
| `CONFIG_QCOM_CPUCP_MBOX` | y/m | **y** | m | cpufreq mailbox |
| `CONFIG_ARM_QCOM_CPUFREQ_HW` | y | **y** | y | |

### GPU / display

| Symbol | need | ALARM | mainline defconfig | note |
| --- | --- | --- | --- | --- |
| `CONFIG_DRM_MSM` | m | **m** | m | adreno + zap shader loader |
| `CONFIG_DRM_MSM_GPU_STATE` | y | **y** | | |
| `CONFIG_DRM_MSM_KMS` / `MDSS` / `DPU` | y | **y / y / y** | | DPU for X1E display |
| `CONFIG_DRM_MSM_DP` | y | **y** | | eDP + USB-C DP alt-mode |
| `CONFIG_DRM_MSM_DSI` (+ 7NM PHY) | y | **y** | | not used by G1q (eDP) but present |
| `CONFIG_DRM_PANEL_EDP` | m | **m** | m | G1q panel is generic `edp-panel` |
| `CONFIG_DRM_PANEL_SAMSUNG_ATNA33XC20` | m | **y** | | OLED variants of other laptops |
| `CONFIG_DRM_DISPLAY_DP_AUX_BUS` / `DP_HELPER` | y | **y / y** | | |
| `CONFIG_DRM_AUX_BRIDGE` / `AUX_HPD_BRIDGE` | y | **y / y** | | |
| `CONFIG_PHY_QCOM_EDP` | m | **m** | | eDP PHY |
| `CONFIG_CLK_X1E80100_GCC` / `TCSRCC` | y | **y / y** | y / y | |
| `CONFIG_CLK_X1E80100_DISPCC` / `GPUCC` / `CAMCC` | m | **m / m / m** | m | |
| `CONFIG_QCOM_UBWC_CONFIG` | m | **m** | | required by DRM_MSM since 6.17 |
| `CONFIG_BACKLIGHT_PWM` | y/m | **y** | | G1q uses `pwm-backlight` on `pmk8550-pwm` |
| `CONFIG_LEDS_QCOM_LPG` | m | **m** | | provides the PMIC PWM used by the backlight |
| `CONFIG_QCOM_PBS` | – | not set | | `hamoa-pmics.dtsi` has no `qcom,pbs` LPG reference at v7.2; not needed |
| `CONFIG_SYSFB` / `FB_EFI` | y | **y / y** | | firmware framebuffer until msm binds |

### USB / Type-C

| Symbol | need | ALARM | mainline defconfig | note |
| --- | --- | --- | --- | --- |
| `CONFIG_USB_DWC3` / `USB_DWC3_QCOM` / `USB_DWC3_DUAL_ROLE` | y | **y / y / y** | | |
| `CONFIG_PHY_QCOM_QMP_COMBO` / `QMP_USB` / `QMP_PCIE` / `QMP_UFS` | y | **y / y / y / y** | | USB3+DP combo, PCIe, UFS PHYs |
| `CONFIG_PHY_SNPS_EUSB2` | m | **m** | m | renamed from `PHY_QCOM_SNPS_EUSB2` |
| `CONFIG_PHY_QCOM_EUSB2_REPEATER` | m | **m** | m | |
| `CONFIG_TYPEC` / `TYPEC_UCSI` / `UCSI_PMIC_GLINK` | y/m | **y / m / m** | m | |
| `CONFIG_TYPEC_MUX_PS883X` | m | **m** | | G1q has `parade,ps8830` retimers |
| `CONFIG_TYPEC_MUX_GPIO_SBU` | m | **m** | | G1q has `onnn,fsusb42`/`gpio-sbu-mux` |
| `CONFIG_QCOM_PMIC_GLINK` | m | **m** | m | `qcom,x1e80100-pmic-glink` |
| `CONFIG_BATTERY_QCOM_BATTMGR` | m | **m** | m | battery reporting |

### PMIC / power / SoC infrastructure

| Symbol | need | ALARM | note |
| --- | --- | --- | --- |
| `CONFIG_SPMI` / `SPMI_MSM_PMIC_ARB` / `MFD_SPMI_PMIC` | y | **y / y / y** | pm8550, pm8550ve, pmc8380, pmk8550 |
| `CONFIG_REGULATOR_QCOM_RPMH` / `QCOM_SPMI` / `QCOM_REFGEN` | y | **y / y / m** | |
| `CONFIG_PINCTRL_MSM` / `PINCTRL_X1E80100` / `PINCTRL_QCOM_SPMI_PMIC` | y | **y / y / y** | |
| `CONFIG_PINCTRL_LPASS_LPI` / `PINCTRL_SM8550_LPASS_LPI` | m | **m / m** | X1E's LPASS TLMM uses the `qcom,sm8550-lpass-lpi-pinctrl` driver |
| `CONFIG_QCOM_RPMH` / `RPMHPD` / `COMMAND_DB` | y | **y / y / y** | |
| `CONFIG_QCOM_CLK_RPMH` / `COMMON_CLK_QCOM` | y | **y / y** | |
| `CONFIG_INTERCONNECT_QCOM` / `_X1E80100` / `_OSM_L3` | y | **y / y / m** | |
| `CONFIG_QCOM_SMEM` / `SMP2P` / `SMSM` / `AOSS_QMP` / `LLCC` | y | **y** (all) | |
| `CONFIG_RPMSG_QCOM_GLINK` / `_SMEM` / `RPMSG_CHAR` | y | **y / y / y** | |
| `CONFIG_QCOM_Q6V5_PAS` / `Q6V5_ADSP` / `RPROC_COMMON` / `PIL_INFO` / `MDT_LOADER` | m | **m** (all) | ADSP/CDSP remoteprocs |
| `CONFIG_QCOM_PD_MAPPER` / `PDR_HELPERS` / `APR` | m | **m / m / m** | audio protection-domain plumbing |
| `CONFIG_QCOM_GENI_SE` / `SERIAL_QCOM_GENI` / `I2C_QCOM_GENI` / `SPI_QCOM_GENI` | y | **y / y / y / y** | |
| `CONFIG_QCOM_GPI_DMA` / `QCOM_BAM_DMA` | m/y | **m / y** | |
| `CONFIG_QCOM_TSENS` / `QCOM_SPMI_TEMP_ALARM` / `QCOM_SPMI_ADC5` / `ADC_TM5` / `QCOM_LMH` | m | **m** (all) | thermal |
| `CONFIG_INPUT_PM8941_PWRKEY` / `RTC_DRV_PM8XXX` | y | **y / y** | |
| `CONFIG_QCOM_WDT` / `QCOM_STATS` / `QCOM_SOCINFO` | m | **m** | |

### Wi-Fi / Bluetooth / storage / input

| Symbol | need | ALARM | note |
| --- | --- | --- | --- |
| `CONFIG_ATH12K` | m | **m** | G1q `wifi@0` is `compatible = "pci17cb,1107"` = `WCN7850_DEVICE_ID 0x1107` in `ath12k/wifi7/pci.c`; not in ath11k's table (0x1101/0x1103/0x1104) |
| `CONFIG_ATH11K` / `ATH11K_PCI` | m | **m / m** | kept because the same board file names its PMU/BT nodes `qcom,wcn6855-pmu` / `qcom,wcn6855-bt` |
| `CONFIG_POWER_SEQUENCING` / `POWER_SEQUENCING_QCOM_WCN` | y/m | **y / m** | WCN PMU power-sequencing |
| `CONFIG_PCI_PWRCTRL_PWRSEQ` / `PCIE_QCOM` | m/y | **m / y** | Wi-Fi is behind `pcie4` |
| `CONFIG_BT_QCA` / `BT_HCIUART_QCA` | m/y | **m / y** | UART BT on `qcom,wcn6855-bt` |
| `CONFIG_SCSI_UFSHCD` / `_PLATFORM` / `SCSI_UFS_QCOM` | y | **y / y / y** | |
| `CONFIG_BLK_DEV_NVME` | y | **y** | G1q is NVMe |
| `CONFIG_I2C_HID_OF` / `HID_MULTITOUCH` | m | **m / m** | keyboard + touchpad are generic `hid-over-i2c` |
| `CONFIG_I2C_HID_OF_ELAN` | – | not set | not referenced by the G1q DT; jhovold sets it for other laptops |
| `CONFIG_KEYBOARD_GPIO` | m | **m** | `gpio-keys` (lid) |
| `CONFIG_MMC_SDHCI_MSM` | y | **y** | |

### Audio (audioreach / q6)

| Symbol | need | ALARM | note |
| --- | --- | --- | --- |
| `CONFIG_SND_SOC_QCOM` / `QCOM_COMMON` / `QCOM_SDW` | m | **m / m / m** | |
| `CONFIG_SND_SOC_X1E80100` | m | **m** | `qcom,x1e80100-sndcard` |
| `CONFIG_SND_SOC_QDSP6` + `_APM` / `_PRM` / `_AFE` / `_APM_LPASS_DAI` / `_PRM_LPASS_CLOCKS` | m | **m** (all) | audioreach |
| `CONFIG_SOUNDWIRE` / `SOUNDWIRE_QCOM` | m | **m / m** | |
| `CONFIG_SND_SOC_WCD938X` / `_SDW` | m | **m / m** | `qcom,wcd9385-codec` |
| `CONFIG_SND_SOC_WSA884X` | m | **m** | `sdw20217020400` speaker amps |
| `CONFIG_SND_SOC_LPASS_{VA,WSA,RX,TX}_MACRO` / `MACRO_COMMON` | m | **m** (all) | |

### Camera / video (Tier-2, informational)

| Symbol | ALARM | note |
| --- | --- | --- |
| `CONFIG_VIDEO_QCOM_CAMSS` | m | ISP driver present; whether X1E80100 CAMSS + the G1q sensor are supported is a separate ticket |
| `CONFIG_VIDEO_QCOM_VENUS` | m | |
| `CONFIG_VIDEO_QCOM_IRIS` | not set | X1E video decoder (`iris`) — the only X1E driver ALARM leaves off; not needed to boot |
| `CONFIG_VIDEO_OV5675` | not set | jhovold enables it for the T14s camera |

### X2 Elite (Glymur) — the gap

| Symbol | ALARM | mainline defconfig | note |
| --- | --- | --- | --- |
| `CONFIG_PINCTRL_GLYMUR` | **not set** | (in `Kconfig.msm`) | TLMM pinctrl — without it nothing on the SoC probes |
| `CONFIG_CLK_GLYMUR_GCC` | **not set** | y | |
| `CONFIG_CLK_GLYMUR_DISPCC` / `TCSRCC` | **not set** | m / m | |
| `CONFIG_CLK_GLYMUR_GPUCC` / `VIDEOCC` | **not set** | – | |
| `CONFIG_INTERCONNECT_QCOM_GLYMUR` | **not set** | y | |
| `CONFIG_EC_QCOM_HAMOA` | not set | | EC driver for Qualcomm Hamoa/Purwa/Glymur *reference* boards only |
| `CONFIG_SND_SOC_GLYMUR` | (does not exist in 7.2) | | |

Glymur support that *is* compiled in generically at 7.2 (per code search on master): `clk-rpmh.c`,
`dpu_12_2_glymur.h` catalog in DRM_MSM, `dp_display.c`, `msm_mdss.c`, `arm-smmu-qcom.c`,
`phy-qcom-{edp,qmp-combo,qmp-pcie,qmp-pcie-multiphy,qmp-usb}.c`, `rpmhpd.c`, `llcc-qcom.c`,
`pmic_glink.c`, `qcom_pd_mapper.c`, `ubwc_config.c`, `spmi-pmic-arb.c`, `ucsi_glink.c`,
`qcom_battmgr.c`, and adreno `a8xx_gpu.c` (`ADRENO_8XX_GEN1/GEN2`, `adreno_is_x285`). So an X2
kernel is ALARM's config plus the six symbols above, not a different tree.

### `johan_defconfig` (6.16) vs ALARM — full delta, filtered to Qualcomm-relevant

Not present in ALARM: `CONFIG_MFD_QCOM_PM8008=m` / `REGULATOR_QCOM_PM8008=m` (camera PMIC, and
jhovold's own branch carries "x1e80100-pmics: Disable pm8010 by default"), `I2C_HID_OF_ELAN=m`,
`VIDEO_OV5675=m`, `RESET_GPIO=m`, `SM_VIDEOCC_8350=m` (not X1E), `FW_LOADER_COMPRESS_ZSTD=y`,
`FW_LOADER_USER_HELPER=y`, `EFI_CAPSULE_LOADER=y`, `TRANSPARENT_HUGEPAGE`, `NUMA`,
`RANDOMIZE_BASE`. Everything else jhovold lists is `y` or `m` in ALARM (18 symbols are built-in in
ALARM where jhovold has `m`: `PCIE_QCOM`, `I2C/SPI_QCOM_GENI`, `QCOM_TSENS`, `DRM`,
`SCSI_UFS_QCOM`, `CLK_X1E80100_{CAMCC,TCSRCC}`, `QCOM_CPUCP_MBOX`, `RPMSG_QCOM_GLINK_SMEM`,
`QCOM_LLCC`, `RESET_QCOM_PDC`, `PHY_QCOM_QMP`, `PHY_QCOM_USB_SNPS_FEMTO_V2`, …). None of the
absent items is on the G1q's boot path. `FW_LOADER_COMPRESS_ZSTD` only matters for
zstd-compressed firmware; ALARM's `linux-firmware` does not compress, and ALARM has `_XZ=y`.

## DTB presence

`arch/arm64/boot/dts/qcom/Makefile` at **v7.2** (the tarball ALARM builds) lists 56 X1/X2 lines,
including:

```
x1e80100-hp-elitebook-ultra-g1q-el2-dtbs := x1e80100-hp-elitebook-ultra-g1q.dtb x1-el2.dtbo
dtb-$(CONFIG_ARCH_QCOM) += x1e80100-hp-elitebook-ultra-g1q.dtb x1e80100-hp-elitebook-ultra-g1q-el2.dtb
```

X1E80100 boards at v7.2: `x1e001de-devkit`, `x1e78100-lenovo-thinkpad-t14s(-oled)`,
`x1e80100-{asus-vivobook-s15, asus-zenbook-a14, crd, dell-inspiron-14-plus-7441,
dell-latitude-7455, dell-xps13-9345, hp-elitebook-ultra-g1q, hp-omnibook-x14, lenovo-yoga-slim7x,
medion-sprchrgd-14-s1, microsoft-denali-oled, microsoft-romulus13, microsoft-romulus15, qcp}`,
`hamoa-iot-evk`, `hamoa-lenovo-ideacentre-mini-01q8x10`. X1P42100 (X Plus): `asus-vivobook-s15`,
`asus-zenbook-a14(-lcd)`, `crd`, `hp-omnibook-x14`, `lenovo-thinkbook-16`, `purwa-iot-evk`;
X1P64100: `microsoft-denali`. **X2 (Glymur) at v7.2: `glymur-crd` only.** Post-7.2 master adds
`glymur-asus-zenbook-a16-ux3607oa`, `x1e80100-honor-magicbook-art-14`, `x1p42100-microsoft-sp12in`.
Every board also gets an `-el2.dtb` overlay build (`x1-el2.dtbo`, for running the kernel at EL2).

The G1q DTS (`x1e80100-hp-elitebook-ultra-g1q.dts`, first commit `2377626fd216`, 2025-10-30) is
a 30-line overlay on the OmniBook X14: it sets `model`, `compatible = "hp,elitebook-ultra-g1q",
"qcom,x1e80100"`, the zap/ADSP/CDSP firmware paths, and the sound-card model.

**Confirmed in the shipped package.** Streaming `tar -tJf` over
`http://mirror.archlinuxarm.org/aarch64/core/linux-aarch64-7.2-2-aarch64.pkg.tar.xz` (64 MB,
2026-08-24) lists 396 `boot/dtbs/qcom/*.dtb` including:

```
boot/dtbs/qcom/x1e80100-hp-elitebook-ultra-g1q.dtb
boot/dtbs/qcom/x1e80100-hp-elitebook-ultra-g1q-el2.dtb
boot/dtbs/qcom/x1e80100-hp-omnibook-x14.dtb
boot/dtbs/qcom/x1p42100-hp-omnibook-x14.dtb
boot/dtbs/qcom/x1e80100-crd.dtb
boot/dtbs/qcom/x1e001de-devkit.dtb
boot/dtbs/qcom/glymur-crd.dtb
boot/dtbs/qcom/hamoa-iot-evk.dtb
boot/dtbs/qcom/hamoa-lenovo-ideacentre-mini-01q8x10.dtb
```

Install path convention: `/boot/dtbs/qcom/<board>.dtb`, `/boot/Image`, `/boot/Image.gz`.

## Community trees — what each adds beyond mainline (Aug 2026)

| Tree | State | Base | Carries beyond mainline | Verdict for us |
| --- | --- | --- | --- | --- |
| **jhovold/linux** `wip/x1e80100-6.16` ([tree](https://github.com/jhovold/linux/tree/wip/x1e80100-6.16), [T14s wiki](https://github.com/jhovold/linux/wiki/T14s)) | Newest wip branch is **6.16** (branch commit 2025-07-28, repo last push 2025-09-19). No 6.17–7.2 wip branches exist. | v6.16 | 48 commits: ath11k/ath12k ring-buffer + key-install fixes, `phy-snps-eusb2` cleanups, `PCI: qcom` modular build, Venus on sc8280xp, T14s eDP/BT/fingerprint DTS bits, a `hack: efi/libstub` T14s `exit_boot_services()` mitigation, `johan_defconfig`. Wiki notes cmdline `clk_ignore_unused pd_ignore_unused efi=noruntime` (T14s firmware bug) and linux-firmware ≥ 20250613. | Historically the reference tree; the fixes have since flowed upstream (7.2 is 4 releases later). Nothing G1q-specific. Stale as a base. |
| **jglathe/linux_ms_dev_kit** `jg/ubuntu-qcom-x1e-7.2.y` ([repo](https://github.com/jglathe/linux_ms_dev_kit)) | Active (pushed 2026-08-22). Base is now Linaro's arm64-laptops tree, "config is basically Ubuntu Concept X1E". | v7.2 | **710 commits ahead of v7.2** (first 250 inspected): ~131 DTS commits for not-yet-upstream boards (Ideapad 5/Slim 3/Slim 5x, ThinkBook 16, Acer Swift, Surface Pro 12, HP OmniBook 5 X1P), Dell XPS EC driver + Hamoa/Glymur/CRD EC nodes, ATNA40CT01/BOE/CMN eDP panels, DP audio guards, `qmp-combo` PM fixes, `clk sync_state` series, `msm.psr` bits, Ubuntu packaging. HP items are OmniBook 5 (X1P) and generic HPD pinctrl — **nothing for the G1q/OmniBook X14**, whose DT is upstream. | Valuable for laptops with no upstream DT. For the G1q it is 700 patches of other people's hardware. |
| **lkarlslund/linux-x1e-jglathe** ([repo](https://github.com/lkarlslund/linux-x1e-jglathe)) | Active (2026-08-16) | jglathe pinned commits (`7.1.7-jg-1`, `7.2-rc6-jg-0`) + a mainline `7.2-rc7` "edge" | Four side-by-side Arch packages (`linux-x1e-jens71/72/72-pdc`, `linux-x1e-t14s-edge`), each with own modules dir, DTB dir, systemd-boot entry; `running.config`; cmdline `rw id_aa64mmfr0.ecv=1 quiet splash msm.psr_enabled=1`; PDC/SS3 v4 suspend series. T14s-only. | The best template *if* we ever need a `linux-x1e` PKGBUILD (immutable commit pins, coexisting variants). |
| **anonymix007/x1e-alarm** ([repo](https://github.com/anonymix007/x1e-alarm)) | Last push 2025-07-04 | own fork of jhovold `wip/x1e80100-6.16-rc4` | `linux-x1e` PKGBUILD cloned from ALARM's `linux-aarch64` (same header), explicit `_dtbs` list incl. `x1e80100-hp-elitebook-ultra-g1q`, plus `x1e-firmware`, `x1e-uki`, `fastrpc`, `x1e-keyring`. | Precedent that a `linux-x1e` PKGBUILD is ~ALARM's PKGBUILD with a different `source=` — and that it goes stale once its maintainer stops. |
| **ironrobin** ([codeberg.org/ironrobin/aarch64](https://codeberg.org/ironrobin/aarch64), [archiso-x13s](https://codeberg.org/ironrobin/archiso-x13s)) | GitHub repos archived 2026-08-03, moved to Codeberg; `archiso-x13s` now uses Arch Linux Ports repos instead of ALARM | – | Packages: `linux-x13s`, `linux-volterra`, `dtbsync`. **No X1E kernel package.** | Not an X1E source. Note the ALARM → Arch Ports switch as a signal. |
| **Arch Linux Ports aarch64** ([ports.archlinux.page/aarch64](https://ports.archlinux.page/aarch64/)) | Unofficial, pre-RFC | mainline | Their `linux` "intends to support devices with mainline Linux support"; ASUS Vivobook S15 and Dell XPS13 9345 (both X1E) listed as confirmed working on the *stock* port kernel, X Elite generally "suspected to work but unconfirmed". | Independent evidence that mainline-config kernels boot X1E laptops. Map's standing preference stays ALARM. |
| AUR | – | – | No `linux-x1e*` / `x1e80100` packages found (AUR RPC search, 2026-08-24). | – |

Note on the ALARM forum thread [T14s Gen6](https://archlinuxarm.org/forum/viewtopic.php?f=67&t=17037):
its "stock kernel does not boot" reports are from Aug–Nov 2024 (kernel 6.10/6.11, before the T14s
DT and most X1E drivers were upstream). They do not describe 7.2.

## Recommendation

**Use ALARM's stock `linux-aarch64` unchanged for the G1q / X1E milestone. Do not create a
kernel package in `omarchy-pkgs` yet.**

Rationale:

1. Every driver the G1q DT references is enabled in ALARM 7.2-2, and the DTB ships in the
   package. There is no config delta to carry. A `linux-x1e` PKGBUILD would exist only to be
   different, and the two existing ones (anonymix007, lkarlslund) show the cost: a ~64 MB kernel
   rebuild per upstream release, on the G1q's own WSL2, forever.
2. The community trees add nothing for this board. jhovold's fixes landed upstream before 7.2 and
   his branch stopped at 6.16; jglathe's 710 patches are DTs/ECs for laptops without upstream
   support plus Ubuntu packaging. The G1q's DT has been upstream since 6.17 and its `compatible`
   is in the qseecom allowlist.
3. It keeps #121's design intact: the aarch64 ISO ships `linux-aarch64`; the Snapdragon layer
   becomes bootloader DTB selection + firmware + quirks, which is model-agnostic by construction
   (396 qcom DTBs are already on disk; selection is a naming rule, not a package).
4. It aligns with the destination's "other X1/X2 laptops can follow": any laptop with an upstream
   DT works the day ALARM bumps the tarball.

**Minimal path if a delta ever becomes necessary** (X2, or a driver ALARM leaves off): a
`linux-aarch64` *rebuild* in `omarchy-pkgs` — copy ALARM's PKGBUILD + `config` verbatim, then in
`prepare()` apply a fragment with `scripts/kconfig/merge_config.sh` (or `scripts/config --enable`)
and `make olddefconfig`. Do not fork a source tree. The fragment for X2 today would be:

```
CONFIG_PINCTRL_GLYMUR=y
CONFIG_CLK_GLYMUR_GCC=y
CONFIG_CLK_GLYMUR_TCSRCC=y
CONFIG_CLK_GLYMUR_DISPCC=m
CONFIG_CLK_GLYMUR_GPUCC=m
CONFIG_CLK_GLYMUR_VIDEOCC=m
CONFIG_INTERCONNECT_QCOM_GLYMUR=y
# optional, matches mainline defconfig / jhovold
CONFIG_QCOM_TZMEM_MODE_SHMBRIDGE=y
CONFIG_VIDEO_QCOM_IRIS=m
```

but it is pointless until an X2 laptop DTB is in the tarball ALARM builds (7.3+ for the ASUS
Zenbook A16), so X2 stays in "Not yet specified". The cheaper alternative is a one-file PR to
`archlinuxarm/PKGBUILDs` enabling the Glymur symbols — the map's no-upstream-contact rule is about
omacom-io, not ALARM — so this is a judgment call for when X2 becomes ticketable.

Choose a `linux-x1e`-style tree-tracking package **only** if hands-on testing shows a G1q
regression that upstream has not fixed by the next 7.x; then pin an immutable commit the
lkarlslund way, never a moving branch.

## Findings for adjacent tickets (not this ticket's question)

- **Firmware (blocking, different ticket).** The G1q DTS names
  `qcom/x1e80100/hp/elitebook-ultra-g1q/{qcdxkmsuc8380.mbn, qcadsp8380.mbn, adsp_dtbs.elf,
  qccdsp8380.mbn, cdsp_dtbs.elf}`. Upstream linux-firmware (`main`, and ALARM's 20260810 build)
  has only `qcom/x1e80100/hp/omnibook-x14/X1E80100-HP-OMNIBOOK-X14-tplg.bin` under `hp/`; no
  `elitebook-ultra-g1q/` directory and no HP zap/ADSP/CDSP blobs. GPU, audio and remoteprocs will
  need those files extracted from the Windows install (`C:\Windows\System32\DriverStore\...`)
  and packaged. Wi-Fi firmware is generic (`ath12k/WCN7850/hw2.0`, present).
- **Boot chain.** `CONFIG_EFI_ARMSTUB_DTB_LOADER` is off in ALARM, so `dtb=` on the kernel
  command line does not work; the bootloader must hand the DTB over as the EFI DT config table.
  Limine's `CONFIG.md` documents `dtb_path` for the Linux protocol ("A device tree blob to pass
  instead of the one provided by the firmware"). ALARM builds DTBs with `-@` (symbols), so
  `-el2.dtb` / overlays are usable.
- **Kernel command line hints from people booting X1E on 7.x:** lkarlslund uses
  `id_aa64mmfr0.ecv=1 msm.psr_enabled=1`; jhovold's 6.16 T14s notes list
  `clk_ignore_unused pd_ignore_unused efi=noruntime`. Which of these the G1q needs is a
  live-USB experiment.
- **X1E80100 `.dtsi` rename.** Upstream renamed `x1e80100.dtsi` → `hamoa.dtsi` and
  `x1e80100-pmics.dtsi` → `hamoa-pmics.dtsi` before 7.2; board DTB *filenames* are unchanged, so
  a `/proc/device-tree/compatible`-based selector keyed on `hp,elitebook-ultra-g1q` is stable.

## Sources

- ALARM PKGBUILD: https://github.com/archlinuxarm/PKGBUILDs/blob/master/core/linux-aarch64/PKGBUILD (commit `11ee3096f`, 2026-08-24 "core/linux-aarch64 to 7.2-2")
- ALARM config: https://github.com/archlinuxarm/PKGBUILDs/blob/master/core/linux-aarch64/config (same commit)
- ALARM package page: https://archlinuxarm.org/packages/aarch64/linux-aarch64 ; package listed via `tar -tJf` of http://mirror.archlinuxarm.org/aarch64/core/linux-aarch64-7.2-2-aarch64.pkg.tar.xz
- ALARM linux-firmware PKGBUILD: https://github.com/archlinuxarm/PKGBUILDs/blob/master/core/linux-firmware/PKGBUILD
- Mainline v7.2 qcom DTS Makefile: https://github.com/torvalds/linux/blob/v7.2/arch/arm64/boot/dts/qcom/Makefile ; master: https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/qcom/Makefile
- G1q DTS: https://github.com/torvalds/linux/blob/v7.2/arch/arm64/boot/dts/qcom/x1e80100-hp-elitebook-ultra-g1q.dts ; board dtsi: https://github.com/torvalds/linux/blob/v7.2/arch/arm64/boot/dts/qcom/x1-hp-omnibook-x14.dtsi ; PMICs: https://github.com/torvalds/linux/blob/v7.2/arch/arm64/boot/dts/qcom/hamoa-pmics.dtsi
- Mainline v7.2 arm64 defconfig: https://github.com/torvalds/linux/blob/v7.2/arch/arm64/configs/defconfig
- Kconfig files consulted at v7.2: `drivers/clk/qcom/Kconfig`, `drivers/pinctrl/qcom/Kconfig(.msm)`, `drivers/interconnect/qcom/Kconfig`, `drivers/phy/qualcomm/Kconfig`, `drivers/firmware/qcom/Kconfig`, `drivers/power/sequencing/Kconfig`, `drivers/platform/arm64/Kconfig`, `sound/soc/qcom/Kconfig`, `drivers/gpu/drm/msm/Kconfig`
- qseecom allowlist: https://github.com/torvalds/linux/blob/v7.2/drivers/firmware/qcom/qcom_scm.c (`qcom_scm_qseecom_allowlist[]`)
- SHM Bridge defconfig commit: https://github.com/torvalds/linux/commit/f5a27053293f
- ath12k WCN7850 ID: https://github.com/torvalds/linux/blob/v7.2/drivers/net/wireless/ath/ath12k/wifi7/pci.c ; ath11k IDs: https://github.com/torvalds/linux/blob/v7.2/drivers/net/wireless/ath/ath11k/pci.c
- Adreno X2: https://github.com/torvalds/linux/blob/v7.2/drivers/gpu/drm/msm/adreno/adreno_gpu.h ; `a8xx_gpu.c`
- jhovold: https://github.com/jhovold/linux/tree/wip/x1e80100-6.16 ; compare `torvalds:v6.16...jhovold:wip/x1e80100-6.16` (48 commits) ; wiki https://github.com/jhovold/linux/wiki/T14s ; `arch/arm64/configs/johan_defconfig`
- jglathe: https://github.com/jglathe/linux_ms_dev_kit (branch `jg/ubuntu-qcom-x1e-7.2.y`, compare vs `torvalds:v7.2`: 710 ahead)
- lkarlslund: https://github.com/lkarlslund/linux-x1e-jglathe (README, `packages/linux-x1e-jens72/PKGBUILD`)
- anonymix007: https://github.com/anonymix007/x1e-alarm (`linux-x1e/PKGBUILD`)
- ironrobin: https://github.com/ironrobin/archiso-x13s (archived 2026-08-03) ; https://codeberg.org/ironrobin/aarch64 ; https://codeberg.org/ironrobin/archiso-x13s
- Arch Linux Ports aarch64: https://ports.archlinux.page/aarch64/
- ALARM forum, T14s Gen6 (2024): https://archlinuxarm.org/forum/viewtopic.php?f=67&t=17037
- AUR RPC search `x1e`, `x1e80100`, `snapdragon`: https://aur.archlinux.org/rpc/v5/search/x1e (no results)
- linux-firmware tree: https://gitlab.com/kernel-firmware/linux-firmware/-/tree/main/qcom/x1e80100/hp ; WHENCE lines 9805–9806
- Limine `dtb_path`: https://github.com/limine-bootloader/limine/blob/trunk/CONFIG.md
