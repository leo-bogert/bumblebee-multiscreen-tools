## High priority
- Anything marked with `FIXME` in all the other files.

- Document the BIOS settings

- The Nouveau drivers provide [much more simple configuration of Optimus](https://nouveau.freedesktop.org/wiki/Optimus/) than Bumblebee. They should also be a lot faster because they probably don't copy around data in memory as much as intel-virtual-output does. Create a copy / branch of this repository which uses Nouveau. If it works nicely put a big fat warning into the README.md to use Nouveau instead (notice that it is already mentioned there, replace that text).

- Allow using the Nvidia GPU with ```optirun``` / ```primusrun``` when having switched to the internal screen using ```switch-screen```, e.g. by adding a screen parameter ```internal-with-nvidia```. Currently the Nvidia GPU is always disabled when only the internal screen is enabled as that is assumed to be the "on battery" mode. Notice: This FIXME is duplicated [here](https://github.com/leo-bogert/bumblebee-multiscreen-tools#run-something-on-the-nvidia-gpu).

	- To make this work automatically switch the Nvidia GPU on when on AC power. Implement this by putting a script into `/etc/pm/power.d/` to enable/disable the Nvidia GPU if AC is plugged/unplugged without a docking station being involved (and also set the CPU governor there). See `/usr/share/doc/pm-utils/HOWTO.hooks.gz`.  
	WARNING: This will be in a race condition with our scripts `/etc/acpi/events/thinkpad-series3dock-*`, `/etc/pm/sleep.d/99_thinkpad-w530.sh` and `/root/.bin/bumblebee-multiscreen-tools/display-setup-script`. It could be resolved by having the new script wait for something like 10 seconds, then check if a docking station is present, and if yes do nothing.

- What do we do with the `nvidia-persistenced` daemon? Should switch-screen terminate it if the Nvidia card isn't enabled?

- Test whether this still works on Ubuntu 16.04. If it doesn't here are some related links:
  - http://danielteichman.blogspot.de/2017/03/how-to-install-bumblebee-on-ubuntu-1604.html
  - https://github.com/Bumblebee-Project/Bumblebee/issues/759

- Update wikis to mention bumblebee-multiscreen-tools once development is sufficiently finished (which it is not yet!):
  - https://github.com/Bumblebee-Project/Bumblebee/wiki/Multi-monitor-setup
  - https://wiki.ubuntuusers.de/Hybrid-Grafikkarten/Bumblebee/
    - "Multimonitorbetrieb" section says that multiscreen mode doesn't work, but in fact with bumblebee-multiscreen-tools it is possible

## Low priority
- Anything marked with `TODO` in all the other files.

- Permanently downclock the Nvidia GPU to reduce fan noise. Here's my previous ThinkPad T61p manual for this, it may be possible to recycle this for the W530 by using Bumblebee's Nvidia Xorg config file instead:
	```bash
	cp -i --preserve=all --no-preserve=links /etc/X11/xorg.conf /etc/X11/xorg.conf.default
	nano /etc/X11/xorg.conf
		Section "Device"
			Option "Coolbits" "1"
			Option "RegistryDwords" "PowerMizerEnable=0x1; PerfLevelSrc=0x2222; PowerMizerLevel=0x3; PowerMizerDefault=0x3; PowerMizerDefaultAC=0x3"
		EndSection
	```
	- Source: http://wiki.etechnik-rieke.de/index.php/NVidia_PowerMizer
	- TODO: Source: https://guilleml.wordpress.com/2011/04/27/nvidia-powermizer-on-linux/
	- TODO: Doesn't work on later drivers according to https://devtalk.nvidia.com/default/topic/572053/linux/-solved-forcing-maximum-power-saving-on-the-desktop-quot-minimum-power-quot-mode-for-powermizer-/ -> validate this & find a new approach
	- TODO: Perhaps the easiest approach would be to do this by shell, not by Xorg settings - then you can also switch it manually when you want to play a game: https://bbs.archlinux.org/viewtopic.php?id=162960   - found by google: "nvidia switch power level command line"

- Renice Xorg, perhaps helps with vsync issues

- Move inlined scripts of README.md to files. Notice that we add the repository root to $PATH so use long names.

- Split this up into multiple repositories:
	- thinkpad-dock-tools:
		- dock-handler script. Will call all scripts in /etc/thinkpad-dock-tools/(un)dock.d
		- /etc/acpi/events/thinkpad-series3dock_(un)docked to trigger the execution of the dock-handler script.
	- bumblebee-multiscreen-tools:
		- switch-screen script to switch the screen.
		- /etc/thinkpad-dock-tools/(un)dock.d/choose-screen script as hook to call switch-screen when dock state changes.
	- thinkpad-pm-tools:
		- /etc/pm/sleep.d/thinkpad script to handle hibernation/suspend and wakeup. Shall trigger the execution of the dock-handler script and thereby /etc/thinkpad-dock-tools/(un)dock.d - which ensures screen switching happens before and after suspend/hibernate. Also shall set the CPU governor before hibernation and after wakeup.
		- /etc/pm/power.d/thinkpad script to handle switching between AC and battery power. Shall also call the dock handler and deal with the CPU governor.
		
		See /usr/share/doc/pm-utils/HOWTO.hooks.gz for how to do the above using /etc/pm/

- Add table of contents to README.md once GitHub markdown supports automatically generating it.