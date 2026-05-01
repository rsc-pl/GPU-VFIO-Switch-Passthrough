# GPU-VFIO-Switch-Passthrough
A robust, state-aware Bash script for dynamically switching an NVIDIA GPU between the Linux host and a Virtual Machine (vfio-pci) without rebooting. It features strict power management to prevent "zombie GPU" states and active pre-flight safe checks that block driver unbinding if host applications are currently using the GPU.
