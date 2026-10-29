# Samba File-Server on local LXC-Container

---

## Overview


* **[Proxmox](#Proxmox)**
  * [adjust firewall](#Firewall-Configuration)

* **[Container-Setup](#Container-Setup)**
  * [download template](#Templates)
  * [create container](#Container-configuration)
  * [adjust firewall](#Adjust-Firewall)

* **[Debian LXC-Container](#Debian-Container)**
    * [Samba system user](#Samba-User-Setup)
    * [Samba download and setup](#Samba-Server-Download-and-Setup)
    * [Samba share access](#Accessing-the-server)


&nbsp;

---
---




## Proxmox
The Samba fileserver will be running on an LXC-Container\
spawned from a local Proxmox setup.

### Firewall Configuration

- **Datacenter** 
  - Firewall should be *ACTIVATED*
  - Input-Policy *DROP*
  - Output-Policy *ACCEPTED*
  - Forward-Policy *ACCEPTED*
  - No additional Rules

- **Proxmox Node**
  - Firewall should be *ACTIVATED*
  - SMURF filter *ACTIVATED*
  - TCP flags filter *ACTIVATED*

It is important that both higher level firewalls are activated,\
for the lower level container firewall settings to take effect.

---

## Container Setup

### Templates
Templates can be downloaded directly through the proxmox webinterface.\
Look for the storage that contain **CT Templates** , most likely **local** or **local-lvm**.\
Within the **CT Templates** tab we open the **Templates** menu and pick one to download.


### Container configuration
I have chosen a lightweight **debian13-standard** LXC-Container.\
Now we create the Container via the **Create CT** Button on the top right corner

Choose a reasonable **ID**, **Hostname**, and **Password** and set the container to **Unprivilaged**.\
Select your **Template** from the storage you downloaded it to.

For a container only running a simple samba file-server the follwoing\
resource configuration should be **more than enough** :

|Resource|Value|
|-------|--------|
|Memory | 2.048 MB|
|Swap | 1.024 MB |
|Cores | 2 |
|Root Disk | 8 GB|
|mp0|500 GB|

* **mp0** is the **Mount-Point** for an additional disk that we add to our setup,\
it will hold our **files** in a disk image separated from our container image.
* **Path** points to the location our disk will be mounted at inside our container,\
pick something like **/mnt/samba** for example
* **Uncheck** the **Backup** checkbox for the Mount-Point, or all files
will be\
duplicated by our proxmox **backups** and may fill up a lot of space!\
( unless you want this behavior )


Under Network, enter a **static IP** in **CIDR** notation, matching the *Container ID* is recommended :
```
Container ID 201 | Container IP 192.168.1.201/24
```
**Gateway** is the routers IP, only use IPv4.\
**DNS** configuration can be left at default.\
**Firewall** should be activated for the network interface

***Create the Container if summary looks okay***

### Adjust Firewall
In the Container Firewall we need to adjust a few things and define\
some **Firewall-Rules**, for our server to stay save.\
Inside the proxmox web interface we select the container and go to\
**Firewall**.

- Firewall Options
  - Firewalls hould be *ACTIVATED*
  - Input-Policy *DROP*
  - Output-Policy *DROP*

We **drop** all in- and outcoming traffic for our container\
and in our **Firewall-Rules** we add the following rules :

|Type|Action|Macro|Protocol|Source|S.Port|Destination|D.Port|
|----|------|-----|--------|------|------|-----------|------|
|out|Accept|HTTPS|-----|Container-IP|-----|-----|-----|
|out|Accept|HTTP|-----|Container-IP|-----|-----|-----|
|out|Accept|DNS|-----|Container-IP|-----|-----|-----|
|in|Accept|SMB|-----|-----|-----|Network-IP|Container-IP|

The outbound ports are used mainly for system updates.\
The inbound rule with the **SMB** macro is used by the samba-protocol.

I have **additionaly** opened the following ports locally for a custom backup script:

|Type|Action|Macro|Protocol|Source|S.Port|Destination|D.Port|
|----|------|-----|--------|------|------|-----------|------|
|out|Accept|Ping|-----|Container-IP|-----|Network-IP|-----|
|out|Accept|SSH|-----|Container-IP|-----|Network-IP|-----|

###### *Example: Container-IP: 192.168.1.201 | Network-IP: 192.168.1.0/24

---

## Debian Container

Now its time to start our newly created **Debian-Container.** via the **Start** button.\
After a short boot period, we enter the **Console** of our Container.\
Login via **root** and enter the configured **Password**. 

Inside the container terminal we first run

```
root:~# apt update && apt upgrade -y
```

To get our container to an **up to date** state.\
Consider activating **unattended-upgrades** to keep the container safe and sound.

### Samba User Setup

First of all lets create a dedicated **user account** for our samba service
```
root:~# useradd -r -M -s /bin/false samba_user
```
+ **-r** makes the account a restricted **system account**
+ **-M** supresses the creation of a home directory
+ **-s /bin/false** deactivates login for the user

### Samba Server Download and Setup

Download the **samba server packages** with :
```
root:~# apt install samba
```
After the installation check if the service is running / installed properly :
```
root:~# systemctl status smbd.service

root:~# systemctl status nmbd.service
```
**smbd.service** is the samba daemon, responsible for file sharing functionality.\
**nmbd.service** is another core component of samba, responsible for the network functionality.

Now we add our samba_user to the **samba service** and give it a password\
which will be needed if we try to access our file-share :
```
root:~# smbpasswd -a samba_user
New SMB password: **********
Retype new SMB password: **********
```
We make our samba_user the owner of the **/mnt/samba** directory :
```
root:~# chowm -R samba_user:samba_user /mnt/samba
```

Our next step is to **configure** the directory that we want to share.\
In this example its the whole **mp0** disk image that is mounted at **/mnt/samba**.

The samba configuration file can be found in **/etc/samba/smb.conf**\
Let's first create a backup of the file, then open it in our editor of choice :
```
root:~# cd /etc/samba
root:/etc/samba# cp smb.conf smb.conf.bak
root:/etc/samba# vim smb.conf
```
The whole top section is used for samba configuration.

Deleted the whole section *below* **Share Definition**.\
It contains rules for printer sharing and so on,\
but we only want to serve **/mnt/samba** via samba.\

Below **Share Definition** we edit the following :

```
#============== Share Definition ==============

[samba_share]
   path = /mnt/samba
   browseable = yes
   writable = yes
   valid users = samba_user
   create mask = 0660
   directory mask = 0770
```
+ [samba_share] - will be the name displayed to the network
+ browseable - defines if logins are allowed to browse the directory
+ writeable - defines if logins are allowed to write ro the directory
+ valid users - is the login name (with password) and owner of our share
+ create mask - defines the rights for newly created files
+ directory mask - defines the rights for newly created directorys

We save the file and check its syntax with the command **testparm**.\
The output will show us if the smb.conf is valid and summerize our parameters :
```
root:~# testparm
```

Now we **reload systemd** - **restart** our services and **enable** them,\
so they are launched automatically after system boot :
```
root:~# systemctl daemon-reload
root:~# systemctl restart smbd.service nmbd.service
root:~# systemctl enable smbd.service nmbd.service
```

### Accessing the server

Our configured share can now be accessed from the local network via :
+ **smb://192.168.178.202** from our file browser in linux
+ **\\\192.168.178.202** from our windows explorer

If we want the samba share to be automaticaly displayed in he **Network** tab,\
we need some additional network discovery services :

+ for linux and mac install these packages on the server :
  ```
  root:~# apt install avahi-daemon avahi-utils
  ```
+ for windows install this package on the server :
  ```
  root:~# apt install wsdd2
  ```

### Done! 
***The samba share should be accessible now if username and password are provided.***