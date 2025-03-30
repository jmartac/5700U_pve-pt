# AMD Ryzen iGPU Passthrough to Ubuntu VM on Proxmox VE

This guide details how to passthrough the integrated GPU from AMD Ryzen processors to a Ubuntu VM running on Proxmox VE.

It's highly based on [philuxes's guide](https://github.com/philuxe/pve/blob/main/5700g_PciPassthrough.md) (which can be found at [5700g_PciPassthrough.md](5700g_PciPassthrough.md) in this repo, left for reference) with additional minor notes and steps I followed for my different hardware configuration.

## Hardware Configuration

While the original guide was written for:

-  CPU: AMD Ryzen 7 5700G
-  Motherboard: Gigabyte Aorus 550M Elite

This fork includes steps I followed for:

-  Beelink SER5 PRO with Ryzen 7 5700U

As said, the process is the same for 5700U, I just rewrote the guide with the necessary adjustments for my hardware.

## Basic Passthrough Configuration

### GRUB Command Line Configuration

Add to your GRUB configuration:

```
quiet iommu=pt initcall_blacklist=sysfb_init
```

### Required Modules

Add to `/etc/modules`:

```
vfio
vfio_pci
vfio_virqfd
vfio_iommu_type1
```

### Blacklist GPU Drivers

Create or modify your blacklist configuration:

```
echo "blacklist amdgpu" >> /etc/modprobe.d/blacklist.conf
echo "blacklist radeon" >> /etc/modprobe.d/blacklist.conf
update-initramfs -u -k all
```

### Identify PCI IDs

Use `lspci -nnk` to identify your GPU PCI ID. In my case, for the 5700U, it was:

```
[...]
04:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Lucienne [1002:164c] (rev c1)
	Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Lucienne [1002:0123]
	Kernel driver in use: vfio-pci
	Kernel modules: amdgpu
[...]
```

My iGPU PCI ID `04:00.0`, device id: `1002:164c`

### Configure VFIO Modules

`nano/etc/modprobe.d/pt.conf` with `options vfio-pci ids={your_pci_id}`:

```
options vfio-pci ids=1002:164c
```

Then update initramfs:

```
update-initramfs -u -k all
```

## VM Configuration

My VM configuration (`/etc/pve/qemu-server/101.conf`, some values omitted for brevity):

```
agent: 1
boot: order=scsi0;net0
cores: 4
cpu: host,hidden=1
hostpci0: 0000:04:00.0,pcie=1,romfile=vbios_1002_164c_1.rom,x-vga=1 # -> see note below for vbios extraction
machine: q35
memory: 8192
net0: virtio=#############,bridge=vmbr0,firewall=1
numa: 0
onboot: 1
ostype: l26
scsi0: local-lvm:vm-101-disk-0,cache=writethrough,discard=on,iothread=1,size=130G
scsihw: virtio-scsi-single
serial0: socket
sockets: 1
```

## Additional Steps I followed

### 1. Machine Type swapping (i440fx to q35)

When swapping from i440fx to q35, network interfaces change names, which can cause SSH access loss. And that's what happened to me when I set the machine type to q35.

Thanks to [dioden94's comment in reddit](https://www.reddit.com/r/Proxmox/comments/st7zlv/comment/kxfio6x/), I figured out that I needed to change a parameter in `/etc/netplan/00-installer-config.yaml` (your filename could be slightly different).

To fix this:

1. Identify your network interface names:

| machine type | interface name (in my case) |
| ------------ | --------------------------- |
| i440fx       | ens18                       |
| q35          | enp6s18                     |

2. Determine the correct interface name for q35:
   -  Check in Proxmox UI: VM Summary > IPs section > "More" button
   -  Or run `ip a` inside the VM (if accessible)
3. Update your network configuration in the VM:
   -  `nano /etc/netplan/00-installer-config.yaml`
   -  Change the interface name to match the q35 name (in my case, I changed `ens18` to `enp6s18`)
4. Now switch back to q35 machine type

### 2. Manual VBIOS ROM Extraction

For the 5700U, I had to extract the VBIOS manually from official Beelink BIOS update files; it didn't work with the already uploaded _vbios_5700U.bin_ file from isc30's repo.

> The extraction steps I followed with no issues nor changes are detailed in **Dump iGPU VBIOS** section of the [boxmox: ASRock 4x4 BOX 5800U + JBOD + Proxmox](https://www.reddit.com/r/homelab/comments/11l0s5j/boxmox_asrock_4x4_box_5800u_jbod_proxmox/) guide.
>
> I highly recommend to read this guide, as it is very useful to understand the entire process and what do you need to do in next steps.

1. Download the official BIOS update files from your manufacturer (for Beelink SER5 PRO Ryzen 7 5700U: [official download link](https://dr.bee-link.cn/?dir=uploads%2FSER%2FSER5700%2FBIOS))

In my case, Beelink BIOS update files for 5700U were in a .zip file with the following structure:

```
SER5_5700
└── BIOS
    └── SER5H507
        ├── AfuEfix64.efi
        ├── EFI
        │   └── BOOT
        │       ├── BOOTX64.efi
        │       ├── Startup.nsh
        │       └── bootia32.efi
        ├── SER5H507.bin        --> this is the BIOS file I needed as input
        └── flash.nsh
```

2. Extract the VBIOS from the BIOS file:

```bash
./vbiosfinder extract ./SER5_5700/BIOS/SER5H507/SER5H507.bin
```

3. Upload it to your Proxmox host with `scp` at `/usr/share/kvm`:

```bash
scp {path to VBiosFinder folder}/output/vbios_1002_164c_1.rom {proxmox_user}@{proxmox_host_ip}:/usr/share/kvm
```

## Guest Ubuntu GPU Driver Installation

Install AMD drivers in the Ubuntu VM (I used latest version available at the time of writing; forked guide used `6.0.2`):

```bash
wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | gpg --dearmor | sudo tee /etc/apt/keyrings/rocm.gpg &gt; /dev/null
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/amdgpu/latest/ubuntu jammy main" | sudo tee /etc/apt/sources.list.d/amdgpu.list
sudo apt update
sudo apt install amdgpu-dkms
reboot
```

## Verification

Verify GPU is working in the VM with:

```
~$ lspci -nn | grep -e 'AMD/ATI'
01:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Lucienne [1002:164c] (rev c1)
```

```
~$ lspci -knns 01:00
01:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Lucienne [1002:164c] (rev c1)
	Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Lucienne [1002:0123]
	Kernel driver in use: amdgpu
	Kernel modules: amdgpu
```

```
~$ ls /dev/dri
by-path/  card0  renderD128
```

## Sources

Original sources:

-  https://www.bilibili.com/video/BV11d4y1G7Nk/
-  [matt22207's notes](https://gist.github.com/matt22207/bb1ba1811a08a715e32f106450b0418a) on Proxmox with 5700G APU GPU PCI Passthrough
-  [isc30's repo](https://github.com/isc30/ryzen-7000-series-proxmox) with vbios rom files

Additional resources:

-  [dioden94's comment on Reddit](https://www.reddit.com/r/Proxmox/comments/st7zlv/comment/kxfio6x/) about machine type switching
-  [boxmox: ASRock 4x4 BOX 5800U + JBOD + Proxmox](https://www.reddit.com/r/homelab/comments/11l0s5j/boxmox_asrock_4x4_box_5800u_jbod_proxmox/)
-  [The Ultimate Beginner's Guide to GPU Passthrough](https://www.reddit.com/r/homelab/comments/b5xpua/the_ultimate_beginners_guide_to_gpu_passthrough/)
-  [i12bretro's guide](https://i12bretro.github.io/tutorials/0650.html)
