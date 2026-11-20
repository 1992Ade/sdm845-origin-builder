# Xiaomi SDM845 Kernel Builder with OriginSU

[![Build](https://img.shields.io/badge/build-GitHub%20Actions-blue?logo=github-actions&logoColor=white)](https://github.com/1992Ade/sdm845-origin-builder/actions/workflows/build.yml)
[![Latest release](https://img.shields.io/github/v/release/1992Ade/sdm845-origin-builder?label=latest&logo=github)](https://github.com/1992Ade/sdm845-origin-builder/releases/latest)

> [!IMPORTANT]
> **This is the recommended kernel for Xiaomi SDM845 — OriginSU `v4.3.0`, the final OriginSU release.**
> OriginSU has been renamed and rebuilt as **[KazeSU](https://github.com/KazeSU/KazeSU)**, which targets modern (GKI 5.10+) kernels and **does not build on this Linux 4.9 kernel** (its SELinux code needs the 5.10+ layout). OriginSU `v4.3.0` still supports it fully (non-GKI hooks, SuSFS, built-in module mounting), so this builder stays on it: the root solution is frozen, but the builder is still maintained by hand for kernel updates and optimizations.
> **4.9 is also the newest kernel LineageOS ships for SDM845.** Mainline Linux runs on some of these phones (POCO F1, Mi MIX 2S), but only for Linux distributions or bare AOSP test builds, not for LineageOS-based ROMs.
>
> KazeSU builds for SDM845 ([sdm845-KazeSU-builder](https://github.com/1992Ade/sdm845-KazeSU-builder)) are **on hold** until KazeSU supports older kernels again. The OriginSU `v4.3.0` manager APK is attached to every release (also on the KazeSU repository: [tag `v4.3.0`, release "legacy-v4.3.0"](https://github.com/KazeSU/KazeSU/releases/tag/v4.3.0)). Note that KazeSU `v1.0.0` works with this kernel too but new root hiding method is not implemented.

> **Your warranty is now void.** Flashing a custom kernel can brick your device. You are responsible for what you flash. Always keep a backup of your stock `boot` partition.

## 📱 About This Repository

This repository automates the compilation of a custom **Linux 4.9** kernel for **Xiaomi Snapdragon 845 (SDM845)** devices — Mi 8, POCO F1, Mi MIX 2S and their siblings. It is based on the LineageOS-derived kernel tree by **[Coconutat](https://github.com/Coconutat/android_kernel_xiaomi_sdm845_lineageos_Exp)**, which already ships **SuSFS** and its inline hooks (originally used with BakaSU).

Every build integrates the **[OriginSU v4.3.0](https://github.com/KazeSU/KazeSU/releases/tag/v4.3.0)** kernel-based root solution in place of BakaSU — together with **SuSFS**, **Baseband Guard**, **NoMount**, **Droidspaces**, **TCP BBR** and a few [optimizations](#-optimizations) (see [Included features](#-included-features)) — and produces **one recovery-flashable `.zip` per device** (TWRP / OrangeFox) with AnyKernel3. The matching **OriginSU manager APK** is attached to each release.

> [!WARNING]
> **LineageOS-based ROMs only.** This is a LineageOS kernel. Kernels for LineageOS (and ROMs built on it, such as /e/OS) and kernels for other ROM families such as crDroid are **not interchangeable** on SDM845: one does not boot on the other. Check that your ROM uses the LineageOS `sdm845` kernel before flashing.

The XDA thread text (BBCode, ready to paste) is kept in [`docs/XDA_POST.txt`](docs/XDA_POST.txt).

**Manual builds only:** OriginSU will not get new releases, so the weekly check was removed. Running the workflow by hand rebuilds OriginSU `v4.3.0` for every device on the latest kernel, with the current optimizations, and replaces the release.

**Always the latest LineageOS:** each run picks the most recent LineageOS version available in the Coconutat tree (today `lineage-22.2-BakaSU`, LineageOS 22.2 / Android 15). When a newer LineageOS branch with SuSFS appears (e.g. LineageOS 23 / Android 16), the next manual run switches to it on its own.

---

## 🧩 Included Features

On top of OriginSU, every build enables the same hardening/usability modules as the base [perf_neon-builder](https://github.com/riarumoda/perf_neon-builder):

| Feature | Upstream | What it does | Kernel config |
|---|---|---|---|
| **SuSFS v2.4.0** | [simonpunk](https://gitlab.com/simonpunk/susfs4ksu), 4.9 port by [JackA1ltman](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd) | Hides root/mounts from detection. The Coconutat tree ships v2.3.0; the build updates it to v2.4.0 (see [SuSFS v2.4.0](#-susfs-v240)) | `CONFIG_KSU_SUSFS_*` |
| **Baseband Guard** | [vc-teahouse](https://github.com/vc-teahouse/Baseband-guard) | LSM that blocks writes to the modem/baseband partitions | `CONFIG_BBG=y` |
| **NoMount** | [maxsteeel](https://github.com/maxsteeel/nomount) | Mounts modules without real mounts (VFS path redirection, nothing in `/proc/mounts`) — driven by OriginSU's *OriginMount* screen, see [Modules](#-modules) | `CONFIG_NOMOUNT=y` |
| **Droidspaces** | [ravindu644](https://github.com/ravindu644/Droidspaces-OSS) | Namespace/cgroup isolation to run isolated Android/Linux spaces | `CONFIG_NAMESPACES`, `CONFIG_*_NS`, cgroups, netfilter… |
| **TCP BBR + fq** | [Google](https://github.com/google/bbr) (mainline since 4.9) | BBR congestion control **set as default** (the stock Coconutat kernel uses CUBIC), with the **`fq` qdisc as default** so BBR can pace its traffic | `CONFIG_DEFAULT_BBR=y`, `CONFIG_NET_SCH_FQ=y`, `CONFIG_DEFAULT_FQ=y` |
| **Optimizations** | this repo | ZRAM in lz4 by default, power-efficient workqueues, TTL/HL targets, `vm.swappiness` and per-cgroup swappiness up to 200, Cortex-A55 tuning (see [Optimizations](#-optimizations)) | `CONFIG_WQ_POWER_EFFICIENT_DEFAULT=y`… |

> **Droidspaces note:** namespaces, overlayfs, networking and the pids, freezer, devices and net cgroups are enabled, and **per-container memory/swap limits work** (`CONFIG_MEMCG` is on in this kernel). The **CPU limit (`--cpus`) is not available**: it needs `CONFIG_CFS_BANDWIDTH`, which this tree only allows when Qualcomm's **WALT** scheduler is off — and WALT drives CPU frequency and task placement on SDM845, so turning it off would cost performance and battery. The build asks for it, Kconfig drops it, and Droidspaces simply greys out `--cpus`. Droidspaces only ships its two "fix" patches for newer trees (the cgroup one targets `kernel/cgroup/cgroup.c`, which exists only since 4.15). On this **4.9** tree the same function and symbols live in `kernel/cgroup.c`, so the build first tries each patch as-is and, if it doesn't apply, retries it with the path rewritten to the 4.9 location — verified to apply cleanly on the Coconutat 4.9 tree. If a future kernel ever changes enough that a patch no longer fits, it is skipped with a `::warning` and the build still completes with the configs enabled. If you use Droidspaces with a systemd-based Linux space, mask the `NetworkManager` and `wpa_supplicant` services inside that space (as noted upstream).

> **TCP BBR note:** on 4.9, BBR has no TCP-internal pacing (it only arrived in 4.13): the kernel's own `tcp_bbr.c` states that BBR **must** run on the `fq` qdisc, otherwise it does not work properly. This 4.9 tree had no config option to make `fq` the default qdisc, so the build **backports the small upstream change** that adds it (`CONFIG_NET_SCH_DEFAULT` + a boot-time call to the existing `qdisc_set_default()`, identical to mainline) and selects `fq`. No `service.d` script is needed any more. The build fails if the final `.config` does not have `CONFIG_DEFAULT_TCP_CONG="bbr"` and `CONFIG_DEFAULT_NET_SCH="fq"`.
>
> Check on the phone (root shell):
> ```sh
> cat /proc/sys/net/ipv4/tcp_congestion_control   # bbr
> cat /proc/sys/net/core/default_qdisc            # fq
> tc qdisc show dev wlan0                          # fq on Wi-Fi (if tc is available)
> ```

### ⚡ Optimizations

The Coconutat kernel already has WireGuard, PSI and memory cgroups. This builder adds what is missing (low risk; the build fails if a config is not in the final `.config`):

| Change | Effect | How to use / check |
|---|---|---|
| `CONFIG_WQ_POWER_EFFICIENT_DEFAULT=y` | Background kernel work runs on CPUs that are already awake instead of waking an idle core | `cat /sys/module/workqueue/parameters/power_efficient` → `Y` |
| **ZRAM in lz4 by default** (was `lzo`) + `CONFIG_CRYPTO_ZSTD=y` | The ROM does not choose a ZRAM algorithm, so the kernel default applies; it was hard-coded to the old `lzo`. **lz4** is much faster, above all to bring apps back from ZRAM, for a slightly lower compression ratio that does not matter with 6–8 GB of RAM. zstd (better compression, more CPU) stays available | `cat /sys/block/zram0/comp_algorithm` → `[lz4]` |
| `CONFIG_IP_NF_TARGET_TTL=y`, `CONFIG_IP6_NF_TARGET_HL=y` | iptables `TTL` / `HL` targets, e.g. to hide tethering from the carrier | `iptables -t mangle -A POSTROUTING -j TTL --ttl-set 64` |
| `vm.swappiness` max 100 → 200 | Lets you make the kernel prefer swapping to ZRAM over dropping app caches. The ROM's value (100) is unchanged | `echo 160 > /proc/sys/vm/swappiness` |
| Per-cgroup swappiness max 100 → 200 | LineageOS' `init.qcom.rc` creates a `bg` memory cgroup and sets its swappiness to **140**, so apps placed there swap to ZRAM sooner than foreground apps. The 4.9 kernel rejected any value above 100, so that ROM setting silently never applied; now it does. If your build does not create that group or leaves it empty, this changes nothing (harmless either way) | `cat /sys/fs/cgroup/memory/bg/memory.swappiness` → `140` |
| `-mcpu=cortex-a55` | Code tuned for the cores of the Snapdragon 845 (Cortex-A75 + Cortex-A55, ARMv8.2) | — |

Not included on purpose: LTO (experimental on this 4.9 tree, and it can rename internal symbols that root/hiding code looks up at boot), MGLRU (too large to backport to 4.9) and a newer LZ4 (its new decompression loop was measured slower on in-order Cortex-A53 cores and with Clang).

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

Other ROMs may work only if they are built on the LineageOS `sdm845` 4.9 kernel and the same vendor firmware, and they are not tested. ROMs built on another kernel tree, such as **crDroid**, are **not** compatible: LineageOS and crDroid kernels do not boot on each other's ROMs on SDM845. Stock MIUI is **not** supported.

> Make sure your device is on the vendor firmware LineageOS asks for (see the *"Update to a newer vendor firmware version"* section of the LineageOS wiki).

---

## 📦 Which zip should I download?

Each release ships **one zip per device**, named:

```
sdm845-<codename>-LOS<lineage>-OriginSU-<version>-susfs-<date>.zip
```

For example `sdm845-dipper-LOS22.2-OriginSU-v4.3.0-susfs-20261115.zip` for the Mi 8. Only the **SuSFS** variant is built: the Coconutat tree has its KernelSU hooks written for SuSFS. The full kernel `.config` of each device is attached to the release for reference.

---

## 🔧 Installation

0. **Coming from Magisk, APatch, BakaSU or another KernelSU?** Read [this warning](#️-coming-from-another-root-solution) first.
1. Back up your stock `boot` partition (from TWRP/OrangeFox, or keep the `boot.img` from your LineageOS / /e/OS zip).
2. Download **the zip matching your device codename** from the [latest release](https://github.com/1992Ade/sdm845-origin-builder/releases/latest).
3. Reboot to recovery and flash it, or use `adb sideload <file>.zip`.
4. Reboot to system.
5. Install the **OriginSU manager** APK attached to the same release (`OriginSU_v4.3.0_…-arm64-v8a-release.apk`, also on the [legacy-v4.3.0 page](https://github.com/KazeSU/KazeSU/releases/tag/v4.3.0)). It must match the kernel: do **not** install the KazeSU v1.x manager with this kernel.
6. **Pick a module mount mode** in the manager's **OriginMount** screen — otherwise modules that change `/system` are not mounted (see [Modules](#-modules)).

> **Updating the ROM?** A LineageOS / /e/OS OTA update replaces the `boot` partition, so the OriginSU kernel is removed. Flash the zip again after each ROM update.

### 🧱 Modules

OriginSU mounts modules itself. Choose how in the manager's **OriginMount** screen (the choice applies from the next boot):

| Mode | What it does |
|---|---|
| `auto` (default) | Uses a metamodule if one is installed, otherwise Magic Mount if you enabled it, otherwise **no mounting** |
| `nomount` | Uses the **NoMount** driver built into this kernel: module files are injected without real mounts, so nothing shows in `/proc/mounts` |
| `builtin` | OriginSU's built-in Magic Mount |
| `metamodule` | Delegates to an installed metamodule (e.g. meta-overlayfs) |
| `off` | No module mounting (scripts, `sepolicy.rule` and `system.prop` still work) |

NoMount is already in the kernel and OriginSU talks to it directly: **do not install the separate NoMount module zip** on top. If your modules are installed but their files don't show up, you are probably in `auto` mode with nothing to mount them: pick a mode.

### 🔍 Checking the kernel after flashing

Root shell (`adb shell` then `su`, or a terminal app):

```sh
uname -r                                              # 4.9.x
cat /proc/sys/net/ipv4/tcp_congestion_control         # bbr
cat /proc/sys/net/core/default_qdisc                  # fq
cat /sys/block/zram0/comp_algorithm                   # [lz4] lzo ... zstd
cat /sys/fs/cgroup/memory/bg/memory.swappiness        # 140 if the ROM created the group (100 = rejected)
wc -l < /sys/fs/cgroup/memory/bg/tasks                # > 0 = background apps are in that group
cat /proc/net/ip_tables_targets | grep TTL            # TTL
cat /proc/net/ip6_tables_targets | grep HL            # HL
cat /sys/module/workqueue/parameters/power_efficient  # Y
dmesg | grep -iE "nomount|baseband|susfs" | head      # NoMount, Baseband Guard, SuSFS lines
ls /proc/self/ns                                      # user net pid uts ipc ... (Droidspaces)
```

In the manager: OriginSU **v4.3.0**, SuSFS **v2.4.0**, and NoMount shown as available in **OriginMount**. Then check Wi-Fi, mobile data and calls, camera, Bluetooth, fingerprint, audio, charging and overnight battery drain. If something is wrong, save the logs right after reproducing it: `dmesg > /sdcard/dmesg.txt` and `logcat -b all -d > /sdcard/logcat.txt`.

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
   - Selects the OriginSU tag (`v4.3.0` by default; only `v4.x` OriginSU tags are accepted) and makes sure its release ships the **manager APK**.
   - Picks the kernel branch: among the `lineage-<version>[-suffix]` branches of the Coconutat tree, the highest LineageOS version first, and for the same version the most recently updated branch. A branch is kept only if it contains SuSFS, its inline hooks and the shared SDM845 configs.
2. **build** (matrix: `beryllium`, `dipper`, `equuleus`, `polaris`, `perseus`, `ursa` — in parallel, `fail-fast: false`)
   - Clones the selected Coconutat SDM845 kernel branch.
   - Downloads **AOSP Clang r547379**, the toolchain recommended by the kernel's author.
   - Integrates OriginSU at the exact tag with its own `setup.sh` from that tag, cloned from the KazeSU repository (the build stops if the tag cannot be checked out) (plus small 4.9 compatibility fixes applied to OriginSU's sources only), then checks that every SuSFS inline hook OriginSU needs is in the tree.
   - Generates the config from `vendor/xiaomi/mi845_defconfig` + `extra.config` + `<device>.config`, then enables the **extra features** (Baseband Guard, NoMount, Droidspaces, TCP BBR + `fq` as default — see [above](#-included-features)) and the [optimizations](#-optimizations), all checked in the final `.config`; the build stops if the NoMount driver speaks an API older than the one OriginSU requires (22).
   - Packages like Coconutat: osm0sis AnyKernel3 + the tree's own `tools/AnyKernel3_sh/anykernel.sh`, with both `Image.gz` and `Image.gz-dtb`. AnyKernel3 flashes `Image.gz` and **keeps the DTB of your ROM's `boot.img`**. Each zip installs on its own device only.
3. **release** — downloads the matching OriginSU manager APK (arm64-v8a) and publishes a GitHub Release `originsu-<tag>` with every device zip, the configs, the manager and release notes (OriginSU version, LineageOS version, kernel branch and commit, kernel version, SuSFS version, the per-device zip table and the build date).

### Schedule

| Trigger | Behaviour |
|---|---|
| Manual (`Run workflow`) | Builds OriginSU `v4.3.0` (or another `v4.x` tag); you can pick a kernel branch, toggle KPM and choose whether to publish a release |

There is no scheduled build any more (OriginSU is discontinued).

A manual run for a version that is already released **replaces** that release (useful to pick up kernel updates). A `bakasu` manual run builds the untouched Coconutat kernel (no extra features) as a control.

---

## 🚀 Using This Repository (forks)

1. Fork the repository.
2. Go to **Settings → Actions → General** and make sure Actions are enabled and **Workflow permissions** allow *Read and write* (needed to publish releases).
3. Run **Build Xiaomi SDM845 Kernels with OriginSU** once manually to create the first release.

### Manual run options

| Input | Default | Description |
|---|---|---|
| `originsu_ref` | `v4.3.0` | OriginSU tag to build (`v4.x` only) |
| `kernel_branch` | *(latest LineageOS with SuSFS)* | Force a branch of the Coconutat kernel tree (e.g. `lineage-22.2-ReSukiSU`) |
| `root` | `originsu` | `bakasu` builds the untouched Coconutat kernel (BakaSU) as a **control build**, with no extra features. Never published |
| `susfs` | `v2.4.0` | `v2.4.0` updates SuSFS to JackA1ltman's 4.9 port (pinned commit); `tree` keeps the tree's v2.3.0 |
| `kpm` | `false` | Enable `CONFIG_KPM` (experimental on 4.9 kernels) |
| `publish` | `true` | Publish a GitHub Release (otherwise only build artifacts) |

---

## 🫥 SuSFS v2.4.0

The Coconutat tree ships SuSFS **v2.3.0** (JackA1ltman's 4.9 port). The build moves it to **v2.4.0**, the same port at a pinned commit: between the two, only `fs/susfs.c`, the two SuSFS headers and the SuSFS hooks in `fs/namei.c`, `fs/readdir.c` and `fs/Makefile` change; every other hook is identical. The build stops if the tree is not exactly the expected v2.3.0, and `susfs` = `tree` keeps v2.3.0.

What changes for you, with the **susfs4ksu** module (sidex15):

| | v2.3.0 | v2.4.0 |
|---|---|---|
| Who cannot see the *sus paths* | Apps root is unmounted for | **Every app** (uid 10000–19999), except the ones you allow |
| Allow an app to see them | — | **sus_path uid filter** (new command `CMD_SUSFS_SET_SUS_PATH_UID_FILTER`): a list of package names |
| After an app is reinstalled | — | Its new uid is picked up automatically (when `packages.list` is rewritten) |

OriginSU v4.3.0 predates v2.4.0, so the build wires that one new command into OriginSU's SuSFS dispatcher and refreshes the allowed apps' uids on package changes. Root processes (`su`) always see everything.

---

## 🛠️ Troubleshooting

| Problem | What to do |
|---|---|
| Build fails with "the tree's SuSFS is not the expected base port" | Coconutat updated SuSFS in the tree: set `SUSFS_BASE_REF` in the workflow to the matching JackA1ltman commit, or run with `susfs` = `tree` |
| SuSFS v2.4.0 problem (an app no longer sees a file it needs, detection) | Run again with `susfs` = `tree` (v2.3.0) and compare, then report it |
| Build fails in *Install the root solution* with "hook … missing" | The kernel tree changed its SuSFS hooks — check the Coconutat repo, or pin an older `kernel_branch` |
| `::warning` about a Droidspaces patch being skipped | Expected on 4.9 — the upstream fix doesn't fit this tree; the build continues with the Droidspaces configs enabled (see the Droidspaces note above) |
| Droidspaces greys out `--cpus` | Expected: the CPU limit needs `CONFIG_CFS_BANDWIDTH`, which Qualcomm's WALT scheduler excludes on this kernel. Memory/swap limits work |
| Build fails with `… missing from the final config` | A config this builder relies on was dropped by Kconfig — check the *Optimizations* lines of the *Extra features* step |
| Modules installed but their files don't show up | Pick a mode in **OriginMount** (`auto` mounts nothing without a metamodule or Magic Mount) — see [Modules](#-modules) |
| Bootloop after flashing on crDroid or another non-LineageOS ROM | Expected: this LineageOS kernel does not boot on crDroid-family ROMs. Restore your `boot` backup |
| Build fails in *Extra features* (Baseband Guard / NoMount setup) | Check `build-<device>.log` in the run artifacts; the setup script of one of the modules failed to download or patch the tree |
| One device fails but others succeed | `fail-fast: false` keeps the rest building; check that device's `build-<device>.log` |
| Build fails with KPM errors | Re-run manually with `kpm` disabled |
| *check* fails with "No LineageOS branch with SuSFS found" | The Coconutat tree changed its branch layout — run manually with `kernel_branch` set |
| Zip refuses to install ("unsupported device") | You flashed the wrong device's zip — pick the one whose codename matches your phone, and make sure your ROM matches the LineageOS version of the release |
| Device stuck on boot logo, or reboots straight to fastboot | Restore the stock `boot.img`. The last kernel log is kept in RAM: boot into TWRP right away and get `/sys/fs/pstore/console-ramoops-0` (e.g. `adb pull /sys/fs/pstore/`), then open an issue with it |
| Boots, but no Wi-Fi / mobile network / camera | Almost always a module conflict — see [Coming from another root solution?](#️-coming-from-another-root-solution) below |
| Root gone after a ROM update | Normal: OTA updates replace `boot`. Flash the zip again |
| Build fails with "fq is not the default qdisc" | The `NET_SCH_DEFAULT` backport no longer applies to the kernel tree — check the *TCP BBR + fq* lines of the *Extra features* step |

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
* **[OriginSU](https://github.com/KazeSU/KazeSU/releases/tag/v4.3.0)** (now [KazeSU](https://github.com/KazeSU/KazeSU)) — For the kernel-based root solution.
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
