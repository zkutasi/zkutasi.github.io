ESXi is a bare-metal virtualization Operating System, meaning it is exactly the OS where you start up Virtual Machines. What are the advantages to have such a machine?
* It is super practical to have a single physical machine and be able to utilize it to provide different services
  * Moreso if you want to separate these services from each other, both in HW and SW
  * Even if these services need different OS (BSD, Linux, Windows, etc...)
* It is free (if you need to use only one such machine)
* It has basically negligible performance penalty using modern CPUs with built-in virtualization support

Hereby I document some things to help out in managing such a machine. Keep in mind I use VmWare ESXi 6.5.

# HW issues

## PWM fan oscillates

It can occur if you install a PWM fan and attach it to the motherboard, that it starts to wildly oscillate its speed. This is due to the Motherboard being set incorrectly to think the fan is defective because it spins too low (server motherboard). To fix this, you need to:

1. Connect to the IPMI interface of the Motherboard, set it up if you did not
1. Check the fans

    ```
    ipmitool -U <USERNAME> -H <KVM-IP> sensor
    ```

1. Set a lower minimal speed so the actual fan speed does not interfere with the defaults (check the manufacturers data and use a bit of a threshold also)

    ```
    ipmitool -U <USERNAME> -H <KVM-IP> sensor thresh FAN1 lower 150 225 300
    ```

Taken from [here](http://www.kaff99.ch/pwm-fan-spin-up-on-supermicro-board.html)

# Passthrough devices

Sometimes you want to utilize one or more of your physical devices only in one VM, but you do not want to send the access through the layers of virtualization, for example your HDD must be formatted to VMFS, making it unusable outside of ESXi. However, there is direct passthrough.

## Passthrough HDD to a VM

First of all, keep in mind that especially with HDDs, passthrough is something you only want to do towards only a single VM, otherwise your data gets corrupted for sure. Later on, if you need this data in other VMs, use NFS sharing or Samba.

### Option A: One-by-one

First you need to create mappings, which can only be done in the CLI:
1. Enable SSH to your ESXi if you have not done so

    ```
    Host -> Actions -> Services -> Enable Secure Shell (SSH)
    ```
    
1. SSH into
1. Once inside, see which devices are attached to the machine

    ```
    ls -l /vmfs/devices/disks/
    ```
    
1. Physical disks take prefix t10
1. Now one needs to map these onto vmdk files that are able to be added to VMs

    ```
    vmkfstools -z <DISK> /vmfs/volumes/<DATASTORE>/<VM>/<VMDKFILE>
    ```


Now, the mapping is done. Then add it to the VM on the ESXi GUI:
1. Under `Add Another Device`, click `SCSI controller`, select your SATA controller
1. Under `Add Another Device`, click `Existing hard disk` and select the newly created vmdk file.
1. Change `Disk Mode` to `Independent - persistent`

Taken from [here](https://gist.github.com/Hengjie/1520114890bebe8f805d337af4b3a064)

### Option B: Whole SATA controller

This is the better option, if you are able to do it.

There is an issue that not too many SATA controllers are properly configured in ESXi to be able to do passthrough, although they are perfectly capable of. To fix this:
1. Check the passthru mapping file for your controller

    ```
    cat /etc/vmware/passthru.map
    ```
    
1. If it does not contain your controller you need to add it yourself. For this you also need the proper information for it (mine was provided though by VmWare themselves).
  
    ```
    # INTEL Sunrise Point-H AHCI Controller
    8086  a102  d3d0  false
    ```
    
1. If you need to check yours, issue the following commands
    
    ```
    lspci
    lspci -n
    ```
    
1. Check for your controller name in the first, and the matching line in the second output. Your data needed is there.
1. Reboot the ESXi box

Now to do the passthrough:
1. Select passthrough for the device in the ESXi GUI

    ```
    Manage -> Hardware -> PCI Devices -> Passthrough -> ENABLE
    ```

1. Reboot the ESXi box

1. Now you just need to add your Controller to the VM of your choice and the VM will see all of the drives on it.

Taken from [here](https://forums.freenas.org/index.php?threads/configure-esxi-to-pass-through-the-x10sl7-f-motherboard-sata-controller.51843/)

# Install a firewall like pfSense into an ESXi Box

This is a bit trickier as you have to reconfigure the structure of the internal switching/routing equipment while connecting to it, but it is pretty straightforward if you know what you are doing. Prerequisite is of course that you need at least two Ethernet ports, one for WAN and one for LAN, otherwise it does not work. 

The full howto is [here](https://doc.pfsense.org/index.php/PfSense_on_VMware_vSphere_/_ESXi)

# USB issues

## GUI is missing the remove option or it is not working due to some error

The ESXi 6.5 GUI is full of bugs, one is pretty annoying: you simply cannot easily remove an already added USB device sometimes, or change it. [This](https://jc-lan.tk/2017/04/17/unable-to-remove-usb-controller-from-esxi-6-5-virtual-machine/) helped me to solve this annoyance.
