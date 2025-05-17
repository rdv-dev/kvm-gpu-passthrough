# Purpose
The purpose of this repository is to document my specific situation for myself and also as a datapoint for others who may want to set up GPU passthrough with KVM who have a similar situation to mine.

# My Situation
* Hardware - Dell Precision 3560 (17" Laptop)
 * Manufactured in 2022
 * NVIDIA Quadro T500 Mobile (TU117GLM)
 * Intel 11th Gen i7-1185G7 3.0 GHz
 * 32 GB RAM
 * HDMI Port
 * Monitors: 1080p LCD panel + 1440p Desktop panel
* Host Operating System - Fedora KDE 42 (at time of commit)
 * LUKS encrypted root partition

FYI I will be providing commands run from the Fedora perspective. I leave it as an exercise for the reader to translate the commands to their host OS, thankfully many commands are the same.

My use case is I want to use Linux first and Windows as needed. I consider the Windows install to be 100% volitile and can be blown away at any time and the guest OS reinstalled if need be without much fuss beyond normal install/setup. 

My work requires me to have a Windows machine, and I would prefer to only maintain one device. Additionally, any processing that I can offload to the GPU will help spread the load and make the VM (hopefully) faster and more responsive. This is especially relevant for meetings that require a camera.

I won't be using CPU pinning because, well it's a laptop, it's cooled by hopes and dreams, but mainly by a 2-inch blower fan. I'm trusting the Linux Scheduler to spread the load across 6 of the 8 CPU threads.

## Nuances
The nuances of this hardware is that the NVIDIA GPU (dGPU) might fall within the Optimus case. In short, this means that the Intel iGPU is primary and the dGPU is secondary. Anything that is rendered by the dGPU goes through the iGPU (massive generalization). In other words, the dGPU does not have its own dedicated port to drive a monitor, [as far as I understand](https://lantian.pub/en/article/modify-computer/laptop-intel-nvidia-optimus-passthrough.lantian/).

Now, this specific hardware configuration may be different from Optimus in some way, but as far as I know, I don't have a real way of verifying that beyond saying "what I've done works" and "trust me bro" which I do not recommend. If there is a way to find out definitively, let me know after reading the Preparation section. Your mileage may vary.

Given these nuances, in my testing of this machine I found that games (Farcry: Primal, No Man's Sky) run at a higher frame rate and at a higher resolution on a Windows VM with GPU passthrough than directly through Linux. This could be an effect of improper or missing Linux configuration, or was treated like an Optimus case by Linux even if it might not be, and therefore hurt the performance. I have not tested Windows natively on this machine.

Somehow, despite the possible Optimus case, the NVIDIA GPU was able to be decoupled from Linux and handed to the VM. The device shows up in the guest as expected, and when it is used to render things, the usage can be monitored through basic Windows Task Manager. I don't understand why it works.

# Preparation

I have followed [these steps provided by SysGuides](https://sysguides.com/install-kvm-on-linux) regarding the configuration for KVM. I did not configure the network bridge as I am using the VM directly, not remotely; I have no need to access the VM from outside the localhost network.

I have also followed [these steps provided by SysGuides](https://sysguides.com/install-a-windows-11-virtual-machine-on-kvm) regarding the Windows guest VM configuration. The most important steps here is the XML configuration. I think there may be some magic config in here that allowed my situation to work better, but I have not identified which part is responsible.

These next steps are the general, heavy handed approach to prepare the dGPU for unbinding from Linux and binding to vfio for use by KVM and the VMs. This is my current configuration.
* Blacklist nouveau driver
* Setup kernel parameters
* Configure vfio
* Autostart vfio kernel modules on boot
* Rebuild initramfs

[These steps from the Arch Wiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) are probably better.

### Side Note
[Here's a fancy resource](https://github.com/bryansteiner/gpu-passthrough-tutorial/?tab=readme-ov-file#----part-2-vm-logistics) for dynamically binding and unbinding the dGPU based on the VM state (up/down), although this is a desktop environment which may be less complex than the laptop environment

## Optimus
This is the [RPM fusion guide for installing NVIDIA proprietary drivers](https://rpmfusion.org/Howto/%20NVIDIA) on Fedora, just using this as a reference for the next few commands.
```
/sbin/lspci | grep -e VGA
```

Returns:

```
00:02.0 VGA compatible controller: Intel Corporation TigerLake-LP GT2 [Iris Xe Graphics] (rev 01)
```

Where is the NVIDIA GPU??

```
/sbin/lspci | grep -e 3D
```

```
01:00.0 3D controller: NVIDIA Corporation TU117GLM [Quadro T500 Mobile] (rev a1)
```

This is the "probably in the Optimus case" as stated by this article. If there's a better way to validate the Optimus status, let me know.

## Blacklist nouveau
```
echo "blacklist nouveau" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
```

## Configure VFIO
Find the Device ID of the dGPU
```
lspci -nnk | grep -A3 NVIDIA
```

```
01:00.0 3D controller [0302]: NVIDIA Corporation TU117GLM [Quadro T500 Mobile] [10de:1fbb] (rev a1)
        Subsystem: Dell Device [1028:0a22]
        Kernel driver in use: nouveau
        Kernel modules: nouveau
```

Important part here is the device ID: 10de:1fbb

File: /etc/modprobe.d/vfio.conf
```
options vfio-pci ids=10de:1fbb disable_vga=1
```

## Load VFIO Kernel Modules On Boot
File: /etc/dracut.conf.d/vfio.conf
```
add_drivers+=" vfio vfio_iommu_type1 vfio_pci "
force_drivers+=" vfio vfio_pci "
```

## Rebuild initramfs
```
sudo dracut --force
```

Reboot!

## Verify
```
sudo dmesg | grep -i vfio
```

Result:
```
[    1.401500] VFIO - User Level meta-driver version: 0.3
[    1.412025] vfio_pci: add [10de:1fbb[ffffffff:ffffffff]] class 0x000000/00000000
[  150.032209] vfio-pci 0000:01:00.0: resetting
[  150.135676] vfio-pci 0000:01:00.0: reset done
[  150.165138] vfio-pci 0000:01:00.0: resetting
[  150.271853] vfio-pci 0000:01:00.0: reset done
```

```
lspci -nnk -d 10de:1fbb
```

Result:
```
01:00.0 3D controller [0302]: NVIDIA Corporation TU117GLM [Quadro T500 Mobile] [10de:1fbb] (rev a1)
        Subsystem: Dell Device [1028:0a22]
        Kernel driver in use: vfio-pci
        Kernel modules: nouveau
```

# Virtual Machine Configuration
Summary
* Add PCI Device > Add GPU
* Remove some Spice things
* Configure Video QXL to Video Virtio


# VM Migration
Given that my use case is using BitLocker and TPM, I've had a terrible time migrating VMs from one machine to another. I would prefer to install anew versus migrating.

# Enhancements
Currently my VMs utilize file-based qcow2 files for storage. One day I would like to utilize a fast external USB4/Thunderbolt SSD or NVMe drive to store, one external drive per VM.

Here is why I believe this would be an improvement:

### Pros
* Increased Security
 * Both VMs are secured using BitLocker. The BitLocker key data is stored via emulated TPM on LUKS encrypted Linux root partition.
 * If someone were to get a hold of the external drive and laptop, in order to decrypt the drive quickly they would need to:
  * Have physical access to the laptop
  * Be able to decrypt the Linux root parition
 * Neither the laptop or the VMs are vulnerable to a physical sniffing for the key to be transferred from TPM to another device
* Reduced space load on Laptop drive
* Theoretical drive read/write speed increase
 * The theory being that qcow2 presents at least one additional abstraction layer in order to read/write
 * Attaching a USB 4/Thunderbolt device will be less than PCIe speed but nonetheless a direct read/write process, eliminating an abstraction layer and performing read/writes directly to the device through normal OS channels

### Cons
* Heat
 * External drives are not known for cooling themselves well and could cause the storage to thermal throttle under continuous or high usage.
