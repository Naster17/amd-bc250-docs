# Alpine Linux Setup Guide

<img src="https://cdn.simpleicons.org/alpinelinux" alt="Alpine Linux" width="48"/>

Alpine Linux works on the BC-250, but it is a manual setup aimed at advanced users. Based on real community install history, Alpine is a good fit for lean server builds, custom desktop setups, and users who prefer OpenRC over systemd.

**Status:** Working with manual setup  
**Difficulty:** Advanced  
**Init System:** OpenRC  
**Best For:** Minimal systems, server workloads, custom builds, and low resource usage

---

## Why Choose Alpine?

### Advantages

- Very small base system
- Low RAM and power usage (150mb & 35W)
- OpenRC instead of systemd
- Good fit for server or appliance-style deployments
- Easy to keep lean if you only install what you need

### Considerations

- Much less BC-250 testing than Fedora, Bazzite, Arch, or Debian
- Graphics and Vulkan setup is manual
- Governor installation is manual
- Some packages and workflows differ across Alpine branches
- Not the easiest choice for gaming-focused installs

!!! info "Best Use Case"
    Alpine is a strong choice if you want a stripped-down BC-250 system, especially for GPU/CPU power server use or custom environments where low overhead matters more than convenience.

---

## BIOS Requirements

Before installing Alpine, ensure BIOS is configured:

1. Flash modified BIOS (P3.00 or later recommended)
2. Set VRAM allocation to 512MB dynamic (recommended for shared VRAM / UMA, see [VRAM Configuration](../bios/vram.md##option-1-512mb-dynamic))

See [BIOS Flashing Guide](../bios/flashing.md).

---

## Installation Overview

### Prerequisites

- Alpine installation media
- Ethernet connection recommended
- Passive DP-to-HDMI adapter if needed
- Comfort with manual post-install configuration

### Base Installation

1. Boot the Alpine installer
2. If the display does not initialize properly, boot once with `nomodeset`
3. Run the normal Alpine install process with `setup-alpine`
4. Install to disk as usual
5. Reboot into the installed system

!!! warning "Remove nomodeset After Setup"
    If you used `nomodeset` during installation, remove it after the graphics stack is working. Leaving it enabled will prevent normal GPU acceleration.

---

## Post-Installation Setup

### 1. Update the Base System

```bash
sudo apk update
sudo apk upgrade
```

---

### 2. Install a Working Kernel

Community history shows Alpine working with `linux-stable`. For kernel recommendations, see [Kernel Configuration](kernel.md):

```bash
sudo apk add linux-stable
```

Reboot after kernel installation:

```bash
sudo reboot
```

After reboot, confirm the active kernel:

```bash
uname -a
```

!!! warning "Kernel Selection Still Matters"
    As with other distros, avoid known-broken BC-250 kernel ranges such as 6.15.0-6.15.6 and 6.17.8-6.17.10. Prefer confirmed working releases such as 6.18.18 LTS, 6.19.x stable, or 6.17.11+ where available.

---

### 3. Install Firmware and Graphics Packages

Setup drivers and firmware:

```bash
sudo apk add linux-firmware-amdgpu # base amdgpu drivers
sudo apk add mesa mesa-gl mesa-glx mesa-dri-gallium # mesa drivers
sudo apk add mesa-vulkan-ati vulkan-loader vulkan-tools # vulkaninfo ...
sudo apk add mesa-demos # glxinfo ...
```

If you need extra Vulkan development tools later:

```bash
sudo apk add vulkan-loader-dev glslang-dev spirv-headers shaderc cmake
```

---

### 4. Configure GRUB and Rebuild Boot Files

If you added `nomodeset` during installation, remove it from GRUB after Mesa and Vulkan are working:

```bash
sudo nano /etc/default/grub
```

See [Environment Variables](../drivers/environment.md#mitigationsoff) for details on `mitigations=off`.
Use a performance-oriented default line such as:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet mitigations=off"
```

Then rebuild boot files:

```bash
sudo mkinitfs
sudo update-grub
sudo reboot
```

---

### 5. Verify AMDGPU and Vulkan

After reboot, verify that firmware is present, the kernel driver loaded correctly, and the system is using the AMD GPU instead of software rendering:

```bash
lsmod | grep amdgpu
# Should show: amdgpu

dmesg | grep -i amdgpu
# Use this if you need the full amdgpu log for troubleshooting

vulkaninfo --summary
# Should list the AMD RADV Vulkan driver and GPU0
```

---

### 6. Install the GPU Governor

The BC-250 still benefits heavily from a governor on Alpine. Your history shows a working manual setup using the SMU branch.

Install build dependencies:

```bash
sudo apk add git rust cargo libdrm-dev dbus
```

Enable D-Bus:

```bash
sudo rc-service dbus start
sudo rc-update add dbus default
```

Clone and build the governor:

```bash
git clone --branch smu https://github.com/filippor/cyan-skillfish-governor.git cyan-skillfish-governor-smu
cd cyan-skillfish-governor-smu
cargo build --release
```

Install the binary and config:

```bash
sudo install -Dm755 target/release/cyan-skillfish-governor-smu /usr/local/bin/cyan-skillfish-governor-smu
sudo mkdir -p /etc/cyan-skillfish-governor-smu
sudo cp default-config.toml /etc/cyan-skillfish-governor-smu/config.toml
```

Test it manually first:

```bash
sudo cyan-skillfish-governor-smu --verbose /etc/cyan-skillfish-governor-smu/config.toml
```

---

### 7. Create an OpenRC Service for the Governor

Create `/etc/init.d/cyan-skillfish-governor-smu`:

```bash
#!/sbin/openrc-run
name="cyan-skillfish-governor-smu"
description="GPU governor for AMD Cyan Skillfish APU"
command="/usr/local/bin/cyan-skillfish-governor-smu"
command_args="/etc/cyan-skillfish-governor-smu/config.toml"
command_background=true
pidfile="/run/${RC_SVCNAME}.pid"
output_log="/var/log/cyan-skillfish-governor-smu.log"
error_log="/var/log/cyan-skillfish-governor-smu.log"
```

Then enable it:

```bash
sudo chmod +x /etc/init.d/cyan-skillfish-governor-smu
sudo rc-update add cyan-skillfish-governor-smu default
sudo rc-service cyan-skillfish-governor-smu start
sudo rc-service cyan-skillfish-governor-smu status
```

If you edit the config later:

```bash
sudo rc-service cyan-skillfish-governor-smu restart
```

---

## Troubleshooting

### Governor Does Not Start
**Symptoms:**  
- Bad GPU performance  
- When start/restart `start-stop-daemon: no matching processes found`  

**Solution:**  
- `dbus` is installed and enabled  
- `libdrm-dev` was present during build  
- The binary is installed in `/usr/local/bin/`  
- The config file exists at `/etc/cyan-skillfish-governor-smu/config.toml`  
- Make sure that higher frequencies do not use less mV  

```bash
sudo rc-service cyan-skillfish-governor-smu status
sudo /usr/local/bin/cyan-skillfish-governor-smu --verbose /etc/cyan-skillfish-governor-smu/config.toml # to debug
```

## Community Resources

- **Alpine Linux:** [alpinelinux.org](https://alpinelinux.org/)
- **Alpine Wiki:** [wiki.alpinelinux.org](https://wiki.alpinelinux.org/)
- **GPU Governor SMU Branch:** [cyan-skillfish-governor-smu](https://github.com/filippor/cyan-skillfish-governor/tree/smu)

---

## Related Guides

- [Debian Setup](debian.md)
- [Arch Linux Setup](arch.md)
- [Fedora Setup](fedora.md)
- [GPU Governor](../system/governor.md)
