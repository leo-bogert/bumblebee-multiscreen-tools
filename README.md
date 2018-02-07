## bumblebee-multiscreen-tools
### Scripts to do screen and GPU switching on NVidia Optimus laptops with [Bumblebee](https://github.com/Bumblebee-Project/Bumblebee).

These scripts allow you to easily switch your laptop's screen from internal to external or enable both screens at once.  
They should work with a docking station.

They've been written for a Lenovo ThinkPad W530 and the "Lenovo ThinkPad Mini Dock Plus Series 3" docking station.  
Other devices are not tested but may work, these scripts are not very hardware dependent.

### WARNING

This is work in progress! Read the scripts first before running them!  
**They contain quite a bit of harcoded configuration, you MUST really read the whole scripts!**

They will be finished in 2018, check for new commits regularly.  
A proper README with instructions on how to configure Bumblebee will be provided as well.

### Depedencies

- The ```switch-screen``` script assumes the display manager is LightDM (which is e.g. used by Kubuntu 14.04). If your distribution uses a different one you need to change the XAUTHORITY variable there.

### Usage

#### Manual screen switching

```shell
# Read the script and adapt its hardcoded configuration to your environment first!
nano switch-screen
# Switch to DisplayPort #2 on docking station.
# This enables the NVidia GPU as the DisplayPort can only be accessed through it.
# Software will keep using the Intel GPU unless you run it with "optirun COMMAND_NAME".
sudo switch-screen external
# Switch to internal screen.
# This disables the NVidia GPU for optimal power usage.
sudo switch-screen internal
# Enable both screens with laptop screen being the primary screen, left of the external one.
# This enables the NVidia GPU just like external mode.
sudo switch-screen dual
```

#### Switching screen automatically with a docking station

If you have a docking station, in particular the "Lenovo ThinkPad Mini Dock Plus Series 3", you can use the ```dock-handler``` script as an ACPI event handler.  
Again, please read and configure the script before using it!

It will automatically switch the screen to the external screen when docked, and to the internal screen when undocked.  
Further, the following powersaving and noise-related optimizations are done by the script:
* When undocked:
  * the NVIDIA GPU is disabled by ```switch-screen```.
  * the "powersave" [CPU governor](https://www.kernel.org/doc/Documentation/cpu-freq/governors.txt) is used to permanently downlock the CPU to its lowest speed.
  * [Intel Turbo Boost](https://en.wikipedia.org/wiki/Intel_Turbo_Boost) (automatic load-based overclocking) is disabled.
* When docked:
  * the NVIDIA GPU is enabled. While this makes sense from a "we're not on battey so no need to save power" perspective it is also strictly necessary: On many laptops, e.g. the ThinkPad W530, the external video outputs are only available through the NVIDIA GPU.
  * the "conservative" CPU governor is used. Compared to the default "ondemand" governor this will not instantly clock the CPU to its maximum speed under load but slowly uplock if the load persists. With the ThinkPad W530 this greatly reduces noise when e.g. browsing JavaScript-heavy websites. It is configured as such:
    * if above 80% CPU load for some time upclocking happens.
    * if below 50% CPU load for some time downclocking happens.
    * CPU load of processes with niceness > 0 is ignored when considering whether to upclock. Niceness is the Linux name for process priority, higher values equal lower priority. To launch a command with the default niceness of 10 use ```nice command_name```.
  * Intel Turbo Boost is enabled

### Known issues and workarounds

#### External screen not working

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

#### Mouse cursor issues

- When using the external screen the mouse cursor may sometimes not be visible.
- After disconnecting an USB mouse and re-connecting it the cursor may jump around in a strange fashion when you move it.

Any such issues can usually be fixed by switching to the internal screen and then back to the external one.

### License

Do whatever you want to do with this. Relicense as you please. No warranty.
