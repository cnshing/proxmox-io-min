
# Proxmox IO minimization guide

## References
[Proxmox on a SD card](https://github.com/iacsvrn/ru_proxmox/blob/master/proxmox-on-flash-drive.md)

[log2ram](https://github.com/azlux/log2ram)

[Minimizing SSD wear through PVE configuration changes](https://forum.proxmox.com/threads/minimizing-ssd-wear-through-pve-configuration-changes.89104/)

[Proxmox RRD Data Sources](https://forum.proxmox.com/threads/list-of-rrd-graph-datasource-ds.106667/)

[rrdcached](https://oss.oetiker.ch/rrdtool/doc/rrdcached.en.html)

## Assumptions
__Note:  It is not required to sastify all assumptions for the guide to work. Some steps will refer to these assumptions to inform you of whether or not it is needed for _your_ specific use-case.__
1. A fresh 7.2 Proxmox Installation
2. Single node, non-cluster enviornment
3. No time-sensitive read file applications
5. No sensitive applications for RRD Data
6. At least 200+ MB Ram

## Optional

* IO performance measurement tool (iostat)

## Warning
This guide is experimentally conducted where IO usage was measured throughout the minimization process. What may have worked for me may not have worked for you. I will provide some supplementary comments that may be helpful to you but may not neccesarily apply to every single Proxmox instance. 

## Steps

Steps 1,2,.. are used to reduce the total amount of write operations to the Proxmox disk.

1) Enable noatime to your root partition and remove the swap partition at `/etc/fstab`:
```
/dev/pve/root / ext4 errors=remount-ro,noatime 0 1
UUID=XXXX-XXXX /boot/efi vfat defaults 0 1
proc /proc proc defaults 0 0
```
noatime will disable any writing updates to the access times for the reads of any file.
Removing the swap partition will prevent writes to your Proxmox disk.
If your Proxmox VM relies on accurate read access times of certain files, it is best for you to remove `,noatime` from your `/etc/fstab` for Assumption [3](#Assumptions)

__WARNING: Misuse of /etc/fstab may leave your operating system unbootable. In those cases be sure to back up`/etc/fstab` and restore them if anything goes awry.___

2) Move some common write directories to virtual memory by appending these lines at `/etc/fstab`
```
tmpfs /tmp tmpfs defaults,nodev,nosuid 0 0
tmpfs /var/tmp tmpfs defaults,nodev,nosuid 0 0
tmpfs /var/lib/rrdcached tmpfs defaults,nodev,nosuid 0 0
```
Whenever new information for certain statistics like network traffic, cpu/disk usage, etc, the data is updated to rrdcache, basically a write-buffer which then updates the corresponding RRD data after a period of time. If the virtual filesystem is flushed, then any new updates will be lost. Remove `tmpfs /var/lib/rrdcached tmpfs defaults,nodev,nosuid 0 0` from your `/etc/fstab` for Assumption [5](#Assumptions) if statistical data is important to you.

__WARNING: Misuse of /etc/fstab may leave your operating system unbootable. In those cases be sure to back up`/etc/fstab` and restore them if anything goes awry.___

3) Disable some PVE cluster-related services:

```
systemctl stop pve-ha-lrm
systemctl stop pve-ha-crm
systemctl stop pvesr.timer
systemctl stop corosync.service
systemctl disable pve-ha-lrm
systemctl disable pve-ha-crm
systemctl disable pvesr.timer
systemctl disable corosync.service
```

These are generally mechanisms to provide redudancy for clusters, such as storage replication between nodes(pvesr), network redudancy(corosync), and some other components that I don't really know too much about. Unfortunately they are also known to apply some heavy write operations. Users who are operating in a clustered enviornment should not disable these services for Assumption 2(#Assumptions)

3) Install and configure log2ram:
```
echo "deb [signed-by=/usr/share/keyrings/azlux-archive-keyring.gpg] http://packages.azlux.fr/debian/ bullseye main" | sudo tee /etc/apt/sources.list.d/azlux.list
sudo wget -O /usr/share/keyrings/azlux-archive-keyring.gpg  https://azlux.fr/repo.gpg
sudo apt update
sudo apt install log2ram
```
Modify the `SIZE` parmaeter at `/etc/log2ram.conf`:
`SIZE=200M`

and reboot to enable log2ram:
`reboot now`

log2ram prevents log-related write operations to the root partition and instead redirects them to the ram disk by mounting /var/logs to ram. Looking at the /var/logs directory I've found that most files that didn't exceed 50MB, so I arbitrarly allocated 200MB for /var/logs.

Depending on your avaliable memory size, 200 MB may be too much (Assumption 6) and depending on how old your Proxmox installation may be, /var/logs may have grown too large to properly hold 200 MBs worth of data(Assumption 1). I've personally encountered instances where I was unable to access the shell through the web-gui after setting too low a configuration, and had to ssh into Proxmox instead.

4) Set maximium disk space usage by journal

Modify the following line at `/etc/systemd/journald.conf`
```
SystemMaxUse=40M
```
If your existing /var/logs/journal may be too large after running the Proxmox Enviornment for a long time(Assumption 1), you may need to increase the disk space or delete the journal logs. Again, configuring both the size of log2ram and journald will be dependent on your current system.

While the previous 3 steps did contribute to lowering IO usage somewhat, setting SystemMaxUsed somehow dropped the usage all the way down from 100 KB/s average to 9 KB/s and a staggering _3 KB/s_ for a previously running 2 month Proxmox Laptop, and a newly fresh server enviornment, respectively.

Laptop at Idle, No running VMs:
![Laptop](https://github.com/cnshing/proxmox-io-min/blob/main/laptop.PNG)


Fresh Server at Idle, No running VMs, local-lvm removed:
![Server](https://github.com/cnshing/proxmox-io-min/blob/main/server.PNG)








