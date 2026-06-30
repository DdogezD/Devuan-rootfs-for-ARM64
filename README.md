# Devuan-rootfs-for-ARM64

Prebuilt Devuan GNU/Linux 6 (excalibur) ARM64 rootfs for use as a Termux
chroot with XFCE4 desktop.

## Quick start

### Prerequisites (Termux)

```sh
pkg update
pkg install x11-repo root-repo tur-repo
pkg update
pkg install tsu termux-x11-nightly pulseaudio
```

### Download

Get the latest `devuan.tar.gz` from
[Releases](https://github.com/DdogezD/Devuan-rootfs-for-ARM64/releases).

### Extract

```sh
tsu
mkdir -p /data/local/tmp/devuan
cd /data/local/tmp/devuan
wget <release-url>/devuan.tar.gz
tar xpvf devuan.tar.gz --numeric-owner
mkdir sdcard
```

### Startup script

Place `termux-login.sh` in `/sdcard/` and `chmod +x` it:

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
    sudo chown root:root /tmp/.ICE-unix 2>/dev/null || true &&
    sudo chmod -R 1777 /tmp/.ICE-unix &&
    export DISPLAY=:0 &&
    export PULSE_SERVER=127.0.0.1 &&
    export XDG_SESSION_TYPE=x11 &&
    sudo rm -rf /run/dbus &&
    sudo mkdir -p /run/dbus &&
    sudo dbus-daemon --system --fork &&
    sudo mount -t binfmt_misc binfmt /proc/sys/fs/binfmt_misc 2>/dev/null || true &&
    echo ':x86_64:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x3e\x00:\xff\xff\xff\xff\xff\xfe\xfe\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/local/bin/box64:' | sudo tee /proc/sys/fs/binfmt_misc/register > /dev/null 2>&1 || true &&
    echo ':x86:M::\x7fELF\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x03\x00:\xff\xff\xff\xff\xff\xfe\xfe\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/local/bin/box86:' | sudo tee /proc/sys/fs/binfmt_misc/register > /dev/null 2>&1 || true &&
    dbus-launch --exit-with-session startxfce4 &
    exec /bin/sh
"
'
```

> **Notes:**
> - Replace `/data/adb/ksu/bin/busybox` with your busybox path.
> - `module-aaudio-sink` requires a Termux pulseaudio build with AAudio support. If unavailable, remove the line — pulseaudio will fall back to a null sink (no sound).
> - Binfmt auto-dispatch for x86/x86_64 requires `CONFIG_BINFMT_MISC=y` in kernel. If your kernel lacks it, run x86 binaries with explicit `box64` / `box86` prefix instead.

## First-time setup

After extracting the rootfs, chroot in as root and run these steps once:

```sh
# From Termux (as root):
/data/adb/ksu/bin/busybox chroot /data/local/tmp/devuan /bin/sh
```

### Hosts & groups

```sh
echo "127.0.0.1 localhost" > /etc/hosts

groupadd -g 3003 aid_inet
groupadd -g 3004 aid_net_raw
groupadd -g 1003 aid_graphics
usermod -g 3003 -G 3003,3004 -a _apt
usermod -G 3003 -a root
```

### Create user

```sh
groupadd storage
groupadd wheel
useradd -m -g users -G wheel,audio,video,storage,aid_inet -s /bin/bash YOUR_USER
passwd -d YOUR_USER
echo "%wheel ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/99-wheel
chmod 440 /etc/sudoers.d/99-wheel
```

### Box64 / Box86

```sh
# Add repos (modern signed-by format)
wget -qO- https://ryanfortner.github.io/box64-debs/KEY.gpg \
  | gpg --dearmor \
  | tee /usr/share/keyrings/box64-archive-keyring.gpg > /dev/null
echo 'deb [signed-by=/usr/share/keyrings/box64-archive-keyring.gpg] https://ryanfortner.github.io/box64-debs/debian ./' \
  | tee /etc/apt/sources.list.d/box64.list

wget -qO- https://ryanfortner.github.io/box86-debs/KEY.gpg \
  | gpg --dearmor \
  | tee /usr/share/keyrings/box86-archive-keyring.gpg > /dev/null
echo 'deb [signed-by=/usr/share/keyrings/box86-archive-keyring.gpg] https://ryanfortner.github.io/box86-debs/debian ./' \
  | tee /etc/apt/sources.list.d/box86.list

# Box86 needs armhf multiarch
dpkg --add-architecture armhf
apt update
apt install -y box64-android box86-android libc6:armhf
```

> `CONFIG_BINFMT_MISC=y` is required for transparent auto-dispatch. The startup
> script handles the mount + registration automatically.

### User environment

Append to `~/.profile`:

```sh
cat >> ~/.profile << 'EOF'

export LC_ALL=zh_CN.UTF-8
export LANG=zh_CN.UTF-8
export SAL_FORCEDPI=175
export WINEDPI=175
EOF
```

## What's included

### Desktop

| Package | Description |
|---------|-------------|
| xfce4 | XFCE4 desktop environment |
| xfce4-terminal | Terminal emulator |
| mousepad | Text editor |
| menulibre | Menu editor |
| thunar-archive-plugin | Archive support in Thunar file manager |

### System

| Package | Purpose |
|---------|---------|
| dbus, dbus-daemon | D-Bus message bus (system + session) |
| consolekit | Session tracking (D-Bus service disabled — no VT in chroot) |
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
- CJK font priority: SC preferred over JP for `en` / `und` locale

### Base utilities

`coreutils bash sudo util-linux findutils grep sed gawk diffutils iproute2
iputils-ping procps psmisc lsof curl wget git nano ca-certificates gnupg locales`

## GPU acceleration (Mesa Turnip)

The desktop uses `llvmpipe` software rendering to avoid crashes on Adreno GPU.
Per-app hardware acceleration is provided via Mesa Turnip at `/opt/mesa-turnip/`.

### Install

Download from [lfdevs/mesa-for-android-container](https://github.com/lfdevs/mesa-for-android-container/releases)
and extract:

```sh
mkdir -p /opt/mesa-turnip
curl -fsSL <turnip-release-url> | tar xz --strip-components=1 -C /opt/mesa-turnip/
```

### Usage

Use the `gpu-run` wrapper to launch individual apps with GPU acceleration.
The desktop environment stays on software rendering — safe, no crashes.

```sh
# Install the wrapper
sudo tee /usr/local/bin/gpu-run << 'EOF' > /dev/null
#!/bin/sh
TURNIP_PATH="/opt/mesa-turnip/usr/lib/aarch64-linux-gnu"
export VK_ICD_FILENAMES="/opt/mesa-turnip/usr/share/vulkan/icd.d/turnip_icd.json"
export LD_LIBRARY_PATH="${TURNIP_PATH}:${LD_LIBRARY_PATH}"
export LIBGL_DRIVERS_PATH="${TURNIP_PATH}/dri"
export GBM_BACKENDS_PATH="${TURNIP_PATH}/gbm"
exec "$@"
EOF
sudo chmod +x /usr/local/bin/gpu-run

# Create Turnip ICD config
sudo tee /opt/mesa-turnip/usr/share/vulkan/icd.d/turnip_icd.json << 'EOF' > /dev/null
{
    "ICD": {
        "api_version": "1.4.354",
        "library_arch": "64",
        "library_path": "/opt/mesa-turnip/usr/lib/aarch64-linux-gnu/libvulkan_freedreno.so"
    },
    "file_format_version": "1.0.1"
}
EOF

# Launch any app with GPU:
gpu-run glmark2
gpu-run chromium
gpu-run <your-3d-app>
```

> Desktop must NOT use GPU globally — xfwm4 + llvmpipe compositing
> will crash if `MESA_LOADER_DRIVER_OVERRIDE` is set system-wide.

## Build-time fixes

Applied automatically during rootfs construction:

| Issue | Fix |
|------|-----|
| `su` 25s timeout | Remove `elogind` + `pam_elogind.so` from PAM |
| `systemctl` timeout | notifyd bypasses `systemctl --user` |
| DNS timeout (no systemd-resolved) | `/etc/resolv.conf` → `8.8.8.8` |
| Timezone defaults to UTC | Configurable (default: `Asia/Shanghai`) |
| colord / xiccd / accountsservice | Removed |
| ConsoleKit crashes (no VT in chroot) | D-Bus service disabled |

## Build your own

Trigger the workflow from **Actions** tab → **Devuan ARM64 Rootfs** → **Run workflow**.

Optionally set your timezone before running. The rootfs tarball will appear in
[Releases](https://github.com/DdogezD/Devuan-rootfs-for-ARM64/releases) when
the build completes.

## License

The workflow and configuration in this repository are provided as-is. The
rootfs contains packages from Devuan GNU/Linux under their respective licenses.
