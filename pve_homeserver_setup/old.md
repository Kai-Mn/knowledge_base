# IPv6 and Proxmox
* DHCPv6 doesnt work with lxc needs be static
* in network bridge gateway needs to be set to the internal ipv6 adress of the router not the public
* then you can asign yourself an ip address from the ragen in the bridge. 

# Route stuff with Traefik in an lxc on docker over ipv6
   1) make sure the lxc has an ipv6 address
      configure the network device to have an Static adress. (yes this sucks but i couldn't figure out how to make it over dhcpv6 so that is that)
      ```
      # set this to the ip you want the lxc to have. and its subnet/64
      IPv6/CIDR = 1234:1234:1234::14/64 
      # Set this to the INTERNAL IPv6 of your router. gl finding that on fritzbox. best to check it on a device that got it over DHCPv6 from the router.
      Gateway (IPv6): 1234:1234:1234:0:52e6:36ff:fe9d:c1e6
      ```
   2) next we need to enable ipv6 on the default network for docker over which it will go out. 
      1) edit or create `/etc/docker/daemon.json`\
         ```
         {
            "ipv6": true,
            "fixed-cidr-v6": "1234:1234:1234:d0c0::/64"
         }
         ```  
      2) check if its worked by inspecting default bridge `docker network inspect bridge` 
         or `docker run -d --rm -p 80:80 traefik/whoami` follwed by `curl http://[::1]:80`
         which should print something like:
         ```
         Hostname: 798bc0f79c61
         IP: 127.0.0.1
         IP: ::1
         IP: 172.17.0.2
         IP: 1234:1234:1234::242:ac11:2
         IP: fe80::42:acff:fe11:2
         RemoteAddr: [1234:1234:1234::1]:34776
         GET / HTTP/1.1
         Host: [::1]
         User-Agent: curl/8.5.0
         Accept: */*
         ```
      3) Configure Traefik, i use a dynamic traefik.yml for configuration which gets passed trough in the `/docker/traefik/config/traefik.yml:/etc/traefik/traefik.yml:ro`.
         I also passed trough a seperate volume that holds all static data from teh serverices in the `docker/` dir
         `docker-compose.yml`

         ```
         traefik:
            image: traefik:v3.1
            container_name: traefik
            command:
               #- "--providers.docker"
               #- "--log.level=DEBUG" # DEBUG, PANIC, FATAL, ERROR, WARN, INFO
               #- "--log.filePath=/logs/traefik.log"
            ports:
               - "80:80"
               - "443:443"
               - "8080:8080"
            volumes:
               # So that Traefik can listen to the Docker events
               - /var/run/docker.sock:/var/run/docker.sock:ro
               - ./logs/:/logs/
               # Dynamic configuration for traefik loaded here
               - /docker/traefik/config/traefik.yml:/etc/traefik/traefik.yml:ro
               # This will contain the acme.json file which will be generated by certbot. and also the certificates for the servces.
               - /docker/traefik/config/certs/:/etc/traefik/certs/
         ```
         `traefik.yml`

         ```
         global:
            checkNewVersion: false
            sendAnonymousUsage: false

         # -- (Optional) Change Log Level and Format here...
         #     - loglevels [DEBUG, INFO, WARNING, ERROR, CRITICAL]
         #     - format [common, json, logfmt]
         log:
            level: ERROR
            format: common
            filePath: /logs/traefik.log

         # -- (Optional) Enable Accesslog and change Format here...
         #     - format [common, json, logfmt]
         # accesslog:
         #   format: common
         #   filePath: /var/log/traefik/access.log

         # -- (Optional) Enable API and Dashboard here, don't do in production
         api:
            dashboard: true
            #disableDashboardAd: true
            insecure: true

         # -- Change EntryPoints here...
         entryPoints:
            web:
               address: :80
               # -- (Optional) Redirect all HTTP to HTTPS
               #http:
                  #redirections:
                  #entryPoint:
                     #to: websecure
                     #scheme: https
            websecure:
               address: :443
            # -- (Optional) Add custom Entrypoint
            # custom:
            #   address: :8080

         # -- Configure your CertificateResolver here...
         certificatesResolvers:
            staging:
               acme:
                  email: cert@kaman.ch
                  storage: /etc/traefik/certs/acme.json
                  caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"
                  httpChallenge:
                  entryPoint: web
            production:
               acme:
                  email: cert@kaman.ch
                  storage: /etc/traefik/certs/acme.json
                  caServer: "https://acme-v02.api.letsencrypt.org/directory"
                  httpChallenge:
                  entryPoint: web
            #       -- (Optional) Configure DNS Challenge
            #      dnsChallenge:
            #        provider: cloudflare
            #        resolvers:
            #          - "1.1.1.1:53"
            #          - "8.8.8.8:53"

         providers:
            docker:
               # -- (Optional) Enable this, if you want to expose all containers automatically
               exposedByDefault: false
            file:
               directory: /etc/traefik
            watch: true
         ```

         1) docker bridge braucht ipv6 daemon
         2) traefik installieren
         3) dns records von hostpoint auf cloudflare
         4) aaaa und cname records erstellen

         docker userland proxy
        `curl -v --connect-to 'mealie.kaman.ch:80:[2001:1620:708d::242:ac11:2]:80' http://mealie.kaman.ch/`
      4) make custom pass trough rule in your router for the ports and the ip of the server. Fritzbox you need to set the ip for which you want
      to create the rule by hand manually. 


# Eaton ellipse USP standalone mode over USB on Proxmox
NUT has 3 daemons associated with it:
   1) The driver which communicates with the UPS, configured in `nut.conf`
   2) A server (upsd) which uses the driver to report the status of the UPS. Created drivers from the configurations in `  nut.conf`
   3) A monitoring daemon (upsmon) which monitors the upsd server and takes action based on information it receives. Monitors the 
      UPS and triggers events which then are handled in `upssched-cmd`
[source](https://networkupstools.org/docs/user-manual.chunked/_installation_instructions.html)
[source_2](https://community.openhab.org/t/guide-to-network-ups-tools-nut-on-multiple-raspberry-pi-s-openhab-2-5-and-other-systems/129363)

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


# Proxmox ZFS full disk encryption with luks 

## Encrypt disks with luks
https://www.cyberciti.biz/security/howto-linux-hard-disk-encryption-with-luks-cryptsetup-command/
1) install cryptsetup
2) format block device with luks: `cryptsetup luksFormat --type luks2 /dev/<drive>`
3) Unlock devices: `cryptsetup -v luksOpen /dev/<drive> luks_<drive>` this will map the drive under /dev/mapper/luks_\<drive\>

## Create ZFS pool cmd
~~1) Create a zpool: `zpool create tank raidz /dev/mapper/luks_<drive> /dev/mapper/luks_<drive> /dev/mapper/luks_<drive>` ~~
1) Create a zpool referencing the drive by id `ls dev/disk/by-id` create in home directory a txt in which you note which drive is in what slot this will make replacing much easier.
   I havent tested this with encryped drives but should work`zpool create tank raidz /dev/disk/by-id/<ata-WDC...> /dev/disk/by-id/<> /dev/disk/by-id/<>`
2) In Datacenter -> Storage -> add zfs and create a "dataset" on the tank pool. content must be Disk image and Container so it will work with pbs. Also enable = true
3) Add the ZFS dataset as mountpoint to your lxc 

## Auto unlock with FDE
[Source](https://github.com/ftann/ctor-blog/blob/master/content/posts/fde-auto-unlock.md)
1) Create keyfiles `dd bs=1024 count=8 if=/dev/random of=root.key` `dd bs=1024 count=8 if=/dev/random of=data.key`
2) Format usb `mkfs.vfat /dev/sd<Xn>`
3) Mount usb `mkdir /mnt/usb` `mount /dev/sd<Xn> /mnt/usb` `cp *.key /mnt/usb`
4) get usb stick uuid `blkid -s UUID -o value /dev/sd<Xn>``
5) Add keys to LUKS for each drive `cryptsetup luksAddKey /dev/sd<A> data.key`
6) Add entries for each drive to crypttab `echo "luks_<id> UUID=$(blkid -s UUID -o value /dev/<id> | tr -d '\n') /data.key:UUID=<stick-uuid> discard,keyfile-timeout=10s,timeout=10s,nofail" >> /etc/crypttab`
7) The zfs-import-cache.service must wait for the encrypted volumes to be available. Add a drop-in configuration and set the devices as requirements in the boot process.
`systemctl edit zfs-import-cache.service`

The content of `/etc/systemd/system/zfs-import-cache.service.d/override.conf` should look like this:
```
[Unit]
After=systemd-cryptsetup@data1.service
After=systemd-cryptsetup@data2.service
Requires=systemd-cryptsetup@data1.service
Requires=systemd-cryptsetup@data2.service
```

## Usefull comands
`zpool import -a`
`zpool get cachefile <pool name>`

## What is ZFS
[source](https://forum.level1techs.com/t/zfs-guide-for-starters-and-advanced-users-concepts-pool-config-tuning-troubleshooting/196035)
[source](https://klarasystems.com/articles/openzfs-understanding-zfs-vdev-types/)
[source](https://www.reddit.com/r/zfs/comments/pjwwxh/benefit_to_using_luks_vs_zfs_native_encryption/)

        zpool
      /      \
   vdev1    vdev2
A zpool is a collection of vdevs. On the zpool level no redundancy is handeled. The zpool can be divided into datasets (which look like folders and mount)
Vdevs can consist of a single disks or a Raid of disks. So a zpool can have multiplle raidz1 vdevs. At this level redundancy is handled. 
To create a raid one should create a vdev and link that too a pool. Then datasets on that vdev can be created. 

`zpool list` will display pools. In my case with 3 hard drives of 4tb in it it will show a roughly 12tb big pool
`zfs list` will display vdevs. In my case this will show the raid created and a capacity of roughly 8tb with a RAIDZ config. IMPORTANT the vdev itself is a dataset.    


* each single disk is a vdev a pool can have many vdevs
* Data gets automatically striped across vdevs 
* Each dataset is its own filesystem you can mount (the pool i s a dataset too)



# PBS with Hetzner StorageBox
1) Instal pbs as a vm on pve host

## Mount Hetzner StorageBox in pbs
[source](https://docs.hetzner.com/robot/storage-box/access/access-samba-cifs/)
0) If you have a Fritzbox disable NetBIOS-Filter
1) Enable Samba-Support in your storage box and create a password in the Hetzner gui. 
2) Create hetzner-credentials.txt on the USB stick or wherever. `touch /etc/hetzner-credentials.txt` contents should look like and put in your pw and username.
```
username=<username>
password=<password>
```
3) Set permissions to 600 `chown 600 hetzner-credentials.txt`
4) Create mountpoint `mkdir /mnt/storage-box-<id>`
4) Add entry to `/etc/fstab`
5) Make sure cifs utils is installed `apt-get install cifs-utils`
```
//<username>.your-storagebox.de/backup /mnt/storage-box-<id> cifs iocharset=utf8,rw,credentials=/etc/hetzner-credentials.txt,uid=34,gid=34,file_mode=0660,dir_mode=0770 0 0
```
6) Reboot and make sure it's connected `df -h`
7) Create Datastore on the hetzner box this may take a while. (add datastore) 
8) create a backup user
9) add permissions for `/` and said user and give it the role `DatastoreBackup` this user can now create backups and use them but not delete
10) NOTE: [using ZFS with hdd's as backupstorage is far from ideal](https://forum.proxmox.com/threads/using-zfs-for-pbs-datastore.125045/) using a [   zfs special device](https://pve.proxmox.com/wiki/ZFS_on_Linux#sysadmin_zfs_special_device) should be considered 
## Create backup job
1) Datacenter add Storage -> select pbs add the storage with fingerprint from storage gui.
2) Select encryption and make sure to save this key in your pw manager or somewhere safe or you'll lose all data.
3) Go to backup and create a backup job for the vm's and lxc you want. It will back up the zfs storage as well

usfullcmnd `mount.cifs -o user=u415744,pass=JjNwmGuwRVi9JnGn //u147544.your-storagebox.de/backup /mnt/hetzner`


## Restore
* Restore to zfs volume then move the root disk to local storage



# Sonarr / prowlarr
1) set passwords
2) add index to prowlarr
3) add sonarr to prowlarr settings -> apps -> add application
4) install deluge as container and replace password
5) add deluge to sonarr settings -> download clients
6) add library
7) 
   ## Acl for sonar
   1) install acl
   2) on host add to `/etc/pve/lxc/100.conf` if not already there. 
      This is the uid/guid mapping from the container to the host mapping the id's with an offset of 100000 and a range of 65536
      ```
      lxc.idmap: u 0 100000 65536
      lxc.idmap: g 0 100000 65536
      ```
   3) restart container
   4) change  dataset from ``acltype:` `posix` to `posixacl` = `zfs set acltype=posixacl tank/subvol-100-disk-0` && `zfs set xattr=sa tank` if not already like this
   5) check change = `zfs get all /tank/subvol-100-disk-0`
   6) was already posix and i think thats enough for acl



# Wireguard-easy
1) password has double dollar for all dollars
2) create entrypoint streaming for udp
3) only expose 51820 with traefik not 51821 cuz thats the gui
4) create a router in the configuration fro udp
```
wireguard:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE #optional
    environment:
      - PUID=9992
      - PGID=9992
      - TZ=Etc/UTC
  GNU nano 7.2                      docker-compose.yml *                             
      - PEERS=1 #optional
      - PEERDNS=auto #optional
      #- INTERNAL_SUBNET=10.13.13.0 #optional
      #- ALLOWEDIPS=0.0.0.0/0 #optional
      #- PERSISTENTKEEPALIVE_PEERS= #optional
      - LOG_CONFS=true #optional
    volumes:
      - /docker/wireguard/config:/config
      #- /lib/modules:/lib/modules #optional
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=0
      - net.ipv6.conf.all.forwarding=1
      - net.ipv6.conf.default.forwarding=1
    restart: unless-stopped
```

# Nextcloud
1) set up db.
2) env variables are probaly not read check flo threema
3) set acl permissions for data `setfacl -R -m u:109998:rwx,g:109999:rwx /tank/subvol-100-disk-2/data/`