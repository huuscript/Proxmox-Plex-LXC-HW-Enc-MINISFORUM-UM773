# Proxmox Plex LXC with HW Encoding on MINISFORUM-UM773 Lite Ryzen7 7735HS
After several weeks of trial and error, I successfully enabled hardware encoding on a Plex Proxmox container using the Minisforum UM773 Lite with a Ryzen 7 7735HS. Surprisingly, it was easier than the conventional passthrough method that I am so used to, going into it with previous expectation of PCI and GPU passthrough that got me into trouble.

# Prequesite 
Assume you already have Proxmox 8 installed.

# BIOS Setting
First, access the BIOS and disable IOMMU. Disabling it is crucial, as enabling it prevents Proxmox from installing the necessary drivers, resulting in an empty /dev/dri directory, which is required for Plex LXC passthrough.
# Getting the GPU PCI ID
We need to get the GPU PCI ID using this command:

`lspci -nn | grep -e ‘AMD/ATI’`

The result should get something like this

```
34:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Rembrandt [Radeon 680M] [1002:1681] (rev 0a)
34:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Rembrandt Radeon High Definition Audio Controller [1002:1640]
```
Note GPU: 1002:1681

Audio Device: 1002:1640

# Adding the PCI ID / Preload to vfio.conf
Add options vfio-pci ids=1002:1681,1002:1640 to /etc/modprobe.d/vfio.conf

`nano /etc/modprobe.d/vfio.conf`

and add the following now it should look like this

```
options vfio-pci ids=1002:1681,1002:1640
softdep radeon pre: vfio-pci
softdep amdgpu pre: vfio-pci
softdep snd_hda_intel pre: vfio-pci
```

# Enable vfio in kernel modules
Add the kernel modules to enable vfio to nano /etc/modules

```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

# Edit/Update the GRUB
At this stage, most instructions suggest adding the IOMMU command, but I found this doesn’t work. Instead, leave it at the default setting. Ensure it appears as shown below, and if necessary edit it, and update GRUB.

```
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX=""
```

update-grub 

update-initramfs -u -k all

# Checking the /dev/dri 
Reboot, and check /dev/dri directory should not be empty.

```
root@PROXMOX:~# ls -l /dev/dri
total 0
drwxr-xr-x 2 root root         80 Aug  5 23:24 by-path
crw-rw---- 1 root video  226,   0 Aug  5 23:24 card0
crw-rw---- 1 root render 226, 128 Aug  5 23:24 renderD128
```

# Installing Plex via TTeck scripts
Install plex from here https://tteck.github.io/Proxmox/#plex-media-server-lxc with the following script

`bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/ct/plex.sh)"`
 
Choose Advance and yes to privilege, as install as privilege will add passthrough mount and groups. Look in

`nano /etc/pve/lxc/100.conf`

It should look something like this (replace 100 with your own container).

```
arch: amd64
cores: 2
features: fuse=1,nesting=1
hostname: plex
memory: 2048
net0: name=eth0,bridge=vmbr0,hwaddr=XX:XX:XX:XX:XX:XX,ip=dhcp,type=veth
onboot: 0
ostype: ubuntu
rootfs: local:100/vm-100-disk-0.raw,size=8G
swap: 512
tags: proxmox-helper-scripts
lxc.cgroup2.devices.allow: a
lxc.cap.drop:
lxc.cgroup2.devices.allow: c 188:* rwm
lxc.cgroup2.devices.allow: c 189:* rwm
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 243:* rwm
lxc.cgroup2.devices.allow: c 509:* rwm
lxc.cgroup2.devices.allow: c 29:0 rwm
lxc.mount.entry: /dev/serial/by-id   dev/serial/by-id   none bind,optional,create=dir
lxc.mount.entry: /dev/ttyUSB0        dev/ttyUSB0        none bind,optional,create=file
lxc.mount.entry: /dev/ttyUSB1        dev/ttyUSB1        none bind,optional,create=file
lxc.mount.entry: /dev/ttyACM0        dev/ttyACM0        none bind,optional,create=file
lxc.mount.entry: /dev/ttyACM1        dev/ttyACM1        none bind,optional,create=file
lxc.mount.entry: /dev/fb0            dev/fb0            none bind,optional,create=file
lxc.mount.entry: /dev/dri            dev/dri            none bind,optional,create=dir
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
lxc.mount.entry: /dev/vfio           dev/vfio           none bind,optional,create=dir
lxc.mount.entry: /dev/dvb            dev/dvb            none bind,optional,create=dir 0 0
```

You don’t need to passthrough every device this way. Since I use it solely as a headless Plex box, I leave it as is. This way, I don’t have to edit the file each time I plug in a device, such as a TV tuner.

# Adding permission for Plex user
Open the plex container console and add plex user to the video card group for permission. Add one at a time to watch for errors.

```
usermod -a -G video plex
usermod -a -G render plex
usermod -a -G input plex
usermod -a -G ssl-cert plex
```

# Stream and confirm HW Enconding
Reboot your system and stream content to a device that doesn’t support direct play to force transcoding. Check your Plex dashboard; it should display “transcode (HW)” next to the playing content. Your CPU usage should be minimal. If “(HW)” is missing, your CPU usage will spike to 100%, indicating that hardware transcoding is not working.
![HW Encode Success](IMG_1167.jpeg)

# Limitation
Premium Plex require for ennabling HW Encoding.
This is a write up for enabling HW Encoding for Plex container, it is not for PCI/GPU passthrough,
there is an excellent write up for GPU passthrough for Windows here
https://github.com/isc30/ryzen-7000-series-proxmox/tree/main

# Credits
During my week long search for a solution i have been to many sites and forum. I Can not list them all here.
My original inspiration is from here
https://www.reddit.com/r/MiniPCs/s/nhsCRhGOQD

For those who use Proxmox, TTeck got an excellent collection of scripts from here
https://tteck.github.io/Proxmox/



