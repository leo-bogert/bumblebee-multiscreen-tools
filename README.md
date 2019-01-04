## bumblebee-multiscreen-tools
### Scripts and HOWTO for automatic screen and GPU switching on Nvidia Optimus laptops with [Bumblebee](https://github.com/Bumblebee-Project/Bumblebee)

There are many laptops nowadays which have both an Intel and a Nvidia GPU, which is advertised as `Nvidia Optimus`. The Intel GPU is typically hardwired to the internal screen, while the Nvidia one connects to the external outputs.

These scripts and configuration instructions allow you to easily switch your laptop's screen from internal to external or enable both screens at once.  
Usage of a docking station is supported, the screen can be switched automatically to/from the one attached to the dock.  
Power management is supported, suitable for both low battery usage in mobile mode and enhanced performance with AC power while still keeping fan noise minimal.

This software has been designed for `Lenovo ThinkPad W530` laptops using the ```Lenovo ThinkPad Mini Dock Plus Series 3``` docking station.  
Other devices are not tested but may work, these scripts are not very hardware dependent.

The key aspect of the hardware this targets is that those laptops, and many other Optimus laptops, do **only** provide access to the external video ports through the Nvidia GPU because they're not physically connected to the Intel GPU.  
This means that in order to use an external screen the Nvidia GPU must be forced to stay enabled, and the Intel GPU's video output must be copied into the video memory of the Nvidia GPU.

We make this setup usable by:
- using Bumblebee to be able to disable/enable the Nvidia GPU and run a secondary X-Server for it.
- using a magic tool called ```intel-virtual-output``` to copy the content of the video memory of the Intel GPU into the secondary X-Server.

This is preferred over the official Nvidia package ```nvidia-prime``` because that would require logging out and in again to switch the GPU.

### WARNING

While this README will provide full instructions on how to configure your system, the included scripts **ALSO** need to be configured which is **NOT** fully explained in this README. They need to be adapted to things such as especially your screen-resolution.  
**Thus please fully read and configure all of the included scripts before you run them!**

You will get to understand the context of what these scripts do by reading this README, so start with that and review and configure the scripts as they're mentioned.

With regards to the state of this repository as a whole:  
It should work, but lacks polishing (see [the TODO list](TODO.md)). That will be finished in 2019, check for new commits regularly.  

Most notably: The Nouveau drivers provide [much more simple configuration of Optimus](https://nouveau.freedesktop.org/wiki/Optimus/) than Bumblebee - however there currently does not seem a toolchain available for them like this repository here. So ideally this repository would be converted to use them instead.  
Once I am done with this repository for Bumblebee I will investigate whether I can create a fork which uses Nouveau instead and if yes, update this README to point to that.

### Features

#### Manual screen switching

```shell
# Switch to DisplayPort #2 on docking station.
# This enables the Nvidia GPU as the DisplayPort can only be accessed through it.
# Software will keep using the Intel GPU unless you run it with "optirun -- COMMAND"
# or "primusrun -- COMMAND".
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
    * CPU load of processes with niceness > 0 is ignored when considering whether to upclock. Niceness is the Linux name for process priority, higher values equal *lower* priority. To launch a command with niceness > 0 use ```nice command_name```.
  * Intel Turbo Boost is enabled.

## Installation and Configuration

This has been developed and tested on Kubuntu 14.04.  
In some instances settings for Debian 9 may be mentioned. The Debian 9 settings aren't tested yet, I got them from a friend.  
The names of the mentioned packages may be different on Debian, I did not check what they're called there.

It is assumed that a single external screen is connected to the ThinkPad dock's DisplayPort #2 (visible as `VIRTUAL8` or `DP-5` in ```xrandr --query``` once you've applied the configuration).  
Port #2 was merely used because it was the most easy to cable for me.  
If you want to use a different port you can change it in the `switch-screen` script later on, you'll be reminded about it.

The ```switch-screen``` script assumes the display manager is LightDM. If your distribution uses a different one (which does also apply to newer versions of Kubuntu!) you need to change the ```XAUTHORITY``` variable there. This will be mentioned later on again.

We will now first configure the standard system drivers and Bumblebee before actually installing `bumblebee-multiscreen-tools`.

### BIOS settings

For the ThinkPad W530 use the following:
```
Config / Display / OS Detection for NVIDIA Optimus: On
```

While this may not support Linux it is better to keep it `On` anyway: I once read that some (unrelated) hardware drivers of the Linux kernel are in fact able to fake being a Windows kernel to ensure the hardware fully enables all of its features. This may apply to Optimus as well.

Also Optimus does work for me with this being `On`.

### Intel GPU drivers

Ensure that ```intel-virtual-output``` is installed - on Ubuntu should be by default as part of the ```xserver-xorg-video-intel``` package:
```shell
which intel-virtual-output || sudo apt install xserver-xorg-video-intel
````

Optionally, for debugging purposes:
```shell
sudo apt install intel-gpu-tools
# To see the available tools:
dpkg --listfiles intel-gpu-tools
```

### Nvidia GPU drivers

```shell
sudo apt install nvidia-384
# Install Bumblebee **after** the Nvidia driver to ensure you don't
# get the older version of the Nvidia driver which Bumblebee chooses
# as dependency. If you want to use a package manager such as aptitude
# you can do this in one step by marking nvidia-384 for installation
# first.
sudo apt install bumblebee-nvidia
```

On my system this resulted in the following packages being installed according to ```/var/log/apt/history.log``` (the order here is random due to using aptitude instead of apt):
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

```bash
sudo -i
# Back up the default config in case you need to have a look at it again.
cp -i --preserve=all --no-preserve=links /etc/bumblebee/bumblebee.conf \
    /etc/bumblebee/bumblebee.conf.default
# Edit the config.
# NOTICE: What's listed here is ONLY the settings you have to change as compared to the default
# configuration! Anything which is not listed must be kept as it is in the default config.
nano /etc/bumblebee/bumblebee.conf
    [bumblebeed]
    # Keep the X server running permanently even if the Nvidia GPU isn't used by optirun/primusrun
    # because the external display ports are only available through the Nvidia GPU and hence we need
    # to keep its server available.
    # Shutting it off when using the internal screen will be handled by bumblebee-multiscreen-tools' own
    # scripts.
    KeepUnusedXServer=true
    
    Driver=nvidia
    
    # On Debian 9:
    XorgBinary=/usr/lib/xorg/Xorg
        
    [optirun]
    # On Debian 9 (in one line!):
    PrimusLibraryPath=/usr/lib/x86_64-linux-gnu/primus:/usr/lib/i386-linux-gnu/primus
        :/usr/lib/primus:/usr/lib32/primus
    
    # Allow optirun/primusrun to fallback to Intel if the Nvidia GPU is unavailable?
    # TODO: Once everything works well set this to true and change your desktop icons to run important
    # stuff such as browsers with optirun/primusrun to get them to use the Nvidia card opportunistically
    # if you have enabled it currently.
    # On the other hand it may be nice to keep this disabled because the Intel GPU probably causes less
    # fan noise.
    AllowFallbackToIGC=false
    
    [driver-nvidia]
    # Nvidia kernel module to load.
    # If we used "nvidia-384" only then the "nvidia_drm" and "nvidia_modeset" modules won't be loaded
    # - which normally are part of the Nvidia drivers without Bumblebee.
    # Lack of the modeset module would cause the X-Server of bumblebee to not detect any screens (see
    # /var/log/Xorg.8.log).
    # We enforce loading all 3 modules by telling bumblebee to load the drm module first - it loads the
    # modeset module as a dependency, and that loads the main module as a dependency, so all 3 are
    # loaded.
    # On Debian 9 this module might have a different name. To find out the name use
    #   lsmod | fgrep --ignore-case nvidia
    KernelDriver=nvidia-384-drm
    
    # Keep Nvidia GPU enabled all the time to keep the external video ports available.
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

### Xorg configuration

There will be two Xorg servers running with our configuration:
- Display ```:0``` - The default main server which is spawned by LightDM and runs on the Intel GPU.
  Its log file is at ```/var/log/Xorg.0.log```.
- Display ```:8``` - Bumblebee's secondary server for the Nvidia GPU. It will only be running while
  the Nvidia GPU is enabled. The log file is  ```/var/log/Xorg.8.log```.

#### Intel Xorg configuration

```bash
sudo -i
cd /usr/share/X11/xorg.conf.d/
# Create the file, it shouldn't exist yet
touch 20-thinkpad-w530-intel-gpu.conf
chown root:root 20-thinkpad-w530-intel-gpu.conf
chmod 644 20-thinkpad-w530-intel-gpu.conf
nano 20-thinkpad-w530-intel-gpu.conf
    Section "Device"
        Identifier "intelgpu0"
        Driver "intel"
        
        # TODO: Those don't seem to help on the external screen. Maybe it wrongly syncs to internal screen?
        # TODO: Test whether they at least help for the internal screen, if not remove.
        Option "TearFree" true
        Option "VSync" "true"
        
        # TODO: The following may be needed for video acceleration of some codecs. See "man 4 intel".
        # But VLC's stderr does say h264 acceleration by VA-API works with the Intel driver *without* this.
        # This may be because IIRC somewhere on Internet it was said that XvMC is an outdated acceleration
        # approach which has been superseded by something else.
        #
        # Notice if you plan to enable this:
        # The man page sounds like this will need additional config in an "XvMCConfig file", read the
        # sections of the below options before enabling them.
        
        #     Option "XvMCSurfaces" 7
        #     Option "XvMC" true
        
        # TODO: The following may be needed on Debian 9 - validate whether it really is.
        
        #     Option "VirtualHeads" "2"
        
        # Add your options here. See: man 4 intel
        # TODO: Read the whole manpage and consider whether there are other useful ones
    EndSection
```

#### Nvidia Xorg configuration
```bash
sudo -i
# Back up the default config in case you need to have a look at it again.
cp -i --preserve=all --no-preserve=links /etc/bumblebee/xorg.conf.nvidia \
    /etc/bumblebee/xorg.conf.nvidia.default
nano /etc/bumblebee/xorg.conf.nvidia
    # As opposed to previous configuration instructions hereby the *full* file is listed, not just
    # the changes as compared to the defaults.
    # For documentation of the options used here and further possible ones see:
    # http://us.download.nvidia.com/XFree86/Linux-x86/384.90/README/xconfigoptions.html
    
    Section "ServerLayout"
        Identifier  "Layout0"
        # Automatically detecting screens as they are attached
        Option      "AutoAddDevices" "true"
        # Don't automatically add the GPU(s) because we want to manually chose which one to use.
        Option      "AutoAddGPU" "false"
    EndSection
    
    Section "Device"
        Identifier  "DiscreteNvidia"
        Driver      "nvidia"
        VendorName  "NVIDIA Corporation"
        
        # Tell driver which GPU to use precisely.
        # If the X server does not automatically detect your GPU you can manually set it here.
        # To get the BusID prop run "lspci | egrep 'VGA|3D'".
        # This setting may be needed in some platforms with more than one Nvidia card.
        # Also may be needed on Ubuntu 13.04.
        # On Debian 9 you might have to remove this.
        # (I don't remember where I read all of the above, sorry.)
        BusID "PCI:01:00:0"
        
        # Setting ProbeAllGpus to false prevents the Nvidia driver from trying to control the other
        # GPU which is already being managed outside bumblebee.
        # Required on platforms with more than one Nvidia GPU, e.g. Macbook Pro pre-2010 with Nvidia
        # 9400M + 9600M GT.
        Option "ProbeAllGpus" "false"
        
        # TODO: This option isn't mentioned in the current documentation anymore, probably obsolete?
        # Likely was intended to avoid displaying the Nvidia logo for some seconds at startup.
        Option "NoLogo" "true"
        # From the documentation:
        # "Enabling this option makes sense in configurations when starting the X server with no
        # display devices connected to the NVIDIA GPU is expected, but one might be connected later"
        Option "AllowEmptyInitialConfiguration" "true"
        # Auto-detect your monitor's resolution etc.
        Option "UseEDID" "true"
    EndSection
    
    # On Ubuntu prevent startup of Bubmblee's Xorg from failing due to trying to use the Intel
    # driver on the Nvidia GPU, see "/var/log/Xorg.8.log".
    Section "Screen"
        Identifier "Screen0"
        Device "DiscreteNVidia"
    EndSection
```

#### Main Xorg configuration

In theory there should not be a ```/etc/X11/xorg.conf``` with this setup:  
Bumblebee is supposed to manage the X config, which is e.g. shown by there not being an
```/etc/X11/xorg.conf``` shipped by the Bumblebee packages.

However, without providing one as specified below, the X server will die when you log out after having
switched to external screen and back to internal screen.

This is because, at least on my Kubuntu 14.04, upon logout something creates a ```/etc/X11/xorg.conf```
which tells X to use the Nvidia GPU - but the Nvidia GPU is disabled when using the internal screen so
starting X will fail.  
**TODO**: I had determined the above before I specified a display-setup-script for LightDM. Check whether
it still applies with the script.

So we provide our own X11 config which tells the X server to use the Intel GPU by default.

Remember: Bumblebee works by always having the video output be composed on the Intel GPU. Rendering
something on the Nvidia GPU means it is rendered on the secondary Nvidia X-server and then copied into the
video memory of the primary X-server which runs on the Intel GPU.  
So the config we hereby provide is for the said primary X-server.

```bash
sudo -i
cd /etc/X11/
mv xorg.conf xorg.conf.default.bumblebee
touch xorg.conf
chown root:root xorg.conf
chmod 644 xorg.conf
nano xorg.conf
    # Based on our above:
    # /etc/bumblebee/xorg.conf.nvidia
    # /usr/share/X11/xorg.conf.d/20-thinkpad-w530-intel-gpu.conf
    # TODO: Could the latter config file be removed and all of it instead be placed in this one here?
    
    Section "ServerLayout"
        Identifier  "Layout0"
        Option "AutoAddDevices" "true"
        Option "AutoAddGPU" "false"
    EndSection
    
    Section "Device"
        Identifier "intelgpu0"
        Driver "intel"
        Option "TearFree" "true"
        Option "VSync" "true"
    EndSection
    
    Section "Screen"
        Identifier "Screen0"
        Device "intelgpu0"
    EndSection
# Make file immutable so whatever tries to create it automatically cannot delete/modify it.
chattr +i xorg.conf
```

### Kernel modules

We have to prevent the Nvidia driver from being loaded at boot, that would cause the Bumblebee X-server to
fail starting. Symptoms would be ```/var/log/Xorg.8.log``` saying:

```
[   147.859] (II) NVIDIA(0): NVIDIA GPU Quadro K2000M (GK107GL) at PCI:1:0:0 (GPU-0)
[   147.859] (--) NVIDIA(0): Memory: 2097152 kBytes
[   147.859] (--) NVIDIA(0): VideoBIOS: 80.07.31.00.18
[   147.859] (II) NVIDIA(0): Detected PCI Express Link width: 16X
[   147.859] (EE) NVIDIA(GPU-0): Failed to acquire modesetting permission.
[   147.859] (EE) NVIDIA(0): Failing initialization of X screen 0
[   147.859] (II) UnloadModule: "nvidia"
[   147.859] (II) UnloadSubModule: "wfb"
[   147.859] (II) UnloadSubModule: "fb"
[   147.859] (EE) Screen(s) found, but none have a usable configuration.
```

To prevent the modules from loading we do this:

```bash
sudo -i
# Preserve default configuration
# - We must copy the backup into a different directory as in *.d directories usually all files are loaded.
mkdir -p ~/defaults/etc/modprobe.d
cp -i --preserve=all --no-preserve=links /etc/modprobe.d/bumblebee.conf ~/defaults/etc/modprobe.d/

nano /etc/modprobe.d/bumblebee.conf
    # "lsmod | fgrep -i nvidia" shows that nvidia_384 is loaded because nvidia_384_modeset depends on it,
    # and nvidia_384_modeset is loaded because nvidia_384_drm depends on it, which is the only module which
    # is loaded automatically on its own.
    # Also in some instances, e.g. when using VLC with VDPAU, the module nvidia_384_uvm is loaded which
    # depends on nvidia_384.
    # So we blacklist this dependency chain in reverse - albeit I'm not sure whether the order matters. It
    # would probably be enough to just blacklist the drm module.
    #
    # We blacklist the actual module file names, not the aliases which lsmod shows. These can be determined
    # by "find / -iname '*nvidia*.ko' - ko files are kernel modules, so the part before the .ko
    # is the name.
    blacklist nvidia_384_drm
    blacklist nvidia_384_modeset
    blacklist nvidia_384_uvm
    blacklist nvidia_384
update-initramfs -u
update-grub
```

Source: [Bumblebee wiki](https://github.com/Bumblebee-Project/Bumblebee/wiki/Troubleshooting#bbswitch-is-ineffective-due-to-nvidia-driver-loading-on-boot----bbswitch-device-xxx-is-in-use-by-driver-nvidia-refusing-off)

### KDE configuration

By default KDE is configured to at least switch off the screen when the laptop lid is closed, or it will even put the machine to sleep.  
This will prevent us from using an external screen with the lid closed so we must disable it:
```
KDE menu / Computer / System Settings / Power Management /
    Energy Saving / On AC Power / When laptop lid is closed: Do nothing
```

**WARNING**: As of Kubuntu 14.04 this only applies to the currently logged in user! This means that if no user is logged in then the system will always be put to sleep when the lid is closed. That's done by LightDM, and I have not found a way to disable it yet.  
**TODO**: After dist-upgrade: If this still applies then find a way to fix it.

Notice: At least on the ThinkPad W530 even with the above KDE setting the internal screen will still be turned off when the lid is closed -  so you don't need to worry about it constantly consuming power. It is probably disabled by a hardware switch.

### bumblebee-multiscreen-tools

Now that the system drivers and Bumblebee are installed and configured you should reboot and then proceed to install `bumblebee-multiscreen-tools` as follows.

```bash
sudo -i
# Download the repository
git clone https://github.com/leo-bogert/bumblebee-multiscreen-tools ~/.bin/bumblebee-multiscreen-tools
cd ~/.bin/bumblebee-multiscreen-tools
# Download my GPG key to verify the signature on this repository as you'll run its programs as root
gpg --recv-key "1517 3ECB BC72 0C9E F420  5805 B26B E43E 4B5E AD69"
# Verify signature of most recent tag. Don't use this repository if there is no valid one!
git verify-tag "$(git describe)"
# IMPORTANT: As lots of the settings are still hardcoded first review the screen-switching script
# for whether it matches your machine's display resolution, DisplayPort, XAUTHORITY, etc.:
# TODO: Make easily configurable
nano ~/.bin/bumblebee-multiscreen-tools/switch-screen
# Add bumblee-multiscreen-tools to $PATH so you can run the scripts from anywhere
cp -i --preserve=all --no-preserve=links .bashrc .bashrc.default
nano .bashrc
    # Append to end of file:
    PATH="$PATH:$HOME/.bin/bumblebee-multiscreen-tools"
# Bash reconfiguration won't be in effect until restart of bash so log out
exit
```

**IMPORTANT**: Now that you've changed the PATH to include all binaries from the repository it is critically important to be aware of the fact that GitHub could serve you arbitrary binaries to replace standard shell commands if you update the repository without checking signatures first!  
So if you want to update it in the future, do **not** use `git pull`. Instead do:
```bash
sudo -i
cd ~/.bin/bumblebee-multiscreen-tools
git fetch
git verify-tag NEWEST_TAG_WHICH_WAS_JUST_FETCHED
git merge --ff-only NEWEST_TAG
```

#### Automatic display switching with docking station

The `dock-handler` script which we will configure as follows will automatically switch the screen to the external screen on DisplayPort #2 when the ThinkPad is docked and switch to the internal screen upon undock.  
While most of the configuration is done in the `switch-screen` script which it uses and which you were already told to review, it might still make sense to have a look at the source code of the `dock-handler` script before you proceed.  
Further, `dock-handler` will also tweak power management upon dock/undock by switching and configuring the CPU governor and enabling/disabling Intel TurboBoost. See [the introduction](https://github.com/leo-bogert/bumblebee-multiscreen-tools#switching-screen-automatically-with-a-docking-station) for details.

Ensure the extra modules package for your kernel is installed, e.g. `linux-image-extra-3.13.0-133-generic`:
```bash
dpkg --list | head -5 ; : Show table header ; dpkg --list | fgrep -i linux-image-extra
```

This is needed so the ThinkPad ACPI module will be loaded (usually automatically), which you can check by:
```bash
lsmod | fgrep -i thinkpad_acpi
```

Configure `acpid` to run the `dock-handler` script upon dock/undock (sources: [1](https://phihag.de/2012/thinkpad-docking.html), [2](http://www.thinkwiki.org/wiki/Thinkpad-acpi), [3](https://www.kernel.org/doc/Documentation/laptops/thinkpad-acpi.txt)):
```bash
sudo -i
nano /etc/acpi/events/thinkpad-series3dock-docked
    # This is a regular expression to match both IBM and Lenovo machines.
    event=ibm/hotkey (IBM|LEN)0068:00 00000080 00004010
    action=/root/.bin/bumblebee-multiscreen-tools/dock-handler docked
nano /etc/acpi/events/thinkpad-series3dock-undocked
    event=ibm/hotkey (IBM|LEN)0068:00 00000080 00004011
    action=/root/.bin/bumblebee-multiscreen-tools/dock-handler undocked
chmod 644 /etc/acpi/events/thinkpad-series3dock-*
# Make acpid reload its configurtion
killall -HUP acpid
```

#### Detecting dock state before and after suspend/hibernate

The aforementioned configuration of `acpid` to invoke `dock-handler` only is in effect when the system is docked/undocked during live operation.  
To ensure `dock-handler` is invoked related to suspend/hibernate we will now add hooks to `pm-utils`.  
They will cause:
- switching to the internal screen and default CPU governor before suspend/hibernate to ensure Bumblebee and kernel code for suspend/hibernate does not have to be bug-free with regards to these special conditions.
- executing `dock-handler` after suspend/hibernate to restore usage of the proper screen and CPU governor.

```bash
sudo -i
touch /etc/pm/sleep.d/99_thinkpad-w530.sh
chown root:root /etc/pm/sleep.d/99_thinkpad-w530.sh
chmod 755 /etc/pm/sleep.d/99_thinkpad-w530.sh
# For documentation on this .d directory see:
#     /usr/share/doc/pm-utils/README
#     /usr/share/doc/pm-utils/HOWTO.hooks.gz
# For output of this script see:
#     /var/log/pm-suspend.log*
nano /etc/pm/sleep.d/99_thinkpad-w530.sh
    #!/bin/bash
    
    # pm-utils by default will save the CPU governor before suspend/hibernate and restore it afterwards which
    # would override dock-handler's choice of governor based on whether we are connected to AC.
    # To prevent that we hereby disable the script /usr/lib/pm-utils/sleep.d/94cpufreq which would do that.
    source "${PM_FUNCTIONS}"
    disablehook 94cpufreq
    
    case "$1" in
        suspend|hibernate)
            echo "Suspend/hibernate -> Calling 'dock-handler undocked' to prepare for wakeup outside dock..."
            /root/.bin/bumblebee-multiscreen-tools/dock-handler undocked
            
            # 94cpufreq by default sets the governor to $TEMPORARY_CPUFREQ_GOVERNOR before hibernation to
            # prevent kernel bugs. We've disabled that so we must do what it did here.
            echo "Setting CPU frequency governor to $TEMPORARY_CPUFREQ_GOVERNOR for CPUs:"
            for cpu in /sys/devices/system/cpu/cpu[0-9]* ; do
                gov="$cpu"/cpufreq/scaling_governor
                # Only set governor if the file is regular: Symlink means another CPU handles the frequency.
                if [ -f "$gov" ] && ! [ -h "$gov" ]; then
                    echo "$cpu"
                    echo "$TEMPORARY_CPUFREQ_GOVERNOR" > "$gov"
                fi
            done
            ;;
        resume|thaw)
            echo "Woke up -> Calling 'dock-handler' to detect dock state and deploy events accordingly..."
            /root/.bin/bumblebee-multiscreen-tools/dock-handler
            ;;
        *)
            echo "ERROR: Unknown parameter: $1" >&2
            ;;
    esac
```

#### Detecting dock state at startup/logout and switching screen accordingly

Use ```display-setup-script``` as follows on LightDM:
```shell
# Source: https://wiki.ubuntu.com/LightDM#Adding_System_Hooks
sudo -i
touch /etc/lightdm/lightdm.conf.d/99-bumblebee.conf
chown root:root /etc/lightdm/lightdm.conf.d/99-bumblebee.conf
chmod 644 /etc/lightdm/lightdm.conf.d/99-bumblebee.conf
nano /etc/lightdm/lightdm.conf.d/99-bumblebee.conf
    # ATTENTION: Later versions of lightdm (15.10 onwards) have replaced the obsolete [SeatDefaults]
    # with [Seat:*]
    [SeatDefaults]
    display-setup-script=/root/.bin/bumblebee-multiscreen-tools/display-setup-script

```

The script will be run by LightDM right at startup of the X server. It will wait for ```bumblebeed``` to start and then run the ```dock-handler``` script.  
It will also be run by LightDM when you log out, which is important as it then restarts the X server which makes it necessary to re-choose the screen.

#### Setting the CPU governor at startup

The above `display-setup-script` configuration will also set the CPU governor at startup, depending on the dock state (`conservative` when docked, `powersave` otherwise).  
However, at least on Ubuntu 14.04, this may be overwritten by an init script about 60 seconds after startup. To check for this, see if the governor is correct by:
```bash
cat /sys/devices/system/cpu/cpu?/cpufreq/scaling_governor
```

To disable the init script which is at fault on Ubuntu 14.04, do:
```bash
sudo update-rc.d -f ondemand remove
```

Newer Ubuntu and Debian versions use `systemd` instead of SysV init, you may have to find a different approach.
Try:
```bash
fgrep -R governor /etc
```

## Testing

### Checking whether the Nvidia GPU is currently enabled

```bash
cat /proc/acpi/bbswitch
```
This will *only* determine whether the GPU is powered on, it does *not* mean that it is actually in use.  

### Checking whether rendering on the Nvidia GPU works

```bash
sudo apt install mesa-utils
# Should show the Nvidia GPU
optirun -- glxinfo | fgrep "OpenGL renderer"
# Should show the Intel GPU
glxinfo | fgrep "OpenGL renderer"
```

### Checking availability of external screens

List screens:
```bash
# On Intel GPU
xrandr --display ':0' --query
# On Nvidia GPU
# - Requires running 'switch-screen external|dual' before, possibly by dock-handler.
xrandr --display ':8' --query
```

The `':0'` and `':8'` in those commands are the display number of the X server - with Bumblebee we have two, 0 is the Intel, 8 is the Nvidia one.

Once `intel-virtual-output` was started by `switch-screen`, the external screens should be listed on the Intel GPU as `VIRTUALX` where X is a number between 1 and 9.  
Our desired target DisplayPort #2 will be `VIRTUAL8`. It will show as `DP-5` on the Nvidia GPU.

The laptop's internal screen is usually labeled as `LVDS1`.

## Usage

### Run something on the Nvidia GPU

*Notice*: This won't work when only the internal screen is enabled, bumblebee-multiscreen-tools currently always disables the Nvidia GPU then since it is assumed that usage of the internal screen means you're running on battery and hence won't play games. See [TODO.md](TODO.md) for details.

Old approach:

    optirun -- COMMAND

[New](https://askubuntu.com/questions/669011/what-is-the-difference-between-optirun-and-primusrun) and [faster](https://www.bitblokes.de/nvidia-karte-unter-linux-primusrun-optirun-auf-speed-mit-benchmark) approach:

    primusrun -- COMAND

**IMPORTANT**: With some applications either of both approaches may wrongly use the Intel GPU, try the other one if things seem very slow. Also much graphics-related software does offer a way to show info about the GPU it is using, if available always have a look at that.  
Also always try only using the Intel GPU, i.e. not using optirun/primusrun, because Bumblebee has a certain performance penalty due to copying video data across PCIe from the video buffer of the Nvidia GPU to the buffer of the Intel GPU.  
Especially on modern screen resolutions such as 1920x1080 and higher you'll run into that problem frequently. If games are slow both with the Intel and the Nvidia GPU try running them with a lower resolution on the Nvidia GPU.

### Run nvidia-settings

    optirun -b none nvidia-settings -c :8

To ensure the settings persist at the next login:
```bash
sudo -i
# Back up the file we modify in case you need to have a look at it again.
cp -i --preserve=all --no-preserve=links --parents \
	/etc/xdg/autostart/nvidia-settings-autostart.desktop ~/defaults
nano /etc/xdg/autostart/nvidia-settings-autostart.desktop
    # Replace Exec= with:
    Exec=sh -c '/usr/bin/optirun -b none nvidia-settings -c :8 --load-config-only'
```

**TODO**: I have not tested this as I don't use any of the options of `nvidia-settings`.

### Manual screen switching
```bash
# Switch to DisplayPort #2 on docking station, enable Nvidia GPU.
sudo switch-screen external
# Switch to internal screen, disable Nvidia GPU.
sudo switch-screen internal
# Enable both screens, enable Nvidia GPU.
sudo switch-screen dual
```

## Optional features

By now the system is ready for basic multiscreen usage.  
Still, there are lots of additional things possible - which the following sections will explain.

### Reducing fan noise by avoiding upclocking

The `dock-handler` script configures the CPU governor to ignore CPU load of processes with a niceness above 0 when considering whether to raise the CPU clock.  
Niceness is the Linux name for process priority, higher values equal *lower* priority.

As modern websites are stuffed to the brim with slow JavaScript it is very advisable to launch your browser with niceness.  
To launch it with niceness > 0 use `nice BROWSER_COMMAND`.

To add a KDE task bar icon for launching Firefox as such:
```
Add a secondary KDE menu shortcut for Firefox (you can also edit the existing one if you like to):
    Right click the KDE-Menu / Edit Applications .. / Right click Internet / New item / Firefox (nice):
        General / Icon (the blank button): Other icons / firefox
        General / Command: nice firefox %u
    Click the "Save" button in the toolbar.

Add a shortcut to task bar:
    Right click task bar / Unlock widgets.
    KDE Menu / Internet / Firefox (nice): right click, select "Add to panel".
    Click configuration icon of task bar at the right. You now can drag the Firefox icon to the right place.
    Right click the desktop / Lock widgets.
```

The same concept can be applied to any other application of course.

### Video acceleration

There are two established video acceleration APIs on Linux - Nvidia's [VDPAU](https://en.wikipedia.org/wiki/VDPAU) and Intel's [VA API](https://en.wikipedia.org/wiki/Video_Acceleration_API).  
We're going to install all libraries required for both APIs to work on both GPUs.

```shell
# Intel GPU - VA API
sudo apt install libva-intel-vaapi-driver i965-va-driver
# Intel GPU - VDPAU
sudo apt install libvdpau-va-gl1 i965-va-driver
# Nvidia GPU - VDPAU
sudo apt install libvdpau1
# Nvidia GPU - VA API
sudo apt install vdpau-vadriver
```

You must add any user account which wants to use the Intel GPU for video acceleration to the ```video``` group:
```shell
sudo adduser USER video
```
As the Intel GPU will be what is used by default for rendering it is recommended to add all your user accounts to that group.  
If you don't do that then programs such as Chromium/Firefox will print this when launched from a shell:
```
libGL error: failed to open drm device: Permission denied
libGL error: failed to load driver: i965
```
This also shows that the ```video``` group may even be required for access to OpenGL in general, not just video acceleration.

**TODO**: Explain how to test whether video acceleration works as explained in German on the [Ubuntuusers.de wiki](https://wiki.ubuntuusers.de/Video-Dekodierung_beschleunigen/#Test).  
**TODO**: While looking for these packages I noticed that searching aptitude for "vaapi" yields the fact that "gstreamer", which is installed on my machine, also has VAAPI plugins which are not installed. Check whether this is used by anything important such as Firefox/Chromium and if yes install the VAAPI plugins.  
**TODO**: Also check for packages for the competing API "vdpau".

Now that we've installed video acceleration libraries we will have to configure various software to actually use them.  
Instructions for Firefox, Chromium, VLC and the Flash player follow.

#### Firefox

We'll use the Intel GPU as browsing isn't a high performance task and using the Intel GPU is more easy as it is always enabled.  
So make sure the user which runs Firefox is in the ```video``` group as explained [above](#video-acceleration).

Browse to ```about:config``` and set:
```
layers.acceleration.disabled: false
layers.acceleration.force-enabled: true
media.hardware-video-decoding.enabled: true
media.hardware-video-decoding.force-enabled: true
```

To check whether it works, browse to ```about:support```.
It should say:
```
Graphics /
  Features /
    Compositing: OpenGL
    WebGL 1 Driver Renderer: Intel Open Source Technology Center ...
    WebGL 2 Driver Renderer: Intel Open Source Technology Center ...
  GPU #1 /
    Active: Yes
    Description: Intel Open Source Technology Center ...
  Decision Log /
    HW_COMPOSITING:
      blocked by default: Acceleration blocked by platform
      force_enabled by user: Force-enabled by pref
    OPENGL_COMPOSITING:
      force_enabled by user: Force-enabled by pref
```

**TODO**: This only enables HTML rendering to be accelerated, not video rendering:  
As of 2018, some googling showed that video acceleration not working in e.g. Firefox isn't really an issue of the GPU-related configuration.  
It's rather that Mozilla don't seem keen on supporting video acceleration on Linux, there are open ancient entries like [this](https://bugzilla.mozilla.org/show_bug.cgi?id=563206), [this](https://bugzilla.mozilla.org/show_bug.cgi?id=1210726), [this](https://bugzilla.mozilla.org/show_bug.cgi?id=1210727) and [this](https://bugzilla.mozilla.org/show_bug.cgi?id=1210729) in the bugtracker about implementing it - which looks like it never got beyond the "planned feature" stage.

#### Chromium

We'll use the Intel GPU as browsing isn't a high performance task and using the Intel GPU is more easy as it is always enabled.  
So make sure the user which runs Chromium is in the ```video``` group as explained [above](#video-acceleration).

Browse to ```chrome://flags``` and set:
```
Override software rendering list: On
GPU rasterization: On
```
To check whether it works, browse to ```chrome://gpu```.  

**TODO**: As of 2018 it is likely that even though Chromium says it does use hardware decoding it doesn't actually do so.  
Check for example [this](https://www.pcsuggest.com/chromium-hardware-accelerated-video-decoding-linux/), [this](https://old.reddit.com/r/linux/comments/60o1l1/chrome_on_linux_needs_to_support_hw_video/) and [this](https://chromium-review.googlesource.com/c/chromium/src/+/532294):  
At the former links people say that overriding the blacklist just means that it will try to use GPU acceleration, but it won't actually do so because it isn't compiled into it.  
The later link shows that a pull request for VAAPI support wasn't merged yet (indicated by "Merge Conflict" which shows that it couldn't even be merged right now because of unresolved Git merge conflicts).

#### VLC

We'll use the Intel GPU as video decoding isn't a high performance task and using the Intel GPU is more easy as it is always enabled.  
So make sure the user which runs VLC is in the ```video``` group as explained [above](#video-acceleration).

In ```Tools / Preferences``` set:
```
Simple settings / "Input / codecs" /
	Hardware-accelerated decoding: Video acceleration (VA) API
```

Close VLC. Start it from a terminal with a video file as parameter. The output should indicate hardware rendering by e.g.:
```
[...] avcodec decoder: Using VA API version 0.35 for hardware decoding. [...]
```

**TODO**: Newer versions of VLC may support VDPAU and thus could be used with the Nvidia GPU.  
However as of 2018 the Intel GPU seems fast enough for very high resolution h264 videos.

#### Flash player

_I don't use the Flash player anymore so this is untested!_

We'll use the Intel GPU as video decoding isn't a high performance task and using the Intel GPU is more easy as it is always enabled.  
So make sure the user which runs the browser is in the ```video``` group as explained [above](#video-acceleration).  

In a terminal, do:
```bash
sudo -i
mkdir /etc/adobe
nano /etc/adobe/mms.cfg
    EnableLinuxHWVideoDecode=1
    OverrideGPUValidation=true
chmod o+rx /etc/adobe
chmod o+r /etc/adobe/mms.cfg
```

To check whether this is working, right click a flash video and select ```Stats for nerds```.

If the Flash plugin crashes often, try without the previous ```/etc``` configuration change, i.e. only install ```libvdpau1``` as [aforementioned](#video-acceleration).  
That will make it only use hardware rendering but not hardware decoding.

Source: [ubuntuusers.de](http://wiki.ubuntuusers.de/Adobe_Flash)

## Donations

[Bitcoin](https://bitcoin.org): 1JdewDzqhfwtDtbzmBbLzJXVWCF6ZH82bm

## Known issues and workarounds

### General troubleshooting

If you run into a situation where you have no video output at all or limited video output (e.g.
one of external/internal video not working) you have these options to recover it:
- trigger a reconfiguration of the video outputs by rerunning the  ```dock-handler``` script by
  undocking and docking again.
  This will resolve the most video output issues!
- switch to terminal mode with ```CTRL+ALT+FX``` where FX is one of the F-keys. Terminals are
  usually attached to F1 to F7, F8 switches back to Xorg. This may vary across distributions, try
  all the F-keys.
- kill the X server with ```ALT+SysRq+K```. The SysRq key is also known as the PRINT key. Notice
  that on some distributions this is disabled by default any might need to be enabled by e.g.:
  ```
  nano /etc/sysctl.d/10-magic-sysrq.conf
      kernel.sysrq = 244
  reboot
  ```

Once you've recovered video output you can follow this procedure to get video output again:
1. read the log files:
  - ```/var/log/thinkpad-w530-dock.log``` - the ```dock-handler``` script's log which also contains
    output of the attempts to switch the screen. Notice that this has to be enabled in the script.
  - ```/var/log/Xorg.0.log``` - Intel X-server log
  - ```/var/log/Xorg.8.log``` - Bumblebee X-server log
  - ```/var/log/lightdm/lightdm.log``` - LightDM display manager log
  - ```/var/log/syslog```
  - ```/var/log/kern.log```
2. edit the configuration files.
3. Retry by either:
  - killing the X-servers and IVO using ```killall Xorg ; killall intel-virtual-output```,
    restarting Bumblebee using ```service bumblebeed restart```, and rerunning the dock-handler.
  - rebooting using ```reboot```.

### Parts of screen are black, KDE tray icons are misplaced

These rendering errors go away on their own by moving windows around a bit.

You can e.g. try to de-maximize a window and maximize it again.

#### Alternative workaround

Enable compositing in KDE = render KDE using OpenGL:

    System settings / Desktop effects / Enable desktop effects at startup: On

**WARNING:**

- Bumblebee usually recommends to disable it, I don't know why, some very basic googling says its due to performance.

- I have not tested this very much.

If you need to use `primusrun` with compositing and see flickering or bad performance, synchronizing the display thread with the application's rendering thread may help according to [this](https://wiki.archlinux.org/index.php?title=Bumblebee&oldid=528317#Primus_issues_under_compositing_window_managers):

    PRIMUS_SYNC=1 primusrun ...

Or just disable compositing on demand with the hotkey as mentioned by the system settings: `ALT + SHIFT + F12`.

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
Meanwhile retrying when the system is idle again may work (e.g. undock, ensure the system is idle,
and dock again), otherwise a reboot will fix the issue.

### Mouse cursor issues

* When using the external screen the mouse cursor may sometimes not be visible.
* After disconnecting an USB mouse and re-connecting it the cursor may jump around in a strange fashion when you move it.

Any such issues can usually be fixed by switching to the internal screen and then back to the external one,
e.g. by undocking and docking again.

They can be avoided by:
- ensuring the mouse is not ever reconnected implicitly when using the external screen: Reconnects can happen when you use the external screen as an USB hub and connect the mouse to it: Some screens will shut off the USB ports when you turn the screen off.
- Also the `ThinkPad Mini Dock Plus Series 3` is a rather jittery construction, touching it only slightly will usually cause a short undock/dock cycle. Better not touch it while the system is running.

### libGL error: failed to open drm device: Permission denied
### libGL error: failed to load driver: i965

You forgot to [add your user account to the video group](#video-acceleration).

## Similar tutorials / tools / sources

- https://github.com/Bumblebee-Project/Bumblebee/wiki/Multi-monitor-setup
- https://askubuntu.com/questions/103564/external-displays-with-bumblebee/525404#525404
- https://www.unixreich.com/blog/2013/linux-nvidia-optimus-on-thinkpad-w520w530-with-external-monitor-finally-solved/
- https://github.com/Witko/nvidia-xrun
	- Never tried it as it requires starting a separate X server from a tty (= terminal, i.e. by switching away from X to raw terminals using CTRL+ALT+F1) which is very inconvenient. Also it probably isn't developed with taking laptops into account, i.e. probably doesn't work with their external video ports.

## License

Do whatever you want to do with this. Relicense as you please. No warranty, especially not with regards to damage to your hardware, your house burning down, whatever.
