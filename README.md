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

## Debian
### set static ip
```
/etc/network/interfaces
```
```
auto ens18
iface ens18 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
```
```
sudo systemctl restart networking
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


## Samba Storage
### pass through a disk
Identify the Disk
```
ls -l /dev/disk/by-id/
```
pass via proxmox command
```
qm set <VMID> -scsi1 /dev/disk/by-id/ata-Samsung_SSD_860_500GB_SERIAL
```

### prepare mount directory
```
sudo mkdir -p /srv/shared
```
```
sudo mount /dev/sdb1 /srv/shared
```

###Make it persistent (auto-mount at boot):
Get UUID:
```
sudo blkid /dev/sdb1
```
Example output:
```
/dev/sdb1: UUID="1234-ABCD" TYPE="ext4"
```
Edit fstab:
```
sudo nano /etc/fstab
```
Add a line:
```
UUID=1234-ABCD   /srv/shared   ext4   defaults   0   2
```

### install and setup samba
```
sudo apt update && sudo apt install samba -y
```
update samba conf
```
sudo vim /etc/samba/smb.conf
```
Add:
```
[sambashare]
   path = /srv/shared
   browseable = yes
   writable = yes
   guest ok = no
   create mask = 0775
```
Create no login user if needed
```
sudo useradd -M -s /sbin/nologin smbuser
```
Give user samba credentials
```
sudo smbpasswd -a smbuser
```
Give permissions:
```
sudo chown -R smbuser:smbuser /srv/shared
```
Restart Samba:
```
sudo systemctl restart smbd
```

### connect from device
Windows Explorer:
\\<samba-vm-ip>\sambashare
Linux:
//<samba-vm-ip>/sambashare
smbclient //samba-vm-ip/sambashare -U smbuser
