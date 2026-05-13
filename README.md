# ZFS_Client-Server
This is the complete guide to recreating a virtual system consisting of a ZFS server and Ubuntu clients connected via SSH on the intranet.

## Hardware requirements
This setup requires:
* Preferably 6 spendable CPU cores (minimum of 3)
* 30GB of free disk space for systems + disk space for zfs storage (e.g. 4 x 1GB)
* Preferably 12GB of RAM

## Software requirements
* [Oracle VirtualBox 7.2.8](https://www.virtualbox.org/wiki/Downloads)
* [Ubuntu Desktop 26.04 .iso image](https://ubuntu.com/download/desktop)
* [FreeBSD 15.0 .iso image](https://www.freebsd.org/where/)

## VirtualBox Configuration 
### Server
First, we create a new VM with the following settings:
* name: vmServer
* ISO Image: FreeBSD .iso file
* OS: BSD
* OS Distribution: FreeBSD
* OS Version: FreeBSD (64-bit)
* 4GB of base memory
* 2 CPU's (at least 1)
* We create a new virtual 10GB hard disk of type VDI (VirtualBox Disk Image)

We finish the initial configurtion and right-click on the newly created VM on the list to go to settings, where we:
* set UEFI in motherboard settings
* create additional disks (e.g. four 1GB disks)
* connect all disks using Controller: AHCI (SATA)
* make sure that the .iso image is in the devices as well (under Controller: IDE)
* In the Network settings, add a second adapter of type Internal Network and name it (the name needs to match the respective adapter names in client machines)

### Client
For the client, we create a VM:
* name: vmClient1 (and vmClient2)
* ISO Image: Ubuntu .iso file
* We can set username and password for the unattended OS installation
* 4GB of base memory
* 2 CPU's (at least 1)
* We create a new virtual 10GB hard disk of type VDI (VirtualBox Disk Image)

As in the server config, we finish the initial configurtion and right-click on the new client vm on the list to go to settings, where we:
* turn the video memory to max 128MB (in case the UI is stuttering or glitching)
* In storage: .iso image (under Controller: IDE), VDI disk (under Controller: SATA)
* In Network: add a second adapter Internal Network with the same name as in the server VM

Repeat the process for the second client.

## FreeBSD Server installation and configuration
### Installation
We launch the server VM and perform mostly default installation with the exception of:
* UFS on the primary 10GB disk
* We add two non-root users for the clients to log into (e.g. user1, user2)

Now we can shut down the machine, remove the .iso from storage settings and boot the vm from the primary disk.

### ZFS Configuration
We create a zfs pool on the additional disks, a dataset, and a large text file.
Assuming that the primary disk is ada0 and we have 4 additional disks (ada1, ada2, ada3, ada4), we clear the data from storage disks.
```bash
gpart destroy -F ada1
```
(The same for ada2, ada3, and ada4)
Then, we create gpt partition tables.
```bash
gpart create -s gpt ada1
```
(The same for ada2, ada3, and ada4)
Next, we create a partition for each disk.
```bash
gpart add -a 4k -t freebsd-zfs -l zdisk1 ada1
```
(zdisk2, zdisk3, zdisk4 for ada2, ada3, ada4 respectively)
We create a pool with appropriate RAID type.
```bash
zpool create testpool raidz gpt/zdisk1 gpt/zdisk2 gpt/zdisk3 gpt/zdisk4
```
We generate a dataset and a file in the new testpool.
```bash
# A dataset
zfs create testpool/testds

# A 100MB file filled with random data
dd if=/dev/urandom of=/testpool/testds/file.txt bs=1M count=100
```
We create a snapshot, and a clone of the dataset.
```bash
# A snapshot
zfs snapshot testpool/testds@base_state

# A clone for the second user
zfs clone testpool/testds@base_state testpool/testds2
```
We add ownerships to the users
```bash
chown -R user1 /testpool/testds

chown -R user2 /testpool/testds2
```
### Network Configuration (Server)
Finally, we configure the internal interface (assuming it's called em1). We modify the /etc/rc.conf file:
```bash
ee /etc/rc.conf
```
by appending the following line to it:
```
ifconfig_em1="inet 192.168.50.1 netmask 255.255.255.0"
```
## Ubuntu Client installation and configuration
As we marked the unattended installation, the ubuntu will boot from .iso file and perform a background installation.
We can focus on the network adapter configuration.

### Network Configuration (Client)
Ubuntu will likely auto-generate a DHCP profile for the internal network interface.
```bash
# Check the name and address
ip a

# Check profiles
nmcli connection show
```
If the second command shows "Wired connection 1" attached to the third interface we need to delete it.
```bash
sudo nmcli connection delete "Wired connection 1"
```
We create a new profile and force it to use IPv4 addressing.
```bash
# Assuming "enp0s8" is the name of the interface
# For client1 "192.168.50.10/24" and for client2 "192.168.50.11/24"
sudo nmcli connection add type ethernet con-name static-enp0s8 ifname enp0s8 ipv4.method manual ipv4.addresses 192.168.50.10/24 ipv6.method ignore
```
We activate the profile and check if it is up, with a correct address.
```bash
sudo nmcli connection up static-enp0s8

ip a
```

## Main ZFS Pool Test
With server and both clients running, we can perform the final test. We connect both clients to the server via SSH.
```bash
# On client1 vm
ssh user1@192.168.50.1

# On client2 vm
ssh user2@192.168.50.1
```
When connected to the server, we modify the appropriate datasets.
```bash
# On client1 vm
dd if=/dev/urandom bs=1M count=20 >> /testpool/testds/file.txt

# On client2 vm
dd if=/dev/urandom bs=1M count=50 >> /testpool/testds2/file.txt
```
Finally, we check the outcome of the modifications on both datasets.
```bash
# On server vm
zfs list -o name,used,refer,written testpool/testds testpool/testds2
```
## Internal Backup Setup
To create a periodic backup mechanism we need to add new disks, create a pool, write a script to automate the process, and add it to Cron Daemon which runs scripts periodically.

### New disks and a backup pool
We perform very similar steps as in ZFS Configuration. In this case, I use 2 additional 10GB virtual disks (ada5, ada6).

```bash
gpart destroy -F ada5

gpart create -s gpt ada5

gpart add -a 4k -t freebsd-zfs -l zdisk5 ada5
```
And, the same for 'ada6'.

Then, we create a 'backuppool' pool as 2 disks mirrored for data redundancy. 

```bash
zpool create backuppool mirror gpt/zdisk5 gpt/zdisk6
```
### Backup Script
We create a shell script 'copy.sh' in the root directory and give it a right to execute
```bash
touch /root/copy.sh
chmod +x /root/copy.sh
```
The script is designed to check for the older snapshot both on the source "testpool" and on the destination "backuppool", and delete them to keep the backup drives operational. If the older backup exists, the script updates the backup incrementally in the destination.

```bash
#!/bin/sh

# Check if 'backuppool is online
if ! zpool status backuppool > /dev/null 2>&1; then
    echo "ERROR: 'backuppool' is unavailable."
    exit 1
fi

# MAIN FUNCTION
backup_dataset() {
    SOURCE=$1
    DEST=$2

    # Generate a timestamp
    TIMESTAMP="backup_$(date +%Y%m%d_%H%M%S)"
    NEW_SNAP="${SOURCE}@${TIMESTAMP}"

    # Look for the older snapshot on source
    OLD_SNAP=$(zfs list -t snapshot -o name -H -d 1 ${SOURCE} 2>/dev/null | grep "@backup_" | tail -n 1)

    echo "Init backup: ${SOURCE} -> ${DEST}"
    
    # Take the new snapshot
    zfs snapshot ${NEW_SNAP}

    # If OLD_SNAP is empty, this is the first run
    if [ -z "${OLD_SNAP}" ]; then
        echo "  -> Initial sync..."
        zfs send ${NEW_SNAP} | zfs receive -F ${DEST}
    else
        echo "  -> Incremental sync..."
        zfs send -i ${OLD_SNAP} ${NEW_SNAP} | zfs receive -F ${DEST}
        

        # Destroy the old snapshot on source
        zfs destroy ${OLD_SNAP}
        
	      # And the one on destination
        zfs destroy ${DEST}@${OLD_SNAP##*@} 
    fi
    
    echo "  -> Backup complete for ${SOURCE}."
    echo "------------------------------------------------"
}

# EXEC
echo "Backup at $(date +%Y%m%d_%H%M%S)"
backup_dataset "testpool/testds" "backuppool/user1_backup"
backup_dataset "testpool/testds2" "backuppool/user2_backup"

echo "All backup operations finished successfully."
```
We can test this script using the following commands:
```bash
# Execute
./root/copy.sh

# List the snapshots
zfs list -t snapshot
```

### Cron Table Update
To register the script in the Cron Daemon, we need to modify the crontab file where we can assign any amount of time between the script runs. Lets assign a daily backup procedure, and let the output be saved to "var/log/my_backup.log". We open the file in an editor:
```bash
ee /etc/crontab
```
Next, append this line to it:
```bash
0  1  *  *  *  root  /root/copy.sh >> /var/log/my_backup.log 2>&1
```

## Conclusions
If the configuration and test were performed correctly, in the output of the last command we should see that out of 270M referred only 170M is actually used by the system. This means that the shared data is not physically copied and the newly modified data is stored separately in a new location.
