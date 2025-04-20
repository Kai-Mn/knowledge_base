-----------
# Semiprofessional homeserver with Proxmox, PBS and Dockercompose
10) Install Proxmox
15) Encrypt harddrives with luks
20) Create a ZFS Pool
30) Create an unprivileged LXC and its drawbacks
40) Install compose in the LXC
45) Add ipv6 support to the dh0 bridge of compose and Proxmox
46) Add local UPS/USV over USB
50) Install PBS on second server
51) Install PBS as vm and connect a Hetzner box to it
60) Setup backup routine within PVE and make it encrypted

# Hosting Webservices in LXC with dockercompose Traefik on IPv6
10) install and configure treafik to use ipv6
20) Issue certs with lets encrypt and a testing / production cert resolver explain acme


todos:
autostart of compose on startup of container
emailing not working for some
-----------

# Semiprofessional homeserver with Proxmox, PBS and Dockercompose
This guide aims to guide you trough the process of setting up a homeserver that has comprabale
security and data integrity to a professional solution or cloud services. This while maintaining
your data sovereignty and having a better user experience over all.
This guide assumes you have some technicall experience like beeing able to create a bootable usb 
stick and installing an os. 

**Hardware requirements**
* Two PC's/Servers 
* Preferably ECC RAM to get the full benefits of ZFS
* At least 3 Hard drives per PC

# Proxmox installation 
Proxmox is a open source virtalization platform based on debian. Maintained and developed by 
Proxmox Server Solutions GmbH.
It enables us to easily create LXC's and VM's as well as manage them in a comprehensive web gui. 
Yes the GUI works really well and is usefull this time. 

1) Create a bootable USB stick with proxmox [(PVE)](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso) and install it.
2) Log in to the web gui and start a shell. Proxmox has the annoying habbit of trying to get you to use their services and annoys you with pop up messages.
So we have to get rid of that and switch the repo from the properitary to the community one. Luckily this is very easy as there are countless [helper scripts](https://tteck.github.io/Proxmox/). Run the postinstall script `bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/post-pve-install.sh)"
` it will prompt you with some options, choose the ones that suit your usecase. 

# (Optional) LUKS full disk encryption (FDE) and automatic unlook with usb key
In this optionall part i'll go over on how to encrypt your hard drives with LUKS and make it unlock autmatically on boot up if a usb stick with a key is pluged in. It's important to do this step **before** you set up your ZFS pools and datasets. \
Alternatively you could use the native encryption of ZFS which is an easier option. I opt to use LUKS as im familiar with it and have read up on some problems with the ZFS implementation. A disadvantage of using LUKS is that if you wish to use a different Backupsolution to PBS like [sanoid](https://github.com/jimsalterjrs/sanoid) you'll have to handle the encryption of the backups seperatly as the datasets will be unencrypted then. This isn't a problem when you are using ZFS's native encryption. I however plan on using PBS which encrypts the backups itself so this isnt an issue.\
The following instructions are more or less coppied from my friend guide [here](https://github.com/ftann/ctor-blog/blob/master/content/posts/fde-auto-unlock.md).\
More [resoureces](https://www.cyberciti.biz/security/howto-linux-hard-disk-encryption-with-luks-cryptsetup-command/)\
[Luks vs Native Encryption](https://www.reddit.com/r/zfs/comments/pjwwxh/benefit_to_using_luks_vs_zfs_native_encryption/)

## Full Disk Encryption
1) Install cryptsetup
2) Forma all block device with luks: `cryptsetup luksFormat --type luks2 /dev/<drive>`
3) Unlock devices: `cryptsetup -v luksOpen /dev/<drive> luks_<drive>` this will map the drive under /dev/mapper/luks_\<drive\>

## Create ZFS pool/raid and datasets
Proceed with [Create ZFS pool/raid and datasets](#create-zfs-poolraid-and-datasets) and then come back here.

## FDE auto unlock with USB stick
In this section we are creating a keyfile that can be used to decrypt the hard drives and store it on an usb stick. Then we will set up `crypttab` to use this file to automatically unlock the drives after bootup. It will have a failover so if you loose or forget the USB stick you will be prompted for the password.

1) Create keyfiles `dd bs=1024 count=8 if=/dev/random of=root.key`. This will create a random 8KB file and write it to root.key.
2) Format usb `mkfs.vfat /dev/sd<Xn>`
3) Mount usb `mkdir /mnt/usb` `mount /dev/sd<Xn> /mnt/usb` 
4) Copy the keyfile to the USB stick. `cp *.key /mnt/usb`
5) We then need the UUID of the USB stick so it can be accesed by crypttab on boot before the /dev mount is available to the system `blkid -s UUID -o value /dev/sd<Xn>`. Take note of this UUID.
6) Add the key to the encryption of each LUKS drive so it can be used to decrypt all of them `cryptsetup luksAddKey /dev/sd<A> root.key`
7) Add entries for each drive to crypttab `echo "luks_<id> UUID=$(blkid -s UUID -o value /dev/sd<A> | tr -d '\n') /root.key:UUID=<stick-uuid> discard,keyfile-timeout=10s,timeout=10s,nofail" >> /etc/crypttab`
don't forget to replace the **\<id>** and **\<A>** for the right values. It tells crypttab which drives to decrypt and which key to use.  
8) The zfs-import-cache.service must wait for the encrypted volumes to be available. Add a drop-in configuration and set the devices as requirements in the boot process.\
Do this by editing the import service with `systemctl edit zfs-import-cache.service`  and put in an entry like the following for each hard drive you want to decrypt.\ 
    ```
    ### Editing /etc/systemd/system/zfs-import-cache.service.d/override.conf
    ### Anything between here and the comment below will become the new content of the file
    After=systemd-cryptsetup@luks_<id_1>.service
    After=systemd-cryptsetup@luks_<id_2>.service
    ...
    Requires=systemd-cryptsetup@luks_<id_1>.service
    Requires=systemd-cryptsetup@luks_<id_2>.service
    ### Lines below this comment will be discarded
    ```

    To verify the changes check the content of content of `/etc/systemd/system/zfs-import-cache.service.d/override.conf` it should look like this:
    ```
    After=systemd-cryptsetup@luks_<id_1>.service
    After=systemd-cryptsetup@luks_<id_2>.service
    ...
    Requires=systemd-cryptsetup@luks_<id_1>.service
    Requires=systemd-cryptsetup@luks_<id_2>.service
    ...
    ```
    Test if its working by rebooting your system the drives should be unlocked automatically and the zpool should appear.

# Create ZFS pool/raid and datasets 
A quick introduction to ZFS:\
[ZFS for starters and advanced users](https://forum.level1techs.com/t/zfs-guide-for-starters-and-advanced-users-concepts-pool-config-tuning-troubleshooting/196035)\
[ZFS vdev types](https://klarasystems.com/articles/openzfs-understanding-zfs-vdev-types/)

1) First we need to create a zpool which is comparable to a classical software raid. We are creating a RAIDZ which is roughly equivialent to a RAID-5 and gives us one parity drive, which is good up to 4-5 drives imo.\
This needs to be created over the CLI as the GUI method prevents us from selecting the RAID in our backup schedule. (iirc!)\
To create the zpool we are going to use the disk-id instead of the mounting point. This will make switching it out if it fails much easier than if we'd go by the mounting point, because then ZFS recognices it as a new drive. We do this the following way:
    * List drives by id `ls dev/disk/by-id`. My reccomendation is to create a txt file in the home directory where you take note which drive is in what slot of the server to make replacement much easier.
    * Afterwards we can create the pool `create tank raidz /dev/disk/by-id/<ata-WDC...> /dev/disk/by-id/<..> /dev/disk/by-id/<..>`.
2) Now we can create datasets that we can pass to our LXC's as containers in the GUI.\
In Datacenter -> Storage -> add zfs and create a "dataset" on the tank pool. content must be Disk image and Container so it will work with pbs. Also enable = true. 
3) Add the ZFS dataset as mountpoint to your lxc
---------
**Following a mental note for myself**
```
        zpool
      /      \
   vdev1    vdev2
```
A zpool is a collection of vdevs. On the zpool level no redundancy is handeled. The zpool can be divided into datasets (which look like folders and mount)
Vdevs can consist of a single disks or a Raid of disks. So a zpool can have multiplle raidz1 vdevs. At this level redundancy is handled. 
To create a raid one should create a vdev and link that too a pool. Then datasets on that vdev can be created. 

`zpool list` will display pools. In my case with 3 hard drives of 4tb in it it will show a roughly 12tb big pool
`zfs list` will display vdevs. In my case this will show the raid created and a capacity of roughly 8tb with a RAIDZ config. IMPORTANT the vdev itself is a dataset.    


* each single disk is a vdev a pool can have many vdevs
* Data gets automatically striped across vdevs 
* Each dataset is its own filesystem you can mount (the pool is a dataset too)

**Usefull comands**\
`zpool import -a`\
`zpool get cachefile <pool name>`
---------

# Create an unprivileged LXC with IPV6 support
In this secction we are going to create an unprivileged Linux Container. The unprivileged part is that the GUID and UID's of the container are going to be offset from the ones of the host machine which will give us an added layer of security because if an malicious actour would be able to gain root on the container it would be much easier to gain root on the host. With the offset this is near impossible. 
A caviat to this is that this makes managing read/write permissions a bit more labour intensive.
If you need hardwareacceleration i'd reccomend against using an LXC as it is  near impossible as of now to get this working.

## Create LXC
1) Download a linuximage into the `CT Template` Section of your host storage.
2) Now you can click `Create CT` and will be guided trough the setup. Choose settings that you deem neccessary. Make sure to check the `Unprivileged Container` box and enable #########.\
In the networktab Configure it as follows:\
`IPv4 = DHCP` \
`IPv6 = Static` and assign it an unused 56 Subnet of your IPv6 Prefix. As `Gateway` set the internal IPv6 address of your router.\
**NOTE:**\
I couldnt get DHCP working and i think its quite tricky with LXC's but what definetly would work is `SLAAC` so feel free to read up on that. 
3) Start the container and change the flag `IPv6AcceptRA` in `/etc/systemd/network/eth0.network` from *false* to *true*.\
    `nano /etc/systemd/network/eth0.network`
    ```
    IPv6AcceptRA = true
    ```
4) To persist this setting on reboot create the file:\
 `touch /etc/systemd/network/.pve-ignore.eth0.network`
1) Either reboot or restart the networkserviec. `systemctl restart systemd-networkd.service`
2) Verify it works by pinging an external server `ping -6 -c 4 google.com`

# Install Docker with support for IPv6
1) Install docker according to the instructions on [docker](https://docs.docker.com/engine/install/) for your distribution. I'll showcase this for an Ubuntu distribution.
    ```
    # Add Docker's official GPG key:
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update
    ```
2) `sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
3) Verify the installation by running the docker hello-world container: `sudo docker run hello-world`
4) Enable IPv6 on the default docker bridge. Choos a subnet no samler than 64 and that doesnt overlap with your prefix or other networks. the fd52 here can be copied as is.\
   `nano /etc/docker/daemon.json`
    ```
    {
        "experimental": true,
        "ipv6": true,
        "fixed-cidr-v6": "fd52::/64"
    }
    ```
5) Restart docker `systemctl restart docker`
6) Check if Docker has IPv6 now by inspecting the default bridge.
   `docker network inspect bridge` it should now have an IPv6 address.
7) Verify it works: `docker run --rm alpine ping -6 -c 4 google.ch` this should return 4 IPv6 pings.

## Configure IPv6 for docker compose



# Eaton Ellipse ECO Standalone mode over USB
To enable automatic shutdown on powerloss we are going to utilize the [NUT driver](https://networkupstools.org/ddl/Eaton/).
Nut consists of 3 Drivers:
   * Drivers which communicates with the UPS, configured in `nut.conf`
   * A server (`upsd`) which uses the drivers to report the status of the UPS. Creates drivers from the configurations in `nut.conf`
   * A monitoring daemon (upsmon) which monitors the upsd server and takes action based on information it receives. Monitors the 
      UPS and triggers events which then are handled in `upssched-cmd`

We are going to configure a driver in `nut.conf` thats going to be instanciated in `upsd` 

[Source](https://networkupstools.org/docs/user-manual.chunked/_installation_instructions.html)  
[Source 2](https://community.openhab.org/t/guide-to-network-ups-tools-nut-on-multiple-raspberry-pi-s-openhab-2-5-and-other-systems/129363)
1) Install the nut package `apt install nut`
2) Copy and configure `cp /etc/nut/nut.conf /etc/nut/nut.example.conf`
```
MODE = standalone
```

3) Copy and configure the driver for the UPS `cp /etc/nut/ups.conf /etc/nut/ups.example.conf`
```
maxretry = 3

[eaton]
# Eaton
driver = usbhid-ups
port = auto
desc = "Eaton UPS"
vendorid = 0463
productid = FFFF
serial = 000000000
# this needs to be set in order for upsmon to use the battery.charge < battery.charge.low comparison instead of waiting for the batterylow signal from the ups
ignorelb
override.battery.charge.low = 30
```
We can now start the driver with `upsdrvctl start` and test it with `upsc eaton` which should print the parameters of the ups.


4) Copy `cp /etc/nut/upsd.users /etc/nut/upsd.example.users` and create a user for upsmon to use to interact with the ups.
The admin user is optional and is needed if you want to set values manually on the ups
```
[admin]
password = admin
actions = SET
actions = FSD
instcmds = ALL
upsmon master

[upswired]
password = upswired
actions = FSD
upsmon = primary
```
5) Copy `cp /etc/nut/upsmon.conf /etc/nut/upsmon.example.conf` and configure the configuration for upsmon and select which systems to monitor and which events to trigger.

```
MONITOR eaton@localhost 1 upswired upswired primary

MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h"
NOTIFYCMD /sbin/upssched
POLLFREQ 4
POLLFREQALERT 2
HOSTSYNC 15
DEADTIME (= POLLFREQ * 6)
MAXAGE (= POLLFREQ * 6)
POWERDOWNFLAG /etc/killpower

NOTIFYMSG ONLINE "UPS %s on line power"
NOTIFYMSG ONBATT "UPS %s on battery"
NOTIFYMSG LOWBATT "UPS %s battary is low"
NOTIFYMSG FSD "UPS %s: forced shutdown in progress"
#NOTIFYMSG COMMOK "Communications with UPS %s established"
#NOTIFYMSG COMMBAD "Communications with UPS %s lost"
NOTIFYMSG SHUTDOWN "Auto logout and shutdown proceeding"
#NOTIFYMSG REPLBATT "UPS %s battery needs to be replaced"
#NOTIFYMSG NOCOMM "UPS %s is unavailable"
#NOTIFYMSG NOPARENT "upsmon parent process died - shutdown impossible"

NOTIFYFLAG ONLINE   SYSLOG+WALL+EXEC
NOTIFYFLAG ONBATT   SYSLOG+WALL+EXEC
NOTIFYFLAG LOWBATT  SYSLOG+WALL+EXEC
NOTIFYFLAG FSD      SYSLOG+WALL+EXEC
#NOTIFYFLAG COMMOK   SYSLOG+WALL+EXEC
#NOTIFYFLAG COMMBAD  SYSLOG+WALL+EXEC
NOTIFYFLAG SHUTDOWN SYSLOG+WALL+EXEC
#NOTIFYFLAG REPLBATT SYSLOG+WALL
#NOTIFYFLAG NOCOMM   SYSLOG+WALL+EXEC
#NOTIFYFLAG NOPARENT SYSLOG+WALL

RBWARNTIME 43200
NOCOMMWARNTIME 600

FINALDELAY 5
```

6) Copy `cp /etc/nut/upssched.conf /etc/nut/upssched.example.conf` and configure how upssched should handle certain triggers from upsmon
```
CMDSCRIPT /etc/nut/upssched-cmd
PIPEFN /etc/nut/upssched/upssched.pipe
LOCKFN /etc/nut/upssched/upssched.lock

#AT ONBATT * START-TIMER onbatt
#AT ONLINE * CANCEL-TIMER onbatt online
AT ONBATT * EXECUTE onbatt
AT ONBATT * START-TIMER earlyshutdown 600
AT ONLINE * CANCEL-TIMER earlyshutdown
AT LOWBATT * EXECUTE lowbatt
#AT COMMBAD * START-TIMER commbad 30
#AT COMMOK * CANCEL-TIMER commbad commok
#AT NOCOMM * EXECUTE commbad
#AT SHUTDOWN * EXECUTE powerdown
#AT SHUTDOWN * EXECUTE powerdown
```

7) Create or edit `/etc/nut/upssched-cmd`, here we configure how the system should behave for certain events
```
#!/bin/sh
EMAIL =  /etc/pve/user.cfg | grep root | awk '{split($0,a,":"); print a[7]}'

 case $1 in
       onbatt)
          logger -t upssched-cmd "UPS running on battery"
          echo "UPS went on battery." | mail -s "UPS" -a "From: pve@pve" pbs@kaman.ch
          ;;
       earlyshutdown)
          logger -t upssched-cmd "UPS on battery critical, forced shutdown"
          /usr/sbin/upsmon -c fsd
          ;;
       lowbatt)
          logger -t upssched-cmd "UPS on low battery, forced shutdown"
          /usr/sbin/upsmon -c fsd
          ;;
       upsgone)
          logger -t upssched-cmd "UPS has been gone too long, can't reach"
          ;;
       *)
          logger -t upssched-cmd "Unrecognized command: $1"
          ;;
 esac
```
Next make it executable `chmod +x /etc/nut/upssched-cmd`

8) Now we can restart the service 
```
service nut-server restart
service nut-client restart
systemctl restart nut-monitor
upsdrvctl stop
upsdrvctl start
```

### Usefull commands
`upsc Eaton ups.status` checks status
`nut-scanner --usb_scan` scans for connected bussses on usb
