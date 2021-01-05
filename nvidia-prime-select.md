# nvidia-prime-select

[Repository](https://github.com/wildtruc/nvidia-prime-select)

- In x86-64 Debain, you need to change the module path in `/etc/nvidia-prime/xorg.nvidia.conf` from `lib64` to `lib`.
- You might need to remove a condition that check for monitor in `/etc/nvidia-prime/xinitrc.prime` since it might not
detect the monitor that connects to dGPU.

