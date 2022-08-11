# Amd Proxmox Windows 10 GPU Passthrough

## Change boot parameters

Inside of /etc/default/grub change the following line to include:



```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt amd_iommu=on video=efifb:off"
```
After changing the line run a:
```bash
update-grub
```

## Blacklist drivers from loading


```bash
echo "blacklist nouveau" >> /etc/modprobe.d/pve-blacklist.conf
echo "blacklist nvidia" >> /etc/modprobe.d/pve-blacklist.conf
echo "blacklist radeon" >> /etc/modprobe.d/pve-blacklist.conf
echo "blacklist nvidiafb" >> /etc/modprobe.d/pve-blacklist.conf
```

## Add these parameters to modules 


```bash
echo vfio >> /etc/modules
echo vfio_iommu_type1 >> /etc/modules
echo vfio_pci >> /etc/modules
echo vfio_virqfd >> /etc/modules
```
When that is done run a:

```bash
update-initramfs -u
```

## vfio-pci config

Once that is done, we’re going to find the PCI ID’s of the NVidia GPU we want to passthrough. Run a:

```bash
lspci -v
```
```bash
08:00.0 VGA compatible controller: NVIDIA Corporation GP106 [GeForce GTX 1060] (rev a1) (prog-if 00 [VGA controller])
        Subsystem: Gigabyte Technology Co., Ltd GP106 [GeForce GTX 1060]
        Flags: fast devsel, IRQ 354
        Memory at f6000000 (32-bit, non-prefetchable) [size=16M]
        Memory at c0000000 (64-bit, prefetchable) [size=256M]
        Memory at d0000000 (64-bit, prefetchable) [size=32M]
        I/O ports at e000 [size=128]
        Expansion ROM at 000c0000 [disabled] [size=128K]
        Capabilities: [60] Power Management version 3
        Capabilities: [68] MSI: Enable- Count=1/1 Maskable- 64bit+
        Capabilities: [78] Express Legacy Endpoint, MSI 00
        Capabilities: [100] Virtual Channel
        Capabilities: [250] Latency Tolerance Reporting
        Capabilities: [128] Power Budgeting <?>
        Capabilities: [420] Advanced Error Reporting
        Capabilities: [600] Vendor Specific Information: ID=0001 Rev=1 Len=024 <?>
        Capabilities: [900] #19
        Kernel driver in use: vfio-pci
        Kernel modules: nvidiafb, nouveau

08:00.1 Audio device: NVIDIA Corporation Device 0fb9 (rev a1)
        Subsystem: Gigabyte Technology Co., Ltd Device 3747
        Flags: fast devsel, IRQ 355
        Memory at f7080000 (32-bit, non-prefetchable) [size=16K]
        Capabilities: [60] Power Management version 3
        Capabilities: [68] MSI: Enable- Count=1/1 Maskable- 64bit+
        Capabilities: [78] Express Endpoint, MSI 00
        Capabilities: [100] Advanced Error Reporting
        Kernel driver in use: vfio-pci
        Kernel modules: snd_hda_intel
```
You will get similar output, find your GPU
Run:
```bash
lspci -n -s 08:00
```
You will see something like this
```bash
08:00.0 0300: 10de:1c81 (rev a1)
08:00.1 0403: 10de:0fb9 (rev a1)
```
Now execute this: 
```bash
echo options vfio-pci ids=10de:1c81,10de:0fb9 disable_vga=1 > /etc/modprobe.d/vfio.conf
```
Reboot system and now check with command ```bash  lspci -v ``` that Nvidia card is now using vfio-pci instead of Nvidia driver

Create Windows 10 VM. 
Your server conf should look something like this: 

```bash 
agent: 1
bios: ovmf
boot: order=scsi0;net0
cores: 6
efidisk0: singlezfs:vm-100-disk-1,size=1M
hostpci0: 08:00.0,x-vga=on,pcie=1,romfile=GP106Patched.rom
machine: pc-q35-6.0
memory: 8192
name: Windows10
net0: virtio=06:12:0E:BE:63:7D,bridge=vmbr0,firewall=1
numa: 0
ostype: win10
scsi0: singlezfs:vm-100-disk-0,size=128G
scsihw: virtio-scsi-pci
smbios1: uuid=952161b0-735d-4b84-b296-efb5e3189a44
sockets: 1
vga: none
vmgenid: 0c1489be-9cde-400f-970d-fddad941a9d7

```

For me ```bash args: -cpu 'host,hv_vendor_id=NV43FIX,kvm=off' ``` isn't necessary any more since Nvidia allowed GPU Passthrough in latest drivers but it still could help you. Also I'm passing GPU ROM just in case. 

Make sure you can remote in before passing GPU since console won't work any more.
``` bash 
hostpci0: 08:00.0,x-vga=on,pcie=1,romfile=GP106Patched.rom
```
In case of pcie errors afer proxmox 7.2.5 replace
``` bash video=efifb:off```  
with 
``` bash initcall_blacklist=sysfb_init ```  
in /etc/default/grub
