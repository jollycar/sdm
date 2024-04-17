# Plugins

## Plugin Overview

Plugins are a modular way to extend sdm capabilities. Plugins are similar to <a href="Custom-Phase-Script.md">Custom Phase Scripts</a>, but can work both during customization and/or when burning an SSD/SD Card.

It makes sense to include some plugins into the IMG you're creating (e.g., postfix, samba) so they are installed onto every system burned from that IMG, but some are typically installed once per network (e.g., apt-cacher-ng), or not needed on every system. In that case you can use the plugin when burning the SSD/SD for that specific system.

The set of plugins provided with sdm are documented here.

Other plugins are planned. If there are any specific plugins you're interested in, let me know!

You can add your own plugins as well. Put your plugin script in /usr/local/sdm/local-plugins, and it will be automatically found. sdm looks at local-plugins first, so you can override an sdm-provided plugin with your modifications if desired.

You can also specify the plugin name with a full path. sdm will copy the plugin to /usr/local/sdm/local-plugins if it does not exist or the one specified is newer than the one in local_plugins.

## Invoking a plugin on the sdm command line

Specify each plugin with a separate `--plugin` switch:

```
sdm --plugin samba:"args" --plugin postfix:"args" . . .
```

Multiple `--plugin` switches can be used on the command line. This includes specifying the same plugin multiple times (the `apps` plugin, for example).

Another way to specify plugins is via the `--plugin @/path/to/filelist`, where `filelist` consists of plugin invocations, one per line, without the `--plugin` switch. For example:
```
user:userlist=/rpi/etc/sdm/bls-users
system:name=0|systemd-config=timesyncd=/rpi/systemd/timesyncd.conf|eeprom=stable|sysctl=/rpi/etc/sysctl.d/01-disable-ipv6.conf|fstab=/rpi/etc/fstab.lan|motd=/dev/null
system:name=1||service-disable=apt-daily.timer,apt-daily-upgrade.timer,wpa_supplicant,avahi-daemon,avahi-daemon.socket,ModemManager,rsync,mdadm-shutdown
network:nmconn=/rpi/etc/NetworkManager/system-connections/eth0.nmconnection,/rpi/etc/NetworkManager/system-connections/homewifi.nmconnection|wifissid=myhomewifif|wifipassword=homewifipassword|wificountry=US
disables:triggerhappy|wifi|bluetooth|piwiz
quietness:consoleblank=300|noquiet=keep|nosplash=keep|noplymouth
L10n:host
```

Plugins are run in the order they are encountered on the command line or the plugin @file,

The complete plugin switch format is:
```sh
--plugin plugname:"key1=val1|key2=val2|key3=val3"
```
Enclose the keys/values in double quotes as above if there is more than one key/value or bash will be confused by the "|".

See below for plugin-specific examples and important information.

All files that you provide to sdm, whether on the command line or in arguments to a plugin, must use full paths. For instance, to use a file in your home directory, don't use `file` or `~/file`, use `/home/<mylogin>/file`. Relative file paths may not work because the current directory in which any sdm function is running may change.


## Burn Plugins

Burn plugins are special plugins that are run after a burn has completed on a disk (`--burn`) or disk image (`--burnfile`). Only plugins designated as burn plugins in this document can be used with `--burn-plugin`. sdm doesn't check whether a plugin is burn-plugin capable, so trying to use a non-burn-plugin as a burn-plugin will likely be *interesting*.

Burn plugins can also be used with `--runonly plugins` to operate on an IMG or disk. The `parted` plugin can be used in this manner.
```sh
sdm --runonly plugins --burn-plugin parted:"addpartition=2048,ext4" 2023-12-05-raspios-bookworm-arm64.img
sdm --runonly plugins --burn-plugin extractfs:"rootfs=/path/to/rootfs|bootfs=/path/to/bootfs" 2023-12-05-raspios-bookworm-arm64.img
```

## Plugin ordering notes

There are a couple of plugin ordering issues to be aware of.
* The `user` plugin(s) should be the first plugin. Several other plugins expect this.
* The `cryptroot` plugin must be after the graphics plugin.

## Plugin-specific documentation

### sdm-plugin-template

sdm-plugin-template can be used to build your own plugin. It contains some code in Phase 0 demonstrating some of the things you can do with the *plugin_getargs* function and how to access the results.

### apps

Use the apps plugin to install applications. The apps plugin can be called multiple times on a command line. In that case, each invocation must include the `name=` parameter. The name can be any alphanumeric (including "-", "_", etc.) you want.

#### Arguments

* **apps** &mdash; Specifies the list of apps to install or @filename to provide a list of apps (one per line) to install. Comments are indicated by a pound sign (#) and are ignored, so you can document your app list if desired. If the specified file is not found, sdm will look in the sdm directory (/usr/local/sdm). 
* **name** &mdash; Specifies the name of the apps list. The default name is *default*, but it can only be used once per customization. If you want to use the apps plugin 2 or more times, all plugin instances after the first must have a name provided.
* **remove** &mdash; Specifies the list of apps to remove ofr @filename to provide a list of apps (one per line) to remove. The `remove` argument is processed before the `apps` argument. If you try to remove an apt packge that doesn't exist it will log in /etc/sdm/apt.log and sdm will notify you at the end of the customize: '? apt reported errors; review /etc/sdm/apt.log'

#### Examples

* `--plugin apps:"remove=wolfram-engine|apps=emacs"` &mdash; Remove wolfram-engine, and install emacs
* `--plugin apps:"apps=@my-apps|name=myapps" --plugin apps:"apps=@my-xapps|name=myxapps"` &mdash; Install the list of apps in the file @my-apps, and the list of apps in @my-xapps
* `--plugin apps:"apps=@mycoreapps|name=core-apps"` `--plugin apps:"apps=@myaddtlapps|name=extra-apps"` &mdash; Install the list of apps from @mycoreapps and @myaddtlapps

### apt-addrepo

apt-addrepo adds Repos and gpgkeys to apt

#### Arguments
* **gpgkey** &mdash; /path/to/keyname.gpg
* **gpgkeyname** &mdash; Provide a different filename for the key in /etc/apt/trusted.gpg.d
* **name** &mdash; Name of the repo file in /etc/apt/sources.list.d for a `repo` string
* **repo** &mdash; A repo string that will be written to the named file in /etc/apt/sources.list.d
* **repofile** &mdash; File containing an apt repo that is copied to /etc/apt/sources.list.d

#### Examples

* `--plugin apt-addrepo:"repo=deb  http://repo.feed.flightradar24.com flightradar24 raspberrypi-stable|gpgkey=/path/to/gpgkey.gpg"`
* `--plugin apt-addrepo:"repofile=/path/to/some-repo.list"`

### apt-cacher-ng

apt-cacher-ng installs the RasPiOS apt-cacher-ng service into the IMG or onto the SSD/SD card (If used with `--burn`).

#### Arguments

**NOTE: All arguments are optional**

* **gentargetmode** &mdash; Possible values: 'Set up once', 'Set up now and update later', and 'No automated setup'. [Default: 'No automated setup']. TBH not sure what this does. If you figure it out, let me know ;)
* **bindaddress** &mdash; the IP address to which the server should bind. [Default: 0.0.0.0], which is all IP addresses on the server.
* **cachedir** &mdash; apt-cacher-ng directory. [Default: */var/cache/apt-cacher-ng*]
* **port** &mdash; TCP port [Default: 3142]
* **tunnelenable** &mdash;Do not enable this. [Default: *false*]
* **proxy** &mdash;TBH not sure what this does. If you figure it out, let me know ;)

The default apt-cacher-ng server install uses port 3142. apt-cacher-ng will be enabled by sdm FirstBoot and ready to process requests after the FirstBoot process completes.

NOTE: The apt-cacher-ng plugin installs the apt-cacher-ng *server*. The `--aptcache` command line switch configures the IMG to be an apt-cacher-ng client and use the specified apt-cacher-ng server.

The plugin configures apt as a client to use itself as the apt caching server. This is typically not what you want on every system, so consider using `--plugin apt-cacher-ng` on the `--burn` command line for those systems that will be actually be deployed as apt caching servers.

### apt-file

apt-file installs the *apt-file* command and builds the database. This is very handy for looking up apt-related information.

#### Arguments

There are no `--plugin` arguments for apt-file

### bootconfig

The `bootconfig` plugin configures the contents of /boot/firmware/config.txt.

#### Arguments

* **comment** &mdash; Append the comment to the end of config.txt. Comments can also be specified by an argument starting with `#` or `\n`. In the latter case, the comment is transformed to `\n# comment string` resulting in a blank line before the comment.
* **inline** &mdash; If `inline` is provided as an argument (does not take a value), the plugin will replace existing settings in config.txt (if they exist) with any new value provided to the plugin. If it doesn't exist, or if `inline` is not provided, new arguments are appended to the end of the file.
* **reset** &mdash; If `reset` is provided /boot/firmware/config.txt will be saved as /boot/firmware/config.txt.sdm. If no value is provided for `reset` then /boot/firmware/config.txt will be set to a null file. If `reset=/path/to/file` is provided, the specified file will replace /boot/firmware/config.txt. To work correctly, `reset` must be specified before any other arguments (this is not enforced or specifically logged by sdm).
* **section** &mdash; The `section` argument takes a value like `pi4` or `[pi4]`, and appends the appropriately-bracketed section value to the end of config.txt preceded by a blank line.

* **somename=somevalue** &mdash; All other key/value settings are presumed to be settings in config.txt and added to it. There is no validity checking, so typos are propagated. But, on the other hand, the `bootconfig` plugin doesn't need to be updated every time a brand new setting is added to config.txt.

#### Examples

* `--plugin bootconfig:"section=[pi4]|somesetting=somevalue"`
* `--plugin bootconfig:"inline|hdmi_group=72|hdmi_force_hotplug=1|hdmi_mode=40|hdmi_ignore_edid"` &mdash; The plugin adds the correct value for `hdmi_ignore_edid` (0xa5000080)
* `--plugin bootconfig:"reset|dtparam=audio=on|camera_auto_detect=1|display_auto_detect=1|dtoverlay=vc4-kms-v3d|max_framebuffers=2|arm_64bit=1|disable_overscan=1|section=cm4|otg_mode=1|section=pi4|arm_boost=1|section=all"` &mdash; An identical replacement for the Bullseye /boot/config.txt but with no comments or blank lines
  
### btwifiset

btwifiset is a service that enables WiFi SSID and password configuration over Bluetooth using an iOS app. Once the service is running, you can use the BTBerryWifi iOS app to connect to the service running on your Pi and configure the WiFi. See https://github.com/nksan/Rpi-SetWiFi-viaBluetooth for details on btwifiset itself.

#### Arguments

* **country** &mdash; The WiFi country code. This argument is mandatory
* **localsrc** &mdash; Locally accessible directory where the btwifiset.py can be found, instead of downloading from GitHub
* **btwifidir** &mdash; Directory where btwifiset will be installed. [Default: */usr/local/btwifiset*]
* **timeout** &mdash; After *timeout* seconds the btwifiset service will exit [Default: *15 minutes*]
* **logfile** &mdash; Full path to btwifiset log file [Default: *Writes to syslog*]

### chrony

Chrony installs the chronyd time service.

#### Arguments

* **conf** &mdash; /full/path/to/confname.conf that will be placed into /etc/chrony/conf.d
* **conf2** &mdash; /full/path/to/confname2.conf that will be placed into /etc/chrony/conf.d
* **conf3** &mdash; /full/path/to/confname3.conf that will be placed into /etc/chrony/conf.d
* **sources** &mdash; /full/path/to/sourcename.conf that will be placed into /etc/chrony/sources.d
* **sources2** &mdash; /full/path/to/sourcename2.conf that will be placed into /etc/chrony/sources.d
* **sources3** &mdash; /full/path/to/sourcename3.conf that will be placed into /etc/chrony/sources.d
* **nodistsources** &mdash; Removes the Debian vendor zone pool from chrony.conf

Chrony processes the files in the conf.d and sources.d directories on startup. Having 3 provides flexibility in how these are structured. See `man chrony.conf` for details.

A RasPiOS system should only have one time service enabled. It's up to you to disable others. For instance, on a standard RasPiOS IMG you should add `--svc-disable systemd-timesyncd` to disable the in-built time service, which is enabled by default.

NOTES:
* At least on Bookworm (didn't check earlier versions) installing chrony causes systemd-timesyncd to be removed.
* Adding `iburst` to a `server` or `pool` statement in a sources file seems to result in chrony syncing the time much more quickly

### clockfake

The fake-hwclock provided with RasPiOS runs hourly as a cron job. clockfake does the same thing as fake-hwclock, but you control the interval, and it's always running. Lower overhead, less processes activated, and more control. Life is good.

#### Arguments

* **interval** &mdash; Interval in minutes between fake hardware clock updates

### copydir

Copy a directory tree from the host system into the IMG

#### Arguments
* **from** &mdash; /full/path/to/sourcedir
* **to** &mdash; /full/path/to/destdir
* **nodirect** &mdash; If `nodirect` is specified, the files are staged into the IMG via /etc/sdm/assets. Useful for destination directories that aren't created until later in the customization. Without nodirect the source directory is copied directly to the destination directory.
* **rsyncopts** &mdash; Additional switches for the `rsync` command. If `rsyncopts` is specified, ALL desired rsync switches must be included. If `rsyncopts` is NOT provided, the default switch `-a` is used
* **stderr** &mdash; /path/to/file where stderr from the rsync command is written (D:/dev/null)
* **stdout** &mdash; /path/to/file where stdout from the rsync command is written (D:/dev/null)

The copydir plugin behavior is dependent on whether the `from` file contains a trailing slash, just like the rsync command. **The rsync man page states:** A trailing  slash on the source changes this behavior to avoid creating an additional directory level at the destination.  You can think of a trailing / on a source as meaning "copy the contents of this directory" as opposed to "copy the directory by name", but in both cases the attributes of the containing  directory are transferred to the containing directory on the destination. 

### copyfile

Copy one or more files from the host system into the IMG

#### Arguments

* **from** &mdash; /full/path/to/sourcefile on the host system
* **to** &mdash; /path/in/IMG to place the file in the IMG. This must be a directory, not a file. The directory must already exist
* **chown** &mdash; The `user:group` to set the file ownership
* **chmod** &mdash; The mode to set the file protection (e.g., 755, 644, etc)
* **mkdirif** &mdash; Create the directory if it doesn't exist
* **runphase** &mdash; Normally files are copied to their final destinations in Phase 1. Use `runphase=postinstall` to have a single file copied in the post-install phase
* **filelist** &mdash; The /full/path/to/file of a file on the host OS of a list of files to copy. See below.

The `filelist=/full/path/to/file` option points to a file that consists of one line per file in the format:
```
from=/path/to/file|to=/some/dir|chown=user:group|chmod=filemode|runphase=postinstall
```
chown and chmod are optional. If not specified, the file attributes will not be set, and will be whatever they were on the host system. `runphase` is optional, and if not specified, the file is copied in Phase 1.

copyfile copies the files into the IMG in /etc/sdm/assets/copyfile during Phase 0, and copies them into their target locations in the phase 1 or post-install phase (conditioned on `runphase`) once all packages have been installed, all users have been added, etc.

#### Examples

* `--plugin copyfile:"from=/usr/local/bin/myconf.conf|to=/usr/local/etc"` The config file will be copied from /usr/local/bin/myconf.conf on the host system to /usr/local/etc/myconf.conf in the IMG during Phase1. The file will be owned by the same user:group as on the host, the file protection will be the same as well.
* `--plugin copyfile:"filelist=/usr/local/bin/`. The list of files in the provided `filelist` will be processed per above.

### cryptroot

Configures the rootfs for encryption. See <a href="Disk-Encryption.md">Disk Encryption</a> for complete details

#### Arguments

* **authkeys** &mdash; Provides an SSH authorized keys file for use in the initramfs
* **crypto** &mdash; Specifies the encryption to use. `aes` used by default. Use `xchacha` on Pi4 and earlier for best performance.
* **dns** &mdash; DNS server address for the intramfs network client to use
* **gateway** &mdash; gateway address for the intramfs network client to use
* **ihostname** &mdash; hostname for the intramfs network client to use
* **ipaddr** &mdash; IP address for the intramfs network client to use
* **netmask** &mdash; Network mask for the intramfs network client to use
* **mapper** &mdash; Mapper name for the rootfs encryption (shows up, for instance, in the `df` listing)
* **ssh** &mdash; Enable SSH in the initramfs
* **uniquesshkey** &mdash; Use a unique SSH key in the initramfs. Default is to use the host SSH key (of the system being encrypted)

These are discussed further in the above-mentioned Disk Encryption page.

#### Examples

* `--plugin cryptroot:"authkeys=/home/bls/.ssh/authorized_keys|ssh" Configures the rootfs for encryption and enables SSH into the initramfs with keys authorized in the named authorized_keys file.

### disables

The disables plugin makes it easy to disable a few *complex* functions.

#### Arguments

* **bluetooth** &mdash; Disables bluetooth via a blacklist file in /etc/modprobe.d
* **piwiz** &mdash; Disables piwiz during the first system boot. You must set up everything with sdm that piwiz does or you may not like the results: User, Password, Keymap, Locale, and Timezone.
* **triggerhappy** &mdash; Disable the triggerhappy service. If you're not using it, this will eliminate the log spew it creates
* **wifi** &mdash; Disables WiFi via a blacklist file in /etc/modprobe.d

#### Examples

* `--plugin disables:"bluetooth|piwiz|triggerhappy"` &mdash; Disable Bluetooth, Triggerhappy, and piwiz, but leave WiFi enabled

### explore

The `explore` plugin is a `--burn-plugin` that can be used to explore or mount the newly-burned device after the burn has completed.

### Arguments

* **mount** &mdash; Mount the device into the host system rather than exploring the device container

#### Examples

* `--burn-plugin explore` &mdash; After the burn completes mount the device and enter the container.
* `--burn-plugin explore:mount` &mdash; Like the previous example, but does not enter the container and operates in the context of the host system.

### extractfs

The `extractfs` plugin is a non-general purpose `--burn-plugin` that is used to copy the `boot` and `root` trees from an IMG into directories in the file system

#### Arguments

* **bootfs** &mdash; Directory where the `boot` tree will be written
* **rootfs** &mdash; Directory where the `root` tree will be written
* **img** &mdash; /path/to/IMG.IMG from which the trees will be copied

#### Examples

* `--burn-plugin extractfs:"bootfs=/path/to/bootfs|rootfs=/path/to/rootfs"

### git-clone

Clones the specified repository to the specified directory.

#### Arguments

* `repo` &mdash; Full path to the git repository. Must be network-accessible either via https or some other mechanism (NFS, etc)
* `gitdir` &mdash; Directory into which to place the clone. sdm will do a `mkdir -p` to ensure the directory exists. The clone is done directly into the specified directory
* `gitsw` &mdash; Additional switches to use on the `git` command
* `user` &mdash; User that will be used to run git. The user must already exist
* `preclone` &mdash; Command to run immediately before the clone. If the command starts with `@` it is the name of a script to run
* `postclone` &mdash; Command to run immediately after the clone. Otherwise same as `preclone`
* `gitphase` &mdash; By default the `git` command is run in sdm Phase 1. Specify `gitphase=post-install` to run git in the post-install phase.
* `cert` &mdash; Not Yet Implemented
* `logspace` &mdash; Specify this flag to have the disk space logged immediately before and after the `git clone`

#### Examples

* `--plugin git-clone:"repo=https://github.com/jollycar/sdm|gitdir=/home/bls/work/sdm|user=bls|logspace` &mdash; Clone the sdm repo into /home/bls/work/sdm as user bls. Disk space will be logged before and after the clone.

### graphics

The graphics plugin configures various graphics-related settings. It doesn't do much for wayland at the current time, although you might use it to set the video mode in /boot/firmware/cmdline.txt.

#### Arguments

* **graphics** &mdash; Supported values for the graphics keyword are `wayland` and `X11`. At the present time `wayland` does very little. If graphics is set to `X11`, the Core X11 packages (xserver-xorg, xserver-xorg-core, and xserver-common) are installed if not already installed. In the post-install phase, the plugin will look for a known Display Manager (lightdm, xdm, or wdm), and make appropriate adjustments (see below)
* **nodmconsole** &mdash; If `graphics=X11`, `nodmconsole` directs sdm to NOT start the Display Manager on the console, if the Display Manager is lightdm, wdm, or xdm.
* **videomode** &mdash; Specifies the string to add to the video= argument in cmdline.txt. See below for an example.

wayland is the Default graphics subsystem on Bookworm with Desktop images, so `graphics=wayland` is ignored on those images. The plugin currently will not install wayland on a Bookworm Lite IMG. Wayland is not supported by sdm on releases prior to Bookworm.

If `graphics=X11` and the Display Manager is known, the graphics plugin makes a few adjustments. Specifically:
* If LXDE is installed, the mouse will be set to left-handed if specified on the command line. This works for wayland as well.
* For Display Managers lightdm, wdm, and xdm, sdm will cause the boot behavior you might specify to be delayed until after the First Boot.

The videomode argument takes a string of the form: 'HDMI-A-1:1024x768M@60D'. sdm will add video=HDMI-A-1:1024x768M@60D to /boot/firmware/cmdline.txt

#### Examples

* `--plugin graphics:"graphics=X11|nodmconsole` &mdash; Installs the X11 core components and disables the Display Manager on the console
* `--plugin graphics:"videomode=HDMI-A-1:1920x1280@60D"` &mdash; Sets the specified video mode in /boot/firmware/cmdline.txt

### hotspot

The hotspot plugin configures the system to be a WiFi hotspot. On Bullseye and earlier the hotspot is implemented with hostapd/dnsmasq. On Bookworm it's implemented using Network Manager (nm).

#### Arguments

* **channel** &mdash; Channel to use (hostapd/dnsmasq only) [D:36]
* **config** &mdash; Config file with all the arguments (see Example)
* **country** &mdash; Country code [D:US]
* **device** &mdash; WiFi device name [D:wlan0]
* **dhcprange** &mdash; DHCP range to use for connected devices (hostapd/dnsmasq only) [D:192.168.4.2,192.168.4.32,255.255.255.0]
* **domain** &mdash; Domain name (hostapd/dnsmasq only) [D:wlan.net]
* **enable** &mdash; If **enable=true** set the hotspot to enable as part of system boot [D:true]
* **hwmode** &mdash; WiFi mode (hostapd/dnsmasq only) [D:a]
* **leastime** &mdash; DHCP lease time (hostapd/dnsmasq only) [D:24h]
* **passphrase** &mdash; WiFi hotspot passphrase [D:password]
* **ssid** &mdash; WiFi hotspot SSID [D:MyPiNet]
* **wlanip** &mdash; IP address of the WiFi hotspot [D:192.168.4.1]
* **type** &mdash; Type of hotspot (*routed* or *bridged*) [D:routed]

**Notes:**
* The Network Manager support is complete enough to be functional, but not all the `hotspot` plugin arguments are supported yet.
* The best way to use nm in my opinion is to have pre-created .nmconf and .nmconn files, and then simply dump them into the appropriate nm directories by providing them as `nmconf` and `nmconn` arguments to the `network` plugin.
* FWIW `type=local` hotspot is supported with hostapd/dnsmasq, but not for nm

#### Examples

* `--plugin hotspot:"type=routed|wlanip=192.168.4.1|hwmode=a"`
* `--plugin hotspot:"config=/path/to/config-file"`

The Config file consists of the above arguments (except for `config`), one per line. e.g.,
```
channel=10
device=wlan0
enable=true
wlanip=192.168.10.1
type=bridged
```

### imon

imon installs an <a href="https://github.com/gitbls/imon">Internet Monitor</a> that can monitor:

* **Dynamic DNS (DDNS) Monitor** &mdash; Monitors your external IP address. If it changes changes, your action script is called to take whatever you'd like, such as update your ddns IP address.
* **Network Failover Monitor** &mdash; If your system has two connections to the internet, imon can provide a higher availability internet connection using a primary/secondary standby model.
* **Ping monitor** &mdash; Retrieve ping statistics resulting from pinging an IP address at regular intervals.
* **Up/down IP Address Monitor** &mdash; Monitors a specified IP address, and logs outages.

#### Arguments

There are no `--plugin` arguments for imon

### knockd

knockd installs the knockd service and <a href="https://github.com/gitbls/pktables">pktables</a> to facilitate easier knockd iptables management.

#### Arguments

* **config** &mdash; Full path to your knockd.conf. If **config** isn't provided, /etc/knockd.conf will be the standard knockd.conf
* **localsrc** &mdash; Locally accessible directory where pktables, knockd-helper, and knockd.service can be found, instead of downloading them from GitHub. If there is a knockd.conf in this directory, it will be used, unless overridden with the **config** argument

### L10n

Use the `L10n` plugin to set the localization parameters: `keymap`, `locale`, and `timezone`. You can find the valid values for these arguments with
```
sudo sdm --info keymap     # Displays list of valid keymaps
sudo sdm --info locale     # Displays list of valid locales
sudo sdm --info timezone   # Displays list of valid timezones
```

#### Arguments

* **keymap** &mdash; Specify the keymap to set
* **locale** &mdash; Specify the locale for the system
* **timezone** &mdash; Specify the timezone
* **host** &mdash; Get the above settings from the host sysetm on which sdm is running

**NOTE:** To disable the RasPiOS initial boot query for these configuration items, add `--plugin disables:piwiz` to your customize or burn command line. This works for both Desktop and Lite IMGs.

#### Examples

* `--plugin L10n:"keymap=us|locale=en_US.UTF-8|timezone=Americas/Los_Angeles"`
* `--plugin L10n:"host"`

### lxde

Use the `lxde` plugin to establish your preferred settings, such as left-handed mouse, and config files for `libfm`, `pcmanfm`, and `lxterminal`. These are not well-documented. The best way to create your personalized versions is to use RasPiOS to configure the desktop as you'd like it, and then save the files.

#### Arguments

* **lhmouse** &mdash; Set LXDE for a left-handed mouse
* **lxde-config** &mdash; Specify existing config files for `libfm`, `pcmanfm`, and `lxterminal`. See the example, and see <a href="Using-LXDE-Config.md">Using LXDE configuration</a> for details
* **user** &mdash; The settings apply to the specified user. If no `user` argument is specified, they apply to the first user created with the `user` plugin. The `user` plugin must be specified on the command line before the `lxde` plugin
* **wayfire-ini** &mdash; Specify existing wayfire.ini config file for `wayfire` which is copied to the ~/.config directory of the specified user
* **wf-panel-pi** &mdash; Specify existing wf-panel-pi.ini config file for `wayfire` which is copied to the ~/.config directory of the specified user. HINT: Use `position=bottom` in this file to move the task bar to the bottom of the screen.

#### Examples

* `--plugin lxde:"lxde-config=libfm:/path/to/libfm.conf,pcmanfm=/path/to/pcmanfm.conf,lxterminal=/path/to/lxterminal.conf"`
* `--plugin lxde:"lhmouse|user=someuser"`

### mkdir

Create the specified directory and optionally set directory owner and protection

#### Arguments

* **dir** &mdash; Full path of the directory to create
* **chmod** &mdash; Directory protection
* **chown** &mdash; Directory owner:group

#### Examples

* `--plugin mkdir:"dir=/usr/local/foobar|chown=bls:users|chmod=740"`

NOTE: The directory is created in Phase 0, so it is available as early as during customization. The owner and protection are not set until the post-install phase, since it's possible that the owner account may not be created until Phase 1.

### ndm

The `ndm` plugin installs ndm (https://github.com/gitbls/ndm), named (bind9) and isc-dhcp-server which enables the resulting system to operate as a full DHCP/DNS server with an easy-to-use command-line interface. ndm makes it super-simple to run your own DHCP/DNS server on RasPiOS with useful logging capabilities.

#### Arguments

* **config** &mdash; Existing ndm config file (dbndm.json) to build a new server with an existing ndm config file

#### Examples

* `--plugin ndm` &mdash; Installs ndm, named, and isc-dhcp-server. Both named and isc-dhcp-server services are disabled, and must be re-enabled once ndm has been used to generate the config files for these two services.

### network

Use the network plugin to configure various network settings

#### Arguments

* **netman** &mdash; Specify which network manager to use. Supported values are `dhcpcd`, `network-manager`, and `nm` (short for network-manager). dhcpcd is the default on Bullseye (Debian 11) and earlier, while Network Manager is the default on Bookworm (Debian 12).
* **dhcpcdappend** &mdash; Specifies a file that should be appended to /etc/dhcpcd.conf. Only processed if `netman=dhcpcd`
* **dhcpcdwait** &mdash; Specifies that dhcpcd wait for network online should be enabled. Only processed if `netman=dhcpcd`
* **ssh** &mdash; Accepts one of the values `service`, `socket`, or `none`. The default if `ssh` is not specified is to enable the SSH service
* **wifissid** &mdash; Specifies the WiFi SSID to enable. If `wifissid`, `wifipassword`, and `wificountry` are all set, the network plugin will create /etc/wpa_supplicant/wpa_supplicant.conf (if `netman=dhcpcd`), or will use nmcli during First Boot to establish the specified WiFi connection.
* **wifipassword** &mdash; Password for the `wifissid` network. See `wifissid`
* **wificountry** &mdash; WiFi country for the `wifissid` network. See `wifissid`
* **wpa** &mdash; Specifies the file to be copied to /etc/wpa_supplicant/wpa_supplicant.conf. Only processed if `netman=dhcpcd`. Network Manager does not use wpa_supplicant.conf
* **noipv6** &mdash; Specifies that IPv6 should be disabled. Works with both `netman=dhcpcd` and `netman=nm`
* **nmconf** &mdash; Specifies a comma-separated list of Network Manager config files that are to be copied to /etc/NetworkManager/conf.d (*.conf)
* **nmconn** &mdash; Specifies a comma-separated list of Network Manager connection definitions (each a separate file) that are to be copied to /etc/NetworkManager/system-connections (*.nmconnection)

#### Examples

* `--plugin network:"netman=dhcpcd|noipv6"` &mdash; On Bookworm, set the network manager to dhcpcd (and disable Network Manager), and direct dhcpcd to not request an IPv6 address.
* `--plugin network:"netman=nm|wifissid=myssid|wifipassword=myssidpassword|wificountry=US|noipv6"` &mdash; Use Network Manager to configure the network and also configure the specified WiFi network.

### parted

parted is a `--burn-plugin` that operates on a device, disk, or disk image and enables you to
* Expand the root partition by a specified number of MiB
* Add one or more partitions of a specified size with a specified file system type on it

Using the `parted` burn plugin implicitly sets `--no-expand-root` when used on a burn command.

#### Arguments

* **rootexpand** &mdash; Expand the root partition by the number of MiB specified as the value for this argument. A value of 0 expands the partition to fill the disk. A value of 0 is not allowed when used with `--burnfile`
* **addpartition** &mdash; Adds another partition at the end of the IMG. Arguments: size[fstype][,partitiontype,] where:
    * `size` is the number of MiB for the partition
    * `fstype` is the type of file system. Supported file systems are: `btrfs`, `ext2`, `ext3`, `ext4` [default], `fat16`, `fat32`, `hfs`, `hfs+`, `ntfs`, `reiserfs`, `udf`, and `xfs`. Some file systems may require you to install additional apt packages on the host before running this plugin
    * `partitiontype` is one of `primary` [default], `logical`, or `extended`
    * NOTE: Multiple partitions can be named on the command line by separating them with `+`. See example below.

#### Examples

* `--burn-plugin parted:"rootexpand=2048|addpartition=1024,ext4"` &mdash; Expand the RasPiOS root partition by 2048MiB and add a 1024MiB ext4 partition
* `--burn-plugin parted:"rootexpand=2048|addpartition=1024,ext4+2048,btrfs"`  &mdash; Expand the RasPiOS root partition by 2048MiB and add a 1024MiB ext4 partition and a 2048MiB btrfs partition
* `--burn-plugin parted:"rootexpand=1024|addpartition=@/path/to/partition-list"` where the file partition-list has one line for each partition to be added in the form: `nnnn,fstype,partitiontype`

### piapps

Installs pi-apps (https://github.com/Botspot/pi-apps). That's it!

#### Arguments

* **user** &mdash; Specify the user that piapps should be installed for. The user must already exist. If not specified, the first created user ($myuser) will be used

#### Examples

* `--plugin piapps:"user=bls"` &mdash; Install piapps for user bls. The user was already created with the `user` plugin

### pistrong

<a href="https://github.com/gitbls/pistrong">pistrong</a> installs the strongSwan IPSEC VPN server and `pistrong`. pistrong provides

* A fully-documented, easy-to-use Certificate Manager for secure VPN authentication with Android, iOS, Linux, MacOS, and Windows clients
* Tools to fully configure a Client/Server Certificate Authority and/or site-to-site/host-to-host VPN Tunnels. Both can be run on the same VPN server instance

#### Arguments

* **ipforward** &mdash; Enable IP forwarding from the VPN server onto the LAN. Value can be `yes` or `no` [Default: *no*]

### postfix

postfix installs the postfix mail server. This plugin installs the postfix server but at the moment doesn't do too much to simplify configuring postfix. BUT, once you have a working /etc/postfix/main.cf, it can be fed into this plugin to effectively complete the configuration.

#### Arguments

* **maincf** &mdash; The full /path/to/main.cf for an already-configured /etc/postfix/main.cf. If provided, it is placed into /etc/postfix after postfix has been installed.
* **mailname** &mdash; Domain name [Default: *NoDomain.com*]
* **main_mailer_type** &mdash; Type of mailer [Default: *Satellite system*]
* **relayhost** &mdash; Email relay host DNS name [Default: *NoRelayHost*]

#### Examples

* `--plugin postfix:"maincf=/path/to/my-postfix-main.cf` &mdash; Uses a fully-configured main.cf, and postfix will be ready to go.
* `--plugin postfix:"relayhost=smtp.someserver.com|mailname=mydomain.com|rootmail=myemail@somedomain.com` &mdash; Set some of the postfix parameters, but more configuration will be required to make it operational. A good reference will be cited here at some point.

### quietness

The quietness plugin controls the quiet and splash settings in /boot/firmware/cmdline.txt

#### Arguments

* **consoleblank** &mdash; Set a console blanking timeout (Default: 300 seconds)
* **quiet** &mdash; Enables 'quiet' in /boot/firmware/cmdline.txt
* **noquiet** &mdash; Disable 'quiet' in /boot/firmware/cmdline.txt. If `noquiet=keep` is NOT specified, sdm will re-enable 'quiet' in cmdline.txt after the First Boot.
* **splash** &mdash; Enables 'splash' in /boot/firmware/cmdline.txt
* **nosplash** &mdash; Disable 'splash' in /boot/firmware/cmdline.txt. If `nosplash=keep` is NOT specified, sdm will re-enable 'splash' in cmdline.txt after the First Boot.
* **plymouth** &mdash; Enables Plymouth in /boot/firmware/cmdline.txt. Not Yet Implemented
* **noplymouth** &mdash; Disables the Plymouth graphical splash screen for the First Boot (only). It is re-enabled at the end of First Boot.

#### Examples

* `--plugin quietness:"consoleblank|noquiet=keep|nosplash=keep"` &mdash; Remove 'quiet' and 'splash' from cmdline.txt and do not re-enable them. Console blanking timeout set to 300 seconds (5 minutes)
* `--plugin quietness:"consoleblank=600|noquiet|nosplash|noplymouth"` &mdash; Remove 'quiet' and 'splash' from cmdline.txt, and disable plymouth. All will be re-enabled after the First Boot. Console blanking timeout set to 600 seconds (10 minutes).

### `raspiconfig` 

the `raspiconfig` plugin is used to modify settings supported by `raspi-config`. This is not necessarily the complete list (done quickly), and one or two of these may not be supportable. There's more work to do on this one!

See <a href="https://www.raspberrypi.com/documentation/computers/configuration.html">RaspberryPi Documentation for raspi-config</a> for details.

#### Arguments

* **audio**
* **audioconf**
* **blanking**
* **boot_behaviour, boot_behavior**
* **boot_order**
* **boot_splash**
* **boot_wait**
* **camera**
* **composite**
* **glamor**
* **gldriver**
* **i2c**
* **leds**
* **legacy**
* **memory_split**
* **net_names**
* **onewire**
* **overclock**
* **overlayfs** &mdash; Enables the readonly file system. Optional value specifies whether bootfs should be 'ro' (default) or 'rw'
* **overscan**
* **pi4video**
* **pixdub**
* **powerled**
* **proxy**
* **rgpio**
* **serial**
* **spi**
* **xcompmgr**

#### Examples

* `--plugin raspiconfig:"net_names=1|boot_splash=1"`
* `--plugin raspiconfig:overlayfs=ro` &mdash; Enable the rootfs readonly file system with a read-only bootfs also
* `--plugin raspiconfig:overlayfs` &mdash; Enable the rootfs readonly file system with a read-only bootfs also
* `--plugin raspiconfig:overlayfs=rw` &mdash; Enable the rootfs readonly file system with a read/write bootfs

#### Notes

The 'overlayfs' setting enables the read-only file system. The file system is not made read-only until sdm FirstBoot has completed and the system restarts. If you need a swapfile, you'll need to configure it on another disk or partition, since the boot disk isn't writeable. At the moment sdm doesn't provide any support for swapfile management with overlayfs.

### runatboot

The `runatboot` plugin provides a way to run an arbitrary script during the First Boot of the system. The script is run as root or `user` if specified, with no other provisions or control made by sdm. Behavior, output, logging content, etc is all the responsibility of the script.

#### Arguments

* **script** &mdash; /full/path/to/the/script that should be run
* **args*** &mdash; The arguments to provide to the script
* **user** &mdash; If provided use sudo to run script as the specified user. User must exist at time of First Boot
* **sudoswitches** &mdash; If `user` provided, include these sudo switches
* **output** &mdash; Where to set stdout. Default is /dev/null. The directory must already exist, and the user (root or `user` if specified) must be able to write the output file in that directory
* **error** &mdash; Where to set stderr. Default is the same as stdout (`2>&1`)

#### Example

* `--plugin runatboot:"script=/path/to/script|args=arg1 arg2 arg3"` &mdash; Run the specified script with the 3 provided arguments
* `--plugin runatboot:"user=me|sudoswitches=-H|script=/path/to/script|args=arg1 arg2 arg3"` &mdash; Run the specified script with the 3 provided arguments as the specified user and include `-H` on the sudo command
* `--plugin runatboot:"script=/path/to/script2|args=arg1 arg2 arg3|output=/var/log/myscript.log"` &mdash; Run the specified script with the 3 provided arguments with output and error going to /var/log/myscript.log

### rxapp

**rxapp** is a handy tool to securely and remotely start X11 apps via SSH without a password. You can read about it [here](https://github.com/gitbls/rxapp).

rxapp is included because it is generally useful, but also as a demonstration of how to copy a file from the network (GitHub in this case) into the IMG in a plugin.

#### Arguments

There are no `--plugin` arguments for rxapp

### samba

#### Arguments

* **smbconf** &mdash; Full */path/to/smb.conf* for an already-configured /etc/samba/smb.conf. If provided it is placed into /etc/samba after samba has been installed.
* **shares** &mdash; Full */path/to/shares.conf* for a file containing only the samba share definitions. If provided it is appended to /etc/samba/smb.conf after samba has been installed.
* **dhcp** &mdash; TBH not sure what this does. If you figure it out, let me know ;)
* **do_debconf** &mdash; TBH not sure what this does. If you figure it out, let me know ;)
* **workgroup** &mdash; Workgroup name to replace WORKGROUP in the default /etc/samba/smb.conf. If *smbconf* is specified, the workgroup is NOT modified.

#### Examples

* `--plugin samba:smbconf=/home/bls/mylan-smb.conf` &mdash; Use the provided fully-configured file for /etc/samba/smb.conf
* `--plugin samba:"shares=/home/bls/mysmbshares.conf"` &mdash; Append the provided share definitions to the end of the default /etc/samba/smb.conf
* `--plugin samba:"workgroup=myworkgroup|shares=/home/bls/mysmbshares.conf"` &mdash; Use the default /etc/samba/smb.conf, set the workgroup name to *myworkgroup* and append the provided share definitions to /etc/samba/smb.conf

### serial

The `serial` plugin is used to configure the serial port. Although the `serial` setting on the `raspiconfig` plugin still works, as of 2023-12-28 it prompts, which is obviously annoying when you're in the middle of an sdm customize.

There's a second issue in that the serial setting for the Pi5 is different than for other Pis, and raspi-config checks the system on which it is running, which can likely be incorrect when doing an sdm customize.

The `serial` plugin addresses these issues. You can use it during a customize if you know the target hardware. Otherwise, when you burn the disk for a target system, you can run the plugin then to set it correctly for the target hardware.

#### Arguments

* **enableshell** &mdash; If set, enable a shell on the console serial port.
* **pi5** &mdash; If set, configure the serial port for a Pi5
* **pi5debug** &mdash; If set, configure the debug serial port for a Pi5

#### Examples

* `--plugin serial:enableshell` &mdash; Configure the serial port for a Pi other than a Pi5 and enable a login shell on it
* `--plugin serial:pi5|enableshell` &mdash; Configure the serial port for a Pi5 and enable a login shell on it
* `--plugin serial:pi5debug` &mdash; Configure the debug serial port for a Pi5

### system

The `system` plugin is a collection of system-related configuration settings. You are responsible for using correct file types expected by each function (e.g., .conf, .rules, etc). The plugin does no checking/modification of file types.

If the system plugin is invoked more than once in an IMG, either on customize or burn, you must include the `name=somename` argument for correct operation.

#### Arguments

* **cron-d** &mdash; Comma-separated list of files to copy to /etc/cron.d
* **cron-daily** &mdash; Comma-separated list of files to copy to /etc/cron.daily
* **cron-hourly** &mdash; Comma-separated list of files to copy to /etc/cron.hourly
* **cron-weekly** &mdash; Comma-separated list of files to copy to /etc/cron.weekly
* **cron-monthly** &mdash; Comma-separated list of files to copy to /etc/cron.monthly
* **cron-systemd** &mdash; Takes no value. Switches from using cron to systemd-based cron timers
* **eeprom** &mdash; Supported values are ***critical***, ***stable***, and ***beta***
* **exports** &mdash; Comma-separated list of files to append to /etc/exports
* **fstab** &mdash; Comma-separated list of files to append to /etc/fstab
* **journal** &mdash; Configure systemd journal. Supported values are ***persistent***, ***volatile***, and ***none***. By default Bullseye uses rsyslog and `journal=volatile` while Bookworm uses `journal=persistent`.
    * `persistent`: Makes a permanent journal in /var/log
    * `volatile`: The journal is in memory and not retained across system restarts
    * `none`: There is no system journal
* **ledheartbeat** &mdash; Enable LED heartbeat flash on Pi systems that have /sys/class/leds/PWR/trigger, such as the Pi4 and Pi5.
* **modprobe** &mdash; Comma-separated list of files to copy to /etc/modprobe.d
* **motd** &mdash; Single /path/to/file to use for /etc/motd. /dev/null results in an empty motd
* **name** &mdash; Name of this invocation. This **must** be included if the `system` plugin is invoked more than once in an IMG, including between customize and burn. Best practice to avoid problems is to give each and every invocation a name.
* **rclocal** &mdash; Comma-separated list of ordered commands to add to /etc/rc.local. An item starting with '@' is interpeted as a file whose contents will be included.
* **service-disable** &mdash; Comma-separated list of services to disable
* **service-enable** &mdash; Comma-separated list of services to enable
* **swap** &mdash; **disable** or integer swapsize in MB to set
* **sysctl** &mdash; Comma-separated list of files to copy to /etc/sysctl.d
* **systemd-config** &mdash; Comma-separated list of `type:file`, where type is one of *login*, *network*, *resolve*, *system*, *timesync*, or *user*. Copies the provided file to /etc/systemd/*type*.conf.d
* **udev** &mdash; Comma-separated list of files to copy to /etc/udev/rules.d

#### Examples

* `--plugin system:"cron-d=/path/to/crondscript|exports=/path/to/e1,/path/to/e2"`
* `--plugin system:"systemd-config=timesync=/path/to/timesync.conf,user=/path/to/user.conf|service-disable=svc1,svc2"`
* `--plugin system:"name=s1|cron-d=/path/to/crondscript|exports=/path/to/e1,/path/to/e2" --plugin system:"name=s2|fstab=myfstab`

### trim-enable

trim-enable will enable <a href="https://en.wikipedia.org/wiki/Trim_(computing)">SSD Trim</a> on all or only selected devices. Trim is not actually enabled on the devices until the system first boots.

This plugin can be run manually on a running sdm-customized system by
```
sdm --runonly plugins --plugin trim-enable:"disks=/dev/sda,/dev/sdb"
```
The optional switch `--oklive` can be used to avoid the Prompt "Do you really want to run plugins live on the running host?"

#### Arguments

* **disks** &mdash; Specifies the disks on which to enable trim. `disks=all` will enable trim on all drives. Multiple disk names can be specified by, for example, `disks=/dev/sda,/dev/sdb`. If no disks are specified, `disks=all` is the default.

Additional information on SSD Trim for RasPiOS and Linux can be found <a href="https://forums.raspberrypi.com/viewtopic.php?t=351443">here</a>, <a href="https://lemariva.com/blog/2020/08/raspberry-pi-4-ssd-booting-enabled-trim">here</a>, and <a href="https://www.jeffgeerling.com/blog/2020/enabling-trim-on-external-ssd-on-raspberry-pi">here</a>.

### ufw

Install and configure the ufw firewall

#### Arguments
* **`ufwscript`** &mdash; a list of one or more /path/to/script containing a she-bang (`#!/bin/bash`) and series of one or more ufw commands to configure the firewall. The traditional `sudo` is not required, since the script is run as root. Multiple scripts, if provided, are run in lexical order.
* **`savescriptdir`** &mdash; Specifies a directory where the ufw plugin will save the provided `ufwscript` scripts. If not provided, the scripts will be saved in `/usr/local/bin`.

#### Examples
* `--plugin ufw:"/ufwscript=/path/to/script1,/path/to/script2"` &mdash; Install ufw and configure it with the two provided script files. Save the script files in the IMG in /usr/local/bin
* `--plugin ufw` &mdash; Install ufw, do not configure any rules. ufw documentation says that all inbound network accesses are denied by default

### user

Use the `user` plugin to delete, create, or set passwords for users

#### Arguments

* **userlist** &mdash; Value is a /path/to/file with a list of "commands". See the discussion below
  * Syntax: userlist=/path/to/file
* **log** &mdash; Value is a /path/to/file on the **host** system where the log is to be created. NOTE: The log is written in Phase 0, while the actual user management is done in Phase 1, except for setting Samba passwords, which is done in the post-install phase.
  * Syntax: log=/path/on/host/to/logfile
* **adduser** &mdash; Add the specified user
  * Syntax: `adduser=username`
* **deluser** &mdash; Delete the specified user
  * Syntax: `deluser=username`
* **setpassword** &mdash; Set the password for the specified user. The user must already exist
  * Syntax: `setpassword=username|password=newpassword`
* **addgroup** &mdash; Add a new group
  * Syntax: `addgroup=groupname,gid`
* **homedir** &mdash; Specify the home directory for a new user. Default is /home/username.  A home directory will not be created if `nohomedir` is specified
  * Syntax: `homedir=/home/not-the-usual-place`
* **uid** &mdash; Force the new user's ID to be the given number Default is the next uid to be assigned
  * Syntax: `uid=name-or-number`
* **password** &mdash; Specify the password for `adduser` and `setpassword`
  * Syntax: `password=topsecretpassword`
* **nohomedir** &mdash; Do not create a home directory for this user
  * Syntax: `nohomedir`
* **noskel** &mdash; Do not copy /etc/skel files to the newly-created login directory
  * Syntax: `noskel`
* **nochown** &mdash; Do not set the home directory file ownership. Useful for home directories that need to be secured from their users
  * Syntax: `nochown`
* **Group** &mdash; Set the initial login group
  * Syntax: `Group=primary-group-name`
* **groupadd** &mdash; Augment the user's groups (see `groups` argument) with these. See discussion below
  * Syntax: `groupadd=groups,to,add`
* **groups** &mdash; Set the list of groups for a user. If not specified, `--groups` is used, with the default:
```
dialout,cdrom,floppy,audio,video,plugdev,users,adm,sudo,users,input,netdev,spi,i2c,gpio
```
* **prompt** &mdash; Prompt for the user's password
  * Syntax: `prompt`
* **rootpwd** &mdash; Set the root account password to this user's password
  * Syntax: `rootpwd`
* **redact** &mdash; At the end of `user` plugin processing, redact all passwords
  * Syntax: `redact`
* **nosudo** &mdash; Do not enable this account for `sudo`
  * Syntax: `nosudo`
* **samba** &mdash; Set a Samba username and password for this user
  * Syntax: `samba`
* **smbpasswd** &mdash; Use the provided password for the Samba password instead of the user's password
  * Syntax: `smbpasswd=smbpasswdforuser`
* **shell** &mdash; Set the user's shell
  * Syntax: `shell=/sbin/nologin`

#### Overview and handling multiple accounts

Conceptually, each invocation of the `user` plugin, or each line in a `userlist` file, consists of a *verb* or *directive* and some arguments. Verbs are:
* `adduser` &mdash; Adds the user as described by the rest of the arguments
* `deluser` &mdash; Deletes the specified user
* `setpassword` &mdash; Set the user's password
* `addgroup` &mdash; Add a new group

So, some example lines in a `userlist` (or each set of arguments for several `--plugin user` command line switches) are:
```
deluser=pi
addgroup=myhomegroup,7654
addgroup=demousers
adduser=bls|uid=4321|password=mypassword|groupadd=myhomegroup|Group=users
adduser=demo1|prompt|nohomedir|groups=demousers|nosudo
adduser=demo2|nohomedir|groups=demousers|nosudo|prompt
adduser=demo3|nosudo
setpassword=demo3|password=thenewpassword

```
In the above
* The pi account will be deleted, if it exists
* The two groups will be added. `myhomegroup` will have gid 7654.
* The user `bls` account will be created with the specified `Group`, and a default login group of `users`. The group `myhomegroup` will be added to the default set of groups (`--groups` or the plugin `group` argument).
* The user `demo1` will be created with no home directory. The group `demousers` will be added to the default set of groups for this user. sdm will prompt for the password.
* The user `demo2` will be created, sdm will prompt for the password
* The user `demo3` will be created and a home directory /home/demo3 will be created
* sdm will set the password for demo3 as a separate step (this is not necessary, btw; one could use the `password=` argument on the demo3 line)

  The plugin will prompt for the user's password

The above userlist can be equivalently placed on the command line:
```
--plugin user:"deluser=pi" \
--plugin user:"addgroup=myhomegroup,7654" \
--plugin user:"addgroup=demousers" \
--plugin user:"adduser=bls|uid=4321|password=mypassword|groupadd=myhomegroup|Group=users" \
--plugin user:"adduser=demo1|prompt|nohomedir|groups=demousers" \
--plugin user:"adduser=demo2|nohomedir|groups=demousers|nosudo|prompt" \
--plugin user:"adduser=demo3|nosudo" \
--plugin user:"setpassword=demo3|password=thenewpassword"
```
Plugins are run in the order they are specified on the command line. I recommend that the `user` plugin be as close to the first plugin run as possible, so that the first created user ($myuser) is available to other plugins.

**NOTE:** If you add any users and/or add a password for the user `pi` you probably don't want the RasPiOS services to run at first system boot that help you configure a user. That is exactly what this plugin does, so you can and **should** disable the RasPiOS services with `--plugin disables:piwiz`.

### vnc

Install and configure either or both of Virtual VNC and RealVNC.

#### Arguments

* **vncbase=port** &mdash; Starting port for VNC Servers (default: 5900)
* **realvnc=resolution** &mdash; Install RealVNC server with the specified resolution on the console. The resolution is optional.
* **tigervnc=res1,res2,...resn** &mdash; Install tigervnc server with virtual VNC servers for the specified resolutions
* **tightvnc=res1,res2,...resn** &mdash; Install tightvnc server with virtual VNC servers for the specified resolutions
* **wayvnc[=res]** &mdash; Enable wayvnc server. If resolution is specified, set it has the Wayland headless resolution

Only one of tigervnc or tightvnc can be installed and configured on a system by sdm.

#### Examples

* `--plugin vnc:"realvnc|tigervnc=1280x1024,1600x1200` &mdash; Install RealVNC server for the console and tigervnc virtual desktop servers for the specified resolutions.
* `--plugin vnc:"realvnc=1600x1200"` &mdash; Install RealVNC server and configure the console for 1600x1200, just as raspi-config VNC configuration does.
* `--plugin vnc:"tigervnc=1024x768,1600x1200,1280x1024"` &mdash; Install tigervnc virtual desktop servers for the specified resolutions. Only configure RealVNC if it is already installed (e.g., RasPiOS with Desktop IMG).

#### Additional details

By default Virtual VNC desktops are configured with ports 5901, 5902, ... This can be modified with the `--vncbase` *base* switch. For instance, `--vncbase 6400` would place the VNC virtual desktops at ports 6401, 6402, ... Setting `--vncbase` does not change the RealVNC server port.

For RasPiOS Desktop, RealVNC Server will be enabled automatically. Well, actually, it will be disabled for the first boot of the system as will the graphical desktop, and the sdm FirstBoot service will-reenable both for subsequent use.

For RasPiOS Lite, if the `nodmconsole` keyword is specified to the graphics plugin AND the Display Manager is xdm or wdm, the Display Manager will not be started on the console, and neither will RealVNC Server. It can be started later, if desired, with `sudo systemctl enable --now vncserver-x11-serviced`. Note, however, that you must enable the Display Manager as well for it to really be enabled. To enable the Display Manager:

* **xdm:**&nbsp;`sed -i "s/\#\:0 local \/usr\/bin\/X :0 vt7 -nolisten tcp/\:0 local \/usr\/bin\/X :0 vt7 -nolisten tcp/"  /etc/X11/xdm/Xservers`
* **wdm:** `sed -i "s/\#\:0 local \/usr\/bin\/X :0 vt7 -nolisten tcp/\:0 local \/usr\/bin\/X :0 vt7 -nolisten tcp/"  /etc/X11/wdm/Xservers`

### wificonfig

wificonfig is used to enable the sdm Captive Portal to delay WiFi SSID/Password configuration until the first system boot.

#### Arguments

* **apssid=APSSID** &mdash;SSID for the Access Point. Default: *sdm*
* **apip=ap.ip.ad.dr** &mdash;IP Address for the Access Point. Default: *10.1.1.1*
* **country=cc** &mdash;Two-letter WiFi country code. The codes are found in /usr/share/zoneinfo/iso3166.tab
* **defaults=/path/to/defaults** &mdash;Path to defaults file. See <a href="Captive-Portal.md#defaults-file">Defaults file</a> for details
* **facility=facname** &mdash;Facility name. Default: *sdm*
* **retries=n** &mdash;Maximum number of retries for the user to set the SSID/Password. Default: *5*
* **timeout=n** &mdash;Captive Portal timeout (interval between network packets from the connecting device). Default: *900 seconds* (15 minutes)
* **wifilog=/path/to/wifilog** &mdash;Log file for the Captive Portal. Default: */etc/sdm/wifi-config.log*

### wsdd

wsdd is the Web Service Discovery host daemon. It's very useful in Windows/Samba environments. You can read about it at https://github.com/christgau/wsdd

Note that wsdd is available in Bookworm via apt, so this plugin is not needed on Bookworm (Debian 12) or later, although it can still be used if you prefer.

#### Arguments

* **wsddswitches=switchlist** &mdash; List of switches to write into /etc/default/wsdd
* **localsrc=/path/to/files** &mdash; Local directory with cached copy of wsdd (files: wsdd.py wsdd.8 wsdd.defaults wsdd.service)

<br>
<form>
<input type="button" value="Back" onclick="history.back()">
</form>
