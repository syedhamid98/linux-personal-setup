# Fedora Workstation: New Installation Guide for personal use

Personal setup notes for a fresh Fedora Workstation install. Work through the sections in order; each one is independent enough to re-run on its own.

> **Target:** Fedora Workstation 41 or newer (DNF5). Package names and options were written from knowledge of Fedora's repos, not verified on a live install. If `dnf` reports "No match for argument", drop that package and carry on.

## Contents

1. [Configure DNF](#1-configure-dnf)
2. [Enable repositories](#2-enable-repositories)
3. [Update and reboot](#3-update-and-reboot)
4. [Multimedia codecs](#4-multimedia-codecs)
5. [Install software (dnf)](#5-install-software-dnf)
6. [Install software (Flatpak)](#6-install-software-flatpak)
7. [Optional: disable IPv6](#7-optional-disable-ipv6)
8. [Optional: mount a NAS share](#8-optional-mount-a-nas-share)
9. [Notes and troubleshooting](#9-notes-and-troubleshooting)

---

## 1. Configure DNF

Append faster-download settings to `/etc/dnf/dnf.conf`:

```bash
echo 'max_parallel_downloads=10' | sudo tee -a /etc/dnf/dnf.conf
echo 'defaultyes=True'          | sudo tee -a /etc/dnf/dnf.conf
```

- `max_parallel_downloads=10` speeds up large updates (the default is much lower).
- `defaultyes=True` makes Enter answer "yes" at prompts.
- `deltarpm` is no longer supported in DNF5, so it is omitted here. `fastestmirror` is also left out; add it only if your DNF version documents it.

## 2. Enable repositories

### RPM Fusion (free and nonfree)

```bash
sudo dnf install \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

Reference: <https://rpmfusion.org/Configuration>

### Flathub

```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

Fedora may already ship a filtered Flathub remote. If apps you expect are missing, remove the filter:

```bash
flatpak remote-modify --no-filter flathub
```

Reference: <https://flatpak.org/setup/>

### Visual Studio Code (Microsoft repo)

VS Code is not in Fedora's default repos. Installing it from Microsoft's repo (instead of Flatpak) avoids sandbox limits when using terminals and toolchains.

```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
printf '[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\nautorefresh=1\ntype=rpm-md\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc\n' \
  | sudo tee /etc/yum.repos.d/vscode.repo > /dev/null
```

## 3. Update and reboot

```bash
sudo dnf upgrade --refresh
sudo reboot
```

## 4. Multimedia codecs

Needed for VLC, HandBrake and Kdenlive to play and encode common formats. Run after RPM Fusion is enabled and the system is updated:

```bash
sudo dnf swap ffmpeg-free ffmpeg --allowerasing
sudo dnf update @multimedia --setopt="install_weak_deps=False" --exclude=PackageKit-gstreamer-plugin
```

For GPU video acceleration (Intel, AMD, NVIDIA), follow the RPM Fusion hardware acceleration and NVIDIA pages for your hardware.

## 5. Install software (dnf)

```bash
sudo dnf install \
  vim-enhanced htop vlc wireshark nmap nmap-ncat tcpdump \
  plocate curl wget git net-tools unzip bash-completion \
  timeshift kdenlive filezilla kvantum \
  android-tools android-file-transfer code
```

Notes:

- `plocate` replaces the older `mlocate`. Build its index with `sudo updatedb`.
- `nmap-ncat` provides `nc`.
- `code` comes from the Microsoft repo added in section 2.
- `ipscan` (Angry IP Scanner) is not in Fedora's repos. Install it from the upstream RPM if you still want it.

### Wireshark without root

```bash
sudo usermod -aG wireshark "$USER"
```

Log out and back in for the group change to apply.

## 6. Install software (Flatpak)

```bash
flatpak install -y flathub \
  com.bitwarden.desktop \
  org.telegram.desktop \
  org.qbittorrent.qBittorrent \
  com.stremio.Stremio \
  fr.handbrake.ghb \
  com.obsproject.Studio \
  org.flameshot.Flameshot \
  org.freefilesync.FreeFileSync \
  com.github.tchx84.Flatseal \
  com.discordapp.Discord \
  com.github.jeromerobert.pdfarranger
```

| App | Purpose |
| --- | --- |
| Bitwarden | Password manager |
| Telegram, Discord | Messaging |
| qBittorrent | Torrent client |
| Stremio | Media center |
| HandBrake | Video transcoding |
| OBS Studio | Recording and streaming |
| Flameshot | Screenshots |
| FreeFileSync | Folder sync |
| Flatseal | Manage Flatpak permissions |
| PDF Arranger | Merge, split and reorder PDFs |

## 7. Optional: disable IPv6

Only do this if you have a reason (for example, a VPN that leaks over IPv6). Disabling it can break some networks and services.

```bash
sudo tee /etc/sysctl.d/99-disable-ipv6.conf > /dev/null <<'EOF'
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
EOF
sudo sysctl --system
```

This single drop-in is enough. The GRUB method below is only needed if you want IPv6 off at the kernel level, and the sysctl file makes it redundant for most setups:

```bash
sudo grubby --update-kernel=ALL --args="ipv6.disable=1"
```

**To undo:** delete `/etc/sysctl.d/99-disable-ipv6.conf`, reboot, and if you used grubby, run `sudo grubby --update-kernel=ALL --remove-args="ipv6.disable=1"`.

## 8. Optional: mount a NAS share

Replace `<IP>`, `<SHARE>`, `<NAS_USER>` and `<LOCAL_USER>` with your values.

```bash
sudo dnf install cifs-utils
mkdir -p ~/nas
```

### Credentials file

```bash
cat > ~/.smbcredentials <<'EOF'
username=<NAS_USER>
password=<NAS_PASSWORD>
EOF
chmod 600 ~/.smbcredentials
```

Using a credentials file keeps the password out of your shell history and process list.

### Temporary mount

```bash
sudo mount -t cifs //<IP>/<SHARE> ~/nas \
  -o uid=$(id -u),gid=$(id -g),credentials=$HOME/.smbcredentials
```

### Permanent mount

Add this line to `/etc/fstab` (use absolute paths; `$HOME` is not expanded there):

```fstab
//<IP>/<SHARE> /home/<LOCAL_USER>/nas cifs uid=1000,gid=1000,credentials=/home/<LOCAL_USER>/.smbcredentials,_netdev,nofail,x-systemd.automount 0 0
```

Then test it:

```bash
sudo systemctl daemon-reload
sudo mount -a
```

- `nofail` stops a missing NAS from blocking boot.
- `_netdev` waits for the network before mounting.
- `x-systemd.automount` mounts on first access.
- If the share needs a specific protocol, add `vers=3.0` to the options.

## 9. Notes and troubleshooting

- **Timeshift and Btrfs:** Fedora's default Btrfs layout (`root` and `home` subvolumes) generally isn't what Timeshift's Btrfs mode expects (`@` and `@home`). Use Timeshift in rsync mode, or use Btrfs Assistant / Snapper instead.
- **Package not found:** run `dnf search <name>`, or check whether it needs RPM Fusion or a third-party repo.
- **Flatpak app can't see files or devices:** open Flatseal and grant the permission.
- **Check Fedora version:** `rpm -E %fedora`
- **List what you installed from Flatpak:** `flatpak list --app`

## Changelog

- Rewrote as a structured guide for DNF5 (Fedora 41+).
- Replaced `mlocate` with `plocate`; added `nmap-ncat`.
- Removed duplicate VS Code install (RPM from Microsoft repo only).
- Added update, codec and Wireshark steps; moved IPv6 and NAS to optional sections.
- Hardened NAS mount (credentials file with `chmod 600`, `nofail`, `_netdev`).
