![Linux on a classic Butterfly Thinkpad](images/installation-header.jpg)

# Safe Boot: Booting Linux Safely

**Be careful** when following these instructions: it is possible to lock
yourself out from your own machine if you forget some of the passwords.
It is best to try this on a non-production system until you're certain
that you understand how to use the recovery mode to fix bad kernel
signatures or hashes.

This guide was written for a Thinkpad X1 Carbon Gen 6 running
Ubuntu 20.04.  Unfortunately there has been churn in the `tpm2` tools,
so the Debian package does not work on 18.04.

The outline for configuring safeboot requires some knowledge of the
command line and familiarity with running commands as `root` with `sudo`.

* Install the `safeboot.deb` package
* Configure the UEFI firmware
* Create a signing key in a Yubikey
* Sign a recovery kernel
* Install the signing key into the UEFI SecureBoot configuration
* Seal a disk encryption secret into the TPM
* Enable System Integrity Protection mode and sign the runtime kernel


## Initial Setup
This is done once when the system is being setup to use Safe Boot mode.
Note that the hardware token and key signing portions can be done offline
on a separate disconnected machine and then signed keys copied to the machine.

### UEFI firmware configuration
The goal of these configuration changes are to remove several of
the easy attacks such as booting from external disks, modifying kernel or initrd on disk,
changing kernel command line parameters, as well as some of the more esoteric ones
like DMA attacks against the Thunderbolt ports during the boot process.

* SecureBoot setup mode, erase keys
* Supervisor password (do not lose it! it is very difficult to bypass)
* Tamper switches
* Thunderbolt 3
* ???
* TPM clear 

### UEFI Secure Boot signing keys
![Yubikey and Nano](images/yubikey.jpg)

UEFI Secure Boot settings that the system will only run bootloaders that
are signed by keys in the SPI flash.  By default these keys are the OEM and
Microsoft, and Microsoft will sign anyone else's key for $99, so it is
important to replace these keys with ones under control of the computer
owner. There is typically a signed "shim" that transfers control to the
unsigned grub bootloader and kernel, which is problematic for security.

A safer way to boot is to package the kernel, initrd and command line
into a single EFI executable that is signed by the computer owner's
key.  This also reduces the attack surface by removing `grub` and
the overly complicated config files, as well as speeds up the boot
process a little bit.

The owner key can be generated by openssl and stored offline, although
using a hardware token like a yubikey greatly enhances the security
of the system since even with root access an attacker can't
gain persistence in the `/` or in the kernel.  

![UEFI SecureBoot setup screen](images/thinkpad-x1-secureboot.jpg)
To replace the Platform Key requires that the UEFI SecureBoot
firmware be put into "`Setup mode`".  On the Thinkpads, select
`Reset to Setup Mode` and `Clear All Secure Boot Keys`, then boot
into Linux.

![Output of `yubikey-init` command](images/yubikey-init.png)

First step is to generate a new key that will be used for UEFI SecureBoot.
Skip this if you have already initialized your hardware token.
The Yubikey needs to have CCID mode enabled and a key generated.
The `safeboot yubikey-init` subcommand will do several steps:

* Use `ykpersonalize` to enable CCID mode. (TODO)
* Generate a new private key inside the Yubikey and export the public key
* Self-sign the public key to generate a new x509 certificate
* Reimport the certificate into the Yubikey so that UEFI variables and images
can be signed with the hardware token.

![Output of `uefi-sign-keys` command](images/uefi-sign-keys.png)

Once the key has been generated, the x509 certificate needs to be
reformatted to be added to the UEFI Secure Boot platform key (`PK`),
Key-Exchange Key (`KEK`) and as a key database entry (`db`).  Each
of these updates needs to be signed with the Platform Key; in the
case of the PK it is self-signed.

The `safeboot uefi-sign-keys` subcommand will:

* Generate the three EFI public key variables updates
* Sign the EFI variables with the yubikey (will require a PIN authentication
for each variable)
* Store public certificate in UEFI platform key (`PK`), key-exchange key (`KEK`) and database (`db`)

**Do not reboot yet!** You have to sign the kernel first, or else the
firmware will refuse to load it.

### Signed Linux setup
![Output of `sign-kernel`](images/sign-kernel.png)

The first step is to sign and install a recovery kernel, which will be able
to read/write mount the root filesystem, and does not have TPM sealing keys,
so it will always require a recovery password.  If you don't have a recovery
entry on the disk, you would need to have a USB drive signed with a key
in the UEFI `db` to recover from errors.

The `safeboot sign-kernel recovery` command will:

* Add UEFI boot menu item for recovery kernel
* Create a directory for it in the EFI System Partition ("ESP")
* Merge the vmlinux, initrd and command line into a single EFI executable
* Sign the merged EFI executable

Typically you will not have to redo this command since the normal
kernel will be hashed and signed during updates.


### TPM Configuration

![Output of `luks-seal` subcommand](images/luks-seal.png)

The Trusted Platform Module serves two purposes in securing the process:
it helps validate that the firmware and boot configuration is unchanged,
and it streamlines the boot process by providing the key for encrypted disks.
On modern systems Boot Guard ensures that the initial boot block is
measured into the TPM, so a local attacker shouldn't be able to modify
the SPI flash contents to bypass the measurements, or to add their
own keys to the UEFI key database.  (Subject to various CVE's and TOCTOUs, etc)

The `safeboot luks-seal` command will:

* Create a new disk encryption key with random data
* Read the current PCRs, seal new key into the TPM with these PCRs
* Add new key slot to disk with new key (will require the recovery password)
* Add initramfs building hooks
* Modify `/etc/crypttab` entry to call unlock script if necessary

The very first time this is run you will need to also run
`update-initramfs -u` to build an initrd with the new crypttab and
hooks in place.

The TPM PCRs are somewhat fragile; they include hashes of the ROM images,
the EFI executables that have been run along the boot path, etc.
For example, entering the UEFI `Setup` menu will cause different measurements,
so the TPM will not automatically unseal on the same boot that the
user has entered the setup application.

### System Integrity Protection mode

The "SIP" mode ensure that even an attacker with root priviledges
can't make persistent changes to the contents of the root filesystem.
The root of the dm-verity Merkle-tree is passed to the kernel as part of the
signed command line, ensuring that an attacker can't change anything
on the filesystem without access to the signing key.
Any modifications to the filesystem will be detected when the modified
blocks are read, allowing the system to enter recovery mode to protect
its data.

#### RO and RW Partition setup

![Resizing `/dev/mapper/vgubuntu-root`](images/lvresize-root.jpg)

There is some manual work in a live CD that needs to be done to prepare
the system for SIP mode (since `/` can't be mounted while it is worked on);
it is easiest to do this **before** setting
the SecureBoot config so that you can still boot a stock Live CD.
Ubuntu installer used to allow you to configure the partitions at
install time, but for some reason it no longer does so.

The root filesystem needs to be resized so that hash computation doesn't
take forever, `/home` and `/var` need to be split into a separate mounts,
`/tmp` needs to be a symlink to `/var/tmp`, etc.

```
fsck -f /dev/vgubuntu/root
resize2fs /dev/vgubuntu/root 8G
lvresize --size 8G /dev/vgubuntu/root
```

Then the new `/var`, `/home` and dmverity hash partitions have to be created:
```
lvcreate --size 16G -n var vgubuntu
lvcreate --size 80G -n home vgubuntu
lvcreate --size 1G hashes vgubuntu
mkfs.ext4 /dev/gvubuntu/home
mkfs.ext4 /dev/gvubuntu/var
```

And then the various partitions need to be adjusted in the fstab:
```
mkdir /root /tmp/var /tmp/home
mount /dev/vgubuntu/root /root
mount /dev/vgubuntu/var /tmp/var
mount /dev/vgubuntu/home /tmp/home
mv /root/var/* /tmp/var/
mv /root/home/* /tmp/home
echo >> /root/etc/fstab '/dev/mapper/vgubuntu-var /var ext4 defaults 0 1'
echo >> /root/etc/fstab '/dev/mapper/vgubuntu-home /home ext4 defaults 0 1'
rm -rf /root/tmp
ln -s ./var/tmp /root
```

Once this is done you should be able to reboot and continue
with the safeboot setup.


#### Hasing and signing the RO root filesystem

![Output of `hash-and-sign` command](images/hash-and-sign.png)

The `safeboot hash-and-sign` command will:

* unmount `/` and remount it `ro`
* `fsck /` to ensure that it is clean
* run `veritysetup format` to compute the Merkle-tree for `/dev/vgubuntu/root`
* write the hashes to `/dev/vgubuntu/hashes`
* merge the linux kernel, initrd, and a command line with the dmverity root hash into an EFI executable
* sign this executable
* and install it as the default entry in the EFI boot manager

Because this might change the EFI boot manager, it is necessary to
reboot into the SIP mode and then re-seal the TPM key with
`safeboot luks-seal` as documented above.

TODO: Detect changes and notify the use rather than just `EIO`.

When changes to `/` need to be made, such as to install new packages
with `apt-get`, the system has to be rebooted into recovery mode
with `safeboot recovery reboot`.  After entering the recovery key
during the reboot, the root file system has to be remounted `rw`, the
updates applied, and then a new hash tree generated and signed.
Alternatively, two `/` filesystems can be on the disk. Updates are
applied to the unused one and signed, and then on the next boot
the correct root filesystem is selected.

* For dm-verity and read-only root:
* Separate `/`, `/var` and `/home`
* (need to show how to do this post install for 20.04)
* Two copies of `/` for A-B switching and atomic updates

## Updates
Nothing ever stays static, so it is important to consider the steps
for updating the system.  Frequently the root file system and kernel
will be updated together, so these steps can be batched.  Hopefully
TPM re-sealing doesn't have to happen as often, since it requires
access to the disk encryption recovery key.

### Root filesystem updates
* Remount `/` read-write
* Perform updates
* Remount `/` read-only
* Compute block hashes and root hash
* Optionally regenerate the initrd
* Add root hash to command line
* Sign kernel/initramfs/commandline
* These could be done offline and the block image pushed to the system for installation

### Kernel and initramfs update
* Re-generate `/boot/initrd` and `/boot/vmlinuz`
* Merge the kernel, initrd and command line into a single EFI executable
* Use the hardware token to sign that executable
* Copy the signed image to the EFI boot partition
* These could be done offline and the block image pushed to the system for installation

### UEFI firmware update
If there are any updates to the UEFI firmware, such as changing the
`Setup` variable, then the TPM sealed keys will no longer be accessible
and the recovery key will be necessary to re-seal the drive.  This
requires access to the recovery key since a new random key is generated
and needs to be stored into a keyslot on the drive.