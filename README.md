# Xiaomi SDM845 Kernel Builder with OriginSU

[![Build](https://img.shields.io/badge/build-GitHub%20Actions-blue?logo=github-actions&logoColor=white)](https://github.com/1992Ade/sdm845-origin-builder/actions/workflows/build.yml)
[![Latest release](https://img.shields.io/github/v/release/1992Ade/sdm845-origin-builder?label=latest&logo=github)](https://github.com/1992Ade/sdm845-origin-builder/releases/latest)

> **Your warranty is now void.** Flashing a custom kernel can brick your device. You are responsible for what you flash. Always keep a backup of your stock `boot` partition.

## 📱 About This Repository

This repository automates the compilation of a custom **Linux 4.9** kernel for **Xiaomi Snapdragon 845 (SDM845)** devices — Mi 8, POCO F1, Mi MIX 2S and their siblings. It is based on the LineageOS-derived kernel tree by **[Coconutat](https://github.com/Coconutat/android_kernel_xiaomi_sdm845_lineageos_Exp)**, which already ships **SuSFS** and its inline hooks (originally used with BakaSU).

Every build integrates the **[OriginSU](https://github.com/samakshkambxj/OriginSU)** kernel-based root solution in place of BakaSU — together with **SuSFS**, **Baseband Guard**, **NoMount** and **Droidspaces** (see [Included features](#-included-features)) — and produces **one recovery-flashable `.zip` per device** (TWRP / OrangeFox) with AnyKernel3.

**Fully automated:** a GitHub Actions workflow checks every week (Sunday) for a new OriginSU release. As soon as a new version is out **with its manager APK**, the kernel is rebuilt for every device with that exact OriginSU version and a new GitHub Release is published automatically — no manual action needed.

**Always the latest LineageOS:** each run picks the most recent LineageOS version available in the Coconutat tree (today `lineage-22.2-BakaSU`, LineageOS 22.2 / Android 15). When a newer LineageOS branch with SuSFS appears (e.g. LineageOS 23 / Android 16), the next weekly run switches to it on its own and rebuilds the current release.

---

## 🧩 Included Features

On top of OriginSU, every build enables the same hardening/usability modules as the base [perf_neon-builder](https://github.com/riarumoda/perf_neon-builder):

| Feature | Upstream | What it does | Kernel config |
|---|---|---|---|
| **SuSFS** | [simonpunk](https://gitlab.com/simonpunk/susfs4ksu) | Hides root/mounts from detection (already in the Coconutat tree) | `CONFIG_KSU_SUSFS_*` |
| **Baseband Guard** | [vc-teahouse](https://github.com/vc-teahouse/Baseband-guard) | LSM that blocks writes to the modem/baseband partitions | `CONFIG_BBG=y` |
| **NoMount** | [maxsteeel](https://github.com/maxsteeel/nomount) | Prevents unwanted mounts used for root detection | `CONFIG_NOMOUNT=y` |
| **Droidspaces** | [ravindu644](https://github.com/ravindu644/Droidspaces-OSS) | Namespace/cgroup isolation to run isolated Android/Linux spaces | `CONFIG_NAMESPACES`, `CONFIG_*_NS`, cgroups, netfilter… |

> **Droidspaces note:** it enables `CONFIG_CFS_BANDWIDTH=y` and applies two non-GKI patches. On this **4.9** tree it is the feature most likely to need adjustment; if a build breaks in the *Extra features* step, Droidspaces is the first suspect. If you use it with a systemd-based Linux space, mask the `NetworkManager` and `wpa_supplicant` services inside that space (as noted upstream).

---

## ✅ Supported Devices & ROMs

One zip is built per device. **Flash only the zip whose codename matches your phone** (each zip refuses to install on any other device).

| Device | Codename | Test status |
|---|---|---|
| Xiaomi Mi 8 | `dipper` | ✅ **Working** (tested by the maintainer) |
| Pocophone F1 / POCO F1 | `beryllium` | 🧪 Primary — please test |
| Xiaomi Mi MIX 2S | `polaris` | 🧪 Primary — please test |
| Xiaomi Mi 8 Pro | `equuleus` | ⚠️ Built, untested |
| Xiaomi Mi MIX 3 | `perseus` | ⚠️ Built, untested |
| Xiaomi Mi 8 Explorer Edition | `ursa` | ⚠️ Built, untested |

> ### 🧪 Testing status
> **`dipper` (Mi 8)** is tested and confirmed working. The other builds are produced automatically and have **not all been verified on real hardware**: **`beryllium` (POCO F1)** and **`polaris` (Mi MIX 2S)** still need testing, and the remaining variants (`equuleus`, `perseus`, `ursa`) are built because the Coconutat tree provides their config but are completely untested — **back up your stock `boot` and flash at your own risk.**
> If you test any of the untested devices, please open an issue reporting whether it boots and whether Wi-Fi / mobile network / camera work.

The kernel comes from the LineageOS kernel tree, so it works with ROMs built on the LineageOS `sdm845` 4.9 kernel. Each release states the LineageOS version it was built for (also in the zip name, e.g. `LOS22.2`) — **flash the release that matches the LineageOS / /e/OS version on your phone**. Currently:

| ROM | Status | Links |
|---|---|---|
| **LineageOS 22.2** (Android 15) | ✅ Built from the LineageOS source | [LineageOS wiki](https://wiki.lineageos.org/devices/dipper/) |
| **/e/OS** (Android 15, community) | ✅ /e/OS is built on LineageOS 22 | [/e/OS docs](https://doc.e.foundation/devices/dipper/) |

Other ROMs may work if they use the LineageOS `sdm845` 4.9 kernel and the same vendor firmware, but they are not tested. Stock MIUI is **not** supported.

> Make sure your device is on the vendor firmware LineageOS asks for (see the *"Update to a newer vendor firmware version"* section of the LineageOS wiki).

---

## 📦 Which zip should I download?

Each release ships **one zip per device**, named:

```
sdm845-<codename>-LOS<lineage>-OriginSU-<version>-susfs-<date>.zip
```

For example `sdm845-dipper-LOS22.2-OriginSU-v1.2.3-susfs-20261115.zip` for the Mi 8. Only the **SuSFS** variant is built: the Coconutat tree has its KernelSU hooks written for SuSFS. The full kernel `.config` of each device is attached to the release for reference.

---

## 🔧 Installation

0. **Coming from Magisk, APatch, BakaSU or another KernelSU?** Read [this warning](#️-coming-from-another-root-solution) first.
1. Back up your stock `boot` partition (from TWRP/OrangeFox, or keep the `boot.img` from your LineageOS / /e/OS zip).
2. Download **the zip matching your device codename** from the [latest release](https://github.com/1992Ade/sdm845-origin-builder/releases/latest).
3. Reboot to recovery and flash it, or use `adb sideload <file>.zip`.
4. Reboot to system.
5. Install the **OriginSU manager** APK (`arm64-v8a` or `universal`) from the OriginSU release linked in the release notes — it is always the same version as the kernel.

> **Updating the ROM?** A LineageOS / /e/OS OTA update replaces the `boot` partition, so the OriginSU kernel is removed. Flash the zip again after each ROM update.

### ↩️ Restore the stock kernel

```sh
su -c rm -rf /data/adb/*        # remove root data first
adb reboot bootloader
fastboot flash boot stock_boot.img
fastboot reboot
```

---

## ⚙️ How the Build Works

The workflow lives in [`.github/workflows/build.yml`](.github/workflows/build.yml) and runs three jobs:

```
check  ──►  build (matrix: 6 devices)  ──►  release
```

1. **check**
   - Reads the latest stable OriginSU release and makes sure it ships the **manager APK** (otherwise nothing is built, and the next weekly run tries again).
   - Picks the kernel branch: among the `lineage-<version>[-suffix]` branches of the Coconutat tree, the highest LineageOS version first, and for the same version the most recently updated branch. A branch is kept only if it contains SuSFS, its inline hooks and the shared SDM845 configs.
   - Builds if there is no `originsu-<tag>` release yet, or if the existing one was built on an older LineageOS branch. Otherwise the run stops there.
2. **build** (matrix: `beryllium`, `dipper`, `equuleus`, `polaris`, `perseus`, `ursa` — in parallel, `fail-fast: false`)
   - Clones the selected Coconutat SDM845 kernel branch.
   - Downloads **AOSP Clang r547379**, the toolchain recommended by the kernel's author.
   - Integrates OriginSU at the exact release tag with its official `setup.sh` (plus small 4.9 compatibility fixes applied to OriginSU's sources only), then checks that every SuSFS inline hook OriginSU needs is in the tree.
   - Generates the config from `vendor/xiaomi/mi845_defconfig` + `extra.config` + `<device>.config`, then enables the **extra features** (Baseband Guard, NoMount, Droidspaces — see [above](#-included-features)).
   - Packages like Coconutat: osm0sis AnyKernel3 + the tree's own `tools/AnyKernel3_sh/anykernel.sh`, with both `Image.gz` and `Image.gz-dtb`. AnyKernel3 flashes `Image.gz` and **keeps the DTB of your ROM's `boot.img`**. Each zip installs on its own device only.
3. **release** — publishes a GitHub Release `originsu-<tag>` with every device zip, the configs and release notes (OriginSU version, LineageOS version, kernel branch and commit, kernel version, SuSFS version, the per-device zip table and the build date).

### Schedule

| Trigger | Behaviour |
|---|---|
| Every Sunday at 05:37 UTC | Builds **only** if a new OriginSU release (with its manager) is out, or a newer LineageOS branch appeared |
| Manual (`Run workflow`) | Always builds; you can pick an OriginSU tag, a kernel branch, toggle KPM and choose whether to publish a release |

A manual run for a version that is already released **replaces** that release (useful to pick up kernel updates). A `bakasu` manual run builds the untouched Coconutat kernel (no extra features) as a control.

---

## 🚀 Using This Repository (forks)

1. Fork the repository.
2. Go to **Settings → Actions → General** and make sure Actions are enabled and **Workflow permissions** allow *Read and write* (needed to publish releases).
3. In the **Actions** tab, enable scheduled workflows on the fork.
4. Run **Build Xiaomi SDM845 Kernels with OriginSU** once manually to create the first release.

> GitHub disables scheduled workflows after **60 days without repository activity**. If automatic releases stop, re-enable the workflow in the Actions tab or push a commit.

### Manual run options

| Input | Default | Description |
|---|---|---|
| `originsu_ref` | *(latest release)* | OriginSU tag to build |
| `kernel_branch` | *(latest LineageOS with SuSFS)* | Force a branch of the Coconutat kernel tree (e.g. `lineage-22.2-ReSukiSU`) |
| `root` | `originsu` | `bakasu` builds the untouched Coconutat kernel (BakaSU) as a **control build**, with no extra features. Never published |
| `kpm` | `false` | Enable `CONFIG_KPM` (experimental on 4.9 kernels) |
| `publish` | `true` | Publish a GitHub Release (otherwise only build artifacts) |

---

## 🛠️ Troubleshooting

| Problem | What to do |
|---|---|
| Build fails in *Install the root solution* with "hook … missing" | The kernel tree changed its SuSFS hooks — check the Coconutat repo, or pin an older `kernel_branch` |
| Build fails in *Extra features* | Most likely Droidspaces on 4.9 — check `build-<device>.log` in the run artifacts; the feature can be dropped if it blocks a device |
| One device fails but others succeed | `fail-fast: false` keeps the rest building; check that device's `build-<device>.log` |
| Build fails with KPM errors | Re-run manually with `kpm` disabled |
| *check* fails with "No LineageOS branch with SuSFS found" | The Coconutat tree changed its branch layout — run manually with `kernel_branch` set |
| No build although OriginSU released | The release has no manager APK yet; the next weekly run will pick it up (or run manually later) |
| Zip refuses to install ("unsupported device") | You flashed the wrong device's zip — pick the one whose codename matches your phone, and make sure your ROM matches the LineageOS version of the release |
| Device stuck on boot logo, or reboots straight to fastboot | Restore the stock `boot.img`. The last kernel log is kept in RAM: boot into TWRP right away and get `/sys/fs/pstore/console-ramoops-0` (e.g. `adb pull /sys/fs/pstore/`), then open an issue with it |
| Boots, but no Wi-Fi / mobile network / camera | Almost always a module conflict — see [Coming from another root solution?](#️-coming-from-another-root-solution) below |
| Root gone after a ROM update | Normal: OTA updates replace `boot`. Flash the zip again |
| No new release after an OriginSU update | Check that the scheduled workflow is still enabled (60-day rule above) |

### ⚠️ Coming from another root solution?

If you previously used **Magisk, APatch, BakaSU, KernelSU, SukiSU or ReSukiSU**, your old modules are still in `/data/adb/modules` and will be loaded by OriginSU.

OriginSU ships its **own Zygisk (OriginZygisk)**. Old modules — especially another Zygisk implementation (ReZygisk, NeoZygisk, Zygisk Next) or Zygisk-based modules such as LSPosed — can conflict with it. The typical symptom is a phone that **boots fine but has no Wi-Fi, no mobile network and/or no camera**.

**Fix: uninstall all modules, then reinstall them one by one.**

From a PC (use the interactive shell — it also works in Windows PowerShell):

```sh
adb shell
su
for m in /data/adb/modules/*/; do touch "$m/remove"; done
rm -rf /data/adb/modules_update/*
reboot
```

Each module's `uninstall.sh` runs on the next boot. If some folders remain afterwards:

```sh
adb shell
su
rm -rf /data/adb/modules/*
reboot
```

Then:

- Reinstall your modules **one at a time**, checking Wi-Fi, network and camera after each reboot.
- Use **only one Zygisk**: keep OriginZygisk and don't install another Zygisk module.
- Reinstall Zygisk-based modules (e.g. LSPosed) last.
- Optional cleanup of leftovers from other root solutions, e.g. APatch: `rm -rf /data/adb/ap /data/adb/apd`.

---

## 🙏 Credits & Acknowledgments

This project relies on the incredible work of the following open-source projects and developers:

* **[Coconutat](https://github.com/Coconutat/android_kernel_xiaomi_sdm845_lineageos_Exp)** — For the Xiaomi SDM845 LineageOS kernel tree with SuSFS already integrated (`lineage-*` branches).
* **[LineageOS](https://github.com/LineageOS/android_kernel_xiaomi_sdm845)** — For the original Xiaomi SDM845 kernel source and device support.
* **[/e/OS](https://e.foundation/)** — For the de-Googled Android 15 builds.
* **[OriginSU](https://github.com/samakshkambxj/OriginSU)** — For the kernel-based root solution.
* **[riarumoda](https://github.com/riarumoda/perf_neon-builder)** — For perf_neon-builder, whose feature integration (Baseband Guard, NoMount, Droidspaces) this build reuses.
* **[osm0sis](https://github.com/osm0sis/AnyKernel3)** — For AnyKernel3.
* **[simonpunk](https://gitlab.com/simonpunk/susfs4ksu)** — For SuSFS.
* **[vc-teahouse](https://github.com/vc-teahouse/Baseband-guard)** — For Baseband Guard.
* **[maxsteeel](https://github.com/maxsteeel/nomount)** — For NoMount.
* **[ravindu644](https://github.com/ravindu644/Droidspaces-OSS)** — For Droidspaces.
* **[JackA1ltman](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd)** — For the non-GKI SuSFS work the kernel tree builds on.
* **[AOSP](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/)** — For the Clang toolchain.
* **[tiann/KernelSU](https://github.com/tiann/KernelSU)** — The project OriginSU is built upon.

## 📄 License

Kernel sources are licensed under **GPL-2.0**. See each upstream project for its own license.
