# Teamspeak 3 Server on local LXC-Container

---

## Overview


* **[Proxmox](#Proxmox)**
  * [adjust firewall](#Firewall-Configuration)

* **[Container-Setup](#Container-Setup)**
  * [download template](#Templates)
  * [create container](#Container-Setup)
  * [adjust firewall](#Adjust-Firewall)

* **[Debian LXC-Container](#Debian-Container)**
    * [Teamspeak 3 system user](#TS3-User-Setup)
    * [Teamspeak 3 server download](#TS3-Server-Download-and-Setup)
    * [Teamspeak 3 server config](#TS3-Server-Configuration)
    * [Teamspeak 3 daemon](#Create-a-Service)
    * [Teamspeak 3 auto updates](#Auto-Updates)
    * [Teamspeak 3 user lockdown](#User-Lockdown)

* **[Router Configuration](#Router-Config)**
  * [open ports](#Open-necessary-Ports)
  * [server address](#Server-Address)

&nbsp;

---
---




### Proxmox
The Teamspeak 3 server will be running on an LXC-Container\
spawned from a local Proxmox setup.

#### Firewall Configuration

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

### Container Setup

#### Templates
Templates can be directly downloaded via the designated proxmox drive\
that **CT Templates** are stored in, in my case can be found in the **local** storage\
within **CT Templates**. Pick one from the **Templates** Tab to download


#### Container Setup
I have chosen a lightweight **debian13-standard** LXC-Container.\
Now we create the Container via the **Create CT** Button on the top right corner

Choose a reasonable **ID**, **Hostname**, and **Password** and set the container to **Unprivilaged**.\
Select your **Template** from the storage you downloaded it to.

For a container only running a simple Teamspeak 3 Server the follwoing\
resource configuration should be **more than enough** :

|Resource|Value|
|-------|--------|
|Memory | 1.00 GB|
|Swap | 512 MB |
|Cores | 2 |
|Root Disk | 8 GB|

In Network enter a **static IP**, in **CIDR** notation, matching the *Container ID* is recommended :\
`Container ID 201 | Container IP 192.168.1.201/24`\
**Gateway** is the routers IP, only use IPv4.\
**DNS** configuration can be left at default.\
**Firewall** should be activated for the network interface

***Create the Container if summary looks okay***

#### Adjust Firewall
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
|in|Accept|-----|TCP|-----|-----|Container-IP|10011|
|in|Accept|-----|TCP|-----|-----|Container-IP|30033|
|in|Accept|-----|UDP|-----|-----|Container-IP|9987|

The outbound ports are used mainly for system updates,\
the inbound ports are used by the Teamspeak server itself.

- 10011 TCP TS3-Server Query
- 30033 TCP TS3-File Transfer
- 9987 UDP TS3-Voice Transfer

SSH and additional protocols can be opend if needed, but should be\
limited to the local network like 192.168.1.0/24, or specific machines or a VPN.

---

### Debian Container

Now its time to start our newly created **Debian-Container.** via the **Start** button.\
After a short boot period, we enter the **Console** of our Container.\
Login via **root** and enter the freshly configured **Password**. 

Inside the container terminal we first run

`root:# apt update && apt upgrade -y`

To get our container to an **up to date** state.\
Consider activating **unattended-upgrades** to keep your container safe and sound.

We also need to install the following packages for later use :
```
root:~# apt install rsync wget unzip bzip2 lsb-release gdb mailutils
```

#### TS3 User Setup
Now we create a dedicated user account for our teamspeak 3 server to keep things organized and separated from our root user.

```
root:# useradd -m -r -s /bin/bash teamspeak_3
```

+ **useradd** creates a new user\
+ **-m** creates a home directory\
+ **-r** creates a system-user used for daemons and services\
+ **-s** sets our shell to bash (for now)

#### TS3 Server Download and Setup
With our new system user configured we **log into** teamspeak_3,\
make sure we are inside our user's home directory\
and download the latest Teamspeak 3 **Server** package for Linux.
```
root:# su - teamspeak_3

teamspeak_3:~$ pwd

teamspeak_3:~$ wget -4 https://files.teamspeak-services.com/releases/server/3.13.8/teamspeak3-server_linux_amd64-3.13.8.tar.bz2
```

Now the compressed archive is placed under /home/teamspeak_3.\
We **decompress** and **unpack** it in place using following tar command :
```
teamspeak_3:~$ tar xjf teamspeak3-server_linux_amd64-3.13.8.tar.bz2 
```

We **delete** the archive and **rename** the extracted directory 
into something shorter\
we can memorize better. Then we **enter** our new teamspeak_server directory.

```
teamspeak_3:~$ rm teamspeak3-server_linux_amd64-3.13.8.tar.bz2 

teamspeak_3:~$ mv teamspeak3-server_linux_amd64 teamspeak_server

teamspeak_3:~$ cd teamspeak_server

```

We need to accept the **Teamspeak 3 Server Licence** before we\
can start our server, otherwise we will get an error.\
We do so by simply creating a file named **.ts3server_license_accepted**\
and we make sure our **ts3server_startscript.sh** is **executable** :

```
teamspeak_3:~/teamspeak_server$ touch .ts3server_license_accepted
teamspeak_3:~/teamspeak_server$ chmod u+x ts3server_startscript.sh
```

Now we can **start** our server manually for the first time with :
```
teamspeak_3:~/teamspeak_server$ ./ts3server_startscript.sh start
```
&emsp;**IMPORTANT:** As soon as the server started up for the first time\
&emsp;it will print the **credencials** for the **serveradmin** only **once** !\
&emsp;The **Privilege Key** will identify you as the server administrator as you first log into the server.\
&emsp;The **ServerQuery credentials** are needed for administrative tasks.

&emsp;***Save them and keep them secret!***

The server should be **running** now, you can check its status with :
```
teamspeak3:~/teamspeak_server$ ./ts3server_startscript.sh status
```

#### TS3 Server Configuration

Next up we can use a **Teamspeak3 Client** to connect to our server,\
using the local network IP of our **Container** *192.168.1.201*.\
The server is not accessable from the internet, yet.

Use **serveradmin** as the username and the secured **credentials**\
to connect to the server. Your **Teamspeak Identity** will automaticaly\
be recorded as the **server administrator**.

You can change your **display name** now without loosing your admin rights.\
Set a **server password** for users to login with, change the **server name**\
and create some Channels as needed.

For administrative tasks like activating logs or changing the server config you will need your **serveradmin credentials**, not the **server password**.\
If you loose your admin password it can be reset with the command :

```
teamspeak3:~/teamspeak_server$ ./ts3server_minimal_runscript.sh serveradmin_password=NewPassWord
```
A new **Privilege Key** can be generated under the **rights** menu\
enty, if we are logged into our server as the administrator.

#### Create a Service

Now we will create a **daemon** or **service**, so our teamspeak server\
is started automatically every time the container boots or\
restart if something goes wrong.

As root we change to the directory /etc/systemd/system/ and\
create a regular file named **teamspeak3.service** :
```
root:~# cd /etc/systemd/system
root:~# touch teamspeak3.service
```
Open the file with an editor and enter the following lines :
```
[Unit]
Description=Teamspeak3_Server
After=network-online.target
Wants=network-online.target
ConditionPathExists=!/home/teamspeak_3/teamspeak3_server/.ts3updatescript.lock

[Service]
Type=forking
User=teamspeak_3
WorkingDirectory=/home/teamspeak_3/
ExecStart=/bin/bash /home/teamspeak_3/teamspeak3_server/ts3server_startscript.sh start
ExecReload=/bin/bash /home/teamspeak_3/teamspeak3_server/ts3server_startscript.sh restart
ExecStop=/bin/bash /home/teamspeak_3/teamspeak3_server/ts3server_startscript.sh stop
PIDFile=/home/teamspeak_3/teamspeak3_server/ts3server.pid
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```
The parameters mean the following:
+ **[Unit]**- (describes the service and conditions needed before start)
  + Description - (Describes the unit)
  + After - (to make sure our service starts **after** a target it depends on)
  + Wants - (defines services it that our service depends on)
  + ConditionPathExists - (only starts the service if a file exists)

+ **[Service]** - (The parameters for our own service are defines here)
  + Type - (defines how the service should be started)
  + User - (defines the user that sould own the service)
  + WorkingDirectory - (defines in what directory the service will be started)
  + ExecStart - (define the action when systemctl start service is run)
  + ExecReload - (define the action when systemctl restart service is run)
  + ExecStop - (define the action when systemctl stop service is run)
  + PIDFile - (define where the PID-File for the Process should be placed)
  + Restart - (define is the service should be restarted automatically)
  + RestartSec - (define a timer for the restart behavior)

+ **[Install]**
  + WantedBy - (define a target or service that softly depends on our service)

**Done** we have successfully created a service unit, now we need to reload systemd with :
```
root:~# systemctl daemon-reload
```
Our service is now loaded into systemd and can be started, stopped\
inspected using **systemctl commands**.

To make our teamspeak3.service start with every reboot\
we have to enable the it and we start it right away : 
```
root:~# systemctl enable teamspeak3.service 
root:~# systemctl start teamspeak3.service
```


#### Auto Updates
To keep our teamspeak server **secure**, it is important that it stays **up to date**.

Instead of manually updating it every now and then we will use the\
[**TS3UpdateScript**](https://github.com/TS3Tools/TS3UpdateScript) provided by **Sebastian Kraetzig (@Sebi94nbg)**\
and **contributors** to automate this process.

First we **login** as our teamspeak_3 user and make sure we are inside  our **home** directory,\
we then download the latest version of the TS3UpdateScript and unzip it.
```
root:~# su - teamspeak_3
teamspeak_3:~$ pwd
teamspeak_3:~$ wget https://github.com/TS3Tools/TS3UpdateScript/archive/main.zip
unzip main.zip
```
Delete the remining **main.zip** archive and rename the unpacked\
TS3UpdateScrit-main direcroty. Then enter the new directory :
```
teamspeak_3:~$ rm main.zip
teamspeak_3:~$ mv TS3UpdateScript-main teamspeak3_update
teamspeak_3:~$ cd teamspeak3_update
```
It is important to make the **TS3UpdateScript** executable or it won't work.\
And we may need to make some changes in the **/configs/config.all** file.
```
teamspeak_3:~/teamspeak3_update$ chmod u+x TS3UpdateScript
teamspeak_3:~/teamspeak3_update$ vim config/config.all
```
Adjust if necessary:
+ DEFAULT_LANGUAGE
+ ADMINISTRATOR_MAIL
+ LATEST_STABLE_RELEASE

When we created our **teamspeak3.sevice** file a few steps ago we included the statement :
```
ConditionPathExists=!/home/teamspeak_3/teamspeak3_server/.ts3updatescript.lock
```

The TS3UpdateScript will create a lock file while its updating the server\
and the line **ConditionPathExists=!** makes sure that our service will not start\
right away as long as this lockfile exists.

If you want to be able to poke someone or improve the detection of your server details,\
you need to place your serveradmin password in the file **.password-file**

The **.password-file** should only be readable and owned by root.

We can now trigger the update script using:
```
teamspeak3:~/teamspeak_update$ ./TS3UpdateScript --check
```
to see if its working properly.\
We can pass a lot more flags documented in the **README.md**.

Lets logout from the teamspeak3 user and return to root :
```
teamspeak3:~/teamspeak_update$ exit
root:~#
```

Assuming the script is working fine, next up we create a **systemd timer-unit**,\
so the update script runs on a regular schedule.

To create a timer-unit we need 2 files with the same name,\
one ending with **.timer** and the other ending with **.service**\
placed in ***/etc/systemd/system/*** :

```
root:~# cd /etc/systemd/system
root:/etc/systemd/system$ touch teamspeak3_update.service
root:/etc/systemd/system$ touch teamspeak3_update.timer
```
Our teamspeak3_update.service should look like this :
```
[Unit]
Description=Teamspeak3 Update Script Service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/home/teamspeak_3/teamspeak3_update_script/TS3UpdateScript --check --delete-old-logs --inform-online-clients
WorkingDirectory=/home/teamspeak_3

[Install]
WantedBy=multi-user.target                          
```
And our teamspeak3_update.timer should look like this:
```
[Unit]
Description=Teamspeak 3 Update Script Timer
Wants=network-online.target

[Timer]
Unit=teamspeak3_update.service
Persistent=True
OnCalendar=*-*-* 8:00:00

[Install]
WantedBy=multi-user.target                        
```
**Persistent** makes sure, in case our timer was unable to run,\
to retry its execution at a later point in time.

**OnCalendar=\*-\*-\* 8:00:00** means **every day** at 8 o'clock.

Reload systemd and start the service manually for testing with :
```
root:~# systemctl daemon-reload
toor:~# systemctl start teamspeak3_update.service
root:~# journalctl -xeu teamspeak3_update.service
```
Using the **journalctl** command we can take a look into the systemd journal\
and inspect the output of our manually started service, to check for success.

We enable and start the ***.timer*** unit and are done with the automated updates :
```
root:~# systemctl enable teamspeak3_update.timer
root:~# systemctl start teamspeak3_update.timer
```
Check if the timer is loaded correctly with :
```
systemctl list-timers
```

#### User Lockdown

Since the server and automated updates are working fine now,\
it is time to **lock down** the login for our **teamspeak_3** user.\
Deactivating the ability to login is an easy and effective way\
to harden our system.

To lock the user down, as root, we use the **usermod** command\
to assign a non existent default shell to the account.

**/usr/sbin/nologin** will promt ***This account is currently not available*** if someone tries to login.\
**/bin/false** won't promt anything and just deny any access.

```
root:~# usermod -s /usr/sbin/nologin teamspeak_3
```
or
```
root:~# usermod -s /bin/false teamspeak_3
```

---

### Router Config
Right now our server is fully functional, but only accessible\
from our local network via the container's IP.

To make it reachable from the internet we'll have to make some last\
adjustments, mainly inside our router.

#### Open necessary Ports
First of all we need to enable **Port-Forwarding** to our container,\
for the ports Teamspeak 3 relies on.

We need to set up some rules to **allow** traffic through our **router**\
forwarded to the **container** for the ports :
|IP|Protocol|Port|
|--|--------|----|
192.168.1.201|TCP|10011|
192.168.1.201|TCP|30033|
192.168.1.201|UDP|9987|

#### Server Address

Now our server is exposed to the internet and reachable for clients\
by the **IP** that we get from the **Internet-Provider**.

But those IP's **change regularly**, about once a week and our server would only\
be reachable under the same address for a short period of time.

Some ISP's also use **CGNAT** or Carrier Grade Network Address Translation\
to share your IP with several other households, in this case your teamspeak server\
also won't be reachable that easy.

We have several options to solve those problems :
* get a custom domain name and create a **SRV-Record** for the server
* register a **Server-Nickname** with your Teamspeak account
* Use the ***MYFritz! ID*** if you are using a FitzBox

**Since I own a FritzBox, and this option is very simple I choose the latter.**

Just create a MyFritz! Account within your router under **Internet/MyFritz!-Account**.\
**Verify** your registration via the E-Mail you received right away.\
**Don't activate** the remote access or any additional funktionallity for the router.\
After a short period of time your ***MyFritz-ID*** should be displayed\
under **Internet/MyFritz!-Account**.

It should look something like this `blablafoobar.myfritz.net`

### ***The server should be reachable from the outside via this address now!***

---