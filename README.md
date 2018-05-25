## bumblebee-multiscreen-tools
### Scripts to do automatic screen and GPU switching on Nvidia Optimus laptops with [Bumblebee](https://github.com/Bumblebee-Project/Bumblebee).

These scripts allow you to easily switch your laptop's screen from internal to external or enable both screens at once.  
Usage of a docking station is supported, the screen can be switched automatically to/from the one attached to the dock.  
Power management is supported, suitable for both low battery usage in mobile mode and enhanced performance with AC power while still keeping fan noise minimal.

This software has been designed for a Lenovo ThinkPad W530 and the ```Lenovo ThinkPad Mini Dock Plus Series 3``` docking station.  
Other devices are not tested but may work, these scripts are not very hardware dependent.

The key aspect of the hardware this targets is that those laptops, and many other Optimus laptops, do **only** provide access to the external video ports through the Nvidia GPU because they're not physically connected to the Intel GPU.  
This means that in order to use an external screen the Nvidia GPU must be forced to stay enabled, and the Intel GPU's video output must be copied into the video memory of the Nvidia GPU.  
We cannot just render everything with the Nvidia GPU because that would prevent disabling the Nvidia GPU when trying to switch to the internal screen:  
Like the external screen depends on the Nvidia GPU, the internal screen is only accessible through the Intel GPU.

We make this setup usable by:
- using Bumblebee to be able to disable/enable the Nvidia GPU and run a secondary X-Server for it.
- using a magic tool called ```intel-virtual-output``` to copy the content of the video memory of the Intel GPU into the secondary X-Server.

This is preferred over the official Nvidia package ```nvidia-prime``` because that would require logging out and in again to switch the GPU.

### WARNING

This is work in progress! Not only read the [Installation and Configuration section](#installation-and-configuration) but in fact read the whole scripts first before running them!  
**They contain quite a bit of harcoded configuration, you MUST really read the whole scripts!**

This will be finished in 2018, check for new commits regularly.  
This README is currently lacking instructions on how to configure Bumblebee, they will be provided as well.

The Nouveau drivers provide [much more simple configuration of Optimus](https://nouveau.freedesktop.org/wiki/Optimus/) than Bumblebee and if they work for you you should prefer them.  
Once I am done with this repository for Bumblebee I will investigate whether I can adapt it to use Nouveau and if yes, update this README to point to that.

### Features

#### Manual screen switching

```shell
# Read the script and adapt its hardcoded configuration to your environment first!
nano switch-screen
# Switch to DisplayPort #2 on docking station.
# This enables the Nvidia GPU as the DisplayPort can only be accessed through it.
# Software will keep using the Intel GPU unless you run it with "optirun COMMAND" or "primusrun COMMAND".
sudo switch-screen external
# Switch to internal screen.
# This disables the Nvidia GPU for optimal power usage.
sudo switch-screen internal
# Enable both screens with laptop screen being the primary screen, left of the external one.
# This enables the Nvidia GPU just like external mode.
sudo switch-screen dual
```

#### Switching screen automatically with a docking station

If you have a docking station, in particular the "Lenovo ThinkPad Mini Dock Plus Series 3", you can use the ```dock-handler``` script as an ACPI event handler.  
Again, please read and configure the script before using it!

It will automatically switch the screen to the external screen when docked, and to the internal screen when undocked.  
Further, the following powersaving and noise-related optimizations are done by the script:
* When undocked:
  * the Nvidia GPU is disabled by ```switch-screen```.
  * the ```powersave``` [CPU governor](https://www.kernel.org/doc/Documentation/cpu-freq/governors.txt) is used to permanently downlock the CPU to its lowest speed.
  * [Intel Turbo Boost](https://en.wikipedia.org/wiki/Intel_Turbo_Boost) (automatic load-based overclocking) is disabled.
* When docked:
  * the Nvidia GPU is enabled. While this makes sense from a "we're not on battey so no need to save power" perspective it is also strictly necessary: On many laptops, e.g. the ThinkPad W530, the external video outputs are only available through the Nvidia GPU. **NOTICE:** By default all rendering still happens on the Intel GPU. If you want an application to use the Nvidia GPU you must launch it using Bumblebee's ```optirun``` or ```primusrun```.
  * the ```conservative``` CPU governor is used. Compared to the default ```ondemand``` governor this will not instantly clock the CPU to its maximum speed under load but slowly uplock if the load persists. With the ThinkPad W530 this greatly reduces noise when e.g. browsing JavaScript-heavy websites. It is configured as such:
    * if above 80% CPU load for some time upclocking happens.
    * if below 50% CPU load for some time downclocking happens.
    * CPU load of processes with niceness > 0 is ignored when considering whether to upclock. Niceness is the Linux name for process priority, higher values equal lower priority. To launch a command with the default niceness of 10 use ```nice command_name```.
  * Intel Turbo Boost is enabled.

## Installation and Configuration

This has been developed and tested on Kubuntu 14.04.  
In some instances settings for Debian 9 may be mentioned. The Debian 9 settings aren't tested yet, I got them from a friend.  
The names of the mentioned packages may be different on Debian, I did not check what they're called there.

It is assumed that a single external screens is connected to the ThinkPad dock's DisplayPort #2 (visible as "DP-5" in ```xrandr --query``` once you've applied the configuration).  
Port #2 was merely used because it was the most easy to cable for me.

The ```switch-screen``` script assumes the display manager is LightDM. If your distribution uses a different one (which does also apply to newer versions of Kubuntu!) you need to change the ```XAUTHORITY``` variable there.

### Intel GPU drivers

Ensure that ```intel-virtual-output``` is installed - on Ubuntu should be by default as part of the ```xserver-xorg-video-intel``` package:
```shell
which intel-virtual-output || sudo apt-get install xserver-xorg-video-intel
````

Optionally, for debugging purposes:
```shell
sudo apt-get install intel-gpu-tools
# To see the available tools:
dpkg --listfiles intel-gpu-tools
```

### Nvidia GPU drivers

```shell
sudo apt-get install nvidia-384
# Install Bumblebee **after** the Nvidia driver to ensure you don't
# get the older version of the Nvidia driver which Bumblebee chooses
# as dependency. If you want to use a package manager such as aptitude
# you can do this in one step by marking nvidia-384 for installation
# first.
sudo apt-get-install bumblebee-nvidia
```

On my system this resulted in the following packages being installed according to ```/var/log/apt/history.log``` (the order here is random due to using aptitude instead of apt-get):
```
primus-libs-ia32:i386 (0~20131127-2, automatic)
primus-libs:amd64 (0~20131127-2, automatic)
primus-libs:i386 (0~20131127-2, automatic)
nvidia-settings:amd64 (331.20-0ubuntu8, automatic)
libcuda1-384:amd64 (384.90-0ubuntu0.14.04.1, automatic)
bumblebee:amd64 (3.2.1-5, automatic)
bumblebee-nvidia:amd64 (3.2.1-5)
nvidia-384:amd64 (384.90-0ubuntu0.14.04.1)
bbswitch-dkms:amd64 (0.7-2ubuntu1, automatic)
nvidia-prime:amd64 (0.6.2.1, automatic)
nvidia-opencl-icd-384:amd64 (384.90-0ubuntu0.14.04.1, automatic)
primus:amd64 (0~20131127-2, automatic)
```

### Bumblebee configuration

Before we configure Bumblebee please take a moment to understand its purpose:

Bumblebee is a daemon which is meant to disable the Nvidia GPU while it is not used.  
For using the Nvidia GPU it offers two shell commands: ```optirun``` and ```primusrun```.  
They can be used as prefix to any other shell command in order to run that command using the Nvidia GPU.

The difference between the two commands is that ```primusrun``` is a rewrite which uses some newer methods.  
You should prefer primusrun, but notice that in some instances only one of the both commands will work. E.g. Steam's hardware information might show that it is using the wrong GPU.  
If that happens then try again with the other command.

Notice: I haven't verified whether strictly ALL of the below config settings are necessary. I just set them all and then tried whether it works. It is possible that some can be ignored.

```
# Back up the default config in case you need to have a look at it again.
cp -i --preserve=all --no-preserve=links /etc/bumblebee/bumblebee.conf /etc/bumblebee/bumblebee.conf.default
# Edit the config.
# NOTICE: What's listed here is ONLY the settings you have to change as compared to the default
# configuration! Anything which is not listed must be kept as it is in the default config.
nano /etc/bumblebee/bumblebee.conf
	[bumblebeed]
		# Source: https://github.com/Bumblebee-Project/Bumblebee/wiki/Multi-monitor-setup
		KeepUnusedXServer=true
		
		Driver=nvidia
		
		# On Debian 9:
		XorgBinary=/usr/lib/xorg/Xorg
		
	[optirun]
		# On Debian 9:
		PrimusLibraryPath=/usr/lib/x86_64-linux-gnu/primus:/usr/lib/i386-linux-gnu/primus:/usr/lib/primus:/usr/lib32/primus
		
		# Allow optirun/primusrun to fallback to Intel if the Nvidia GPU is unavailable?
		# FIXME: Once everything works well set this to true and change your desktop icons to run important
		# stuff such as browsers with optirun/primusrun to get them to use the Nvidia card opportunistically
		# if you have enabled it currently.
		AllowFallbackToIGC=false
	
	[driver-nvidia]
		# Nvidia kernel module to load.
		# If we used "nvidia-384" only then the "nvidia_drm" and "nvidia_modeset" modules won't be loaded
		# - which normally are part of the NVidia drivers without Bumblebee.
		# Lack of the modeset module would cause the X-Server of bumblebee to not detect any screens (see
		# /var/log/Xorg.8.log).
		# We enforce loading all 3 modules by telling bumblebee to load the drm module first - it loads the
		# modeset module as a dependency, and that loads the main module as a dependency, so all 3 are
		# loaded.
		# On Debian 9 this module might have a different name. To find out the name use
		#   lsmod | fgrep nvidia
		KernelDriver=nvidia-384-drm
		
		# Disable power management because for Laptops where the external screen is only available through
		# the Nvidia GPU we must keep the Nvidia GPU running all the time even if nothing is using it for 3D
		# rendering using Bumblebee's optirun/primus run.
		# Shutting it off when using the internal screen will be handled by bumblebee-multiscreen-tools' own
		# scripts.
		PMMethod=none
		
		# This was determined by taking the existing paths it defaulted to and listing for "nvidia*" in
		# their parent directory.
		LibraryPath=/usr/lib/nvidia-384:/usr/lib32/nvidia-384:/usr/lib/nvidia
		XorgModulePath=/usr/lib/nvidia-384/xorg,/usr/lib/xorg/modules
		# On Debian 9 this might be:
		LibraryPath=/usr/lib/x86_64-linux-gnu/nvidia:/usr/lib/i386-linux-gnu/nvidia:/usr/lib/nvidia
		XorgModulePath=/usr/lib/nvidia/nvidia,/usr/lib/xorg/modules
		
	[driver-nouveau]
		PMMethod=none
```

FIXME: Add X server configuration

### Video acceleration

For the Intel GPU - **FIXME**: Test whether this works with vlc:
```shell
sudo apt-get install libva-intel-vaapi-driver
```

On my system this resulted in the following packages being installed according to ```/var/log/apt/history.log```:
```
i965-va-driver:amd64 (1.3.0-1ubuntu1, automatic)
libva-intel-vaapi-driver:amd64 (1.3.0-1ubuntu1)
```

**FIXME**: While looking for these packages I noticed that searching aptitude for "vaapi" yields the fact that "gstreamer", which is installed on my machine, also has VAAPI plugins which are not installed. Check whether this is used by anything important such as Firefox/Chromium and if yes install the VAAPI plugins.  
**FIXME**: Also check for packages for the competing API "vdpau".

### Automatic display switching with docking station

FIXME

### Detecting dock state at startup/logout and switching screen accordingly

Use ```display-setup-script``` as follows on LightDM:
```shell
# Source: https://wiki.ubuntu.com/LightDM#Adding_System_Hooks
sudo nano /etc/lightdm/lightdm.conf.d/99-bumblebee.conf
	# ATTENTION: Later versions of lightdm (15.10 onwards) have replaced the obsolete [SeatDefaults]
	# with [Seat:*]
	[SeatDefaults]
	display-setup-script=/root/.bin/bumblebee-multiscreen-tools/display-setup-script
chmod 644 /etc/lightdm/lightdm.conf.d/99-bumblebee.conf
```

The script will be run be LightDM right at startup of the X server. It will wait for ```bumblebeed``` to start and then run the ```dock-handler``` script.  
It will also be run by LightDM when you log out, which is important as it then restarts the X server which makes it necessary to re-choose the screen.

## Donations

FIXME

## Known issues and workarounds

### External screen not working

switch-screen may fail to switch to the external screen due to Nvidia GPU initialization failing.  
The kernel log will contain something like:

```
Nov 24 22:40:32 hostname kernel: [351621.701163] NVRM: failed to copy vbios to system memory.
Nov 24 22:40:32 hostname kernel: [351621.701637] NVRM: RmInitAdapter failed! (0x30:0xffff:660)
Nov 24 22:40:32 hostname kernel: [351621.701735] NVRM: rm_init_adapter failed for device bearing minor number 0
```

This is **triggered by high CPU and/or I/O load** - so you can easily avoid it by ensuring the system is idle before you try to switch the screen.  
It should be fixed by newer kernels so consider upgrading your distribution.  
Meanwhile retrying when the system is idle again may work, otherwise a reboot will fix the issue.

### Mouse cursor issues

* When using the external screen the mouse cursor may sometimes not be visible.
* After disconnecting an USB mouse and re-connecting it the cursor may jump around in a strange fashion when you move it.

Any such issues can usually be fixed by switching to the internal screen and then back to the external one.

## Similar tutorials

- http://www.unixreich.com/blog/2013/linux-nvidia-optimus-on-thinkpad-w520w530-with-external-monitor-finally-solved/

## License

Do whatever you want to do with this. Relicense as you please. No warranty.
