# ZFS_Client-Server
This is the complete guide to recreating a virtual system consisting of a ZFS server and Ubuntu clients connected via SSH on the intranet.

## Hardware requirements
This setup requires:
* Preferably 6 spendable CPU cores (minimum of 3)
* 30GB of free disk space for systems + disk space for zfs storage (e.g. 4 x 1GB)
* Preferably 12GB of RAM

## Software requirements
* Oracle VirtualBox 7.2.8
* Ubuntu Desktop 26.04 .iso image
* FreeBSD 15.0 .iso image

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
Then, we create a gpt partition tables.
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
We activate the profile and check if it is up and with a correct address.
```bash
sudo nmcli connection up static-enp0s8

ip a
```

## Main ZFS Pool Test
With server and both clients running we can perform the final test. We connect both clients to the server via SSH.
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

## Conclusions
If the configuration and test were performed correctly, in the output of the last command we should see that out of 270M referred only 170M is actually used by the system. This means that the shared data is not physically copied and the newly modified data is stored separately in a new location.
