# KLP1 Printer System Setup

This document records the changes made to the printer's operating system, why each change was needed, and how to do it again. Use it after you reflash the SD card or move to a new controller board.

The Klipper configuration is in this repository. The changes in this document are not. They live on the printer's SD card and are lost when the card is reflashed.

---

## System facts

| Item | Value |
| --- | --- |
| Controller board | MKS Pi (Armbian board name `mkspi`) |
| Operating system | Armbian 22.05.0-trunk on Debian 10 "buster" |
| Kernel | `5.16.20-rockchip64` (edge branch) |
| Klipper | v0.11.0-122 (February 2023) |
| Python for Klipper, Moonraker, KlipperScreen, Obico | 3.7.3 |
| SSH user | `mks` |
| Config folder | `/home/mks/printer_data/config` |
| Network | Ethernet `eth0` is the main connection. Wi-Fi `wlan0` is also active. |

Replace `<printer-ip>` in the commands below with the printer's address. Fluidd shows it, and so does your router.

---

## Summary of changes

| Date | Change | Section |
| --- | --- | --- |
| 2026-09-16 | SSH key login from the Mac | [1](#1-ssh-key-login) |
| 2026-10-02 | Time zone set to `America/Sao_Paulo` | [2](#2-clock-and-time-zone) |
| 2026-10-02 | `systemd-timesyncd` masked, so `ntp` starts at boot | [2](#2-clock-and-time-zone) |
| 2026-10-02 | Debian package sources moved to `archive.debian.org` | [3](#3-debian-package-sources) |
| 2026-10-02 | 156 Debian packages updated. Kernel and bootloader kept. | [4](#4-system-update) |
| 2026-10-02 | vnStat `eth0` database reset | [5](#5-vnstat-error-in-the-login-banner) |

---

## 1. SSH key login

**Why.** A key lets the Mac connect to the printer without a password. Scripts and tools that copy config files or read logs need this, because they cannot type a password.

**How.** Run these on the Mac.

1. Create a key if `~/.ssh/id_ed25519` does not exist:
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
   ```
2. Copy the key to the printer. This asks for the `mks` password one time:
   ```bash
   ssh-copy-id -i ~/.ssh/id_ed25519.pub mks@<printer-ip>
   ```
3. Test it. This must connect without a password prompt:
   ```bash
   ssh -o BatchMode=yes mks@<printer-ip> 'echo ok'
   ```

The key gives login access only. Commands that start with `sudo` still ask for the `mks` password.

---

## 2. Clock and time zone

**Symptom.** The clock was 9 days behind and set to Hong Kong time (UTC+8). Fluidd showed wrong times for print history. `apt update` failed with `Release file ... is not valid yet`.

**Cause.** Four things combine:

1. The board has no clock battery. Its hardware clock reads 2016.
2. `fake-hwclock` restores the time saved at the last shutdown. After the printer is off for 9 days, the clock starts 9 days behind.
3. MKS added a boot script, `/root/ntp_time.sh` (service `makerbase-ntp-time`). It runs `ntpdate` one time, but it discards the result. It prints "time synced successfully" (in Chinese) even when the sync fails.
4. The `ntp` service never started at boot. Debian enables both `ntp` and `systemd-timesyncd`, and the two services conflict. At boot, systemd queues `systemd-timesyncd` first, which cancels the `ntp` start. Then `systemd-timesyncd` skips itself because `ntp` is installed. Neither service runs.

The MKS image also ships with the time zone set to `Asia/Hong_Kong`.

**Fix.** Connect with `ssh mks@<printer-ip>`, then run these one at a time:

```bash
sudo timedatectl set-timezone America/Sao_Paulo
```
```bash
sudo systemctl mask systemd-timesyncd
```
```bash
sudo systemctl daemon-reload && sudo systemctl enable --now ntp
```

Wait until `ntpq -pn` shows a line that starts with `*`. That line is the server `ntp` uses. Then save the correct time for the next boot:

```bash
sudo fake-hwclock save
```

`ntp` starts with the `-g` option (set in `/etc/default/ntp`). This option lets `ntp` correct a large error in one step at startup, so a 9-day gap does not block it.

**Check.**

```bash
date
```
```bash
ntpq -pn
```

`date` must show the local time with `-03`. `ntpq -pn` must show one server marked `*` with an offset of a few milliseconds. `timedatectl` can show `System clock synchronized: no` for about 15 min after boot. This is a status flag only. Trust `ntpq`.

The MKS script still runs at every boot. When `ntp` is already running, its `ntpdate` call fails and the script still prints "success". This is harmless because `ntp` sets the clock.

---

## 3. Debian package sources

**Symptom.** `apt update` printed `404 Not Found` for every Debian source.

**Cause.** Debian 10 "buster" reached end of life in 2024. Debian moved its packages from `deb.debian.org` and `security.debian.org` to `archive.debian.org`.

**Fix.** Fix the clock first ([section 2](#2-clock-and-time-zone)). A wrong clock makes `apt` reject the Armbian source.

Then change the two Debian addresses. The command keeps a copy of the old file as `/etc/apt/sources.list.bak`. It does not change the Armbian source, which is in `/etc/apt/sources.list.d/`.

```bash
sudo sed -i.bak -e 's#http://deb.debian.org/debian#http://archive.debian.org/debian#' -e 's#http://security.debian.org/#http://archive.debian.org/debian-security/#' /etc/apt/sources.list
```

The archive release files have no `Valid-Until` date, so you do not need to turn off the `apt` date check.

**Check.** `sudo apt update` must finish with no `Err` or `E:` lines.

**Undo.**

```bash
sudo mv /etc/apt/sources.list.bak /etc/apt/sources.list
```

Debian publishes no new updates for buster. After this change, `apt` installs the final buster versions and nothing newer.

---

## 4. System update

**Why.** The update installed the final Debian 10 security fixes, for example OpenSSL, GnuTLS, curl, OpenSSH, the DHCP client and time zone data.

**What must not change.** MKS put these packages on hold. Keep them on hold:

| Package | Why |
| --- | --- |
| `linux-image-edge-rockchip64` | Kernel. A different kernel can stop the MKS Pi from booting. |
| `linux-dtb-edge-rockchip64` | Device tree that describes the MKS Pi hardware. |
| `linux-u-boot-mkspi-edge` | Bootloader. |

`apt` also keeps back `armbian-bsp-cli-mkspi`, the MKS board support package.

**Do not run:**

- `sudo apt autoremove`. It removes `armbian-bsp-cli-mkspi`.
- `sudo apt full-upgrade` or `sudo apt dist-upgrade`.
- `sudo apt-mark unhold ...` on the packages in the table.
- The kernel options in `armbian-config`.

**How.**

1. Make sure no print is running.
2. Check the free space. The update needs about 0.8 GB:
   ```bash
   df -h /
   ```
3. Run the update:
   ```bash
   sudo apt update && sudo apt upgrade
   ```
   Before you type `Y`, read the summary. The `linux-...` packages must be in the list "kept back".
4. Answer the prompts:
   - "Modified configuration file": keep the installed version. This is the default, so press Enter.
   - "Restart services without asking": Yes.
5. Do not lose power or close the session during the install. If the connection drops, connect again and run `sudo dpkg --configure -a`.

**Check before you reboot.** The update rebuilds the boot image (`/boot/uInitrd`). A broken boot image stops the board from booting, so check it:

```bash
dpkg --audit
```
```bash
gzip -t /boot/initrd.img-$(uname -r) && echo "gzip OK"
```
```bash
zcat /boot/initrd.img-$(uname -r) | cpio -t 2>/dev/null | wc -l
```

`dpkg --audit` must print nothing. The `gzip` check must print `gzip OK`. The file count was 964 on 2026-10-02.

During the update, this message appears:

```
ln: failed to create hard link '/boot/initrd.img-...dpkg-bak' => ...: Operation not permitted
```

It is harmless. `/boot` uses the FAT file system, which cannot make hard links, so `update-initramfs` skips its backup copy and continues.

Then clean up and reboot:

```bash
sudo apt clean
```
```bash
sudo reboot
```

**Check after the reboot.** `uname -r` must still show `5.16.20-rockchip64`. Fluidd must show Klipper as ready.

---

## 5. vnStat error in the login banner

**Symptom.** The SSH login banner showed `Error: eth0: Invalid database daily date order`.

**Cause.** vnStat counts network traffic. Its `eth0` database dated from 2019 and the clock jumps made it unreadable. Nothing in the printer uses vnStat.

**Fix.** This system has vnStat 1.18. The `--remove` and `--add` options belong to vnStat 2 and do not work here. Delete the database and create a new one:

```bash
sudo systemctl stop vnstat
```
```bash
sudo rm /var/lib/vnstat/eth0 /var/lib/vnstat/.eth0
```
```bash
sudo -u vnstat vnstat -u -i eth0
```
```bash
sudo systemctl start vnstat
```

The third command prints `Unable to read database` and then `A new database has been created`. This is correct. Run it as the `vnstat` user so that the service can write to the new file.

---

## 6. Boot errors you can ignore

`systemctl --failed` lists these 4 services. They came with the MKS image and do not affect printing.

| Service | Why it fails |
| --- | --- |
| `networking` | The MKS config starts a CAN bus (`can0`). This printer connects the toolhead board with a serial cable, not CAN. Another service manages the network. |
| `makerbase-net-mods` | It copies Wi-Fi settings from a USB stick. No USB stick is present. |
| `haveged` | An extra random-number service that does not work with this kernel. The kernel has its own. |
| `smartd` | A disk health monitor. The SD card does not support SMART. |

---

## 7. Klipper version

The printer runs Klipper v0.11 from February 2023. Do not update Klipper from Fluidd or Moonraker without a plan:

- A Klipper update usually needs new firmware on both microcontrollers: the main board (STM32) and the toolhead board (`MKS_THR`).
- Some macros in `macros.cfg` work around missing v0.11 features. For example, `_ADAPTIVE_MESH` replaces `BED_MESH_CALIBRATE ADAPTIVE=1`, which came in v0.12.

---

## 8. Klipper configuration

The Klipper configuration files are in this repository. See the README for the restore steps. Remember these points:

- On the printer, `mainsail.cfg` is a link to `~/mainsail-config/client.cfg`. Do not replace the link with a file.
- `moonraker-obico.cfg` contains the Obico token and is not in the repository. Link the printer to Obico again after a reflash.
- Before you copy a changed file to the printer, keep a backup of the old one. After the copy, run `FIRMWARE_RESTART` and check that Klipper reports ready.
- The OrcaSlicer profiles are in [`orca/`](../orca/). The start G-code, end G-code, pause G-code and filament change G-code must match the macros. See the README.
