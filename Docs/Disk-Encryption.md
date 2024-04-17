# rootfs Disk Encryption

One tool that can be used to increase system security is encryption. This article discusses using sdm to configure encryption for the rootfs partition on a Raspberry Pi system disk. This makes the loss of a disk far less of a problem, since the content cannot be read, nor can the disk be booted,  without knowing the rootfs password.

Although encryption does increase security, it is not a panacea. For instance, consider the following:

* The rootfs password must be typed **on the console** *every* time the Pi reboots, but this can be done remotely; see below re SSH
* rootfs encryption does not make the running system more secure
* This tool does not provide any way to undo disk encryption if you decide you don't want it
  * You'll have to rebuild the system disk
  * **Good news:** if you're using sdm, rebuilding your disk is much less of an issue

With those caveats, if rootfs encryption is useful for you, sdm makes it extremely simple to configure an encrypted rootfs.

**NOTE:** This tool only supports RasPiOS Bookworm and later.

## Overview

There are many articles about rootfs disk encryption on the Internet. If you're interested in learning more about it, your favorite search engine will reveal a bazillion articles on the subject.

Additionally, <a href="https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup">this Wikipedia article</a> discusses LUKS encryption, which is utilized to encrypt your rootfs.

sdm supports rootfs encryption configuration two ways:

* sdm-integrated, using the `cryptroot` plugin
* Standalone on RasPiOS without needing sdm itself by using sdm-cryptconfig on your running system

These two methods are discussed in the following sections.

## sdm-integrated rootfs encryption configuration

### Terse description
The system reboots a couple of times, performing steps during each reboot:

* **First system boot:** At the end, the `sdm-firstboot` service runs, which completes sdm system configuration and reboots
  * An sdm boot script, `/etc/sdm/0piboot/099-enable-auto-encrypt.sh` runs and sets up the service `sdm-auto-encrypt` to run on reboot
* **Second boot:** At the end of system startup, the `sdm-auto-encrypt` service runs which reconfigures the system for encryption and reboots
  * The `sdm-auto-encrypt` service runs `sdm-cryptconfig` to configure the system for encryption and sets up the `sdm-cryptfs-cleanup` service
* **Third boot:** System startup drops into initramfs to perform the encryption process
  * In initramfs use the `sdmcryptfs` command to encrypt rootfs
  * Upon initramfs exit, system continues booting
  * At end of system startup the `sdm-cryptfs-cleanup` service runs that tidies up unneeded services and the initramfs and then reboots
* **Final reboot:** System is now running on an encrypted rootfs
  * All system boots now require the rootfs unlock passphrase to continue

### Details

When customizing an IMG or burning an IMG to an SSD/SD Card, you can start the encryption configuration process by using the `cryptroot` plugin. See <a href="Plugins.md#cryptroot">cryptroot plugin documentation</a> for details on the `cryptroot` plugin. This plugin performs several steps:

* Installs the required crypto software: `cryptsetup`, `cryptsetup-initramfs`, and `cryptsetup-bin`
* Configures sdm to create and run the `sdm-auto-encrypt` service after the system has fully completed the sdm FirstBoot process
* Provides an overview of what steps will be taken to encrypt your system's rootfs

The `sdm-auto-encrypt` service runs the script `sdm-cryptconfig` with arguments indicating that it should operate in sdm-integrated mode. See below for details on sdm-cryptconfig.

sdm-integrated rootfs encryption is nearly fully automatic if you use the sdm switch `--restart`. From the time you first boot your newly-burned disk you'll only need to type 2 commands, both when running in initramfs:
* A command to encrypt rootfs (see below)
* `exit`, to exit the initramfs and continue the system boot on a newly-encrypted rootfs

All other steps are done automatically.

## Standalone rootfs encryption configuration

If your system was not built with sdm (why not?), you can still use a single sdm script to encrypt your rootfs: `sdm-cryptconfig`.

When your system is running, simply download and run sdm-cryptconfig:
```
sudo curl --fail --silent --show-error -L https://github.com/jollycar/sdm/raw/master/sdm-cryptconfig -o /usr/local/bin/sdm-cryptconfig
sudo chmod 755 /usr/local/bin/sdm-cryptconfig
sudo sdm-cryptconfig [optional switches; see below]
```

## The sdm-cryptconfig script

sdm-cryptconfig performs all necessary configuration before the rootfs can be encrypted:

* Downloads the script `sdmcryptfs` from Github to /usr/local/bin if it's not present on the system (sdm-enhanced systems will already have it)
* If not an sdm-enhanced system, installs the required disk crypto software: `cryptsetup`, `cryptsetup-initramfs`, and `cryptsetup-bin`
* If SSH access to initramfs was requested, `dropbear-initramfs` and `dropbear-bin` are also installed
* Updates the initramfs configuration to enable encrypting rootfs (primarily `bash` and `sdmcryptfs`)
* Builds the updated initramfs with encryption support
* Updates /boot/firmware/cmdline.txt, /etc/crypttab, and /etc/fstab for an encrypted rootfs
* If sdm-cryptconfig was run via the cryptroot sdm plugin, it will automatically reboot the system
  * If not, reboot the system manually when you are ready to proceed

### sdm-cryptconfig switches

sdm-cryptconfig has several switches. When using sdm-integrated rootfs encryption the `cryproot` plugin takes care of setting the appropriate switches, based on the arguments to the plugin. All switches are optional.

Switches to sdm-cryptconfig include:

* `--authorized-keys keyfile` &mdash; Specifies an SSH authorized_keys file to use in the initramfs. Required with `--ssh`
* `--crypto crypt-type` &mdash; Specifies the encryption to use. `aes` used by default. Use `xchacha` on Pi4 and earlier for best performance. See Encryption/Decryption performance comparison below.
* `--dns dnsaddr` &mdash; Set IP Address of DNS server
* `--gateway gatewayaddr` &mdash; Set IP address of gateway
* `--hostname hostname` &mdash; Set hostname
* `--ipaddr ipaddr` &mdash; set IP address to use in initramfs
* `--mapper cryptmapname` &mdash; Set cryptroot mapper name [Default: cryptroot]
* `--mask netmask` &mdash; Set network mask for initramfs
* `--quiet` &mdash; Keep graphical desktop startup quiet (see 'known issues' below)
* `--ssh` &mdash; Enable SSH in initramfs
* `--reboot` &mdash; Reboot the system (into initramfs) when sdm-cryptconfig is complete
* `--sdm` &mdash; sdm `cryptroot` plugin sets this
* `--unique-ssh` &mdash; Use a different SSH host key in initramfs than the host OS SSH key

The network configuration switches (dns, gateway, hostname, ipaddr, and mask) are only needed and should only be used if you know that the system is unable to get an IP address and network configuration information from the network (e.g., via DHCP). These settings are ONLY used in the initramfs if SSH is enabled and are not automatically removed.

RasPiOS uses a different mechanism to configure a static network, such as Network Manager, or other network configuration tools.
  
## SSH and initramfs

If the Pi is *headless* (no keyboard/video/mouse) it is quite difficult (OK, it's impossible or close to it) to unlock the rootfs partition. To address this, you can enable SSH in the initramfs. When the system boots you can SSH into the initramfs and you'll be prompted for the rootfs unlock passphrase. After you enter it correctly, the system boot will proceed.

SSH can also be used during the initial rootfs encryption process, discussed in the next section. Everything works exactly the same as if you are sitting on the console, with the exception that log entries (e.g. plugging in a disk) do not show up in the SSH session.

You can use `parted -l` in the initramfs to determine which disks are plugged in to decide, for example, what scratch disk to use with `sdmcryptfs`.

Note that once you've enabled SSH in the initramfs, sdm does not provide an easy way to disable it. That said, it is typically not active for very long, and once encryption has been configured the SSH port is locked down to only prompting for the unlock passphrase.

`sdm-cryptconfig` switches relevant for SSH are:

* `--authorized-keys keyfile` &mdash; Specifies an SSH authorized_keys file to use in the initramfs. This is required with SSH, since there is no password authentication in initramfs
* `--unique-ssh` &mdash; Use a different SSH host key in the initramfs. The default is to use the host OS SSH key
* Network configuration settings &mdash; You may need to use some or all of these depending on your network configuration

### SSH and initramfs notes

Things to know when using SSH as documented here

* You must use `ssh root@ip.ad.dd.rs`. The username `root` is important, as that's how SSH is configured in the initramfs
* You won't be able to SSH using ".local" names when initramfs SSH is running; avahi is not running in the initramfs so the system is unknown to MDNS (that's the protocol that is used for ".local")
* WiFi is not supported for the SSH initramfs connection at the moment

## initramfs

initramfs is one of the first programs run during a RasPiOS system boot. Since the system has been configured to use an encrypted rootfs, initramfs will try to boot using that configuration, and it will fail.

But not for lack of trying! initramfs will try to open the encrypted disk 30 (!) times before it gives up, reports an error, and prompts with:
```
(initramfs)
```
At this point, you need to know two important details:
* The name of your system disk, which will most likely be `/dev/mmcblk0` (integrated SD reader) or `/dev/sda` (a USB-connected device)
* The name of the scratch disk, which must be larger than the *used* space on your rootfs. sdm-cryptconfig will tell you what size this must be

It's best to not plug in the scratch disk until you are at the (initramfs) prompt, because log messages will tell you the name of the disk you just plugged in.

With this information, you'll enter the command (for example):
```
(initramfs) sdmcryptfs /dev/mmcblk0 /dev/sda
```
This will cause sdmcryptfs to encrypt the rootfs on /dev/mmcblk0, using /dev/sda as a scratch disk.

sdmcryptfs will then:

* Print the size of the rootfs
* Save the contents of the rootfs to the scratch disk
* Enable encryption on the rootfs
  * You will be prompted to enter YES (all in upper case) to continue
  * You will then be prompted to provide the passphrase for $rootfs
  *  **Be sure that your CapsLock is set correctly (in case you changed it to type YES)!!!**
* After a short pause you'll be prompted for the passphrase again to unlock the now-encrypted rootfs
* The saved rootfs content will be restored from /dev/sdX to the encrypted rootfs
* When the restore finishes sdmcryptfs will exit and drop you to the (initramfs) prompt
* Type `exit` to continue the boot sequence
* Once the system boot has completed the sdm-cryptfs-cleanup service will run which:
  * Removes some content that is no longer needed (`bash` and `sdmcryptfs`) and rebuilds initramfs 
  * Reboots the system one last time
* As the system reboots you'll once again be prompted for the rootfs passphrase
  * NOTE: Without the 30 tries!
* The system will now ask for the rootfs passphrase like this every time the system boots

**Do not lose or forget the rootfs password. It is not possible to unlock the encrypted rootfs.**

### Sample initramfs/sdmcryptfs console interaction
Lines not preceded by '> yyyy-mm-dd hh:mm:ss' are the output of programs (dd, cryptsetup, resize2fs) that are run by sdmcryptfs.

```
(initramfs) sdmcryptfs /dev/sda /dev/sdb
> 1970-01-01 00:00:59 Shrink partition /dev/sda2 and get its size
> 1970-01-01 00:01:17 Device ‘/dev/sda’ rootfs size 743448 4K blocks (3.0GB; 2.86GiB)
> 1970-01-01 00:01:17 Save rootfs '/dev/sda2' to /dev/sdb
> 1970-01-01 00:01:17 rootfs Save should take less than 3 minutes
743448+0 records in
743448+0 records out
> 1970-01-01 00:02:04 Enable luks2 encryption on '/dev/sda2'
> 1970-01-01 00:02:04 OK to ignore superblock signature warning
> 1970-01-01 00:02:04 Enabling encryption could take up to a minute or two
WARNING: Device /dev/sda2 already contains a 'ext4' superblock signature.
WARNING!
=========
This will overwrite data on /dev/sda2 irrevocably.

Are you sure? (Type ‘yes’ in capital letters): YES
Enter passphrase for /dev/sda2:
Verify passphrase:
Ignoring bogus optimal-io size for data device (33553920 bytes).
> 1970-01-01 00:03:31 Unlock encrypted partition ‘/dev/sda2’
> 1970-01-01 00:03:31 Unlock will take several seconds
Enter passphrase for /dev/sda2:
> 1970-01-01 00:03:41 Restore ‘/dev/sda2’ from /dev/sdb
> 1970-01-01 00:03:41 rootfs Restore should take about the same time as the rootfs Save
743448+0 records in
743448+0 records out
> 1970-01-01 00:04:53 Restore complete; Expand rootfs…
Resize2fs 1.47.0 (5-Feb-2023)
Resizing the filesystem on /dev/mapper/cryptroot to 117076694 (4k) blocks.
The file system on /dev/mapper/cryptroot is now 117076694 (4k) blocks long.

> 1970-01-01 00:04:56 rootfs partition size: 479546138624 Bytes (479.5GB, 446.6GiB)

Enter the 'exit' command to resume the system boot

(initramfs) exit
```

## Exploring and mounting encrypted disks

Encrypted disks can be explored or mounted with the `--encrypted` switch.

## Encryption performance

As hinted above, it's best to use `aes` encryption on the Pi5, which has built-in crypto instructions. All other Pis lack these instructions, so for them `xchacha` is used/recommended.

### Encryption/Decryption performance comparison

Here's a performance comparison on a Pi5, first showing `xchacha`, then `aes`. As you can see, `aes` encryption is more than twice as fast as `xchacha`, and more than 4 times as fast on decryption.
```
pw/ssdy/work$ sudo cryptsetup benchmark -c xchacha20,aes-adiantum-plain64
# Tests are approximate using memory only (no storage IO).
#            Algorithm |       Key |      Encryption |      Decryption
xchacha20,aes-adiantum        256b       394.4 MiB/s       421.6 MiB/s

pw/ssdy/work$ sudo cryptsetup benchmark -c aes
# Tests are approximate using memory only (no storage IO).
# Algorithm |       Key |      Encryption |      Decryption
    aes-cbc        256b       921.8 MiB/s      1885.4 MiB/s
```

On a Pi4, the results are quite different. Using `xchacha` is more than twice as fast as `aes` on both encryption and decryption.
```
p84~$ sudo cryptsetup benchmark -c xchacha20,aes-adiantum-plain64
# Tests are approximate using memory only (no storage IO).
#            Algorithm |       Key |      Encryption |      Decryption
xchacha20,aes-adiantum        256b       170.9 MiB/s       180.0 MiB/s

p84~$ sudo cryptsetup benchmark -c aes
# Tests are approximate using memory only (no storage IO).
# Algorithm |       Key |      Encryption |      Decryption
    aes-cbc        256b        74.8 MiB/s        81.8 MiB/s
p84~#
```


## Known issues

* When running RasPiOS with Desktop (both X11 and Wayland) sdm-cryptconfig will unconditionally make these adjustments to your system:
  * Remove 'quiet' and 'splash' from /boot/firmware/cmdline.txt, making the system boot far less quiet
  * Disable the plymouth splash screen
  * Configure the system so that the boot following the disk encryption boots to the CLI
    * The systemd service `sdm-cryptfs-cleanup` restores the boot to graphical desktop

  The above changes are made to ensure that you have full visibility into what's happening in the boot process. You can override these changes with the `--quiet` switch.

  After the system has fully completed the encryption process, if you want you can `sudoedit /boot/firmware/cmdline.txt` and add `quiet splash`. Also, if you want to re-enable plymouth, do the following once you're logged in:
```sh
for svc in plymouth-start plymouth-read-write plymouth-quit plymouth-quit-wait plymouth-reboot
do
    sudo systemctl unmask $svc
done
```

Encryption configuration is based on https://rr-developer.github.io/LUKS-on-Raspberry-Pi. As of 2024-01-01 it predates Bookworm, but was very helpful in working out the initramfs configuration.


<br>
<form>
<input type="button" value="Back" onclick="history.back()">
</form>
