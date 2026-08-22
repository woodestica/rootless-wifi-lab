# Rootless WiFi Lab

**Rootless WiFi Lab** is a lightweight distribution for running an Alpine Linux ARM64 virtual machine on Android through Termux and QEMU, without rooting Android. It provides a practical environment for authorized Wi-Fi diagnostics, wireless security research, monitor-mode experiments, and packet-injection testing with compatible USB adapters.

The project is not an Android kernel module and it does not turn every phone or every adapter into a wireless assessment platform. QEMU provides the Linux guest, Termux provides the Android-side process and USB permission bridge, and the guest kernel must contain a compatible driver and firmware. The supported and development-tested target is the Atheros AR9271 USB adapter using the `ath9k_htc` driver and firmware.

> **Authorized-use requirement:** Use monitor mode, packet transmission, capture, and security tools only on networks and devices that you own or are explicitly authorized to assess. This project does not include automated credential theft, client disruption, WPS cracking, or unauthorized-access workflows.

> **Hardware status:** The VM, launchers, persistent storage, and local QEMU paths are tested in the development environment. Actual Android OTG permission handling, AR9271 passthrough, adapter power stability, and wireless-driver behavior must still be verified on the target phone.

## What the project does

The project packages two independent Alpine Linux ARM64 environments in one release. The Full environment is the broader and heavier option for users who want a more complete guest and more room for additional packages. The Lite environment is a compact Wi-Fi-focused option with a smaller footprint and fewer preinstalled packages. Lite is designed to reach its console in approximately **30 seconds or less** on a supported, warmed-up Android device, although the actual time depends on the phone, storage speed, QEMU load, selected profile, and USB/network setup.

Both variants use external persistent storage. A project update replaces launchers, scripts, kernels, and documentation without replacing the writable guest image. Installed Alpine packages, `/root` files, persistent PATH entries, backups, and exports therefore survive normal project updates.

| Capability | Full | Lite |
|---|---|---|
| Intended role | Broader, heavier general-purpose guest | Compact Wi-Fi-focused guest |
| Storage model | Independent persistent Full image | One persistent image shared by all Lite tiers |
| Kernel | Linux LTS kernel and initramfs path | `safe`, `tiny`, and `lts` kernel choices |
| Preinstalled scope | Larger guest lineage and broader utilities | Minimal guest with wireless diagnostics and AR9271 firmware |
| Boot target | Heavier and device-dependent | Approximately 30 seconds or less on supported devices; not a guarantee |
| Recommended use | Full Linux workspace and broader package expansion | Fast daily Wi-Fi work and adapter troubleshooting |

## Supported adapter and other Wi-Fi cards

The primary supported adapter is the **Atheros AR9271**, commonly exposed by Android as USB ID `0cf3:9271`. The Lite build includes the Alpine Linux `linux-lts` kernel path, `ath9k_htc` firmware package, wireless regulatory data, `iw`, `usbutils`, `kmod`, `ethtool`, and diagnostic helpers. The adapter must still be visible to Android, granted to Termux through Termux:API, and accepted by QEMU USB passthrough.

Other USB Wi-Fi adapters may work, but they are not automatically supported merely because Android can list them. A different adapter requires all of the following: a Linux ARM64 driver compatible with the exact guest kernel, the required firmware, a USB passthrough path that works on the target phone, and tools that understand the resulting wireless interface. The adapter's Android driver is irrelevant to the Linux guest; the guest needs its own Linux driver and firmware.

A user can install additional drivers manually inside the VM when the driver is available as an Alpine package or as a module and firmware set built for the exact guest kernel. The safe process is to start the VM with Internet access, install the required signed Alpine packages, load the module with `modprobe`, and verify it with `dmesg`, `modinfo`, `lsusb`, and `iw dev`. If the driver is not available for the guest kernel, the user must build or provide a compatible kernel/module/initramfs set; installing an arbitrary Android driver or an x86 Linux module will not work. Installing a driver also does not bypass Android USB permissions, OTG limitations, regulatory restrictions, or adapter power problems.

Example diagnostic sequence inside Lite is:

```sh
wifi-diagnose
lsusb
iw dev
modinfo ath9k_htc 2>/dev/null || true
dmesg | tail -n 80
```

The project deliberately identifies AR9271 as the supported target rather than claiming universal support for every Realtek, MediaTek, Ralink, or Broadcom adapter.

## Host requirements

The host is an Android device running a current ARM64-capable Termux installation with enough free storage for the selected release and persistent images. The host does not need Android root. A functional USB OTG connection is required for an external adapter, and the phone must expose that adapter to Termux.

The general installer requires `pkg`, `bash`, `curl`, `tar`, `coreutils`, and `sha256sum`. It installs the remaining host-side packages used by the VM, including the headless AArch64 QEMU package, Termux:API integration, `socat`, and `e2fsprogs`. USB use requires the Termux:API application and the matching `termux-api` package. The Android-side USB permission prompt is controlled by Termux:API; it cannot be simulated by the guest.

The guest needs Internet access only when downloading Alpine packages, firmware, source dependencies, or other user-selected tools. The launcher asks about Internet access during an interactive start. Offline operation is preferred when networking is not needed.

## Install for a new user

The supported new-user path is the public `install.sh` bootstrap installer. It downloads the latest full release, verifies the release SHA-256 file before extraction, creates the short project directory `~/awvm`, and runs the bundled host installer.

Run the following commands in Termux:

```sh
pkg update -y
pkg install -y bash curl tar coreutils
curl -fL https://raw.githubusercontent.com/Mohzh344/rootless-wifi-lab/main/install.sh \
  -o install.sh
bash install.sh
```

The installer refuses to overwrite an existing project directory. It keeps writable VM data outside the project so that the project can later be replaced safely. The normal locations are:

```text
Project checkout:  $HOME/awvm
Persistent data:  $XDG_DATA_HOME/awvm/
                  or $HOME/.local/share/awvm/
```

If a previous v0.3.5 or v0.3.6 installation already has persistent data under `$HOME/.local/share/android-wifi-monitor-injection-rootless`, the compatibility logic prefers that existing directory instead of creating duplicate images. You can select an explicit location before installation with `VM_STATE_ROOT`:

```sh
export VM_STATE_ROOT="$HOME/awvm-data"
bash install.sh
```

After installation, check the environment and start the launcher:

```sh
cd "$HOME/awvm"
bash bin/vmctl.sh doctor
bash bin/vmctl.sh info
bash bin/vm-launcher.sh
```

## Update an existing installation

The supported update path for an existing user is the updater bundled inside the current checkout:

```sh
cd "$HOME/awvm"
bash bin/awvm-update.sh
```

The updater reads the latest GitHub Release, downloads the smaller thin update archive, verifies its SHA-256 file, runs the new host installer while the old checkout remains available for image adoption, and replaces the project only after the installer succeeds. The thin archive intentionally contains no writable Full or Lite image, so it does not re-download or replace installed guest data.

| Command | Function |
|---|---|
| `bash bin/awvm-update.sh` | Check, confirm, download, verify, install, and replace safely |
| `bash bin/awvm-update.sh --check` | Show the latest release and make no download or file changes |
| `bash bin/awvm-update.sh --yes` | Skip the confirmation prompt |
| `bash bin/awvm-update.sh --yes --keep-old` | Keep a dated copy of the previous project checkout |

If the current project is not `~/awvm`, use its actual location:

```sh
cd "/path/to/current/awvm-project"
bash bin/awvm-update.sh
```

Do not delete the old project manually before the updater finishes. If a download, checksum, extraction, package installation, or replacement step fails, the updater keeps the existing checkout and persistent images available. If it is launched from inside the old checkout, it retains a temporary old-checkout slot so the parent shell does not remain in a deleted working directory; run `cd "$HOME/awvm"` after the update.

## Manual release installation

Most users should use `install.sh`. Manual installation is useful for offline staging or for selecting a specific release. Download the archive and its matching checksum from the [v0.3.7 Release](https://github.com/Mohzh344/rootless-wifi-lab/releases/tag/v0.3.7), verify it, extract it, and run the host installer:

```sh
sha256sum -c termux-ath9k-vm-v037-full-lite-ready.tar.gz.sha256
mkdir -p "$HOME/awvm"
tar --sparse -xzf termux-ath9k-vm-v037-full-lite-ready.tar.gz \
  -C "$HOME/awvm" --strip-components=1
cd "$HOME/awvm"
bash bin/install-termux.sh
```

Use the full archive for a first installation or when no persistent image exists. Use the thin update archive only when the existing Full/Lite image is already in persistent storage:

| Release asset | Use case | Writable images included |
|---|---|---:|
| `termux-ath9k-vm-v037-full-lite-ready.tar.gz` | New installation or first adoption | Yes |
| `termux-ath9k-vm-v037-update.tar.gz` | Existing installation update | No |
| Matching `.sha256` file | Integrity verification | Not applicable |

## Persistent storage and data preservation

The project checkout contains launchers and read-only release artifacts. The writable guest images are adopted into external persistent storage on the first Full installation. The storage library creates separate Full and Lite directories together with `backups/`, `exports/`, and an image-operation lock directory.

| Environment | Default persistent state |
|---|---|
| `XDG_DATA_HOME` is set | `$XDG_DATA_HOME/awvm/` |
| `XDG_DATA_HOME` is not set | `$HOME/.local/share/awvm/` |
| Existing legacy data is present | Existing `$HOME/.local/share/android-wifi-monitor-injection-rootless/` is preferred for compatibility |
| Explicit override | `$VM_STATE_ROOT` |

A later full archive does not overwrite an image that already exists. A thin update has no writable image at all. This design preserves installed packages, `/root`, `/home`, `/opt`, `/usr/local`, managed PATH files, and the Lite image shared by the `safe`, `tiny`, and `lts` kernel tiers.

The storage path must remain outside a checkout that will be replaced. The installer and updater reject configurations where the persistent state is inside the project directory. Always stop QEMU before editing, backing up, resizing, exporting, or importing an image.

## Launching the VM

The recommended entry point is the unified launcher:

```sh
cd "$HOME/awvm"
bash bin/vm-launcher.sh
```

In interactive mode it detects available Full and Lite bundles, asks which variant to run, offers a Lite kernel tier, asks whether Internet access should be granted, offers console authentication modes, and optionally requests USB permission. A dry run prints the selected plan without starting QEMU:

```sh
bash bin/vm-launcher.sh --full --dry-run --non-interactive
bash bin/vm-launcher.sh --lite --dry-run --non-interactive
```

Typical scripted starts are:

```sh
VM_VARIANT=full AUTH_MODE=root-console ENABLE_NET=0 \
  bash bin/vm-launcher.sh --non-interactive

VM_VARIANT=lite KERNEL_TIER=safe PROFILE=wifi-only \
  AUTH_MODE=root-console ENABLE_NET=0 \
  bash bin/vm-launcher.sh --non-interactive
```

The launcher accepts these important options:

| Option | Purpose |
|---|---|
| `--full` | Select the Full environment |
| `--lite` | Select the Lite environment |
| `--auth-mode MODE` | Select `root-console`, `login`, or `login-empty` |
| `--root-console` | Direct root console without a password prompt |
| `--login` | Normal BusyBox username/password console |
| `--login-empty` | Empty-root-password console for private local testing only |
| `--dry-run` | Print the plan without starting QEMU |
| `--non-interactive` | Disable prompts and require explicit choices when ambiguous |

The launcher synchronizes the Alpine clock from the Android/Termux clock at each start unless `TIME_SYNC=0` is explicitly set. It asks about guest Internet access on interactive starts. `ENABLE_NET=1` enables QEMU user networking and guest DHCP; `ENABLE_NET=0` keeps the guest offline.

## Console authentication modes

`root-console` is the recommended local mode. The console starts `/bin/sh -l` directly as root, reads `/etc/profile` and `/etc/profile.d`, and does not ask for a username or password. The `-l` flag means login shell; it does not invoke a password manager.

`login` starts a BusyBox `getty` username/password prompt using the guest's root password. `login-empty` deliberately clears the root password for private local testing and must never be combined with Internet access or an untrusted console. Authentication databases are intentionally excluded from portable migration archives.

A PATH exported interactively disappears when that shell ends. A PATH entry written through `vmctl.sh path add` is stored in the persistent guest at `/etc/profile.d/vmctl-path.sh` and is loaded by both the direct root login shell and a normal login shell.

## USB passthrough and AR9271 workflow

Connect the adapter through a compatible OTG connection, then check whether Android exposes it to Termux:

```sh
termux-usb -l
bash bin/vmctl.sh usb
bash bin/vmctl.sh doctor
```

Start the unified launcher interactively. It lists Android-visible USB devices and requests permission for the selected device. The direct passthrough path recognizes the AR9271 USB identity `0cf3:9271` when the adapter reports that identity.

After entering Alpine, verify the device and driver before using any wireless tool:

```sh
wifi-diagnose
lsusb
iw dev
iw reg get
```

Seeing `0cf3:9271` in `lsusb` proves only that USB enumeration reached the guest. It does not prove that `ath9k_htc` loaded, that firmware was found, that a wireless interface was created, or that monitor mode and injection are available. The diagnostic output and `dmesg` are the correct next checks.

If the adapter disconnects repeatedly, first try a short, high-quality OTG cable and a charge-through or Y-OTG arrangement with an appropriate 5 V source. A Linux guest cannot repair an Android-side power reset, USB permission reset, cable problem, or phone-specific OTG limitation. USB power behavior is device-specific and must be tested on the target phone.

## Installing Wi-Fi security tools inside the guest

The images are intentionally not filled with every security utility. This keeps Lite small and allows users to install only what they need. Start the VM with Internet access, enter the guest as root, and run:

```sh
install-wifi-tools
```

The installer first requires a guest network interface and a DHCP lease. It then uses Alpine `apk` with certificate verification enabled and installs the signed base set:

```text
aircrack-ng tcpdump hostapd wireless-regdb iw usbutils
kmod ethtool wireless-tools
```

The script records its result in `/etc/wifi-tools-install-status`. It does not start capture, deauthentication, WPS, credential, or packet-transmission actions automatically. Alpine repository availability changes over time; at this release, `wifite`, `bully`, `reaver`, `kismet`, `hcxdumptool`, and `hcxtools` are recorded as optional tools rather than guaranteed repository packages. They require separate, user-directed installation or source builds and are not silently fetched by the base installer.

An explicitly requested hcx source build is available for advanced users:

```sh
BUILD_HCX=1 install-wifi-tools
```

This downloads pinned source archives and compiler dependencies, builds the tools inside the guest, installs the resulting binaries, and removes build dependencies where possible. It requires a working guest network and considerably more time and storage than the base package set. The WPS-related path is intentionally opt-in and is not part of the default installation.

All normal Alpine package operations retain apk package signature verification. The installer may expose an explicitly reported TLS fallback for unusual guest certificate paths, but it does not disable apk package signatures.

## Administration with `vmctl.sh`

`bin/vmctl.sh` is the single administration interface. It reduces the need to remember several separate maintenance commands and protects image-writing operations with an offline check, ext4 validation, sparse-aware copies, and an image lock.

| Command | Function | Example |
|---|---|---|
| `doctor` | Check host tools, release layout, storage, images, and USB readiness | `bash bin/vmctl.sh doctor` |
| `info` | Show image paths, sizes, auth entries, PATH entries, and package metadata | `bash bin/vmctl.sh info --lite` |
| `status` | Show QEMU processes and persistent image state | `bash bin/vmctl.sh status` |
| `backup` | Create a sparse-aware safety backup of a stopped image | `bash bin/vmctl.sh backup --lite` |
| `export` | Export portable user data and an apk world list | `bash bin/vmctl.sh export --lite` |
| `import` | Import portable user data into a stopped target image | `bash bin/vmctl.sh import --full /path/to/export.tar.gz` |
| `resize` | Grow a stopped ext4 image after creating a backup | `bash bin/vmctl.sh resize --lite 3G` |
| `path add` | Persist a guest PATH directory across reboot and updates | `bash bin/vmctl.sh path add --lite /opt/tools/bin` |
| `path list` | List managed persistent PATH entries | `bash bin/vmctl.sh path list --lite` |
| `usb` | Print non-destructive Android USB and host diagnostics | `bash bin/vmctl.sh usb` |

Operations that write an image refuse to proceed while QEMU is using that image. If a Termux process is interrupted, the lock can be recovered after the owning process is no longer running. Never force concurrent `debugfs`, `resize2fs`, `e2fsck`, export, import, or backup operations against the same image.

## Backups, export, and migration

Create a safety backup before major changes:

```sh
bash bin/vmctl.sh backup --full
bash bin/vmctl.sh backup --lite
```

Backups are local sparse-aware copies under the persistent state's `backups/` directory. Export is intended for moving user data between Lite and Full or between separate installations. It does not copy the whole incompatible filesystem image:

```sh
bash bin/vmctl.sh export --lite
bash bin/vmctl.sh export --full
```

Import into a stopped target image:

```sh
bash bin/vmctl.sh import --full /path/to/lite-user-data-YYYYMMDDTHHMMSSZ.tar.gz
bash bin/vmctl.sh import --lite /path/to/full-user-data-YYYYMMDDTHHMMSSZ.tar.gz
```

The portable archive includes user-oriented paths such as `/root`, `/home`, `/opt`, `/usr/local`, `/etc/profile.d`, and an explicit `/etc/apk/world` package list. The importer writes the package list to `/root/.vm-migration/apk-world` and creates `/root/.vm-migration/apply-packages.sh`. Boot the target with Internet access when packages are needed, then run that helper as root. Packages are resolved from the target's repositories and signatures instead of being copied blindly from the old filesystem.

Authentication databases, `/etc/inittab`, repositories, kernels, firmware, and `/lib/modules` are excluded intentionally. Reconfigure console authentication with the launcher rather than copying `/etc/shadow` or `/etc/passwd` between variants. Export archives may contain private files from `/root`; store them securely.

## Full, Lite, and Lite kernel tiers

Full and Lite are separate environments. Lite's kernel tiers are not monitor-mode switches and they do not reinstall packages:

| Selection | Meaning | Persistent image |
|---|---|---|
| `Full` | Broader and heavier Alpine guest with its own image | Full image |
| `Lite / safe` | Recommended compatibility-oriented custom kernel | Shared Lite image |
| `Lite / tiny` | Smallest AR9271-focused custom kernel | Shared Lite image |
| `Lite / lts` | Fallback Linux LTS kernel and initramfs path | Shared Lite image |

Switching from `safe` to `tiny` or `lts` changes the boot kernel only. It does not remove installed packages, firmware, `/root` files, or managed PATH entries.

## Project scripts and how to run them

The public user workflow intentionally has only two top-level release entry points: `install.sh` for a new user and `bin/awvm-update.sh` for an existing user. Other files are the launcher, administration interface, compatibility wrappers, guest helper, or developer/build tooling.

| Script | Role | How to run |
|---|---|---|
| `install.sh` | Public new-user installer; installs bootstrap packages, downloads the latest full archive, verifies SHA-256, extracts it, and runs host installation | `bash install.sh` from Termux |
| `bin/awvm-update.sh` | Existing-user updater; downloads and verifies the thin archive and replaces the checkout safely | `bash bin/awvm-update.sh` from the current checkout |
| `bin/vm-launcher.sh` | Main Full/Lite dispatcher with auth, network, clock, tier, profile, and USB choices | `bash bin/vm-launcher.sh` |
| `bin/vm-launcher-unified.sh` | Compatibility wrapper for the main dispatcher | `bash bin/vm-launcher-unified.sh` |
| `bin/vm-launcher-legacy.sh` | Preserved advanced legacy launch path | Use only when a legacy layout is required |
| `bin/vmctl.sh` | Unified diagnostics, backup, export/import, resize, PATH, and USB administration | `bash bin/vmctl.sh doctor` |
| `bin/vm-backup.sh` | Compatibility wrapper for `vmctl.sh backup` | `bash bin/vm-backup.sh --lite` |
| `bin/vm-export.sh` | Compatibility wrapper for `vmctl.sh export` | `bash bin/vm-export.sh --lite` |
| `bin/vm-import.sh` | Compatibility wrapper for `vmctl.sh import` | `bash bin/vm-import.sh --lite file.tar.gz` |
| `bin/install-termux.sh` | Internal host dependency installer and persistent-image adoption step | `bash bin/install-termux.sh` after manual extraction |
| `bin/launch-vm-full-unified.sh` | Full QEMU adapter used by the dispatcher | Normally called by `vm-launcher.sh` |
| `bin/launch-vm-lite.sh` | Lite QEMU adapter used by the dispatcher | Normally called by `vm-launcher.sh` |
| `bin/qemu-lite-direct-inner.sh` | Inner Lite USB/QEMU command path | Normally called by the Lite launcher |
| `bin/usb-attach-lite-direct.sh` | Termux USB direct-passthrough helper | Normally called by the launcher |
| `src/vm-storage-lib.sh` | Shared persistent-image adoption, locks, sparse backup, and ext4 helpers | Sourced by host scripts; not normally run directly |
| `src/configure-console-auth.sh` | Changes a stopped image between auth modes with validation and backup | Developer/advanced maintenance |
| `src/patch-console-login-shell.sh` | Applies the `/bin/sh -l` console fix to a stopped image | Developer/advanced maintenance |
| `src/guest-install-wifi-tools.sh` | Guest-side package installer copied to `/usr/local/sbin/install-wifi-tools` | Inside Alpine: `install-wifi-tools` |
| `src/package-v037-unified-release.sh` | Developer packager for Full and thin v0.3.7 archives | Run only on a prepared build host |

The image builder and kernel-tier scripts under `src/` are developer operations. End users do not need to run them to install or update a published release.

## Manual rebuilding for developers

Rebuilding is optional. It requires a Linux build host with the project inputs, Alpine artifacts, AArch64 emulation, filesystem tools, and enough temporary storage. The canonical Full input, Lite build input, and kernel sources are selected through the packager's environment variables.

```sh
./src/download-artifacts.sh
./src/build-image.sh
./src/build-lite-image.sh
./src/package-v037-unified-release.sh
```

For a thin update archive, set `INCLUDE_IMAGES=0` and choose an update output path. The packager verifies ext4 images, applies the console-shell fixes, copies the current launchers and documentation, removes old release notes from the bundle, and writes archive-level and guest-level checksums. The historical Full v0.3.0 tag is not rewritten.

## Troubleshooting

When the VM does not start, begin with a dry run and the non-destructive diagnostics:

```sh
bash bin/vm-launcher.sh --lite --dry-run --non-interactive
bash bin/vmctl.sh doctor
bash bin/vmctl.sh info --lite
bash bin/vmctl.sh status
```

If Lite reports a missing image, confirm that `VM_STATE_ROOT` points to the directory containing `lite/alpine-ath9k-v030-lite.img`. If an old installation exists, check the legacy path before creating another image. If login fails, use the recommended `root-console` mode; normal `login` deliberately depends on the guest's password database and `login-empty` is for private local testing only.

If a Wi-Fi tool is not found after installation, verify that it was installed inside the guest rather than on the Termux host, check `echo "$PATH"`, and use `bash bin/vmctl.sh path list --lite` or `install-wifi-tools` as appropriate. Start Lite with `ENABLE_NET=1` when apk needs Internet access. A transient host network connection is not the same as a guest DHCP lease.

If `lsusb` sees the adapter but `iw dev` does not show an interface, inspect `wifi-diagnose`, `dmesg`, `modinfo ath9k_htc`, firmware availability, and the selected kernel tier. If Android does not list the adapter in `termux-usb -l`, troubleshoot the cable, OTG mode, Android permission, and power path before changing the guest.

## Repository layout

| Path | Purpose |
|---|---|
| `install.sh` | General new-user bootstrap installer |
| `bin/awvm-update.sh` | Bundled existing-user updater |
| `bin/vm-launcher.sh` | Recommended Full/Lite dispatcher |
| `bin/vmctl.sh` | Unified administration interface |
| `full/guest/` | Full release kernel and image bundle before adoption |
| `lite/guest/` | Lite kernels, initramfs, and image bundle before adoption |
| `src/vm-storage-lib.sh` | Persistent storage and image-safety library |
| `src/guest-install-wifi-tools.sh` | Guest Wi-Fi package installer source |
| `src/package-v037-unified-release.sh` | v0.3.7 Full/thin release packager |
| `docs/RELEASE-v0.3.7.md` | Release migration and verification notes |
| `docs/LITE-v0.3.7.md` | Lite-specific release notes |
| `docs/UNIFIED-RELEASE.md` | Full/Lite distribution contract |
| `README-UNIFIED.md` | README copy embedded into unified release archives |

## Release integrity and safety model

The public installer and updater download only the selected GitHub Release asset and its matching SHA-256 file. They verify the archive before extraction, reject absolute and traversal paths, refuse symbolic-link project targets where applicable, keep writable VM state outside the replaceable checkout, and replace an existing checkout only after the host installer succeeds.

Alpine package signatures remain enabled. The project does not automatically install or launch offensive workflows. The user remains responsible for authorization, regulatory-domain compliance, transmit-power limits, adapter firmware licensing, private export archives, and the physical safety of the Android device and USB connection.

## References

[1] [QEMU Arm `virt` machine documentation](https://www.qemu.org/docs/master/system/arm/virt.html) — virtual ARM machine model used by the launchers.

[2] [Termux:API project](https://github.com/termux/termux-api) — Android-facing APIs used for USB permission and host integration.

[3] [Alpine Package Keeper documentation](https://wiki.alpinelinux.org/wiki/Alpine_Package_Keeper) — signed package installation and repository behavior.

[4] [Linux Wireless ath9k documentation](https://wireless.docs.kernel.org/en/latest/en/users/drivers/ath9k.html) — Linux driver family background for the AR9271 target path.

[5] [Rootless WiFi Lab repository](https://github.com/Mohzh344/rootless-wifi-lab) — source, release assets, issue tracker, and project history.
