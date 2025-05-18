# Purpose
The purpose of this repository is to document my specific situation for myself and also as a datapoint for others who may want to set up GPU passthrough with KVM who have a similar situation to mine.

# My Situation
* Hardware - Dell Precision 3560 (17" Laptop)
  * Manufactured in 2022
  * NVIDIA Quadro T500 Mobile (TU117GLM)
  * Intel 11th Gen i7-1185G7 3.0 GHz (4 cores, 8 threads)
  * 32 GB RAM
  * HDMI Port, 2x Thunderbolt ports with DisplayPort
  * Monitors: 1080p LCD panel + 1440p Desktop panel
* Host Operating System - Fedora KDE 42 (at time of commit)
  * LUKS encrypted root partition

Paid $480 on eBay for the computer, so fairly good deal given the recent hardware.

FYI I will be providing commands run from the Fedora perspective. I leave it as an exercise for the reader to translate the commands to their host OS, thankfully many commands are ubiquitous.

My use case is I want to use Linux first and Windows as needed. I consider the Windows install to be 100% volitile and can be blown away at any time and the guest OS reinstalled if need be without much fuss beyond normal install/setup. 

My work requires me to have a Windows machine, and I would prefer to only maintain one device. Additionally, any processing that I can offload to the GPU will help spread the load and make the VM (hopefully) faster and more responsive. This is especially relevant for meetings that require a camera.

I won't be using CPU pinning because, well it's a laptop, it's cooled by hopes and dreams, but mainly by a 2-inch blower fan. I'm trusting the Linux Scheduler to spread the load across 6 of the 8 CPU threads.

## Discovery
This procedure helped me to understand the hardware of my machine better and helped guide my next steps.

```
cd /sys/class/drm
ls -la
```

Result:
```
total 0
drwxr-xr-x  2 root root    0 May 17 13:27 .
drwxr-xr-x 88 root root    0 May 17 13:27 ..
lrwxrwxrwx  1 root root    0 May 17 13:28 card1 -> ../../devices/pci0000:00/0000:00:02.0/drm/card1
lrwxrwxrwx  1 root root    0 May 17 13:28 card1-DP-1 -> ../../devices/pci0000:00/0000:00:02.0/drm/card1/card1-DP-1
lrwxrwxrwx  1 root root    0 May 17 13:28 card1-DP-2 -> ../../devices/pci0000:00/0000:00:02.0/drm/card1/card1-DP-2
lrwxrwxrwx  1 root root    0 May 17 13:28 card1-eDP-1 -> ../../devices/pci0000:00/0000:00:02.0/drm/card1/card1-eDP-1
lrwxrwxrwx  1 root root    0 May 17 13:28 card1-HDMI-A-1 -> ../../devices/pci0000:00/0000:00:02.0/drm/card1/card1-HDMI-A-1
lrwxrwxrwx  1 root root    0 May 17 13:28 renderD128 -> ../../devices/pci0000:00/0000:00:02.0/drm/renderD128
-r--r--r--  1 root root 4096 May 17 13:28 version
```

The important things here are card1-DP1, card1-DP-2, card1-eDP-1 and card1-HDMI-A-1 . As far as I understand, these map to Thunderbolt ports 1 and 2, the laptop's internal LCD panel, and the HDMI port respectively.

All of these are symbolic links, and they point to pci device 0000:00:02.0. Which device is that?

```
lspci -s 0000:00:02.0
```

Result:
```
00:02.0 VGA compatible controller: Intel Corporation TigerLake-LP GT2 [Iris Xe Graphics] (rev 01)
```

So what this means is that the Intel iGPU is the only one physically connected to any display output, which means that the Intel iGPU does the final rendering. The NVIDIA GPU only does heavy lifting rendering/computation tasks, and ultimately will ALWAYS pass any final rendering to screen to the Intel iGPU. So, we need to create a similar situation with our VM and keep in mind that the Intel iGPU will always be engaged no matter what we do.

# Preparation

I have followed [these steps provided by SysGuides](https://sysguides.com/install-kvm-on-linux) regarding the configuration for KVM. I did not configure the network bridge as I am using the VM directly, not remotely; I have no need to access the VM from outside the localhost network.

I have also followed [these steps provided by SysGuides](https://sysguides.com/install-a-windows-11-virtual-machine-on-kvm) regarding the Windows guest VM configuration. The most important steps here is the XML configuration. I think there may be some magic config in here that allowed my situation to work better, but I have not identified which part is responsible.

## Bind GPU to VFIO
These are the steps I'm using to bind the dGPU to VFIO for use by KVM and the VMs.
* Configure modprobe
* Configure dracut
* Rebuild initramfs

## Configure modprobe
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
Note Kernel driver in use is nouveau, meaning vfio is not bound to the dGPU.

File: /etc/modprobe.d/vfio.conf
```
softdep drm pre: vfio-pci
options vfio-pci ids=10de:1fbb disable_vga=1
```

Note the inclusion of the Device ID with the "ids" option

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
[    2.340558] vfio-pci 0000:01:00.0: optimus capabilities: enabled, status dynamic power, hda bios codec supported
[  172.365263] vfio-pci 0000:01:00.0: resetting
[  172.468808] vfio-pci 0000:01:00.0: reset done
[  172.503476] vfio-pci 0000:01:00.0: resetting
[  172.604890] vfio-pci 0000:01:00.0: reset done
```

I think it's interesting that optimus is detected with vfio, implying some amount of power management may come into play. I've found that I got slightly better performance when this appeared.

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
[VM Configuration](games-vm.xml)

Summary
* Add PCI Device > Add GPU
* Remove Tablet device
* Remove Channel (spice)

## Add PCI Device for GPU
```
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
</hostdev>
```

Review the VM configuration and validate that no other devices are on bus 0x06 and slot 0x00. I had an existing VM that had some extra controller tags that conflicted with the bus and slot that was connected to the GPU passthrough. I had to remove that controller tag, which thankfully didn't hurt anything and allowed the GPU to be utilized correctly.

## Removing Channel (spice)
I believe what this did was allow clipboard data to pass between guest and host as well as seamless mouse transition from outside to inside VM window. Removing this was not a big deal for me and in fact increases security (password leaks) and also just capturing the mouse normally into the VM makes more sense to me.

# Tuning
## Games VM
Regarding my VM for games, I found that while QXL is significantly more performant that VFIO for Video, there were massive issues with the audio. I worked around this by passing a cheap set of USB speakers to this VM, which resolved the audio issues. Additionally, updated the following line in the KVM configuration which disables any audio output to the host via spice:

Original:
```
    <audio id="1" type="spice"/>
```

Updated:
```
    <audio id="1" type="none"/>
```

Also for games VM, I reduced the number of cores from 3 to 2 to give the host system more power to handle CPU frame generation for the game. This did reduce vertical tearing.

## Work VM
Regarding the work VM, I went to Settings > System > Displays > Graphics > Advanced and set NVIDIA T500 to the default GPU.

# Backups
More details to come on this when I get it 100% ironed out. The general idea is to perform an external backup once per month. When the month is done, commit the external backup to the main qcow2, then create a new backup point. Repeat. This way if anything goes wrong during the month, rolling back is as easy as setting the storage to point to the main qcow2 file.

# VM Migration
Given that my use case is using BitLocker and TPM, I've had a terrible time migrating VMs from one machine to another. I would prefer to install anew versus migrating.

# Enhancements
Currently my VMs utilize file-based qcow2 files for storage. One day I would like to utilize a fast external USB4/Thunderbolt SSD or NVMe drive to store, one external drive per VM. This could be passed to the VM as a block storage device, so it would appear to be a normal drive to the VM and to Windows (not yet tested).

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
* Lack of in-built backup mechanism
  * While qcow2 is relatively slow compared to a nvme external device, what qcow2 does provide is a reliable backup solution with internal and external backups in case something goes wrong.
