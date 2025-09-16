# self-host

## ProxMox
### Enable IOMMU
Edit GRUB
```
nano /etc/default/grub
```
Change "GRUB_CMDLINE_LINUX_DEFAULT=" to this line below exactly

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
```
Run the command update-grub to finalize changes
```
update-grub
```
Reboot Proxmox
Verify
```
dmesg | grep -e DMAR -e IOMMU
```
Should see something like:
```
DMAR: IOMMU enabled
```

### VFIO modules
Add VFIO modules:
```
nano /etc/modules
```
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
Blacklist NVIDIA drivers:
```
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf
```

### Identify your GPU
```
lspci -nn
```
Should see something like
```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation ...
01:00.1 Audio device [0403]: NVIDIA Corporation ...
```

### Bind GPU to VFIO
```
nano /etc/modprobe.d/vfio.conf
```
Add:
```
options vfio-pci ids=10de:1b81,10de:10f0
```
(Replace with your GPU and its audio function IDs.)
Update initramfs:
```
update-initramfs -u
reboot
```
Verify binding:
```
lspci -nnk -d 10de:1b81
```

### Windows VM
```
/etc/pve/qemu-server/101.conf
```
```
hostpci0: 03:00.0,pcie=1,x-vga=1
hostpci1: 03:00.1,pcie=1
```
```
bios: ovmf
boot: order=scsi0;ide0
cores: 6
cpu: x86-64-v2-AES
efidisk0: local-lvm:vm-101-disk-0,efitype=4m,pre-enrolled-keys=1,size=4M
hostpci0: 0000:03:00.0,pcie=1,x-vga=1
hostpci1: 0000:03:00.1,pcie=1
hostpci2: 0000:00:14
hostpci3: 0000:00:1b
ide0: local:iso/virtio-win-0.1.271.iso,media=cdrom,size=709474K
machine: pc-q35-10.0
memory: 8192
meta: creation-qemu=10.0.2,ctime=1757839618
name: windows-10
net0: e1000=BC:24:11:6A:E8:2D,bridge=vmbr0,firewall=1
numa: 0
onboot: 1
ostype: win11
scsi0: local-lvm:vm-101-disk-1,iothread=1,size=256G
scsihw: virtio-scsi-single
smbios1: uuid=fc7bc40a-e0dd-4461-9257-6a9e595f914f
sockets: 2
tpmstate0: local-lvm:vm-101-disk-2,size=4M,version=v2.0
vmgenid: 93a7925b-8581-4611-a20d-9d9a225a951f
```

## GitLab

https://docs.gitlab.com/install/package/

```
/etc/gitlab/gitlab.rb
```
```
external_url 'http://192.168.0.202:4376'

nginx['listen_https'] = false
nginx['listen_port'] = 4376
nginx['redirect_http_to_https'] = false
letsencrypt['enable'] = false
```
