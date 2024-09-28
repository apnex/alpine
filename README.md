## PXE
### Tips and Tricks using iPXE for an automated Alpine netboot
The following is a collection of notes on how I use iPXE to fully automate an Alpine installation.

iPXE is a modern lightweight replacement for the legacy BIOS PXE agent that comes pre-loaded with most systems.  
It is useful for bootstrapping automated OS builds via the network, with a highly flexible scripting syntax.  
iPXE also supports a number of external sources beyond standard TFTP such as:  
- HTTP
- iSCSI
- FC

This provides a more modern method of retrieving necessary configurations scripts, kickstart files and boot images.  
iPXE can be found here:  
https://ipxe.org

### Deployment Methods
iPXE can be setup in a number of ways depending on your situation  
Build options for iPXE are listed here:  
https://ipxe.org/download

#### Chain-loading
This involves leveraging the legacy BIOS PXE TFTP method to load iPXE code, and then call iPXE scripts.  
This uses the existing legacy BIOS PXE code to replace itself, and then proceed with iPXE.  
iPXE scripts can then load other iPXE scripts (via any supported network method - i.e HTTP).  
More info:  
https://ipxe.org/howto/chainloading

#### Embedded ROM
This involves crafting a pre-baked iPXE ROM image, and "flashing" it into the chipset of the machine NIC.
This replaces the existing legacy PXE code.

#### Embedded ISO
This involves crafting a pre-baked bootable ISO CDROM image, and mounting this to a machine to boot from.
This replaces the existing legacy PXE code.

### Unattended Alpine Installation using ISO
Here is an example of how to leverage iPXE to kickoff an unattended Centos installation on a new VM.  
This uses the "embedded iso" method described above.

#### Usage
Simply download the pre-made ISO from here:  
http://pxe.apnex.io/alpine.iso

This ISO has a menu, that let's you select initial network as DHCP or manually enter static addressing

It is a tiny 1MB ISO - as it contains only embedded iPXE code.  
All of the remaining kernel, OS files will be bootstrapped over the Internet via HTTP.  
Just mount this ISO to a CDROM of a VM and power on.  

Minimum Recommended VM Specifications:  
- vCPU: 2  
- MEM: 2 GB  
- DISK: 8 GB  

Boot Order (must be **BIOS**):  
- 1: HDD  
- 2: CDROM

This is to ensure that after installation, the VM will boot normally.  
If CDROM is before HDD, the VM will be in an infinite loop restarting and rebuilding itself!  

After install, the login credentials will be `root` / `google1!`  

#### Backstory Stuff
There's quite a bit going on under the hood for this to work - here is a rough sequence:
- VM boots from ISO
- iPXE bootloader executes embedded script  
-- fetches netboot/initramfs from public http repo  
-- fetches netboot/kernel from public http repo  
-- sets kernel options and boots  
- Alpine kernel begins boot sequence  
-- sets temp local networking, hostname, and package repo  
-- fetches modloop overlay from public http repo  
-- fetches apkovl file from nominated public http repo  
- initial netboot succeeds  
- execute install.sh from apkovl file  
-- read kernel params  
-- configure permanent static networking/dns/hostname  
-- set default root password to `root:google1!`  
-- setup package repo and install os to disk  
-- if BOOTSCRIPT kernel param defined, download ready for next boot  
-- reboot from local installation  
- if BOOTSCRIPT present, execute post restart  

PHEW - luckily all of the above occurs in a relatively quick 2 minutes - minus any additional scripts.

**Minimal `alpine.ipxe` example that uses dhcp**
```
#!ipxe

# set variables
set hostname alpine
set alpine-repo http://dl-cdn.alpinelinux.org/alpine
set version latest-stable
set arch x86_64
set flavor virt
set netboot ${alpine-repo}/${version}/releases/${arch}/netboot
set apkovl http://pxe.apnex.io/alpine/install.apkovl.tar.gz
set bootscript http://pxe.apnex.io/alpine/alpine-runonce.start

# init and set net0
ifopen net0
dhcp

# show net stats
echo ADDRESS-: ${net0/ip}
echo NETMASK-: ${net0/netmask}
echo GATEWAY-: ${net0/gateway}
echo DNS-----: ${dns}
route
ifstat net0

# init kernel with params and boot
initrd ${netboot}/initramfs-${flavor}
kernel ${netboot}/vmlinuz-${flavor} \
    console=ttyS0,19200 \
    ip=dhcp \
    hostname=${hostname} \
    modules=loop,squashfs nomodeset \
    modloop=${netboot}/modloop-${flavor} \
    apkovl=${apkovl} \
    bootscript=${bootscript}
boot
```
**NOTE**  
DHCP is only used to build Alpine over the network.  
Currently `install.sh` converts the dynamic IP into static networking during install.  
This means that the final booted VM will have static IP addressing.

**Minimal `alpine.ipxe` example that embeds static networking**
```
#!ipxe

# set variables
set hostname alpine
set alpine-repo http://dl-cdn.alpinelinux.org/alpine
set version latest-stable
set arch x86_64
set flavor virt
set netboot ${alpine-repo}/${version}/releases/${arch}/netboot
set apkovl http://pxe.apnex.io/alpine/install.apkovl.tar.gz
set bootscript http://pxe.apnex.io/alpine/alpine-runonce.start

# init and set net0
ifopen net0
set net0/ip 172.16.10.10
set net0/netmask 255.255.255.0
set net0/gateway 172.16.10.1
set dns 8.8.8.8

# show net stats
echo ADDRESS-: ${net0/ip}
echo NETMASK-: ${net0/netmask}
echo GATEWAY-: ${net0/gateway}
echo DNS-----: ${dns}
route
ifstat net0

#kernel param syntax for ip=
#https://www.kernel.org/doc/Documentation/filesystems/nfs/nfsroot.txt

# init kernel with params and boot
initrd ${netboot}/initramfs-${flavor}
kernel ${netboot}/vmlinuz-${flavor} \
    console=ttyS0,19200 \
    ip=${net0/ip}::${net0/gateway}:${net0/netmask}:::none:${dns} \
    hostname=${hostname} \
    modules=loop,squashfs nomodeset \
    modloop=${netboot}/modloop-${flavor} \
    apkovl=${apkovl} \
    bootscript=${bootscript}
boot
```

### That's it! - go forth and create / customise further as you see fit
