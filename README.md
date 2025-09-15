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
hostpci0: 03:00.0,pcie=1,x-vga=1
hostpci1: 03:00.1,pcie=1
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
