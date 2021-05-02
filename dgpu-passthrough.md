# dGPU Passthrough

## Machine
- Acer Nitro 5 AN515-51
- CPU: Intel(R) Core(TM) i5-7300HQ CPU @ 2.50GHz
- RAM: 16GB
- dGPU: NVIDIA GeForce GTX 1050 Mobile (GP107M) 4GB

## Latest configuration
- Same with the previous one
- HDMI Audio controller now working
- POST screen on HDMI monitor
- vender_id = `GenuineIntel`
- Passed the ROM from [here](https://www.techpowerup.com/vgabios/219078/219078) in libvirt XML (`<rom file='...'/>`)
- The ROM in OVMF is extract from BIOS update.
- `rombar=on` (which is the default value)
- GVT-g with address `0000:00:02.0`
- Windows still not recognize this as an Optimus setup.

### Notes
- POST screen on HDMI works only if `rombar=on` and dGPU's audio controller passed through.
- It is required to use that ROM in the XML, which have an EFI entry, to show POST on HDMI.
- Not sure if this is enforced for ROM in OVMF. Need more tests.
- UpGOPd stuff not working. Probably because that ROM have some code to initialize the screen. And/or upGOPd not worked with my ROM.
- macOS also works if it have no second screen (no GVT-g or QXL or whatever).
- During Windows boot, there might be some graphical glitches, then the dGPU was turned off and on again when the login screen show up.
- NVIDIA Control Panel with the ROM from TechPowerUp does not recognize GVT-g monitor.
- HDMI must be connected to a monitor during boot.

## Older configuration
- Debian 11 with Linux 5.10.0-5-amd64 kernel
- QEMU 5.2.0
- OVMF (commit c640186) with VBIOS ACPI patch[1] (slightly modified)
- VBIOS extracted from Windows Registry (the one extracted from BIOS update also working)
- Battery patch[2] (included and compiled with the OVMF patch above)
- vendor_id = `1234567890a` (11 characters)
- KVM state hidden
- dGPU with address `0000:01:00.0` with sub-vendor-id and sub-device-id set
- Before assign dGPU to vfio-pci, refresh it first by turning off, load NVIDIA module on host, assign it to the module, and running `nvidia-smi -r`.
- Windows 10 guest
- No HDMI sound
- Hyper-V stuff

### Notes
- Optimus still not working. The NVIDIA control panel did not have the option to select GPU.
- If you don't need Optimus, pass only the dGPU and maybe QXL for controls.
- PCI rescan will work only if the devices you want to rescan is in D0 state.
- For some reason, dGPU need to be initialized (?) by Linux first (either from host or a Linux VM),
  and it will working on Windows (no code 43, yay!) until you turned dGPU off (e.g. using bbswitch),
  then you need to redo this process again. I only tested this against the proprietary driver, not sure about nouveau.
- GOPupd ROMs did not work, but good news is we are not required to use an UEFI ROM.
- However, [this ROM](https://www.techpowerup.com/vgabios/219078/219078) works, and it also have EFI entry.
  But we did not need it, so I used ROM extracted from my machine.
- I think Windows will get a blue screen (VIDEO TDR FAILURE) from the Intel driver when I forced the resolution of GVT-g screen.
  Not sure if this true though.
- Windows and Linux guest freeze after a few minutes if I launched it with dGPU. If we force it off or let it crash,
  we likely to unable to boot any VM with disks that use virtio. Those VM could not get passed the Tianocore screen
  and used 100% on a CPU core. We can switch to SATA but dGPU will not be working. The host need to be reset.
  [This was fixed](https://lore.kernel.org/kvm/bug-209253-28872@https.bugzilla.kernel.org%2F/T/).
- I need to change some AppArmor config for file access (that also got facl configured) and for executing nvidia-smi.
- "Unknown header type 7f" message appears in `lspci -v` if the device is turned off.
- Do not keep bbswitch loaded when removing the dGPU PCI device, otherwise it will not usable and cannot unload using `modprobe -r`.
- Enable MSI for HDA controller in Linux guest with `options snd-hda-intel enable_msi=1` in modprobe.conf
- Keep bbswitch loaded after VM shutdown, so the machine can awake with dGPU enabled after sleep.

#### Audio controller

If I don't reset the audio controller before binding it to vfio-pci, then

- Things broke if you pass it to VM. I don't know why.
- When I start the VM with it passed through, the host will stuttering a lot even though the CPU usage was normal.
  Same situation can be observed if you turn off the audio controller in VM (e.g. using nvhda).
- I need to restart the host in order to make the dGPU working again.

There are other problems too.

- dGPU fails to work in Windows guest when I bind the audio controller to vfio-pci. Works normally when using pci-stub.
- This means for now I can't get any audio out on HDMI.

### Reset procedure
Here is a reset procedure in case the NVIDIA driver complains about devices fallen off the bus.

1. Turn off dGPU using bbswitch
2. Turn off its HDA controller using nvhda
3. Turn on dGPU
4. Turn on HDA
5. Unload bbswitch and nvhda
6. Remove HDA controller PCI using `# echo 1 > "/sys/bus/pci/devices/0000:01:00.1/remove"`
7. Reset dGPU PCI using `# echo 1 > "/sys/bus/pci/devices/0000:01:00.0/reset"`
8. Remove dGPU PCI
9. Remove the PCI bridge at `0000:00:01.0`
10. Rescan PCI devices using `# echo 1 > "/sys/bus/pci/rescan"`
11. Unbind dGPU from drivers
12. Load bbswitch
13. Turn off dGPU
14. Load NVIDIA driver and bind it to dGPU
15. Run `# nvidia-smi -r`

Most of the time, you can just run step 3, 9-11, 14, and 15.

### State-transition table of bbswitch/nvhda
All of them are idempotent operations. You cannot turn on HDA controller if dGPU was off.

| Current state (bbswitch/nvhda) | Operation    | Next state (bbswitch/nvhda) | Changed? |
|--------------------------------|--------------|-----------------------------|----------|
| OFF/OFF                        | bbswitch OFF | OFF/OFF                     | No       |
| OFF/OFF                        | bbswitch ON  | ON/OFF                      | Yes      |
| OFF/OFF                        | nvhda OFF    | OFF/OFF                     | No       |
| OFF/OFF                        | nvhda ON     | OFF/OFF                     | No       |
| OFF/ON                         | bbswitch OFF | OFF/ON                      | No       |
| OFF/ON                         | bbswitch ON  | ON/ON                       | Yes      |
| OFF/ON                         | nvhda OFF    | OFF/OFF                     | Yes      |
| OFF/ON                         | nvhda ON     | OFF/ON                      | No       |
| ON/OFF                         | bbswitch OFF | OFF/OFF                     | Yes      |
| ON/OFF                         | bbswitch ON  | ON/OFF                      | No       |
| ON/OFF                         | nvhda OFF    | ON/OFF                      | No       |
| ON/OFF                         | nvhda ON     | ON/ON                       | Yes      |
| ON/ON                          | bbswitch OFF | OFF/ON                      | Yes      |
| ON/ON                          | bbswitch ON  | ON/ON                       | No       |
| ON/ON                          | nvhda OFF    | ON/OFF                      | Yes      |
| ON/ON                          | nvhda ON     | ON/ON                       | No       |

## Resources and interesting stuffs
- Optimus laptop dGPU passthrough guide: [https://gist.github.com/Misairu-G/616f7b2756c488148b7309addc940b28](https://gist.github.com/Misairu-G/616f7b2756c488148b7309addc940b28)
- Another guide: [https://lantian.pub/en/article/modify-computer/laptop-intel-nvidia-optimus-passthrough.lantian/](https://lantian.pub/en/article/modify-computer/laptop-intel-nvidia-optimus-passthrough.lantian/)
- Someone got the NVIDIA Control Panel to work on a Muxless Optimus laptop: [https://www.reddit.com/r/VFIO/comments/i80ffe/i_just_got_the_nvidia_control_panel_to_work_on_a/](https://www.reddit.com/r/VFIO/comments/i80ffe/i_just_got_the_nvidia_control_panel_to_work_on_a/)
- Current State of Optimus Muxless Laptop GPU Passthrough (Successful but Limited): [https://www.reddit.com/r/VFIO/comments/8gv60l/current_state_of_optimus_muxless_laptop_gpu/](https://www.reddit.com/r/VFIO/comments/8gv60l/current_state_of_optimus_muxless_laptop_gpu/)
- libvirt XML format: [https://libvirt.org/formatdomain.html](https://libvirt.org/formatdomain.html)
- The original ACS override patch: [https://lkml.org/lkml/2013/5/30/513](https://lkml.org/lkml/2013/5/30/513)
- ACS override kernel builds: [https://queuecumber.gitlab.io/linux-acs-override/](https://queuecumber.gitlab.io/linux-acs-override/)
- A VFIO blog: [https://vfio.blogspot.com/](https://vfio.blogspot.com/)

## References
- [1] ACPI VBIOS stuff and useful information: [https://github.com/jscinoz/optimus-vfio-docs/issues/2](https://github.com/jscinoz/optimus-vfio-docs/issues/2)
- [2] Battery ACPI table: [https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#%22Error_43:_Driver_failed_to_load%22_with_mobile_(Optimus/max-q)_nvidia_GPUs](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#%22Error_43:_Driver_failed_to_load%22_with_mobile_(Optimus/max-q)_nvidia_GPUs)

