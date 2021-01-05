# dGPU Passthrough

## Machine
- Acer Nitro 5 AN515-51
- CPU: Intel(R) Core(TM) i5-7300HQ CPU @ 2.50GHz
- RAM: 16GB
- dGPU: NVIDIA GeForce GTX 1050 Mobile (GP107M) 4GB

## Working configuration
- OVMF with VBIOS ACPI patch[1] (slightly modified)
- VBIOS extracted from Windows Registry (the one extracted from BIOS update also working)
- Battery patch[2] (included and compiled with the OVMF patch above)
- vendor_id = 1234567890a
- KVM state hidden
- GVT-g with address 0000:00:02.0, using ROM from [https://github.com/HouQiming/i915ovmfPkg/](https://github.com/HouQiming/i915ovmfPkg/)
- dGPU with address 0000:01:00.0 with sub-vendor-id and sub-device-id set
- Before assign dGPU to vfio-pci, refresh it first by turning off, load NVIDIA module on host, assign it to the module, and running `nvidia-smi -r`.

## Notes
- Optimus still not working. The NVIDIA control panel did not have the option to select GPU.
- If you don't need Optimus, pass only the dGPU and maybe QXL for controls.
- PCI rescan will work only if the devices you want to rescan is in D0 state.
- For some reason, dGPU need to be initialized (?) by Linux first (either from host or a Linux VM), and it will working on Windows (no code 43, yay!) until you turned dGPU off (e.g. using bbswitch), then you need to redo this process again. I only tested this against the proprietary driver, not sure about nouveau.
- GOPupd ROMs did not work, but good news is we are not required to use an UEFI ROM.
- However, [this ROM](https://www.techpowerup.com/vgabios/219078/219078) works, and it also have EFI entry. But we did not need it, so I used ROM extracted from my machine.
- I think Windows will get a blue screen (VIDEO TDR FAILURE) when I forced the resolution of GVT-g screen. Not sure if this true though.
- Windows guest freeze after a few minutes if I launched it with dGPU. Not sure about Linux guest though. At least it worked perfectly fine when only display on GVT-g screen and I can use nvidia-smi in it.
- I need to change some AppArmor config for file access (that also got facl configured) and for executing nvidia-smi.

### Audio controller
- Things broke if you pass it to VM. I don't know why.
- When I start the VM with it passed through, the host will stuttering a lot even though the CPU usage was normal.
- I need to restart the host in order to make the dGPU working again.

## Resources and interesting stuffs
- Optimus laptop dGPU passthrough guide: [https://gist.github.com/Misairu-G/616f7b2756c488148b7309addc940b28](https://gist.github.com/Misairu-G/616f7b2756c488148b7309addc940b28)
- Another guide: [https://lantian.pub/en/article/modify-computer/laptop-intel-nvidia-optimus-passthrough.lantian/](https://lantian.pub/en/article/modify-computer/laptop-intel-nvidia-optimus-passthrough.lantian/)
- Someone got the NVIDIA Control Panel to work on a Muxless Optimus laptop: [https://www.reddit.com/r/VFIO/comments/i80ffe/i_just_got_the_nvidia_control_panel_to_work_on_a/](https://www.reddit.com/r/VFIO/comments/i80ffe/i_just_got_the_nvidia_control_panel_to_work_on_a/)
- Current State of Optimus Muxless Laptop GPU Passthrough (Successful but Limited): [https://www.reddit.com/r/VFIO/comments/8gv60l/current_state_of_optimus_muxless_laptop_gpu/](https://www.reddit.com/r/VFIO/comments/8gv60l/current_state_of_optimus_muxless_laptop_gpu/)
- libvirt XML format: [https://libvirt.org/formatdomain.html](https://libvirt.org/formatdomain.html)

## References
- [1] ACPI VBIOS stuff: [https://github.com/jscinoz/optimus-vfio-docs/issues/2](https://github.com/jscinoz/optimus-vfio-docs/issues/2)
- [2] Battery ACPI table: [https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#%22Error_43:_Driver_failed_to_load%22_with_mobile_(Optimus/max-q)_nvidia_GPUs](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#%22Error_43:_Driver_failed_to_load%22_with_mobile_(Optimus/max-q)_nvidia_GPUs)

