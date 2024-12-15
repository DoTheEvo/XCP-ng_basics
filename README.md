# XCP-ng

###### guide-by-example

![logo](https://i.imgur.com/FBH2aII.png)

# Purpose & Overview

A hyperviso rbased around Xen to host virtual machines.
An alternative to vmware esxi, or proxmox, or hyper-v.
Opensource.


A virtualization platform build around
[Xen](https://en.wikipedia.org/wiki/Xen) a type 1 **hypervisor**.<br>
An alternative to vmware esxi/vsphere, or hyper-v, or proxmox.

Xen itself is now developed by **Linux Foundation** with backing from
intel, amd, arm, aws, google, alibaba cloud, huawei,... 
previously Citrix stood behind the project before opening it.

### the components

* **xen** - the hypervisor, developed by Linux Foundation
* **xcpng** - a single purpose linux distro preconfigured with xen
* **XO** - Xen Orchestra - an opensource web interface for centralized managment
  of xcpng servers, can be deployed as a container or a virtual machine
* **XOA** - Xen Orchestra Appliance - a paid version of XO with full support and some
  extra features like [XOSTOR](https://vates.tech/xostor/) through webGUI
* **XCP-ng Center** - a windows desktop application, works but abandonware
  after citrix cut XenCenter devleopment

# Xen Orchestra

<!-- ![diagram](https://i.imgur.com/EuRpfe1.png) -->

![diagram](https://i.imgur.com/MumtDzU.png)

Unlike esxi or proxmox, to which you connect easily after fresh install,
xcpng expects you to have XO deployed somewhere and mange xcpng servers with it.<br>
They are adding XO Lite, but one can as well skip it
and from the get-go plan to deploy Xen Orchestra somewhere for the managment
of all xcpng servers.

Options are:

* **docker** container - if you have docker host you can easily deploy XO there
* **a VM** - in any hypervisor spin up a debian and run a install script
* **XCP-ng Center** - windows desktop application


<details>
<summary><h3>XO running as a docker container</h3></summary>

* [ronivay github](https://github.com/ronivay/xen-orchestra-docker)

`compose.yml`
```yml
services:

  xen-orchestra:
    image: ronivay/xen-orchestra:latest
    container_name: xen-orchestra
    hostname: xen-orchestra
    restart: unless-stopped
    env_file: .env
    stop_grace_period: 1m
    expose:
        - "80"                  # webGUI
    cap_add:                    # capabilities are needed for NFS/SMB mount
      - SYS_ADMIN
      - DAC_READ_SEARCH
    # additional setting required for apparmor enabled systems. also needed for NFS mount
    security_opt:
      - apparmor:unconfined
    volumes:
      - ./xo_data:/var/lib/xo-server
      - ./redis_data:/var/lib/redis
    # these are needed for file restore.
    # allows one backup to be mounted at once which will be umounted after some minutes if not used (prevents other backups to be mounted during that)
    # add loop devices (loop1, loop2 etc) if multiple simultaneous mounts needed.
    devices:
     - "/dev/fuse:/dev/fuse"
     - "/dev/loop-control:/dev/loop-control"
     # - "/dev/loop0:/dev/loop0"

networks:
  default:
    name: $DOCKER_MY_NETWORK
    external: true
```

`.env`
```bash
# GENERAL
DOCKER_MY_NETWORK=caddy_net
TZ=Europe/Bratislava

# XO
HTTP_PORT=80
```

</details>

<details>
<summary><h3>XO virtual machine script</h3></summary>

using script to install on debian 12

https://forums.lawrencesystems.com/t/how-to-build-xen-orchestra-from-sources-2024/19913

</details>


<details>
<summary><h3>windows desktop app - XCP-ng Center</h3></summary>

[Windows executable tool.](https://github.com/xcp-ng/xenadmin)

Vibe is that its kinda abandoned. 

</details>

# Installation XCP-ng

![xcpng-statusscree](https://i.imgur.com/iiZlGWa.png)

# Basics 

### First XO login

`admin@admin.net` // `admin`

* add server
  * label whatever
  * ip address 
  * root / password
  * check - allow unauthorized certificates

### Updates

* Home > Pools > your-host > Patches

### Create DVD ISO storage

* New > Storage > Select host
* Name: `ISOs`
* Select storage type: `ISO SR: Local`
* Path: `/media`
* Create

afterwards to upload an iso

* Import > Disk 
* To SR: `ISOs`

The storage is created on the root partition.<br>
This one can be just 18GB if that disk was also selected for
storing VMs.

### Storage

### GENERAL

`xsconsole`

### Spin a new VM

Arch Linux 

* New > VM
* Generic Linux UEFI
* vCPU, RAM, 1socket
* iso
* network default
* disks - change name, size
*  

### guest additions

#### Windows

https://xcp-ng.org/docs/guests.html#windows

* github ng opernsoruce version<br>
  https://github.com/xcp-ng/win-pv-drivers/releases
* citrix closed source<br>
  Advanced > Manage Citrix PV drivers via Windows Update<br>
  but still requires hunt for agent app or something

I use the version `8.2.2.200-RC1` from github
* download ISO; run setup.exe; restart

#### Arch

* `yay xe-guest-utilities-xcp-ng`
* `sudo systemctl enable --now xe-linux-distribution.service`


# Passthrough

![passthrough-pic](https://i.imgur.com/nLNT9iH.gif)

When you want to give a virtual machine direct full hardware access to some device.<br>
Since v8.3 it is very easy to do through webGUI.

#### intel igpu passthrough

* On the server host
  * Home > Hosts > the-host > Advanced > PCI Devices<br>
    Enable slidder next to VGA compatible controller
  * Reboot the host, go check if the slider is on
* On the Virtual Machine
  * Home > VMs > the-vm > Advanced ><br>
    At the end a button - `Attach PCIs`, there pick the igpu listed.

In VM you can check with `lspci | grep -i vga`

Tested with jellyfin and enabled transcoding,
monitored with btop and intel_gpu_htop.

<details>
<summary><h5>The old way - cli passthrough</h5></summary>

[lawrance video](https://www.youtube.com/watch?v=KIhyGvuCDcc)

* ssh in on to xcpng host
* `lspci -D` list the devices that can be passthrough
* pick the device you want, note the HW address at the begining,
  in this case it was `0000:00:02.0`
* hide the device from the system<br>
  `/opt/xensource/libexec/xen-cmdline --set-dom0 "xen-pciback.hide=(0000:00:02.0)"`
  * be aware, the command is overrwriting the current blacklist,
    so for multiple devices it would be<br>
    `/opt/xensource/libexec/xen-cmdline --set-dom0 "xen-pciback.hide=(0000:00:02.0)(0000:00:01.0)"`
* reboot the hypervisor
* can use command `xl pci-assignable-list` to check device that can be passthrough    

After reboot of the VM I had igpu in and successfully used it in jellyfin.

### amd igpu passthrough

`udevadm info --query=all --name=/dev/dri/renderD128`
`dmesg | grep -i amdgpu` - if loaded correctly

</details>

#### opnsense as VM in xcpng

[Setup here](https://github.com/DoTheEvo/selfhosted-apps-docker/tree/master/opnsense#xcp-ng)


# Backups 

[lawrance video](https://youtu.be/weVoKm8kDb4?si=3YgoQUvYx4I2a_7u)

So far seems 

https://xen-orchestra.com/docs/backups.html#interface

Veeam Support - [seems](https://forums.veeam.com/veeam-backup-replication-f2/xcp-ng-support-t93030-60.html#p531802)
theres a prototype and positive response from veeam developers.

## create a remote

* Settings > Remotes
* NFS or Local or SMB
* ...

## Create a backkup job

* Backup > New > VM Backup & Replication
