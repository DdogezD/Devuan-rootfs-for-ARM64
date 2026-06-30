# Devuan-rootfs-for-ARM64

Prebuilt Devuan GNU/Linux 6 (excalibur) ARM64 rootfs for use as a Termux
chroot with XFCE4 desktop.

## Quick start

### Download

Get the latest `devuan.tar.gz` from
[Releases](https://github.com/DdogezD/Devuan-rootfs-for-ARM64/releases).

Extract into your chroot directory:

```sh
mkdir -p /data/local/tmp/devuan
tar xzf devuan.tar.gz -C /data/local/tmp/devuan
```

### Startup script

A working `termux-login.sh` for Termux with `termux-x11` and KernelSU:

```sh
#!/bin/sh

# step1: termux side
settings put global settings_enable_monitor_phantom_procs false
killall -9 termux-x11 pulseaudio 2>/dev/null
XDG_RUNTIME_DIR=${TMPDIR} termux-x11 :0 -ac &
pulseaudio --start \
  --load="module-aaudio-sink" \
  --load="module-native-protocol-tcp auth-ip-acl=127.0.0.1 auth-anonymous=1" \
  --exit-idle-time=-1
sudo /data/adb/ksu/bin/busybox mount --bind $PREFIX/tmp /data/local/tmp/devuan/tmp
sleep 3
am start --user 0 -n com.termux.x11/com.termux.x11.MainActivity
sleep 1

# step2: chroot mounts + XFCE
sudo sh -c '
/data/adb/ksu/bin/busybox mount -o remount,dev,suid /data
/data/adb/ksu/bin/busybox mount --bind /dev /data/local/tmp/devuan/dev
/data/adb/ksu/bin/busybox mount --bind /sys /data/local/tmp/devuan/sys
/data/adb/ksu/bin/busybox mount --bind /proc /data/local/tmp/devuan/proc
/data/adb/ksu/bin/busybox mount -t devpts devpts /data/local/tmp/devuan/dev/pts
mkdir -p /data/local/tmp/devuan/dev/shm
/data/adb/ksu/bin/busybox mount -t tmpfs -o size=256M tmpfs /data/local/tmp/devuan/dev/shm
mkdir -p /data/local/tmp/devuan/sdcard
/data/adb/ksu/bin/busybox mount --bind /sdcard /data/local/tmp/devuan/sdcard
/data/adb/ksu/bin/busybox chroot /data/local/tmp/devuan /bin/su - DogEZ -c "
    sudo chmod -R 1777 /tmp &&
    sudo chown root:root /tmp/.ICE-unix 2>/dev/null &&
    sudo chmod -R 1777 /tmp/.ICE-unix &&
    export DISPLAY=:0 &&
    export PULSE_SERVER=127.0.0.1 &&
    export XDG_SESSION_TYPE=x11 &&
    sudo rm -rf /run/dbus &&
    sudo mkdir -p /run/dbus &&
    sudo dbus-daemon --system --fork &&
    dbus-launch --exit-with-session startxfce4 &
    exec /bin/sh
"
'
```

> **Note:** Replace `/data/adb/ksu/bin/busybox` with your busybox path.
> `module-aaudio-sink` requires a Termux pulseaudio build with AAudio support.

## What's included

### Desktop

| Package | Description |
|---------|-------------|
| xfce4 | XFCE4 desktop environment (with Sapphire theme) |
| xfce4-terminal | Terminal emulator |
| mousepad | Text editor |
| menulibre | Menu editor |
| thunar-archive-plugin | Archive support in Thunar file manager |

### System

| Package | Purpose |
|---------|---------|
| dbus, dbus-daemon | D-Bus message bus (system + session) |
| consolekit | Session tracking (no systemd) |
| mesa-utils | GPU diagnostics (`glxinfo`) |

### Fonts

| Package | Description |
|---------|-------------|
| fonts-noto | Noto base fonts |
| fonts-noto-cjk | Noto CJK (Chinese/Japanese/Korean) |
| fonts-noto-cjk-extra | Extra CJK variants |
| JetBrainsMono Nerd Fonts | 96 ttf files, 4 variants × 8 weights + italics |

### Locale

- `en_US.UTF-8` (default)
- `zh_CN.UTF-8`
- CJK font priority: SC variant preferred over JP for `en` / `und` locale

### Base utilities

`coreutils bash sudo util-linux findutils grep sed gawk diffutils iproute2
iputils-ping procps psmisc lsof curl wget git nano ca-certificates gnupg`

## Build-time fixes

The following are applied automatically during rootfs construction:

| Issue | Fix |
|------|-----|
| `su` takes 25 seconds | Remove `elogind`, delete `pam_elogind.so` from PAM |
| xfwm4 compositor stalls on llvmpipe | `use_compositing=false` in `/etc/skel/` |
| `systemctl` timeout in chroot | notifyd autostart bypasses `systemctl --user` |
| DNS timeout (no systemd-resolved) | `/etc/resolv.conf` → `8.8.8.8` |
| Timezone defaults to UTC | Configurable via workflow input (default: `Asia/Shanghai`) |
| `colord` / `xiccd` / `accountsservice` timeouts | Removed (unused in chroot) |

## Build your own

Trigger the workflow from **Actions** tab → **Devuan ARM64 Rootfs** → **Run workflow**.

Optionally set your timezone before running. The rootfs tarball will appear in
[Releases](https://github.com/DdogezD/Devuan-rootfs-for-ARM64/releases) when
the build completes.

## License

The workflow and configuration in this repository are provided as-is. The
rootfs contains packages from Devuan GNU/Linux under their respective licenses.
