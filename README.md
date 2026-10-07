# Fedora Workstation (GNOME) Post Install Guide

Things to do after installing Fedora Workstation (GNOME). Written for an ASUS VivoBook X409DA (AMD Ryzen 5 3500U / Vega 8, NVMe, Btrfs); most steps suit any Fedora GNOME machine.

Run top to bottom. Each block is copy-paste into a terminal. Reboot where told. Graphics, session, and kernel changes need it.

## DNF tuning

Speed up dnf before the first upgrade (10 parallel downloads, assume yes):

```bash
sudo dnf config-manager setopt max_parallel_downloads=10 defaultyes=True
```

Check it (should print `max_parallel_downloads = 10` and `defaultyes = 1`):

```bash
dnf --dump-main-config | grep -E '^(max_parallel_downloads|defaultyes)'
```

## Update

Update everything first, then reboot:

```bash
sudo dnf upgrade --refresh -y
sudo systemctl reboot
```

## RPM Fusion

Fedora leaves out non-free software (codecs, drivers, firmware) by default. Enable RPM Fusion:

```bash
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

```bash
sudo dnf upgrade --refresh -y
```

Show RPM Fusion apps in GNOME Software:

```bash
sudo dnf install 'rpmfusion-*-appstream-data'
```

Enable the Cisco OpenH264 repo (Firefox uses it for WebRTC calls):

```bash
sudo dnf config-manager setopt fedora-cisco-openh264.enabled=1
```

## Multimedia

Full ffmpeg instead of the stripped `ffmpeg-free`, plus the codec complements for GStreamer apps (straight from the [RPMFusion docs](https://rpmfusion.org/Howto/Multimedia)):

```bash
sudo dnf swap ffmpeg-free ffmpeg --allowerasing || sudo dnf install -y ffmpeg
```

```bash
sudo dnf install @multimedia --setopt="install_weak_deps=False" --exclude=PackageKit-gstreamer-plugin
```

AMD hardware decode (H.264/H.265/VC-1 on this Vega 8 — stock Mesa leaves these out):

```bash
sudo dnf install -y mesa-va-drivers-freeworld
```

```bash
sudo dnf swap mesa-vulkan-drivers mesa-vulkan-drivers-freeworld || sudo dnf install -y mesa-vulkan-drivers-freeworld
```

If you play 32-bit Steam/Wine games, add the i686 freeworld packages too:

```bash
sudo dnf install -y mesa-va-drivers-freeworld.i686
```

## [Optional] Gaming (Steam)

Steam comes from RPM Fusion (already enabled). Installing it pulls in `ntsync-autoload` — which loads the kernel's NTSYNC module at boot, used automatically by Wine 11 / Proton — plus `gamemode` and the 32-bit bits, as weak dependencies:

```bash
sudo dnf install -y steam
```

Reboot (or run `sudo modprobe ntsync`) and `/dev/ntsync` will exist. Steam only pulls `ntsync-autoload` through *weak* dependencies, so if anything was installed with `--setopt=install_weak_deps=False`, add it explicitly:

```bash
sudo dnf install -y ntsync-autoload
```

## Firmware

If your system supports firmware delivery through LVFS:

```bash
fwupdmgr refresh --force
```

```bash
fwupdmgr get-devices
```

Lists devices with available updates.

```bash
fwupdmgr get-updates
```

Fetches the list of available updates.

```bash
fwupdmgr update
```

## Flatpak

Enable access to all Flathub flatpaks (skip if you ticked "Enable Third Party Repositories" on first boot):

```bash
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

## Extra apps

Compilers and build tools as a group:

```bash
sudo dnf group install -y development-tools
```

This installs the extra apps. Workstation already ships GNOME Shell, Nautilus, Ptyxis, GNOME Text Editor, and PipeWire, so those are not listed. `nss-mdns` below covers `.local` discovery (it pulls in Avahi itself). Niche extras are left out; install them from GNOME Software when you need them.

```bash
sudo dnf install -y gnome-tweaks gnome-shell-extension-appindicator gnome-shell-extension-dash-to-dock file-roller python3-pip python3-virtualenv java-latest-openjdk-devel golang mesa-dri-drivers vulkan-tools libva libva-utils dav1d libheif libavif libjxl libwebp mpv pipewire-pulseaudio pipewire-alsa alsa-sof-firmware alsa-ucm alsa-utils bluez firefox qbittorrent libreoffice 7zip unzip xdg-user-dirs cups snapper btrfs-assistant btrfsmaintenance easyeffects lsp-plugins calf smartmontools nvme-cli zram-generator flatpak fwupd nss-mdns openssh rsync dosfstools mtools usbutils unrar yt-dlp zsh
```

## Battery charge limit (60%) [ASUS-only]

[ASUS VivoBook X409DA only — skip on other machines.] This ASUS exposes charge control directly, so no TLP is needed. Stop charging at 60% to slow battery wear. Skip if the path below does not exist:

```bash
[ -e /sys/class/power_supply/BAT0/charge_control_end_threshold ] && echo 60 | sudo tee /sys/class/power_supply/BAT0/charge_control_end_threshold || echo "no BAT0 charge control — skipping"
```

Make it survive reboots (ASUS-only — skip if the path above does not exist):

```bash
[ -e /sys/class/power_supply/BAT0/charge_control_end_threshold ] && printf 'w /sys/class/power_supply/BAT0/charge_control_end_threshold - - - - 60\n' | sudo tee /etc/tmpfiles.d/battery-charge-limit.conf >/dev/null || echo "no BAT0 charge control — skipping"
```

Check it (should print `60`):

```bash
[ -e /sys/class/power_supply/BAT0/charge_control_end_threshold ] && cat /sys/class/power_supply/BAT0/charge_control_end_threshold || echo "no BAT0 charge control — skipping"
```

## Shell (zsh + Starship)

Make zsh the login shell, install the Starship prompt upstream (no Fedora package), and add the shell setup block:

```bash
chsh -s $(command -v zsh)
```

```bash
curl -sS https://starship.rs/install.sh | sh -s -- -y
```

Append this base block to `~/.zshrc`. It is guarded, so re-running is safe. It needs nothing beyond zsh + Starship:

```zsh
# BEGIN SETUP BLOCK
export PATH="$HOME/.local/bin:$PATH"
autoload -U compinit
compinit
setopt COMPLETE_IN_WORD
HISTFILE=~/.zsh_history
HISTSIZE=10000
SAVEHIST=10000
setopt appendhistory
setopt SHARE_HISTORY
setopt autocd
unsetopt nomatch
eval "$(starship init zsh)"
bindkey "^[[1;5D" backward-word
bindkey "^[[1;5C" forward-word
bindkey '^H' backward-kill-word
bindkey '^[[3;5~' kill-word
bindkey "^[[3~" delete-char
bindkey '^[[H' beginning-of-line
bindkey '^[[F' end-of-line
# END SETUP BLOCK
```

[Optional — only if you did the Node.js (fnm) section and want the plugins.] First install the plugins, then append this second block:

```bash
sudo dnf install -y zsh-autosuggestions fzf zsh-syntax-highlighting
```

```zsh
# BEGIN OPTIONAL BLOCK (fnm + plugins)
export PATH="$HOME/.local/share/fnm:$PATH"
if command -v fnm >/dev/null 2>&1; then
    eval "$(fnm env --use-on-cd --resolve-engines --shell zsh)"
fi
[ -f /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh ] && source /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh
[ -f /usr/share/fzf/shell/key-bindings.zsh ] && source /usr/share/fzf/shell/key-bindings.zsh
[ -f /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh ] && source /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
# END OPTIONAL BLOCK
```

## Node.js (fnm) [Optional]

Fedora ships no `fnm` package, so install it upstream, then take the latest Node:

```bash
curl -fsSL https://fnm.vercel.app/install | bash -s -- --skip-shell
```

```bash
export PATH="$HOME/.local/share/fnm:$PATH" && eval "$(fnm env --shell bash)" && fnm install --latest && fnm default $(fnm ls | grep -oE 'v[0-9]+\.[0-9]+\.[0-9]+' | sort -V | tail -1)
```

## GNOME polish

Dark theme, volume overamplification (up to 150%), three window buttons, two-finger touchpad scrolling, and quietening GNOME Software:

```bash
gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'
```

```bash
gsettings set org.gnome.desktop.sound allow-volume-above-100-percent true
```

```bash
gsettings set org.gnome.desktop.wm.preferences button-layout 'appmenu:minimize,maximize,close'
```

```bash
gsettings set org.gnome.desktop.peripherals.touchpad two-finger-scrolling-enabled true
```

Stop GNOME Software's background updates and search provider [Optional — you lose background updates; GNOME Software still updates when you open it]:

```bash
systemctl --user mask gnome-software.service
```

```bash
gsettings set org.gnome.desktop.search-providers disabled "['org.gnome.Software.desktop']"
```

Enable the extensions installed above (AppIndicator for tray icons, Dash to Dock):

```bash
gnome-extensions enable appindicatorsupport@rgcjonas.gmail.com
```

```bash
gnome-extensions enable dash-to-dock@micxgx.gmail.com
```

## [Optional] Extensions (catalog)

These two aren't in the Fedora repos. Install them from extensions.gnome.org with the Extensions app — it checks GNOME version compatibility and handles updates:

```bash
sudo dnf install -y gnome-extensions-app
```

Open it, search for **Clipboard Indicator** and **Bluetooth Quick Connect**, install them, and toggle them on. (Log out and back in if they don't appear.) Or enable them from a terminal:

```bash
gnome-extensions enable clipboard-indicator@tudmotu.com
```

```bash
gnome-extensions enable bluetooth-quick-connect@bjarosze.gmail.com
```

## Fonts

Metric-compatible fonts (same layout as Arial, Times, Courier, Calibri, Cambria — no EULA):

```bash
sudo dnf install -y liberation-sans-fonts liberation-serif-fonts liberation-mono-fonts google-carlito-fonts google-crosextra-caladea-fonts
```

Real Microsoft fonts [Optional]:

```bash
sudo dnf install -y lpf-mscore-fonts
```

Then follow the prompts (approves the EULA, downloads from SourceForge, builds and installs a local RPM):

```bash
lpf update mscore-fonts
```

## System tuning

Cap the journal at 200M:

```bash
sudo mkdir -p /etc/systemd/journald.conf.d && printf '[Journal]\nSystemMaxUse=200M\nSystemMaxFiles=5\nSyncIntervalSec=5m\n' | sudo tee /etc/systemd/journald.conf.d/99-ssd.conf >/dev/null && sudo systemctl restart systemd-journald
```

zstd compression for zram (Fedora defaults to lzo-rle; keep the default size, only switch the algorithm):

```bash
printf '[zram0]\ncompression-algorithm = zstd\n' | sudo tee /etc/systemd/zram-generator.conf >/dev/null && sudo systemctl daemon-reload && sudo systemctl start systemd-zram-setup@zram0.service
```

Kernel and memory sysctls (aggressive swap into zram, Proton map count, inotify capacity):

```bash
sudo mkdir -p /etc/sysctl.d && printf 'vm.swappiness = 180\nvm.page-cluster = 0\nvm.watermark_boost_factor = 0\nvm.watermark_scale_factor = 125\nvm.max_map_count = 1048576\nvm.vfs_cache_pressure = 50\nfs.inotify.max_user_watches = 524288\nfs.inotify.max_user_instances = 8192\n' | sudo tee /etc/sysctl.d/99-performance.conf >/dev/null && sudo sysctl --system
```

Skip the boot-delaying waiter:

```bash
sudo systemctl disable NetworkManager-wait-online.service
```

## Firewall

Fedora ships `firewalld`. Use it and don't install `ufw` alongside it:

```bash
sudo systemctl enable --now firewalld
```

## Btrfs

The installer already sets `compress=zstd:1` on `/` and `/home`. Add `noatime` to Btrfs lines missing it (safe to re-run), then verify:

```bash
sudo sed -i -E '/[[:space:]]btrfs[[:space:]]/ { /noatime/! s/(btrfs +)([^ ]+)/\1noatime,\2/; }' /etc/fstab
```

Must pass before rebooting:

```bash
sudo findmnt --verify
```

`fstrim.timer` (enabled below) stays for `/boot`, which is ext4. The Btrfs mounts discard async on their own.

## Snapshots

Snapper takes timeline snapshots (via `snapper-timeline.timer`, enabled below). Note: Fedora 44's default `dnf` is dnf5, which cannot load the dnf4 `python3-dnf-plugin-snapper`, so there are **no automatic pre/post snapshots** on dnf transactions — use `btrfs-assistant` or `snapper create` before risky changes. (A `libdnf5-plugin-actions` workaround exists; see Fedora Discussion.) Create the configs if the installer didn't:

Skip if the installer already made it, since it errors when the config exists:

```bash
sudo snapper -c root create-config /
```

```bash
sudo snapper -c home create-config /home
```

Monthly scrubs are covered by `btrfs-scrub.timer` (enabled below).

`grub-btrfs` is not in the Fedora repos, so there is no GRUB snapshot menu. Restore from a live USB if needed.

## Boot

Microcode ships via `linux-firmware`. No action is needed.

2s GRUB timeout:

```bash
sudo sed -i 's/^GRUB_TIMEOUT=.*/GRUB_TIMEOUT=2/' /etc/default/grub && sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

## Sudo feedback

Show `*` while typing the sudo password:

```bash
echo 'Defaults pwfeedback' | sudo tee /etc/sudoers.d/pwfeedback >/dev/null && sudo chmod 0440 /etc/sudoers.d/pwfeedback
```

## Services

Enable everything in one go (the rest — display manager, NetworkManager, firewalld, Bluetooth, printing, trim, smartd, Avahi — ships enabled):

```bash
sudo systemctl enable snapper-timeline.timer snapper-cleanup.timer btrfs-scrub.timer fwupd-refresh.timer
```

```bash
sudo systemctl reboot
```

## Verify after reboot

VA-API on Vega 8:

```bash
vainfo
```

RADV Vulkan:

```bash
vulkaninfo --summary
```

Compressed swap:

```bash
zramctl
```

Btrfs mount options:

```bash
findmnt /
```

Firewall running:

```bash
firewall-cmd --state
```

[ASUS-only] `60` = limit active:

```bash
[ -e /sys/class/power_supply/BAT0/charge_control_end_threshold ] && cat /sys/class/power_supply/BAT0/charge_control_end_threshold || echo "no BAT0 charge control — skipping"
```

Pre/post snapshots:

```bash
sudo snapper list
```

SELinux enforcing:

```bash
getenforce
```

Node.js through fnm:

```bash
node -v
```

Latest OpenJDK SDK:

```bash
java --version
```

`/usr/bin/zsh` after re-login:

```bash
echo $SHELL
```

## Notes

**Rollback** is native:

```bash
sudo dnf history rollback
```

```bash
sudo dnf downgrade '<pkg>-<ver>'
```

**Hibernation is not configured** — swap is zram only.
