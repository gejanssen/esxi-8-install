# Install Esx 8

Using ISO image to install:
ESXi-8.0b-21203435-USBNIC.iso

## SSH

Install the machine, go to the webinterface and enable ssh.
(insert image here)

### Enable ssh permanently

We need to edit the /etc/rc.local.d/local.sh
```
vim-cmd hostsvc/enable_ssh

sed -i '/exit 0/d' /etc/rc.local.d/local.sh
echo "vim-cmd hostsvc/enable_ssh" >> /etc/rc.local.d/local.sh && \
echo "exit 0" >> /etc/rc.local.d/local.sh
cat /etc/rc.local.d/local.sh 
```

### Disable SSH warning on webinterface
Login to the server and enter the following:

```
esxcli system settings advanced set -o /UserVars/SuppressShellWarning -i 1
```

## Name the server

Setting the hostname of the server
```
esxcli system hostname set --host=kuiper4
esxcli system hostname set --fqdn=kuiper4.fritz.box
```


### DHCP Hostname

Making sure that the fritzbox DNS is working:

```
[root@kuiper4:~] echo "send host-name = \"`hostname -s`\";" >> /etc/dhclient-vmk0.conf
[root@kuiper4:~] cat /etc/dhclient-vmk0.conf 
#
# This file can be used to specify advanced dhclient options for vmk0
# (usually the management interface). Please refer to dhclient.conf(5)
# packaged with ISC DHCP 4.0.0 for available options.
#
send host-name = "kuiper2";
[root@kuiper2:~]
```

OR:
```
echo "send host-name = \"`hostname -s`\";" >> /etc/dhclient-vmk0.conf
echo "send host-name = \"`hostname -s`\";" >> /etc/dhclient6-vmk0.conf

```

## NTP

Setting NTP 
```
[root@localhost:~] esxcli system ntp set --server 1.nl.pool.org --server 2.nl.pool.org --server 3.nl.pool.org --enabled=true
[root@localhost:~] esxcli system ntp get
   Enabled: true
   Loglevel: warning
   Servers: 1.nl.pool.org, 2.nl.pool.org, 3.nl.pool.org
[root@localhost:~]
```


## SSH-Keys
ssh keys are stored in /etc/ssh/keys-[username]

Please mind: This only works for RSA and XXX keys. Not for EDSA

so create a file authorized_keys and set the rights to rw - - and copy the id_rsa.pub of the other user.

```
[root@kuiper2:~] ls -l /etc/ssh/keys-root/authorized_keys 
-rw------T    1 root     root           393 Jan 18 11:50 /etc/ssh/keys-root/authorized_keys
[root@kuiper2:~]
```

## NFS

Mounting my NFS shares on the diskstation

https://vdc-download.vmware.com/vmwb-repository/dcr-public/0a40d9c5-4d4b-490d-8efa-e373a0ff2109/43a3c005-3878-4e05-8b60-35aca804d61d/doc/GUID-F637CD95-B8DD-417D-B9F0-EBE84C672DA9.html

List all known NAS file systems.

```
esxcli <conn_options> storage nfs list
```

For each NAS file system, the command lists the mount name, share name, and host name and whether the file system is mounted. 
If no NAS file systems are available, the system does not return a NAS filesystem and returns to the command prompt.

Add a new NAS file system to the ESXi host.

Specify the NAS server with --host, the volume to use for the mount with --volume-name, and the share name on the remote system to use for this NAS mount point with --share.

```
esxcli <conn_options> storage nfs add --host=dir42.eng.vmware.com --share=/<mount_dir> --volume-name=nfsstore-dir42
```

This command adds an entry to the known NAS file system list and supplies the share name of the new NAS file system. You must supply the host name, share name, and volume name for the new NAS file system.

Add a second NAS file system with read-only access.

```
esxcli <conn_options> storage nfs add --host=dir42.eng.vmware.com --share=/home --volume-name=FileServerHome2 --readonly
```

Delete one of the NAS file systems.

```
esxcli <conn_options> storage nfs remove --volume-name=FileServerHome2
```

This command unmounts the NAS file system and removes it from the list of known file systems. 

```
[root@kuiper2:~] esxcli storage nfs list
Volume Name  Host          Share                 Accessible  Mounted  Read-Only   isPE  Hardware Acceleration
-----------  ------------  --------------------  ----------  -------  ---------  -----  ---------------------
nfsbackup    192.168.1.22  /volume1/vmnfsbackup        true     true      false  false  Not Supported
[root@kuiper2:~]
[root@kuiper2:~] esxcli storage nfs add --host=192.168.1.22 --share=/volume1/vmnfsbackup --volume-name=nfsbackup 
Unable to create new NAS volume.  192.168.1.22:/volume1/vmnfsbackup is already exported by a volume with the name nfsbackup
[root@kuiper2:~] 
```

## Autostart machines

Enable autostart for autostart the most important virtual machines

```
vim-cmd help hostsvc/autostartmanager
```

Enable autostart
```
vim-cmd hostsvc/autostartmanager/enable_autostart true
vim-cmd hostsvc/autostartmanager/update_autostartentry $VMId $StartAction $StartDelay $StartOrder $StopAction $StopDelay $WaitForHeartbeat
vim-cmd hostsvc/autostartmanager/update_autostartentry 1 
```
## Enable USB NIC
```
esxcli system module parameters set -p "usbBusFullScanOnBootEnabled=1" -m vmkusb_nic_fling
```

## Power settings

Enable powersave settings

```
[root@kuiper3:~] vsish -e get /power/currentPolicy
Host power management policy
   ID: 2
   Short name:dynamic
   Long name:Balanced
   Description:Reduce energy consumption with minimal performance compromise
[root@kuiper3:~] vsish -e get /power/hardwareSupport
Hardware power management support {
   CPU power management:ACPI P-states, ACPI C-states
[root@kuiper3:~]  vsish -e get /power/policy/1
Host power management policy 
   ID: 1
   Short name:static
   Long name:High Performance
   Description:Do not use any power management features
[root@kuiper3:~]  vsish -e get /power/policy/3
Host power management policy 
   ID: 3
   Short name:low
   Long name:Low Power
   Description:Reduce energy consumption at the risk of lower performance
[root@kuiper3:~] vsish -e set /power/currentPolicy 3
[root@kuiper3:~]
```

## License

Enable the correct license

```
[root@kuiper3:~] vim-cmd vimsvc/license --set=XXXX-XXXXX-XXXXX-XXXXX

   serial: XXXX-XXXX-XXXX-XXXX
   vmodl key: esx.hypervisor.cpuPackageCoreLimited
   name: vSphere 7 Hypervisor
   total: 0
   used: 1
   unit: cpuPackage:32core
   Properties:
     [ProductName] = VMware ESX Server
     [ProductVersion] = 7.0
     [count_disabled] = This license is unlimited
     [feature] = vsmp:8 ("Up to 8-way virtual SMP")
     [FileVersion] = 7.0.3.1
```



# Maintenance

## Update Esx

Unfortunately I did not get it to work and since VMware has been bought by Broadcom.....

### Show current version
```
[root@kuiper:~] vmware -l
VMware ESXi 6.7.0 Update 3
[root@kuiper:~]
```
And build number
```
[root@kuiper1:~] vmware -v
VMware ESXi 7.0.3 build-20328353
[root@kuiper1:~]
```

### First set the server in maintenance

```
esxcli system maintenanceMode set -e true
```
And exit maintenance
```
esxcli system maintenanceMode set -e false
```


```
[root@localhost:/vmfs/volumes/66ca1182-f7e1fe33-483a-9c2dcd535edd] esxcli software vib update -d /vmfs/volumes/NVME/VMware-ESXi-8.0U2-22380479-depot.zip
 [ProfileValidationError]
 Profile (Updated) ESXi-8.0b-21203435-USBNIC is missing component(s) ESXi,Mellanox-nmlx5. Make sure the image contains these component(s) at a version equal or higher than the version found in the ESXi release version 8.0.2-0.0.22380479.
 Please refer to the log file for more details.
[root@localhost:/vmfs/volumes/66ca1182-f7e1fe33-483a-9c2dcd535edd]
```

## Smartd

Information about the disks via smartctl on esx (smartd)

```
[root@kuiper4:~] esxcli storage core device list
t10.NVMe____Lexar_SSD_NM790_2TB_____________________05000020035BF2CA
   Display Name: Local NVMe Disk (t10.NVMe____Lexar_SSD_NM790_2TB_____________________05000020035BF2CA)
   Has Settable Display Name: true
   Size: 1953514
   Device Type: Direct-Access
   Multipath Plugin: HPP
   Devfs Path: /vmfs/devices/disks/t10.NVMe____Lexar_SSD_NM790_2TB_____________________05000020035BF2CA
   Vendor: NVMe
   Model: Lexar SSD NM790
   Revision: 1129
   SCSI Level: 7
   Is Pseudo: false
   Status: on
   Is RDM Capable: false
   Is Local: true
   Is Removable: false
   Is SSD: true
   Is VVOL PE: false
   Is Offline: false
   Is Perennially Reserved: false
   Queue Full Sample Size: 0
   Queue Full Threshold: 0
   Thin Provisioning Status: yes
   Attached Filters:
   VAAI Status: unsupported
   Other UIDs: vml.05358ec898462096d458ff29446c712a2f2bade07e07508540810a5d2dc1de12a8
   Is Shared Clusterwide: false
   Is SAS: false
   Is USB: false
   Is Boot Device: false
   Device Max Queue Depth: 1023
   No of outstanding IOs with competing worlds: 32
   Drive Type: physical
   RAID Level: NA
   Number of Physical Drives: 1
   Protection Enabled: false
   PI Activated: false
   PI Type: 0
   PI Protection Mask: NO PROTECTION
   Supported Guard Types: NO GUARD SUPPORT
   DIX Enabled: false
   DIX Guard Type: NO GUARD SUPPORT
   Emulated DIX/DIF Enabled: false

t10.ATA_____SK_hynix_SC311_SATA_256GB_______________MJ7CN7053106B120O___
   Display Name: Local ATA Disk (t10.ATA_____SK_hynix_SC311_SATA_256GB_______________MJ7CN7053106B120O___)
   Has Settable Display Name: true
   Size: 244198
   Device Type: Direct-Access
   Multipath Plugin: HPP
   Devfs Path: /vmfs/devices/disks/t10.ATA_____SK_hynix_SC311_SATA_256GB_______________MJ7CN7053106B120O___
   Vendor: ATA
   Model: SK hynix SC311 S
   Revision: 0P10
   SCSI Level: 5
   Is Pseudo: false
   Status: on
   Is RDM Capable: false
   Is Local: true
   Is Removable: false
   Is SSD: true
   Is VVOL PE: false
   Is Offline: false
   Is Perennially Reserved: false
   Queue Full Sample Size: 0
   Queue Full Threshold: 0
   Thin Provisioning Status: yes
   Attached Filters:
   VAAI Status: unsupported
   Other UIDs: vml.01000000004d4a37434e37303533313036423132304f202020534b2068796e
   Is Shared Clusterwide: false
   Is SAS: false
   Is USB: false
   Is Boot Device: true
   Device Max Queue Depth: 31
   No of outstanding IOs with competing worlds: 31
   Drive Type: unknown
   RAID Level: unknown
   Number of Physical Drives: unknown
   Protection Enabled: false
   PI Activated: false
   PI Type: 0
   PI Protection Mask: NO PROTECTION
   Supported Guard Types: NO GUARD SUPPORT
   DIX Enabled: false
   DIX Guard Type: NO GUARD SUPPORT
   Emulated DIX/DIF Enabled: false
[root@kuiper4:~]
```

30-08-2024 - Kuiper4
```
[root@kuiper4:~] esxcli storage core device smart get -d t10.NVMe____Lexar_SSD_NM790_2TB_____________________05000020035BF2CA
Parameter                 Value   Threshold  Worst  Raw
------------------------  ------  ---------  -----  ---
Health Status             OK      N/A        N/A    N/A
Power-on Hours            0       N/A        N/A    N/A
Power Cycle Count         2       N/A        N/A    N/A
Reallocated Sector Count  0       90         N/A    N/A
Drive Temperature         36      90         N/A    N/A
Write Sectors TOT Count   139000  N/A        N/A    N/A
Read Sectors TOT Count    296000  N/A        N/A    N/A
[root@kuiper4:~] esxcli storage core device smart get -d t10.ATA_____SK_hynix_SC311_SATA_256GB_______________MJ7CN7053106B120O___
Parameter                        Value  Threshold  Worst  Raw
-------------------------------  -----  ---------  -----  ---
Health Status                    OK     N/A        N/A    N/A
Media Wearout Indicator          1      0          1      100
Read Error Count                 166    6          166    0
Power-on Hours                   94     0          94     6126
Power Cycle Count                100    20         100    364
Reallocated Sector Count         253    36         253    0
Drive Temperature                31     0          20     85902098463
Write Sectors TOT Count          100    0          100    5974127980
Read Sectors TOT Count           100    0          100    6649624297
Uncorrectable Error Count        253    0          253    0
Sector Reallocation Event Count  253    36         253    0
Uncorrectable Sector Count       253    0          253    0
[root@kuiper4:~]
```


## Backup
General Idea:
  Make snapshot
  copy server
  remove snapshot

### Get VM IDs
```
[root@kuiper:~] vim-cmd vmsvc/getallvms
Vmid     Name                    File                     Guest OS       Version         Annotation       
2      xx         [datastore1] xx/xx.vmx               rhel8_64Guest     vmx-14                           
3      makemake   [datastore1] makemake/makemake.vmx   centos8_64Guest   vmx-14    Rocky linux 8
autostart
[root@kuiper:~] 
```

### Create snapshot
```
vim-cmd vmsvc/snapshot.create VMID snapshotName description includeMemory quiesced

vim-cmd vmsvc/snapshot.create 5 backup includeMemory quiesced

```

### Show snapshots
```

[root@kuiper:~] vim-cmd vmsvc/snapshot.get 3
Get Snapshot:
|-ROOT
--Snapshot Name        : snapshot_backup
--Snapshot Id        : 5
--Snapshot Desciption  : 0
--Snapshot Created On  : 12/17/2022 15:34:49
--Snapshot State       : powered on
[root@kuiper:~]
```

### Make backup
```
[root@kuiper:~] cp /vmfs/volumes/datastore1/Haumea/* /vmfs/volumes/nfsbackup/haumea/
cp: can't open '/vmfs/volumes/datastore1/Haumea/Haumea-000001-sesparse.vmdk': Device or resource busy
cp: can't open '/vmfs/volumes/datastore1/Haumea/Haumea-2967b2b4.vswp': Device or resource busy
cp: can't open '/vmfs/volumes/datastore1/Haumea/Haumea.vmx.lck': Device or resource busy
cp: can't open '/vmfs/volumes/datastore1/Haumea/vmx-Haumea-694661812-1.vswp': Device or resource busy
[root@kuiper:~]
```


### Remove snapshot
```

[root@kuiper:~] vim-cmd vmsvc/snapshot.removeall 3
Remove All Snapshots:
[root@kuiper:~]
```

### All at once
```
vim-cmd vmsvc/snapshot.create 3 "snapshot_backup" 0 1
cp /vmfs/volumes/datastore1/Haumea/* /vmfs/volumes/nfsbackup/haumea/
vim-cmd vmsvc/snapshot.removeall 3
```

### Restore / Register VM

```
[root@kuiper1:/vmfs/volumes/63e4e9c9-658b983b-d375-00e04c4021e7/truenas] vim-cmd solo/registervm /vmfs/volumes/datastore
1/truenas/truenas.vmx
```



