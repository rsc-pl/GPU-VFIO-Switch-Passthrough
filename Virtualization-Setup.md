Setting up a Dual-GPU passthrough (Host GPU + Work/Gaming GPU) with Looking Glass is one of the most advanced and rewarding projects you can do on Linux. It can be tricky, but this guide breaks it down step-by-step from a completely fresh install. 

This guide assumes you are using your primary GPU (e.g., AMD Radeon W7500 Pro or an integrated GPU) for the Linux host, and passing through a secondary GPU (e.g., NVIDIA RTX 4090) to the virtual machine.

---

## 🛠️ Phase 1: Package Installation

The required virtualization packages, networking tools, and Looking Glass dependencies vary depending on your Linux distribution. Choose the tab/section that matches your OS.

### 🔵 Arch Linux / CachyOS
**1. Install core virtualization and networking tools:**
```bash
sudo pacman -Syu
sudo pacman -S --needed \
    base-devel git wget curl nano dialog \
    dkms linux-headers \
    qemu-full libvirt virt-manager virt-viewer edk2-ovmf \
    libguestfs dnsmasq bridge-utils openbsd-netcat swtpm
```
*(Note for CachyOS: You may need `linux-cachyos-headers` instead of `linux-headers`).*

**2. Install Looking Glass & Dependencies (via AUR):**
```bash
paru -S --needed looking-glass looking-glass-module-dkms
```

### 🟠 Debian / Ubuntu / Pop!_OS
**1. Install core virtualization and networking tools:**
```bash
sudo apt update
sudo apt install \
    build-essential git wget curl nano \
    dkms linux-headers-generic \
    qemu-kvm qemu-utils libvirt-daemon-system libvirt-clients virt-manager ovmf \
    libguestfs-tools dnsmasq bridge-utils netcat-openbsd swtpm
```
**2. Install Looking Glass Dependencies:**
Debian-based systems usually require compiling Looking Glass from source. Install the build prerequisites:
```bash
sudo apt install cmake pkg-config libgl1-mesa-dev libegl1-mesa-dev libfontconfig1-dev libspice-protocol-dev nettle-dev libx11-dev libxext-dev libxi-dev libxinerama-dev libxcursor-dev libxpresent-dev libxss-dev libxkbcommon-dev libwayland-dev wayland-protocols libpipewire-0.3-dev libpulse-dev libsamplerate0-dev
```
*(You will need to pull the Looking Glass source from GitHub and run `cmake` & `make` to build the client and the KVMFR kernel module).*

### 🔴 Fedora / Nobara / RHEL
**1. Install core virtualization and networking tools:**
```bash
sudo dnf upgrade
sudo dnf install \
    @virtualization \
    kernel-devel kernel-headers dkms \
    bridge-utils nc swtpm nano git wget curl
```
**2. Install Looking Glass Dependencies:**
```bash
sudo dnf install cmake gcc-c++ pkgconf-pkg-config libX11-devel libXext-devel libXi-devel libXinerama-devel libXcursor-devel libXpresent-devel libXScrnSaver-devel libxkbcommon-devel wayland-devel wayland-protocols-devel mesa-libGL-devel mesa-libEGL-devel fontconfig-devel spice-protocol nettle-devel pipewire-devel pulseaudio-libs-devel libsamplerate-devel
```
*(Like Debian, you will compile Looking Glass and the KVMFR module from source).*

> **Optional (OpenSnitch Firewall):** If you use OpenSnitch (as per your CachyOS notes), install it and enable the service: `sudo systemctl enable --now opensnitchd`.

---

## 👤 Phase 2: User Permissions & Services

We need to ensure your Linux user has the right to manage virtual machines and access the input devices for Looking Glass (Spice).

**1. Add your user to the necessary groups:**
```bash
sudo usermod -aG libvirt,kvm,input $(whoami)
```

**2. Enable and start the Libvirt services:**
```bash
sudo systemctl enable --now libvirtd.service
sudo systemctl enable --now virtlogd.socket
```

**3. Configure QEMU to run as your user:**
Open the QEMU configuration file:
```bash
sudo nano /etc/libvirt/qemu.conf
```
Find the `user` and `group` variables, uncomment them, and change them to your specific Linux username:
```text
user = "your_username"
group = "your_username"
```
Restart Libvirt to apply:
```bash
sudo systemctl restart libvirtd
```

---

## 🌐 Phase 3: Network Bridge Setup

A network bridge allows your VM to appear as a physical device on your local network, rather than hiding behind a NAT. This is crucial for seamless local networking and Looking Glass functionality.

**Warning:** The IP addresses below are examples. **You must adjust them** to match your router's subnet (e.g., `192.168.0.x` or `10.0.0.x`).

**1. Identify your ethernet interface:**
```bash
nmcli device
```
*(Look for your active ethernet connection, e.g., `enp16s0` or `eth0`).*

**2. Delete the default connection and create the bridge:**
Replace `enp16s0` with your actual interface name.
```bash
sudo nmcli connection delete "Wired connection 1"
sudo nmcli connection add type bridge ifname br0 con-name br0
sudo nmcli connection modify br0 bridge.stp no
sudo nmcli connection add type ethernet ifname enp16s0 master br0 con-name br0-slave
```

**3. Assign a Static IP (Adjust these IPs to your network!):**
```bash
sudo nmcli connection modify br0 ipv4.addresses "192.168.1.10/24"
sudo nmcli connection modify br0 ipv4.gateway "192.168.1.1"
sudo nmcli connection modify br0 ipv4.dns "1.1.1.1,8.8.8.8"
sudo nmcli connection modify br0 ipv4.method manual
```

**4. Bring the bridge online:**
```bash
sudo nmcli connection up br0
```

---

## 🪞 Phase 4: Looking Glass (KVMFR) Setup

Looking Glass uses a kernel module called `kvmfr` to create a shared memory space between the host and the VM, allowing for near-zero latency frame passing.

**1. Set the static memory size:**
For a 4K resolution setup, 128MB is recommended.
```bash
echo "options kvmfr static_size_mb=128" | sudo tee /etc/modprobe.d/kvmfr.conf
```

**2. Ensure the KVMFR module loads on boot:**
* **Arch/CachyOS:** Add `kvmfr` to your `MODULES` array in `/etc/mkinitcpio.conf`, then run `sudo mkinitcpio -P`.
* **Debian/Ubuntu:** Add `kvmfr` to `/etc/modules` and run `sudo update-initramfs -u`.
* **Fedora:** Add `add_drivers+=" kvmfr "` to a `.conf` file in `/etc/dracut.conf.d/` and run `sudo dracut -f`.

**3. Create Udev rules for permissions:**
This ensures the KVMFR device is created with the right permissions so QEMU and your user can read/write to it.
```bash
sudo nano /etc/udev/rules.d/99-kvmfr.rules
```
Paste the following:
```text
KERNEL=="kvmfr*", GROUP="kvm", MODE="0660"
```
Reload Udev rules to apply immediately:
```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

**4. Update QEMU Device ACLs:**
You must explicitly tell Libvirt/QEMU that it is allowed to touch the `/dev/kvmfr0` device.
Open `/etc/libvirt/qemu.conf`:
```bash
sudo nano /etc/libvirt/qemu.conf
```
Find `cgroup_device_acl`, uncomment the block, and add `/dev/kvmfr0` to the list:
```text
cgroup_device_acl = [
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm",
    "/dev/kvmfr0"
]
```
Restart Libvirt: `sudo systemctl restart libvirtd`

---

## 🖥️ Phase 5: Looking Glass Client Configuration

**1. Create the Client Config File:**
```bash
nano ~/.looking-glass-client.ini
```
Paste your preferred configuration:
```ini
[app]
shmFile=/dev/kvmfr0
renderer=EGL
allowDMA=yes

[win]
jitRender=yes
showFPS=yes
fpsMin=-1

[egl]
vsync=no
doubleBuffer=no
multisample=yes

[spice]
enable=yes
audio=yes
input=yes
clipboard=yes
```

**2. Create the KDE Wayland Startup Script:**
If you are using KDE Plasma on Wayland, standard `.desktop` files sometimes struggle with forcing fullscreen on a specific virtual desktop. This script uses `kdotool` to bypass those limitations.
```bash
mkdir -p ~/bin
nano ~/bin/lg-start.sh
```
Paste the following:
```bash
#!/bin/bash
export PATH="/usr/bin:/bin:$PATH"

# 1. Switch to Desktop 2
kdotool set_desktop 2
sleep 0.5

# 2. Launch Looking Glass
looking-glass-client -S no &

# 3. Wait for the window to appear
for i in {1..40}; do
    WID=$(kdotool search --class "looking-glass-client" 2>/dev/null | head -n 1)
    if [ -n "$WID" ]; then break; fi
    sleep 0.2
done
sleep 1.5

# 4. Force Fullscreen using kdotool DBus interface
if [ -n "$WID" ]; then
    kdotool windowactivate "$WID"
    kdotool windowstate "$WID" --add fullscreen
fi
```
Make it executable: `chmod +x ~/bin/lg-start.sh`

**3. Create the Desktop Shortcut:**
```bash 
nano ~/Desktop/LookingGlassClient.desktop
```
```ini
[Desktop Entry]
Type=Application
Name=Looking Glass (Client)
Comment=Starts LG on desktop 2 in fullscreen
Exec=/usr/bin/bash ~/bin/lg-start.sh
Icon=looking-glass
Terminal=false
StartupWMClass=looking-glass-client
Categories=System;Emulator;
```

---

## 🪝 Phase 6: UEFI Firmware & Libvirt Hooks

**1. Set up your UEFI/NVRAM files:**
When creating your VM or migrating XML, ensure your `OVMF_CODE.fd` paths match your OS:
* **Arch/CachyOS:** `/usr/share/edk2/x64/OVMF_CODE.fd`
* **Debian/Ubuntu:** `/usr/share/OVMF/OVMF_CODE.fd`
* **Fedora:** `/usr/share/edk2/ovmf/OVMF_CODE.fd`

*(Tip: You can easily select the correct UEFI binary from the dropdown list in Virt-Manager under **Overview -> Firmware**).*

**2. Prepare the Libvirt Hooks directory:**
If you have automated scripts (like CPU pinning scripts or audio routing scripts) that need to run when the VM starts, place them in the hooks directory.
```bash
sudo mkdir -p /etc/libvirt/hooks
```
*(Copy your scripts into this folder and ensure they are executable: `sudo chmod +x /etc/libvirt/hooks/*`).*

***

## ⚙️ Phase 7: Kernel Parameters (systemd-boot)

To ensure the Linux kernel properly isolates the GPU and groups IOMMU devices upon boot, you need to append specific kernel parameters. 

Since CachyOS uses **systemd-boot** (and often UKIs - Unified Kernel Images), you define these in the `cmdline` file.

**1. Edit the kernel command line:**
```bash
sudo nano /etc/kernel/cmdline
```

**2. Append the following parameters to the end of the line:**
*(Replace `10de:2684,10de:22ba` with your actual GPU Video and Audio IDs)*
```text
amd_iommu=on iommu=pt vfio-pci.ids=10de:2684,10de:22ba
```
* **`amd_iommu=on iommu=pt`**: Enables IOMMU and sets it to pass-through mode (improves host performance).
* **`vfio-pci.ids=...`**: Tells the kernel to bind the `vfio-pci` driver to your secondary GPU *before* the NVIDIA driver can claim it during boot.

**3. Apply the changes:**
In CachyOS, you must regenerate the boot entries/UKIs for this to take effect.
```bash
sudo reinstall-kernels
```

---

## 🪝 Phase 8: Libvirt QEMU Hook (Dynamic Switching & Hugepages)

Libvirt hooks allow you to run scripts automatically when a VM starts or stops. We will use a hook to dynamically allocate RAM (Hugepages) to prevent micro-stuttering, and to trigger our `gpu-toggle` script.

**1. Create the hook file:**
```bash
sudo nano /etc/libvirt/hooks/qemu
```

**2. Paste the following cleaned-up template:**
*(Adjust `VM_NAME`, `GPU_BDF`, and `AUDIO_BDF` to match your setup).*

```bash
#!/bin/bash
# Libvirt QEMU Hook: hugepages + GPU toggle + VFIO wait

# --- Configuration ---
enable_switching=1 # Set to 1 to enable automatic GPU switching

VM_NAME="$1"
VM_ACTION="$2"
VM_PHASE="${3:-}"

LOG_DIR="/var/log/libvirt/hooks"; mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/qemu-hook-${VM_NAME}.log"

GPU_TOGGLE="/usr/local/bin/gpu-toggle"
WAIT_VFIO="/usr/local/bin/wait-vfio" # Ensure you have a script or function for this, or rely on gpu-toggle's internal waits
LOCKFILE="/run/lock/gpu-toggle.lock"

# --- Device IDs ---
GPU_BDF="0000:01:00.0"
AUDIO_BDF="0000:01:00.1"

# --- Hugepages Configuration (32 GiB) ---
STATE_DIR="/run/libvirt/hooks"; mkdir -p "$STATE_DIR"
HP_STATE="$STATE_DIR/${VM_NAME}.hugepages"
VM_RAM_KiB=32768000
HP_SIZE_KiB=$(awk '/Hugepagesize/ {print $2}' /proc/meminfo || echo 2048)
NEED_PAGES=$((VM_RAM_KiB / HP_SIZE_KiB))
BUF_PAGES=$(( (128*1024) / HP_SIZE_KiB )) # 128 MiB buffer for Looking Glass/QEMU overhead
TARGET_PAGES=$((NEED_PAGES + BUF_PAGES))

exec >>"$LOG_FILE" 2>&1

log(){ echo "[HOOK][$VM_NAME][$VM_ACTION${VM_PHASE:+/$VM_PHASE}] $(date '+%F %T') - $*"; }

# --- Main Logic ---
if [ "$VM_NAME" != "windows-vfio" ]; then exit 0; fi

case "$VM_ACTION/$VM_PHASE" in
  prepare/begin)
    log "Prepare begin: Allocating hugepages..."
    awk '/HugePages_Total/ {print $2}' /proc/meminfo > "$HP_STATE" 2>/dev/null || echo 0 > "$HP_STATE"
    sync; echo 3 > /proc/sys/vm/drop_caches; echo 1 > /proc/sys/vm/compact_memory
    sysctl -w vm.nr_hugepages="$TARGET_PAGES" >/dev/null

    if [ "$enable_switching" -eq 1 ]; then
      log "Binding GPU to VFIO..."
      flock -w 30 "$LOCKFILE" -c "GPU_TOGGLE_NOLOCK=1 '$GPU_TOGGLE' vfio" || true
    fi

    udevadm settle -t 5 >/dev/null 2>&1 || true
    sleep 2.0
    ;;

  release/end)
    log "Release end: Releasing hugepages & returning GPU..."
    if [ "$enable_switching" -eq 1 ]; then
      flock -w 30 "$LOCKFILE" -c "GPU_TOGGLE_NOLOCK=1 '$GPU_TOGGLE' nvidia" || true
    fi

    INIT_HP=$(cat "$HP_STATE" 2>/dev/null || echo 0)
    rm -f "$HP_STATE"
    sysctl -w vm.nr_hugepages="$INIT_HP" >/dev/null
    ;;
esac
exit 0
```

**3. Make it executable and restart Libvirt:**
```bash
sudo chmod +x /etc/libvirt/hooks/qemu
sudo systemctl restart libvirtd
```

---

## 📝 Phase 9: VM XML Configuration (Ryzen 7950X3D Template)

Here is a heavily optimized XML template for your Windows 11 VM. 

### Why is it configured this way?
1. **CPU Pinning (7950X3D):** The Ryzen 7950X3D has two CCDs (Core Chiplet Dies). CCD0 has the 3D V-Cache (best for gaming), and CCD1 does not. This XML pins the VM strictly to **CCD0** (Cores 0-7 and their SMT threads 16-23) to ensure minimum latency and maximum gaming performance. It also isolates the QEMU emulator and IO threads to remaining cores.
2. **Hugepages Backing:** Instructs QEMU to use the memory allocated by our hook script.
3. **QEMU Commandline Args:** Injects the Looking Glass IVSHMEM device directly into QEMU, pointing it at `/dev/kvmfr0`.

### The XML Template
*(Run `virsh edit windows-vfio` and adapt this to your needs. Replace disk paths with your actual `qcow2` or `raw` paths).*

```xml
<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">
  <name>windows-vfio</name>
  <memory unit="KiB">32768000</memory>
  <currentMemory unit="KiB">32768000</currentMemory>
  
  <memoryBacking>
    <hugepages/>
    <nosharepages/>
    <locked/>
    <access mode="shared"/>
  </memoryBacking>

  <vcpu placement="static">16</vcpu>
  <iothreads>2</iothreads>

  <cputune>
    <vcpupin vcpu="0" cpuset="0"/>
    <vcpupin vcpu="1" cpuset="16"/>
    <vcpupin vcpu="2" cpuset="1"/>
    <vcpupin vcpu="3" cpuset="17"/>
    <vcpupin vcpu="4" cpuset="2"/>
    <vcpupin vcpu="5" cpuset="18"/>
    <vcpupin vcpu="6" cpuset="3"/>
    <vcpupin vcpu="7" cpuset="19"/>
    <vcpupin vcpu="8" cpuset="4"/>
    <vcpupin vcpu="9" cpuset="20"/>
    <vcpupin vcpu="10" cpuset="5"/>
    <vcpupin vcpu="11" cpuset="21"/>
    <vcpupin vcpu="12" cpuset="6"/>
    <vcpupin vcpu="13" cpuset="22"/>
    <vcpupin vcpu="14" cpuset="7"/>
    <vcpupin vcpu="15" cpuset="23"/>
    <emulatorpin cpuset="24-27"/>
    <iothreadpin iothread="1" cpuset="28-29"/>
    <iothreadpin iothread="2" cpuset="30-31"/>
  </cputune>

  <os firmware="efi">
    <type arch="x86_64" machine="pc-q35-9.2">hvm</type>
    <firmware>
      <feature enabled="yes" name="secure-boot"/>
    </firmware>
    <loader readonly="yes" secure="yes" type="pflash" format="raw">/usr/share/edk2/x64/OVMF_CODE.secboot.4m.fd</loader>
    <nvram template="/usr/share/edk2/x64/OVMF_VARS.4m.fd" templateFormat="raw" format="raw">/var/lib/libvirt/qemu/nvram/windows-vfio_VARS.fd</nvram>
  </os>

  <features>
    <acpi/>
    <apic/>
    <hyperv mode="custom">
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vpindex state="on"/>
      <runtime state="on"/>
      <synic state="on"/>
      <stimer state="on"/>
      <reset state="on"/>
      <frequencies state="on"/>
      <reenlightenment state="on"/>
      <tlbflush state="on"><direct state="on"/></tlbflush>
      <ipi state="on"/>
    </hyperv>
    <kvm><hidden state="on"/></kvm>
    <smm state="on"/>
    <ioapic driver="kvm"/>
  </features>

  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" clusters="1" cores="8" threads="2"/>
    <feature policy="require" name="topoext"/>
    <feature policy="require" name="invtsc"/>
  </cpu>

  <clock offset="localtime">
    <timer name="hpet" present="no"/>
    <timer name="hypervclock" present="yes"/>
    <timer name="tsc" present="yes" mode="native"/>
  </clock>

  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    
    <disk type="file" device="disk">
      <driver name="qemu" type="qcow2" cache="none" io="native" discard="unmap"/>
      <source file="/path/to/your/Windows11.qcow2"/>
      <target dev="vda" bus="virtio"/>
      <boot order="1"/>
    </disk>

    <interface type="bridge">
      <mac address="52:54:00:87:8f:44"/>
      <source bridge="br0"/>
      <model type="virtio"/>
    </interface>

    <input type="mouse" bus="virtio"/>
    <input type="keyboard" bus="virtio"/>

    <tpm model="tpm-crb">
      <backend type="emulator" version="2.0">
        <profile name="default-v1"/>
      </backend>
    </tpm>

    <hostdev mode="subsystem" type="pci" managed="yes">
      <driver name="vfio"/>
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/> </source>
      <address type="pci" domain="0x0000" bus="0x02" slot="0x00" function="0x0" multifunction="on"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <driver name="vfio"/>
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x1"/> </source>
      <address type="pci" domain="0x0000" bus="0x02" slot="0x00" function="0x1"/>
    </hostdev>
  </devices>

  <qemu:commandline>
    <qemu:arg value="-device"/>
    <qemu:arg value="ivshmem-plain,memdev=looking-glass,addr=0x10"/>
    <qemu:arg value="-object"/>
    <qemu:arg value="memory-backend-file,size=128M,mem-path=/dev/kvmfr0,share=on,id=looking-glass"/>
    
    <qemu:env name="QEMU_AUDIO_DRV" value="pa"/>
    <qemu:env name="QEMU_PA_SERVER" value="/run/user/1000/pulse/native"/> 
    <qemu:env name="QEMU_AUDIO_TIMER_PERIOD" value="50"/>
    <qemu:env name="QEMU_PA_LATENCY_OUT" value="20"/>
  </qemu:commandline>
</domain>
```

---

## 🪟 Phase 10: Guest OS Finalization

Once your VM boots into Windows via Virt-Manager (or Looking Glass with a dummy plug/EDID emulator), you must complete the final software links:

1. **Install the NVIDIA Drivers:** Download and install the latest standard drivers inside the VM.
2. **Install Looking Glass Host:** Download the **Looking Glass Host application for Windows**. 
   * ⚠️ **CRITICAL:** The version of the host application installed in Windows (e.g., `B6`) **must exactly match** the version of the Looking Glass client you compiled/installed on your Linux host. If there is a version mismatch, Looking Glass will fail to connect or display a black screen.

**Next Steps:** With the host system, networking, and Looking Glass configured, you are now ready to set up the **GPU Toggle Script** to dynamically unbind your Work GPU from the host and pass it to the Virtual Machine.
