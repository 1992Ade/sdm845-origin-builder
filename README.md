# Xiaomi Mi 8 (dipper) Kernel Builder with OriginSU

[![Build](https://img.shields.io/badge/build-GitHub%20Actions-blue?logo=github-actions&logoColor=white)](https://github.com/1992Ade/dipper-origin-builder/actions/workflows/build.yml)
[![Latest release](https://img.shields.io/github/v/release/1992Ade/dipper-origin-builder?label=latest&logo=github)](https://github.com/1992Ade/dipper-origin-builder/releases/latest)

> **Your warranty is now void.** Flashing a custom kernel can brick your device. You are responsible for what you flash. Always keep a backup of your stock `boot` partition.

## 📱 About This Repository

This repository automates the compilation of a custom **Linux 4.9** kernel for the **Xiaomi Mi 8** (`dipper`, Snapdragon 845). It is based on the LineageOS-derived kernel tree by **[Coconutat](https://github.com/Coconutat/android_kernel_xiaomi_sdm845_lineageos_Exp)**, which already ships **SuSFS** and its inline hooks (originally used with BakaSU).

Every build integrates the **[OriginSU](https://github.com/samakshkambxj/OriginSU)** kernel-based root solution in place of BakaSU, and is packaged as a recovery-flashable `.zip` (TWRP / OrangeFox) with AnyKernel3.

**Fully automated:** a GitHub Actions workflow checks every week (Sunday) for a new OriginSU release. As soon as a new version is out **with its manager APK**, the kernel is rebuilt with that exact OriginSU version and a new GitHub Release is published automatically — no manual action needed.

**Always the latest LineageOS:** each run picks the most recent LineageOS version available in the Coconutat tree (today `lineage-22.2-BakaSU`, LineageOS 22.2 / Android 15). When a newer LineageOS branch with SuSFS appears (e.g. LineageOS 23 / Android 16), the next weekly run switches to it on its own and rebuilds the current release.

---

## ✅ Supported Device & ROMs

| Device | Codename |
|---|---|
| Xiaomi Mi 8 (global / standard) | `dipper` |

> The Mi 8 Explorer/Pro (`equuleus`) and Mi 8 EE (`ursa`) are **not** targeted: the zip refuses to install on any device other than `dipper`.

The kernel comes from the LineageOS kernel tree, so it works with ROMs built on the LineageOS kernel for `dipper`. Each release states the LineageOS version it was built for (also in the zip name, e.g. `LOS22.2`) — **flash the release that matches the LineageOS / /e/OS version on your phone**. Currently:

| ROM | Status | Links |
|---|---|---|
| **LineageOS 22.2** (Android 15) | ✅ Supported — the kernel is built from the LineageOS source | [LineageOS wiki — dipper](https://wiki.lineageos.org/devices/dipper/) |
| **/e/OS** (Android 15, community) | ✅ Supported — /e/OS for `dipper` is built on LineageOS 22 | [/e/OS docs — dipper](https://doc.e.foundation/devices/dipper/) |

Other ROMs may work if they use the LineageOS `sdm845` 4.9 kernel and the same vendor firmware, but they are not tested. Stock MIUI is **not** supported.

> Make sure your device is on the vendor firmware LineageOS asks for (see the *"Update to a newer vendor firmware version"* section of the LineageOS wiki).

---

## 📦 Which zip should I download?

Each release ships one zip:

| File | Hook mode | Notes |
|---|---|---|
| `Mi8-dipper-LOS<lineage>-OriginSU-<version>-susfs-<date>.zip` | SuSFS inline hook | Root + SuSFS root hiding |

Only the SuSFS variant is built: the Coconutat tree has its KernelSU hooks written for SuSFS, and OriginSU's manual-hook mode would need a different set of hooks.

The full kernel `.config` is attached to each release for reference.

---

## 🔧 Installation

0. **Coming from Magisk, APatch, BakaSU or another KernelSU?** Read [this warning](#️-coming-from-another-root-solution) first.
1. Back up your stock `boot` partition (from TWRP/OrangeFox, or keep the `boot.img` from your LineageOS / /e/OS zip).
2. Download the zip from the [latest release](https://github.com/1992Ade/dipper-origin-builder/releases/latest).
3. Reboot to recovery (**Volume Up + Power** until the "Mi 8" logo appears) and flash it, or use `adb sideload <file>.zip`.
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
check  ──►  build (susfs)  ──►  release
```

1. **check**
   - Reads the latest stable OriginSU release and makes sure it ships the **manager APK** (otherwise nothing is built, and the next weekly run tries again).
   - Picks the kernel branch: among the `lineage-<version>[-suffix]` branches of the Coconutat tree, the highest LineageOS version first, and for the same version the most recently updated branch. A branch is kept only if it contains SuSFS, its inline hooks and the `dipper` configs.
   - Builds if there is no `originsu-<tag>` release yet, or if the existing one was built on an older LineageOS branch. Otherwise the run stops there.
2. **build**
   - Clones the selected Coconutat SDM845 kernel branch.
   - Downloads **AOSP Clang r547379**, the toolchain recommended by the kernel's author.
   - Integrates OriginSU at the exact release tag with its official `setup.sh`, then checks that every SuSFS inline hook OriginSU needs is in the tree.
   - Generates the config from `vendor/xiaomi/mi845_defconfig` + `extra.config` + `dipper.config`, with `CONFIG_KSU_SUSFS` and the SuSFS features enabled.
   - Compiles `Image.gz-dtb` and packages it with AnyKernel3 (only the kernel is replaced, the ROM's ramdisk is left alone; the zip only installs on `dipper`).
3. **release** — publishes a GitHub Release `originsu-<tag>` with the zip, the config and release notes (OriginSU version, LineageOS version, kernel branch and commit, kernel version, SuSFS version, build date).

### Schedule

| Trigger | Behaviour |
|---|---|
| Every Sunday at 05:37 UTC | Builds **only** if a new OriginSU release (with its manager) is out, or a newer LineageOS branch appeared |
| Manual (`Run workflow`) | Always builds; you can pick an OriginSU tag, a kernel branch, toggle KPM and choose whether to publish a release |

A manual run for a version that is already released **replaces** that release (useful to pick up kernel updates).

---

## 🚀 Using This Repository (forks)

1. Fork the repository.
2. Go to **Settings → Actions → General** and make sure Actions are enabled and **Workflow permissions** allow *Read and write* (needed to publish releases).
3. In the **Actions** tab, enable scheduled workflows on the fork.
4. Run **Build Xiaomi Mi 8 (dipper) Kernel with OriginSU** once manually to create the first release.

> GitHub disables scheduled workflows after **60 days without repository activity**. If automatic releases stop, re-enable the workflow in the Actions tab or push a commit.

### Manual run options

| Input | Default | Description |
|---|---|---|
| `originsu_ref` | *(latest release)* | OriginSU tag to build |
| `kernel_branch` | *(latest LineageOS with SuSFS)* | Force a branch of the Coconutat kernel tree (e.g. `lineage-22.2-ReSukiSU`) |
| `kpm` | `false` | Enable `CONFIG_KPM` (experimental on 4.9 kernels) |
| `publish` | `true` | Publish a GitHub Release (otherwise only build artifacts) |

---

## 🛠️ Troubleshooting

| Problem | What to do |
|---|---|
| Build fails in *Integrate OriginSU* with "hook … missing" | The kernel tree changed its SuSFS hooks — check the Coconutat repo, or pin an older `kernel_branch` |
| Build fails with "You lost … hook" / "incompatible hook" | OriginSU expects a newer SuSFS than the tree has — check `build.log` in the run artifacts |
| Build fails with KPM errors | Re-run manually with `kpm` disabled |
| *check* fails with "No LineageOS branch with SuSFS found" | The Coconutat tree changed its branch layout — run manually with `kernel_branch` set |
| No build although OriginSU released | The release has no manager APK yet; the next weekly run will pick it up (or run manually later) |
| Build fails on a new LineageOS branch | Run manually with `kernel_branch` set to the previous branch until the new one is fixed upstream |
| Zip refuses to install ("unsupported device") | The zip is for `dipper` only (no Android version check: make sure yourself that your ROM matches the LineageOS version of the release) |
| Device stuck on boot logo | Restore the stock `boot.img`, and check that your ROM matches the LineageOS version of the release, with the right firmware |
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
* **[LineageOS](https://github.com/LineageOS/android_kernel_xiaomi_sdm845)** — For the original Xiaomi SDM845 kernel source and the `dipper` device support.
* **[/e/OS](https://e.foundation/)** — For the de-Googled Android 15 build for `dipper`.
* **[OriginSU](https://github.com/samakshkambxj/OriginSU)** — For the kernel-based root solution.
* **[osm0sis](https://github.com/osm0sis/AnyKernel3)** — For AnyKernel3.
* **[simonpunk](https://gitlab.com/simonpunk/susfs4ksu)** — For SuSFS.
* **[JackA1ltman](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd)** — For the non-GKI SuSFS work the kernel tree builds on.
* **[AOSP](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/)** — For the Clang toolchain.
* **[tiann/KernelSU](https://github.com/tiann/KernelSU)** — The project OriginSU is built upon.

## 📄 License

Kernel sources are licensed under **GPL-2.0**. See each upstream project for its own license.
