## High priority
- Anything marked with `FIXME` in all the other files.

- The Nouveau drivers provide [much more simple configuration of Optimus](https://nouveau.freedesktop.org/wiki/Optimus/) than Bumblebee. They should also be a lot faster because they probably don't copy around data in memory as much as intel-virtual-output does. Create a copy / branch of this repository which uses Nouveau. If it works nicely put a big fat warning into the README.md to use Nouveau instead (notice that it is already mentioned there, replace that text).

- Allow using the Nvidia GPU with ```optirun``` / ```primusrun``` when having switched to the internal screen using ```switch-screen```, e.g. by adding a screen parameter ```internal-with-nvidia```. Currently the Nvidia GPU is always disabled when only the internal screen is enabled as that is assumed to be the "on battery" mode. Notice: This FIXME is duplicated [here](https://github.com/leo-bogert/bumblebee-multiscreen-tools#run-something-on-the-nvidia-gpu).

- What do we do with the `nvidia-persistenced` daemon? Should switch-screen terminate it if the Nvidia card isn't enabled?

- Update wikis to mention bumblebee-multiscreen-tools once development is sufficiently finished (which it is not yet!):
  - https://github.com/Bumblebee-Project/Bumblebee/wiki/Multi-monitor-setup
  - https://wiki.ubuntuusers.de/Hybrid-Grafikkarten/Bumblebee/
    - "Multimonitorbetrieb" section says that multiscreen mode doesn't work, but in fact with bumblebee-multiscreen-tools it is possible

## Low priority
- Anything marked with `TODO` in all the other files.

- Renice Xorg, perhaps helps with vsync issues

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