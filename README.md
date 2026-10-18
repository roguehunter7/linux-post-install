# CachyOS Post-Install: ASUS VivoBook X409DA

The CachyOS installer already handled KDE Plasma, systemd-boot, ext4, the x86-64-v3 repositories, and zram. This guide starts after first boot and covers only post-install changes. Package installs use Shelly, which prompts for authentication when needed.

Hardware: Ryzen 5 3500U, Vega 8, 5.7 GiB usable RAM, 235 GB NVMe, ASUS VivoBook X409DA.
Assumptions: Secure Boot is off, the laptop stays plugged in, and there is no gaming or low-latency audio workload.
Last verified: 2026-10-18.

## 1. Update

Use Cachy-Update, the CachyOS update app, for normal package updates. Open it from the app menu or run:

```bash
cachy-update
```

It runs the full upgrade, shows Arch news, and offers post-update cleanup. For a terminal upgrade, use `shelly upgrade standard`.

Reboot if the update includes a kernel, Plasma, or GPU stack change.

## 2. Kernel fallback

The default `linux-cachyos` kernel is already installed. Open CachyOS Kernel Manager (`cachyos-kernel-manager`), select `linux-cachyos-lts`, and apply. It installs the kernel and regenerates the systemd-boot entries.

For a terminal install, use Shelly:

```bash
shelly install standard linux-cachyos-lts
```

If the entries are not updated, run:

```bash
sudo sdboot-manage gen
```

If `sdboot-manage` is missing, install its package first:

```bash
shelly install standard systemd-boot-manager
```

Keep the default kernel as the normal boot target. The LTS build covers a bad kernel update. RT, BORE, BMQ, and RC builds target other workloads.

## 3. Security

UFW is enabled by default. Check it, then remove the KDE Connect firewall rules if KDE Connect is unused:

```bash
sudo ufw status verbose
systemctl is-enabled sshd
```

Check whether KDE Connect is installed:

```bash
pacman -Q kdeconnect
```

If it is installed, remove it with Shelly. Shelly cascades unneeded dependencies by default:

```bash
shelly remove standard kdeconnect
```

If those UFW rules exist, remove them:

```bash
sudo ufw delete allow 1714:1764/udp
sudo ufw delete allow 1714:1764/tcp
sudo ufw reload
```

If direct deletion fails, list them with `sudo ufw status numbered` and delete the two KDE Connect entries by number, highest first.

The SSH server should stay disabled. If you enable it later, use key authentication and limit the port:

```bash
sudo systemctl enable --now sshd
sudo ufw limit ssh
```

Shelly is the default package manager; use it for AUR packages.

## 4. Power and battery

The active power profile should be `balanced`:

```bash
powerprofilesctl get
```

Use power-profiles-daemon; TLP is not part of this setup. The 3500U is Zen+ without CPPC, so it uses `acpi-cpufreq` with `schedutil`; `amd_pstate` does not apply.

Check the battery thresholds:

```bash
upower -i /org/freedesktop/UPower/devices/battery_BAT0 | grep -Ei 'threshold|capacity|state'
```

Set an 80% charge limit in KDE System Settings under Power Management. If the UI does not expose it, set it directly:

```bash
echo 80 | sudo tee /sys/class/power_supply/BAT0/charge_control_end_threshold
```

To make the sysfs method persistent:

```bash
printf 'w /sys/class/power_supply/BAT0/charge_control_end_threshold - - - - 80\n' | sudo tee /etc/tmpfiles.d/battery-charge-limit.conf
sudo systemd-tmpfiles --create /etc/tmpfiles.d/battery-charge-limit.conf
```

Use either the KDE/UPower control or the tmpfiles file, not both. This battery is at about 65% health after 335 cycles; 80% is a reasonable limit for a laptop that stays plugged in. The 3500U is 4C/8T, so `game-performance` is not part of this setup.

## 5. Filesystem and backups

The installer already mounts ext4 with `noatime`. Confirm it after first boot:

```bash
findmnt -no FSTYPE,OPTIONS /
systemctl is-enabled fstrim.timer
sudo systemctl enable --now fstrim.timer
```

Expected: `ext4` with `noatime` in the options, and `fstrim.timer` enabled. No `/etc/fstab` edit is needed.

This setup has no snapshots. Use restic or borg to back up to an external disk:

```bash
shelly install standard restic
```

Initialize the repository on the external disk and follow the restic documentation. The package cache and `shelly downgrade` handle package rollback, but they are not a substitute for a backup.

## 6. Maintenance

Clean the package cache and keep the last three versions:

```bash
shelly install standard pacman-contrib
sudo paccache -r -k3
sudo systemctl enable --now paccache.timer
```

Shelly can perform the same cleanup on demand:

```bash
shelly purify standard -c 3
```

Use one method. Enable SMART monitoring:

```bash
shelly install standard smartmontools
sudo systemctl enable --now smartd.service
sudo smartctl -a /dev/nvme0n1
```

The network wait service is optional on a laptop with no network mounts:

```bash
sudo systemctl disable NetworkManager-wait-online.service
```

It runs in parallel with the desktop, so disabling it may not change time-to-desktop.

## 7. Boot timeout

The systemd-boot timeout is usually five seconds. Back up the loader config, check the timeout, and set it to two:

```bash
sudo cp /boot/loader/loader.conf /boot/loader/loader.conf.bak
sudo grep -nE '^timeout[[:space:]]' /boot/loader/loader.conf
sudo sed -i -E 's/^timeout[[:space:]].*/timeout 2/' /boot/loader/loader.conf
```

If there is no `timeout` line, add `timeout 2` at the top with `sudoedit /boot/loader/loader.conf`. The new timeout applies at the next boot.

Check boot space after installing the LTS kernel:

```bash
df -h /boot
```

Run `sudo sdboot-manage gen` after any kernel or initramfs change.

## 8. Desktop and hardware

KDE Plasma runs on Wayland with fish as the login shell. Baloo is disabled by CachyOS; leave it that way unless you need file search indexing.

Check the GPU stack:

```bash
shelly install standard libva-utils vulkan-tools
vainfo
vulkaninfo --summary
```

Mesa, RADV, and VA-API are included in the normal repositories.

Add metric-compatible fonts:

```bash
shelly install standard ttf-liberation ttf-carlito ttf-caladea
```

Flatpak already has the Flathub remote; check it with `flatpak remotes`. No `remote-add` is needed.

Printing is optional:

```bash
shelly install standard cups print-manager
sudo systemctl enable --now cups.socket
```

Dolphin uses Ark for archive handling. Ark covers ZIP and TAR through libarchive and libzip; add 7zip and unrar for 7z and RAR:

```bash
shelly install standard 7zip unrar
```

NTSYNC is loaded by CachyOS even without a gaming stack. It is harmless; no action is needed.

## 9. Optional applications

Repository packages:

```bash
shelly install standard yt-dlp nvme-cli
```

Wayland clipboard:

```bash
shelly install standard wl-clipboard
```

Node.js, if needed. The repos carry the current Node release and npm:

```bash
shelly install standard nodejs npm
```

For multiple Node versions or a newer upstream release than the repos carry, install `fnm` with Shelly:

```bash
shelly install standard fnm
```

Build tools:

```bash
shelly install standard base-devel gcc clang cmake ninja meson
```

Python, current from the repos:

```bash
shelly install standard python python-pip python-virtualenv
```

Go:

```bash
shelly install standard go
```

Java, if needed:

```bash
shelly install standard jdk-openjdk
```

Zed:

```bash
shelly install standard zed
```

Audio tools, if needed:

```bash
shelly install standard easyeffects lsp-plugins calf
```

AUR packages use Shelly:

```bash
shelly install aur ttf-ms-fonts
```

The Microsoft fonts package requires accepting its EULA.

If the gaming stack is installed, remove the metapackages and Steam with Shelly. The default cascade removes dependencies that nothing else needs, including 7zip, Wine, and the launchers if only the gaming stack used them:

```bash
shelly remove standard cachyos-gaming-applications cachyos-gaming-meta
shelly remove standard steam
```

## 10. Verify

```bash
uname -r
grep -E '^\[cachyos-(v3|core-v3|extra-v3)\]' /etc/pacman.conf
vainfo
vulkaninfo --summary
zramctl
findmnt -no FSTYPE,OPTIONS /
systemctl is-enabled fstrim.timer
sudo bootctl status
sudo ufw status verbose
systemctl --failed
powerprofilesctl get
upower -i /org/freedesktop/UPower/devices/battery_BAT0 | grep -Ei 'threshold|capacity'
df -h /boot
shelly list-updates
```

Expected: the default CachyOS kernel with the LTS fallback installed, v3 repositories, working VA-API and Vulkan, zram swap, ext4 on `/` with `noatime`, `fstrim.timer` enabled, systemd-boot active, UFW active, no failed units, an 80% battery limit, and free space in `/boot`.

## 11. Recovery

This setup has no bootable snapshots. Boot a CachyOS live USB, or restore from the external restic or borg backup.

For package rollback, use Shelly:

```bash
shelly downgrade
```

The pacman cache keeps the last three versions. List the exact filename before a manual install:

```bash
ls /var/cache/pacman/pkg/ | grep '^linux-cachyos'
```

If the system will not boot, use a CachyOS live USB and its `cachy-chroot` helper. Install the `cachy-chroot` package in the live session if it is missing.

Hibernation is not configured because swap is zram only. To enable it, add a swapfile or swap partition and configure resume; zram cannot hibernate.

## Notes

The installer already configured KDE, systemd-boot, ext4, the v3 repositories, and zram. This guide covers only the changes worth making after first boot. CachyOS is rolling; re-check package names and defaults after major updates.
