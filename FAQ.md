<div align="center">

## **[ 🏠 Home ](./README.md)** | **[ 💾 Script Install ](./INSTALL.md)** | **[ ⚙️ Virtualization Setup Guide ](./Virtualization-Setup.md)** | **[ 🐛 Troubleshooting ](./FAQ.md)**

</div>


# 🛠️ Troubleshooting Guide

Setting up dynamic GPU passthrough and Looking Glass is one of the most complex things you can do on a Linux desktop. If things break, don't panic—it is almost a rite of passage. 

Below are the most common issues you might encounter and exactly how to fix them.

---

## 1. GPU Switching & Freezing Issues

### Symptom: The system completely freezes when the VM shuts down (or when running `gpu-toggle nvidia`).
**The Reality:** You have encountered the dreaded "Zombie GPU." This happens when the GPU enters a deep sleep state (`D3cold`) while it has no driver bound, causing the PCIe bus to lock up when the host tries to wake it.
**The Fix:**
1. Check if the script's power management rules applied correctly by running `sudo gpu-toggle status`. Look at the `d3cold_allowed` value—it MUST be `0`.
2. Ensure you actually ran `sudo gpu-toggle setup` and rebooted.
3. Check your BIOS/UEFI settings. Disable **ASPM (Active State Power Management)** or set PCIe power management to "Maximum Performance."

### Symptom: The script aborts with "ERROR: Some processes are using NVIDIA/DRM/ALSA nodes."
**The Reality:** The script's safety mechanism worked. If it unbinds the driver while an app is using it, your host will likely kernel panic.
**The Fix:** The script output will list the PIDs (Process IDs) blocking the switch. 
1. Close common culprits: **Steam**, **Discord**, **OBS Studio**, **Lutris**, or **Web Browsers** (WebGL keeps the GPU awake).
2. If you are using KDE/GNOME, ensure your display manager (e.g., KWin or Mutter) is not accidentally rendering on the secondary GPU.

### Symptom: The GPU binds to NVIDIA, but games on the host run at 5 FPS.
**The Reality:** The GPU successfully bound to the host, but the PCIe link speed degraded during the transition and failed to negotiate back to PCIe Gen 4 x16.
**The Fix:** Run `sudo gpu-toggle status` and check the `link` speed. If it shows something like `2.5GT/s PCIe x1` instead of `16.0GT/s PCIe x16`, you need a deep reset. 
Run the following to force a PCIe bus rescan:
```bash
echo 1 | sudo tee /sys/bus/pci/devices/0000\:01\:00.0/reset
echo 1 | sudo tee /sys/bus/pci/rescan
```

---

## 2. Looking Glass Issues

### Symptom: Looking Glass launches but stays on a black screen, or crashes instantly.
**The Reality:** The most common cause is a version mismatch between the Windows Host app and the Linux Client app, or incorrect KVMFR permissions.
**The Fix:**
1. **Version Match:** Check that your compiled Linux client (e.g., `B6`) perfectly matches the Windows `.exe` you installed. A mismatch of even a minor commit will cause a black screen.
2. **Permissions:** Run `ls -l /dev/kvmfr0`. It should be owned by `root:kvm` with `crw-rw----` permissions. If not, your Udev rules (Phase 4 of the install guide) didn't apply.
3. **Resolution:** Ensure the Windows VM is actually outputting a display. If you aren't using a physical dummy plug, ensure the Virtual Display Driver is installed and active in Windows.

### Symptom: No audio coming from the Windows VM through Looking Glass.
**The Reality:** QEMU does not have permission to talk to your Linux user's Pipewire/PulseAudio server.
**The Fix:**
1. Verify `/etc/libvirt/qemu.conf` has `user = "your_username"` configured (not `root`).
2. Verify the `QEMU_PA_SERVER` variable in your VM XML matches your user ID exactly. Run `id -u` in your terminal. If it outputs `1000`, the path in your XML MUST be `/run/user/1000/pulse/native`.

---

## 3. Virtual Machine & Memory Errors

### Symptom: VM fails to start with "Cannot allocate memory" or "Failed to allocate hugepages".
**The Reality:** Over time, your host's RAM becomes fragmented. Even if you have 32GB of *free* RAM, you might not have 32GB of *contiguous* RAM required for hugepages.
**The Fix:**
1. The libvirt hook tries to drop caches automatically, but if your system has been running heavily for days, it might fail.
2. **Manual flush:**
   ```bash
   sudo sync
   echo 3 | sudo tee /proc/sys/vm/drop_caches
   echo 1 | sudo tee /proc/sys/vm/compact_memory
   ```
3. If it still fails, you simply need to reboot the host to defragment the memory, or reduce the amount of RAM assigned to the VM.

### Symptom: Windows is stuttering or audio is crackling.
**The Reality:** The VM CPU is competing with host processes, or DPC latencies in Windows are spiking.
**The Fix:**
1. **CPU Pinning:** Double-check your XML `<cputune>` block. Do not allocate *all* your host cores to the VM. Always leave at least 2 cores (4 threads) dedicated strictly to the host.
2. **Hyper-V Timers:** Ensure `<timer name="hypervclock" present="yes"/>` is active in your XML.
3. **Message Signaled Interrupts (MSI):** Inside Windows, use the "MSI Utility V3" to set your NVIDIA GPU and Audio controllers to MSI mode (check the box) and set priority to "High".

---

## 4. Hardware & IOMMU Grouping

### Symptom: "Please ensure all devices within the iommu_group are bound to their vfio bus driver."
**The Reality:** Your motherboard grouped your secondary GPU with another device (like an NVMe drive or a USB controller) on the same PCIe lane. You cannot pass through just one device in a group; you must pass them all.
**The Fix:**
1. **Hardware fix (Best):** Move the GPU to a different PCIe slot on your motherboard.
2. **Software fix (Advanced):** Use the ACS Override Patch kernel parameter (`pcie_acs_override=downstream,multifunction`). *Warning: This breaks security isolation and can cause host instability if you split devices that share physical hardware paths.*
