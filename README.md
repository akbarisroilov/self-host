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

## GitLab

https://docs.gitlab.com/install/package/
