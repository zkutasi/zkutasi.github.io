ESXi is a bare-metal virtualization Operating System, meaning it is exactly the OS where you start up Virtual Machines. What are the advantages to have such a machine?
* It is super practical to have a single physical machine and be able to utilize it to provide different services
  * Moreso if you want to separate these services from each other, both in HW and SW
  * Even if these services need different OS (BSD, Linux, Windows, etc...)
* It is free (if you need to use only one such machine)
* It has basically negligible performance penalty using modern CPUs with built-in virtualization support

Hereby I document some things to help out in managing such a machine. Keep in mind I use VmWare ESXi 6.5.

# Passthrough devices

Sometimes you want to utilize one or more of your physical devices only in one VM, but you do not want to send the access through the layers of virtualization, for example your HDD must be formatted to VMFS, making it unusable outside of ESXi. However, there is direct passthrough.

## Passthrough HDD to a VM

First of all, keep in mind that especially with HDDs, passthrough is something you only want to do towards only a single VM, otherwise your data gets corrupted for sure. Later on, if you need this data in other VMs, use NFS sharing or Samba.

### Option A: One-by-one

Taken from [here](https://gist.github.com/Hengjie/1520114890bebe8f805d337af4b3a064)

First you need to create mappings, which can only be done in the CLI:
1. Enable SSH to your ESXi if you have not done so
```Host -> Actions -> Services -> Enable Secure Shell (SSH)```
1. SSH into
1. Once inside, see which devices are attached to the machine
```ls -l /vmfs/devices/disks/```
1. Physical disks take prefix t10
1. Now one needs to map these onto vmdk files that are able to be added to VMs
```vmkfstools -z DISK /vmfs/volumes/DATASTORE/VM/VMDKFILE```

Now, the mapping is done. Then add it to the VM on the ESXi GUI:
1. Under `Add Another Device`, click `SCSI controller`, select your SATA controller
1. Under `Add Another Device`, click `Existing hard disk` and select the newly created vmdk file.
1. Change `Disk Mode` to `Independent - persistent`
