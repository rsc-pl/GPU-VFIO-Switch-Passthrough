# 🔄 Stable VFIO/NVIDIA Switcher

A robust, state-aware Bash script designed to dynamically switch an NVIDIA GPU between the Linux host (`nvidia`) and a Virtual Machine (`vfio-pci`) without rebooting. 

This project was built specifically to solve the dreaded **"Zombie GPU"** or "Code 43" issues that often occur during dynamic driver binding on modern NVIDIA cards (like the RTX 4000 series). It features strict power management overrides and active pre-flight safety checks.

## 🎯 What is it for?

If you are running a Single-GPU Passthrough setup or a Dual-GPU system where you want to reclaim your secondary GPU for host tasks (like AI rendering, gaming, or encoding) when your VM is off, standard unbind commands often crash the system or leave the GPU in an unrecoverable sleep state. 

This script safely handles:
1. **Pre-flight Blocker Checks:** Ensures no host applications (like X11/Wayland, Steam, OBS, or WebGL browsers) are currently using the GPU before attempting an unbind. If something is using the GPU, it will safely abort and tell you exactly which PID to close.
2. **Strict Power Management:** Prevents the GPU from entering `D3cold` sleep states during the driver handoff, which is the primary cause of system lockups.
3. **Deep PCIe Resetting:** Safely removes the device from the PCI bus and rescans it if the standard driver unbind fails.

---

## 🛠️ Installation & Setup

Because Linux distributions handle early-boot module loading (initramfs) differently, setup instructions vary depending on your OS. 

**Please read the [Installation & Setup Guide (INSTALL.md)](INSTALL.md)** for detailed instructions on:
1. Finding your specific hardware IDs.
2. Installing prerequisites.
3. Setting up the script for Arch/CachyOS, Debian/Ubuntu, or Fedora.
4. Configuring passwordless execution via `sudoers`.

---

## 🚀 Usage

Once installed and configured, the script accepts four main commands. (Note: These assume you have set up a `sudoers` exception as detailed in the install guide).

### Switch GPU to the VM (`vfio`)
```bash
sudo gpu-toggle vfio
```
Run this before starting your VM. It will:
* Check for processes blocking the GPU (e.g., Discord hardware acceleration).
* Safely unload the NVIDIA driver stack.
* Bind the GPU to `vfio-pci`.

### Switch GPU back to the Host (`nvidia`)
```bash
sudo gpu-toggle nvidia
```
Run this after shutting down your VM. It will:
* Temporarily disable the VFIO boot blacklist.
* Unbind the GPU from VFIO.
* Reload the `nvidia`, `nvidia_uvm`, `nvidia_modeset`, and `nvidia_drm` modules.
* Bind the GPU back to the host, making it available for Linux applications.

### Check Status (`status`)
```bash
sudo gpu-toggle status
```
Prints out the current PCIe power state, active drivers, link speed, and whether the boot blacklist is currently active or disabled.

### Run First-Time Setup (`setup`)
```bash
sudo gpu-toggle setup
```
*(Only run during initial installation. Generates the necessary config files and rebuilds the initramfs).*

---

## 🧠 How It Works (Under the Hood)

This script goes far beyond basic `modprobe` commands. Here is a detailed breakdown of what it is actively doing to protect your system stability:

### 1. Blocker Detection (`fuser`)
Before unbinding the `nvidia` driver, the script dynamically maps out all DRM (`/dev/dri/renderD*`) and ALSA sound (`/dev/snd/*`) nodes associated with your specific PCI bus. It then uses `fuser` to check if any active PIDs are touching those files. If an app is rendering on the GPU and you yank the driver, the Linux kernel will often kernel panic. This script catches that *before* it happens and tells you exactly which apps to close.

### 2. Defeating the "Zombie GPU" (Strict PM Policies)
Modern GPUs utilize advanced PCIe power management (like `D3cold`). If a GPU enters `D3cold` while it has no driver bound, the motherboard effectively cuts power to that PCIe slot. When you try to bind a new driver, the GPU is unresponsive, locking up the bus and requiring a hard reboot. 
To fix this, the script:
* Writes `on` to `/sys/bus/pci/devices/.../power/control`.
* Writes `0` to `d3cold_allowed`.
* It does this for the Video node, the Audio node, **and the parent PCIe bridge** to ensure the entire lane stays awake during the transition.

### 3. Dynamic Blacklisting State
To ensure your VM can claim the GPU on boot without race conditions, the script writes a strict module blacklist (`99-vfio-blacklist-nvidia.conf`). However, to give the GPU back to the host, this blacklist must be bypassed. When switching to `nvidia`, the script renames this config file to `.disabled_by_gpu_toggle`. When switching back to `vfio`, it restores it. It even uses a `trap` to guarantee the blacklist is restored if the script fails midway, saving you from a broken boot state.

### 4. The Deep-Reset Fallback Path
Sometimes, a VM shuts down poorly and leaves the VFIO device in a dirty state. If the script fails to bind the NVIDIA driver gracefully, it automatically engages a "deep reset" path. This involves:
* Sending a standard FLR (Function Level Reset) via `/sys/.../reset`.
* Resetting the parent PCIe bus.
* Echoing `1` to the PCIe `remove` node (virtually unplugging the GPU from the motherboard).
* Echoing `1` to the PCI `rescan` node to force the kernel to rediscover the hardware fresh before probing the drivers.

### 5. Safe Global Locking
The script uses `flock` on `/run/lock/gpu-toggle.lock`. This ensures that if you accidentally trigger the script twice, or if a libvirt QEMU hook tries to run it while you are running it manually, the second instance will safely abort instead of causing a race condition that crashes the system.
