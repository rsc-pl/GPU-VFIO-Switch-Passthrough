---
layout: page
title: "Installation"
permalink: /installation/
---

This document covers how to install and configure the script across different Linux distributions. Ensure you choose the correct version of the script for your OS before proceeding.

## Step 1: Find Your GPU IDs (All Systems)

Before installing, you must identify your specific NVIDIA hardware IDs and PCI bus addresses.

Run the following command:
```bash
lspci -nn | grep -i nvidia
```

**Example Output:**
> `01:00.0` VGA compatible controller [0300]: NVIDIA Corporation AD102 [GeForce RTX 4090] [`10de`:`2684`] (rev a1)
> `01:00.1` Audio device [0403]: NVIDIA Corporation AD102 High Definition Audio Controller [`10de`:`22ba`] (rev a1)

Open your chosen script version in a text editor and update the top section to match your results:
```bash
NVIDIA_VENDOR_ID="10de"
NVIDIA_DEVICE_VIDEO_ID="2684"      # Update this
NVIDIA_DEVICE_AUDIO_ID="22ba"      # Update this
GPU_VIDEO_BUS_ID="0000:01:00.0"    # Update this (ensure 0000: prefix)
GPU_AUDIO_BUS_ID="0000:01:00.1"    # Update this
```

---

## Step 2: Install Prerequisites & Run Setup

The required packages and setup commands differ slightly depending on your distribution. I created the script primarily with Arch and Arch-based distributions in mind, since that’s what I use myself, which is why it has been tested most thoroughly in that environment.

### 🔵 Arch Linux / CachyOS / Manjaro
**Initramfs tool:** `mkinitcpio`

1. **Install required packages:**
   ```bash
   sudo pacman -S pciutils psmisc
   ```
2. **Use the Arch script version** (`gpu-toggle-arch.sh`).
3. **Run the setup command:**
   ```bash
   sudo ./gpu-toggle-arch.sh setup
   ```
4. **Reboot** to apply boot-time configuration.

### 🟠 Debian / Ubuntu / Pop!_OS
**Initramfs tool:** `initramfs-tools`

1. **Install required packages:**
   ```bash
   sudo apt update
   sudo apt install pciutils psmisc
   ```
2. **Use the Debian script version** (`gpu-toggle-debian.sh`).
3. **Run the setup command:**
   ```bash
   sudo ./gpu-toggle-debian.sh setup
   ```
4. **Reboot** to apply boot-time configuration.

### 🔴 Fedora / Nobara / RHEL
**Initramfs tool:** `dracut`

1. **Install required packages:**
   ```bash
   sudo dnf install pciutils psmisc
   ```
2. **Use the Fedora script version** (`gpu-toggle-fedora.sh`).
3. **Run the setup command:**
   ```bash
   sudo ./gpu-toggle-fedora.sh setup
   ```
4. **Reboot** to apply boot-time configuration.

---

## Step 3: System-Wide Installation (All Systems)

Once you have configured and tested the script, move it to your local binary path so you can call it from anywhere.

1. **Copy the script and rename it:**
   ```bash
   # Replace 'gpu-toggle-YOURDISTRO.sh' with the script you are using
   sudo cp gpu-toggle-YOURDISTRO.sh /usr/local/bin/gpu-toggle
   sudo chmod +x /usr/local/bin/gpu-toggle
   ```

2. **Run without a password (Optional but recommended):**
   To allow your user to seamlessly switch GPUs without typing a password every time, add a `sudoers` exception.

   ```bash
   sudo visudo
   ```
   Scroll to the very bottom of the file and add the following line. **Replace `your_username` with your actual Linux user account name!**
   ```text
   your_username ALL=(root) NOPASSWD: /usr/local/bin/gpu-toggle
   ```
   Save and exit `visudo`.

---

## Testing the Script

You can now use the script system-wide:

* **Check current status:**
  ```bash
  sudo gpu-toggle status
  ```
* **Switch GPU to VM (VFIO):**
  ```bash
  sudo gpu-toggle vfio
  ```
* **Switch GPU to Host (NVIDIA):**
  ```bash
  sudo gpu-toggle nvidia
  ```
